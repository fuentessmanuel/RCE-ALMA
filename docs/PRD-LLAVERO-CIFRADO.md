# PRD — Llavero cifrado: conectar un PC de la estación con la contraseña de la unidad

**Estado:** Borrador v2 (versión mínima) para revisión de Diego · **Dueño:** Diego Melo (repo y decisión) · propone Manuel Fuentes · **Creado:** 16-nov-2026 · **Revisado:** 17-nov-2026 · base `main` = `a855a6a`

**Alcance:** un PC de la estación sin llaves, o con llaves viejas, se conecta escribiendo **solo la contraseña de la unidad**. El servidor la comprueba, con límite de intentos, y recién entonces entrega las llaves cifradas.

**NO toca:** los celulares (siguen con su enlace, que nunca lleva la llave de estación) · el login personal (correo + contraseña) · el cifrado de nombres y RUT · el primer PC de una unidad nueva (sigue con «Configurar este PC como el primero») · el enlace «Copiar enlace para otro PC» (queda como plan B).

> **Qué cambió respecto del borrador v1 (16-nov):** se eliminó la ventana de 15 minutos que abría alguien desde el editor. En la v1, durante esa ventana, cualquiera con la URL `/exec` podía llevarse el llavero y adivinar la contraseña en su propio computador sin límite. En la v2 el llavero **nunca** sale sin que el servidor haya comprobado la contraseña: es más seguro y no depende de ninguna persona.

---

## 1 · Resumen

**Hoy:** para conectar o reconectar un PC hace falta otro PC conectado, copiar un enlace que es la llave completa de la unidad y llevarlo a mano (correo, pendrive).

**Después:** en el PC se escribe la contraseña de la unidad y queda conectado, con nombres incluidos. Nadie más tiene que hacer nada.

## 2 · La historia

**ANTES.** Domingo 16 de noviembre. Manuel se sienta en un PC de la unidad. Escribe su correo y su contraseña y Kaptus le contesta: «Este equipo tiene una llave antigua de la app… Abre de nuevo el enlace para celulares». Está en un PC, no en un celular, y su contraseña está bien: lo viejo es la llave guardada en ese navegador, porque en algún momento se crearon llaves nuevas. Para arreglarlo tiene que encontrar otro PC que sí funcione, sacar de ahí un enlace que ES la llave de la unidad, mandárselo por correo o pasarlo en un pendrive, pegarlo al final de la dirección `/exec` sin que se corte y, después, escribir igual la contraseña de la unidad para ver los nombres. Ese turno no revisó las camas en ese PC.

**DESPUÉS.** Mismo PC, mismo aviso. Debajo aparece «Conectar este PC con la contraseña de la unidad». Manuel la escribe —la misma que ese PC le iba a pedir de todos modos para ver nombres— y en unos segundos el PC queda conectado y con los nombres visibles. No tuvo que llamar a nadie, ni buscar otro PC, ni mover un enlace.

## 3 · Objetivos / No-objetivos

- **O1** Conectar un PC de la estación con un solo dato, la contraseña de la unidad, sin ninguna otra persona ni otro PC.
- **O2** Que nadie pueda llevarse las llaves sin saber la contraseña, y que adivinarla esté limitado a unos pocos intentos por hora.
- **O3** Que Google siga sin poder leer las llaves ni la contraseña.
- **O4** Que conectar e instalar la contraseña de la unidad sean un solo paso.
- **O5** Que, después de crear llaves nuevas, los demás PC se reconecten igual: con la contraseña.
- **O6** Que en un PC el aviso de «llave antigua» diga qué hacer en un PC.

- **NO1** No cambia cómo se conectan los celulares. La opción no existe en la PWA de celulares.
- **NO2** No se agrega ninguna ventana, botón de coordinación ni paso en el editor.
- **NO3** No cambia el cifrado de nombres y RUT ni la contraseña de la unidad.
- **NO4** No incluye avisos ni alertas a otros PC (queda como P03).
- **NO5** No toca el login personal (su límite de intentos va aparte, ver P04).

## 4 · Cómo funciona hoy → cómo va a funcionar

```
HOY                                         DESPUÉS
───                                         ───────
PC con «llave antigua»                      PC con «llave antigua»
   │                                           │
   ▼                                           ▼
buscar otro PC que funcione                 «Conectar este PC con la contraseña
   │                                         de la unidad» → la escribe
   ▼                                           │
Estado → Llaves de la unidad →                 ▼
«Copiar enlace para otro PC»                el servidor comprueba la contraseña
   │                                         (con límite de intentos)
   ▼                                           │
llevarlo por correo / pendrive                 ├─ no coincide → no entrega nada,
   │                                           │   cuenta el intento
   ▼                                           ▼
pegarlo al final de /exec                   entrega el llavero cifrado
   │                                           │
   ▼                                           ▼
Estado → «Contraseña de la unidad»          el PC lo abre con la contraseña y
   │                                         guarda las llaves y la contraseña
   ▼                                           │
conectado con nombres                          ▼
                                            conectado con nombres
```

**Lo que se publica solo, sin que nadie lo haga a mano:** cada vez que un PC conectado instala la contraseña de la unidad o crea llaves nuevas, sube el llavero y la huella de comprobación. En una unidad que ya funciona, basta con que un PC conectado abra Estado una vez después de la actualización (ver P02).

**Piezas que ya existen y se reutilizan:**

| Pieza | Dónde vive | Para qué sirve aquí |
|---|---|---|
| Cifrado AES-GCM + PBKDF2 (600.000 iteraciones) | `src/core/cifrado.ts` | Cerrar y abrir el llavero; derivar la huella de comprobación |
| Llamada al servidor sin llave (`google.script.run`) | `servicios.ts` `configurarPrimerPc` | Molde para pedir el llavero desde un PC sin llaves |
| Dónde quedan las llaves en el PC (meta `conexion` y `tokenEstacion`) | `servicios.ts` | Mismo lugar, sin cambios |
| Instalación de la contraseña (`instalarContrasena`) | `claveUnidad.ts` | Se ejecuta en el mismo paso (O4) |
| Huellas en propiedades del script, como `APP_TOKEN_HASH` | `Codigo.gs` | Mismo lugar para la huella de comprobación y el contador |
| Registro de auditoría | `Codigo.gs` `registrarAuditoria` | Una fila por entrega y por intento fallido |

## 5 · Los datos

**En el servidor (propiedades del script, no en la hoja)**
- **`LLAVERO`**: un bloque cifrado (~400 bytes) con la llave de la app y la de estación. ← No va en la hoja `referencia`: ahí contaría como paciente (`soloEpisodios` solo excluye `_verificacion`) y lo borraría `limpiarTodo_`.
- **`LLAVERO_COMPROBACION`**: la huella de la contraseña con la que el servidor compara. Se deriva de la contraseña con **otra etiqueta** que la llave del llavero, de modo que **con la huella no se puede abrir el llavero**. ← Esto es lo que impide que Google, o quien lea las propiedades, abra las llaves.
- **`LLAVERO_FALLOS`**: cuántos intentos fallidos van y desde cuándo. ← El candado contra adivinar.

**Las dos acciones nuevas del servidor**
- *Publicar el llavero*: exige la llave de estación. Guarda `LLAVERO` y `LLAVERO_COMPROBACION`.
- *Pedir el llavero*: es una acción sin llave, como `registrarLlaves`. Recibe la huella, la compara y entrega el llavero solo si coincide y el candado no está cerrado.

**En el PC:** nada nuevo. Las llaves y la contraseña quedan donde ya quedan.

**Candados**
- Sin la huella correcta, el servidor no entrega el llavero: no hay forma de bajarlo para adivinar en otro computador.
- **5 intentos fallidos en una hora cierran la entrega por una hora**, y se reabre sola.
- El llavero se reemplaza entero en cada publicación: si se crean llaves nuevas, el llavero viejo deja de existir en el mismo paso.

## 6 · Reglas, promesas y pendientes — el acuerdo

**Reglas**
- **R1** Solo un PC con la llave de estación y la contraseña de la unidad instalada puede publicar el llavero.
- **R2** El llavero se publica solo al instalar la contraseña de la unidad y al crear llaves nuevas; nadie tiene que acordarse.
- **R3** El servidor entrega el llavero únicamente si la huella coincide. Si no coincide, responde lo mismo que si no existiera llavero y cuenta el intento.
- **R4** Con 5 intentos fallidos en una hora, la entrega se cierra una hora. El PC lo dice: «Demasiados intentos. Vuelve a probar en una hora, o usa el enlace para otro PC».
- **R5** Al abrir el llavero, el PC comprueba que las llaves sean las vigentes antes de darse por conectado, y en el mismo paso instala la contraseña de la unidad.
- **R6** La opción solo aparece en la vista de estación servida desde `/exec`, nunca en la PWA de celulares.
- **R7** En un PC, el aviso de «llave antigua» dice: «Conecta este PC con la contraseña de la unidad, o abre el enlace para otro PC». En un celular dice lo de hoy.
- **R8** Cada entrega y cada intento fallido quedan en la auditoría.

**Promesas**
- Google nunca tiene las llaves ni la contraseña en claro, y la huella que guarda no sirve para abrir el llavero.
- Nadie puede bajar el llavero sin saber la contraseña.
- Un celular nunca recibe la llave de estación.
- El enlace «Copiar enlace para otro PC» sigue funcionando igual.

**Lo que hay que saber, dicho de frente**
- Apps Script no sabe desde qué computador viene cada intento, así que el límite es **para toda la unidad**. Alguien con la URL `/exec` podría gastar los 5 intentos a propósito y dejar una hora sin poder conectar PC nuevos. No roba nada, y los PC ya conectados siguen funcionando.
- Si alguien entra a la cuenta de Google dueña del proyecto, esto no lo protege; pero en ese caso ya tiene la planilla entera, así que no es un riesgo nuevo.

**Pendientes (decide Diego)**
- **P01** ¿5 intentos por hora le parece bien a la unidad? *Mientras tanto: 5.*
- **P02** Unidades que ya funcionan: el primer llavero se publica cuando un PC conectado vuelve a instalar la contraseña. ¿Se hace solo al abrir Estado, si ese PC ya tiene la contraseña instalada? *Propuesta: sí, sin preguntar.*
- **P03** ¿Avisar en los PC conectados cuando se conecta uno nuevo («se conectó un PC hoy a las 14:32»)? *Mientras tanto: solo queda en la auditoría.*
- **P04** El mismo candado de intentos podría proteger el login personal, hallazgo de la auditoría del 16-nov. Va en su propio PRD, pero conviene programarlos juntos.

**Cómo se prueba (para el PR que lo implemente)**
- Tests unitarios: publicar y pedir el llavero; contraseña equivocada no entrega nada; la huella no abre el llavero; el candado cierra a los 5 fallos y se reabre a la hora; crear llaves nuevas republica el llavero.
- `npm run e2e:gas`: un segundo PC que se conecta solo con la contraseña, y otro que lo intenta con la contraseña equivocada.
- Despliegue: pegar el `Codigo.gs` nuevo y el `index.html` cohete, y publicar versión nueva sobre la implementación existente.
