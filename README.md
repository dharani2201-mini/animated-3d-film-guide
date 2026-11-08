# Making an animated 3D story film entirely in code

A step-by-step guide to the whole pipeline:
- the characters, their motion and acting;
- the places, the cameras and the lens;
- the look, the captions, the voice, the music and the sound;
- the render, and how to check each stage;
- feedback from Muse, posting to Instagram, and daily engagement (§18–20).

Everything is written as code (three.js in the browser, Python for audio) and rendered deterministically to video. You can rebuild any frame or any sound at any time, and get exactly the same result.

---

## 0. The whole pipeline at a glance

```
  base characters (VRoid Studio .vrm)
        │  recolour, add hair/beard/headwear, dress in code, reshape faces
        ▼
  cast of variants ──► motion library (Mixamo FBX + walk/run mocap), retargeted
        │                     │  + gesture layer (IK) for acting
        ▼                     ▼
  places (terrain, sky, trees, water, interiors, fire)  ──►  shots: place + cast + poses + effects + camera
                                                                 │
  story data (parts → shots → captions, reading-time rule) ─────┤
                                                                 ▼
  lens + painted post-process (depth of field, frame bars, brush, grade)  ──►  frame(t)
                                                                 │
        ┌────────────────────────────────────────────────────────┤
        ▼                                                        ▼
  scene scan (dry pass, every frame) ──► sound design      HyperFrames project per part
  (footsteps, objects, fires, camera)    + score + voice    (captions, cards, fades, audio)
                                              │                  │
                                              └────► render ◄────┘   (one worker, MP4)
                                                      │
                                              join parts with ffmpeg
```

---

## 1. Tools

| Job | Tool | Notes |
|---|---|---|
| Base characters | **VRoid Studio** (free) | Export `.vrm` (VRM 1.0). Make 2 or 3 bases: a young man, a grown man, a woman. |
| 3D in the browser | **three.js 0.181.2** | Pin the version. Load from a CDN with an import map, or bundle it. |
| VRM loading and toon shading | **@pixiv/three-vrm 3.5.5** | Gives the humanoid rig, spring bones, expressions and the MToon toon material. |
| Motion clips | **Mixamo** (free with an Adobe account) | Download each clip as FBX, "Without Skin", 30 fps. |
| Walk, run and idle cycles | Any humanoid mocap `.glb` with Idle/Walk/Run | We used a soldier sample model's three clips. |
| Video composition and render | **HyperFrames** (`npx hyperframes`) | HTML pages rendered frame by frame to MP4. Captions, cards and audio live in it. |
| Headless capture for checking | **Puppeteer** with system Chrome | Opens a page, waits for a ready flag, grabs PNG or JPEG sheets. |
| Sound and music | **Python 3 + numpy + scipy** (+ matplotlib for check sheets) | Use a project virtual environment: `python3 -m venv tools/.venv`. |
| Voice | **Kokoro TTS** (`kokoro-onnx`, model `kokoro-v1.0.onnx` + `voices-v1.0.bin`) | Free, local, on CPU. `npx hyperframes tts` wraps it for English. |
| Joining parts, probing, frames | **ffmpeg / ffprobe** | |
| Local server | `python3 tools/serve.py 8766` (or any static server) | Pages must be served over http, not opened from disk. |

**Machine:** the whole film was made on an 8 GB laptop. That is why it renders one part at a time with one worker, and why every capture takes a shared lock (see §13).

---

## 2. Golden rules (read before writing anything)

1. **Every frame is a pure function of time.**
   - `renderAt(t)` must set everything from `t`: poses, cameras, particles, fire, water, cloth.
   - No `Math.random()`, no `Date.now()`, no `performance.now()`, and no state carried from the last frame.
   - Use seeded hashes, e.g. `hash(n) = fract(sin(n*127.1+311.7)*43758.5453)`, and pre-computed tables.
   - The renderer may ask for frame 900 before frame 10, and it must look the same either way.
2. **Anything that integrates over time is replayed or tabulated.** Spring-bone hair and cloth are re-simulated from a fixed number of seconds before `t` on a fixed step (§5.6). Gaits and fire flicker are functions of `t`.
3. **Python randomness must be seeded too.** Python's built-in `hash()` changes between runs; use `zlib.crc32` for seeds.
4. **Don't render the video after every change.** Check with stills and contact sheets, and render only when a stage is approved.
5. **Check motion over time, not single frames.** A frozen walk cycle looks fine in one still and terrible in motion. Always look at strips of frames (1 per second, or 4 per second for a walk).
6. **One file, one owner.** When several people or agents work at once, each owns its files. Shared files change only through the owner.

---

## 3. Project layout

```
anime3d/                       browser side
  assets/vrm/*.vrm             base characters (+ rigs.js: their bone rest positions)
  assets/mocap/mixamo/mx_*.fbx motion clips
  assets/mocap/src/*.glb       walk/run/idle mocap
  assets/toon/                 engine: VRM loading, costume, faces, motion retargeting, post-process
  assets/toon/ep01/            one film: film.js (the shot engine), places/, parts/, cast, story, lens, fire, acting
  ep01.html                    dev page: one part, renderAt(t), contact sheets
  ep01cast.html                cast comparison page
  ep01acting.html              gesture test bench
ep01/p01 … p17/                one HyperFrames project per part (index.html, compositions/, symlinked assets)
assets/audio/…                 rendered audio (score, voice, final mix per part)
tools/                         Python + Node tools: build, scan, sound, score, capture, render
renders/                       output videos and check sheets
```

A **part** is a 1–3 minute chapter, and a long film is many parts. Each part is its own HyperFrames project because only about 12 characters fit in memory at once. The parts are joined at the end.

---

## 4. Characters

### 4.1 Make the base models

1. In VRoid Studio, make 2–3 base bodies with good proportions, for example a young man, a grown man and a woman.
2. Export them as VRM. Keep the VRoid face. Don't graft another face onto a VRoid body; it looks mismatched.
3. Run a small script that reads each VRM's skeleton (bone rest positions in metres, +Y up, facing +Z) into a JS table (`rigs.js`). Code that places props or IK targets can then know each body's size before the model loads.

### 4.2 Load a VRM with the toon look

- Load the file with `GLTFLoader` and the `VRMLoaderPlugin` from three-vrm.
- Keep the MToon materials, the anime toon shader. Add an outline: an inverted hull, i.e. the mesh again with front faces culled, pushed out along its normals and drawn in a dark ink tinted from the base colour.
- **Hide the VRoid outfit** and dress the body in code (§4.4). VRoid also paints underwear onto the body skin texture; recolour any non-skin pixels to skin.
- **MToon has two colour textures:** `map` (lit) and `shadeMultiplyTexture` (shade). Faces are mostly drawn from the shade texture, so **anything you paint on a face must be painted on both** (§4.5).

### 4.3 Make many characters from a few bases ("guises")

A guise is a small data object describing what changes from a base model. One loader builds any guise:

```js
GUISES.minister = {
  base: 'man',                       // which .vrm
  skin: 0xe6b890, hair: 0x8a847c, brow: 0x6a645c, iris: 0x2c1a10,
  beard: { len: 0.3, thick: 0.06 }, mustache: { style: 'droop', thick: 0.3 },
  turban: { color: 0xf2ead8, gem: 0xc0262f },
  attire: 'epic', shirt: 0x5a3a7a, dhoti: 0xf2ead8, border: 0xb3242c,   // dress (§4.4)
  face: 'calm', scale: 1.0, bulk: 1.0,
};
```

How each change is made:
- **Recolouring keeps the painted shading.** For skin, hair, brows and irises, scale every texture pixel by its brightness relative to the texture's median tone, so that tone lands on the target colour. For white or grey hair, also flatten the contrast a little.
- **Measure the face once, in the rest pose,** from the face mesh: the eye height, the brow height, the nose tip (the most forward point near the centre), the mouth and chin, the face width and the jaw width. Ray casts against the face mesh find surface points.
- **Beards, moustaches, braids and locks** are tubes swept along curves, sized from the face measurements. **Turbans and headwear** are lofted shapes. Each piece is built in the world rest pose, then parented to the **head bone**, so it turns with the head.
- **Marks** (lines of age, ash or a forehead mark) are painted onto the face skin texture at ray-cast UV positions.
- **Build:** scale the bones (shoulders, chest) for a broader or slighter figure, and scale the whole holder for height.

### 4.4 Dress the body in code

Every garment is a **skinned mesh made from the body's own skin mesh**, so it moves exactly with the body:
- **Fitted garments** (a tunic, a wrapped lower garment, a blouse): take the body vertices in a height band and push them out along their normals by an "ease". The ease grows toward the hem, so cloth hangs looser lower down. Keep the body's skin weights.
- **Skirts and draped cloth:** a loft (rings of points) around the hips and legs, weighted between the hips and thighs, using the convex envelope of the two legs so it isn't a figure eight.
- **Sashes, stoles, cords and necklaces:** curves lying on the torso surface, swept into tubes or flat ribbons and weighted from the skin underneath.
- **A long hanging sash end:** a flat cloth panel, not a squashed tube. Keep each point just outside the garment under it, so it never sinks in.
- **Armlets, bangles and anklets:** rings sized to the limb where they sit.
- **Fabric:** a canvas texture of silk with small woven motifs in a staggered grid and a gold woven border at the hem. Cotton is plainer.
- **Gold that reads as metal in toon shading:**
  - a painted matcap with hard bands (a light sky band, a dark horizon band, a warm bounce, a bright rim);
  - a rim term;
  - reflections that take the light's colour.
- **Gems:** an unlit shader with dark facet edges, a hot core where a facet faces the eye, and sharp glints.

### 4.5 Making faces read as natural (not anime)

VRoid faces have huge eyes with sparkles, almost no nose, a tiny mouth, a fringe over the forehead and a pointed chin. To make a natural, Indian-looking cartoon face, do all of this **in the rest pose, before measuring the face**:

1. **Eyes.** The visible eye opening is a **hole cut in the skin texture**, not the eyeball mesh.
   - Warp the **skin vertices around each eye** toward the eye's centre: about 0.84–0.9 across and 0.72–0.86 tall at the opening, easing to no change at about 1.9× the eye's radius. This makes the opening smaller and almond-shaped.
   - Move the eyeball, eyeline and highlight vertices the same way, and scale their **blend-shape deltas** by the same factors, so blinks still close fully.
2. **Irises:** repaint each iris on the iris texture as a round natural iris, with a gradient, fine radial streaks, a pupil and a soft shadow from the upper lid.
3. **Highlights:** clear the star sparkles and paint one small soft catchlight.
4. **Brows:** a little fuller, and lowered toward the smaller eyes.
5. **Nose:**
   - sculpt the skin a little forward along the centre line, from between the brows to the tip;
   - **paint** a thin bridge shadow, a soft curve under the tip and two small nostrils; toon shading flattens a sculpted nose seen from the front.
6. **Lips:** VRoid's mouth is a **separate mesh layered over the skin**. Paint the lips on that mesh's texture too, not only on the skin.
7. **Paint mode:** paint with the canvas `multiply` mode on **both** the lit and shade textures, so the colours sit in the skin's own tone.
8. **Fringe:** delete the hair triangles in front of the forehead (the fringe or bangs) so the forehead and hairline show.
9. **Jaw:** below the mouth, widen toward the chin and draw the chin up a little. This rounds the anime point.
10. **Head size:** scale the head bone to about 0.86–0.9; anime heads are large for the body.

Then give the guise fair skin, black hair (grey or white only with age), dark brown eyes, jewellery (bell earrings, a nose ring, necklace strands, a waist chain, ear studs) and painted forehead marks.

**Check faces at three distances:** a straight-on close-up, a portrait, and the full figure. Compare old and new side by side.

### 4.6 Expressions and eyes

- **Expressions:** named blends of the VRM expression presets, e.g. `kind = { relaxed: 0.5, happy: 0.08 }` or `grieve = { sad: 0.8, blink: 0.3 }`. Relaxed expressions lower the eyelids, so watch for sleepy eyes on a reshaped face.
- **Blinks:** seeded every 2.5–5 s, each 0.16 s long, sometimes a double.
- **Look-at:** turn the neck and head partly toward a target (the eyes do the rest), clamped to about ±0.9 rad of yaw and ±0.5 rad of pitch.

### 4.7 Props and headwear

- **Hand props:** mount each to a hand bone with a fixed grip transform, the prop's +Y along the grip.
- **Crowns:** build them from the measured head band, so they sit where the hair is and fit every head.
- **Flowing parts** (hair, long cloth) use the VRM spring bones (§5.6).

---

## 5. Motion

### 5.1 The motion library

From Mixamo, download one FBX per clip ("Without Skin", 30 fps, keep the root motion) on the same Mixamo skeleton, for example:
- prayer standing and kneeling, a formal bow, kneeling down;
- falling back, being hit, a big reaction, a high block;
- bow and arrow (take up, draw, aim, loose), one- and two-handed clubs, swords;
- throw, charge, a battle cry, a fighting stance.

Load them all with `FBXLoader` as one source:
- take the first file's skeleton;
- rename each clip to its key;
- add a `TPose` clip made from the skeleton's rest, since Mixamo's rest is the T-pose.

Mark which clips **loop**. A looping clip with `loop` unset clamps at its last frame: the stride freezes and the figure **glides** across the ground. This was the cause of characters "flying instead of walking".

### 5.2 Retargeting onto VRM

- **Rotations:** for each frame, compute each bone's **world-space rotation relative to the T-pose**, then apply that change to the same VRM bone in its normalized (rest = identity) rig. This is proportion-independent.
- **Hip travel:** scale it by the ratio of the hip heights.
- **Ground contact:** bake the clip's **lowest joint height** per frame relative to its T-pose. At runtime, set the figure's height so its own lowest joint matches. Feet and kneeling knees then touch the floor instead of floating or sinking.
- **Pace:** store each cycle's stride speed (travel ÷ contacts), so walks can be driven at the speed they were captured at.

### 5.3 Walking along a path

`walk(path, u, …)` moves a figure along a curve:
- a natural pace (about 1.15–1.3 m/s walking, up to about 2 m/s hurrying);
- a trapezoid speed profile with an ease in and out;
- the clip time set from the distance travelled ÷ stride speed, so the feet never slide;
- a crossfade into **Idle** as the figure stops;
- an optional end yaw to turn and face someone.

### 5.4 Crossfades

A pose may carry `mix: { clip, t | off, w }`: a second clip blended in at weight `w`, slerped per bone (matched by name across different retargets) and lerped at the hips. Use it for walk → stand, stand → kneel and kneel → rise.

### 5.5 Sitting and other poses the library lacks

Write procedural poses as functions, for example sitting: the hips dropped onto a seat, thighs forward, shins down, hands on the knees or raised in a gesture.

### 5.6 Hair and cloth (spring bones), deterministically

Before posing at `t`:
1. Pose the figure at `t − 2.5 s`.
2. Reset the springs.
3. Step the springs at a fixed **60 Hz** grid up to `t`, posing at each step.

If the previous frame was just before `t`, continue from it instead. The result depends only on `t`.

### 5.7 Creatures from code

Build animals with spine functions instead of rigs:
- a snake-like body is a chain of points along `slither(len, amp, t)`, `coil(radius, turns, t)` or `strike(t)`, skinned along a tube;
- four-legged animals use per-leg phase cycles;
- wings flap on a cycle.

Tag each object with `userData.sound = { kind }` so the sound pass (§12) can find it.

### 5.8 Checking motion

- **Side-on strips:** 4–8 frames a second across a walk. Look for gliding (a planted foot moving) and for floating or sinking feet.
- **Plots:** foot and toe height over time against the floor; contacts should be clean dips to the floor.
- **Audit:** every figure's lowest joint against the floor, flagging anything above about 5 cm (except figures meant to fly or float).

---

## 6. Acting (the gesture layer)

The motion library is mostly fighting and prayer, and storytelling needs more. Add a **gesture layer** applied after the clip:
- **Placement:** two-bone IK places each hand at a point defined in the figure's own chest frame (its forward, left and up), scaled to its shoulder width. A gesture then fits any body and follows a lean or a walk.
- **Hand shape:** fingers curl by a 0–1 amount, and the thumb has its own curl.
- **Wrist turns:** where the gesture needs them (joined palms, a raised palm), turn the wrists by orienting the hand bone to a target basis: fingers up and forward, palms facing.

The vocabulary:
- **Arms:**
  - point;
  - reach, give, take, offer (cupped hands), lift (both hands up);
  - cradle (a child);
  - a hand on the heart, a raised hand for a vow, a raised palm for stop or refuse, a raised palm in blessing;
  - plead (both hands open, leaning in);
  - grieve (the face in the hands, head bowed);
  - joined palms, open arms;
  - a hand to the chin;
  - talk (one hand moving with the words);
  - clenched fists;
  - beckon.
- **Head:** nod (n times), shake, bow the head, turn away.
- **Stance:** an "upright" stance is laid over the walk cycle's idle (which is a soldier's knees-bent guard) at the idle's weight: straight legs, the chest lifted.

```js
stand(pos, yaw, { t, face: 'kind', look: () => head('other'),
  act: [act('talk', lf, 0.4, 3.6), act('halt', lf, 3.4, 6.8), act('nod', lf, 9.0, 10.2, { n: 1 })] })
```

`act(name, now, t0, t1, { in, out, k, at, side })` returns the gesture's weight: it eases in from `t0`, holds, and eases out before `t1`.

**Directing rules:**
1. **Act every caption.** Whoever speaks, asks, promises, refuses or agrees gestures as they do it. Whoever listens reacts: a nod, a turn away, a hand on the heart, a lean. Physical actions in the story are shown: giving, taking, kneeling, rising, holding something.
2. **Nobody stays frozen** for more than about 3 s in a medium or close shot.
3. **Fighting clips only for fighting** or archery. Never use a block to mean "stop", or a charge to mean "a vow".
4. **Eyelines:** characters look at whom they address.
5. **Timing:** ease gestures over 0.3–0.6 s. Hold a strong gesture 2–5 s. Don't flick between many.
6. **Elders and royalty** move slower and less; children and animals more.
7. **Check with a 1-frame-per-second strip of the whole part,** captions visible. Each caption's action should be visible while it reads.

---

## 7. Places

Each place is a module: `build(env)` returns `{ group, look, update(t, camera, opts) }`.

| Element | How |
|---|---|
| **Terrain** | A height grid from layered noise, with flat circles where people stand, and a fade at the edges. |
| **Sky** | A dome shader: horizon band, sun disc with corona, heaped clouds (noise, lit on the sun side, silver edges, shaded undersides), high streaks, drifting with time. Presets: day, golden hour, sunset, night, storm. |
| **Horizon** | Rings of land beyond the playable area: continuing ground, woods, hills, then far mountain ranges, each paler with distance and misty at its foot. Without this, a wide shot ends in fog. |
| **Trees** | A packed leafy core plus alpha-cut leaf-cluster cards, lit with rounded normals and varied in colour. A shader sways leaves with the wind, and their shadows sway too. Trunks have root flare and bark. |
| **Grass and ground** | Instanced tufts near the camera, pebbles, twigs, footprints. |
| **Water** | A shader with noise ripples, sky-colour reflections, sun glints and a depth tint. Rings and a foam collar form round anything standing in it (found automatically), and a boat leaves a V-shaped wake. |
| **Interiors** | The building's shell casts the sun's shadow, so light enters only through openings. Add soft light shafts with dust, pools of light under lamps, smoke banks under the roof, and banners that move. |
| **Crowds** | Instanced simple figures, seeded poses, a gentle sway. |
| **Weathering** | Procedural grime at the base, rain streaks, soot and moss (world-space noise), so no two pillars match. Seeded jitter of repeated pieces. |
| **Fire** | See below. |

**Fire** (one module used everywhere):
- **Flames:** a few camera-facing planes with domain-warped 3D noise scrolling upward, a silhouette that narrows with height and tears into tongues, and a black-body colour ramp (white-yellow core, orange, red edges). Blending is additive with a soft body; tone mapping and depth writes are off.
- **Fuel:** charred logs with glowing cracks, and a bed of coals.
- **Smoke:** billboards that rise, widen and drift with the wind.
- **Embers:** streak quads that cool as they rise.
- **Light:** a flickering point light driven by the same noise as the flames.
- **Kinds:** pyre, hearth, brazier, torch, lamp.
- `burst(t0)` throws sparks and a flare from time `t0`, stored as data so the output stays a function of `t`.
- `userData.heat` feeds the lens's heat haze (§8).

---

## 8. Camera and lens

### 8.1 Lens maths

- **Focal length to field of view** on a 24 mm-high frame: `fov = 2·atan(12 / mm)`.
- **Distance for a framing:** `d = size / 2 / tan(fov/2)`, where `size` is how much of the figure fills the frame's height:

| Framing | Height shown | Default lens | f-stop |
|---|---|---|---|
| extreme close-up (eyes) | 0.20 m | 100 mm | 1.8 |
| close-up (face) | 0.42 m | 100 mm | 2 |
| medium close-up (chest up) | 0.78 m | 75 mm | 2.8 |
| medium shot (waist up) | 1.25 m | 40 mm | 4 |
| full shot | 2.4 m | 40 mm | 5.6 |
| wide shot | 4.6 m | 40 mm | 8 |

### 8.2 The shot grammar (camera rigs)

Write cameras as functions of the shot's local time, built from a small set of rigs:
- `on(figure, { size, angle, height, mm, push, third, aim })`: a framing by size, from any angle round the figure; `push` dollies in over the shot.
- `ots(a, b)`: over a's shoulder onto b, with a's shoulder soft in the foreground.
- `two(a, b)`: a two-shot.
- `wide(centre, …)`: a far camera, slow lateral drift, small figures.
- `aerial(centre, { r, r1, h, h1, a0, a1, bank })`: an arc that banks into the turn, like a helicopter.
- `insert(point, …)`: a close-up on an object.
- `track(figure, …)`: alongside a walker at their pace, handheld.
- `vertigo(figure, …)`: dolly out while zooming in, keeping the figure the same size.

**One caption can hold several setups:** `cuts(localTime, [[0, setupA], [4.8, setupB], …])`. Cut on a story beat, at least about 3.5 s apart and not within 1 s of a shot's ends.

**Camera body:**
- by default, `still` (a tripod);
- `hand: 0.008–0.012` for handheld: a sum of seeded sines for sway, plus a look jitter;
- `shake` for impacts.

**Framing rules:**
- keep eyes above the caption band (place the eyes at about 40% down the frame);
- never put the camera inside geometry;
- for crowned or tall figures, aim the look point lower so the headwear isn't cut.

### 8.3 Depth of field (post-process)

1. **Depth pass:** render a depth texture with **only solid objects**. Hide sprites, points and transparent things that don't write depth (smoke, glows, light shafts) during the pass. Otherwise a puff of smoke in front reads as near and blurs the whole frame.
2. **Blur size** per pixel (thin lens): `CoC_px = (f² / (N·(F−f))) / 0.024 · H · |d − F| / d`, capped at about 18 px.
3. **Gather** over about 28 golden-angle samples. A sample counts if its own blur reaches this pixel, and sharp things behind never bleed onto a blurred foreground.
4. **Focus point:** focus on the **face surface** (between the eyes, a little in front), not the head bone. The "eyes" point sits inside the head, which is enough to miss focus on a long lens.
5. **Focus-puller cap:** limit the blur so a head (±8 cm about the focus) never blurs more than about 1.5 px, even on extreme close-ups.

### 8.4 Frame format and heat haze

- **Format:** by default, 2.20:1 bars top and bottom (a bar of `(1 − W/2.2/H)/2` of the height each). Open to the full 16:9 frame for big wides, aerials and revelations, about 15–20% of the setups. Keep the caption band just above the bottom bar.
- **Heat haze:** over each fire on screen, displace the picture in a column with scrolling noise, strongest just above the flames.
- **Shadows:** fit the shadow camera to the setup's start and end views, so it holds still through the move.

---

## 9. The look (post-processing chain)

Composer order:

```
render → god rays (sun shafts from a mask render) → bloom → output/tone map
      → depth of field → painterly brush (Kuwahara filter, radius by depth) → colour grade → paper texture → SMAA → frame bars + heat haze
```

- **Grade:** slightly lowered contrast, light diffusion, warm highlights over cool shadows, a light vignette, a touch of chromatic aberration, and film grain seeded by time.
- **Per-place looks:** fog colour and range, key light colour, intensity and direction, sky light, exposure, bloom, height fog. A shot may override the place's look, for example for a night room.
- **Characters and props** use the same toon shading and ink outlines, so everything sits in one world.

---

## 10. Story, timing and captions

1. **The story file** holds the data: parts, then shots, then captions. Each shot has a name, a minimum length, and captions with the time they should start.
2. **Reading time sets the timing:**
   - each caption needs `words / 2.7 + 1.1` seconds;
   - the next caption starts at least 0.3 s after the previous ends;
   - a shot lasts at least its last caption's end + 0.6 s.

   The music and sound read the same timeline, so nothing drifts.
3. **Cards:** titles, chapter cards, a question, a curse or oath, the end card. Each shows over a softly lit, smoky backdrop.
4. **Each part's shot file** gives, per shot: the place, the cast, `act` (poses at time t), `fx` (effects and creatures) and `cam` (the camera at t).
5. **HyperFrames projects:** generate one per part:
   - `index.html`: the 3D canvas, driven by the shot engine;
   - `compositions/captions.html`: timed caption divs;
   - `compositions/cards.html`;
   - `compositions/fades.html`;
   - an `<audio>` element with the part's final mix.
6. **Captions:** keep them in a fixed band (in a 1920×1080 frame, roughly y 858–968, x 230–1690), on a soft dark gradient for contrast.

---

## 11. Voice (narration)

- **Engine:** Kokoro TTS, local and free:
  ```bash
  pip install kokoro-onnx soundfile
  ```
  It uses the model `kokoro-v1.0.onnx` and the voices file `voices-v1.0.bin`. English voices include `bm_george` (a deep British male narrator).
- **English through HyperFrames:** `npx hyperframes tts` reads lines into WAV files.
- **Other languages:** if the built-in phonemizer is weak (Japanese dropped vowels), write your own text→IPA converter and pass phonemes straight to Kokoro. Write breath pauses as spaces.
- **Timing:** time each line's audio, then place captions or subtitles to match. Keep one JSON with each line's text, start, end and subtitle.
- **Under the voice:** duck the music under every line.
- **Without a voice:** captions carry the story, and the music and sound design carry the mood. Our final film was made this way.
- **Optional:** transcribe the rendered voice (`npx hyperframes transcribe`) to align word-level captions.

---

## 12. Music and sound design (all synthesized)

### 12.1 The score

- **Instruments,** written in numpy:
  - a drone (tanpura-like plucks with a delayed harmonic bloom);
  - pads and a choir (additive voices);
  - a bamboo flute (a sine with breath noise, vibrato and glides);
  - bells (inharmonic partials), plucked strings (a decaying harmonic stack);
  - frame drums, taiko-like hits, a gong, a conch.
- **Scales:** use modal scales that suit the culture of your story.
- **Moods:** each shot gets a list of layers (drone, chant, heartbeat, drums, flute, bells, choir, plucks), each fading in and out across the shot's edges, so shots blend.
- **Reverb:** a hall reverb on the music.

### 12.2 Scan the picture

A **dry render mode** poses, moves and aims everything for time t without drawing. A scan steps through the whole part at 30 fps and writes JSON:
- the camera position, rotation and lens per frame, and every cut;
- every figure's floor height and key joints (hips, head, knees, feet, toes, hands);
- every tagged object (animals, a boat, creatures) and its position;
- every fire with its strength, and every burst;
- at water, how deep the camera and each figure are.

### 12.3 The mixer hears the picture by itself

- **Footsteps:**
  - **Detecting a step:** a foot lands when its height (the lower of foot and toe) drops to within about 4.5 cm of its usual floor and its horizontal speed is under about 0.55 m/s.
  - **The sound:** a heel strike, then the ball of the foot 45–80 ms later, or a single strike when running.
  - **Ground:** each place has a ground: stone, wet stone, wood, dirt, grass, leaves, sand, water (splash, bubbles), marble, cloud, dust or ash.
  - **Character extras:** shod characters add a wooden sandal knock; women add small anklet bells.
  - **Filtering:** merge two steps within 0.12 s into one. A foot shifting under a still body is a soft scuff, not a step.
- **Body:** kneels, body falls (a heavy thump, a bounce, settling cloth), cloth rustle driven by how fast the hands and hips move, and air swishes when a hand moves faster than about 3.6 m/s.
- **Animals and objects:** paws and hooves spaced by stride, a slither (only along the ground, never while falling or lifted), wingbeats, and a boat's hull slap with occasional pole dips.
- **Fires:** a roar plus crackles and hiss, sized by kind and levelled. Every burst adds a whoomph and sparks.
- **Place air (beds):**
  - river: water wash, babble, lapping;
  - forest: leaves, birds, insects by time of day;
  - halls and courts: room tone, a murmuring crowd, banners;
  - high places: wind;
  - an ocean's swells;
  - drips in a dark space; chimes in a bright one.
- **Placing every sound by the camera:**
  - gain falls off with distance (a reference of about 1.6 m);
  - pan from the left-right angle;
  - a lowpass for distance;
  - about 25% louder on screen, quieter behind;
  - each place's room is a convolution reverb (RT60 from 0.25 s outdoors to 3.6 s in a great hall).
- **Cuts** change the perspective, as the picture does.

### 12.4 Story cues

For each part, a cue sheet lists, per shot, `(time in shot, kind, {params, at: figure | figure.bone | object})`. Use it for the moments the picture can't tell by itself: a curse taking hold, thunder, magic, a bowstring, an egg cracking, a door, a gasp. It also holds:
- **hushes**, which pull the air and bodies down for a held breath;
- **music dips**;
- **extra beds**.

A cue may define its own small synth function when the library has nothing that fits.

### 12.5 Levels and checking

- **Levelling:** level every spot effect to a reference loudness, with a cap on its plain energy; otherwise bass-heavy sounds come in hot. Cut infrasound below 30 Hz.
- **Music:** under everything at about −24 LUFS, ducked under loud moments.
- **Master:** each part normalized to **−16 LUFS** integrated, then a look-ahead limiter at −1 dB.
- **Check sheet** per part:
  - a spectrogram with the shots marked;
  - each bus's level over time (air, bodies, effects, music);
  - every event placed;
  - a figure's foot heights with the detected steps.

  You can't listen to everything, but you can read this.

---

## 13. Render

```bash
# per part project
npx hyperframes check --json        # lint, runtime, layout, contrast
npx hyperframes snapshot --at 12,40 # stills to look at
npx hyperframes render --quality delivery --workers 1 -o renders/part01.mp4

# join the parts (same codec: copy video, re-encode audio across joins)
ffmpeg -f concat -safe 0 -i list.txt -c:v copy -c:a aac -b:a 192k -movflags +faststart film.mp4
```

- **One worker on an 8 GB machine.** A 2-minute part at 1080p, 30 fps renders in about 6–10 minutes.
- **Shared lock:** if several tools capture or mix at once, take a lock (a lock directory, taken over if older than ~12 minutes) so only one browser or mix runs at a time.
- **After changes:** re-run the scene scan and the sound mix whenever a part's motion or timing changes, then re-render.

---

## 14. Checking (before every render)

| Check | How |
|---|---|
| **Every setup framed well** | A contact sheet at each camera setup's midpoint, with the caption band outlined: faces clear of captions, no figure cut by the bars, no camera inside a wall, focus on the face. |
| **Motion over time** | 1-frame-per-second strips of whole parts; 4–8 fps strips of walks. No frozen figures, no gliding, no floating. |
| **Acting** | Each caption's action visible while it reads. |
| **Faces** | Straight-on close-ups, old against new. |
| **Focus** | A probe: the focus distance against the distance to each head. |
| **Sound** | The sound check sheet per part. Mix length equals part length. LUFS and peak printed. |
| **Project** | `hyperframes check` passes with no errors. |
| **Final** | Frame sheets from the rendered MP4 (`ffmpeg -vf fps=1,tile=…`) and loudness (`ffmpeg -af ebur128`). |

---

## 15. Building from a reference picture (measured method)

When you need something to match a picture (a character, a prop):
1. **One reference.** It sets the look; rebuild it, never import it.
2. **Measure it:** height in heads, shoulder width, where the hands are, garment lengths as shares of the height, and colours sampled from the picture.
3. **Write a checklist** of 10–12 numbered lines with numbers in them. Put "never" next to the default you are fighting, e.g. "never big round anime eyes".
4. **The still loop:** build, render a still posed like the picture, stack *reference | still | close-up*, and grade every line yes / partial / no.
5. **Stop after round 2** and show it before going further.

---

## 16. Working with several AI agents (optional)

- Give each agent a narrow brief:
  - the goal;
  - the files it owns;
  - the files it must not touch;
  - the tools it can run;
  - how to verify;
  - what to report.
- **Shared engine files** belong to one person; agents report problems in them instead of editing.
- **Read the reports critically** and check their pictures and sheets yourself. Fix shared problems at the source (the engine), not in every part.

---

## 17. Pitfalls we hit (and the fix)

| Symptom | Cause | Fix |
|---|---|---|
| Characters glide or "fly" while walking | A looping clip without its loop flag clamps at its last frame | Set loop on cycles; bake ground contact |
| Feet float or sink, knees in the floor | Hip height not matched to the clip | Per-frame lowest-joint contact |
| Whole frames blurry with depth of field | Smoke and sprites wrote depth | Depth pass with solid objects only |
| Close-ups out of focus | Focus on the head bone (inside the head) | Focus on the face surface; cap blur within ±8 cm |
| Painted face marks invisible | MToon draws from `shadeMultiplyTexture` | Paint both textures (multiply) |
| Painted lips invisible | The mouth is a separate mesh over the skin | Paint the mouth mesh's texture too |
| Shrinking the eyeballs doesn't shrink the eyes | The opening is a hole in the skin | Warp the skin vertices round the eye |
| Hands cross instead of palms joining | IK places wrists, not their turn | Orient the hand bones to a target basis |
| Night birds in daylight | A default "night" variant used for every place | Only places with a time of day use it; mark night-lit rooms per shot |
| Sound different every render | Python `hash()` is salted per run | `zlib.crc32` seeds |
| NaN audio | A swept filter's cutoff went below 0 Hz | Clamp cutoffs (≥ 10–80 Hz) |
| Thunder or rumble far too loud or too quiet | Infrasound counted in energy but not heard | High-pass 30 Hz; level by K-weighted loudness, capped by energy |
| Fires drown the scene | Unlevelled fire beds | Level fires by size before distance |
| Footstep storms in a fighting stance | Feet shuffling in place | A step needs body or foot travel; shuffles become sparse scuffs |
| Glow dots, flat gold | Toon shading flattened metal | A matcap gold with hard bands and a rim |
| Overexposed check stills | A bright backdrop palette and too much light | Neutral grey-brown backdrop, moderate light |

---

## 18. Getting feedback from Muse

**Muse** ([muse.ai](https://muse.ai)) is an AI agent connected to your Instagram account, and it can read the account's real insights. Use it as an outside reviewer:
- before posting, to judge the hook and the opening;
- after posting, to read retention numbers that Instagram's website doesn't show.

### 18.1 Setup
1. Sign in at muse.ai and connect the Instagram account (it must be a professional account: Creator or Business).
2. Keep **one long thread** for the account, so Muse remembers earlier audits and reels.
3. To let Claude drive it, open the thread in Claude's built-in browser. Sign in yourself; never hand over a password.

### 18.2 How to send a question (when Claude drives it)
1. Click the **Message** box and type the prompt. Newlines are fine; they don't send.
2. Click the **Send** button by its element reference (`find "Send"`), not by screen position. A click by position missed once and nothing was sent.
3. Wait **1–3 minutes**. The tab title shows "(1)" when the reply lands.
4. Read the reply with `get_page_text` and take the last assistant message. Screenshots are too small for long replies.

### 18.3 What to ask (prompts that worked)
- **Audit:**

  > "Audit my Instagram @name using its real insights. For EACH reel give a table: views, average watch time, skip rate (first 3 s), % watched to the end, replays, likes, comments, saves, shares, follows. Then: which hook types keep people past 3 s, where viewers drop off, best CTA, best posting time, top countries/ages. Concrete numbers, no generic advice."

- **Opening review before posting:** describe frame 1 exactly (every text layer, what's on screen, how long the first spoken line is) and the full script. Then ask for:

  > "1) a hook text of max 6–7 words, 2) a spoken first line under 2.5 s, 3) the ONE thing frame 1 should show, 4) a short caption (2 lines + max 5 hashtags). Keep every fact accurate. Give 2 options and pick one."

- **24-hour recheck:** "Recheck reel X at its 24-hour mark and give the short verdict." Muse can schedule this itself.
- **Research:** "Which stories from the last 3 days are getting the most engagement, with source links? Which accounts in my niche are growing fastest, and what do their first 2 seconds look like?"

### 18.4 Rules for using its answers
- **Muse answers are long, so ask for numbered outputs** with word limits.
- **Muse's per-reel saves show 0, which is wrong.** Take saves from Instagram's own insights page (`instagram.com/insights/media/<media id>/`).
- **Muse can't return a per-second retention graph.** Average watch time against the reel's length is the nearest measure.
- **Check every fact Muse suggests against the original source** before it goes in a script; its hook suggestions sometimes misstate facts.
- **Judge a reel at 24–48 hours,** not the same evening.
- **Muse is one opinion.** Where its advice and your own numbers disagree, your numbers win. Write down which one you followed and why.

---

## 19. Posting to Instagram

Export an upload copy first. The browser extension's upload limit is 10 MB, so use a two-pass H.264 encode at a bitrate that lands under 10 MB, 1080×1920, AAC audio:

```bash
ffmpeg -y -i reel.mp4 -c:v libx264 -b:v 2150k -pass 1 -an -f null /dev/null
ffmpeg -y -i reel.mp4 -c:v libx264 -b:v 2150k -pass 2 -c:a aac -b:a 128k reel-upload.mp4
```

For a ~30 s reel, ~2150 kbit/s gives about 9 MB. For longer reels, lower the bitrate: size ≈ (video + audio bitrate) × seconds ÷ 8.

There are three ways to post, from the most hands-on to fully automatic.

### 19.1 By hand (phone app)
- **This is the only way to pin a comment.** Instagram's website and the API can't pin.
- Post, then immediately post and pin the prepared comment, and reply to the first comments within the first hour.

### 19.2 Assisted, through the browser (Claude in Chrome or the built-in browser)
1. You sign in to instagram.com yourself.
2. Claude goes through: **Create → Post** → upload the file → OK on the "video posts are now shared as reels" notice (it doesn't always appear) → crop **9:16** → **Next** (cover, trim) → **Next**.
3. Claude types the caption into "Add a caption…".
4. **Claude stops and asks you before clicking Share.** Nothing is published without your explicit yes. Sharing takes about 30 s.
5. **Scheduling also works on the web.** On the last screen, turn on **Schedule content**, set the hours and minutes (24-hour, local time), and click **Schedule**. Check it at `instagram.com/scheduled_content/`.
6. The **"Add AI label"** toggle on the last screen is your call; decide it per reel.
7. Click through the dialogs by element reference, not screen position, because the window can resize mid-run.

### 19.3 Fully automated: Meta's official Content Publishing API
This is the sanctioned way to publish from code, with no browser.

**What you need:**
- a **Creator or Business** Instagram account linked to a **Facebook Page**;
- a Meta developer app with the **instagram_content_publish** permission;
- a long-lived access token, kept in an environment variable or a secrets file, never in the code or the repo.

**The flow** (Graph API):
1. **Host the MP4 at a public HTTPS URL.** Instagram fetches it from there, e.g. a temporary S3 link or your own server.
2. **Create a container:**
   ```
   POST /{ig-user-id}/media
     media_type=REELS
     video_url=<public mp4 url>
     caption=<caption text>
     share_to_feed=true
   ```
3. **Poll** `GET /{container-id}?fields=status_code` until it says `FINISHED`. It says `ERROR` if the file is wrong.
4. **Publish:** `POST /{ig-user-id}/media_publish?creation_id={container-id}`.
5. **Optional:** post the prepared comment with `POST /{media-id}/comments?message=…`. You still pin it from the phone.

**Rules:**
- **There is a daily cap** on API-published posts per account. Check your remaining quota with `GET /{ig-user-id}/content_publishing_limit` before a batch.
- **To schedule from code,** run the script at the posting time (a cron job or scheduled task). Kallaway avoids *third-party* schedulers (rumoured to be demoted); native scheduling and the official API are the safe routes.
- **Keep the same file rules:** H.264 + AAC MP4, 9:16, with the caption and hashtags in the caption field.
- **Log every post:** the media id, the time, and which reel file it was. You need these later for reading insights.

**No-code alternative:** Meta Business Suite (business.facebook.com) can upload and schedule reels for weeks ahead. It's official and safe, the same as in-app scheduling.

---

## 20. Daily engagement (do this by hand)

**What not to do:** don't run a bot that scrolls, likes, follows or comments on your behalf, and don't try to make one look human.
- **It's against Instagram's terms,** and automated activity gets action-blocked, shadow-limited or banned.
- **Disguising a bot as human is exactly what its detection is built to catch,** and the account is what you lose.
- **It doesn't grow the account anyway.** Reach comes from how the ~200 strangers in each reel's first test group react to *your reel*, not from your account liking other posts.
- **Bought or automated engagement makes reach worse.** It fills your audience with the wrong people, and the next reel's test group reacts badly.

**What works instead** (common creator-growth advice, e.g. Kallaway on YouTube), about 20–30 minutes a day:
1. **Warming up a new account** (first 2 days, before posting): use the app like a normal viewer for ~15 minutes a day in your niche. Search niche terms, watch good videos to the end, like and save the relevant ones, scroll past unrelated ones. This trains your own feed and tells Instagram what the account is about.
2. **The first hour after posting:**
   - post and pin the prepared comment;
   - reply to every comment (each reply is another touchpoint);
   - answer DMs.
3. **"Barnacle on a whale":** turn on post notifications for 10–20 bigger creators in your niche. Be one of their first 5 comments, with something specific and useful, not "great post". Their audience reads it and finds you.
4. **Trial Reels** (once the account has ~200 followers): test 2–4 hooks on the same reel. Trial Reels go to non-followers only; post the winner normally.
5. **Stories:** reshare a reel to Stories after 24 hours, once its first push has run out. Repost a reel that did well 4–6 months later.
6. **Post daily at a steady time.** Gap days cut daily viewers sharply.
7. **Track the right numbers:** shares, average watch time, follows per reel (shares > comments > saves > likes > views). Use Muse (§18) for skip rate and watch time.
