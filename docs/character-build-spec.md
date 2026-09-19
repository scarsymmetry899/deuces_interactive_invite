# Mounika — character build spec

Hand this to Tripo and Blender. It encodes everything that went wrong with Abhi so it doesn't repeat.

**Context that changes the priorities:** Mounika is the *far* player. She renders roughly **60 pixels tall**, seen from the **front**. Nobody will ever see her face, her hands, or the detail of her follow-through. What they will see is her **silhouette**, her **stance**, and her **lateral movement**. Spend the effort there and nowhere else.

---

## 1. Source image for Tripo

One image, one character, plain flat background.

- **T-pose.** Arms straight out horizontally, palms down, legs shoulder-width apart with a clear gap at the crotch. Clear air between arms and torso.
- **No racket.** This is the big one. Abhi's racket was fused into the mesh and cost a full Blender separation pass. Model her empty-handed and reuse Abhi's racket asset — it's already separated, rigid, and grip-tuned.
- **Kit:** white top, coral or teal skirt, teal headband, ponytail, white shoes. Avoid hot pink — it fights the terracotta clay and the two colours vibrate against each other.
- Neutral even lighting, no shadows cast on the body, no cropping.

---

## 2. Tripo, in this order

Order matters. Rigging before retopo throws away skin weights.

1. **Generate** from the T-pose image. Check the back view before proceeding — image-to-3D guesses it, and a ponytail often comes out mangled. Regenerate once if it's bad; you can't fix it later.
2. **Retopo** — Quad topology, ~10,000 polygons. Quads give clean edge loops at shoulders, elbows and knees so joints don't crease.
3. **Auto Rig** — Humanoid, **Mixamo skeleton preset**. Non-negotiable: it's what makes Mixamo clips retarget without hand-mapping bones.
4. **Export** GLB, 2K texture. Don't pre-optimise — I compress it and 7 MB reliably becomes ~650 KB.

Skip segmentation entirely if the racket isn't in the image.

---

## 3. Animation — the part that actually failed on Abhi

Abhi's clips were procedurally generated and the legs gave it away. These are the specific fixes.

### Required in every single clip
**`foot.L` and `foot.R` must have real keyframes.** On Abhi they were static in five of seven clips, so the ankles never rolled and nothing read as weight-bearing.

### Weight transfer
During every stroke the **pelvis must translate horizontally over the planted foot**, then drive back through it. Hips lead the stroke; the arm follows. Abhi's strokes happened entirely from the waist up, which is why they looked like a puppet.

### Foot planting
During a stroke the planted foot's **world position must not move** — key it to hold. Sliding feet are the loudest tell of synthetic animation.

### Ready stance
Not standing at attention. Feet **wider than shoulders**, knees bent ~20°, weight on the balls of the feet, torso leaning slightly forward, racket held up across the body with the free hand at the throat. Add a **split-step**: a small two-per-cycle bounce. This single clip does more for believability than any stroke, because she's in it most of the time.

### Lateral movement — replace Run
A forward run cycle is wrong for tennis, and it's why Abhi looked like he was strolling across the court. Replace it with **two side-shuffle clips**, `ShuffleL` and `ShuffleR`: chest square to the net, feet stepping sideways without crossing, weight low. If you must keep a forward run for wide balls, name it `Run` and keep it as well.

---

## 4. Clip list — exact names, exact spellings

I select clips by name in code. A stray `idle` or `Forehand_01` breaks the template.

| Name | Length | Loop | Notes |
|---|---|---|---|
| `Ready` | 1.2 s | yes | athletic stance, split-step bounce |
| `Idle` | 2.5 s | yes | relaxed, between points |
| `ShuffleL` | 0.6 s | yes | side-step to her left |
| `ShuffleR` | 0.6 s | yes | side-step to her right |
| `Forehand` | 0.85 s | no | her right side |
| `Backhand` | 0.85 s | no | two-handed |
| `Victory` | 1.8 s | no | she wins the point |
| `Defeat` | 1.6 s | no | hands on knees — for the Deuce ending |
| `Serve` | 1.6 s | no | optional; she never serves in Deuce, but include it for reuse |

### Tell me the contact frame
For `Forehand`, `Backhand` and `Serve`, **write down the exact time in seconds when the racket meets the ball.** For example: "Forehand, contact at 0.47 s of 0.85 s."

This is the single most useful number you can give me. I drive the swing playhead from scroll position so contact lands exactly when the ball leaves the strings — on Abhi I had to guess it was halfway, and it wasn't.

---

## 5. Export settings

- **GLB**, all clips embedded, one file
- 2K texture is fine — I resize to 1K and drop the normal and metallic-roughness maps, which on Abhi were 4.5 MB of a 7.2 MB file and completely invisible at this scale
- ~60k triangles is fine — I simplify to ~13k
- Bone names with dots are fine; I handle three.js stripping them
- Keep the same scale as Abhi: **1.75 m tall, Y-up, feet at origin**

---

## 6. What to send me

- `mounika.glb` — rigged, all clips, racket-free
- The contact times for the three stroke clips
- Nothing else. The racket is already done.

---

## Quick reference: what went wrong on Abhi

| Problem | Cause | Prevented by |
|---|---|---|
| Racket fused to hand | Racket present in the source image | Model her empty-handed |
| Racket sheared during swings | Auto-weights spread it across the forearm | Rigid racket, parented in code |
| Swings looked wrong | Forehand and Backhand animated the *left* arm | Check the racket-side arm has keys |
| Everything looked stiff | Legs and feet static in five of seven clips | Keyframe feet in every clip |
| Running looked like strolling | Forward run cycle used for lateral movement | `ShuffleL` / `ShuffleR` |
| Swing out of sync with the ball | Contact frame unknown, assumed halfway | State the contact time |
| Character rendered pure white | Embedded texture blocked as a `blob:` URL | Handled in code now |
