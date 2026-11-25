# AGENTS.md — Calm Corners

> Instructions for an AI coding agent (Codex, Antigravity, Claude Code, Cursor, etc.).
> Place this file at the root of the project folder, or paste it as the first message.
> Reference implementation (read it before building): https://github.com/lindbda/calm-corners
> — `still-water.html` (WebGL water) and `cozy-cat-room.html` (2D canvas characters) implement everything below.

---

## 1. Objective

Build **Calm Corners**: full-screen, touch-first, calm-down scenes for classrooms (special education, calm corners, smartboards, Chromebooks, tablets). Each scene = **one self-contained HTML file** with a unique theme and a shared set of calm tools. A static hub page (`index.html`) links all scenes. Hosted on GitHub Pages.

When the human gives you a theme, produce a complete scene file, add it to the hub, verify it, and (if asked) publish it.

## 2. Hard constraints (MUST)

1. **One file per scene.** Inline all CSS and JS. No frameworks, no bundler, no build step, no npm.
2. **No external requests** except Google Fonts (`fonts.googleapis.com`, `fonts.gstatic.com`). No images, audio files, CDNs, analytics, or trackers. Draw everything in code (Canvas 2D, WebGL, SVG). Synthesize all sound with the Web Audio API.
3. **No personal data.** No student names, photos, accounts, or network calls. Optional personalization (e.g. naming a pet) is stored only in `localStorage`, every access wrapped in `try/catch`, keys namespaced per scene (`<scene>.<key>`).
4. **Generic and reusable.** Content must suit any classroom. No copyrighted characters or brands.
5. **No failure states.** No scores, levels, countdown pressure, or "wrong" input. Every touch produces a gentle positive response.
6. **Calm by design.** No flashing or strobing (nothing changes brightness faster than ~3 Hz), no sudden loud sounds, slow motion. Honor `prefers-reduced-motion` (slow animations, disable ambient auto-effects).
7. **Works everywhere:** touch + mouse + keyboard; multi-touch; landscape smartboard (1920×1080) down to phone width (~360 px) and short wide windows (~450×340). No horizontal page scroll.
8. **Never crash the loop.** One rendering error must not freeze the animation (see §8).

## 3. File layout

```
/index.html            hub page: one card per scene
/<scene-slug>.html     one file per scene, e.g. dino-valley.html
/.nojekyll             empty; disables Jekyll on GitHub Pages
/README.md             scene list with links
```

Scene slug: lowercase, hyphenated, 2–3 words. `<title>` = the scene's display name (e.g. `Dino Valley`).

## 4. Required page skeleton

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<meta name="description" content="<one sentence>">
<title>Scene Name</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Atkinson+Hyperlegible:wght@400;700&family=Fredoka:wght@400;500;600&display=swap">
<style>
  :root { /* color tokens + --f-display: 'Fredoka'...; --f-body: 'Atkinson Hyperlegible'... */ color-scheme: dark; }
  [hidden]{display:none!important}          /* REQUIRED: display:grid/flex otherwise overrides [hidden] */
  html,body{height:100%}
  body{margin:0; overflow:hidden; user-select:none; -webkit-tap-highlight-color:transparent}
  #stage{position:fixed; inset:0; touch-action:none}
</style>
</head>
<body>
  <div id="stage" role="application" aria-label="<what to do>"><canvas id="scene"></canvas></div>
  <!-- brand (top-left), #timerPill (top-right), breathing panel, #hint, #sheet, #dock, #done -->
  <script>(() => { 'use strict'; /* all code */ })();</script>
</body>
</html>
```

Fonts: **Atkinson Hyperlegible** (body, designed for low vision) + **Fredoka** (display/large words). Always include fallback stacks.

## 5. Required UI (same in every scene)

| Element | id | Behavior |
| --- | --- | --- |
| Brand | `.brand` | Scene name + small subtitle, top-left, `pointer-events:none`, text-shadow for legibility |
| Hint | `#hint` | e.g. "Touch the water" + "Tap, drag, or use both hands. Every note fits." Fades out on first touch; hidden while breathing or a sheet is open |
| Dock | `#dock` | Bottom-center floating toolbar. Buttons (icon SVG + text label, ≥48 px targets, `aria-pressed` for toggles), in this order: **Breathe, Timer, [scene-specific e.g. "My cat"], Rain, [Lamp/optional], Music, Sound, Full screen** |
| Sheet | `#sheet` | One popover above the dock; panes for breathing patterns, timer durations, personalization. Clicking the stage closes it |
| Breathing display | `#breath` | Large phase word ("Breathe in" / "Hold" / "Breathe out" / "Rest"), count (counts UP 1..n), and "Breath N" |
| Timer pill | `#timerPill` | Top-right; SVG progress ring + `m:ss` (tabular numbers) |
| Done dialog | `#done` | `role="dialog"`; "Calm time is done" + "Notice how your body feels now." Buttons: "1 more minute", "I'm ready" (focused) |

Center fixed elements with `left:0; right:0; margin-inline:auto; width:max-content; max-width:calc(100vw - 32px)`. **Do not** use `left:50%; transform:translateX(-50%)` (it halves the available width and wraps the dock early).

Phone (`max-width:600px`): shrink dock buttons (min-width ~44px, 10.5px labels), hide Full screen, stack sheet grids to 1–2 columns.

Respect safe areas: offsets use `calc(env(safe-area-inset-*, 0px) + Npx)`.

## 6. Required behaviors & constants

### 6.1 Musical notes (every touch must sound good)
```js
const SCALE  = [57,60,62,64,67,69,72,74,76,79,81,84];   // MIDI, C-major pentatonic, A3..C6
const CHORDS = [[48,55,62,64],[45,52,60,67],[43,50,57,62],[50,57,64,67]]; // pad voicings
const mtof   = m => 440 * Math.pow(2, (m - 69) / 12);
const noteFor = x => SCALE[Math.max(0, Math.min(SCALE.length-1, Math.floor(x / W * SCALE.length)))]; // left=low
const panFor  = x => x / W * 1.6 - 0.8;
```
Dragging across zones plays a note each time the zone changes (throttle ≥65 ms) → harp glissando.

### 6.2 Breathing patterns
```js
const BREATHS = {
  balloon: [['in',4],['out',5]],                          // default, labeled "Balloon"
  square:  [['in',4],['hold',4],['out',4],['rest',4]],    // "Square"
  sleepy:  [['in',4],['hold',7],['out',8]]                // "4-7-8" (theme name allowed)
};
```
- Visual level = eased 0→1 on `in`, 1 on `hold`, 1→0 on `out`, 0 on `rest` (`ease = 0.5 - 0.5*cos(PI*p)`).
- Drive a scene element with the level (a glowing ring, a character's belly/chest scale ≈ +5%, flowers opening).
- On phase change: soft chime (`in` → MIDI 76, vel 0.2; `out` → MIDI 69, vel 0.18); music pad low-pass → 1600 Hz on `in`, 520 Hz on `out` (`setTargetAtTime`, τ = dur/3).
- Characters close their eyes / settle during breathing; optional purr/hum on `out`.

### 6.3 Calm timer
Durations: 1, 2, 3, 5 minutes. Use `performance.now()` end time (survives tab backgrounding). On finish: chime arpeggio `[72,76,79,84]` 220 ms apart, theme reaction (character sound, hearts/sparkles), show `#done`.

### 6.4 Toggles
- **Rain**: visual rain + noise bed (§7). Fades in/out (~1 s).
- **Music**: ambient pad (§7), default OFF.
- **Sound**: master mute, default ON, persisted.
- **Full screen**: `document.documentElement.requestFullscreen()`; hide button if `!document.fullscreenEnabled`.
- **Wake lock**: `navigator.wakeLock.request('screen')` while breathing or timer is active; release otherwise; re-request on `visibilitychange`; swallow errors.

### 6.5 Keyboard
`B` breathe toggle · `T` timer sheet · `R` rain · `H` music · `M` mute · `F` full screen · `Esc` close sheet/dialog · `Space` (only when focus is on body/stage) = one free interaction at a random spot. Scene-specific extras allowed (e.g. `C` customize, `L` lamp, `P` pet). Ignore all shortcuts while focus is in a text input.

### 6.6 Characters (when the theme has them)
- **Eyes follow** the active finger / moving object (pupil offset + slight head tilt + feature shift for fake 3D). Pupils dilate when excited or in dim light.
- **Idle life**: random blinks (2.5–6 s), ear/tail twitches, slow breathing (scale belly ±5% at ~0.25 Hz), occasional naps with floating "z".
- **Touch response**: happy squint eyes, blush, hearts/sparkles rising from the finger, a sound (purr, hum, chirp). On release: a slow blink.
- Initialize "last touched" timestamps to the past (e.g. `-10`), never `0`.

## 7. Audio engine (Web Audio) — reference code

Create the `AudioContext` on the **first user gesture** (pointerdown or any button); call `ctx.resume()` on later gestures. Graph: sources → `bus` (dry 0.8) + `convolver` (wet 0.4–0.55) → `master` (0.9 or 0 when muted) → `DynamicsCompressor(threshold −16, ratio 3)` → destination.

```js
function makeIR(ctx, sec){            // generated reverb impulse
  const len = Math.floor(ctx.sampleRate*sec), b = ctx.createBuffer(2, len, ctx.sampleRate);
  for (let ch=0; ch<2; ch++){ const d=b.getChannelData(ch); let lp=0;
    for (let i=0;i<len;i++){ const t=i/len; lp += ((Math.random()*2-1)-lp)*(0.2+0.6*(1-t)); d[i]=lp*Math.pow(1-t,3); } }
  return b;
}

function playNote(m, vel, pan){       // soft bell; cap polyphony at ~28 voices
  const t=ctx.currentTime, f=mtof(m), out=ctx.createGain(), p=ctx.createStereoPanner();
  p.pan.value=pan; out.connect(p); p.connect(bus);
  for (const [r,a,dec] of [[1,1,2.6],[2,0.22,1.2],[3.01,0.07,0.6],[4.2,0.03,0.3]]){
    const o=ctx.createOscillator(), g=ctx.createGain(); o.frequency.value=f*r;
    g.gain.setValueAtTime(0.0001,t); g.gain.exponentialRampToValueAtTime(Math.max(2e-4,vel*a*0.3),t+0.006);
    g.gain.exponentialRampToValueAtTime(0.0001,t+dec); o.connect(g); g.connect(out); o.start(t); o.stop(t+dec+0.05);
  }
}
```

| Sound | Recipe |
| --- | --- |
| Water "plip" (optional on taps) | sine, freq ramps f·0.9 → f·2.2 in 70 ms, 110 ms decay, vel·0.12 |
| Music pad | per chord note: triangle (detune −5¢, 0.16) + sine (+6¢, 0.12) → chord gain (fade in 4 s) → low-pass 650 Hz Q 0.4 (LFO 0.07 Hz ±160 Hz) → pad gain 0.24. New chord every 12 s; old chord fades out over 5 s then oscillators stop |
| Purr | sawtooth 25 Hz → low-pass 240 Hz (gain 0.8) + looped white noise → low-pass 380 Hz (gain 0.35) → AM gain (base 0.5, + sine 25 Hz × 0.5 into `gain`) → cadence gain (0.72 + sine 0.55 Hz × 0.28) → purr gain (0–0.7, `setTargetAtTime` τ 0.12) → master |
| Meow | sawtooth → two band-pass filters (Q 5 and Q 7, second at 0.6 gain). Pitch 620→900→780→520 Hz over 0.8 s; F1 700→1200→650 Hz; F2 2100→1700→1000 Hz; vibrato 7 Hz ±14¢; envelope peak 0.9. Multiply all freqs by a per-character pitch (kitten 1.45) |
| Chirp ("mrrp") | same chain, 0.34 s, pitch 480→760 Hz, 34 Hz tremolo (depth 0.35) on output gain |
| Rain | looped white noise → high-pass 450 Hz → low-pass 2.6 kHz → gain 0–0.12; plus random soft high notes (SCALE[6..11], vel 0.05–0.12) every 0.7–2.4 s |
| Click (lamp etc.) | square 1800→600 Hz in 30 ms, gain 0.06 |

All gains change with `setTargetAtTime` or exponential ramps (never instant jumps → clicks).

## 8. Rendering requirements

- `requestAnimationFrame` loop; `dt = min(0.05, elapsed)`. Canvas backing size = CSS size × `min(devicePixelRatio, 2)`.
- **Schedule the next frame first, then guard, then draw:**
```js
function frame(now){
  requestAnimationFrame(frame);
  const dt = Math.min(0.05, (now - last)/1000); last = now;
  if (W < 80 || H < 80) return;              // page not laid out yet (iframes/previews start at 0×0)
  try { step(dt); } catch (e) { console.warn(e); }
}
```
- On `resize` (debounced ~120 ms): recompute layout, repaint static layers to offscreen canvases, re-clamp positions. If the first layout happened at a tiny size, rebuild the scene entirely.
- Static scenery (walls, sky gradient, furniture, rooftops) → offscreen canvas painted once per resize. Per frame: only moving things.
- Never pass negative radii to `arc`/`ellipse`/`arcTo` (throws `IndexSizeError`) — clamp with `Math.max`.
- Keep key characters above the dock: `y = min(desiredY, H - 100)`.
- When a sheet covers the main character, either move the character into view or render a live preview canvas inside the sheet.
- Depth order: sort/partition objects by ground y relative to the main character.
- **WebGL (optional, for water/light effects):** WebGL2, full-screen triangle, R16F height texture uploaded each frame via `texSubImage2D(..., gl.RED, gl.FLOAT, Float32Array)`. CPU wave sim: two `Float32Array` buffers, `next = (L+R+U+D)*0.5 - prev; next *= 0.986`, fixed 90 Hz steps (max 4 per frame), grid ≈ 190–340 cells across. Shader: normals from height gradient → refraction offset, caustics (animated Voronoi edges), specular highlight, vignette. Fall back to Canvas 2D if WebGL2/shader compile fails or on `webglcontextlost`. Adaptive quality: if average frame time > 24 ms over 90 frames, reduce render scale by 0.15 (min 0.5).
- Physics objects (e.g. yarn balls): position-based integration (save prev pos, add gravity, integrate, apply constraints — string length to finger, ground y, walls — then velocity = (pos − prev)/dt; restitution ≈ 0.52 on ground; rolling friction). Play a note on impacts above a threshold, throttled per object (≥80 ms).

## 9. Visual design rules

- One cohesive palette per scene defined as CSS custom properties; deliberately calm (dusk, night, soft pastels). Single dark theme is acceptable; set `color-scheme` and explicit `body` background.
- UI chrome: translucent "glass" panels (`backdrop-filter: blur`), 1 px light borders, warm accent (e.g. lamp gold `#f3c97a`) for active toggles and focus rings (`:focus-visible` outline 2 px).
- Ambient effects (caustics, glows, motes) subtle: tune down until they read as background.
- Starts alive: something gently moving on load; 2–3 silent ambient events in the first 2 seconds.
- Text over the scene: text-shadow or a translucent backing.

## 10. Hub page (`index.html`)

Static page (no canvas). Header: name, one-paragraph lede, fact chips ("Free", "No accounts or logins", "No student data collected", "Smartboards, Chromebooks, tablets"). Grid of cards (`grid-template-columns: repeat(auto-fill, minmax(min(100%,440px),1fr))`). Each card:

```html
<article class="card">
  <a class="art" href="dino-valley.html" aria-label="Open Dino Valley" tabindex="-1"><svg viewBox="0 0 640 340">…inline illustration…</svg></a>
  <div class="body">
    <h2>Dino Valley</h2>
    <p>One sentence on what students do.</p>
    <ul class="tags"><li>Signature interaction</li><li>Breathing guide</li><li>Calm timer</li></ul>
    <div class="actions">
      <a class="btn" href="dino-valley.html">Open Dino Valley</a>
      <button class="btn ghost copy" type="button" data-page="dino-valley.html">Copy link</button>
    </div>
  </div>
</article>
```
Copy link: `navigator.clipboard.writeText(new URL(page, location.href).href)` with a toast; fall back to showing the URL. Footer "For teachers" section: how to share (Clever, Google Classroom), privacy, sound-after-first-tap note, keyboard shortcuts.

Adding a scene = new file + one card + README row. Never rename existing files (links are shared).

## 11. Workflow for each new scene

1. Read the human's theme input (template in §13). Fill any gaps with sensible calm defaults; do not block on questions.
2. Design: palette tokens, signature interaction, character list, how breathing drives the scene, theme sounds (all pentatonic).
3. Write the complete file (expect 1,000–2,000 lines). Reuse the shared UI/audio/breathing/timer code verbatim from an existing scene where possible; change only the theme layer.
4. Verify (§12). Fix and re-verify.
5. Add the hub card + README row.
6. Publish if asked (§14).
7. Report: scene name, what to try first, anything unverified (e.g. "audio not tested in headless browser").

## 12. Acceptance checks (run before reporting done)

- [ ] File opens with **zero console errors** (if you can run a browser — e.g. Playwright/Chromium headless — load the page at 1920×1080, 1280×800, 800×600, 454×837, 454×343 and assert no `pageerror`).
- [ ] First paint is not blank; animation is running (canvas pixels change between two screenshots ~1 s apart).
- [ ] Main characters fully visible above the dock at every size above.
- [ ] Simulated pointerdown on stage: hint gets hidden; something visibly reacts.
- [ ] Breathe → pattern → panel shows "Breathe in" and count advances; Stop works; Esc closes sheets.
- [ ] Timer set to 1 min (or temporarily patch duration) → `#done` appears with focus on "I'm ready".
- [ ] All toggles flip `aria-pressed`; Mute sets master gain to 0.
- [ ] No requests other than Google Fonts (inspect network log).
- [ ] `localStorage` failures don't break the page (wrap in try/catch; test in a context where it throws).
- [ ] `prefers-reduced-motion: reduce` emulation: no auto ambient effects, slower motion.
- [ ] No horizontal scroll at 360 px wide.

## 13. Input template (what the human provides)

```
Theme: <setting, mood, time of day>
Signature interaction: <what students touch and what happens>
Characters: <who lives there, how they react>
Theme sounds: <e.g. soft hums, chirps — all on the pentatonic scale>
Breathing tie-in: <what moves with the breath>
Personalization (optional): <e.g. name the pet, pick its color>
Scene name: <2–3 words>
```

Example themes that fit: Dino Valley (pat a sleepy dinosaur; ferns sway and hum), Night Train (tap windows to light them; train slows on breathe out), Space Station (drift a rocket among planets; stars chime), Puppy Park (toss a ball, belly rubs), Snow Globe (drag to swirl snow that settles on breathe out), Aquarium Wall (feed fish, bubbles rise on breathe out), Garden at Dusk (seeds grow into flowers; fireflies follow the finger), Horse Meadow (brush the mane; soft nicker).

## 14. Publishing (GitHub Pages)

1. Public repository (free Pages requires public; content is generic so this is fine).
2. Settings → Pages → Deploy from branch `main`, folder `/` (or API: `POST /repos/{owner}/{repo}/pages` with `{"source":{"branch":"main","path":"/"}}`).
3. Commit with the account's GitHub **noreply email** (`<id>+<user>@users.noreply.github.com`; id from `gh api user --jq .id`) so no work/school email appears in public history.
4. **If `git push` fails with "Could not connect to server"** (school/district firewalls often block git while allowing the API), upload through the Contents API instead:

```powershell
# For each file: base64 the bytes, include the existing sha when updating
$b64 = [Convert]::ToBase64String([IO.File]::ReadAllBytes($path))
$sha = gh api "repos/$Repo/contents/$file" --jq '.sha' 2>$null     # empty for new files
$body = @{ message="Add $file"; content=$b64; branch='main'; committer=@{name=$user; email=$noreply} }
if ($sha) { $body.sha = $sha }
$body | ConvertTo-Json -Compress | Set-Content tmp.json -Encoding ascii
gh api -X PUT "repos/$Repo/contents/$file" --input tmp.json
```

5. Verify the live URL loads (Pages build takes ~30–90 s: `gh api repos/{owner}/{repo}/pages/builds/latest --jq .status` → `built`).
6. Remind the human: student devices may sit behind a different web filter — test the link on a student device before sharing via Clever/Google Classroom.

## 15. Do not

- Do not add analytics, ads, external images/audio, or any third-party script.
- Do not collect, display, or transmit student information.
- Do not use `alert`/`confirm`/`prompt`.
- Do not use copyrighted characters, logos, or music.
- Do not introduce scoring, competition, or time pressure.
- Do not rename or delete existing scene files.
