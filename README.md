# Deuce — an interactive tennis invitation

A scroll-driven 3D tennis match that doubles as a way to ask someone out for coffee.

Two people matched on a dating app. Twenty-one days of talking and not one minute in the same room. The rally is the conversation — every shot is a line, and the recipient physically keeps it alive by scrolling. Twice she has to tap to return the ball. At match point he asks the question, and if she says yes she picks the café.

Built as a single self-contained web page. No framework, no build step, no backend.

---

## Try it

Scroll to play. Sound on. Best on a phone.

- `index.html` — the full 3D build
- `deuce-2d.html` — the original SVG-only build (lighter, faster, works everywhere)

---

## How it works

### The court is fake perspective, not a 3D camera

The entire stadium — clay surface, net, sponsor hoardings, grandstands, floodlights, umpire chair, ball kids — is procedurally generated SVG built on a hand-rolled perspective warp:

```js
warp(d) = d(1+F) / (1 + F·d)        // F = 1.7
cy(d)   = NEAR_Y + (FAR_Y - NEAR_Y)·warp(d)
chw(d)  = NEAR_HW + (FAR_HW - NEAR_HW)·warp(d)
csc(d)  = 1 + (0.30 - 1)·warp(d)
```

`d` is depth from the near baseline (0) to the far one (1), `u` is lateral position as a fraction of half-court width. Everything in the scene — crowd rows, hoardings, players, the ball — is placed through those four functions, so the whole thing is internally consistent and resolution-independent.

### The 3D characters are registered to that same maths

Rather than approximating the SVG perspective with a 3D camera, the three.js layer uses an **orthographic camera whose frustum is the SVG viewBox exactly**, including the `preserveAspectRatio="slice"` behaviour. Characters are then placed at `cxAt(d, u)` and scaled by `csc(d)`. They are locked to the court by construction, not by eye.

### Animation is driven by scroll position, not wall clock

This is the part that matters. A swing clip is 0.867 s of fixed animation, but the ball's speed depends on how fast you scroll. Playing the clip on a timer desyncs immediately.

Instead, the swing's playhead is computed from scroll progress:

```
contact time (measured in Blender) ──► fraction of the clip
shot start in scroll space         ──► the frame contact must land on
                                       ──► remap a window around it
```

Contact lands on the exact frame the ball leaves the strings, at any scroll speed. Stop mid-swing and the player holds mid-swing.

| Clip | Duration | Contact |
|---|---|---|
| Forehand | 0.867 s | 0.483 s |
| Backhand | 0.867 s | 0.500 s |
| Serve | 1.617 s | 1.083 s |

### The ball is aimed at the real racket

Each frame, the world position of each player's racket face is read from the 3D scene and fed back into the ball engine. The ball genuinely launches from and lands on the strings rather than near them. Shot *i*'s landing point is shot *i+1*'s launch point, so the trajectory is continuous across every exchange.

### The board is a real dot-matrix

Text on the stadium videoboard is rendered through a **hand-built 5×7 bitmap font** — every glyph defined dot by dot, variable width, drawn as individual lamps with glow and a sweep-on reveal. Rasterising a webfont and thresholding it produced unreadable mush; defining the pixels directly does not.

### Scoring is one point

A tennis rally is a single point, so the score never climbs. Abhi is at **advantage**, serving for the match at 5–4 in the deciding set. Yes wins it. *Ask me again* loses the point and returns to **deuce** — which is the title, and means the rally simply continues.

### Sound is synthesised

No audio files. Racket impacts, ball bounces, shoe squeaks on clay, effort grunts, the umpire's mic chime, crowd murmur and applause are all generated with the Web Audio API.

---

## Project structure

```
index.html              the 3D build
deuce-2d.html           the SVG-only build
three.min.js            tree-shaken three.js subset (~500 KB)
models/
  abhi.glb              near player, 18-bone rig, 9 clips
  mounika.glb           far player, 65-bone Mixamo rig, 9 clips
  racket.glb            rigid racket, grip-origin pivot
textures/
  abhi.jpg              base colour
assets/source-art/      character reference sheets and T-pose renders
```

### Editing the script

Everything that changes between commissions lives in one `CONFIG` block at the top of the main script:

```js
var CONFIG = {
  studio:  { name, url },
  players: { near, far },
  brief, ask,
  board:   { a, b },
  gates:   [18, 22],      // shots she must tap to return
  closeIn: [14, 15],
  rally:   [ /* 24 lines with ball placements */ ],
  venues:  [ /* three cafés with their endings */ ],
  deuce:   { verdict, l1, l2 }
};
```

Rebuilding for a different couple means editing that block. Nothing below it changes.

---

## Character pipeline

Both characters went: **T-pose image → Tripo → Blender → gltf-transform**.

**Tripo** — generate from a single front T-pose image, retopologise to quads at ~10k polys, auto-rig with the Mixamo skeleton preset, export GLB. Order matters: retopo *before* rigging, or the skin weights are discarded.

**Blender** — author nine clips at 60 fps with planted-foot IK baked down to ordinary bone keyframes. Foot rotation channels in every clip, hip translation over the planted foot through both strokes, an athletic Ready stance with a split-step, and dedicated `ShuffleL` / `ShuffleR` instead of a forward run.

**gltf-transform** — resize textures, drop normal and metallic-roughness maps, prune, resample, simplify, quantize. Roughly 7 MB becomes 500 KB with all clips and joints intact.

### Clip set

`Ready` · `Idle` · `ShuffleL` · `ShuffleR` · `Forehand` · `Backhand` · `Serve` · `Victory` · `Defeat`

Names are matched exactly between characters. The code selects by name.

---

## Things learned the hard way

**three.js strips punctuation from node names.** `hand.R` becomes `handR`, `mixamorig:LeftArm` becomes `mixamorigLeftArm`. Dots and colons are deleted, not replaced.

**A tree-shaken three bundle only contains what you export.** Calling a missing class inside a GLTFLoader callback throws as an *unhandled promise rejection*, which `window.onerror` never catches. Add an `unhandledrejection` listener.

**Auto-riggers fail on segmented meshes.** Rigging a model split into parts binds each fragment to a single nearest bone — feet end up weighted to the head. Join the mesh first.

**Never model the racket in the character's hand.** It fuses into the mesh and the auto-weights shear it across the forearm during swings. Model it separately, origin at the grip, and parent it rigidly to the hand bone.

**Procedural animation fails at the legs.** Weight transfer and foot plants are what make motion read as human, and they are exactly what gets lost. Keyframe the feet in every clip or the character looks like a mannequin.

---

## Credits

Concept, direction and build by [Abhiteja Charugundla](https://github.com/scarsymmetry899).

Cafés referenced are real Hyderabad venues. Sponsor hoardings are fictional.
