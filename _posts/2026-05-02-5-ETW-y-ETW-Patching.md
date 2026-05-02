---
title: "5. ETW y ETW Patching"
date: 2026-05-02 12:00:00 -0500
categories: ["Fundamentos de Ejecución en Windows: PEB, Gestión de Memoria y Técnicas de Hooking."]
tags: [windows, etw, patching, evasion, telemetria]
---

#### ¿Qué es ETW?

**ETW** (Event Tracing for Windows) es la infraestructura de telemetría del sistema operativo. No es una herramienta de seguridad por sí sola — es un mecanismo de observabilidad construido dentro del kernel de Windows que permite registrar eventos del sistema en tiempo real con overhead mínimo.

Su arquitectura tiene tres componentes:

- **Providers** — cualquier componente del sistema que emite eventos: el kernel, ntdll, el CLR de .NET, PowerShell, drivers. Cada provider tiene un GUID único que lo identifica.
- **Sessions** — canales de transporte que reciben eventos de uno o más providers y los entregan a los consumers.
- **Consumers** — los receptores: herramientas de análisis, EDRs, el propio Defender.

Microsoft Defender y los EDRs modernos se suscriben como consumers a sesiones ETW críticas — especialmente **Microsoft-Windows-Threat-Intelligence**, un provider de kernel que emite eventos para operaciones sensibles: allocación de memoria ejecutable, creación de threads remotos, mapeo de secciones. Este provider específico opera desde el kernel y sus eventos no pasan por modo usuario.

Pero ETW también tiene providers en modo usuario. Y ahí está la grieta.

---

#### Cómo ETW registra eventos desde modo usuario

Cuando un proceso en modo usuario emite un evento de telemetría — por ejemplo, cuando PowerShell registra qué script está ejecutando — lo hace a través de una función que vive en `ntdll.dll`:

```
EtwEventWrite()
```

Esta función es el punto de emisión universal para cualquier provider de modo usuario. Cuando el proceso la llama, ETW empaqueta el evento y lo envía a las sesiones suscritas a través del kernel. Los consumers — incluyendo Defender — reciben ese evento en tiempo real.

El flujo es exactamente análogo al de las syscalls que vimos en el módulo 1:

```
Proceso (anillo 3)
    ↓ llama a EtwEventWrite()
ntdll.dll
    ↓ empaqueta el evento
    ↓ syscall al kernel
Kernel
    ↓ distribuye el evento a las sesiones suscritas
EDR / Defender
    ↓ recibe el evento y analiza el comportamiento
```

---

#### El error arquitectónico: ETW patching

El patrón debería ser familiar. `EtwEventWrite()` vive en `ntdll.dll`. `ntdll.dll` está mapeada en el espacio de memoria virtual del proceso — en modo usuario. El proceso tiene permisos de lectura y escritura sobre su propia memoria.

Lo que demostró el módulo 4 con el unhooking de AMSI aplica aquí con precisión quirúrgica.

El malware localiza `EtwEventWrite()` en la copia de `ntdll.dll` mapeada en su propio proceso y sobreescribe sus primeros bytes con:

asm

```asm
xor eax, eax    ; pone el valor de retorno en 0 (ERROR_SUCCESS)
ret             ; retorna inmediatamente
```

Dos instrucciones. Cuatro bytes. La función ahora retorna inmediatamente sin emitir ningún evento, devolviendo un código de éxito para no levantar sospechas.

El resultado es estructuralmente idéntico al unhooking — pero invertido en propósito. El unhooking eliminaba un hook que el antivirus puso para interceptar el malware. El ETW patching elimina el mecanismo que el proceso usa para reportar su propia actividad al sistema.

```
Antes del patch:
Proceso → EtwEventWrite() → kernel → EDR ve el evento

Después del patch:
Proceso → EtwEventWrite() → xor eax, eax; ret → nadie ve nada
```

No hubo syscall sospechosa. No se tocó el kernel. Solo una escritura en memoria propia — el mismo principio que el unlinking del PEB del módulo 3, la misma mecánica que el unhooking del módulo 4.

---

#### Qué queda ciego y qué no

ETW patching neutraliza la telemetría de **modo usuario** del proceso que lo aplica. Pero como establecimos, ETW tiene dos mundos distintos:

**Lo que queda ciego:**

- Eventos emitidos por providers de modo usuario dentro del proceso patcheado
- La visibilidad de AMSI sobre scripts, ya que AMSI usa ETW para reportar sus escaneos
- Cualquier consumer que dependa de esos eventos para detección comportamental

**Lo que sobrevive:**

- **Microsoft-Windows-Threat-Intelligence** — este provider opera exclusivamente desde el kernel. Sus eventos se generan directamente en el kernel cuando ocurren operaciones como `NtAllocateVirtualMemory` con permisos de ejecución, `NtMapViewOfSection`, o `NtCreateThreadEx`. El proceso de modo usuario no tiene acceso a silenciar este canal.
- **Kernel callbacks** — `PsSetCreateProcessNotifyRoutine`, `PsSetLoadImageNotifyRoutine` — estos se ejecutan en el kernel cuando ocurren eventos de ciclo de vida de procesos y módulos, independientemente de cualquier patch en modo usuario.
- **WdFilter.sys minifilter** — las operaciones de I/O en el filesystem siguen siendo interceptadas a nivel de kernel antes de llegar al disco.

---

#### La consecuencia estratégica

ETW patching combinado con unhooking de AMSI representa la neutralización completa de la capa de observabilidad de modo usuario. Un proceso que aplica ambas técnicas ha eliminado:

- Los hooks de AMSI sobre funciones Nt* de ntdll
- Su propia emisión de telemetría ETW

Lo que queda operativo son exclusivamente las defensas que viven en el kernel — y acceder a ellas requiere un driver firmado, exactamente el límite arquitectónico que el módulo 4 estableció como infranqueable desde modo usuario.

El punto ciego que el módulo 4 describió como inherente al diseño se profundiza: no solo el proceso puede mentir sobre sus módulos cargados a través del PEB, no solo puede eliminar los hooks que el antivirus puso — también puede silenciar su propia voz en el sistema de telemetría, volviéndose opaco para cualquier consumer que dependa de esa señal.

La arquitectura de Windows NT fue diseñada bajo la premisa de que las aplicaciones no deben poder dañar el sistema operativo. Lo que estos cuatro módulos han construido es la demostración de que esa premisa tiene un corolario inevitable: un proceso que controla su propia memoria controla su propia narrativa ante el sistema de seguridad.