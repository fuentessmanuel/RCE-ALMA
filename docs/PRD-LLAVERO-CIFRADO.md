# PRD — Llavero cifrado: conectar un PC de la estación con la contraseña de la unidad

**Estado:** Borrador para revisión de Diego · **Dueño:** Diego Melo (repo y decisión) · propone Manuel Fuentes · **Creado:** 16-nov-2026 · base `main` = `a855a6a`

**Alcance:** conectar o reconectar un **PC de la estación** (las dos llaves) escribiendo la contraseña de la unidad, durante una ventana abierta desde el editor de Apps Script. Incluye guardar ese llavero cifrado, republicarlo cuando se rotan las llaves y corregir el mensaje de «llave antigua» en un PC.

**NO toca:** los celulares (siguen con su enlace, que nunca lleva la llave de estación) · el login personal correo + contraseña · el cifrado de nombres y RUT que ya existe · el primer PC de la unidad (sigue con «Configurar este PC como el primero») · el límite de intentos del login (va en su propio PRD, ver P05).

---

## 1 · Resumen

**Hoy:** para conectar un PC, o reconectarlo cuando dice «llave antigua», hace falta otro PC ya conectado, copiar un enlace con las dos llaves y llevarlo a mano (correo, pendrive) hasta el PC nuevo.

**Después:** alguien con acceso al editor abre la ventana; en el PC se escribe la contraseña de la unidad —la misma que ese PC ya iba a pedir para ver nombres— y queda conectado, con nombres incluidos. Nadie transporta llaves.

## 2 · La historia

**ANTES.** Domingo 16 de noviembre. Manuel se sienta en un PC de la unidad para revisar las camas. Escribe su correo y su contraseña y Kaptus le contesta: «Este equipo tiene una llave antigua de la app… Abre de nuevo el enlace para celulares». Está en un PC, no en un celular. Su contraseña está bien; lo que está vieja es la llave del navegador, porque en algún momento se crearon llaves nuevas. Para arreglarlo tiene que encontrar otro PC que sí funcione, entrar a Estado → «Llaves de la unidad», copiar un enlace que ES la llave completa de la unidad, mandárselo por correo o pasarlo en un pendrive, pegarlo pegado al final de la dirección /exec sin que se corte, y después escribir igual la contraseña de la unidad para ver los nombres. Si ningún PC tiene las llaves vigentes, no hay arreglo sin Diego y sin reconectar a todos. Ese turno no revisó las camas en ese PC.

**DESPUÉS.** Mismo PC, mismo aviso. Manuel le escribe a Diego —o a quien tenga el editor—: «abre la ventana de conexión». Diego ejecuta `abrirConexionPc` desde el celular o su casa. En el PC, Manuel pulsa «Conectar este PC a la unidad», escribe la contraseña de la unidad y en unos segundos el PC queda con las dos llaves y con los nombres visibles. No pasó ningún enlace por correo ni por pendrive. Cuando Diego rota las llaves porque un enlace circuló de más, los demás PC se reconectan igual: ventana + contraseña.

## 3 · Objetivos / No-objetivos

- **O1** Conectar un PC de la estación sin un segundo PC y sin transportar enlaces: basta la ventana abierta y la contraseña de la unidad.
- **O2** Que Google siga sin poder leer las llaves: en el servidor queda solo un bloque cifrado con la contraseña de la unidad, igual que nombres y RUT.
- **O3** Que rotar las llaves no obligue a repartir enlaces a todos los PC: el llavero se vuelve a publicar solo al rotar.
- **O4** Que conectar y poner la contraseña de la unidad sean un solo paso en un PC nuevo.
- **O5** Que el aviso de «llave antigua» en un PC diga qué hacer en un PC (hoy manda al enlace de celulares).

- **NO1** No cambia cómo se conectan los celulares. El llavero jamás se entrega a un celular ni a la PWA de GitHub Pages: solo dentro de `/exec#estacion`.
- **NO2** No reemplaza al enlace «Copiar enlace para otro PC»: sigue existiendo como plan B.
- **NO3** No cambia el cifrado de nombres y RUT ni la contraseña de la unidad.
- **NO4** No abre una vía permanente: sin ventana abierta desde el editor, el llavero no se entrega.
- **NO5** No toca el login personal ni su límite de intentos.

## 4 · Cómo funciona hoy → cómo va a funcionar

```
HOY                                              DESPUÉS
───                                              ───────
PC nuevo / «llave antigua»                       PC nuevo / «llave antigua»
   │                                                │
   ▼                                                ▼
buscar un PC que funcione                        quien tiene el editor ejecuta
   │                                             abrirConexionPc  (ventana 15 min)
   ▼                                                │
Estado → Llaves de la unidad →                      ▼
«Copiar enlace para otro PC»                     en el PC: /exec#estacion →
   │                                             «Conectar este PC a la unidad»
   ▼                                                │
llevarlo por correo / pendrive                      ▼
   │                                             escribe la contraseña de la unidad
   ▼                                                │
pegarlo al final de /exec                           ▼
   │                                             el PC baja el llavero cifrado,
   ▼                                             lo abre con la contraseña y guarda
Estado → «Contraseña de la unidad»               las dos llaves + la contraseña
   │                                                │
   ▼                                                ▼
conectado con nombres                            conectado con nombres
```

**Piezas que ya existen y se reutilizan:**

| Pieza | Dónde vive | Para qué sirve aquí |
|---|---|---|
| Cifrado AES-GCM + PBKDF2 (600.000 iteraciones) con la contraseña de la unidad | `src/core/cifrado.ts` | Cerrar y abrir el llavero |
| Ventana de 15 minutos desde el editor (`REGISTRO_LLAVES_HASTA`) | `Codigo.gs` `abrirRegistroLlaves` | Molde para la ventana de conexión |
| Llamada al servidor sin llave, por `google.script.run` | `servicios.ts` `configurarPrimerPc` | Molde para pedir el llavero desde un PC sin llaves |
| Guardado de llaves en el navegador (meta `conexion` y `tokenEstacion`) | `servicios.ts` | Donde quedan las llaves al abrir el llavero |
| Instalación de la contraseña (`instalarContrasena`, fila `_verificacion`) | `claveUnidad.ts`, `referencias.ts` | Se ejecuta justo después de conectar (O4) |
| Registro de auditoría | `Codigo.gs` `registrarAuditoria` | Dejar rastro de cada entrega del llavero |
| Creación y rotación de llaves | `PanelLlave.tsx`, `llaves.ts` | Momento en que se publica o republica el llavero |

## 5 · Los datos

**Disparadores**
- *Publicar el llavero:* un PC de la estación que tiene las dos llaves y la contraseña de la unidad instalada. Ocurre al crear las llaves en el primer PC, al pulsar «Crear llaves nuevas…», y a mano desde Estado («Publicar el llavero») para unidades que ya funcionan hoy.
- *Entregar el llavero:* un PC sin llaves (o con llaves viejas) que lo pide mientras la ventana de conexión está abierta.

**En el servidor (Apps Script)**
- **Propiedad del script `LLAVERO`:** un solo bloque cifrado (formato `v1.<iv>.<datos>` de `cifrado.ts`, ~400 bytes) que contiene la llave de la app y la de estación, más la fecha de publicación. ← Va en propiedades del script y **no** en la hoja `referencia`: ahí una fila nueva contaría como paciente (`soloEpisodios` solo excluye `_verificacion`) y la borraría `limpiarTodo_`.
- **Propiedad `CONEXION_PC_HASTA`:** fecha y hora de cierre de la ventana de conexión. ← Separada de `REGISTRO_LLAVES_HASTA` a propósito: la ventana de registro se cierra al primer uso y **rota** las llaves; la de conexión solo **entrega** el llavero y no rota nada.
- **Hoja `auditoria`:** una fila por cada entrega del llavero (`llavero_entregado`) y por cada publicación (`llavero_publicado`), sin datos personales.
- **Acciones nuevas:** una para publicar el llavero (exige llave de estación) y una para pedirlo (sin llave, solo con ventana abierta). Es la segunda acción sin llave del servidor, después de `registrarLlaves`.

**En el PC**
- Las llaves quedan donde ya quedan hoy (meta `conexion` y `tokenEstacion`); la contraseña de la unidad, donde ya queda (`claveUnidad`).
- Nada nuevo se guarda en el navegador.

**Candados**
- Sin ventana abierta, el servidor no entrega el llavero (NO4).
- La ventana se cierra sola a los 15 minutos (P02: si además se cierra a las N entregas).
- Si las llaves cambian y nadie republica, el llavero quedaría con llaves viejas: por eso «Crear llaves nuevas…» lo republica en el mismo paso, y la entrega comprueba que la llave descifrada sea la vigente antes de dar el PC por conectado.

## 6 · Reglas, promesas y pendientes — el acuerdo

**Reglas**
- **R1** Solo un PC de la estación con las dos llaves y la contraseña de la unidad instalada puede publicar el llavero.
- **R2** El llavero se entrega solo mientras la ventana de conexión esté abierta, y la ventana solo se abre desde el editor de Apps Script, por quien tenga acceso a él.
- **R3** El llavero solo se ofrece en la vista de estación servida desde `/exec`. La PWA de celulares no muestra el botón ni conoce la acción.
- **R4** El PC abre el llavero en el navegador con la contraseña escrita; si la contraseña no sirve, no guarda nada y lo dice («Contraseña incorrecta o llavero de otra unidad»).
- **R5** Al abrir el llavero, antes de darse por conectado, el PC comprueba con el servidor que las dos llaves sean las vigentes. Si no lo son, avisa «El llavero tiene llaves viejas: pide que se publique de nuevo desde un PC conectado».
- **R6** En el mismo paso queda instalada la contraseña de la unidad (O4); no se pide dos veces.
- **R7** Rotar las llaves con «Crear llaves nuevas…» republica el llavero en ese mismo paso, cifrado con la contraseña ya instalada en ese PC.
- **R8** En un PC, el aviso de «llave antigua» dice: «Pide que abran la ventana de conexión y pulsa “Conectar este PC a la unidad”, o abre el enlace para otro PC que muestra un PC conectado». En un celular sigue diciendo lo de hoy.
- **R9** Cada entrega y cada publicación queda en la auditoría.

**Promesas**
- Google nunca recibe las llaves ni la contraseña en claro: solo el bloque cifrado y, como hoy, sus huellas.
- Un celular nunca recibe la llave de estación.
- Sin ventana abierta, pedir el llavero no devuelve nada útil, ni siquiera si existe.
- El enlace «Copiar enlace para otro PC» sigue funcionando igual.

**El riesgo que este diseño agrega, dicho de frente**
Hoy el bloque `_verificacion` solo sirve para comprobar la contraseña y lo lee quien ya tiene la llave de estación. El llavero, en cambio, **contiene las llaves**, y durante la ventana lo puede bajar cualquiera que tenga la URL `/exec` (que no es secreta). Con ese bloque, alguien podría intentar adivinar la contraseña de la unidad en su propio computador, sin límite de intentos. Lo frenan tres cosas: la ventana corta, las 600.000 iteraciones de PBKDF2 y el largo de la contraseña. **La tercera es la que decide**: con una contraseña de 8 caracteres comunes el freno es débil; con una frase larga es fuerte. Por eso P01.

**Pendientes (decide Diego)**
- **P01** Largo mínimo de la contraseña de la unidad para habilitar el llavero (hoy el mínimo general es 8). Propuesta: no publicar el llavero si la contraseña tiene menos de 14 caracteres, y avisar en Estado. *Mientras no se decida: el llavero no se publica.*
- **P02** ¿La ventana se cierra también tras N entregas (propuesta: 3), o solo por tiempo? *Mientras tanto: solo por tiempo.*
- **P03** ¿Sal propia y aleatoria para el llavero, en vez de la sal fija `upc-pulse/identidad/v1`? Con sal fija, un mismo trabajo de adivinanza sirve para el llavero y para cualquier bloque de identidad. Propuesta: sal aleatoria guardada junto al llavero. *Mientras tanto: sal propia (no cambia nada de lo ya cifrado).*
- **P04** ¿Quién abre la ventana? Hoy solo el dueño del editor. Si se quiere que coordinación lo haga sin editor, es otro diseño. *Mientras tanto: solo desde el editor.*
- **P05** El límite de intentos del login personal y el largo del PIN quedan para su propio PRD (hallazgo de la auditoría del 16-nov).
- **P06** Las unidades que ya funcionan publican su primer llavero a mano desde Estado. ¿Se agrega un aviso en Estado mientras no haya llavero publicado? *Propuesta: sí.*

**Cómo se prueba (para el PR que lo implemente)**
- Tests unitarios: cerrar y abrir el llavero; contraseña equivocada; llavero con llaves viejas; republicación al rotar.
- `npm run e2e:gas`: ampliar el escenario del segundo PC (hoy entra por `#estacion?a=..&t=..`) con uno que entre por ventana + contraseña, y otro que lo intente con la ventana cerrada.
- Despliegue: hay que pegar el `Codigo.gs` nuevo y el `index.html` cohete, y publicar versión nueva sobre la implementación existente.
