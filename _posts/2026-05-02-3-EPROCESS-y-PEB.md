---
title: "3. EPROCESS y PEB"
date: 2026-05-02 12:00:00 -0500
categories: ["Fundamentos de Ejecución en Windows: PEB, Gestión de Memoria y Técnicas de Hooking."]
tags: [windows, eprocess, peb, unlinking]
---

### Conceptos Previos:

Un **puntero** es una variable que no contiene un dato directamente — contiene una **dirección de memoria**. En lugar de guardar el valor, guarda la ubicación exacta en memoria donde ese valor vive. Es como si en vez de escribir tu número de teléfono en un papel, escribieras la dirección de la agenda donde está guardado. El EPROCESS no copia toda la información de cada estructura dentro de sí mismo — almacena la dirección de memoria donde esa estructura vive, y accede a ella cuando la necesita.

Un **handle** es un identificador numérico que el kernel le entrega a un proceso cuando este solicita acceso a un recurso del sistema — un archivo, un hilo, una clave de registro, un socket de red. El proceso no toca el recurso directamente — le presenta su handle al kernel, y el kernel verifica si ese handle tiene los permisos necesarios para la operación solicitada. El handle por sí solo no es el recurso, es el ticket de acceso a él. Todos los handles que un proceso tiene abiertos en un momento dado se registran en su **handle table** — una estructura que el kernel mantiene y que el proceso no puede manipular directamente.

**EPROCESS** (Executive Process) — es una ==estructura de datos opaca fundamental en el núcleo de Windows, que actúa como el objeto de proceso que representa a cada proceso en ejecución== en su propio espacio de memoria protegido, el anillo 0. Contiene todo lo que el sistema necesita saber sobre el proceso: su PID (Process ID, el número único que lo identifica), sus handles abiertos, su token de seguridad (que define qué puede y qué no puede hacer), y un puntero hacia la segunda estructura. Al vivir en modo kernel, el proceso en sí nunca puede leerla ni tocarla directamente.

¿Qué contiene? Todo lo que define al proceso legalmente ante los ojos del sistema operativo. Su **PID** (Process Identifier) — el número único que lo identifica dentro del sistema. Un puntero a su **handle table** — la tabla donde se registran todas las referencias a recursos del sistema que el proceso tiene abiertos: archivos, hilos, claves de registro. Un puntero a su **EPROCESS_QUOTA_BLOCK** — la estructura que define los límites de memoria y recursos que puede consumir, compartida entre procesos del mismo usuario. Y lo más crítico: su **token de acceso**.

El token de acceso es la credencial de seguridad del proceso — el carnet de identidad que Windows consulta cada vez que el proceso intenta hacer algo sensible. ¿Puede abrir este archivo? ¿Puede modificar este registro? ¿Tiene privilegios de administrador? Todo eso está codificado en el token. Sin un token válido, el proceso no es nadie ante el sistema.

Al vivir en modo kernel, el EPROCESS es completamente intocable desde modo usuario. Ningún programa puede leerlo directamente, y mucho menos modificarlo.

Sin embargo, el sistema no puede mantener absolutamente todo en ese espacio protegido. El proceso mismo necesita conocer ciertas cosas sobre su propio entorno para funcionar — qué DLLs tiene cargadas en memoria, con qué argumentos fue ejecutado, cuál es su directorio de trabajo. Consultar esos datos a través de una syscall cada vez que los necesita sería lento e ineficiente.

Entonces el sistema hace algo deliberado: crea una segunda estructura, el **PEB** (Process Environment Block), y la coloca directamente dentro del espacio de direcciones virtual del propio proceso — en modo usuario, accesible sin necesidad de cruzar al kernel.

El PEB contiene exactamente esa información práctica del día a día: los parámetros de línea de comando, la imagen base del ejecutable, configuraciones del heap, y lo más relevante para nosotros — su campo `Ldr`.

`Ldr` es un puntero, una dirección de memoria que apunta a una estructura llamada **PEB_LDR_DATA**. Y esa estructura contiene tres listas enlazadas — cadenas de nodos donde cada nodo apunta al siguiente y al anterior, como una cadena de eslabones — que registran todas las DLLs que el proceso tiene cargadas en memoria:

- **InLoadOrderModuleList** — ordenadas por el orden en que fueron cargadas
- **InMemoryOrderModuleList** — ordenadas por su posición en memoria
- **InInitializationOrderModuleList** — ordenadas por el orden en que se inicializaron

Cada eslabón de esas cadenas es una estructura llamada **LDR_DATA_TABLE_ENTRY** — la ficha individual de cada DLL: su nombre, su dirección base en memoria, su tamaño.

PEB
└─ Ldr → PEB_LDR_DATA
└─ InLoadOrderModuleList
├─ [ntdll.dll] ←→ [kernel32.dll] ←→ [tu_dll.dll] ←→ ...
└─ cada nodo es un LDR_DATA_TABLE_ENTRY

Esa lista es el directorio oficial. Es lo que el sistema considera verdad. Cuando Windows Defender quiere saber qué módulos tiene cargados un proceso, no escanea toda la memoria en busca de código extraño — eso sería prohibitivamente lento. Consulta esta lista.

Y aquí es donde la arquitectura empieza a mostrar sus primeras grietas.

Esa verdad vive en modo usuario. En el espacio del proceso. Donde el proceso puede escribir libremente, sin cruzar al kernel, sin emitir ninguna syscall, sin que nadie le pregunte nada.

Eso significa que un proceso puede modificar su propio `Ldr`. Puede tomar el eslabón que lo registra — su **LDR_DATA_TABLE_ENTRY** — y reconectar los nodos que lo rodean entre sí, saltándoselo. El módulo sigue ahí, sigue ocupando memoria, sigue ejecutando código — pero desapareció del directorio oficial. El sistema ya no lo ve.

No hubo alarma. No hubo syscall. No hubo cruce al kernel. Solo una escritura en memoria propia, completamente legal desde la perspectiva del sistema.

Esto se conoce como **unlinking** — borrar un módulo de la lista enlazada del PEB para hacerlo invisible ante cualquier herramienta que consulte ese directorio.

_El unlinking engaña a herramientas que solo miran el PEB. No engaña a un **EDR** (Endpoint Detection and Response) — un sistema de seguridad que, a diferencia de un antivirus tradicional, no confía ciegamente en el directorio oficial y además escanea la memoria física directamente._

