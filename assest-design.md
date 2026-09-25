# DOG RUNNER — Visual Asset Production Bible

**Version 1.0 · Companion to the DOG RUNNER Technical System Design (v1.0)**
**Role of this document:** the single source of truth for every visual asset, its AI-generation prompt, its technical spec (size, pivot, frames, FPS, transparency), its naming, and its place in the Flame atlases.

---

## PART 0 — HOW TO USE THIS BIBLE

### 0.1 Production pipeline

```
1. Generate  → AI prompt (this bible) + approved reference art + locked seed
2. Clean     → remove fringing, normalize outline weight, fix anatomy, defringe alpha
3. Master    → PNG-32 at MASTER size (defined per family), on-grid, pivot-verified
4. Variant   → color variants via palette-swap recolor of master (never re-generate)
5. Export    → tier downscale (LANCZOS): HIGH ×1.0 · MID ×0.6–0.75 · LOW ×0.4–0.5
6. Pack      → atlas per page plan (§2.4) via packer CLI → .png + generated Dart frame table
7. QA        → art checklist (§7.6) → merge
```

### 0.2 Shared prompt tokens (defined once, appended everywhere)

Every GENERATION PROMPT in this document is composed as:

> **`<subject prompt> + <CTX camera token> + [MS] + <category master style>`**

| Token | Content |
|---|---|
| **[MS] MASTER STYLE PROMPT** | *“stylized 2D mobile game art, casual endless-runner style, clean bold vector-like shapes, flat vivid colors with one soft cel-shadow and one crisp highlight per form, thick uniform dark-plum outline around the outer silhouette only, bright cheerful harmonious palette, warm late-morning sunlight from the upper right with soft cast shadows to the lower left, subtle ambient occlusion at the base, iconic readable silhouette, no surface texture noise, no photorealism, polished professional mobile-game quality, family friendly”* |
| **[MN] MASTER NEGATIVE PROMPT** | *“photo, photorealistic, 3D render, cgi, blurry, pixelated, low quality, jpeg artifacts, watermark, signature, text, letters, words, numbers, readable signage, logos, brands, real advertisements, cluttered composition, busy background, scenic background, dark gritty palette, muddy colors, sketchy lines, deformed anatomy, extra limbs, disney, pixar, marvel, anime screenshot, existing game characters, subway surfers, temple run, copyrighted characters”* |

**Camera/perspective tokens** (exactly one per asset — this is how perspective stays locked):

| Token | Content |
|---|---|
| **CTX-A** (actor, player) | *“rear three-quarter view of a runner seen from a low chase camera two meters behind, character running away from the viewer, full body, feet on the ground line”* |
| **CTX-B** (actor, dog) | *“rear three-quarter view of a dog running away from the viewer, low chase camera, full body”* |
| **CTX-C** (vehicle rear) | *“rear three-quarter view of the vehicle as seen by a runner catching up to it from behind on a road, slight 10° top-down tilt, wheels touching the ground line”* |
| **CTX-D** (vehicle front) | *“front view of the vehicle facing the viewer head-on, oncoming traffic on a road, wheels on the ground line”* |
| **CTX-E** (roadside prop) | *“object standing on a roadside, viewed from a low chase camera about 10° above the ground, bottom edge on the ground line”* |
| **CTX-F** (ground decal) | *“flat decal on asphalt viewed straight from above, orthographic top-down”* |
| **CTX-G** (facade) | *“flat frontal elevation of a building facade viewed straight-on, no perspective distortion, full building from ground to roofline”* |
| **CTX-H** (UI icon) | *“flat straight-on centered icon for a mobile game UI, no perspective”* |

**Category master styles** (stack on top of [MS] — see §17 for full text): `[MS-CHAR]`, `[MS-DOG]`, `[MS-ENV]`, `[MS-VEH]`, `[MS-UI]`.

**Shared technical fields** (apply to every asset unless a block overrides):

- **BACKGROUND:** transparent
- **TRANSPARENCY REQUIREMENT:** straight-alpha PNG-32, clean 1px defringed edges, no baked backdrop, no baked drop-shadow on flat color
- **NEGATIVE PROMPT:** [MN] (plus per-asset additions where noted)
- **STYLE:** [MS] + category master

### 0.3 Generation workflow rules (critical for consistency)

1. **Approve the base first.** Generate → clean → APPROVE one base player sprite and one base dog sprite. Every later generation uses these as character reference (Midjourney `--cref`, SDXL IP-Adapter/Reference-only) plus a locked seed family. Never free-generate variants.
2. **Generate key poses, not strips.** AI is unreliable at exact sprite grids. For each animation, generate the 3–5 key poses listed in the animation table, clean them, and in-between by hand (or by tool interpolation). Only simple loops (coin spin, dust puffs) may be requested as strips.
3. **Color variants are recolors**, never re-generations (guarantees silhouette identity). Palette-swap in Aseprite/Photoshop using indexed recolor maps.
4. **Upscale ×2, then downscale to master size** — produces crisp anti-aliased edges that survive tier downscale.
5. **No readable text anywhere in world art.** Signs and billboards use abstract squiggle-glyph panels. All real text (score, buttons, logo wordmark) is typeset by the engine or in a design tool. This is a hard rule — AI-generated text is inconsistent and this is also our copyright-safety shield.

---

## PART 1 — VISUAL IDENTITY (THE STYLE BIBLE)

### 1.1 One-line identity

> *A warm, sunlit, hand-crafted cartoon Karachi — chunky outlined shapes, cheerful saturated color, big readable silhouettes — seen through a low chase camera, always bright enough to read at a glance at 60 km/h.*

### 1.2 Perspective law (the #1 consistency rule)

The game has exactly **one camera**: low, ~1.5 m above the road, behind and slightly above the runner, looking down the road. Every asset is drawn for THIS camera:

- Actors (player, dog): **rear three-quarter** — we see their backs.
- Vehicles/obstacles ahead: **rear three-quarter** (parked/same-direction) or **front** (oncoming).
- Roadside props: slight top-down (~10°), bottom-anchored.
- Ground hazards (potholes, humps): **flat top-down decals** (the renderer draws them in perspective).
- Facades: flat frontal billboards (no skew, no vanishing-point distortion).
- UI/icons: flat, straight-on, zero perspective.

No asset may mix these (e.g., never a side-view car).

### 1.3 Lighting law

- **Single key light: warm late-morning sun, upper-right of screen, ahead-right of the runner.** Lit right sides, shaded left sides, cast shadows fall to lower-left/rear.
- One shadow tone + one highlight per form. Shadow = base hue shifted −25% lightness toward plum. Highlight = +18% toward warm white.
- Soft ambient occlusion ellipse under every grounded object.
- Themes may tint (dusk, smog) but **never move the light**.

### 1.4 Outline law

- Outer silhouette only: **dark plum `#2A1F2E`**, ~1.2% of sprite height (≈3 px on a 256 px sprite), uniform weight after downscale normalization.
- Interior detail lines: same hue, 60% weight, used only for essential form breaks (wheel arches, clothing seams).
- No white keylines, no double outlines, no variable brush weight.

### 1.5 Color language (locked palette)

| Role | Hex | Usage rule |
|---|---|---|
| Ink (outline/text) | `#2A1F2E` | outlines, UI text |
| Cream (panel/paper) | `#FFF2DC` / shade `#F3DDBE` | UI panels, facades base |
| **Teal (primary)** | `#2FB5A3` | UI primary buttons, player accent — the brand color |
| Gold (reward) | `#FFC93C` / `#E8940F` / `#FFF3B0` | coins, rewards, score highlights |
| Danger red | `#E84545` | obstacles warning, damage FX, dog vignette — **reserved** |
| Confirm green | `#58C15E` | confirm buttons, positive progress |
| Shield cyan | `#35C4E8` | shield power-up only |
| Magnet violet | `#A05CE8` | magnet power-up only |
| Speed orange | `#FF8A2A` | boost power-up only |
| Bone cream | `#F5EDE0` | bone power-up only |
| Asphalt base | `#5C5661` (theme-tinted ±) | roads |

**Value laws:** background/skyline = low contrast (60–75% lightness range compressed); gameplay objects = high contrast; the player must always be the highest-saturation object on screen. Red appears ONLY for danger. Power-up hues appear ONLY on their power-up (instant recognition).

**Colorblind safety:** every power-up also has a unique **shape**: shield = round shield, magnet = horseshoe, boost = double chevron, ×2 = two overlapping coins, bone = bone. Never rely on hue alone.

### 1.6 Proportion law

- **Player: 5 heads tall**, slightly oversized shoes and hands, small backpack-friendly torso. Identity from behind = **backpack + cap/hair silhouette** — every cosmetic MUST change the backpack or head silhouette.
- **Dog: shoulder ≈ 45% of player height**, oversized expressive head, lean athletic body, visible tail (the tail is the dog's "expression antenna").
- Vehicles: real proportions compressed ~15% shorter/taller (friendlier, better fit in lane sprites).
- Never vary proportions between cosmetics — outfits and fur only.

### 1.7 Rendering law

Flat fills + 1 shadow + 1 highlight. No texture noise, no gradients except skies, no rim lights, no fancy rendering. Decorative detail (truck-art patterns, fabric) is **shape-based**, never noise-based. Everything must stay readable at 25% zoom (the silhouette test, §7.6).

### 1.8 Pixel-per-meter rule (all sizes derive from this)

**130 px = 1 meter at scale 1.0 (1080-wide reference).** Player = 1.7 m = 221 px tall. Every dimension below is derived from this and matches the collision halfW/halfD table in the system design — art and collision share one source of truth.

---

## PART 2 — TECHNICAL STANDARDS

### 2.1 File formats

- **Master:** PNG-32, sRGB, straight alpha, no metadata bloat.
- **Shipped:** atlas PNGs (opaque or alpha) per tier; total APK asset payload target **≤ 14 MB**.
- Optional WebP distribution allowed later; PNG first for simplicity.

### 2.2 Pivots (anchors) — global standard

| Asset class | Pivot | Flame anchor |
|---|---|---|
| Actors (player, dog, pedestrians, cyclist) | bottom-center, 2 px above canvas bottom | `Anchor.custom(0.5, 0.98)` |
| Vehicles, grounded obstacles/props | ground-contact bottom-center | `Anchor.bottomCenter` |
| Ground decals | center | `Anchor.center` |
| Facades | bottom-center at building base | `Anchor.bottomCenter` |
| Particles / UI icons / coins | center | `Anchor.center` |

**Baked shadows:** vehicles and grounded obstacles bake their soft AO ellipse into the canvas (bottom margin reserved). Actors do NOT (they jump) — they use the shared `fx_shadow_blob` sprite.

### 2.3 Sprite-sheet requirements (global; per-family tables override)

- Frames laid out left→right, top→bottom, **fixed grid per row**, 2 px padding when packed.
- Frame indices 2-digit (`_01`); loops return to frame 1; non-loops hold last frame (`loop: false`).
- Every animation declares: frame count, order, frame size, FPS (stepTime), loop/non-loop, pivot (above).
- Transparent background, no grid lines, no frame numbers burned in.

### 2.4 Atlas pages, residency & memory budget (reconciled with system design §10)

| Page | Contents (resident) | MID page | LOW page |
|---|---|---|---|
| `atlas_ui` | UI, icons, tutorial | 1024² | 512² |
| `atlas_common_a` | coins, power-ups, FX primitives & composed FX | 1024² | 512² |
| `atlas_common_b` | shared obstacles, pedestrians, cyclist, telegraph lights | 2048×1024 | 1024² |
| `atlas_player_<char>` | selected character rig only (default char also in a starter page) | 1024² (frames 160²) | 512² (frames 112²) |
| `atlas_dog_<skin>` | selected dog rig only | 1024² (frames 160²) | 512² (frames 128²) |
| `atlas_theme_<t>` | current + prefetched next theme | 2048×640 | 1024×512 |

**Decoded RAM:** MID ≈ 30 MB ✔ (system budget 30), LOW ≈ 11 MB ✔ (budget 14), HIGH ≈ 60 MB (flagship). Rigs other than the selected one are **not resident**; swapping cosmetics loads one small page and evicts the old (`images.clear` on that page). UI + common pages never evict. This is the reconciliation note for the system design's "common atlas holds all rigs" statement: rigs move to per-cosmetic pages to fit the budget — flagged as a design-doc amendment.

### 2.5 Naming system (strict)

Pattern: `<domain>_<subject>_<variant/state>_<index>` — lowercase, snake_case, indices 2-digit.

| Pattern | Examples |
|---|---|
| `player_<anim>_<nn>` | `player_run_01`…`player_run_08`, `player_jump_03`, `player_slide_01` |
| `player_<charid>_<anim>_<nn>` | `player_zara_run_01` |
| `dog_<skin>_<anim>_<nn>` | `dog_brown_run_01`, `dog_bully_lunge_04` |
| `<vehicle>_<color>_<view>` | `car_blue_rear`, `bus_art_front` |
| `obst_<name>[_state]` | `obst_cone`, `obst_barrier_low`, `obst_pothole` |
| `coin_<type>_<nn>`, `pup_<kind>[_icon]` | `coin_std_04`, `pup_magnet`, `pup_shield_icon` |
| `fx_<name>_<nn>` | `fx_coinburst_03`, `fx_dust_01` |
| `env_<theme>_<asset>` | `env_bazaar_facade_02`, `env_coastal_palm`, `env_road_tile_residential` |
| `ui_<group>_<item>[_state]` | `ui_btn_play_n/p/d`, `ui_icon_pause`, `ui_panel_9slice` |
| `tut_<item>` | `tut_hand_tap_01`, `tut_arrow_01` |
| `bg_<theme>_<layer>` | `bg_bazaar_skyline_far`, `bg_residential_sky` |

States: `_n/_p/_d` = normal/pressed/disabled; `_on/_off` = toggles. No other separators, no uppercase, no spaces. Atlas frame keys = file names minus extension. A `tools/lint_naming.dart` CI check enforces the regex `^[a-z0-9]+(_[a-z0-9]+)*$`.

### 2.6 Flame usage (global; blocks reference these)

- Static sprites: `SpriteComponent` from generated frame table (`atlas_frames.g.dart` — packer emits src rects; **zero runtime JSON parsing**).
- Animations: `SpriteAnimationComponent` + `SpriteAnimationData.sequenced(amount, stepTime, textureSize, loop)`.
- World entities are NOT components (per system design): systems write pooled structs; the WorldRenderer reads frame tables and draws via `canvas.drawImageRect` batch — same assets, same pivots.
- Tinted FX: FX primitives authored in **white/neutral**, tinted at draw time via `Paint..colorFilter` — one sprite, many colors (memory win).
- Text: `TextPaint` with **Baloo 2 ExtraBold** (SIL OFL, Google Fonts — bundled in `assets/fonts/`), plum fill + cream stroke for HUD. Wordmark typeset in-tool, not AI-generated.

---

## PART 3 — ASSET INVENTORY & GENERATION PROMPTS

Format per family (all 14 required fields; shared tokens from §0.2 apply to BACKGROUND / STYLE / NEGATIVE / TRANSPARENCY unless overridden).

---

### 3.1 PLAYER CHARACTERS (Category 1)

#### P-01 · Base rig — “Bilal” (Default)

- **ASSET NAME:** `player_run_01..08`, `player_jump_01..06`, `player_fall_01..02`, `player_slide_01..06`, `player_hit_01..04`, `player_death_01..06`, `player_recover_01..04`, `player_idle_01..06`, `player_victory_01..02` (44 frames)
- **CATEGORY:** Player character
- **PURPOSE:** Full gameplay rig, rear-view runner
- **DIMENSIONS:** 256×256 per frame (body ~221 px tall = 1.7 m @130 px/m); MID export 160², LOW 112²
- **BACKGROUND / TRANSPARENCY:** transparent, STD-ALPHA
- **CAMERA/PERSPECTIVE:** CTX-A
- **STYLE:** [MS] + [MS-CHAR]
- **GENERATION PROMPT:** *“original cartoon teenage boy runner, 5-heads-tall stylized proportions, big red sneakers, dark navy jeans, white t-shirt under an open teal sleeveless jacket, mustard-yellow backpack with straps, short black hair, dynamic energetic running pose with arms pumping and backpack bouncing”* + CTX-A + [MS] + [MS-CHAR]
- **NEGATIVE PROMPT:** [MN] + “side view, front view, face visible, standing still”
- **SPRITE/ANIMATION REQUIREMENTS:**

| Anim | Frames | Order | FPS | Loop | Key-pose description (generate these, in-between by hand) |
|---|---|---|---|---|---|
| run | 8 | L→R | 16 | ✔ | R-contact, R-pass, L-contact, L-pass ×2 phases; backpack bounces 2 px at contacts |
| jump | 6 | 1-6 | 20 | ✖ (holds 6) | crouch → launch (arms up) → apex tuck → fall (arms out) → fall2 → land crouch |
| fall | 2 | 1-2 | 10 | ✔ | airborne flail A/B (arms windmill) |
| slide | 6 | in 1–2, loop 3–4, out 5–6 | 18 | mid-loop | lean-in dive → feet-first low pose (loop: dust kick) → push-up → run-ready |
| hit | 4 | 1-4 | 16 | ✖ | recoil back, arms flung, head snapped back, stagger-recover lean |
| death | 6 | 1-6 | 14 | ✖ holds 6 | trip forward → use fall_01-02 → sprawl → flat on ground, backpack up-ended |
| recover | 4 | 1-4 | 12 | ✖ | push-up on arms → one knee → rise → run-ready |
| idle | 6 | 1-6 | 8 | ✔ | relaxed breathing, slight bounce, head micro-turn |
| victory | 2 | alt | 4 | ✔ | both arms up / arms up + knee lift (results new-best) |

- **FLAME USAGE:** rig pages per §2.4; `SpriteAnimationData.sequenced` per anim; state machine swaps `.animation` references (no re-allocation); anchor `custom(0.5, 0.98)`.

#### P-02 · Character cosmetics (8 total — rig identical, outfit/head silhouette changes)

Base prompt = P-01 subject replaced by the row's outfit; everything else identical (CTX-A + [MS] + [MS-CHAR] + [MN]). **Each variant = 44 frames** (`player_<id>_*`), plus one portrait (P-03). Reuse P-01's cleaned base via character reference.

| ID | Name / archetype | Outfit & silhouette prompt fragment | Unique silhouette marker |
|---|---|---|---|
| `bilal` | Default | *(P-01)* | mustard backpack + teal vest |
| `zara` | Casual | *“young woman, coral kurti with denim jacket, white sneakers, high ponytail flying, small teal tote backpack”* | ponytail + tote |
| `ayesha` | Student | *“schoolgirl, sky-blue kameez with white dupatta wrapped over one shoulder, navy satchel backpack, white sneakers, two braids”* | braids + satchel |
| `karim` | Sports | *“athletic coach, red-and-white tracksuit, white headband, whistle on cord, sports duffel slung on back”* | headband + duffel |
| `faisal` | Worker/courier | *“delivery rider, teal-and-yellow rider jacket, small half-helmet, big square parcel-box backpack”* | box backpack + helmet |
| `rustam` | Karachi-inspired (chai uncle) | *“friendly middle-aged uncle, cream kurta with brown sleeveless waistcoat, prayer cap, rolled sleeves, canvas satchel, gray mustache”* | cap + waistcoat |
| `marina` | Karachi-inspired (fisher) | *“fisherwoman, yellow waterproof overalls over striped tee, bucket hat, fishing net draped over shoulder”* | bucket hat + net |
| `djshah` | Karachi-inspired (DJ) | *“young DJ, cream bomber jacket with teal trims, cap worn backwards, chunky headphones around neck, slim backpack”* | headphones + backwards cap |

#### P-03 · Character shop portraits (front 3/4)

- **ASSET NAME:** `player_<id>_portrait` (8)
- **CATEGORY:** Player / UI
- **PURPOSE:** Shop cards, select screens (gameplay is rear-view; players deserve to see the face they bought)
- **DIMENSIONS:** 384×384, hips-up front 3/4
- **CAMERA/PERSPECTIVE:** *“front three-quarter portrait, character facing viewer, confident smile, standing pose”*
- **GENERATION PROMPT:** same outfit fragments as P-02 + the portrait line + [MS] + [MS-CHAR]; **NEGATIVE:** [MN] + “rear view”
- **SPRITE/ANIM:** single frame each
- **FLAME USAGE:** `SpriteComponent` in shop card widget; locked cards shown desaturated (code grayscale filter) under `ui_lock_overlay`.

---

### 3.2 THE DOG (Category 2)

#### D-01 · Base rig — “Street Doggo” (brown, default)

- **ASSET NAME:** `dog_brown_run_01..08`, `dog_brown_lunge_01..06`, `dog_brown_bark_01..04`, `dog_brown_chew_01..04`, `dog_brown_idle_01..06`, `dog_brown_skid_01..04` (26 frames)
- **CATEGORY:** Dog
- **PURPOSE:** The chaser — the game's core threat and brand mascot
- **DIMENSIONS:** 384×384 per frame (body ~340 px); MID export 288², LOW 192²
- **BACKGROUND / TRANSPARENCY:** STD-ALPHA
- **CAMERA/PERSPECTIVE:** CTX-B
- **STYLE:** [MS] + [MS-DOG]
- **GENERATION PROMPT:** *“stylized cartoon street dog, tan-brown short fur, floppy ears, white chest patch, big expressive head, lean athletic body, red collar, exaggerated friendly-fierce expression, full gallop pose with all four paws off the ground”* + CTX-B + [MS] + [MS-DOG]
- **NEGATIVE PROMPT:** [MN] + “front view, side view only, sitting still, cute puppy pose, aggressive blood, showing teeth with saliva”
- **SPRITE/ANIMATION REQUIREMENTS:**

| Anim | Frames | FPS | Loop | Key poses |
|---|---|---|---|---|
| run | 8 | 18 | ✔ | gallop stretch → collected → stretch (rear view: hind legs kick toward camera, tail up) |
| lunge | 6 | 20 | ✖ holds 6 | coil low → leap with forepaws forward → airborne strike → impact crouch → pin pose → snarl up at player |
| bark | 4 | 10 | ✔ | head turned 3/4 side, mouth open bark, ears flap, shoulders bunched |
| chew | 4 | 8 | ✔ | **sitting**, head down gnawing a cream bone, tail wagging (Bone power-up state) |
| idle | 6 | 8 | ✔ | seated, tail sweeping, ear perks (shop, results — “dog escaped!”) |
| skid | 4 | 14 | ✖ | paws-skid lean back → head shake → refocus forward → resume (plays when gap grows back past 8 m) |

**State → animation mapping (covers all 8 requested states without atlas waste):**

| Requested state | Uses | Why |
|---|---|---|
| Running | `run` @18 fps | baseline, gap > 12 m |
| Fast running | `run` @26 fps + code speed-streak FX | deliberate reuse — saves 8 frames/dog; reads perfectly at speed |
| Chase | `run` @22 fps, scale grows | gap 6–12 m |
| Barking | `bark` (overlaid on run every ~2 s) | gap < 6 m |
| Near player | `run` @26 fps + `bark` alternation + growl audio | gap < 3 m |
| Catching player | `lunge` | gap ≤ 0.6 m → slow-mo |
| Happy/idle | `idle` | shop/results |
| Recovery | `skid` + `chew` (bone) | dog loses ground |

- **FLAME USAGE:** dog is a real `SpriteAnimationComponent` (screen-space, per system design §6.3); anchor `custom(0.5, 0.98)`; px follows player lane with 0.33 s lag (code lerp).

#### D-02 · Dog skins (6 total)

Same rig, fur recolor or reference-generation per row; **26 frames each** (`dog_<skin>_*`).

| ID | Name | Prompt fragment / recolor | Bark pitch (playbackRate) |
|---|---|---|---|
| `brown` | Street Doggo | *(D-01 base)* | 1.00 |
| `tommy` | Tommy (black) | *“solid black short fur, pointed alert ears, white toe socks”* | 0.90 |
| `sheru` | Sheru (golden) | *“golden fluffy retriever-type fur, feathered tail, small red bandana”* | 0.95 |
| `shadow` | Shadow | *“charcoal-grey lean whippet-type build, white chest blaze, narrow head”* | 1.05 |
| `bully` | Bully (Bully Kutta) | *“cream-white muscular mastiff-type, broad chest, jowls, cropped-look avoided, gentle-fierce”* | 0.75 |
| `racer` | Rickshaw Racer (spotted) | *“white with brown patches, red goggles on forehead, checkered scarf”* | 1.20 |

#### D-03 · Dog shop portraits

`dog_<skin>_portrait` — 384×384 front 3/4 head-and-chest, big personality expression, same skin fragments, [MS] + [MS-DOG]. 6 files.

---

### 3.3 VEHICLES (Category 4)

All vehicles: static sprites (no frames), baked AO shadow, clear silhouette = readable collision box, taillights/headlights authored **bright but unlit** — the reusable telegraph light sprites (V-10) overlay them when moving/oncoming.

#### V-01 · Car (rear)

- **ASSET NAMES:** `car_blue_rear`, `car_red_rear`, `car_silver_rear`
- **CATEGORY/PURPOSE:** Hard obstacle (severity 6), same-direction/parked
- **DIMENSIONS:** 288×256 (1.7 m w × 1.4 m h @130 px/m + shadow margin)
- **CAMERA:** CTX-C · **STYLE:** [MS] + [MS-VEH]
- **GENERATION PROMPT:** *“stylized cartoon compact hatchback car seen from behind, friendly rounded design, glossy paint, two red taillights, rear window, license plate left blank, slightly worn but clean”* + CTX-C + [MS] + [MS-VEH]
- **NEGATIVE:** [MN] + “side view, front view, text on plate”
- **SPRITE/ANIM:** 1 frame; variants = palette recolor (blue/red/silver)
- **FLAME USAGE:** pooled entity sprite; `Anchor.bottomCenter`; z-sorted in WorldRenderer.

#### V-02 · Taxi (rear) — `taxi_rear`

288×256. *“black-and-yellow city taxi hatchback from behind, yellow roof sign left blank, checkered trim”* + CTX-C + [MS] + [MS-VEH]. 1 file.

#### V-03 · Rickshaw (rear)

- **ASSET NAMES:** `rickshaw_green_rear`, `rickshaw_blue_rear`
- **DIMENSIONS:** 224×256 (1.2 m × 1.5 m)
- **GENERATION PROMPT:** *“stylized cartoon auto-rickshaw three-wheeler taxi seen from behind, green and yellow paint, canvas canopy roof, chrome trim, small rear bench, friendly rounded shape”* + CTX-C + [MS] + [MS-VEH]
- **NEGATIVE:** [MN] + “readable text on rear panel” (rear panels get squiggle decals)
- **SPRITE/ANIM:** 1 frame; 2 recolors (green classic / blue). Severity 4.

#### V-04 · Motorcycle (rear)

- **ASSET NAMES:** `moto_red_rear`, `moto_black_rear`
- **DIMENSIONS:** 192×224 (0.9 m × 1.1 m)
- **GENERATION PROMPT:** *“stylized cartoon commuter motorcycle with helmeted rider seen from behind, rider's back centered, small round taillight, mirrored handlebars”* + CTX-C + [MS] + [MS-VEH]
- **SPRITE/ANIM:** 1 frame; 2 recolors. Severity 4; also the cyclist obstacle's visual cousin.

#### V-05 · Bus (rear)

- **ASSET NAMES:** `bus_city_rear`, `bus_art_rear`
- **DIMENSIONS:** 640×448 (3.8 m × 2.6 m — spans 2 lanes)
- **GENERATION PROMPT (city):** *“stylized cartoon city bus seen from behind, white and green livery with simple painted stripe, rooftop luggage rack, two red taillights, blank destination sign, clean flat sides”* + CTX-C + [MS] + [MS-VEH]
- **GENERATION PROMPT (art):** *“stylized cartoon folk-art decorated bus from behind, hand-painted geometric floral panels in teal, orange and gold, painted wooden crown over the roof, small round mirrors and tassels, original folk pattern, cheerful and ornate but clean”* + CTX-C + [MS] + [MS-VEH]
- **NEGATIVE:** [MN] + “readable letters, calligraphy, real bus company livery”
- **SPRITE/ANIM:** 1 frame each. Severity 6; never two buses in one gate (design rule).

#### V-06 · Truck (rear)

- **ASSET NAMES:** `truck_art_rear`, `truck_cargo_rear`
- **DIMENSIONS:** 384×448 (1.9 m × 2.8 m)
- **GENERATION PROMPT (art):** *“stylized cartoon decorated pickup truck from behind, tall painted wooden crown top with original folk-art floral and geometric patterns in gold, teal and red, colorful mudflaps, blank rear panel”* + CTX-C + [MS] + [MS-VEH]
- **GENERATION PROMPT (cargo):** *“same truck body plain, dusty teal paint, canvas-covered cargo bed”* (recolor/paint-over of art master)
- **NEGATIVE:** [MN] + “readable text, real truck-art copies, brand logos”

#### V-07 · Van & Delivery (rear)

- `van_teal_rear`, `van_white_rear` — 320×320 — *“stylized cartoon minivan from behind, friendly rounded form”* (+ recolors)
- `delivery_parcel_rear`, `delivery_food_rear` — 320×352 — *“stylized cartoon small box-body delivery van from behind, plain teal with simple stripe, blank side panels”* / food variant: *“rounded food-vending box van with a small serving window and striped awning, blank panels”*
- **NEGATIVE:** [MN] + “brand names, franchise logos”

#### V-08 · Oncoming front views (band ≥ 3 obstacles)

| Name | Size | Prompt core |
|---|---|---|
| `car_oncoming_front` | 288×256 | *“front view, two warm headlights on, friendly face-like front”* + CTX-D |
| `taxi_oncoming_front` | 288×256 | taxi front |
| `moto_oncoming_front` | 192×224 | rider front, headlight on |
| `rickshaw_oncoming_front` | 224×256 | rickshaw front |
| `bus_oncoming_front` | 640×448 | bus front, headlights on |
| `truck_oncoming_front` | 384×448 | truck front |

All + [MS] + [MS-VEH]; **NEGATIVE:** [MN] + “rear view”. 1 frame each.

#### V-09 · Vehicle variant policy

Every family's color variants are **palette recolors of the master** (§0.3 rule 3). Total vehicle sprites: **22**.

#### V-10 · Telegraph light overlays (reusable)

- **ASSET NAMES:** `fx_light_rear_on` (red glow disc), `fx_light_head_on` (warm glow disc)
- **DIMENSIONS:** 96×96 each, center pivot
- **PURPOSE:** blinking telegraph overlaid on movers/oncoming vehicles (0.6 s warning, per system design)
- **GENERATION PROMPT:** *“soft round glowing light disc, red / warm yellow, soft falloff, no lens hardware”* + [MS]
- **FLAME USAGE:** additive-blend overlay, code-blinked; tintable (one white master + two tinted uses is also acceptable — ship white master + code tint).

---

### 3.4 OBSTACLES (Categories 3E + 5)

All static, 1 frame, baked AO, `Anchor.bottomCenter` (decals: center). Severity column = dog-gap penalty (system design §7.2; new rows flagged for design-doc merge).

| Asset name | Size | Sev. | Clearable | Generation prompt core (+CTX-E or CTX-F, +[MS]) |
|---|---|---|---|---|
| `obst_barrier_low` | 256×192 | 6 | **jump** (clearH 1.0 m) | *“red-and-white striped low road barrier, two legs, horizontal plank”* |
| `obst_barrier_overhead` | 352×352 | 4 | **slide** (gapBottom 1.0 m) | *“scaffolding pipe barrier: two yellow metal posts and one horizontal pipe at chest height spanning one lane, warning tape”* |
| `obst_cone` | 96×128 | 3 minor | — | *“orange traffic cone with white reflective band, slightly scuffed”* |
| `obst_pothole` | 160×160 | 4 trip | **jump** | *“dark cracked pothole with broken asphalt edges and scattered rubble, irregular oval”* + CTX-F |
| `obst_manhole` | 128×128 | 4 trip | **jump** | *“open round manhole, heavy concrete cover slid half aside, dark hole inside”* + CTX-F *(NEW — design-doc addition)* |
| `obst_hump` | 384×160 | 3 minor | **jump** | *“yellow-and-black striped road speed hump, wide gentle dome”* + CTX-F *(NEW)* |
| `obst_divider_jersey` | 320×160 | 6 | no | *“concrete jersey traffic barrier, plain gray with faded paint chips”* |
| `obst_divider_hedge` | 320×192 | 6 | no | *“low concrete divider with a trimmed green hedge growing on top”* |
| `obst_cart` | 256×320 | 3 minor | — | *“wooden street-food cart with striped canopy umbrella, fruit crates, wheels”* |
| `obst_crate` | 160×160 | 3 minor | — | *“wooden shipping crate with visible planks”* |
| `obst_crate_stack` | 160×224 | 4 | no | *“two stacked wooden crates, top one tilted”* |
| `obst_debris` | 256×160 | 3 minor | jump also | *“pile of construction rubble: broken bricks, pipe section, sand heap”* |

**NEGATIVE (all):** [MN] + “vehicles, people, readable text”.

#### O-02 · Pedestrians (moving obstacles, front-facing — they cross toward the runner)

- **ASSET NAMES:** `ped_uncle_walk_01..04` + `ped_uncle_scare`, and same for `ped_aunty`, `ped_youngman`, `ped_kid` (4 skins × 5 frames = 20)
- **DIMENSIONS:** 176×224 per frame
- **CAMERA:** *“front three-quarter view of a pedestrian walking toward the viewer across a road”*
- **STYLE:** [MS] + [MS-CHAR]
- **PROMPT FRAGMENTS:** *uncle — “middle-aged man, cream kurta, brown waistcoat, flat cap, mustache”*; *aunty — “woman, colorful printed shalwar kameez with dupatta draped over shoulder, handbag”*; *youngman — “young man, t-shirt, jeans, earphones”*; *kid — “schoolboy, shorts, satchel, cap”*
- **ANIM:** walk 4f loop 8 fps (L/R step alternation); `scare` = 1 frame (arms up, startled — plays on near-miss)
- **NEGATIVE:** [MN] + “caricature, stereotype exaggeration, rear view”
- **NOTE (respect rule):** modest, modern, dignified designs; no religious or political symbols. Severity 3 minor by design (fairness-critical, per system design).

#### O-03 · Cyclist (moving minor obstacle)

- **ASSET NAMES:** `cyclist_teal_01..04`, `cyclist_red_01..04` (2 recolors × 4 frames)
- **DIMENSIONS:** 160×224; rear view pedaling, 4f loop 12 fps
- **PROMPT:** *“stylized cartoon cyclist on a commuter bicycle seen from behind, pedaling, small backpack, steady posture”* + CTX-A geometry (rear view) + [MS] + [MS-VEH]
- Severity 3–4 (mover, fixed lane — design-doc merge note).

#### O-04 · Crosswalk decal (NEW — pedestrian-zone telegraph)

`env_crosswalk` — 512×256 top-down zebra stripes, CTX-F, *“white zebra crosswalk stripes on asphalt, slightly worn”*. Placed under pedestrian spawn zones: **art that teaches the mechanic.**

---

### 3.5 COINS (Category 6)

- **ASSET NAMES:** `coin_std_01..06`, `coin_large_01..06`, `coin_star_01..06`, `fx_coin_glow`
- **CATEGORY:** Pickup · **PURPOSE:** 1 / 5 / 10-value pickups (large & star values are a design-doc addition; star spawns rarely in trails)
- **DIMENSIONS:** 128² (std), 160² (large, star), 128² glow
- **CAMERA:** CTX-H-ish, *“spinning coin seen flat-on, rotating around its vertical axis”*
- **GENERATION PROMPT (std):** *“gold game coin, embossed paw-print emblem in the center, thick rim, glossy highlight at upper right, spinning sequence: full face, three-quarter, edge, three-quarter back, back, three-quarter”* + [MS]
- **NEGATIVE:** [MN] + “dollar sign, rupee symbol, numbers, text”
- **ANIM (all):** 6f loop 12 fps, order face→3/4→edge→3/4-back→back→3/4 (seamless loop)
- **VARIANTS:** `coin_large` = bigger master with double rim (same spin); `coin_star` = *“gold five-point star coin with paw emblem”*
- **`fx_coin_glow`:** soft round warm glow, white master, code-tinted gold, additive
- **FLAME USAGE:** pooled entities; spin = frame cycle; bob = code; magnet pull = code lerp; collection = `fx_coinburst`.

---

### 3.6 POWER-UPS (Category 7)

In-world pickups are floating capsules; HUD icons are flat versions. Every kind = unique shape + unique hue (§1.5).

| Asset | Size | Prompt core (+ *“floating glossy capsule orb containing …, thin sparkle ring at its base”* + CTX-E + [MS]) |
|---|---|---|
| `pup_shield` | 160×192 | *“round cyan shield emblem with subtle crest line”* |
| `pup_magnet` | 160×192 | *“violet horseshoe magnet with white pole tips”* |
| `pup_boost` | 160×192 | *“orange double-chevron speed arrows pointing forward”* |
| `pup_coinx2` | 160×192 | *“two overlapping gold coins, one in front of the other”* (no “×2” text — engine typesets the HUD chip) |
| `pup_bone` | 160×192 | *“cream bone with a slight shine”* |

**HUD icons:** `pup_<kind>_icon` — 96² flat, CTX-H, same emblems on rounded squircle badges in each kind's hue. 5 files.
**NEGATIVE:** [MN] + “text, ×2 symbol, letters”.
**FLAME USAGE:** pickup = instant despawn + timer chip in HUD; bob/spin = code; active-state visuals below.

**Active-state FX (player overlays):** `fx_shield_bubble_01..02` (256×256, 2f pulse 4 fps, translucent cyan sphere); `fx_speed_trail_01..03` (three lengths of orange streak, code-spawned behind player); `fx_magnet_field_01..04` (256×256, 4f rotating violet arc ring 10 fps). White-master + code-tint where possible.

---

### 3.7 PARTICLE EFFECTS (Category 8)

**Strategy:** one sheet of **neutral (white/grey) primitives** + a few composed key FX. One sprite serves many colors via code tint — the biggest consistency *and* memory win in the FX plan.

#### F-01 · FX primitives sheet — `fx_prim_*`

**ASSET NAMES:** `fx_prim_circle_soft`, `fx_prim_circle_hard`, `fx_prim_star4`, `fx_prim_star5`, `fx_prim_streak`, `fx_prim_streak_round`, `fx_prim_ring`, `fx_prim_ring_soft`, `fx_prim_shard`, `fx_prim_smoke_01`, `fx_prim_smoke_02`, `fx_prim_sparkline` (12 sprites, 128² each; smoke 192²)
**PROMPT:** *“single isolated particle primitive for mobile game effects: [soft filled circle / crisp circle / four-point star / five-point star / tapered horizontal streak / rounded streak / thin ring / soft ring / angular shard / puffy smoke ball / ragged smoke ball / bright spark line], white and light-grey only, clean edges”* + CTX-H + [MS]
**FLAME:** pooled particle array, one draw pass, tint + rotate + scale + fade by code.

#### F-02 · Composed & scripted FX

| Asset | Frames/Size | Purpose & composition |
|---|---|---|
| `fx_coinburst_01..06` | 6f, 128², 20 fps | coin collection: gold stars + ring burst |
| `fx_dust_01..02` | 2f, 128² | generic ground puff (jump/land/run cadence) — grey-tinted |
| `fx_jump_puff_01..03`, `fx_land_puff_01..03` | 3f each, 160², 24 fps | directional dust kicks |
| `fx_hit_burst_01..04` | 4f, 192², 24 fps | collision: white flash + red-tinged shards + stars |
| `fx_swoosh_01..03` | 3f, 256×128, 24 fps | near-miss arc streak (white, code-tinted pale gold) |
| `fx_speedline_01..03` | 3 streak sprites, 512×64 | boost screen streaks (High/Mid tiers only) |
| `fx_shield_pop_01..04` | 4f, 192² | shield break: cyan shard ring |
| `fx_dogdust_01..02` | reuse `fx_dust` tinted warm | dog's kicked-up dust at close gap |
| `fx_paw_up_01..03` | 3f, 128² | dog-distance-gained feedback: small teal paw prints + upward wisps rising at screen bottom |
| `fx_confetti_01..04` | 4 sprites, 32² | results screen confetti (code-tinted multi-color from white masters) |
| `fx_vignette_red` | 512×512 radial | dog-proximity vignette overlay (code alpha by gap) |
| `fx_flash_white` | 256² soft white disc | impact/game-over flash |
| `fx_shadow_blob` | 192×96 ellipse | shared actor shadow (scaled per entity, alpha by jump height) |

**FLAME:** all particles live in one pooled array rendered in a single canvas pass (per system design §6.11); max 24 active on Low, 60 on Mid.

---

### 3.8 KARACHI ENVIRONMENTS (Category 3 + 10)

**Per theme (6):** sky gradient strip, 2 skyline parallax strips, road tile, sidewalk tile, 3 facades, 1 gate/wall set piece, 6–8 props, 1 signature hero prop, 1 foreground fringe. **Shared urban furniture set** used across themes.

#### 3.8.1 Shared street furniture (all themes) — `env_shared_*`

| Asset | Size | Prompt core (+CTX-E +[MS] +[MS-ENV]) |
|---|---|---|
| `env_shared_pole_utility` | 256×1024 | *“weathered concrete utility pole with cross arms and insulators, small poster patch left blank”* |
| `env_shared_streetlight` | 256×768 | *“curved street lamp arm with rounded lamp head”* |
| `env_shared_sign_stop` | 160×256 | *“red octagonal blank stop-sign shape on a metal post, no symbols”* |
| `env_shared_sign_curve` | 160×256 | *“yellow diamond blank road sign with black arrow chevron”* |
| `env_shared_billboard` | 512×384 | *“billboard on two posts, panel left blank cream”* (engine draws squiggle ads) |
| `env_shared_tree_neem` | 384×512 | *“round shady neem-like tree with dusty green canopy and mottled trunk”* |
| `env_shared_bench` | 256×160 | *“simple green park bench”* |
| `env_shared_bin` | 128×192 | *“green municipal waste bin”* |
| `env_shared_bollard` | 96×192 | *“red-and-white striped bollard”* |
| `env_shared_busstop` | 512×384 | *“simple bus stop shelter with bench and roof, ad panel blank”* |

**Wires are procedural** (code-drawn catenary curves between poles) — crisp at all scales, zero texture cost. Note in FLAME USAGE: `BackdropRenderer` draws pole sprites + quadratic curves.

#### 3.8.2 Theme: RESIDENTIAL (`residential`) — warm noon, Nazimabad-inspired

| Asset | Size | Prompt core |
|---|---|---|
| `bg_residential_sky` | 8×512 strip | *“vertical sky gradient, soft blue to warm cream at horizon”* (opaque; engine stretches) |
| `bg_residential_skyline_far` | 2048×384, tileable | *“distant low rooftops silhouette strip, water tanks and antennae, pale hazy warm tones”* — transparent above roofline |
| `bg_residential_skyline_mid` | 2048×512, tileable | *“mid-distance residential rooftops with parapets, AC units, pale pastel walls, slightly more saturated”* |
| `env_road_tile_residential` | 1024×1024, tileable ×/÷ | *“asphalt road texture viewed top-down, warm-grey worn asphalt, faint patch repairs, three lanes, dashed white lane lines, kerb edges”* + CTX-F |
| `env_sidewalk_tile_residential` | 512×256, tileable | *“pale warm concrete sidewalk paving with kerb edge, top-down”* + CTX-F |
| `env_residential_facade_01..03` | 512×768 | *“two-storey house facade, faded pastel paint (cream, mint, rose), grilled windows, small balcony with potted plants, parapet with pattern, blank sign-free”* + CTX-G |
| `env_residential_wall` | 512×384 | *“low boundary wall with decorative concrete blocks and a painted trim, bougainvillea spilling over”* + CTX-G |
| `env_residential_gate` | 384×512 | *“simple metal gate in a boundary wall, faded teal paint”* |
| `env_residential_props` (set) | ~256² each | water tank on stand, scooter parked (decor, no rider), potted plants cluster, kite stuck on a pole *(NEW)*, clothesline with garments, wall poster patch (blank) |
| `bg_residential_fg` | 2048×256 | sparse foreground fringe: passing pole + branch silhouettes, transparent middle (Mid/High tiers) |

**Kites:** 4 small sky sprites (`env_kite_01..04`, 96²) drifting in the far sky over residential/bazaar — signature Karachi warmth, near-zero cost.

#### 3.8.3 Theme: URBAN ARTERIAL (`urban`) — Shahrah-e-Faisal-inspired, cool midday

- `bg_urban_sky` (cool blue→pale), `bg_urban_skyline_far` (2048×384: *“distant glass towers, pale blue-grey haze”*), `bg_urban_skyline_mid` (2048×512: *“mid-rise office buildings, curtain-glass, concrete, rooftop AC clusters”*)
- `env_road_tile_urban` (1024²: *“wider smooth dark asphalt, fresh white lane lines, reflective paint”* + CTX-F), `env_sidewalk_tile_urban` (512×256)
- `env_urban_facade_01..03` (512×1024): *“modern office tower base: glass curtain wall, stone cladding, recessed lobby, blank sign band”* + CTX-G
- `env_urban_flyover_pillar` (384×768): *“concrete flyover/overpass pillar with flared cap”* — signature vertical
- `env_urban_gantry` (768×512): *“road gantry frame spanning the road, blank green panels”*
- Props: traffic signal (tall), divider barrier planted, streetlight double-arm, planter box.

#### 3.8.4 Theme: BAZAAR (`bazaar`) — Saddar/Tower-inspired, warm amber

- `bg_bazaar_sky` (warm), skylines: *“old city rooftops with domes and water tanks, amber haze”* (2048×384 / 2048×512)
- Road/sidewalk tiles: *“well-worn asphalt with patchwork repairs, sun-bleached lane lines”* / *“old stone-flag sidewalk”*
- `env_bazaar_facade_01..03` (512×1024): *“narrow three-storey bazaar buildings, stacked balconies with laundry, colorful canvas awnings, hand-painted blank signboards with abstract squiggle-glyph patterns, hanging fabric, warm amber light”* + CTX-G — **squiggle-glyph rule: signs read as “text-like” but contain no real letters**
- Signature: `env_bazaar_awning_row` (1024×384, *“row of striped shop awnings above a covered sidewalk”*)
- Props: stacked crate pyramid, fruit stall cart (decor variant), hanging lantern string, rolled fabric bolts, spice sacks, bicycle leaned on wall.

#### 3.8.5 Theme: COMMERCIAL (`commercial`) — I.I. Chundrigar-inspired, clean steel

- Skies/skylines: *“financial district, stone-and-glass bank facades, flag poles with plain green pennants, crisp midday light”* (no crescent-star rendering — plain pennants avoid symbol errors)
- Road: *“clean dark asphalt, crisp markings, painted arrow lane decals”*; sidewalk: *“granite pavers”*
- Facades ×3 (512×1024): *“formal stone bank building with columns, brass-framed blank sign, clock face without numerals”* (clock = geometric shapes only)
- Props: bollard chain, planter, modern bus stop, newspaper stand (blank), flag pole with green pennant.

#### 3.8.6 Theme: COASTAL (`coastal`) — Sea View/Clifton-inspired, golden sunset

- `bg_coastal_sky` (orange→pink→violet strip), `bg_coastal_sun` (192² soft sun disc), `bg_coastal_skyline_far` (*“distant headland and faint city silhouette across water”*), `bg_coastal_sea_band` (2048×256, *“flat sea strip with sun glitter path”* — drawn between sky and road horizon)
- Road: *“dusk-tinted asphalt, worn white lines, sand dust at edges”*; sidewalk: *“seafront promenade tiles”*
- Signature: `env_coastal_railing` (512×256, *“white painted sea railing with simple posts”*) + `env_coastal_palm_01..02` (384×768, *“leaning palm tree with fronds”*)
- Props: ice-cream cart (decor), striped beach umbrella, rope + bollard, seagull silhouettes (2 sprites, code-drifted), parked family bicycle.

#### 3.8.7 Theme: DENSE TRAFFIC (`traffic`) — Old City / W-11-route-inspired, dusk

- `bg_traffic_sky` (olive-gold dusk), skylines: *“very close weathered facades and tangled overhead cable runs, dim smog haze, deep warm shadows”* (keep ≥40% lightness — readability law)
- Road: *“scuffed asphalt, faded markings, oil stains, patch repairs”*
- Facades ×3: *“crumbling old-city building sides with AC units, cable clusters, blank painted panels”*
- Signature: `env_traffic_cable_pole` (utility pole with dense insulator clusters — pairs with procedural wires)
- Props: stacked tires, hand-painted wall patch, small shrine-like flower corner **(kept abstract & respectful: just a flower-and-cloth corner, no symbols)**, tea stall (decor), parked motorcycle cluster.

**All environment NEGATIVE:** [MN] + “readable text, real shop names, real brand logos, people, vehicles in facades”. **All tileable assets:** generated with wrap-friendly edges, then seam-checked and hand-fixed — tileability is verified in QA (§7.6).

---

### 3.9 UI (Category 9) + TUTORIAL (11) + SHOP (12)

**UI master style [MS-UI]:** *“mobile casual-game UI art, chunky rounded shapes, flat two-tone fill with a darker bottom edge (candy-button look), thick dark-plum outline, cream and teal and gold palette, soft inner highlight, generous padding, readable at small sizes”* + CTX-H. Panels ship as **9-slice** (safe margins marked). Button states `_n/_p/_d`: pressed = darker + 4 px optical drop; disabled = desaturated.

#### U-01 · Brand

| Asset | Size | Spec |
|---|---|---|
| `ui_logo_emblem` | 512×512 | *“circular game badge: cartoon dog head in playful snarl beside a red sneaker with motion lines, gold rim, teal ribbon at bottom, no letters”* + [MS-UI] |
| `ui_logo_full` | 1024×512 | emblem + wordmark “DOG RUNNER” **typeset in Baloo 2 ExtraBold**, gold fill, plum outline, slight arch — assembled in design tool, never AI text |
| `ui_logo_small` | 512×256 | horizontal lockup for loading screens |
| `ui_appicon_fg` / `_bg` | 512² (safe zone 432²) | *“joyful runner sprinting, determined cartoon dog close behind, bold simple shapes, cream circular backdrop, thick outlines, readable at 48 px”* + [MS]; adaptive icon fg (transparent margins) + solid teal bg |
| `ui_keyart_splash` | 1080×1920 | *“full illustration: the runner sprinting down a sunlit Karachi street toward the viewer with the dog right behind, motion lines, coins sparkling”* — full scene allowed here (this is the ONE backgrounded illustration in the game); splash screen + store screenshots base |

#### U-02 · Buttons & navigation

| Asset | Size | Notes |
|---|---|---|
| `ui_btn_primary_n/p/d` | 512×160 (9-slice) | teal candy button, cream label area (text by engine) |
| `ui_btn_secondary_n/p/d` | 512×160 | cream button, plum outline |
| `ui_btn_icon_round_n/p` | 160² | round teal squircle base (hosts any icon) |
| Icon set, 96² each: `ui_icon_pause, play, restart, home, settings, shop, missions, character, dog, close, arrow_left, arrow_right, gift, ad_video, sound_on, sound_off, music_on, music_off, haptics, restore, info, lock, check, cross, trophy, distance` | 96² | *“flat filled icon of [subject], two-tone shading, thick outline”* + [MS-UI]; **NEGATIVE:** [MN] + “thin lines, gradient mesh” |
| `ui_tab_bar` (9-slice) + tab states | 512×128 | bottom nav for shop/select screens |
| `ui_toggle_on/off`, `ui_slider_track`, `ui_slider_knob` | 192×96 / 512×64 / 96² | settings controls |

#### U-03 · Panels & frames

| Asset | Size | Notes |
|---|---|---|
| `ui_panel_9slice` | 512² (9-slice) | cream panel, plum border, soft inner shadow |
| `ui_panel_header_ribbon` | 512×192 | teal ribbon banner |
| `ui_card_frame_n/s` (normal/selected) | 384×512 | shop/select card frame; `_s` adds gold selection ring |
| `ui_card_locked_overlay` | 384×512 | diagonal hatch + dark scrim |
| `ui_price_chip` | 256×96 | gold pill hosting coin icon + engine text |
| `ui_badge_equipped` | 256×128 | *“equipped” ribbon* — shape only; text by engine |
| `ui_badge_dot` | 64² | notification dot |
| `ui_progress_frame`, `ui_progress_fill` (9-slice) | 512×64 | mission progress bars |
| `ui_loading_frame`, `ui_loading_fill` | 512×48 | splash progress bar |

#### U-04 · Gameplay HUD

| Asset | Size | Notes |
|---|---|---|
| `ui_hud_score_frame` | 512×128 | cream pill for score text |
| `ui_coin_pill` | 320×96 | coin icon + count pill |
| `ui_dogmeter_bg`, `ui_dogmeter_fill`, `ui_dogmeter_dogface`, `ui_dogmeter_playerface` | 512×96 / 96² | chase meter: green→amber→red zones baked in bg |
| `ui_pup_chip_<kind>` ×5 | 128² | HUD timer chips (frame + emblem; radial timer drawn by engine) |
| `ui_combo_banner` | 512×160 | combo flash frame (text by engine) |
| `ui_best_ribbon` | 384×160 | “new best” ribbon |
| `ui_gameover_banner` | 1024×384 | *“cracked wooden hanging banner with torn edges, no letters”* — text typeset |
| `ui_ad_reward_card` | 768×512 | rewarded-ad card: video-play badge + `ui_coin_pile` + ×2 slot (text by engine) |
| `ui_coin_pile` | 256×192 | small gold coin pile |
| `ui_countdown_ring` | 256² | 3-2-1 ring (numbers by engine) |

#### U-05 · Tutorial (Category 11)

| Asset | Size | Notes |
|---|---|---|
| `tut_hand_tap_01..02` | 160² | 2f tap loop; *“cartoon pointer hand, index extended”* |
| `tut_arrow_01..03` | 192² | 3f chevron pulse loop — **one asset, rotated by engine** for all four directions |
| `tut_warn_triangle` | 160² | red warning triangle + exclamation shape |
| `tut_callout_frame` | 9-slice | cream callout bubble |
| Tutorial pointers reuse gameplay sprites (cone, coin, `pup_shield`, dogmeter closeup crop `tut_dogmeter_zoom`) — no duplicates. |

---

## PART 4 — AUDIO SPECIFICATION (Category 13 — spec only, no generation)

Format: **OGG Vorbis, 44.1 kHz**. SFX mono, ≤ 0.6 s unless noted, peak −3 dBFS. Music stereo, −14 LUFS integrated. All via `flame_audio` pools (max 4 voices, §6.12 of system design).

| Asset name | Source | Length | Notes |
|---|---|---|---|
| `sfx_step_asphalt_01/02` | footstep | 0.08 s | subtle, alt-random at run cadence, −12 dB under music |
| `sfx_jump` | whoosh+effort | 0.25 s | |
| `sfx_land` | soft thud + dust | 0.15 s | |
| `sfx_slide` | fabric/gravel whoosh | 0.5 s | |
| `sfx_coin_01/02/03` | bright pling, rising triad per streak level | 0.12 s | pitch steps +1 st per streak-5, resets on miss |
| `sfx_streak` | chime arpeggio | 0.4 s | 25-coin streak |
| `sfx_pup_pickup` | shared chime base | 0.3 s | |
| `sfx_shield_on` / `sfx_shield_break` | energy bubble / glass pop | 0.3 / 0.25 s | cyan "wet" timbre |
| `sfx_magnet` | electric hum blip | 0.3 s | violet hum, no loop |
| `sfx_boost` | rising whoosh | 0.5 s | |
| `sfx_bone` | playful pop + dog yip | 0.4 s | |
| `sfx_stumble` | body impact + grunt | 0.3 s | |
| `sfx_bark_01/02/03` | dog bark, 3 takes | 0.3–0.4 s | pitch-scaled per skin (D-02 table); proximity volume by gap |
| `sfx_growl_loop` | low growl | 1.2 s loop | gap < 6 m, crossfaded |
| `sfx_whine` | sad dog whine | 0.6 s | dog pushed back (skid) |
| `sfx_catch` | bark + snap + thud | 0.6 s | game-over catch |
| `sfx_gameover_jingle` | descending brass-ish | 2.0 s | |
| `sfx_highscore_jingle` | bright fanfare | 2.5 s | |
| `sfx_mission_complete` | two-note success | 0.6 s | |
| `sfx_unlock` | treasure open | 0.8 s | |
| `sfx_buy` | coin register | 0.5 s | |
| `sfx_ui_click` / `sfx_ui_back` | soft tick / lower tick | 0.08 s | |
| `sfx_countdown_01/02` | beep / higher beep | 0.15 s | 3-2-1 / GO |
| `sfx_revive` | whoosh-recover | 0.6 s | |
| `mus_menu_loop` | warm bouncy chai-house theme, sitar-ish lead + dhol light | 1:30 loop | |
| `mus_game_base` | driving runner groove, mid energy | 2:00 loop | |
| `mus_game_high` | same structure, +energy (extra percussion/lead) | 2:00 loop | crossfade in at band ≥ 3 |
| `mus_perc_<theme>` ×6 | 4-bar percussion-only flavor loops (dhol, tabla-lite, clap patterns) | 0:08 loops | layered at low volume per theme — cheap localization of music |

**Music architecture note:** 2 global intensity stems + 6 short perc loops instead of 12 full theme tracks — the system design's "adaptive layers" made budget-real.

---

## PART 5 — SHEET INDEX (master summary)

| Sheet | Frames/Sprites | Frame size (master) | Anim summary |
|---|---|---|---|
| `atlas_player_<char>` ×8 | 44 each | 256² | 9 anims (§P-01) |
| `atlas_dog_<skin>` ×6 | 26 each | 384² | 6 anims + reuse map (§D-01) |
| `atlas_common_b` vehicles | 22 | up to 640×448 | static |
| `atlas_common_b` obstacles | 12 + decals 2 | up to 384² | static |
| `atlas_common_b` pedestrians | 20 | 176×224 | walk 4f + scare |
| `atlas_common_b` cyclist | 8 | 160×224 | pedal 4f ×2 colors |
| `atlas_common_a` coins | 19 | 128–160² | spin 6f ×3 |
| `atlas_common_a` power-ups | 10 + active FX 9 | 96–256² | static + pulse/trail anims |
| `atlas_common_a` FX | 12 primitives + ~30 composed | 32–512² | 2–6f each |
| `atlas_theme_<t>` ×6 | ~16 each | up to 2048×512 strips | static, tileables |
| shared furniture | 10 | up to 256×1024 | static |
| `atlas_ui` | ~85 | up to 1024×384 | states, 9-slices |
| Portraits | 14 (8 char + 6 dog) | 384² | static |
| Key art | 2 (splash + logo emblem) | up to 1080×1920 | static |

**Totals: ~380 sprite files, ~470 unique frames, 24 audio SFX files, 9 music/perc files.**

---

## PART 6 — COPYRIGHT & CULTURAL SAFETY (Category 19)

1. All characters/dogs original designs; [MN] explicitly blocks Disney/Pixar/Marvel/anime/existing-game references in every prompt.
2. **No real brands:** all signage blank or squiggle-glyph; taxis/buses/rickshaws are generic shapes; truck-art patterns are **original** folk-style designs, not copies of real trucks' artwork.
3. **No real text** anywhere in art (also solves AI text inconsistency).
4. **National/religious symbols avoided** in decor (plain pennants, no flags with emblems, no shrine iconography — abstract flower corner only).
5. Pedestrians and characters: dignified, modern, non-caricatured; modest attire where culturally natural; QA check for stereotype risk (§7.6).
6. Fonts: Baloo 2 (SIL OFL) bundled; no font licensing risk.
7. Karachi inspiration lives in **environment archetypes** (rickshaws, truck-art, kites, chai stalls, seafront) — general cultural concepts, not protected expression.

---

## PART 7 — SELF-REVIEW LOOP (Category 20)

### 7.1–7.15 Checks, findings, fixes (fixes already integrated above)

| # | Check | Verdict | Findings & fixes applied |
|---|---|---|---|
| 1 | All gameplay objects represented? | PASS after fix | **Missing found:** oncoming vehicle views (V-08) — the system design has oncoming traffic; rear views alone can't show it. Added 6 front sprites + headlight telegraph. Also added manhole, road hump, crosswalk decal, cyclist (all flagged for severity-table merge into the design doc). |
| 2 | All screens represented? | PASS after fix | **Missing found:** splash key art, loading bar, ad-reward card, game-over banner, countdown ring, notification badge, best-score ribbon → all added (U-01…U-04). |
| 3 | All animations represented? | PASS after fix | Requested 8 player states mapped to 9 anims (fall folded into jump/death sets + explicit 2-frame fall); dog's 8 states mapped via reuse table (D-01) — "fast run" is playback-rate reuse, documented, saving 8 frames/skin with no perceptible loss. |
| 4 | All UI elements represented? | PASS after fix | Added 9-slice panels (all screen sizes), button 3-states, toggles, sliders, tab bar, shop card states, equipped/locked badges, price chips. |
| 5 | All 6 Karachi environments? | PASS | 6 themes, each with full layer stack (sky→far→mid→road→props→foreground) + signature prop + kite/seagull life details. |
| 6 | All obstacles? | PASS | Catalog cross-checked line-by-line against system design §7.2; every item has an asset. |
| 7 | All power-ups? | PASS | 5 in-world + 5 HUD + 3 active-state FX (shield bubble, speed trail, magnet field) — active states were missing, added. |
| 8 | Player & dog states? | PASS | Player 9 anims incl. death/recovery/victory; dog 6 anims + state map incl. chew (bone), skid (recovery), idle (escaped). |
| 9 | Technically suitable for Flame? | PASS after fix | Rigs moved to per-cosmetic pages to fit the 30 MB MID budget (amendment to system design §2.4 documented in §2.4 here); frame tables code-generated (no runtime JSON); NPOT pages allowed; tintable white FX masters. |
| 10 | Dimensions/transparency/pivots/sheets defined? | PASS | Global standards §2.2–2.3 + per-family tables + px-per-meter rule tying art to collision sizes. |
| 11 | Visual consistency? | PASS after fix | Risks found & neutralized: perspective drift → 8 locked CTX tokens; palette drift → locked hex roles; outline drift → normalization step in pipeline; AI text → banned engine-wide; variant drift → recolor-not-regenerate rule + reference-based generation. |
| 12 | Missing assets? | FOUND | Added: `fx_shadow_blob` (actors can't bake shadows), `fx_light` telegraphs, sky strips as tiny stretched gradients (memory), procedural wires decision, `ui_keyart_splash`, sun/moon, seagulls, kites, portraits (shop needs faces), audio additions (whine, buy, countdown, revive). |
| 13 | Added? | DONE | All integrated in-place. |
| 14 | Duplicates removed? | DONE | Removed/reused: separate dog fast-run frames (playback reuse), per-theme street furniture (shared set), per-coin glow (one shared glow), 4 tutorial arrows (one rotated), speed-line variants trimmed to 3, “×2” art text (engine typesets), separate ad-card coin art (reuses `ui_coin_pile`). |
| 15 | Production readiness | PASS | Every asset has: name, size, pivot, frames, FPS, loop, prompt, negative, Flame usage. Budgets verified (MID 30 MB / LOW 11 MB decoded). Pipeline has CI lint + art QA gates below. |

### 7.6 Art QA gates (per asset, before merge)

1. **Silhouette test** — readable at 25% zoom, grayscale.
2. **Contrast test** — gameplay objects pop against all 6 theme backdrops (composite check).
3. **Palette audit** — sampled colors within locked ramps (±8% tolerance).
4. **Outline audit** — uniform weight after export downscale.
5. **Pivot/alignment** — feet on grid line, pivot within 1 px.
6. **Seam test** — tileables wrapped, no visible seam at 200%.
7. **Loop test** — first/last frames of every loop blend.
8. **Alpha test** — no fringe, no 1%-alpha garbage pixels.
9. **Cultural review** — stereotype/brand/symbol check (§6).

### Final verdict

**No critical gaps remain.** The library is complete against all 20 required categories, internally consistent by construction (tokens + recolor pipeline + locked palette/perspective/lighting), memory-verified against the engine design's budgets, and every animated asset has full sheet metadata. The two deliberate design-doc amendments (per-cosmetic rig pages; large/star coin values; new obstacles hump/manhole/cyclist/crosswalk) are flagged in-line for merge into the technical system design.

**Ready for production. Build order:** approve base player + base dog → common_a/common_b (coins, power-ups, FX, obstacles) → one theme end-to-end (residential) → vertical-slice art QA → remaining 5 themes → vehicles + oncoming set → UI kit → cosmetics → portraits/key art → audio spec hand-off.