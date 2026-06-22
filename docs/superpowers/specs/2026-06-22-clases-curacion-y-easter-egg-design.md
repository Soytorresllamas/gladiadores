# Gladiadores — Mejoras: Clases, Curación Konami y Easter Egg

- **Fecha:** 2026-06-22
- **Repo:** `gladiadores` — el juego completo vive en un solo archivo (`index.html`).
- **Estado actual:** juego local para 2 jugadores, pixel-art 16-bits, canvas 480×270. Gana el primero en conectar **3 espadazos** al rival. Ya existen 2 easter eggs: tecla `8` ×3 → tarjeta "GRACIAS POR JUGAR · 8-BITS"; tecla `g` ×3 → perritos fantasma.

## Objetivo

Tres mejoras:

1. **Selección de personaje:** gladiador (el actual), arquero o bombardero.
2. **Curación por código Konami:** completas una secuencia con tus teclas, tu perrito ladra y te cura una vez por partida.
3. **Tercer easter egg:** invasión de perritos.

## Decisiones tomadas (sesión de brainstorming, 2026-06-22)

- **Clases = "base común + un sello".** Las tres comparten la misma base (mover + los 4 botones); cada clase cambia su ataque principal, pierde una herramienta y gana un rasgo propio.
- **Curación = recuperar una vida** (restar 1 al marcador de golpes del rival).
- **Easter egg = invasión de perritos** (estampida cosmética de mini-perritos).

## Modelo mental clave

No hay barra de vida. `fighter.hits` cuenta los golpes que **ese** jugador ha conectado al rival; gana el primero en llegar a 3 (`WIN_HITS`). Por tanto la "vida" de un jugador es `3 − hits del rival`. **Curar = `rival.hits = max(0, rival.hits − 1)`.**

---

## Feature 1 — Clases (gladiador / arquero / bombardero)

Cada jugador elige su clase por separado. Se permite espejo (dos arqueros, etc.). Todos ganan igual: primero en conectar 3 golpes.

### Mapa de botones (Rojo `A/D/W/S/Q/E` · Azul `J/L/I/K/O/U`)

| Botón | 🗡️ Gladiador (actual) | 🏹 Arquero | 💣 Bombardero |
|---|---|---|---|
| Mover (`A/D` · `J/L`) | mover | mover | mover |
| Ataque (`W` · `I`) | espadazo melee (crece a los 2 aciertos) | **flecha** recta a distancia | **bomba** en arco con explosión en área |
| Escudo (`S` · `K`, mantener) | bloquea | **esquive (dash con i-frames)** | bloquea |
| Tirar escudo (`Q` · `O`) | proyectil garantizado | — (sin escudo) | proyectil garantizado |
| Parry (`E` · `U`) | refleja el golpe | refleja el golpe | **— (sin parry)** |

### Mecánicas por clase

**Gladiador** — sin cambios. Es la configuración por defecto de `makeFighter`. Conserva el perk de crecer la espada a los 2 aciertos (`swordLvl 1 → 1.5`).

**Arquero**
- **Ataque (flecha):** proyectil recto a la altura del torso (`y ≈ GROUND_Y − 34`). Velocidad ~7 px/frame. Pequeño wind-up al disparar (~4 frames) y cooldown entre flechas (~25 frames) para que no sea spameable.
- **Resolución del impacto:**
  - Alcanza la `x` del rival dentro del ancho de cuerpo (`FW`) → impacto (`scoreHit`).
  - Rival bloqueando con escudo (gladiador o bombardero) → flecha bloqueada; cuenta como fallo del arquero (`registerMiss`).
  - Rival hace parry dentro de la ventana → flecha reflejada como golpe del defensor (misma regla que el melee).
  - Rival en i-frames de dash (otro arquero) → la flecha lo atraviesa, sin daño.
  - La flecha se destruye al impactar, ser bloqueada, parry o salir de pantalla.
- **Sello:** SIN escudo (ni mantener ni tirar). El botón de escudo (`S`/`K`) hace **esquive**: un dash corto en la dirección que está moviendo (si no mueve, hacia atrás), con ~8 frames de invulnerabilidad (`invuln`) y cooldown ~30 frames. Conserva el parry.

**Bombardero**
- **Ataque (bomba):** proyectil parabólico (con gravedad) lanzado en `dir`. Al tocar el suelo **explota**: área de radio ~30 px alrededor del punto de caída.
  - Rival dentro del radio y sin bloquear → impacto (`scoreHit`) + empuje fuerte (knockback).
  - Rival dentro del radio y bloqueando (escudo arriba) → no recibe golpe, pero **sí el empuje** (queda descolocado).
  - Rival en i-frames de dash → no recibe golpe ni empuje.
  - La explosión solo evalúa al rival, nunca al propio lanzador (no hay daño propio).
  - Cooldown largo (~45 frames). Trayectoria visible y relativamente lenta (telegrafiada).
- **Sello:** SIN parry. Conserva escudo (mantener para bloquear) y tirar-escudo (proyectil garantizado). La bomba **no se puede frenar con parry**: solo se esquiva o se bloquea el golpe directo.

### Balance (triángulo, se afina jugando)

- 🗡️ vs 🏹: el arquero pega de lejos, pero el gladiador bloquea flechas con el escudo y al cerrar distancia domina.
- 🏹 vs 💣: el arquero esquiva las bombas lentas y pica de lejos; una bomba bien puesta lo arrincona.
- 💣 vs 🗡️: la bomba ignora el parry (le quita al gladiador su mejor herramienta) y niega el acercamiento; el gladiador aguanta con escudo y castiga con el tirar-escudo.

### Tunables iniciales (descriptor `CLASSES`, todos ajustables)

- **Arquero:** `arrowSpeed: 7`, `arrowWindup: 4`, `arrowCooldown: 25`, `dashFrames: 8`, `dashSpeedMult: 2.5`, `dashCooldown: 30`, `dashInvuln: 8`.
- **Bombardero:** `bombVx: 3.2 * dir`, `bombVy: -4.5`, `bombGravity: 0.3`, `blastRadius: 30`, `bombKnockback: 10`, `bombCooldown: 45`.
- **Gladiador:** valores actuales (sin cambios).

---

## Feature 2 — Pantalla de selección

Se extiende la pantalla de inicio actual (`state === 'ready'`, que ya deja elegir perrito).

- **Clase:** cada lado la cambia con sus teclas de mover (`A/D` Rojo · `J/L` Azul). Se muestra la silueta de la clase elegida y su mini-tabla de controles.
- **Perrito:** click en tu lado lo cambia (igual que hoy).
- **Empezar:** `Espacio` / `Enter` arranca con lo seleccionado (así una sola persona puede configurar ambos lados).
- Se permite espejo.
- La leyenda HTML bajo el canvas se adapta o muestra una nota: "los controles de escudo/parry cambian según tu clase".

**Cambio de comportamiento respecto a hoy:** actualmente cualquier tecla de juego inicia la partida. Ahora las teclas de mover se reservan para elegir clase en la pantalla de inicio, y el arranque pasa a `Espacio`/`Enter`.

---

## Feature 3 — Curación por Konami + ladrido

- **Secuencia (por jugador, con sus propias teclas, solo durante `play`):** `izquierda, izquierda, derecha, derecha, escudo, ataque`.
  - Rojo: `A A D D S W` · Azul: `J J L L K I`.
  - Usa teclas que **todas las clases tienen** (mover/escudo/ataque), para que funcione con cualquier personaje.
  - Cada tecla debe entrar dentro de ~1.2 s de la anterior (mismo patrón de timing que los easter eggs actuales). Una entrada fuera de secuencia reinicia el progreso de ese jugador.
- **Efecto:** `rival.hits = max(0, rival.hits − 1)`. **Una vez por jugador por partida** (`fighter.healUsed`). Si `rival.hits` ya es 0, el perrito ladra pero no se consume el uso.
- **El ladrido se ve:** el perrito de tu esquina da un brinco, aparece un globo grande "¡GUAU!" con ondas de sonido; el luchador destella verde y sale un `floatMsg("+1 VIDA")`.
- **Sonido:** se sintetiza con WebAudio (un "woof" de dos tonos descendentes), para no agregar archivos al repo. Alternativa si se prefiere: `sonidos/ladrido.mp3`.
- `healUsed` se reinicia en `reset()` (cada nueva partida).

---

## Feature 4 — Easter egg #3: invasión de perritos

- **Trigger:** tecla `p` ×3 dentro de ~1.3 s (mismo patrón que `8`×3 y `g`×3). `p` no es tecla de juego, así que no choca.
- **Efecto (cosmético, ~4 s):** una estampida de ~12 mini-perritos de razas aleatorias cruza la arena rebotando, con ladridos ocasionales. Reusa la función de dibujo de perritos existente (`drawBreed`) a escala pequeña. No afecta el combate. Puede activarse en `ready`, `play` o `win`.

---

## Arquitectura

- Todo sigue en `index.html` (un solo archivo).
- Nuevo descriptor `CLASSES = { glad:{…}, archer:{…}, bomber:{…} }` con tunables y flags por clase (`attackType`, `hasShield`, `canThrowShield`, `hasParry`, `hasDodge`, etc.).
- `makeFighter` recibe `cls` (default `'glad'` = comportamiento idéntico al actual).
- **Puntos de ramificación (los únicos que cambian):**
  1. **Entrada del botón escudo** (`S`/`K`): mantener-para-bloquear (gladiador/bombardero) vs. tap-para-dash (arquero).
  2. **`tryAttack` / `resolveAttack`:** según `attackType` (`melee` / `arrow` / `bomb`).
  3. **`updateProjectiles`:** además del escudo lanzado, maneja flechas (recta) y bombas (gravedad + explosión de área). Chequea `def.invuln` (dash) y `def.shielding`.
  4. **`drawFighter` / dibujo del arma:** arco de flecha, pose de dash, bomba en vuelo y explosión.
  5. **Pantalla `ready`:** selección de clase + render de silueta y controles.
  6. **`keydown`:** detección de la secuencia Konami por jugador + el egg `p`×3.
- **Riesgo bajo:** el gladiador es la config por defecto e intacta, así que lo que ya funciona no se toca.

---

## Fuera de alcance (por ahora)

- Rampa de poder para arquero/bombardero equivalente al crecer-espada del gladiador. Posible polish futuro.
- Modo un jugador / IA. Sigue siendo 2 jugadores local.
- Sonido de ladrido como archivo (usamos synth salvo que se pida).
- Balance fino entre clases: se ajusta jugando, no en este spec.

---

## Verificación

Manual, en navegador (abrir `index.html` o GitHub Pages), apoyado en las herramientas de preview (snapshot/console/screenshot). Checklist:

- Cada clase ataca y resuelve correctamente contra cada una de las otras (incluido espejo).
- Arquero: flecha vuela, la bloquea el escudo, la refleja el parry; dash con i-frames evita golpes.
- Bombardero: bomba cae en arco, explota en área, empuja; bloquear evita el golpe pero no el empuje; no tiene parry.
- Konami: la secuencia de cada jugador cura una sola vez por partida y dispara el ladrido; no se gasta si no hay daño.
- Easter egg: `p`×3 dispara la estampida; los dos easter eggs viejos (`8`×3, `g`×3) siguen funcionando.
- Revancha (`reset`) reinicia `healUsed` y las selecciones siguen disponibles.
