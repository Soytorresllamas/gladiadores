# Gladiadores — Clases, Curación Konami y Easter Egg · Plan de Implementación

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Añadir selección de personaje (gladiador/arquero/bombardero), una curación por código Konami con ladrido del perrito, y un tercer easter egg (invasión de perritos) al juego `Gladiadores`.

**Architecture:** Todo el juego vive en un solo archivo `index.html` (IIFE con canvas 2D). Se añade un descriptor `CLASSES` con tunables/flags por clase y se ramifica solo en los puntos que cambian (entrada del botón escudo, resolución de ataque, proyectiles, dibujo del arma, pantalla de inicio y `keydown`). El gladiador es la configuración por defecto, así que lo existente no se altera.

**Tech Stack:** HTML + CSS + JavaScript vanilla, Canvas 2D API, WebAudio (para el ladrido sintetizado). Sin build, sin dependencias, sin framework de tests.

**Verificación:** No hay test runner. Cada tarea se verifica en el navegador con las herramientas de preview (`preview_start`, `preview_console_logs`, `preview_screenshot`, `preview_click`) o abriendo `index.html`. Spec de referencia: `docs/superpowers/specs/2026-06-22-clases-curacion-y-easter-egg-design.md`.

**Convención de edición:** las tareas reemplazan **funciones completas** (ancladas por nombre), no por número de línea, porque el archivo se edita activamente y las líneas se desplazan.

---

## Estructura de archivos

- **Modify:** `index.html` — único archivo de código. Todos los cambios ocurren dentro del `<script>` IIFE, salvo una nota en la leyenda HTML de controles (Task 8).
- **Sin archivos nuevos** (el ladrido se sintetiza con WebAudio; no se añade `sonidos/ladrido.mp3`).

### Estado nuevo que se introduce (referencia para todas las tareas)

Campos nuevos en cada `fighter` (se definen todos en Task 1 para mantener consistencia):
`cls, dash, dashCd, dashDir, invuln, windup, healUsed, konami, konamiLast, healFlash`

Variables nuevas de módulo: `selRed, selBlue` (Task 2), `blasts` (Task 5), `eggP, eggPLast, stampede` (Task 7), `actx` (Task 6).

Funciones nuevas: `cycleClass, startMatch` (T2); `cooldownOf, spawnArrow, drawBow` (T3); `tryDodge` (T4); `spawnBomb, explodeBomb, drawBombHand, drawBlast` (T5); `konamiInput, doHeal, barkDog, woof` (T6); `startStampede, updateStampede, drawStampede` (T7).

Tipos de proyectil (`pr.kind`): `'shield'` (el actual), `'arrow'`, `'bomb'`.

---

## Task 1: Descriptor de clases + `makeFighter` recibe clase (sin cambio de comportamiento)

**Files:** Modify `index.html` (zona de constantes y `makeFighter`/`reset`).

- [ ] **Step 1: Añadir el descriptor `CLASSES` justo después de `PAL_BLUE` (línea ~172).**

```js
  const CLASSES = {
    glad:   { name:'GLADIADOR',  kit:'escudo · tira · parry · espada',
              attackType:'melee', hasShield:true,  canThrow:true,  hasParry:true,  hasDodge:false },
    archer: { name:'ARQUERO',    kit:'esquive · parry · flecha',
              attackType:'arrow', hasShield:false, canThrow:false, hasParry:true,  hasDodge:true,
              arrowSpeed:7, arrowWindup:4, arrowCooldown:25,
              dashFrames:8, dashSpeedMult:2.6, dashCooldown:30 },
    bomber: { name:'BOMBARDERO', kit:'escudo · tira · bomba',
              attackType:'bomb',  hasShield:true,  canThrow:true,  hasParry:false, hasDodge:false,
              bombVx:3.2, bombVy:-4.6, bombGravity:0.3, blastRadius:30, bombKnock:10, bombCooldown:45 }
  };
  const CLASS_ORDER = ['glad','archer','bomber'];
```

- [ ] **Step 2: Reemplazar `makeFighter` por la versión con clase y campos nuevos.**

```js
  function makeFighter(o) {
    const cls = o.cls || 'glad';
    return {
      name:o.name, pal:o.pal, x:o.x, dir:o.dir, controls:o.controls,
      cls,
      hits:0, misses:0, swordLvl:1,
      hasShield: CLASSES[cls].hasShield, shielding:false,
      attack:0, resolved:false, cooldown:0,
      parry:0, parryCd:0,
      dash:0, dashCd:0, dashDir:0, invuln:0, windup:0,
      healUsed:false, konami:0, konamiLast:0, healFlash:0,
      slowUntil:0, flash:0, knock:0, msg:'', msgT:0
    };
  }
```

- [ ] **Step 3: Añadir las selecciones de módulo y reemplazar `reset` para que lea la clase elegida.**

Justo antes de `function reset()` añade:
```js
  let selRed = 'glad', selBlue = 'glad';
```
Reemplaza `reset`:
```js
  function reset() {
    p1 = makeFighter({ name:'ROJO', pal:PAL_RED,  x:150, dir:+1, cls:selRed,
      controls:{left:'a', right:'d', attack:'w', shield:'s', thrw:'q', parry:'e'} });
    p2 = makeFighter({ name:'AZUL', pal:PAL_BLUE, x:330, dir:-1, cls:selBlue,
      controls:{left:'j', right:'l', attack:'i', shield:'k', thrw:'o', parry:'u'} });
    projectiles = [];
    state = 'ready'; winner = null;
  }
```

- [ ] **Step 4: Verificar en navegador.**

Run: `preview_start` sobre `index.html`, luego `preview_console_logs`.
Expected: el juego carga sin errores en consola y se juega **idéntico** a antes (ambos gladiadores, espada/escudo/parry funcionan). `selRed`/`selBlue` son `'glad'`.

- [ ] **Step 5: Commit.**

```bash
git add index.html
git commit -m "feat: descriptor CLASSES y makeFighter por clase (sin cambio de comportamiento)"
```

---

## Task 2: Selección de clase en la pantalla de inicio

**Files:** Modify `index.html` (`keydown` handler, `drawReady`, helpers nuevos).

- [ ] **Step 1: Añadir helpers `cycleClass` y `startMatch` (cerca de `reset`).**

```js
  function cycleClass(cur, d) {
    const i = (CLASS_ORDER.indexOf(cur) + d + CLASS_ORDER.length) % CLASS_ORDER.length;
    return CLASS_ORDER[i];
  }
  function startMatch() { reset(); state = 'play'; }   // reset() reconstruye p1/p2 con selRed/selBlue
```

- [ ] **Step 2: Reemplazar el bloque `if (state === 'ready')` dentro de `keydown`.**

Actual:
```js
    if (state === 'ready') { if (GAME_KEYS.has(k)) state = 'play'; return; }
```
Nuevo:
```js
    if (state === 'ready') {
      if (k === 'a' || k === 'd') selRed  = cycleClass(selRed,  k === 'd' ? 1 : -1);
      if (k === 'j' || k === 'l') selBlue = cycleClass(selBlue, k === 'l' ? 1 : -1);
      if (k === ' ' || k === 'enter') startMatch();
      return;
    }
```

- [ ] **Step 3: Asegurar que Espacio/Enter no se filtren como scroll.**

En `GAME_KEYS` ya están `' '` y `'enter'`, así que `e.preventDefault()` ya aplica. Verifica que la línea `const GAME_KEYS = new Set([...,' ','enter']);` siga incluyéndolos (no cambiar).

- [ ] **Step 4: Reemplazar `drawReady` para mostrar la clase elegida por lado.**

```js
  function drawReady() {
    ctx.fillStyle = 'rgba(10,7,3,0.6)'; ctx.fillRect(0,0,W,H);
    ctx.textAlign = 'center';
    ctx.fillStyle = '#e8c66a'; ctx.font = 'bold 21px "Courier New", monospace'; ctx.fillText('⚔ GLADIADORES ⚔', W/2, 40);
    ctx.fillStyle = '#c9b48a'; ctx.font = '8px "Courier New", monospace';
    ctx.fillText('elige clase con tus teclas de mover · perrito con click', W/2, 56);

    // panel
    ctx.fillStyle = 'rgba(20,14,6,0.85)'; roundRect(40, 66, W-80, 150, 8); ctx.fill();
    ctx.strokeStyle = '#6b4a17'; ctx.lineWidth = 1; roundRect(40, 66, W-80, 150, 8); ctx.stroke();

    const sides = [
      { x:W*0.30, sel:selRed,  pal:PAL_RED,  lbl:'ROJO', dog:dogL, keys:'A / D' },
      { x:W*0.70, sel:selBlue, pal:PAL_BLUE, lbl:'AZUL', dog:dogR, keys:'J / L' }
    ];
    for (const s of sides) {
      ctx.fillStyle = s.pal.plume; ctx.font = 'bold 9px "Courier New", monospace'; ctx.fillText(s.lbl, s.x, 84);
      drawDog(s.dog, { x:s.x, y:150, scale:1.8 });
      ctx.fillStyle = '#fff';    ctx.font = 'bold 11px "Courier New", monospace';
      ctx.fillText('‹ ' + CLASSES[s.sel].name + ' ›', s.x, 168);
      ctx.fillStyle = '#c9b48a'; ctx.font = '7.5px "Courier New", monospace';
      ctx.fillText(CLASSES[s.sel].kit, s.x, 182);
      ctx.fillStyle = '#8a734a'; ctx.font = '7px "Courier New", monospace';
      ctx.fillText(s.keys + ' cambia · ' + BREEDS[s.dog.breed], s.x, 196);
    }

    const blink = (Math.floor(performance.now()/450) % 2) === 0;
    if (blink) { ctx.fillStyle = '#fff'; ctx.font = 'bold 11px "Courier New", monospace';
      ctx.fillText('Espacio / Enter para comenzar', W/2, 232); }
  }
```

- [ ] **Step 5: Verificar en navegador.**

Run: recargar preview. En la pantalla de inicio pulsa `A`/`D` y `J`/`L`.
Expected: el nombre de clase de cada lado cicla GLADIADOR → ARQUERO → BOMBARDERO; el click en cada lado sigue cambiando el perrito; `Espacio`/`Enter` inicia la partida. (Arquero/bombardero aún se juegan como gladiador — su comportamiento llega en Tasks 3–5.)

- [ ] **Step 6: Commit.**

```bash
git add index.html
git commit -m "feat: seleccion de clase en pantalla de inicio (teclas de mover + Espacio/Enter)"
```

---

## Task 3: Arquero — ataque de flecha (+ refactor de proyectiles con `kind`)

**Files:** Modify `index.html` (`tryAttack`, `tryThrow`, `resolveAttack`, `updateProjectiles`, `drawProjectile`, `updateFighter`, `drawFighter`; helpers `cooldownOf`, `spawnArrow`, `drawBow`).

- [ ] **Step 1: Añadir `cooldownOf` y `spawnArrow` (cerca de `resolveAttack`).**

```js
  function cooldownOf(f) {
    const C = CLASSES[f.cls];
    return C.attackType === 'arrow' ? C.arrowCooldown
         : C.attackType === 'bomb'  ? C.bombCooldown
         : COOLDOWN;
  }
  function spawnArrow(f) {
    const C = CLASSES[f.cls];
    projectiles.push({ kind:'arrow', x:f.x + f.dir*16, y:GROUND_Y - 34,
      vx:f.dir*C.arrowSpeed, dir:f.dir, owner:f });
  }
```

- [ ] **Step 2: Reemplazar `tryAttack` para ramificar por tipo de ataque.**

```js
  function tryAttack(f) {
    if (f.cooldown > 0 || f.parry > 0 || f.attack > 0) return;
    const C = CLASSES[f.cls];
    if (C.attackType === 'arrow') {
      if (f.dash > 0) return;
      f.windup = C.arrowWindup; f.attack = ATTACK_DUR; f.resolved = false; sfx('swing', 0.5);
      return;
    }
    if (C.attackType === 'bomb') {
      f.attack = ATTACK_DUR; f.resolved = false; sfx('swing', 0.7);
      return;
    }
    if (f.shielding) return;                       // melee: no atacar con escudo arriba
    f.attack = ATTACK_DUR; f.resolved = false; sfx('swing', 0.6);
  }
```

- [ ] **Step 3: Reemplazar `tryThrow` para etiquetar el proyectil y respetar `canThrow`.**

```js
  function tryThrow(f) {
    if (!CLASSES[f.cls].canThrow) return;
    if (!f.hasShield || f.attack > 0 || f.cooldown > 0) return;
    f.hasShield = false; f.shielding = false;
    projectiles.push({ kind:'shield', x: f.x + f.dir*12, y: GROUND_Y - 34, dir: f.dir, owner: f, ang: 0 });
    sfx('swing', 0.8);
    floatMsg(f, '¡ESCUDO!');
  }
```

- [ ] **Step 4: Reemplazar `resolveAttack` para que flecha/bomba generen proyectil y el melee gane el chequeo de esquive (`invuln`).**

```js
  function resolveAttack(att, def) {
    const C = CLASSES[att.cls];
    if (C.attackType === 'arrow') { spawnArrow(att); return; }
    if (C.attackType === 'bomb')  { spawnBomb(att);  return; }   // spawnBomb llega en Task 5
    const reach = REACH_BASE * att.swordLvl;
    const gap = Math.abs(att.x - def.x) - FW;
    if (gap > reach) { registerMiss(att); return; }
    if (def.parry > 0) {
      sfx('block', 0.85); floatMsg(def, '¡PARRY!'); scoreHit(def, att, '¡REGRESADA!');
      def.parry = 0; def.parryCd = PARRY_CD; return;
    }
    if (def.invuln > 0) { registerMiss(att); return; }          // esquivó (dash del arquero)
    if (def.shielding) { sfx('block', 0.8); def.flash = 4; registerMiss(att); floatMsg(def, 'BLOQUEO'); return; }
    sfx('hit', 0.85); scoreHit(att, def);
  }
```

> Nota: `spawnBomb` se referencia aquí pero se define en Task 5. Hasta entonces, el bombardero se sigue jugando como gladiador (no se elige en pruebas de esta tarea). Si quieres aislar Task 3, prueba solo con ARQUERO.

- [ ] **Step 5: Reemplazar `updateProjectiles` para manejar `kind` (`shield`/`arrow`; `bomb` se completa en Task 5).**

```js
  function updateProjectiles() {
    for (let i = projectiles.length - 1; i >= 0; i--) {
      const pr = projectiles[i];
      const foe = (pr.owner === p1) ? p2 : p1;

      if (pr.kind === 'arrow') {
        pr.x += pr.vx;
        if (Math.abs(pr.x - foe.x) <= FW/2 + 3) {
          if (foe.invuln > 0) { /* la atraviesa */ }
          else if (foe.parry > 0) { sfx('block',0.85); floatMsg(foe,'¡PARRY!'); scoreHit(foe, pr.owner, '¡REGRESADA!'); foe.parry=0; foe.parryCd=PARRY_CD; projectiles.splice(i,1); continue; }
          else if (foe.shielding) { sfx('block',0.8); foe.flash=4; registerMiss(pr.owner); floatMsg(foe,'BLOQUEO'); projectiles.splice(i,1); continue; }
          else { sfx('hit',0.8); scoreHit(pr.owner, foe, '¡FLECHAZO!'); projectiles.splice(i,1); continue; }
        }
        if (pr.x < -24 || pr.x > W + 24) projectiles.splice(i,1);
        continue;
      }

      if (pr.kind === 'bomb') {                    // cuerpo real en Task 5
        if (typeof updateBomb === 'function') { updateBomb(pr, foe, i); }
        continue;
      }

      // kind === 'shield' (escudo lanzado)
      pr.x += pr.dir * THROW_SPEED; pr.ang += 0.5;
      if (Math.abs(pr.x - foe.x) <= FW/2 + 3) {
        if (foe.invuln > 0) { projectiles.splice(i, 1); continue; }
        sfx('hit', 0.85); scoreHit(pr.owner, foe, '¡ESCUDAZO!');
        projectiles.splice(i, 1); continue;
      }
      if (pr.x < -24 || pr.x > W + 24) projectiles.splice(i, 1);
    }
  }
```

> El `if (typeof updateBomb === 'function')` es un puente temporal para que Task 3 funcione sin Task 5. En Task 5 se reemplaza por la lógica de bomba inline.

- [ ] **Step 6: Añadir `drawBow` y ramificar el dibujo del arma en `drawFighter`.**

Añade `drawBow`:
```js
  function drawBow(f, hx, hy) {
    ctx.save(); ctx.translate(hx, hy); ctx.scale(f.dir, 1);
    ctx.strokeStyle = '#8a5a1f'; ctx.lineWidth = 2;
    ctx.beginPath(); ctx.arc(2, 0, 12, -Math.PI*0.62, Math.PI*0.62); ctx.stroke();
    const drawn = (f.attack > 0 && f.windup > 0) ? 5 : 1;       // cuerda tensada al cargar
    ctx.strokeStyle = '#efe7cf'; ctx.lineWidth = 1;
    ctx.beginPath();
    ctx.moveTo(2 + 12*Math.cos(-Math.PI*0.62), 12*Math.sin(-Math.PI*0.62));
    ctx.lineTo(2 - drawn, 0);
    ctx.lineTo(2 + 12*Math.cos(Math.PI*0.62), 12*Math.sin(Math.PI*0.62));
    ctx.stroke();
    if (f.attack > 0 && f.windup > 0) { ctx.fillStyle = '#6b4a17'; ctx.fillRect(2 - drawn, -1, 12, 2); }
    ctx.restore();
    px(hx - 2, hy - 1, 4, 4, f.pal.skin);                       // mano
  }
```
En `drawFighter`, localiza la llamada `drawSword(f, handX, handY);` (cerca del final del brazo delantero) y reemplázala por:
```js
    const at = CLASSES[f.cls].attackType;
    if (at === 'arrow') drawBow(f, handX, handY);
    else if (at === 'bomb' && typeof drawBombHand === 'function') drawBombHand(f, handX, handY);
    else drawSword(f, handX, handY);
```

- [ ] **Step 7: Reemplazar `drawProjectile` para dibujar flecha además del escudo (bomba en Task 5).**

```js
  function drawProjectile(pr) {
    const pal = pr.owner.pal;
    if (pr.kind === 'arrow') {
      ctx.save(); ctx.translate(pr.x, pr.y);
      ctx.fillStyle = '#6b4a17'; ctx.fillRect(-7*pr.dir, -1, 14, 2);          // asta
      ctx.fillStyle = '#dadada';
      ctx.beginPath(); ctx.moveTo(7*pr.dir, -3); ctx.lineTo(11*pr.dir, 0); ctx.lineTo(7*pr.dir, 3); ctx.closePath(); ctx.fill();
      ctx.fillStyle = pal.main; ctx.fillRect(-7*pr.dir - (pr.dir>0?0:2), -3, 2, 6);  // plumas
      ctx.restore();
      return;
    }
    if (pr.kind === 'bomb') {
      ctx.save(); ctx.translate(pr.x, pr.y); ctx.rotate(pr.ang || 0);
      ctx.fillStyle = '#1c1c1c'; ctx.beginPath(); ctx.arc(0,0,5,0,Math.PI*2); ctx.fill();
      ctx.fillStyle = '#3a3a3a'; ctx.beginPath(); ctx.arc(-1.5,-1.5,1.6,0,Math.PI*2); ctx.fill();
      ctx.fillStyle = '#caa54a'; ctx.fillRect(-1,-8,2,3);
      ctx.fillStyle = '#ff7a2a'; ctx.fillRect(0,-9,2,2);
      ctx.restore();
      return;
    }
    ctx.save();
    ctx.translate(pr.x, pr.y); ctx.rotate(pr.ang);
    px(-4, -7, 8, 14, pal.metalD);
    px(-3, -6, 6, 12, pal.metal);
    px(-1, -2, 2, 4, pal.main);
    ctx.restore();
  }
```

- [ ] **Step 8: Añadir decremento de `windup` y usar `cooldownOf` en `updateFighter`.**

En `updateFighter`, dentro de `if (state === 'play')`, localiza:
```js
      if (f.attack > 0) {
        f.attack--;
        if (!f.resolved && f.attack <= STRIKE_AT) { resolveAttack(f, foe); f.resolved = true; }
        if (f.attack === 0) f.cooldown = COOLDOWN;
      } else if (f.cooldown > 0) f.cooldown--;
```
Reemplázalo por:
```js
      if (f.attack > 0) {
        f.attack--;
        if (f.windup > 0) f.windup--;
        if (!f.resolved && f.attack <= STRIKE_AT) { resolveAttack(f, foe); f.resolved = true; }
        if (f.attack === 0) f.cooldown = cooldownOf(f);
      } else if (f.cooldown > 0) f.cooldown--;
```

- [ ] **Step 9: Verificar en navegador (ARQUERO).**

Run: recargar preview. Elige ARQUERO de un lado, gladiador del otro; inicia.
Expected: el arquero dispara flechas con `W`/`I` que cruzan a la altura del torso; la flecha conecta (marcador sube, "¡FLECHAZO!"); si el gladiador mantiene escudo la bloquea ("BLOQUEO"); un parry a tiempo la regresa ("¡REGRESADA!"). El arquero todavía bloquea/parrea como antes (su esquive llega en Task 4). Sin errores en consola.

- [ ] **Step 10: Commit.**

```bash
git add index.html
git commit -m "feat: arquero dispara flechas + refactor de proyectiles por kind"
```

---

## Task 4: Arquero — esquive (sello: sin escudo, dash con i-frames)

**Files:** Modify `index.html` (`keydown` dispatch, `tryDodge`, `updateFighter`, `drawFighter`).

- [ ] **Step 1: Añadir `tryDodge` (cerca de `tryParry`).**

```js
  function tryDodge(f) {
    const C = CLASSES[f.cls];
    if (!C.hasDodge) return;
    if (f.dash > 0 || f.dashCd > 0 || f.attack > 0) return;
    f.dash = C.dashFrames; f.invuln = C.dashFrames;
    f.dashDir = down.has(f.controls.right) ? 1 : down.has(f.controls.left) ? -1 : -f.dir;  // sin input: hacia atrás
    sfx('swing', 0.4);
    floatMsg(f, '¡ESQUIVE!');
  }
```

- [ ] **Step 2: Conectar el botón de escudo al esquive en `keydown`.**

En el `forEach` de acciones del `keydown`:
```js
    [p1, p2].forEach(f => {
      if (k === f.controls.attack) tryAttack(f);
      if (k === f.controls.thrw)   tryThrow(f);
      if (k === f.controls.parry)  tryParry(f);
    });
```
Añade la línea del esquive y protege el parry del bombardero (Task 5 lo necesita; lo añadimos ya para no re-tocar):
```js
    [p1, p2].forEach(f => {
      if (k === f.controls.attack) tryAttack(f);
      if (k === f.controls.thrw)   tryThrow(f);
      if (k === f.controls.parry)  tryParry(f);
      if (k === f.controls.shield) tryDodge(f);
    });
```

- [ ] **Step 3: Guardar `tryParry` contra clases sin parry.**

```js
  function tryParry(f) {
    if (!CLASSES[f.cls].hasParry) return;
    if (f.parry > 0 || f.parryCd > 0 || f.attack > 0 || f.cooldown > 0) return;
    f.parry = PARRY_WINDOW;
  }
```

- [ ] **Step 4: Añadir movimiento de dash y decrementos en `updateFighter`.**

Localiza el bloque de caminata:
```js
      if (f.attack === 0 && f.parry === 0) {
        let sp = speedOf(f);
        if (f.shielding) sp *= 0.6;
        if (down.has(f.controls.left))  f.x -= sp;
        if (down.has(f.controls.right)) f.x += sp;
      }
```
Reemplázalo por (añade `&& f.dash === 0` y el bloque de dash):
```js
      if (f.dash > 0) {
        f.x += f.dashDir * speedOf(f) * CLASSES[f.cls].dashSpeedMult;
        f.dash--; if (f.dash === 0) f.dashCd = CLASSES[f.cls].dashCooldown;
      } else if (f.attack === 0 && f.parry === 0) {
        let sp = speedOf(f);
        if (f.shielding) sp *= 0.6;
        if (down.has(f.controls.left))  f.x -= sp;
        if (down.has(f.controls.right)) f.x += sp;
      }
      if (f.invuln > 0) f.invuln--;
      if (f.dashCd > 0) f.dashCd--;
```

- [ ] **Step 5: Señal visual del esquive en `drawFighter`.**

Al inicio de `drawFighter`, justo después de `const C = f.pal;` añade un alpha reducido durante i-frames:
```js
    if (f.invuln > 0) ctx.globalAlpha = 0.45;
```
Y al final de `drawFighter` (antes de cerrar la función) restaura:
```js
    ctx.globalAlpha = 1;
```

- [ ] **Step 6: Verificar en navegador (ARQUERO esquive).**

Run: recargar preview, arquero vs gladiador.
Expected: pulsar `S`/`K` con el arquero hace un dash corto (se ve semitransparente durante ~8 frames) y durante ese lapso las flechas/espadazos/escudazos no le pegan ("¡ESQUIVE!"). El arquero ya **no bloquea** manteniendo escudo (no tiene). Cooldown perceptible entre esquives.

- [ ] **Step 7: Commit.**

```bash
git add index.html
git commit -m "feat: esquive del arquero (dash con i-frames) en el boton de escudo"
```

---

## Task 5: Bombardero — bomba en arco con explosión de área y empuje (sello: sin parry)

**Files:** Modify `index.html` (`spawnBomb`, lógica de bomba en `updateProjectiles`, `explodeBomb`, `blasts`, `drawBombHand`, `drawBlast`, `update`, `render`, `reset`).

- [ ] **Step 1: Añadir `let blasts = [];` junto a las otras variables de módulo (cerca de `projectiles`) y resetearlo en `reset`.**

En `reset`, cambia `projectiles = [];` por:
```js
    projectiles = []; blasts = [];
```

- [ ] **Step 2: Añadir `spawnBomb` y `explodeBomb` (cerca de `spawnArrow`).**

```js
  function spawnBomb(f) {
    const C = CLASSES[f.cls];
    projectiles.push({ kind:'bomb', x:f.x + f.dir*14, y:GROUND_Y - 40,
      vx:f.dir*C.bombVx, vy:C.bombVy, dir:f.dir, owner:f, ang:0 });
  }
  function explodeBomb(pr, foe) {
    const C = CLASSES[pr.owner.cls];
    shakeT = 8; sfx('hit', 0.7);
    if (Math.abs(pr.x - foe.x) <= C.blastRadius && foe.invuln === 0) {
      const dirAway = (foe.x < pr.x) ? -1 : 1;
      if (foe.shielding) { foe.flash = 4; floatMsg(foe, 'BLOQUEO'); }
      else { scoreHit(pr.owner, foe, '¡BOMBA!'); }
      foe.knock = dirAway * C.bombKnock;        // empuje (tras scoreHit, que si no lo pisaría)
    }
    blasts.push({ x:pr.x, y:GROUND_Y - 4, t:14, r:C.blastRadius });
  }
```

- [ ] **Step 3: Reemplazar el puente de bomba en `updateProjectiles` por la física real.**

Localiza el bloque puente:
```js
      if (pr.kind === 'bomb') {                    // cuerpo real en Task 5
        if (typeof updateBomb === 'function') { updateBomb(pr, foe, i); }
        continue;
      }
```
Reemplázalo por:
```js
      if (pr.kind === 'bomb') {
        pr.x += pr.vx; pr.vy += CLASSES[pr.owner.cls].bombGravity; pr.y += pr.vy; pr.ang += 0.3;
        if (pr.y >= GROUND_Y - 6) { explodeBomb(pr, foe); projectiles.splice(i, 1); continue; }
        if (pr.x < -24 || pr.x > W + 24) projectiles.splice(i, 1);
        continue;
      }
```

- [ ] **Step 4: Añadir `drawBombHand` (bomba en la mano del bombardero en reposo) y `drawBlast`.**

```js
  function drawBombHand(f, hx, hy) {
    px(hx - 2, hy - 1, 4, 4, f.pal.skin);                       // mano
    ctx.fillStyle = '#1c1c1c'; ctx.beginPath(); ctx.arc(hx + f.dir*3, hy + 1, 4, 0, Math.PI*2); ctx.fill();
    ctx.fillStyle = '#caa54a'; ctx.fillRect(hx + f.dir*3 - 1, hy - 5, 2, 3);
    ctx.fillStyle = '#ff7a2a'; ctx.fillRect(hx + f.dir*3,     hy - 6, 2, 2);
  }
  function drawBlast(b) {
    const a = b.t / 14, grow = 1.1 - a*0.5;
    ctx.globalAlpha = a;
    ctx.fillStyle = '#ffd27a'; ctx.beginPath(); ctx.arc(b.x, b.y, b.r*grow, 0, Math.PI*2); ctx.fill();
    ctx.fillStyle = '#ff7a2a'; ctx.beginPath(); ctx.arc(b.x, b.y, b.r*0.6*grow, 0, Math.PI*2); ctx.fill();
    ctx.globalAlpha = 1;
  }
```

> `drawBombHand` ya queda enganchada porque en Task 3 el dibujo del arma usa `else if (at === 'bomb' && typeof drawBombHand === 'function')`. Una vez definida, el bombardero la usa automáticamente.

- [ ] **Step 5: Actualizar y dibujar los `blasts`.**

En `update()`, antes de `if (shakeT > 0) shakeT--;`, añade:
```js
    for (let i = blasts.length - 1; i >= 0; i--) { if (--blasts[i].t <= 0) blasts.splice(i, 1); }
```
En `render()`, localiza `projectiles.forEach(drawProjectile);` y justo después añade:
```js
    blasts.forEach(drawBlast);
```

- [ ] **Step 6: Verificar en navegador (BOMBARDERO).**

Run: recargar preview, bombardero vs gladiador.
Expected: `W`/`I` lanza una bomba en arco que cae y explota con un destello de área; si el rival está cerca, recibe golpe + empujón ("¡BOMBA!"); si bloquea con escudo, no recibe golpe pero **sí sale empujado** ("BLOQUEO"); si esquiva (otro arquero) no recibe nada. El bombardero **no tiene parry** (`E`/`U` no hace nada). Cooldown largo entre bombas. Sin errores en consola.

- [ ] **Step 7: Commit.**

```bash
git add index.html
git commit -m "feat: bombardero con bomba de area, empuje y sin parry"
```

---

## Task 6: Curación por Konami + ladrido del perrito

**Files:** Modify `index.html` (`makeDog`, `keydown`, `konamiInput`, `doHeal`, `barkDog`, `woof`, `drawDog`, `drawFighter`, `updateFighter`).

- [ ] **Step 1: Añadir campo `barkT` a `makeDog`.**

```js
  function makeDog(x, dir, pal, msgs, breed) {
    return { x, dir, pal, msgs, breed, msg:'', msgUntil:0, barkT:0, nextAt: performance.now() + 1500 + Math.random()*2500 };
  }
```

- [ ] **Step 2: Añadir el sintetizador de ladrido `woof` (WebAudio) y la variable `actx`.**

Cerca del bloque de Audio (arriba), añade:
```js
  let actx = null;
  function woof() {
    try {
      actx = actx || new (window.AudioContext || window.webkitAudioContext)();
      if (actx.state === 'suspended') actx.resume();
      const t0 = actx.currentTime;
      [[330,0],[185,0.09]].forEach(([fr,dt]) => {
        const o = actx.createOscillator(), g = actx.createGain();
        o.type = 'square';
        o.frequency.setValueAtTime(fr, t0+dt);
        o.frequency.exponentialRampToValueAtTime(fr*0.55, t0+dt+0.08);
        g.gain.setValueAtTime(0.16, t0+dt);
        g.gain.exponentialRampToValueAtTime(0.001, t0+dt+0.12);
        o.connect(g).connect(actx.destination); o.start(t0+dt); o.stop(t0+dt+0.13);
      });
    } catch (e) {}
  }
```

- [ ] **Step 3: Añadir `konamiInput`, `doHeal`, `barkDog`.**

```js
  const KONAMI = f => [f.controls.left, f.controls.left, f.controls.right, f.controls.right, f.controls.shield, f.controls.attack];
  function konamiInput(f, k) {
    if (state !== 'play' || f.healUsed) return;
    const seq = KONAMI(f), now = performance.now();
    if (now - f.konamiLast > 1300) f.konami = 0;
    f.konamiLast = now;
    if (k === seq[f.konami]) {
      f.konami++;
      if (f.konami >= seq.length) { f.konami = 0; doHeal(f); }
    } else {
      f.konami = (k === seq[0]) ? 1 : 0;
    }
  }
  function doHeal(f) {
    f.healUsed = true;
    const foe = (f === p1) ? p2 : p1;
    barkDog(f);
    if (foe.hits > 0) { foe.hits--; f.healFlash = 24; floatMsg(f, '+1 VIDA'); }
    else { floatMsg(f, '¡SANO!'); }
  }
  function barkDog(f) {
    const d = (f === p1) ? dogL : dogR;
    const t = performance.now();
    d.msg = '¡GUAU!'; d.msgUntil = t + 1800; d.nextAt = t + 3200; d.barkT = t + 600;
    woof();
  }
```

- [ ] **Step 4: Llamar a `konamiInput` en `keydown` durante el juego.**

En el `keydown`, dentro del flujo de `state === 'play'` (después de `down.add(k);`), añade antes del `forEach` de acciones:
```js
    [p1, p2].forEach(f => konamiInput(f, k));
```

- [ ] **Step 5: Brinco del perrito al ladrar en `drawDog`.**

En `drawDog`, localiza `const base = (opts.y != null ? opts.y : 252);` y justo después añade un offset de brinco:
```js
    const hop = (performance.now() < d.barkT) ? -Math.abs(Math.sin(performance.now()/40))*6 : 0;
```
Luego en la línea `ctx.translate(cx, base);` cámbiala por:
```js
    ctx.translate(cx, base + hop);
```

- [ ] **Step 6: Destello verde de curación en `drawFighter` + decremento de `healFlash`.**

En `drawFighter`, localiza:
```js
    const tint = f.flash > 0 && (f.flash % 2 === 0);
```
y justo debajo añade:
```js
    const healTint = f.healFlash > 0 && (f.healFlash % 4 < 2);
```
Luego, donde se definen `skin/main/metal`, hazlos respetar el verde:
```js
    const skin  = healTint ? '#9cffb0' : tint ? '#fff' : C.skin;
    const main  = healTint ? '#3fd06a' : tint ? '#fff' : C.main;
    const metal = healTint ? '#9cffb0' : tint ? '#fff' : C.metal;
```
En `updateFighter`, junto a `if (f.flash > 0) f.flash--;` añade:
```js
    if (f.healFlash > 0) f.healFlash--;
```
Y en el `else` de `loop()` (el que decrementa `flash`/`msgT` fuera de `play`) añade también `f.healFlash` para que se apague si la partida termina:
```js
      else { [p1,p2].forEach(f => { if (f.msgT>0) f.msgT--; if (f.flash>0) f.flash--; if (f.healFlash>0) f.healFlash--; }); if (shakeT>0) shakeT--; }
```

- [ ] **Step 7: Verificar en navegador (curación).**

Run: recargar preview, iniciar partida. Con el rival AZUL conéctale 1–2 golpes al ROJO. Luego como ROJO teclea en orden `A A D D S W` (con pausas < 1.3 s).
Expected: el perrito ROJO brinca y muestra "¡GUAU!", suena un ladrido sintetizado, el ROJO destella verde con "+1 VIDA" y el marcador de AZUL baja en 1. Repetir la secuencia **no** vuelve a curar (una vez por partida). En revancha vuelve a estar disponible. Verifica lo mismo para AZUL con `J J L L K I`.

- [ ] **Step 8: Commit.**

```bash
git add index.html
git commit -m "feat: curacion por codigo Konami con ladrido del perrito (WebAudio)"
```

---

## Task 7: Easter egg #3 — invasión de perritos (`p` ×3)

**Files:** Modify `index.html` (`keydown` zona de easter eggs, `startStampede`, `updateStampede`, `drawStampede`, `loop`, `render`).

- [ ] **Step 1: Añadir variables de módulo (junto a `eggUntil`/`eggCount`).**

```js
  let eggP = 0, eggPLast = 0, stampede = [];
```

- [ ] **Step 2: Añadir el trigger `p` ×3 en `keydown` (junto a los eggs de `8` y `g`).**

```js
    if (k === 'p') {
      const now = performance.now();
      eggP = (now - eggPLast < 1300) ? eggP + 1 : 1;
      eggPLast = now;
      if (eggP >= 3) { startStampede(); eggP = 0; }
    }
```

- [ ] **Step 3: Añadir `startStampede`, `updateStampede`, `drawStampede`.**

```js
  function startStampede() {
    stampede = [];
    for (let i = 0; i < 12; i++) {
      const dir = Math.random() < 0.5 ? 1 : -1;
      stampede.push({
        x: dir > 0 ? -20 - Math.random()*220 : W + 20 + Math.random()*220,
        y: 150 + Math.random()*(H - 162),
        dir, sp: 2.2 + Math.random()*1.8,
        breed: (Math.random()*BREEDS.length)|0,
        pal: Math.random() < 0.5 ? PAL_RED : PAL_BLUE,
        bob: Math.random()*6.28, scale: 0.8 + Math.random()*0.5
      });
    }
  }
  function updateStampede() {
    for (let i = stampede.length - 1; i >= 0; i--) {
      const s = stampede[i]; s.x += s.dir * s.sp;
      if (s.x < -40 || s.x > W + 40) stampede.splice(i, 1);
    }
  }
  function drawStampede() {
    const t = performance.now();
    stampede.forEach(s => {
      const yy = s.y + Math.sin(t/90 + s.bob)*2;
      drawDog({ x:s.x, dir:s.dir, pal:s.pal, breed:s.breed }, { x:s.x, y:yy, scale:s.scale });
    });
  }
```

- [ ] **Step 4: Actualizar la estampida en `loop` y dibujarla en `render`.**

En `loop()`, junto a `updateDog(dogL); updateDog(dogR);` añade:
```js
    updateStampede();
```
En `render()`, después de `drawVignette();` y antes de `ctx.restore();` añade:
```js
    drawStampede();
```

- [ ] **Step 5: Verificar en navegador (estampida).**

Run: recargar preview. En cualquier estado teclea `p p p` (rápido, < 1.3 s entre cada una).
Expected: ~12 mini-perritos de razas/colores variados cruzan la arena rebotando y desaparecen por los bordes en ~4 s; no afecta el combate. Confirmar que los easter eggs viejos siguen: `8 8 8` saca la tarjeta de gracias y `g g g` vuelve fantasmas a los perritos de esquina.

- [ ] **Step 6: Commit.**

```bash
git add index.html
git commit -m "feat: easter egg de invasion de perritos (tecla p x3)"
```

---

## Task 8: Leyenda de controles + verificación integral

**Files:** Modify `index.html` (bloque HTML `#rules`/`#controls`).

- [ ] **Step 1: Añadir nota de clases en la leyenda HTML bajo el canvas.**

En el `<div id="rules">`, al final del texto existente, añade una frase:
```html
 · <b>Clases:</b> en la pantalla de inicio cada quien elige Gladiador, Arquero o Bombardero con sus teclas de mover; el botón de <b>escudo</b> y <b>parry</b> cambian de función según la clase.
```

- [ ] **Step 2: Verificación integral (matriz de clases).**

Run: recargar preview y jugar cada combinación al menos una vez (incluye espejos): glad–glad, glad–archer, glad–bomber, archer–archer, archer–bomber, bomber–bomber.
Expected checklist:
- Cada clase ataca y resuelve correctamente contra cada otra.
- Arquero: flecha vuela/bloquea/parry; esquive con i-frames evita golpes; sin escudo.
- Bombardero: bomba en área + empuje; bloquear evita golpe pero no empuje; sin parry.
- Konami cura una vez por jugador por partida y dispara el ladrido; no se gasta sin daño.
- `p`×3 dispara la estampida; `8`×3 y `g`×3 siguen funcionando.
- Revancha reinicia `healUsed` y conserva selección de clase/perrito.
- `preview_console_logs` sin errores.

- [ ] **Step 3: Screenshot de evidencia.**

Run: `preview_screenshot` en pantalla de inicio (mostrando selección de clase) y en combate (arquero vs bombardero con una explosión/flecha en vuelo).

- [ ] **Step 4: Commit.**

```bash
git add index.html
git commit -m "docs: nota de clases en la leyenda de controles + verificacion integral"
```

---

## Self-Review (cobertura del spec)

- **Selección de personaje** → Tasks 1 (descriptor + clase en fighter), 2 (pantalla de inicio). ✓
- **Arquero (base común + un sello)** → Task 3 (flecha + bloqueo/parry/destrucción), Task 4 (esquive con i-frames, sin escudo). ✓
- **Bombardero (sello)** → Task 5 (bomba arco + AoE + empuje, sin parry, bloqueo deja empuje). ✓
- **Gladiador intacto** → default en Task 1; melee sin cambios salvo el chequeo añadido de `invuln`. ✓
- **Curación Konami = recuperar una vida, 1 vez/jugador, con ladrido visible + sonido** → Task 6 (`doHeal` resta `foe.hits`, `healUsed`, `barkDog` brinco+globo, `woof` WebAudio, destello verde). ✓
- **Easter egg invasión de perritos, consistente con los 2 existentes** → Task 7 (`p`×3, cosmético, reusa `drawDog`/`drawBreed`). ✓
- **Arranque pasa a Espacio/Enter** → Task 2. ✓
- **Verificación browser** → pasos de verificación por tarea + Task 8 matriz. ✓

**Consistencia de tipos/nombres:** `pr.kind` ∈ {shield, arrow, bomb} usado igual en `updateProjectiles`/`drawProjectile`/spawns. Campos de fighter definidos todos en Task 1. `cooldownOf`, `spawnArrow`, `spawnBomb`, `explodeBomb`, `tryDodge`, `drawBow`, `drawBombHand`, `drawBlast`, `konamiInput`, `doHeal`, `barkDog`, `woof`, `startStampede`/`updateStampede`/`drawStampede` referenciadas de forma consistente. Puentes temporales (`typeof updateBomb`, `typeof drawBombHand`) se eliminan/quedan inertes al llegar Task 5.

**Notas de balance/visual:** los pixel-art nuevos (arco, bomba, explosión, destello verde, brinco) son primer pase y se afinan jugando; los tunables viven en `CLASSES` para ajuste rápido.
