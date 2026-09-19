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

## Phase 2 — roadmap to a 9+ production build

This section is the handoff document for the next development phase. It records the current quality assessment, the target standard, the implementation order, and the conditions that must be met before Phase 2 is considered complete.

The current build is live at [deuces-interactive-invite.vercel.app](https://deuces-interactive-invite.vercel.app/). The central concept and storytelling are already strong enough for a 9+ product. The remaining gap is primarily execution consistency: Abhi's animation, mobile performance, accessibility, responsive presentation, maintainability, and the repeatability of the character-production pipeline.

### Current scorecard

Overall assessment at the end of Phase 1: **8.1/10**.

| Area | Current | Phase 2 target | What closes the gap |
|---|---:|---:|---|
| Core concept | 9.5 | 9.6 | Preserve the tennis-as-conversation mechanic; avoid adding unrelated features. |
| Originality | 9.3 | 9.5 | Strengthen the authored match presentation and SHAPESHYFT identity. |
| Emotional storytelling | 8.8 | 9.3 | Refine pacing, interaction cues, ending flow and replay behaviour. |
| Visual direction | 8.5 | 9.3 | Match character lighting, shadows, transitions and desktop framing. |
| Character design | 8.3 | 9.2 | Bring Abhi's rig, deformation and movement quality up to Mounika's standard. |
| Character animation | 7.0 | 9.1 | Shared 65-joint rig, planted feet, weight transfer, wrist action and clean transitions. |
| Interaction design | 8.2 | 9.2 | Clear onboarding, progress feedback, pause, replay, skip and alternate input modes. |
| User-flow clarity | 7.7 | 9.2 | Make scroll/tap expectations obvious and prevent users becoming stuck at gates. |
| Copywriting | 8.8 | 9.2 | Retain the specific conversational voice; tighten only where pacing requires it. |
| Sound design | 7.7 | 9.0 | Mix sound layers, improve distance cues and make sound state accessible. |
| Mobile experience | 7.6 | 9.2 | Test real phone sizes, safe areas, browser chrome, touch behaviour and low-power devices. |
| Desktop experience | 7.8 | 9.0 | Use wide space intentionally instead of presenting only a narrow phone-shaped column. |
| Performance | 7.4 | 9.0 | Staged loading, compressed textures/models, adaptive DPR and render-loop pausing. |
| Accessibility | 5.8 | 9.0 | Expose controls to assistive technology, keyboard navigation and a complete reduced-motion path. |
| Sharing | 8.5 | 9.3 | Branded social preview, metadata, platform fallbacks and venue-aware share copy. |
| Branding | 8.2 | 9.2 | Apply SHAPESHYFT consistently but keep the couple as the emotional focus. |
| Technical reliability | 8.0 | 9.3 | WebGL fallback, asset validation, end-to-end flow checks and failure states. |
| Code maintainability | 6.8 | 9.0 | Split the 1,599-line page into modules and eliminate duplicated implementations. |
| Reusability | 8.0 | 9.3 | One commission schema, one animation standard and automated character validation. |
| Commercial potential | 8.6 | 9.3 | Reduce the manual work and uncertainty involved in producing each new commission. |
| Production readiness | 7.8 | 9.2 | Cross-device QA, performance budgets, accessibility gates and documented release checks. |

### Phase 2 priorities

The work should be done in the order below. Later stages depend on the animation and application-state decisions made in the earlier stages.

#### 1. Rebuild Abhi to the Mounika rig standard

This is the largest visible quality improvement and the first major Phase 2 task.

Implementation:

1. Start from Abhi's clean, textured T-pose.
2. Join all character geometry into one coherent mesh before rigging.
3. Remove the current 18-bone armature, Armature modifier and old vertex groups.
4. Rig Abhi with the same 65-joint Mixamo-compatible hierarchy used by Mounika.
5. Match bone names, rest pose, scale, forward axis and coordinate conventions between both characters.
6. Correct shoulder, wrist, hip, knee, ankle, neck and clothing weights manually.
7. Test extreme poses before authoring or retargeting animation: arms overhead, deep knee bend, torso twist, forearm rotation and split-step stance.
8. Retarget a shared motion library rather than maintaining unrelated animation systems.

Required shared clip set:

`Idle` · `Ready` · `SplitStep` · `ShuffleL` · `ShuffleR` · `Run` · `Forehand` · `Backhand` · `Serve` · `Victory` · `Defeat`

Each tennis stroke must contain hip and shoulder rotation, weight transfer, a planted support foot, knee compression, wrist action, follow-through and recovery. Forehand and backhand should take approximately **0.75–0.95 seconds**; serve should take approximately **1.5–1.8 seconds**.

Acceptance criteria:

- No foot sliding during planted phases.
- No vertices follow anatomically unrelated bones.
- No visible snapping between Ready, strokes and recovery.
- Both characters use the same clip names and animation-selection logic.
- Abhi and Mounika look as though they belong to the same animation system.

#### 2. Standardise racket attachment and contact physics

The racket must remain a rigid piece of sporting equipment rather than a skinned part of the character.

Implementation:

- Keep `racket.glb` as a separate rigid mesh with its origin at the grip.
- Remove all racket skin weights and Armature modifiers.
- Parent it directly to the correct hand bone.
- Store a six-value grip profile per character: position XYZ and rotation XYZ.
- Preserve one known local orientation for the shared racket asset.
- Tune the grip in Ready, Forehand, Backhand and Serve before accepting the offset.

Every stroke should define explicit animation events:

```text
takeBack → forwardSwing → contact → followThrough → recovery
```

At the exact `contact` time, the application should change the ball trajectory, play the impact sound, trigger the ball trail, apply restrained racket/camera feedback and advance any score or dialogue state. Contact must be data-driven and measured from the animation clip rather than inferred from a generic percentage.

Acceptance criteria:

- The racket never bends, stretches or disconnects.
- The hand stays aligned with the grip throughout each clip.
- The racket face has a believable orientation at contact.
- Ball direction changes on the exact contact frame at every scroll speed.

#### 3. Formalise the interaction and story state machine

Replace scattered state transitions with one explicit flow:

```text
INTRO
  → RALLY
  → GATE
  → INVITATION
  → VENUE_SELECTION
  → CONFIRMATION
  → MATCH_CARD
```

Implementation:

- Add a short opening instruction: “Scroll to play the point. Tap when the ball waits for you.”
- Hide the instruction after the first successful interaction.
- Support mouse wheel, touch scrolling, arrow keys, Space and an accessible Next action.
- Make every gate recoverable if the user misses the tap cue.
- Add unobtrusive Pause, Replay point, Skip to invitation and Sound controls.
- Preserve the current point and scroll state across orientation changes.
- Make the ending sequence unambiguous: invitation, venue selection, confirmation, match card, share, replay.
- Include the selected venue and date in the final match card and share copy.

Acceptance criteria:

- A first-time user understands the primary interaction within five seconds.
- No user can become permanently stuck at a scroll or tap gate.
- The experience can be completed with touch, mouse or keyboard.
- The story can be replayed without refreshing the page.

#### 4. Complete the accessibility layer

The current `.stage` element is marked `aria-hidden="true"`, which hides post-intro controls and content from assistive technology. Phase 2 must treat accessibility as a release requirement rather than a final patch.

Implementation:

- Remove `aria-hidden` from the interactive application container.
- Apply it only to decorative SVG and WebGL layers.
- Give invitation, venue and match-card overlays appropriate dialog semantics.
- Move focus into newly opened interactive panels and restore it when they close.
- Add a visually hidden live region for the current rally line, score, server, gate prompt, selected venue and final result.
- Add `aria-pressed` and a complete label to the sound control.
- Ensure every control has a visible focus state and a minimum 44×44 px touch target.
- Strengthen the contrast and size of instructional text.
- Make sure colour is never the only indication of state.

Reduced-motion mode must be a complete alternate presentation, not only two disabled CSS animations. It should remove camera pushes, shakes, trails and rapid zooms; shorten character transitions; stop crowd motion; and offer tap-to-advance while preserving every story line and decision.

Acceptance criteria:

- The complete experience is operable with a keyboard only.
- A screen reader can follow the rally and complete the invitation and venue selection.
- The reduced-motion experience contains the complete story.
- Controls remain usable at 200% browser zoom.

#### 5. Improve loading and runtime performance

The current production core is roughly **2.5 MB** before fonts and generated audio: about 500 KB of Three.js, 527 KB for Abhi, 1.2 MB for Mounika and 204 KB for the racket. Phase 2 should improve both transfer size and runtime cost.

Asset work:

- Compress textures with KTX2/Basis where supported.
- Apply Meshopt compression to production GLBs.
- Remove unused nodes, materials, cameras and animation tracks.
- Resample animation curves and eliminate redundant keyframes.
- Quantise vertex attributes where visual quality is unchanged.
- Keep large source GLBs and working assets outside the deployed output.

Loading sequence:

1. Load the HTML, CSS and lightweight court presentation.
2. Show the intro immediately.
3. Preload Three.js and the near character.
4. Load the far character, racket and secondary assets.
5. Enable First serve when the minimum playable scene is ready.
6. Display a themed status such as “Players are warming up…” if loading is visible.

Runtime work:

- Begin at a device pixel ratio of 1–1.5 and increase only when the device sustains the target frame time.
- Pause the render loop and animation mixers while the tab is hidden.
- Stop updating characters that are fully static or offscreen.
- Reduce or disable antialiasing and secondary effects on low-power devices.
- Reuse scene objects rather than allocating them during frames.
- Prefer one WebGL scene/canvas if it can render both players without breaking the SVG registration.
- Monitor frame time and adjust quality automatically.

Performance budgets:

- First meaningful content below **1.5 s** on normal mobile broadband.
- Largest Contentful Paint below **2.5 s**.
- Interaction response below **200 ms**.
- Stable **45–60 FPS** on a representative mid-range phone.
- No blank screen while models download.
- No visible layout shift when 3D content becomes ready.

#### 6. Finish responsive presentation

Mobile remains the primary format, but desktop should feel intentionally composed rather than like unused space around a phone viewport.

Mobile test matrix:

- 320×568
- 360×800
- 390×844
- 412×915
- iOS Safari with browser chrome expanded and collapsed
- Android Chrome on a mid-range device
- narrow/tall foldable viewport

Implementation:

- Respect notch and home-indicator safe areas.
- Keep score, prompts and controls visible when browser chrome changes height.
- Preserve application state during orientation changes.
- Create a landscape arrangement rather than restarting or clipping the court.
- On desktop, retain the vertical court while using the surrounding area for the current rally line, match information or restrained stadium lighting.
- Do not enlarge the court until the characters or perspective become distorted.

#### 7. Perform a coordinated visual and audio polish pass

Visual work:

- Harmonise lighting, colour grading and shadow strength between both character models.
- Add consistent contact shadows under both players.
- Smooth camera transitions with intentional easing curves.
- Keep contact flashes, ball trails and camera impulses brief and restrained.
- Improve crowd depth without creating distracting visual noise.
- Verify that the ball remains legible against every part of the court.

Audio should be separated into controllable layers: crowd ambience, shoes, ball bounce, racket contact, interface/scoreboard, applause and the final confirmation sting. Near-player impacts should sound closer than far-player impacts. Crowd energy should rise toward match point without masking other feedback.

Acceptance criteria:

- No single effect overpowers the story or character action.
- Impact, bounce and visual contact remain synchronised.
- Sound-off mode loses atmosphere but not essential information.
- Audio levels remain comfortable on both phone and desktop speakers.

#### 8. Complete sharing and SHAPESHYFT presentation

The native Share button is the correct primary implementation because it lets the device present any compatible installed application. Retain the copy-link fallback and add explicit desktop fallbacks only where native sharing is unavailable.

Implementation:

- Create a branded 1200×630 social preview image.
- Add Open Graph, X/Twitter and canonical URL metadata.
- Add a site-specific favicon and mobile touch icon.
- Include the selected venue and confirmed date in the final share text.
- Offer WhatsApp, X, Facebook, email and Copy link fallbacks on unsupported desktop browsers.
- Apply `SHAPESHYFT` consistently to the final card, metadata, preview image and documentation without competing with the couple's names.

Suggested base share copy:

> Game. Set. Match. 🎾 Abhi and Mounika are finally taking this match off-screen. See how the point played out.

#### 9. Refactor the application into maintainable modules

`index.html` is currently approximately 1,599 lines, while the 3D, 2D and standalone versions repeat substantial sections. Phase 2 should produce one source of truth without changing the visible experience.

Proposed structure:

```text
index.html
styles/
  base.css
  court.css
  interface.css
src/
  app.js
  config.js
  story-engine.js
  court-renderer.js
  character-scene.js
  animation-controller.js
  ball-physics.js
  audio-controller.js
  accessibility.js
  sharing.js
data/
  deuce-config.js
models/
  abhi.glb
  mounika.glb
  racket.glb
```

Implementation rules:

- Narrative/application state must be independent of SVG and WebGL rendering.
- Animation contact data must live in configuration rather than rendering functions.
- The 3D, 2D fallback and standalone outputs must be produced from shared source and configuration.
- Asset loading and failure handling must be centralised.
- Devices without WebGL must automatically receive the complete 2D experience.
- Preserve the static deployment model unless a genuine server-side requirement appears.

#### 10. Productise the commission workflow

Future builds should be driven by one validated commission file containing:

- Player names and model paths.
- Intro and rally copy.
- Invitation and venue choices.
- Date and time.
- Animation mapping and contact times.
- Character grip offsets.
- Palette and SHAPESHYFT details.
- Share title, copy and preview information.

Create an internal character-audit page or script that reports triangle count, texture sizes, skeleton and bone names, missing required bones, clip names and durations, static versus animated bones, racket attachment, scale/orientation and final download size.

Every new character must pass this production checklist:

1. Clean T-pose.
2. One correctly weighted mesh.
3. Standard 65-joint rig.
4. Shared animation names.
5. Rigid racket attachment.
6. Contact-frame verification.
7. Extreme-pose deformation test.
8. Optimised GLB export.
9. Browser playback test.
10. Mobile performance test.

### Testing and release gates

Add automated checks for:

- Missing production assets or broken paths.
- Required bones and clips in each character GLB.
- Animation contact times falling within their clip durations.
- Complete venue and share-copy configuration.
- WebGL failure correctly activating the 2D fallback.
- Intro-to-match-card completion on mobile and desktop viewports.
- Keyboard completion of every decision.
- No JavaScript errors during the complete flow.

Run manual acceptance testing on at least one iPhone, one mid-range Android phone and one desktop browser before declaring Phase 2 complete.

### Recommended execution plan

| Phase | Scope | Estimated focused effort |
|---|---|---:|
| 2.1 Foundation | Accessibility structure, state machine, clearer controls, WebGL fallback, smoke tests | 2–3 days |
| 2.2 Character rebuild | Re-rig Abhi, shared clips, weight correction, rigid racket and contact timing | 3–5 days |
| 2.3 Motion polish | Ball physics, transitions, camera and audio synchronisation | 2–3 days |
| 2.4 Delivery quality | Asset compression, staged loading, adaptive rendering and responsive layouts | 2–3 days |
| 2.5 Productisation | Module refactor, shared builds, commission schema, validation tools and documentation | 3–4 days |

Expected total: **12–18 focused working days**, with Abhi's mesh and rig quality being the largest uncertainty.

### Definition of Phase 2 complete

Phase 2 is complete only when:

- Abhi and Mounika have equally convincing movement and deformation.
- Both use a consistent skeleton, clip set and animation vocabulary.
- Rackets remain rigid and correctly aligned throughout every stroke.
- Ball contact is synchronised frame-accurately at every scroll speed.
- Touch, mouse, keyboard and screen-reader users can complete the experience.
- Reduced-motion users receive the complete story.
- Mid-range phones maintain the performance budget.
- Desktop, portrait and landscape layouts all feel intentional.
- Shared links have a polished SHAPESHYFT preview.
- Devices without WebGL receive the complete 2D build automatically.
- The three deliverable formats come from shared source rather than duplicated edits.
- A new commission can be configured and validated without rewriting application logic.
- Automated tests complete the primary journey before every deployment.

Expected result after completing these gates: **9.2+ overall**, with every previously weak category at or above 9.0.

### How to resume Phase 2 later

1. Pull the latest `main` branch.
2. Confirm the current production site and both fallback builds still load.
3. Re-read this Phase 2 section and begin with **2.1 Foundation** unless Abhi's replacement 65-joint GLB is already available.
4. If the replacement Abhi model is available, audit its skeleton, weights, clips and scale before changing application code.
5. Keep the existing production build live while Phase 2 is developed on a separate `codex/phase-2` branch.
6. Do not replace the current build until the Definition of Phase 2 complete is satisfied.

---

## Credits

Concept, direction and build by [Abhiteja Charugundla](https://github.com/scarsymmetry899).

Cafés referenced are real Hyderabad venues. Sponsor hoardings are fictional.
