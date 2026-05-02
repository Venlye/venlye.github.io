---
title: "2. Memoria Virtual"
date: 2026-05-02 12:00:00 -0500
categories: ["Fundamentos de Ejecución en Windows: PEB, Gestión de Memoria y Técnicas de Hooking."]
tags: [windows, memoria virtual, eprocess, peb]
---

_El proceso del bloc de notas nunca se movió. Siguió viviendo exactamente donde siempre estuvo — en su espacio propio de memoria, esperando que el hilo regresara con el resultado. Y aquí surge la siguiente pregunta natural: **¿qué es ese "espacio propio"?** ¿Dónde vive exactamente un proceso mientras existe?_

Pues esa ilusión de propiedad absoluta se logra mediante la memoria virtual. Cuando un proceso inicia, el sistema operativo no le da acceso a la memoria física real, o sea, a la RAM. En su lugar, le entrega un mapa en blanco enorme.

¿Por qué un mapa y no memoria real? Porque la memoria física es limitada y compartida entre todos los procesos corriendo simultáneamente. Tu bloc de notas, tu navegador, tu videojuego — todos compiten por los mismos gigabytes de RAM. Si cada proceso viera la memoria real directamente, un programa mal escrito podría leer o sobreescribir la memoria de otro.

La memoria virtual es una técnica de gestión del sistema operativo que utiliza espacio en el disco duro o SSD para simular RAM adicional cuando la memoria física (RAM) está llena. Sirve para ==**evitar cierres de programas, mantener la estabilidad del sistema y ejecutar aplicaciones exigentes** al mover datos inactivos al disco==

Imagina que la RAM es un estacionamiento con 100 espacios numerados del 1 al 100. Cada proceso que corre en tu PC ocupa algunos de esos espacios.

Sin memoria virtual, el estacionamiento es **público y visible para todos**. El bloc de notas sabe que ocupa los espacios del 1 al 20. El navegador sabe que ocupa del 21 al 60. Y como todos ven el mismo estacionamiento, un programa podría simplemente ir a mirar el espacio 21 — que es del navegador — y leer lo que hay ahí. Tal vez hay una contraseña guardada. No tuvo que romper nada, solo fue a mirar un espacio que no era suyo.

Con memoria virtual, cada proceso recibe su **propio estacionamiento privado**, numerado también del 1 al 100. El bloc de notas cree que sus espacios son del 1 al 100. El navegador también cree que sus espacios son del 1 al 100. Pero son estacionamientos distintos — el kernel es el único que sabe dónde está el estacionamiento real de cada uno.

Entonces cuando el bloc de notas intenta ir al espacio 50 del navegador, simplemente no puede — no tiene el mapa de ese estacionamiento. Solo conoce el suyo.


![[Pasted image 20260502054116.png]]

**Columna 1 — Procesos a cargar**

Un solo bloque rosado con varios datos dentro (A, B y otros) esperando ser cargados en memoria. Los puntos (•) indican que ese proceso tiene más de un segmento — sus datos no son un bloque sólido, sino partes separadas.

---

**Columna 2 — Memoria Virtual Segmentada**

El sistema le asigna al proceso su espacio virtual privado — su estacionamiento propio. Aquí aparecen los colores: rosado, amarillo, morado, cyan y gris. Cada color representa un segmento distinto del proceso. Los guiones son espacios vacíos reservados pero no usados todavía.

---

**Columna 3 — Vista de cada segmento compuesto por páginas**

Cada segmento de color se divide en pedazos más pequeños llamados **páginas**. El gráfico te muestra cada segmento por separado para que veas que internamente también tienen huecos. Esto permite que el sistema mueva a la RAM solo las páginas que el proceso necesita en este momento, no todo el segmento completo.

---

**La Tabla de páginas**

El kernel es el único que sabe a qué dirección física corresponde cada página virtual. Ningún proceso puede consultar esta tabla directamente — solo el kernel tiene ese mapa.

---
**Columna 4 — Memoria Física**

La RAM real donde todo convive. Los datos de todos los segmentos (A, D, F, G, B, Q, M) están ahí juntos, pero cada proceso solo ve su propio mapa virtual y nunca sabe que los demás existen al lado.
___

La pregunta que surge es: ¿quién lleva el registro de todo eso?

Si cada proceso tiene su espacio de direcciones virtual privado, el sistema operativo necesita mantener una estructura de control por cada proceso que existe — saber qué memoria tiene asignada, qué hilos están corriendo dentro de él, qué DLLs cargó, qué permisos tiene, cuál es su identificador único en el sistema. Sin ese registro, el kernel sería ciego.

Windows mantiene ese control en dos estructuras de datos por cada proceso:

**EPROCESS** (Executive Process) — es una ==estructura de datos opaca fundamental en el núcleo de Windows, que actúa como el objeto de proceso que representa a cada proceso en ejecución== en su propio espacio de memoria protegido, el anillo 0. Contiene todo lo que el sistema necesita saber sobre el proceso: su PID (Process ID, el número único que lo identifica), sus handles abiertos, su token de seguridad (que define qué puede y qué no puede hacer), y un puntero hacia la segunda estructura. Al vivir en modo kernel, el proceso en sí nunca puede leerla ni tocarla directamente.

**PEB** (Process Environment Block) — es una ==estructura de datos interna de Windows en modo usuario.== Contiene información que el proceso necesita consultar constantemente: qué DLLs tiene cargadas en memoria a través de su campo `Ldr`, cuál es su directorio de trabajo, y los argumentos con los que fue ejecutado. Al vivir en el espacio del propio proceso, este puede leerla y modificarla libremente — sin pedirle permiso a nadie.