# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Ejecutar el proyecto

No hay build, ni tests, ni linter, ni `package.json`. Es un sitio estático de tres archivos.

```bash
start index.html                # Windows: abrir directo con file://
python -m http.server 8000      # servidor estático (recomendado)
npx serve .
```

Verificación = abrir el juego en el navegador y jugar. No existe suite de tests, así que
cualquier cambio en la lógica se comprueba manualmente: colocar una pieza, completar una
línea, subir de nivel, provocar Game Over y reiniciar.

## Arquitectura

`game.js` es un único script global (`'use strict'`, sin módulos, sin `export`), cargado al
final del `<body>`. El estado vive en variables `let` de nivel superior
(`board, current, next, score, lines, level, paused, gameOver, dropInterval, dropAccum, animId`)
y las referencias al DOM se cachean una sola vez al cargar el archivo. `init()` se ejecuta al
final del script y también es el handler del botón *Reiniciar*: reinicializa todo el estado
en su sitio, no recrea nada.

Bucle: `requestAnimationFrame(loop)` acumula `dt` en `dropAccum` y baja la pieza una fila
cuando supera `dropInterval`; `draw()` repinta el canvas completo en cada frame.

### Invariantes no obvias

- **El índice de tipo de pieza (1–7) es a la vez el índice de color y lo que se almacena en
  el tablero.** `PIECES[3]` está rellena de `3`, `COLORS[3]` es el morado de la T, y una
  celda ocupada del tablero guarda ese mismo `3`. `0` = celda vacía (por eso `COLORS[0]` y
  `PIECES[0]` son `null`). Al añadir piezas o colores hay que mantener las tres tablas
  alineadas y actualizar el `* 7` de `randomPiece()`.
- **Las dimensiones de los canvas están duplicadas en `index.html`.** `<canvas id="board">`
  es `300 × 600` = `COLS * BLOCK` × `ROWS * BLOCK`. Cambiar `COLS`, `ROWS` o `BLOCK` en
  `game.js` sin tocar el HTML deforma o recorta el tablero. Lo mismo con
  `<canvas id="next-canvas">` (`120 × 120`), que asume una rejilla 4×4 con el `NB = 30`
  local de `drawNext()` — ese valor es independiente de `BLOCK`.
- **`collide()` permite `ny < 0` a propósito.** Solo comprueba el contenido del tablero
  cuando `ny >= 0`, de modo que una pieza parcialmente por encima de la fila 0 (spawn,
  rotación pegada al techo) no cuenta como colisión.
- **Rotación casera, no SRS.** `rotateCW()` es transposición + inversión sobre la matriz
  cuadrada de la propia pieza; `tryRotate()` prueba desplazamientos horizontales
  `[0, -1, 1, -2, 2]` como *wall kick* y descarta el giro si ninguno cabe. No hay tabla de
  patadas por pieza ni estado de rotación.
- **El Game Over solo se detecta en `spawn()`**, cuando la pieza nueva ya colisiona.
- **`clearLines()` muta `board` con `splice` + `unshift`** e incrementa `r++` tras eliminar
  para volver a evaluar el mismo índice, que ahora contiene la fila de arriba.
- **La puntuación se multiplica por el nivel** (`LINE_SCORES[cleared] * level`), y el nivel
  se recalcula *después* de sumar (`Math.floor(lines / 10) + 1`), junto con
  `dropInterval = max(100, 1000 - (level - 1) * 90)`.
- **Al reanudar la pausa hay que reasignar `lastTime = performance.now()`** antes de relanzar
  `loop`, o el primer `dt` valdría todo el tiempo pausado y la pieza caería de golpe.

### Quirk conocido

Tras `endGame()` el bucle sigue vivo: `endGame()` se invoca desde dentro de `loop()`
(vía `lockPiece → spawn`), así que su `cancelAnimationFrame(animId)` cancela un frame ya
consumido y el `requestAnimationFrame` del final de `loop()` vuelve a programarse. Las piezas
siguen cayendo y bloqueándose detrás del overlay; solo el `keydown` está bloqueado por la
guarda `if (paused || gameOver) return`. Tenerlo en cuenta antes de tocar el ciclo de vida
del bucle.

## Convenciones

- Todo el texto visible al usuario está en español (`PAUSA`, `GAME OVER`, `Puntuación`,
  `Reiniciar`, la lista de controles en `index.html`). El código —identificadores y
  comentarios— está en inglés. El README también está en español; si se cambian mecánicas,
  puntuación o controles, sus secciones *Controles*, *Cómo funciona* y *Personalización*
  quedan desactualizadas.
- El input se maneja con un único `keydown` sobre `document` que despacha por `e.code`
  (`ArrowUp`/`KeyX` rotan, `Space` hace hard drop con `preventDefault`, `KeyP` pausa incluso
  estando en pausa). Al final del handler siempre se llama a `updateHUD()`.
- Estilos con un tema oscuro fijo mediante literales de color en `style.css` (no hay
  variables CSS pese a lo que dice el README); el azul de acento es `#7aa2f7`.
