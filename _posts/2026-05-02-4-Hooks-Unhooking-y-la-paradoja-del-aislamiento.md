---
title: "4. Hooks, Unhooking y la paradoja del aislamiento"
date: 2026-05-02 12:00:00 -0500
categories: ["Fundamentos de Ejecución en Windows: PEB, Gestión de Memoria y Técnicas de Hooking."]
tags: [windows, hooks, unhooking, defender, amsi]
---

Hemos establecido que existe un perímetro estricto entre el modo usuario y el kernel. Hemos visto cómo la memoria virtual aísla los procesos, y cómo `ntdll.dll` construye el puente controlado hacia el kernel. Teniendo en cuenta este diseño, ¿cómo opera un antivirus como Windows Defender?

Si necesita vigilar a un programa malicioso, se enfrenta a un proceso que reside en su propia isla de memoria virtual en el anillo 3, totalmente convencido de que es intocable — y que, como acabamos de ver, es capaz de mentir sobre su propio PEB.

Es un problema arquitectónico enorme para los defensores. La solución que encontró Microsoft requiere un diseño de tres niveles.

El primero es la **interfaz gráfica** — el panel de control que tú ves, el icono verde que te dice que tu sistema está protegido. Vive en el anillo 3, no tiene poder real, solo comunica.

El segundo es un **servicio de fondo**, también en el anillo 3 pero con privilegios de sistema elevados, que recibe alertas y coordina respuestas.

Pero el verdadero núcleo de la vigilancia es el tercero: un **driver** (`WdFilter.sys`) — un controlador que se instala permanentemente en el modo kernel, en el anillo 0. Al vivir ahí, el antivirus adquiere lo que podríamos llamar la vista de Dios sobre el sistema. No tiene que pedir permiso para nada. Puede auditar absolutamente todo.

Desde el anillo 0, el driver despliega múltiples mecanismos simultáneamente. Instala **filtros en el sistema de archivos** — minifilters — de forma que cada vez que una aplicación realiza cualquier operación de I/O sobre un archivo: leerlo, escribirlo, crearlo, renombrarlo o borrarlo, esa operación debe pasar físicamente por la inspección del antivirus en el kernel antes de llegar al disco. También utiliza **ETW** (Event Tracing for Windows), un sistema de telemetría masivo que le permite monitorear en tiempo real exactamente cuándo el image loader crea un nuevo proceso o mapea una nueva DLL en memoria.

Pero aquí aparece una limitación física. Si un programa descarga un archivo cifrado desde Internet, en la red se ve como ruido. Cuando ese archivo se escribe en el disco a través de los filtros del kernel, sigue siendo ruido cifrado. El único lugar en todo el sistema donde ese archivo se descifra y se convierte en código ejecutable real es dentro de la memoria privada del proceso, en el anillo 3 — justo milisegundos antes de ejecutarse.

A esto se le llama **la paradoja del aislamiento**.

Para ver qué está haciendo realmente el código en su estado puro y descifrado, el antivirus no puede quedarse sentado únicamente en el kernel. Tiene que infiltrarse en el modo usuario. Y para lograrlo, sistemas de seguridad como Defender inyectan sus propias DLLs — como `amsi.dll` (Antimalware Scan Interface) — directamente dentro del espacio de memoria privada de los procesos que usan motores de scripting: PowerShell, VBScript, JScript, macros de Office. Cualquier entorno donde código arbitrario pueda ejecutarse de forma interpretada.

La librería del antivirus localiza las funciones de `ntdll.dll` que el malware usaría para hacer daño — específicamente las funciones `Nt*` de bajo nivel, como `NtWriteVirtualMemory`, que es la que finalmente ejecuta cualquier escritura en memoria de otro proceso. Localiza el punto exacto en memoria donde comienza esa función y sobreescribe sus primeros bytes con una instrucción `JMP` — un salto incondicional en ensamblador que redirige la ejecución hacia el motor de análisis del antivirus.

A esto se le conoce como **hook** o gancho. Es literalmente un letrero de desvío insertado en el código.

Cuando el proceso intenta usar `NtWriteVirtualMemory`, el procesador lee los primeros bytes, se encuentra con el `JMP` y es forzado a saltar primero hacia el escáner del antivirus. Si los datos son maliciosos, bloquea la operación. Si son benignos, ejecuta un **trampolín** — los bytes originales guardados — y permite que la función continúe normalmente.

Y aquí es donde la arquitectura se vuelve en contra del propio sistema de seguridad. El espacio de memoria del anillo 3 le pertenece al proceso. El antivirus inyectó `amsi.dll` y modificó los bytes de `ntdll.dll` para poner su letrero de desvío — pero lo hizo dentro de la memoria del proceso. Y el proceso tiene permisos de lectura y escritura sobre su propia memoria. No puede ser de otra forma — es su casa.

Entonces nada impide que el malware haga esto:

1. Lea los primeros bytes de `NtWriteVirtualMemory` en memoria — ve el `JMP` que puso el antivirus
2. Obtenga una copia limpia de `ntdll.dll` — ya sea leyéndola directamente del disco, o mapeando una segunda instancia limpia en memoria usando `NtCreateSection` y `NtMapViewOfSection`, sin pasar por ningún hook
3. Compare ambas versiones y extraiga los bytes originales
4. Sobreescriba el `JMP` con los bytes originales

El hook desapareció. `amsi.dll` sigue cargada en memoria, sigue creyendo que vigila — pero su letrero de desvío ya no existe. El malware llama directamente a `NtWriteVirtualMemory` sin pasar por ningún escáner. El antivirus nunca se entera.

No hubo alarma. No hubo syscall sospechosa. No se tocó el kernel — y no es que el malware eligiera no tocarlo tácticamente, es que simplemente no puede: sin un driver firmado por una CA reconocida por Windows, el anillo 0 es inalcanzable desde modo usuario.

A esta maniobra se le conoce como **unhooking**.

La próxima vez que el malware llame a la función, la instrucción de salto ya no existe. El código fluye hacia la función original de Windows sin alertar jamás a las herramientas de escaneo. El malware se vuelve invisible desde adentro de su propia prisión de memoria virtual.

El punto ciego es inherente al diseño base del sistema operativo. Si le das la memoria a un proceso, este la controla.