# Handoff: app de tareas "Tareas"

Resumen para continuar el trabajo en otra sesión. La especificación completa y al día está en `CLAUDE.md`.

## 1. Qué es y cómo está montado
- **Producto:** gestor personal de tareas. Sustituye las banderas y categorías de color de Outlook. Principio: capturar y mover tareas tiene que ser "pim pam".
- **Tecnología:** un único `index.html` con HTML, CSS y JS vanilla. Sin build, sin dependencias y sin CDN.
- **Datos:**
  - `localStorage` (clave `tareas.v1`).
  - Un `tareas.json` vinculado con la File System Access API. El handle se guarda en IndexedDB y, si existe el archivo, manda el archivo. Está en `.gitignore` y nunca se sube al repo.
  - Exportar e importar JSON desde Ajustes.
  - El usuario tiene el `tareas.json` en una carpeta sincronizada (OneDrive).
- **Despliegue:** `main` se publica con GitHub Pages (repo público) en la URL de GitHub Pages del repo. Incluye `manifest.json` e iconos (`icono.svg`, `icono.ico`, `icono-192.png`, `icono-512.png`) para instalarla como aplicación en Edge. Motivo: Edge no deja instalar páginas locales y, como acceso directo `--app`, mostraba el icono de Edge en la barra de tareas.
- **Flujo de trabajo:**
  - Rama de trabajo `claude/sleepy-heisenberg-bliaqw`.
  - Cada cambio va en un PR contra `main` que se fusiona (PR #1 a #11).
  - Las pruebas se hicieron solo con Playwright en Chromium sin interfaz (sin errores de consola). **No se ha probado en Edge.**
  - El usuario pidió respuestas mínimas.

## 2. Modelo de datos actual
Programas (con `color`), proyectos (con `programId` y `fav`), personas (`me` siempre existe) y tareas. Cada tarea tiene `id`, `title`, `personId`, `projectId`, `type` (`hacer` o `urgente`), `date` (cuándo quiero verla, no un vencimiento), `waiting`, `remindOn` (fecha de reclamo), `note`, `order`, `done`, `doneAt` y `createdAt`.

## 3. Comportamiento implementado

**Columnas** (calculadas, nunca guardadas):

| Columna | Qué contiene |
|---|---|
| Hoy | Panel grande a la izquierda, con tarjetas |
| Mañana, Esta semana, Próxima semana, Más adelante | Listas compactas |
| Esperando | Lo que depende de otros o lo aparcado a mano |

- **Carriles:** Hoy separa "Mías" arriba y "De otros" abajo. Las demás columnas hacen lo mismo y solo muestran las cabeceras si hay tareas de ambos tipos.
- **Tarea de otra persona** (chip distinto de Yo, no urgente): está siempre **en espera** hasta su `remindOn` (si falta, usa `date`). Ese día aparece en Hoy como reclamo, con fondo ámbar.
  - Tocar su fecha cambia el reclamo.
  - Asignar una tarea mía a otra persona la pone en espera. Devolverla a Yo la saca.
  - Una urgente de otro va a Hoy, sin espera.
- **Esperar:** por defecto son 5 días (ajustable). Si el día cae en sábado o domingo, pasa al lunes siguiente.

**Línea rápida** (un solo input, con Intro crea la tarea):
- `@persona` y `#proyecto`, con desplegable. "Crear persona" y "Crear proyecto" son opciones explícitas, y un texto suelto nunca crea nada.
- Fechas: `hoy`, `mañ`, `lun`…`dom`, `12/10`, `+3`, `nov`.
- `!` marca urgente. Una urgente sin fecha explícita queda para hoy.
- `Ctrl+Intro` crea la tarea directamente en Hoy.
- Si la tarea es de otra persona, entra en Esperando.
- Sin fecha, se usa el ajuste "Capturar en" (por defecto, el viernes).

**Mover y editar:**
- Botones en la tarjeta: `→ mañ`, `→ vie`, `⏸ espera`, `!` y `✓`. En las filas aparecen al pasar el ratón.
- Teclado, con una fila seleccionada: `H M S P L E`, `X`, `U` (alterna urgente), `A` (arriba del todo en Hoy) y `Alt+↑/↓` (reordenar dentro del carril).
- Arrastre entre columnas, con reorden dentro de Hoy y autoscroll en los bordes de la lista.
- Edición en sitio del título, del chip de persona, de la fecha y del proyecto. Panel lateral con doble clic.
- Filtros: Todo, Mías, Reclamar y programas con sus proyectos. Búsqueda con `/`.
- Hechas, con opción de recuperarlas y purgar las de más de 90 días. Ajustes con personas, programas y proyectos (color de programa editable) y archivo vinculado.

**Visual:**
- Estilo igualado a la maqueta del repo.
- **Punto de color** delante del título con el color del **programa**. Si no hay proyecto, es rosa discontinuo y al hacer clic abre el desplegable de proyectos.
- **Chip de proyecto** como etiqueta de color, con el color del programa. Va antes que el de persona. El chip de persona "Yo" es discreto.
- Urgente: barra roja a la izquierda. Tarea de otro o reclamo: fondo ámbar.
- Fechas futuras en azul, atrasadas en rojo.

**Persistencia segura:**
- Si hay cambios sin guardar en el archivo, sale un aviso rojo en el pie y `beforeunload` al cerrar.
- Al conceder el permiso con cambios pendientes, se pregunta qué versión conservar.
- El primer clic de la sesión pide el permiso si falta.

## 4. Pendiente o sin probar
- **Integración con Outlook** (Outlook nuevo y Power Automate disponible). Se propuso un flujo "correo marcado" que crea un archivo por correo en una carpeta de OneDrive, y que la app lea esa carpeta al abrirse. El usuario dijo que de momento no se haga nada.
- **No probado:** Edge, instalar la app, vincular y reabrir `tareas.json`, ni los avisos al cerrar con un archivo real.

## 5. Decisión de estrategia (cerrada el 30/10/2026)
Se discutió durante varias vueltas cómo pasar de "inventario de tareas por fecha" a "estado de las cosas". Se maquetaron y **descartaron**: hitos con fecha, frentes con semáforo y línea de estado, riesgos y bloqueos, vista de calendario mensual, fecha límite en tareas, y tres looks de tablero (bloques, carriles, matriz). Motivo: todo eso vive en otra herramienta (hitos, riesgos) o en Outlook (calendario), y mezclarlo convertía la app en "todo y nada".

Lo que queda, ya recogido en `CLAUDE.md`:
- **Una pregunta**: qué muevo yo y a quién reclamo. Si un hito o riesgo exige algo de Rubén, entra como tarea con fecha.
- **Tema**: la única entidad nueva. Un nombre más una nota para juntar tareas de lo mismo; cuelga de proyecto, programa o nada. Sin estado, sin fecha, sin semáforo. `temaId` opcional en la tarea. Token `^tema` en la línea rápida.
- **Dos vistas**: Día (la pantalla actual, con chip de tema) y Proyectos (bloques fijos: Global, y por programa un bloque transversal más uno por proyecto; dentro, temas con sus tareas y las sueltas). Dos avisos: "sin tarea siguiente" y "demasiados temas / muchas".
- **Look "Color"**: barra lateral del color del programa en tarjetas y filas (sustituye al punto), bloques de Proyectos teñidos con el color del programa, tema claro y oscuro del mismo diseño.

Maqueta de referencia: `maqueta-proyectos.html` (datos de ejemplo; pestañas Día/Proyectos y conmutador de look en la franja superior; el look elegido es C/D "Color"). `maqueta-seguimiento.html` se ha eliminado por superada.

## 6. Orden de implementación propuesto
1. Modelo: `temas` y `temaId`, migración desde `v: 1` sin temas, Ajustes de temas.
2. Línea rápida y edición en sitio: token `^`, desplegable con "Crear tema", chip de tema en tarjetas y filas.
3. Vista Proyectos con el conmutador y las teclas `1`/`2`; filtros aplicados a ambas vistas.
4. Look Color: barra lateral, filas sobre blanco, bloques teñidos, ajustes para tema oscuro.
5. Probar en Edge con `tareas.json` real.

### Estado de la implementación (30/10/2026)
Hechos los pasos 1 a 4 en `index.html`: temas y migración, token `^tema` (con "Crear tema", que pide de qué cuelga) y chip de tema, vista Proyectos con teclas `1`/`2` y filtros, y look Color en claro y oscuro (barra lateral por programa, bloques teñidos, filas blancas). Ajustes incluye la sección Temas. Probado solo en Chromium sin interfaz; falta el paso 5 (Edge con un `tareas.json` real).

Detalles a tener en cuenta:
- En la línea rápida, un tema o proyecto de varias palabras se escribe con guiones bajos (`^Tema_de_ejemplo`), igual que las personas. Para nombres con espacios también valen "+ tema" en Proyectos y Ajustes.
- "+ tarea" en un tema rellena la línea rápida con `^tema` y `#proyecto`; el usuario solo escribe el título.
- Al borrar un proyecto, sus temas pasan a transversales del programa; al mover un proyecto de programa, sus temas lo siguen; al borrar un programa, sus temas pasan a Global.
