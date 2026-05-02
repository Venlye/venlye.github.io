---
title: "1. Qué hay detrás de abrir una aplicación..."
date: 2026-05-02 12:00:00 -0500
categories: ["Fundamentos de Ejecución en Windows: PEB, Gestión de Memoria y Técnicas de Hooking."]
tags: [windows, syscalls, kernel, api]
---

- **¿Qué clase de magia negra hay detrás de simplemente abrir una aplicación?**
- **¿Qué pasa dentro de nuestro sistema para que pueda asegurar: "okey, este programa es seguro, adelante"?**
- **¿Qué comprobaciones pasa este programa para que nuestra PC pueda decir: ¡Ey, es seguro!, que ejecute todo lo que necesite ejecutar para su correcto funcionamiento**

La realidad técnica es un sistema super paranoico.... en este artículo podrás leer mi comprensión del modelo de seguridad del sistema operativo de Windows.

Cuando se creó Windows NT (New Technology), que en pocas palabras fue el Windows que creó los cimientos para que Windows pueda usarse a nivel industrial, creando los dominios de protección jerárquica también llamado "Protection ring", se creó bajo la premisa de:

> "El sistema debe protegerse a sí mismo tanto de fallos internos como de manipulaciones externas". El diseño estipula como regla que **"las aplicaciones no deben ser capaces de dañar el sistema operativo u otras aplicaciones"** 

![[Pasted image 20260502043748.png|491]]
Se crearon fronteras muy estrictas para proteger al SO, el famoso modo usuario (anillo 3) y modo kernel (anillo 0).

En **el anillo 3** vive todo lo que comúnmente conocemos, nuestro videojuego favorito, nuestro navegador web, el bloc de notas, etc, lo podemos imaginar como un perímetro restringido que tiene **prohibido comunicarse** con el hardware.

En el anillo 0 podemos encontrar, el kernel, de nuestro sistema, el **intermediario** entre el software y el hardware. Puede leer cualquier parte de la memoria y ejecutar cualquier instrucción del procesador sin que nadie le pregunte nada. Pero, esto me plantea un problema logístico bastante evidente. 

Una aplicación, por ejemplo, un simple bloc de notas en el anillo 3, por muy aislada que esté, eventualmente necesita abrir, modificar o guardar un archivo en el disco duro, **tiene que interactuar con el hardware de alguna forma.**

Entonces si antes mencionamos que tiene **prohibido comunicarse con el hardware**, *¿Cómo lo hace?* 

Bien, lo hace por system calls, su instrucción en la CPU tiene el nombre de: SYSCALL que es simplemente un mecanismo que permite al programa que está "encerrado" en la capa 3, solicitar servicios de acceso al hardware como gestión de archivos y procesos directamente a nuestro kernel, de esta manera se busca garantizar la estabilidad del sistema. 

El procesador tiene una instrucción específica, se llama al SYSCALL y este actúa como un puente temporal. La aplicación emite esta instrucción para invocar una función interna del sistema.
Y de esto se encarga el  ``ntdll.dll``.

Pero antes de llegar a `ntdll.dll`, hay una capa intermedia que usamos constantemente sin saberlo: la **Windows API** (también llamada WinAPI o Win32 API).

Piénsala como el **menú de un restaurante**. Tú no vas a la cocina a cocinar tu plato (el hardware), simplemente pides al mesero (la API) lo que quieres, y él se encarga del resto. Funciones como `CreateFile()`, `ReadFile()`, o `OpenProcess()` son ese menú — están escritas en lenguaje humano y ocultan toda la complejidad interna del sistema.

Estas funciones viven principalmente en una DLL llamada **`kernel32.dll`**. Es la cara amigable de Windows: la biblioteca que tu aplicación conoce y con la que habla directamente. Cuando tu bloc de notas quiere guardar un archivo, llama a `CreateFile()` de `kernel32.dll`. Sin embargo, `kernel32.dll` no habla directamente con el kernel porque no es su responsabilidad — fue diseñada para hablar con aplicaciones, no con el núcleo del sistema, tiene que pedirle el favor a alguien más.

Ese alguien es `ntdll.dll`.

`kernel32.dll` toma tu solicitud de `CreateFile()` y la traduce a su equivalente nativo: `NtCreateFile()`, que vive en `ntdll.dll`. Aquí es donde termina el lenguaje amigable y empieza el lenguaje que el kernel realmente entiende.

Dentro de `ntdll.dll`, cada función `Nt*` es lo que se llama un **stub** — no es un programa complejo, es un fragmento de código mínimo, de apenas unas pocas instrucciones, cuyo único trabajo es preparar los argumentos correctos y lanzar la instrucción `SYSCALL`. Es como un botón de llamada de ascensor: no sabe nada de mecánica, solo sabe qué piso apretar.

Para apretar el piso correcto, ese stub necesita un número. Ese número se llama **SSN** (System Service Number) — un índice único que identifica qué servicio del kernel se está solicitando. `NtCreateFile` tiene su SSN, `NtReadVirtualMemory` tiene el suyo, y así para cada operación. El stub coloca ese número en el registro **EAX** del procesador (un cajón de memoria ultrarrápido dentro de la CPU), ejecuta `SYSCALL`, y el procesador sabe exactamente a qué función del kernel dirigirse.

El flujo completo, entonces, queda así:

Nuestro Bloc de notas (anillo 3)
    ↓ llama a CreateFile() en kernel32.dll
kernel32.dll
    ↓ llama a NtCreateFile() en ntdll.dll
ntdll.dll (stub de syscall)
    ↓ coloca SSN en EAX, ejecuta instrucción SYSCALL
CPU transita de ring 3 a ring 0
    ↓ KiSystemCall64 (punto de entrada del kernel)
Kernel ejecutivo (ntoskrnl.exe) 
    ↓ ejecuta el servicio real → nos crea nuestro Archivo.txt
Retorna resultado a ntdll → kernel32 → tu aplicación

Y aquí hay algo que vale la pena detenerse a pensar: no es el kernel quien viene a buscar tu solicitud. Es el mismo hilo del procesador que está ejecutando nuestro bloc de notas el que cruza la frontera, ejecuta el trabajo dentro del kernel, y regresa. Como si el mesero entrara a la cocina él mismo a buscar el plato — pero con una diferencia clave: en cuanto sale de la cocina, pierde todo acceso a ella

_Pero espera — si el hilo "regresa" al anillo 3,"¿Dónde vive el bloc de notas mientras el hilo hace ese viaje?"_

_La respuesta nos lleva a uno de los conceptos más elegantes del diseño de Windows: **la memoria virtual._**