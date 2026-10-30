# Tareas — app personal de escritorio

## Comunicación
Respuestas mínimas. Sin explicar lo que vas a hacer ni resumir lo hecho. Si hay duda real, una pregunta corta; si no, decide y sigue.

## Qué es
Gestor personal de tareas. Un solo `index.html` (HTML + CSS + JS vanilla, sin build, sin dependencias externas, sin frameworks). Se abre en Edge/Chrome desde el disco. Sustituye las banderas y categorías de color de Outlook. No es Planner: nada de recordatorios, notificaciones, subtareas, adjuntos ni colaboración.

Principio: capturar y mover tareas tiene que ser "pim pam". Una línea de texto para crear, un clic o una tecla para mover.

Responde a una sola pregunta: **qué muevo yo y a quién tengo que reclamar**. Los hitos, riesgos y bloqueos de los proyectos viven en la herramienta corporativa de seguimiento de proyectos; el calendario, en Outlook. Si uno de ellos exige algo del usuario, entra como tarea con fecha. Lo que no sea una acción suya o un reclamo no entra.

Dos vistas sobre la misma lista (pestañas en cabecera, teclas `1` y `2`): **Día** (Hoy + columnas por fecha, la pantalla principal) y **Proyectos** (bloques fijos por proyecto, con sus temas y tareas). Marcar hecha en una la quita de la otra.

## Modelo de datos

```json
{
  "v": 1,
  "settings": { "captureIn": "week", "waitDays": 5, "warnToday": 5, "theme": "light" },
  "programs":  [{ "id": "p1", "name": "Programa A", "color": "#3d7ebf" }, { "id": "p2", "name": "Programa B", "color": "#2e9e7a" }],
  "projects":  [{ "id": "pr1", "programId": "p1", "name": "Proyecto A1", "fav": true }],
  "people":    [{ "id": "me", "name": "Yo", "color": "black" }, { "id": "u1", "name": "Persona 1", "color": "blue" }],
  "temas":     [{ "id": "m1", "programId": "p1", "projectId": "pr1", "name": "Incidencia abierta", "note": "" }],
  "tasks": [{
    "id": "t1", "title": "Pedir presupuesto",
    "personId": "u1", "projectId": "pr1", "temaId": "m1",
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
- Cada proyecto pertenece a un programa. Las tareas llevan proyecto; solo si no tienen proyecto pueden llevar un `programId` opcional, que las deja sueltas en el bloque transversal de ese programa (se fija arrastrando en la vista Proyectos).
- **Tema** = un nombre para juntar tareas de lo mismo ("Informes para dirección", "Incidencia abierta", "Plan de equipo"). Cuelga de un proyecto (`projectId`, y `programId` el del proyecto), de un programa (`programId` solo) o de nada (global). Lleva `name` y `note` libre, y **nada más**: sin estado, sin fecha, sin semáforo, sin dueño. Su estado es su siguiente tarea abierta. `temaId` en la tarea es opcional e independiente de `projectId` (una tarea puede tener proyecto sin tema, tema sin proyecto, ambos o ninguno). Un tema se crea solo desde "Crear tema" en la línea rápida o desde Proyectos/Ajustes; se borra desde Ajustes (sus tareas quedan sin tema).

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
Pantalla de escritorio, 1280+ px. Fondo claro `#eeeff2`, tipografía system-ui (sin fuentes externas), texto `#1b1b1a`. Referencia visual: `maqueta-proyectos.html` con el look **Color** (claro y oscuro).

1. **Cabecera**: título, fecha de hoy, conmutador de vista **Día | Proyectos**. A la derecha: filtros (ver abajo), botón Hechas (N), botón Ajustes.
2. **Línea rápida**: input grande (46 px, 16 px de texto). Debajo, una línea de ayuda con la sintaxis y los atajos.
3. **Hoy**: panel blanco a la izquierda, ~35 % del ancho, con dos carriles (swimlanes): "Mías" arriba y "De otros" abajo (fondo ámbar `#f1eae1`, borde `#dccbb8`), cada uno con su cabecera y contador. El resto de columnas hacen lo mismo y muestran las cabeceras de carril solo si hay tareas de ambos tipos. Orden manual dentro de cada carril. Tarjetas grandes: título 16 px, chip de persona, chip de proyecto, fecha, y botones siempre visibles: `→ mañ`, `→ vie`, `⏸ espera`, `✓`. Orden manual por arrastre dentro de Hoy.
4. **Resto**: cinco columnas a la derecha en formato lista compacta (una fila = título 13 px; segunda línea chip persona + fecha). Sin botones visibles; al pasar el ratón aparecen: sol (a Hoy), `→ mañ`, `⏸`, `✓`. Esperando con fondo `#f1eae1`, borde `#dccbb8`, cabecera ámbar, y la fecha de cada fila dice "reclamar jue 8".
5. **Pie**: leyenda de colores y estado de guardado ("Guardado en tareas.json · hh:mm" o "Solo navegador · vincular archivo en Ajustes").

Las tarjetas y filas muestran, tras el chip de proyecto, el **chip de tema** si lo hay: punteado, gris, con prefijo "›" (`› Incidencia abierta`). Clic abre el desplegable de temas.

### Vista Proyectos
Sustituye a Hoy + columnas cuando está activa; la línea rápida, cabecera y pie se mantienen. **Bloques fijos** (uno por proyecto, más el global y uno transversal por programa), siempre los mismos aunque estén vacíos, en una cuadrícula de 4 por fila agrupada en bandas: banda **Global** (1 bloque: temas y tareas sin programa), y una banda por programa (un bloque "Programa · transversal" con los temas de programa, más un bloque por proyecto). Los bloques se derivan de `programs` y `projects`; si se crean proyectos nuevos aparecen bloques nuevos.

Cada bloque: cabecera con nombre y "N tareas"; dentro, sus **temas** (nombre en negrita con "›", contador, `+ tarea` al pasar el ratón) con sus tareas abiertas debajo ordenadas por fecha de ver/reclamar, y al final el grupo **Sueltas** con las tareas del bloque sin tema. Pie con `+ tema`. Cada fila de tarea: círculo para marcar hecha, título, chip de persona, fecha o "reclamar"; fondo ámbar si es de otro, título en negrita y círculo rojo si es urgente. Clic en título edita en sitio; doble clic abre el panel lateral. **Arrastre entre bloques**: arrastrar una fila y soltarla sobre un tema la pasa a ese tema (y a su proyecto si el tema lo tiene; si el tema es transversal o global, la tarea conserva el proyecto que tuviera); soltarla sobre un bloque de proyecto fuera de un tema la deja suelta en ese proyecto; sobre Global, suelta y sin proyecto. Soltarla sobre un bloque transversal, fuera de un tema, la deja suelta en ese programa (`programId`, sin proyecto ni tema). Todo bloque tiene su grupo **Sueltas** cuando hay tareas sueltas en él. El destino se resalta con borde discontinuo y el tablero se desplaza solo cerca de sus bordes.

Avisos, y ningún otro: tema sin tarea abierta → "sin tarea siguiente" en ámbar; más de 4 temas en un bloque → "demasiados temas"; más de 5 tareas en un tema → "muchas". Bloque sin nada → "Nada abierto" en cursiva.

Los filtros de cabecera actúan también aquí: Mías/Reclamar y el buscador filtran las filas; un programa activo oculta las otras bandas. Los bloques de la banda visible no desaparecen nunca.

Referencia visual: la maqueta acordada (panel Hoy grande a la izquierda, listas compactas a la derecha). No copiar el estilo de otras apps: nada de campos "Persona / Fecha límite" vacíos en cada tarjeta.

## Colores
Dos dimensiones separadas:

**Barra lateral = programa** (look Color): cada tarjeta de Hoy lleva una barra izquierda de 4 px y cada fila de columna una de 3 px con el color del programa, de una paleta de 8 (`#3d7ebf #2e9e7a #8a5fc4 #e07b39 #1f9aa5 #d0527f #8d9a1f #6b7889`) asignada en orden al crear programas y editable en Ajustes. Sin proyecto: barra rosa `#c58aa0` y, en lugar del chip de proyecto, un chip punteado "proyecto" que abre el desplegable. Las filas de columna van sobre fondo blanco con 3 px de separación, sin línea inferior. Sustituye al punto de color anterior: ya no hay punto. Urgente: la barra pasa a roja `#c2321e`, título en 600 y fondo `#fff7f5`. **Tarjeta ámbar** (fondo `#faf6f1`, borde `#dccbb8`): la tarea es de otra persona (chip ≠ Yo) o es un reclamo que vuelve de Esperando. Si es urgente y de otra persona, fondo ámbar con barra roja.

En **Proyectos** el programa tiñe el bloque: fondo `color-mix(programa 7%, panel)`, borde `color-mix(programa 30%, panel)`, borde superior de 4 px del color, nombre del bloque en `color-mix(programa 70%, texto)`. Los bloques transversales y Global, con borde discontinuo y tinte al 3 %. En tema oscuro los porcentajes suben a 10 % / 35 % y los chips de persona se oscurecen (`color-mix(texto del chip 35%, panel)` de fondo).

**Chip de proyecto = etiqueta de color** (es lo principal de la segunda línea, antes que la persona): fondo `color-mix(programa 18%, panel)` y texto `color-mix(programa 70%, texto)`, con el color del programa. **Chip = persona**: el de Yo es discreto (contorno gris, sin relleno); los demás llevan su color. "Depende de alguien" no es un tipo: es que el chip no es Yo, y la tarjeta se pinta ámbar. Paleta de 8, fondo claro + texto oscuro (contraste ≥ 4.5:1), asignada en orden al crear personas; Yo siempre negro/blanco:

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
- `^tema` idéntico, con "Crear tema: xxx" (pide a qué cuelga: el proyecto de la tarea si lo tiene, un programa, o global). Opcional; la mayoría de tareas no llevan tema.
- Fecha: `hoy`, `mañ`/`mañana`, `lun`…`dom` (próximo día de esa semana; si es hoy, hoy), `12/10`, `12/10/26`, `+3` (días), `nov`/`dic`… (día 1 del mes). Admite acentos y mayúsculas.
- `!` urgente. Sin marca: hacer. (`?` no tiene significado.)
- `Ctrl+Intro` en vez de `Intro` crea la tarea directamente en Hoy.
- Al crear: urgente sin fecha explícita → `date = hoy`. Persona ≠ Yo (y no urgente) → entra en Esperando con `remindOn = fecha explícita` o, si no hay, `hoy + waitDays`.
- Sin `@`: persona Yo. Sin fecha: según `settings.captureIn` (por defecto `week` = viernes de esta semana; si hoy es viernes o después, viernes de la siguiente). Opciones: today, tomorrow, week, nextweek.

Lo que queda después de extraer los tokens es el título, recortado.

## Mover tareas
Tres caminos, los tres cambian `date`/`waiting`:

- **Botones** en tarjeta (descritos en Layout).
- **Teclado** con una fila seleccionada (clic o flechas arriba/abajo): `H` hoy · `M` mañana · `S` esta semana (viernes) · `P` próxima semana (lunes) · `L` más adelante (+30 días) · `E` esperando · `A` arriba del todo en Hoy · `Alt+↑/↓` subir/bajar un puesto dentro de Hoy · `X` o `Supr` hecha · `Intro` abrir edición · `Esc` deseleccionar.
- **Arrastre** entre columnas: soltar en Mañana → hoy+1; Esta semana → viernes de esta semana (si hoy ≥ viernes, viernes siguiente); Próxima semana → lunes siguiente; Más adelante → +30 días; Esperando → `waiting=true`, `remindOn = hoy + settings.waitDays`; Hoy → hoy. Dentro de Hoy, arrastrar reordena (`order`); al arrastrar cerca del borde superior o inferior de la lista, esta se desplaza sola.

Pasar a Esperando (botón, tecla, arrastre, panel o al crear) usa `remindOn = hoy + waitDays`; si cae en sábado o domingo, pasa al lunes siguiente. Sol en una tarea de Esperando = reclamar ya: `waiting=false`, `date=hoy`. Tecla `U` o botón `!` alterna urgente; al marcar urgente, `date = hoy` y sale de Esperando.

Hecha: `done=true, doneAt=ahora`; desaparece de las columnas. Botón "Hechas (N)" muestra una lista con fecha y permite recuperar. Las hechas de más de 90 días se pueden purgar desde Ajustes.

## Editar
**En sitio**, sin abrir nada: clic en título → edición inline (Intro guarda, Esc cancela; acepta la misma sintaxis `@ # ^ !` y la aplica). Clic en chip persona → desplegable (incluye Yo). Clic en fecha → selector con atajos hoy · mañ · vie · lun + calendario nativo. Clic en chip proyecto → desplegable. Clic en chip tema → desplegable con "Sin tema" y "Crear tema".

**Panel lateral** (doble clic o Intro): todos los campos + nota libre (`note`, textarea) + botón Borrar (con confirmación). Esc cierra.

## Filtros
Cabecera: chip "Todo", "Mías" (persona = Yo), "Reclamar" (persona ≠ Yo o waiting), y pestañas por programa; con un programa activo aparecen los chips de sus proyectos. Un filtro afecta a todas las columnas y a la vista Proyectos. `/` enfoca un buscador de texto que filtra por título, nota y nombre del tema.

## Atajos globales
`N` enfoca la línea rápida · `/` buscador · `?` muestra la tabla de atajos · `1` vista Día · `2` vista Proyectos. Las teclas de mover solo actúan con una fila seleccionada y nunca mientras se escribe en un input.

## Persistencia
1. Siempre: `localStorage` (clave `tareas.v1`), guardado en cada cambio.
2. Preferente: archivo `tareas.json` vinculado con File System Access API (`showSaveFilePicker` / `showOpenFilePicker`). Botón "Vincular archivo" en Ajustes; el handle se guarda en IndexedDB para no pedirlo cada vez; al abrir se pide permiso si hace falta. Guardado en cada cambio, con debounce 500 ms. Si hay archivo y localStorage, manda el archivo.
   Si hay cambios sin guardar en el archivo (sin permiso o error de escritura): aviso rojo en el pie, `beforeunload` al cerrar, y al conceder permiso con cambios pendientes se pregunta qué versión conservar. El primer clic de la sesión pide el permiso si falta.
3. Exportar / Importar JSON manual siempre disponible en Ajustes (respaldo y para navegadores sin la API).

`tareas.json` nunca se sube al repo (`.gitignore`).

## Ajustes (modal)
Capturar en · Alerta de espera (días) · Aviso en Hoy a partir de N · Tema claro/oscuro · Personas (lista editable: renombrar actualiza sus tareas, color, borrar solo si no tiene tareas) · Programas y proyectos (crear, renombrar, favorito, mover de programa) · Temas (renombrar, mover de proyecto/programa, borrar: sus tareas quedan sin tema) · Archivo vinculado · Exportar / Importar · Purgar hechas antiguas.

## Fuera de alcance (no implementar)
Recordatorios, notificaciones, correo, integración con Outlook (futuro), subtareas, adjuntos, multiusuario, sincronización, login, backend. Nada de librerías externas ni CDN.

Tampoco: hitos, riesgos, bloqueos, semáforos ni líneas de estado de proyecto (están en la herramienta de seguimiento), vista de calendario (está en Outlook), fecha límite en las tareas, estado o fecha en los temas. Se descartaron a propósito tras maquetarlos: la app no es el seguimiento del proyecto, es la lista de lo que el usuario mueve.

## Calidad
- Todo en `index.html`. Código organizado en secciones: estado, parser, cálculo de columnas, render (Día y Proyectos), acciones, persistencia, ajustes.
- Migración de datos: un `tareas.json` con `v: 1` sin `temas` ni `temaId` debe cargar sin más (`temas: []`, `temaId: null`).
- Render completo a partir del estado tras cada acción (sin DOM parcial frágil). Debe ir fluido con 500 tareas.
- Fechas siempre en local, formato ISO `YYYY-MM-DD` en datos; mostrar como "jue 8", "12/10" o "nov".
- Botones y chips son `<button>` reales; `aria-label` en los de icono.
- Probar en Edge y Chrome. Sin consola con errores.
