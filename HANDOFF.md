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

## 5. Decisión de estrategia (cerrada el 8/11/2026)
Se discutió durante varias vueltas cómo pasar de "inventario de tareas por fecha" a "estado de las cosas". Se maquetaron y **descartaron**, por este orden: hitos con fecha, frentes con semáforo y línea de estado, riesgos y bloqueos, vista de calendario mensual, fecha límite en tareas, tres looks de tablero (bloques, carriles, matriz), y por último los **temas como contenedores de tareas** (vista Proyectos con `temaId`, llegó a implementarse). Motivo del último descarte: obligaba a clasificar cada tarea al capturarla, rompía el "pim pam", y el usuario no la usaba.

Lo que queda, recogido en `CLAUDE.md`:
- **Dos listas separadas, dos vistas.** Día (tareas, la pantalla de siempre) y **Radar** (asuntos que el usuario vigila a su nivel, por proyecto). Nada enlaza una con la otra. Un asunto puede tener tareas en Día que se hagan sin que se resuelva, no tener ninguna, o resolverse solo.
- **Asunto**: texto, nota, a qué cuelga (proyecto, programa o global), orden manual, foco ★, aparcado, fecha de entrada, resuelto. Sin estado, sin fecha límite, sin dueño, sin avisos.
- **Radar es una lista, no cajas**: una fila por bloque, siempre las mismas; un proyecto sin asuntos es una línea; uno con siete, siete líneas.
- **Temas de color**: Gris (claro) y Medio (oscuro), elegidos entre seis candidatos. Tokens en la tabla de `CLAUDE.md`.

Maqueta de referencia: `maqueta-radar.html` (datos de ejemplo; pestañas Día/Radar y conmutador de tema). `maqueta-proyectos.html` queda obsoleta y se borra con la implementación.

## 6. Encargo de implementación (pendiente)
Partir del `index.html` actual, que ya tiene temas, vista Proyectos y look Color en claro/oscuro. Cambios, en este orden:
1. **Quitar temas**: entidad `temas`, `temaId` y `programId` en tareas, token `^`, chip de tema, sección Temas de Ajustes, vista Proyectos y su arrastre. Migración: un `tareas.json` con `temas`/`temaId`/`programId` en tareas carga y, al guardar, los descarta; opcionalmente convertir cada tema con tareas abiertas en un asunto del radar con el mismo nombre y bloque.
2. **Modelo `radar`** y persistencia, purga de resueltos antiguos junto con las hechas.
3. **Vista Radar** según `CLAUDE.md` y la maqueta: filas fijas, asuntos, "+ asunto", foco/aparcar/resuelto, resueltos plegados, `→ tarea`, edición en sitio y panel lateral, teclado y arrastre, filtros Todo/★ Foco/programas, buscador.
4. **Temas Gris y Medio**: sustituir los tokens actuales por la tabla de `CLAUDE.md`; mantener `settings.theme` `light`/`dark`.
5. Borrar `maqueta-proyectos.html`. Probar en Chromium sin errores de consola y, por parte del usuario, en Edge con `tareas.json` real.
