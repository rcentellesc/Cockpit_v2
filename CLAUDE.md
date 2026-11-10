# Tareas — app personal de escritorio

## Comunicación
Respuestas mínimas. Sin explicar lo que vas a hacer ni resumir lo hecho. Si hay duda real, una pregunta corta; si no, decide y sigue.

## Qué es
Gestor personal de tareas. Un solo `index.html` (HTML + CSS + JS vanilla, sin build, sin dependencias externas, sin frameworks). Se abre en Edge/Chrome desde el disco. Sustituye las banderas y categorías de color de Outlook. No es Planner: nada de recordatorios, notificaciones, subtareas, adjuntos ni colaboración.

Principio: capturar y mover tareas tiene que ser "pim pam". Una línea de texto para crear, un clic o una tecla para mover.

Responde a una sola pregunta: **qué muevo yo y a quién tengo que reclamar**. Los hitos, riesgos y bloqueos de los proyectos viven en la herramienta corporativa de seguimiento de proyectos; el calendario, en Outlook. Si uno de ellos exige algo del usuario, entra como tarea con fecha. Lo que no sea una acción suya o un reclamo no entra.

Dos vistas (pestañas en cabecera, teclas `1` y `2`) sobre **dos listas separadas**: **Día** (tareas: Hoy + columnas por fecha, la pantalla principal) y **Radar** (asuntos: lo que el usuario vigila a su nivel en cada proyecto, sin tareas colgando). Nada enlaza una lista con la otra. Un asunto puede tener tareas en Día que se hagan sin que el asunto se resuelva, no tener ninguna, o resolverse solo.

## Modelo de datos

```json
{
  "v": 1,
  "settings": { "captureIn": "week", "waitDays": 5, "warnToday": 5, "theme": "light" },
  "programs":  [{ "id": "p1", "name": "Programa A", "color": "#3d7ebf" }, { "id": "p2", "name": "Programa B", "color": "#2e9e7a" }],
  "projects":  [{ "id": "pr1", "programId": "p1", "name": "Proyecto A1", "fav": true }],
  "people":    [{ "id": "me", "name": "Yo", "color": "black" }, { "id": "u1", "name": "Persona 1", "color": "blue" }],
  "radar":     [{ "id": "r1", "programId": "p1", "projectId": "pr1", "text": "Bloqueo en el conector externo", "note": "", "order": 0, "foco": true, "parked": false, "createdAt": "2026-10-28", "doneAt": null }],
  "tasks": [{
    "id": "t1", "title": "Pedir presupuesto",
    "personId": "u1", "projectId": "pr1",
    "type": "hacer | urgente",
    "date": "2026-10-08",
    "waiting": false, "remindOn": null,
    "note": "", "order": 0,
    "done": false, "doneAt": null, "createdAt": "2026-10-05T10:00:00Z"
  }]
}
```

- `date` es la fecha en la que quiero ver la tarea, no un vencimiento. Siempre tiene valor.
- `waiting: true` = la tarea está en Esperando; `remindOn` es la fecha de reclamo.
- Persona `me` existe siempre, no se puede borrar, color negro.
- Cada proyecto pertenece a un programa. Las tareas solo llevan proyecto (opcional).
- **Asunto** (lista `radar`) = un tema que el usuario vigila a su nivel: "bloqueo en el conector", "fechas que se ha dado validación". No son los riesgos del proyecto (esos están en la herramienta corporativa), son los que él decide tener en el radar esta temporada. Cuelga de un proyecto (`projectId`, y `programId` el del proyecto), de un programa (`programId` solo) o de nada (global). Lleva `text`, `note` libre, `order` (manual dentro de su bloque), `foco` (★, lo sube y lo pone en negrita), `parked` (aparcado, abajo en gris), `createdAt` (fecha en que entró, se muestra) y `doneAt` (resuelto; sale de la fila y queda plegado en "resueltos"). **Nada más**: sin estado, sin semáforo, sin fecha límite, sin dueño, sin enlace con tareas. Se crea solo desde la vista Radar ("+ asunto" en cada bloque). Un asunto resuelto se puede reabrir desde "resueltos"; los resueltos de más de 90 días se purgan junto con las hechas.

## Columnas (calculadas, nunca almacenadas)
Las columnas se derivan de `date` / `waiting` cada vez que se renderiza, con la fecha de hoy. Una tarea para el jueves está en "Esta semana" el lunes y en "Hoy" el jueves, sin intervención.

| Columna | Regla |
|---|---|
| Hoy | `date <= hoy` y no en espera, **o** en espera con `remindOn <= hoy` |
| Mañana | `date == hoy+1` |
| Esta semana | `date` entre hoy+2 y el domingo de esta semana |
| Próxima semana | `date` en la semana siguiente (lunes a domingo) |
| Más adelante | `date` posterior |
| Esperando | en espera y `remindOn > hoy` |

**En espera** = `waiting: true` **o** tarea de otra persona (chip ≠ Yo) no urgente. Las tareas de otros nunca están en las columnas por fecha: esperan hasta su fecha de reclamo (`remindOn`, y si falta, `date`) y ese día aparecen en Hoy como reclamo. Tocar la fecha de una tarea en espera cambia su `remindOn`. Mover una tarea de otro a Mañana/Esta semana/etc. fija `remindOn` a esa fecha (sigue en Esperando). Asignar una tarea mía a otra persona la pone en espera (`remindOn = date` si es futura, si no `hoy + waitDays`); devolverla a Yo la saca. Urgente de otro: va a Hoy, sin espera. Esperando = lo que depende de otros (o aparcado a mano); Más adelante = lo mío a largo plazo.

Semana empieza en lunes. Hoy no tiene límite; si hay más de `warnToday` tareas, el contador se pone en rojo con texto "demasiadas, reparte". Las de Esperando que llegan a Hoy se muestran como reclamo (fondo ámbar suave) y con texto "reclamar · desde dd/mm".

## Layout
Pantalla de escritorio, 1280+ px. Tipografía system-ui (sin fuentes externas). Colores por tokens, ver **Temas**. Referencia visual: `maqueta-radar.html` (datos de ejemplo, pestañas Día/Radar, conmutador de tema en la franja superior).

1. **Cabecera**: título, fecha de hoy, conmutador de vista **Día | Radar**. A la derecha: filtros (ver abajo), botón Hechas (N), botón Ajustes.
2. **Línea rápida**: input grande (46 px, 16 px de texto). Debajo, una línea de ayuda con la sintaxis y los atajos. Se oculta en la vista Radar.
3. **Hoy**: panel (`--panel`) a la izquierda, ~35 % del ancho, con dos carriles (swimlanes): "Mías" arriba y "De otros" abajo (fondo `--wait-bg`, borde `--wait-line`), cada uno con su cabecera y contador. El resto de columnas hacen lo mismo y muestran las cabeceras de carril solo si hay tareas de ambos tipos. Orden manual dentro de cada carril. Tarjetas grandes: título 16 px, chip de persona, chip de proyecto, fecha, y botones siempre visibles: `→ mañ`, `→ vie`, `⏸ espera`, `✓`. Orden manual por arrastre dentro de Hoy.
4. **Resto**: cinco columnas a la derecha en formato lista compacta (una fila = título 13 px; segunda línea chip persona + fecha). Sin botones visibles; al pasar el ratón aparecen: sol (a Hoy), `→ mañ`, `⏸`, `✓`. Esperando con fondo `--wait-bg`, borde `--wait-line`, cabecera ámbar, y la fecha de cada fila dice "reclamar jue 8".
5. **Pie**: leyenda de colores y estado de guardado ("Guardado en tareas.json · hh:mm" o "Solo navegador · vincular archivo en Ajustes").

### Vista Radar
Sustituye a Hoy + columnas y a la línea rápida cuando está activa; cabecera, filtros y pie se mantienen. Es **una lista, no una cuadrícula**: un panel único con una banda por programa (banda **Global** primero) y, dentro, **una fila por bloque**, siempre las mismas filas aunque estén vacías: en Global una fila "Global · sin programa"; en cada programa una fila "Programa · transversal" y una por proyecto. Las filas se derivan de `programs` y `projects`.

Encima del panel, una línea de resumen: "N asuntos en el radar · N en foco ★ · N aparcados · N resueltos esta semana".

Cada fila: a la izquierda (220 px, borde izquierdo de 4 px del color del programa) el nombre del bloque en el color del programa, debajo en gris "transversal" o "sin programa" si procede y "N asuntos · k ★". A la derecha, sus asuntos abiertos uno por línea: ★ foco primero (en negrita), luego los normales por `order`, al final los aparcados (gris, cursiva). Cada asunto: asa de arrastre (visible al pasar el ratón), botón estrella, texto, nota corta en gris tras " · " si la hay, fecha de `createdAt` como "dd/mm" a la derecha, y al pasar el ratón tres botones: `→ tarea`, `aparcar`/`reactivar`, `✓ resuelto`. Debajo de los asuntos, un input en línea "+ asunto" (Intro crea con el texto escrito, sin sintaxis, y vacía el input). Debajo, si hay, "▸ resueltos (N)" plegado: clic despliega la lista tachada con su fecha; clic en uno lo reabre. Fila sin asuntos abiertos: una sola línea con "Nada en el radar" en cursiva gris y el input "+ asunto" al lado.

`→ tarea` **crea directamente una tarea** copiada del asunto: título = texto del asunto, proyecto = el del bloque (ninguno si es transversal o global), persona Yo, fecha según `settings.captureIn` como cualquier captura. No abre la línea rápida ni cambia de vista; avisa con un mensaje breve ("Tarea creada · vie 13"). Al revés, `R` con una tarea seleccionada (o "→ Radar" en su panel) **crea un asunto** copiando el título de la tarea en el bloque de su proyecto (Global si no tiene proyecto); la tarea se queda en Día; avisa con "Asunto creado". En los dos sentidos es una copia: ningún vínculo entre asunto y tarea (ni id, ni contador, ni chip). Clic en el texto lo edita en sitio (Intro guarda, Esc cancela). Doble clic abre el panel lateral con texto, nota, a qué cuelga (proyecto / programa / global) y botón Borrar con confirmación.

Teclado con un asunto seleccionado (clic o flechas): `F` foco · `A` aparcar/reactivar · `X` resuelto · `Alt+↑/↓` reordenar dentro de su bloque · `Intro` editar · `Esc` deseleccionar. Arrastre: reordena dentro del bloque o mueve a otro bloque (cambia `projectId`/`programId`).

Sin avisos de ningún tipo. Los filtros de cabecera en Radar son: Todo, **★ Foco** (atenúa al 30 % los que no están en foco, no los oculta) y programas (un programa activo oculta las otras bandas). El buscador `/` filtra por texto y nota.

Referencia visual de Día: panel Hoy grande a la izquierda, listas compactas a la derecha. No copiar el estilo de otras apps: nada de campos "Persona / Fecha límite" vacíos en cada tarjeta.

## Colores
Dos dimensiones separadas:

**Barra lateral = programa**: cada tarjeta de Hoy lleva una barra izquierda de 4 px y cada fila de columna una de 3 px con el color del programa, de una paleta de 8 (`#3d7ebf #2e9e7a #8a5fc4 #e07b39 #1f9aa5 #d0527f #8d9a1f #6b7889`) asignada en orden al crear programas y editable en Ajustes. Sin proyecto: barra rosa `#c58aa0` y, en lugar del chip de proyecto, un chip punteado "proyecto" que abre el desplegable. Las filas de columna van sobre fondo de panel con 3 px de separación, sin línea inferior. No hay punto de color. Urgente: la barra pasa a roja (`--red`), título en 600 y fondo `--urgent-bg`. **Tarjeta ámbar** (fondo `--claim-bg`, borde `--wait-line`): la tarea es de otra persona (chip ≠ Yo) o es un reclamo que vuelve de Esperando. Si es urgente y de otra persona, fondo ámbar con barra roja.

En **Radar** el color del programa va solo en el borde izquierdo de la fila y en el nombre del bloque (`color-mix(programa 70%, texto)` en claro; `color-mix(programa 45%, #fff)` en oscuro). La estrella de foco usa `--star`. Nada más lleva color.

### Temas
Dos temas, elegidos tras probar seis: **Gris** (claro) y **Medio** (oscuro). `settings.theme` sigue siendo `light` | `dark`. Todo color sale de estos tokens; ningún componente lleva un color literal que solo funcione en un tema.

| token | Gris (light) | Medio (dark) |
|---|---|---|
| --bg | #b9bdc5 | #3a3e46 |
| --panel | #e3e5e9 | #4b5059 |
| --col (fondo de columna) | #c9ccd3 | = --bg |
| --soft | #eaecef | #535862 |
| --hover | #d4d7dd | #5d626c |
| --text | #1b1b1a | #f6f6f3 |
| --muted | #4a4d53 | #cdd0d6 |
| --line | #c4c7ce | #5e636d |
| --line2 | #949aa4 | #7f858f |
| --future | #27598f | #b4d4f7 |
| --accent | #1f4e8c | #b4d4f7 |
| --red | #b02c1a | #ff8d7a |
| --amber | #7a4f24 | #f5bd68 |
| --star | #b0800c | #f7cd5a |
| --urgent-bg | #efdcd8 | #5c3b37 |
| --claim-bg | #e9e3d9 | #574f45 |
| --wait-bg | #ded6c9 | #524a40 |
| --wait-line | #bfb09a | #7c6d59 |
| radio de tarjetas y paneles | 10 px | 8 px |

Chips de proyecto: fondo `color-mix(programa 26%, panel)` y texto `color-mix(programa 70%, texto)` en Gris; en Medio, fondo al 32 % y texto `color-mix(programa 35%, #fff)`. Chips de persona en Medio: fondo `color-mix(texto del chip 55%, panel)`, texto `color-mix(fondo del chip 92%, #fff)`. En Medio, `color-scheme: dark` para que los controles nativos sigan al tema. Los botones de tarjeta en Medio usan fondo `--soft`, borde `--line2` y texto `--text`.

**Chip de proyecto = etiqueta de color** (es lo principal de la segunda línea, antes que la persona), con el color del programa según la tabla de Temas. **Chip = persona**: el de Yo es discreto (contorno gris, sin relleno); los demás llevan su color. "Depende de alguien" no es un tipo: es que el chip no es Yo, y la tarjeta se pinta ámbar. Paleta de 8, fondo claro + texto oscuro (contraste ≥ 4.5:1), asignada en orden al crear personas; Yo siempre negro/blanco:

| nombre | fondo | texto |
|---|---|---|
| blue | #dbe8f7 | #1f4e8c |
| green | #d9efe8 | #1b6b54 |
| violet | #ece0f5 | #5e3a8c |
| orange | #fbe4d0 | #8a4a0f |
| teal | #d4eef0 | #0f6a70 |
| pink | #f8dde6 | #8c2f52 |
| olive | #e8ecd2 | #5a6414 |
| slate | #e2e5ea | #3b4756 |

## Línea rápida
Un input. Intro crea la tarea y vacía el input. Sintaxis, en cualquier orden:

- `@nombre` persona. Al teclear `@` se abre un desplegable con las personas guardadas, filtrando al escribir; flechas + Intro o clic selecciona. Última opción siempre "Crear persona: xxx"; una persona nueva **solo** se crea eligiendo esa opción explícitamente, nunca por un texto suelto. Nueva persona recibe el siguiente color de la paleta.
- `#proyecto` idéntico, con "Crear proyecto: xxx" (pide el programa en un desplegable pequeño).
- Fecha: `hoy`, `mañ`/`mañana`, `lun`…`dom` (próximo día de esa semana; si es hoy, hoy), `12/10`, `12/10/26`, `+3` (días), `nov`/`dic`… (día 1 del mes). Admite acentos y mayúsculas.
- `!` urgente. Sin marca: hacer. (`?` no tiene significado.)
- `Ctrl+Intro` en vez de `Intro` crea la tarea directamente en Hoy.
- Al crear: urgente sin fecha explícita → `date = hoy`. Persona ≠ Yo (y no urgente) → entra en Esperando con `remindOn = fecha explícita` o, si no hay, `hoy + waitDays`.
- Sin `@`: persona Yo. Sin fecha: según `settings.captureIn` (por defecto `week` = viernes de esta semana; si hoy es viernes o después, viernes de la siguiente). Opciones: today, tomorrow, week, nextweek.

Lo que queda después de extraer los tokens es el título, recortado.

## Mover tareas
Tres caminos, los tres cambian `date`/`waiting`:

- **Botones** en tarjeta (descritos en Layout).
- **Teclado** con una fila seleccionada (clic o flechas arriba/abajo): `H` hoy · `M` mañana · `S` esta semana (viernes) · `P` próxima semana (lunes) · `L` más adelante (+30 días) · `E` esperando · `A` arriba del todo en Hoy · `Alt+↑/↓` subir/bajar un puesto dentro de Hoy · `X` o `Supr` hecha · `R` crear un asunto en el Radar con el título de la tarea · `Intro` abrir edición · `Esc` deseleccionar.
- **Arrastre** entre columnas: soltar en Mañana → hoy+1; Esta semana → viernes de esta semana (si hoy ≥ viernes, viernes siguiente); Próxima semana → lunes siguiente; Más adelante → +30 días; Esperando → `waiting=true`, `remindOn = hoy + settings.waitDays`; Hoy → hoy. Dentro de Hoy, arrastrar reordena (`order`); al arrastrar cerca del borde superior o inferior de la lista, esta se desplaza sola.

Pasar a Esperando (botón, tecla, arrastre, panel o al crear) usa `remindOn = hoy + waitDays`; si cae en sábado o domingo, pasa al lunes siguiente. Sol en una tarea de Esperando = reclamar ya: `waiting=false`, `date=hoy`. Tecla `U` o botón `!` alterna urgente; al marcar urgente, `date = hoy` y sale de Esperando.

Hecha: `done=true, doneAt=ahora`; desaparece de las columnas. Botón "Hechas (N)" muestra una lista con fecha y permite recuperar. Las hechas de más de 90 días se pueden purgar desde Ajustes.

## Editar
**En sitio**, sin abrir nada: clic en título → edición inline (Intro guarda, Esc cancela; acepta la misma sintaxis `@ # !` y la aplica). Clic en chip persona → desplegable (incluye Yo). Clic en fecha → selector con atajos hoy · mañ · vie · lun + calendario nativo. Clic en chip proyecto → desplegable.

**Panel lateral** (doble clic o Intro): todos los campos + nota libre (`note`, textarea) + botón Borrar (con confirmación). Esc cierra.

## Filtros
Cabecera: chip "Todo", "Mías" (persona = Yo), "Reclamar" (persona ≠ Yo o waiting), y pestañas por programa; con un programa activo aparecen los chips de sus proyectos. Un filtro afecta a todas las columnas. En Radar, ver su sección. `/` enfoca un buscador de texto que filtra por título y nota (en Radar, por texto y nota del asunto).

## Atajos globales
`N` enfoca la línea rápida · `/` buscador · `?` muestra la tabla de atajos · `1` vista Día · `2` vista Radar. Las teclas de mover solo actúan con una fila seleccionada y nunca mientras se escribe en un input.

## Persistencia
1. Siempre: `localStorage` (clave `tareas.v1`), guardado en cada cambio.
2. Preferente: archivo `tareas.json` vinculado con File System Access API (`showSaveFilePicker` / `showOpenFilePicker`). Botón "Vincular archivo" en Ajustes; el handle se guarda en IndexedDB para no pedirlo cada vez; al abrir se pide permiso si hace falta. Guardado en cada cambio, con debounce 500 ms. Si hay archivo y localStorage, manda el archivo.
   Si hay cambios sin guardar en el archivo (sin permiso o error de escritura): aviso rojo en el pie, `beforeunload` al cerrar, y al conceder permiso con cambios pendientes se pregunta qué versión conservar. El primer clic de la sesión pide el permiso si falta.
3. Exportar / Importar JSON manual siempre disponible en Ajustes (respaldo y para navegadores sin la API).

`tareas.json` nunca se sube al repo (`.gitignore`).

## Ajustes (modal)
Capturar en · Alerta de espera (días) · Aviso en Hoy a partir de N · Tema claro/oscuro · Personas (lista editable: renombrar actualiza sus tareas, color, borrar solo si no tiene tareas) · Programas y proyectos (crear, renombrar, favorito, mover de programa) · Archivo vinculado · Exportar / Importar · Purgar hechas y asuntos resueltos antiguos (más de 90 días).

## Fuera de alcance (no implementar)
Recordatorios, notificaciones, correo, integración con Outlook (futuro), subtareas, adjuntos, multiusuario, sincronización, login, backend. Nada de librerías externas ni CDN.

Tampoco: hitos, riesgos, bloqueos, semáforos ni líneas de estado de proyecto (están en la herramienta de seguimiento), vista de calendario (está en Outlook), fecha límite en las tareas, estado o fecha en los asuntos, **enlace entre asuntos y tareas** (ni id, ni contador, ni chip que relacione una copia con su origen; ni `temaId` ni agrupar tareas por asunto), avisos en Radar. Se descartaron a propósito tras maquetarlos: la app no es el seguimiento del proyecto, es la lista de lo que el usuario mueve más el radar de lo que vigila.

## Calidad
- Todo en `index.html`. Código organizado en secciones: estado, parser, cálculo de columnas, render (Día y Radar), acciones, persistencia, ajustes.
- Migración de datos: un `tareas.json` con `v: 1` sin `radar` carga con `radar: []`. Si trae `temas`, `temaId` en tareas o `programId` en tareas (versión intermedia ya retirada), se ignoran y se descartan al guardar; opcionalmente, cada tema con tareas abiertas se convierte en un asunto con el mismo nombre y bloque.
- Render completo a partir del estado tras cada acción (sin DOM parcial frágil). Debe ir fluido con 500 tareas.
- Fechas siempre en local, formato ISO `YYYY-MM-DD` en datos; mostrar como "jue 8", "12/10" o "nov".
- Botones y chips son `<button>` reales; `aria-label` en los de icono.
- Probar en Edge y Chrome. Sin consola con errores.
