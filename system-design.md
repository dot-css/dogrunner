# DOG RUNNER — Complete Technical System Design

**Version:** 1.0 (implementation-ready)
**Platform:** Android-first, Flutter + Flame 1.x, low/mid-range optimized
**Scope:** Full game architecture, systems, data, monetization, performance, QA, security, self-review.

---

## 1. Summary & Key Technical Decisions

| Area | Decision | Rationale |
|---|---|---|
| View | Pseudo-3D behind-the-player, 3 lanes, 2D sprites with perspective scaling | Subway-Surfers feel at 2D cost; scales well on low-end |
| Collision domain | Logical world space (x=lane, y=height, z=distance), never screen space | Cheap, exact, resolution-independent |
| Entity management | **Pooled lightweight structs + one custom renderer**, NOT one Flame component per entity | Avoids component-tree churn & GC on low-end; player/dog are real Flame components |
| Dog | Screen-space anchored chaser (Temple-Run-style), pure state math, **zero pathfinding** | Believable, cheap, meaningful |
| Death model | No instant-death obstacles; dog catch is the only death; hits feed the dog | Dog is the core mechanic per spec |
| Procedural gen | Authored template library + runtime validator + bot-fuzzed fairness | Deterministic quality, no impossible patterns |
| Storage | Single versioned JSON file (atomic write, validated, checksummed) + `shared_preferences` for boot flags | Local-first, corruption-safe |
| Monetization | AdMob rewarded + interstitial with hard frequency caps; optional Remove-Ads IAP; **one soft currency only** | F2P, not pay-to-win, minimal tech |
| FPS | 60 target, 30 optional (battery), variable `dt` clamped to 33 ms + swept collision | Simpler than fixed-step; no tunneling |
| Device tiering | First-launch frame benchmark + manual override; no extra plugins | Honest measurement of real performance |

---

## 2. Technology & Dependencies (deliberately minimal)

```yaml
dependencies:
  flutter:                # stable channel
  flame: ^1.18.0          # game engine (pin exact version)
  flame_audio: ^2.x       # pooled SFX + streamed music
  google_mobile_ads: ^5.x # AdMob + UMP consent
  shared_preferences: ^2.x# fast boot flags only
  path_provider: ^2.x     # save file, logs
  in_app_purchase: ^3.x   # optional "Remove Ads" (non-consumable) — Phase 2, interface stubbed in Phase 1

dev_dependencies:
  flutter_test, flame_test, flutter_lints
```

**Explicitly excluded:** no backend, no Firebase (analytics optional behind an interface, default no-op), no device_info (tiering via benchmark), no leaderboards/cloud saves in v1, no second currency.

---

## 3. High-Level Architecture

### 3.1 Layering

```
┌──────────────────────────────────────────────────────┐
│ UI LAYER (Flutter widgets)                           │
│ Screens: Splash, Home, Game, Pause, GameOver, Shop,  │
│ Missions, CharacterSelect, DogSelect, Settings       │
│ Communicates with game only via ValueNotifiers/Events│
├──────────────────────────────────────────────────────┤
│ GAME LAYER (Flame)                                   │
│ DogRunnerGame (FlameGame)                            │
│  ├─ GameSession (state machine, run lifecycle)       │
│  ├─ Systems (ordered, pure-Dart, no widget refs)     │
│  ├─ WorldState + EntityStore (pooled entities)       │
│  ├─ Renderers (Backdrop, World, Player, Dog, FX)     │
├──────────────────────────────────────────────────────┤
│ SERVICE LAYER (interfaces + impls)                   │
│ AudioService, AdService, IapService, AnalyticsService│
│ SaveRepository, SettingsService, TierService         │
├──────────────────────────────────────────────────────┤
│ DATA LAYER                                            │
│ Models, catalogs (const data), save JSON, configs    │
└──────────────────────────────────────────────────────┘
```

**Rules:**
- Systems never import `package:flutter/widgets.dart`; they are unit-testable pure Dart.
- UI never mutates game state directly; it sends `Intent`s / calls `Session` methods.
- Services are injected via a simple `Services` bundle (no DI framework).

### 3.2 App flow state machine

```
Splash ──► (firstRun? Tutorial) ──► Home ◄────────────┐
                                        │             │
   Home ─► Shop / Missions / Selects / Settings ──────┘
   Home ─► GameScreen
             │
   GameSession states:
   Loading ─► Ready(tap) ─► Countdown(3s) ─► Playing ◄─┐
                                    │        │         │(revive)
                                    ▼        ▼         │
                                 Paused   Dying(1.2s)──┘
                                             │
                                             ▼
                                   GameOver(ReviveWindow 5s)
                                             │
                                     Results ─► Home / Run Again
```

### 3.3 Lifecycle policy (`WidgetsBindingObserver`)

| Event | Behavior |
|---|---|
| `paused` (background) | If `Playing` → auto-pause, stop audio, persist `pendingRun` snapshot |
| `resumed` | Show Pause menu (never auto-resume into action); reload ads |
| `inactive` (call/overlay) | Same as paused |
| `detached` / OS kill | `pendingRun` snapshot (written every 5 s during play) → on next launch: “Interrupted run — +N coins” recovery dialog |
| Music/SFX | Muted while any ad or system call UI is showing |

---

## 4. World Model: Coordinates, Projection, Camera

### 4.1 Logical space (meters)

- `x` ∈ [−1.7, +1.7]; lane centers at −1, 0, +1; **lane width = 2.2 m** (lane index 0/1/2).
- `y` = height above ground (0 = ground).
- `z` = meters ahead of the player; **player is always z = 0**. Entities move toward the player (`z -= speed·dt`). Total run distance is a separate monotonically growing `double` (safe to 2⁵³).
- Spawn horizon `+120 m`; recycle threshold `−20 m`.

### 4.2 Projection (reference resolution 1080×1920, scale-to-fit-width)

```dart
class RoadProjection {
  static const double camD = 8.0;  // camera distance behind player plane
  static const double camK = 8.0;  // scale(0) == 1.0
  double horizonY, groundY, cx, halfLanePx;   // set on resize

  double scale(double z) => camK / (camD + z);              // z >= 0 only for road entities
  double screenX(double px, double s) => cx + px * halfLanePx * s + camSwayX;
  double groundScreenY(double s) => horizonY + (groundY - horizonY) * s;
  // Sprite: draw at (screenX, groundScreenY) anchored bottom-center,
  // size = basePx * s. Height offset: groundScreenY - h * s * pxPerMeter.
}
```

- Far clip: entities with `scale < 0.06` are skipped (culling).
- Camera motion: subtle run-bob (±2 px, disabled on Low tier), lateral sway `camSwayX = playerPx * 0.15 * halfLanePx`, shake on hits (decaying, ≤6 px, disabled on Low).

### 4.3 Draw order (painter's algorithm)

1. Sky gradient + skyline parallax strip (theme-tinted, cached bitmap).
2. Road plane (pre-rendered road tile strip drawn as 3 scaled slices; lane dashes as pooled decor sprites).
3. Entities with `z ≥ 0`, **far → near**. The entity array is spawned in z-order and motion is monotonic, so a single **insertion-sort pass is O(n)** per frame (list is nearly sorted; only changes when entities pass the player). Zero allocations.
4. Player (Flame `SpriteAnimationComponent`).
5. Dog (screen-space overlay, see §6.3).
6. FX/particles, proximity vignette.

---

## 5. Game Loop — Exact Per-Tick Specification

`DogRunnerGame.update(dt)` — `dt = rawDt.clamp(0, 1/30)`:

| # | Step | Owner | Work |
|---|---|---|---|
| 1 | Drain input queue | InputSystem | Convert gestures → `Intent{left,right,jump,slide}` (only in `Playing`) |
| 2 | Session timers | GameSession | Invulnerability, power-up timers, revive countdown |
| 3 | Difficulty | DifficultySystem | `band = min(5, 1 + distance ~/ 250)`; target speed recompute |
| 4 | Speed | WorldState | `speed → lerp toward target (rate 0.5 m/s per s)` |
| 5 | Scroll | MovementSystem | `distance += speed·dt`; all pooled entities `z -= speed·dt`; movers add own velocity; store `prevZ` for sweep |
| 6 | Player | PlayerSystem | Lane tween (0.28 s ease-out), jump physics, slide timer, animation state |
| 7 | Dog chase | DogChaseSystem | Gap regen/decay, catch check, dog anchor/anim update, proximity audio |
| 8 | Spawn/recycle | SpawnSystem | While `spawnedUntil < distance + 120` → place next segment (validated); free entities with `z < −20` to pools |
| 9 | Entity behavior | MovementSystem | Pedestrian crossing walk, coin spin, magnet pull toward player, power-up bob |
| 10 | Collision | CollisionSystem | Narrow window `|z| ≤ 3 m`, swept along z, severity resolution, near-miss detection |
| 11 | Score & coins | ScoreSystem | score, coin streaks, multiplier application |
| 12 | Missions | MissionSystem | Emit events (`coin_picked`, `near_miss`, `jump`, …) to active trackers |
| 13 | FX/Camera | FxSystem | Shake decay, vignette intensity by dog gap, particle update (capped) |
| 14 | HUD sync | HudBridge | Dirty-flag `ValueNotifier` updates; score text throttled to 10 Hz |
| 15 | Audio cues | AudioCueSystem | Bark/growl volume by gap, music intensity layer, event SFX |
| 16 | Autosave | AutosaveSystem | Every 5 s write `pendingRun{coins, score}` (throttled, async) |

Render pass then draws per §4.3. **No allocations in steps 1–16**: all scratch vectors, event objects, and entities are pooled/reused; strings are never concatenated in the hot path.

---

## 6. Core Systems

### 6.1 Game Session & State

`GameSession` owns the state machine (§3.2) and is the only writer of run-scoped state:

- `phase`, `speed`, `distance`, `score`, `coinsThisRun`, `nearMisses`, `comboLevel`
- `invulnTimer`, `reviveUsed`, `activePowerUps` (bitmask + timers)
- Transitions: `start()`, `pause()`, `resume()`, `stumble(severity)`, `dogCatch()`, `offerRevive()`, `revive()`, `finish()`.

**Revive contract:** once per run, rewarded ad required; on revive → clear all obstacles within 25 m ahead, dog gap reset to 18 m, 2 s shield, score/coins retained, coins already collected are **not** re-awarded (nearby coins are also cleared to prevent double-collect).

### 6.2 Input & Player Controller

- **Gesture layer** (Flutter, wraps `GameWidget`): custom 4-way swipe recognizer — activation threshold 40 px or velocity > 500 px/s, 350 ms max gesture time, first-direction-wins arbitration. Also: tap = jump (accessibility). Optional “button mode” in Settings (4 on-screen buttons).
- Intents go into a fixed `RingBuffer<Intent>(8)`; consumed once per tick; stale intents (>120 ms old) dropped.
- **Player physics:**
  - Lane change: `px` tweens to target lane over **0.28 s** (ease-out). No queued mid-air restriction (jumping + lane change allowed — feels fair).
  - Jump: apex 1.6 m, airtime 0.70 s (`v₀ = 9.14 m/s`, `g = 26.1 m/s²`). Clears obstacles with `clearHeight ≤ 1.2 m`.
  - Slide: 0.8 s, hitbox height 1.7 → 0.8 m. Passes obstacles with `gapBottom ≥ 1.0 m`.
  - Stumble: 0.8 s at 60% speed + flinch animation; **1.5 s invulnerability** (prevents chained hits, dog catch still possible).

### 6.3 Dog Chase System (the core mechanic)

**State:** `gap` (meters behind player), start **18**, range `[0, 24]`.

```dart
void update(double dt) {
  // Baseline recovery after 3 s clean running
  if (timeSinceLastHit > 3.0) {
    gap = min(gap + 0.25 * dt, pressureCap);   // pressureCap = 24 - 2*(band-1), min 14
  }
  if (gap <= 0.6) session.dogCatch();          // checked every frame, even during invulnerability
}

void onObstacleHit(Severity s) {               // 3 = minor, 4 = trip, 6 = major
  timeSinceLastHit = 0;
  gap -= s;
  fx.dogLunge(); audio.bark();
  if (gap <= 0.6) session.dogCatch();
}

void onNearMiss()  { gap = min(gap + 0.5, pressureCap); }  // rate-capped: max +1 per 5 s
void onCoinStreak(int streak25) { gap = min(gap + 1.0, pressureCap); }  // each 25-coin streak
```

**Design consequences (why it’s meaningful):**
- The dog is the *only* death. Two quick major hits (−12) kill a full-gap player → constant tension without instant-death frustration.
- `pressureCap` shrinks with difficulty (24 → 14 m), so late-game recovery is weaker — the run ends by accumulated pressure.
- Near-misses and coin streaks reward skilled play by pushing the dog back.
- Power-ups interact: **Bone** (dog stops to chew 4 s → gap grows by `4·speed`, capped), **Speed Boost** (player +40%, dog matches base speed → gap grows).

**Visual representation (zero pathfinding):**
- Dog is a screen-space overlay anchored below/behind the player, like the guard in Subway Surfers.
- `gap ≥ 12 m`: dog not on screen; HUD chase-meter shows a dog icon; periodic distant bark.
- `gap < 12 m`: dog rises from bottom edge over the player’s shoulder. Screen position/scale = `lerp(playerAnchor, bottomEdge, 1 − gap/12)`. Its `px` follows the player lane with a 0.33 s lag (pure lerp) — “almost got you” lunges.
- `gap ≤ 0.6`: lunge animation, slow-mo 0.3× for 1.2 s, game over.
- Proximity feedback: growl loop volume ∝ (12−gap)/12, red vignette at < 6 m, bark + vibration at < 3 m.

**Hard rules:** the dog never changes gameplay state except through `gap`; no pathfinding, no collision with obstacles; it is deterministic given player events.

### 6.4 Collision System

- **Broad phase:** entities are in one array; a moving pointer tracks the first entity with `z > −3`. Only entities in `[−3, +3]` are tested (≤ ~12 candidates).
- **Swept z:** if `prevZ > 0 && z < 0` (entity crossed the player plane this frame), evaluate overlap at crossing time using interpolated player `px, y`.
- **AABB in logical space:** lateral `|e.px − p.px| < e.halfW + 0.35`; depth `|e.z| < e.halfD + 0.4`; vertical: standing height 1.7, sliding 0.8, airborne `p.y`.
- **Resolution by severity** (table §7.2): apply dog penalty, stumble, despawn the hit entity (roll-past), trigger invulnerability, FX.
- **Near-miss:** when an obstacle crosses `z = 0` and lateral distance is `(halfW+0.35 … halfW+0.75)` → near-miss event (score +25, dog +0.5 rate-capped).
- **Coins/power-ups:** circle test radius 0.7 m; magnet active → coins within 8 m ahead lerp to player.

### 6.5 Spawning & Procedural Generation

**Structure:** fixed **50 m segments**. Each segment = (theme, template). Templates are **authored const data** (~70 templates across 6 themes × 5 difficulty bands) — designers add patterns without touching code.

```dart
class Placement { final double z; final int lanesMask; final EntityKind kind; }
class SegmentTemplate {
  final String id; final ThemeId theme; final int band; // 1..5
  final List<Placement> placements; final List<String> tags; // 'wall','oncoming','ped'
}
```

**Example:**

```dart
SegmentTemplate(
  id: 'trf_wall2_b2', theme: ThemeId.traffic, band: 2,
  placements: [
    Placement(z: 6,  mask: 0b011, kind: EntityKind.bus),      // blocks lanes 0+1
    Placement(z: 6,  mask: 0b100, kind: EntityKind.coinLine), // free lane rewarded
    Placement(z: 22, mask: 0b100, kind: EntityKind.car),
    Placement(z: 34, mask: 0b010, kind: EntityKind.rickshaw),
    Placement(z: 34, mask: 0b001, kind: EntityKind.motorcycle),
  ],
)
```

**Spawner algorithm:**

```
while (spawnedUntil < distance + 120):
  candidates = templates where theme == currentTheme && band <= currentBand(±1 allowed)
  filter: id != lastTemplateId
        && (tag 'wall' not in both current & previous if prev was 'wall')
  pick weighted random (higher bands weight up with difficulty)
  padStart = max(0, minGap(band) - (lastGateZ - firstGateZ of new template))
  instantiate placements from pools at z = spawnedUntil + padStart + placement.z
  spawnedUntil += 50 + padStart
```

**Difficulty rules by band:**

| Band | Reached at | Target speed | minGap between gates | Notes |
|---|---|---|---|---|
| 1 | 0 m | 9 m/s | speed·(0.80 + n·0.28) + 1.5 m | single obstacles, wide coin trails |
| 2 | 250 m | 10.5 | ↓ reaction 0.72 s | 2-lane blocks allowed |
| 3 | 500 m | 12.5 | reaction 0.66 s | moving traffic, pedestrians |
| 4 | 1000 m | 15 | reaction 0.60 s | mixed jump/slide chains |
| 5 | 1750 m | 18 → cap 21 | reaction 0.55 s | dense patterns, movers, cap reached at 3000 m |

**Fairness rules (hard guarantees):**

1. A **gate** = z-slice where ≥1 lane is blocked. Every gate must have ≥1 lane that is **free OR jump-clearable OR slide-clearable**. Max 2 lanes blocked by *hard* obstacles; the third, if blocked, must be low (jump) or overhead (slide).
2. Consecutive gates: `dz ≥ speedMax(band) × (reactionTime + laneChangesNeeded × 0.28) + 1.5 m`.
3. Templates are **validated at build time** (unit test) and the same validator runs as a `assert` in debug builds; runtime also checks **cross-template boundaries** (last gate of segment N vs first gate of N+1) and inserts `padStart` if violated.
4. **Movers:** moving vehicles never change lanes (fixed lane, slower/oncoming); they are telegraphed (headlights/blinkers 0.6 s). Only **pedestrians** cross lanes — and pedestrians are *minor* severity by design, so even a worst-case contact is survivable (fairness-critical objects are never soft-severity-ambiguous).
5. Power-ups/coins are only placed in lanes verified clear ±3 m.
6. **Fuzz harness guarantees** (see QA §14): a simulated perfect bot (450 ms reaction, 280 ms lane change) must take **0 hard hits** over 100 km of generated band-5 track.

**Theme switching:** every 700 m ± 200 (random), transition segment uses neutral decor; next theme’s atlas is prefetched asynchronously (never blocks the frame — see §10.4).

**Procgen performance:** selection is a precomputed per-theme filtered index + weighted pick — O(templates); placement is pool `obtain()` + struct writes. No parsing, no string ops at runtime.

### 6.6 Difficulty Progression

Single source of truth `DifficultyConfig` (const): speed curve, band thresholds, `pressureCap` curve, spawn weights, mover frequency. Progression is a pure function of `distance` → fully testable.

### 6.7 Score, Coins & Economy

- `score = floor(distance × 10) + coinsThisRun × 5 + nearMisses × 25 + comboBonus` (combo: every 10 near-misses/coins-streak without a hit → +250, resets on hit).
- Coins are awarded **at run end** (game over or revive-continue). Coin values: pickup 1 (2 with Multiplier), streak bonus 5 per 25.
- **Single soft currency.** Justification for *no premium currency*: nothing to buy except cosmetics; a second currency adds UI, IAP, and fraud surface with zero design benefit. The only real-money item is **Remove Ads** (convenience, not power).
- Price tuning: average coins/run ≈ 80–150 → cheap cosmetic ≈ 6–10 runs; missions add 150–450/day; ad gift 150/3 h. Cosmetic prices 500–5,000.

### 6.8 Power-Ups

| Power-up | Duration | Effect | Spawn rule |
|---|---|---|---|
| Shield | 8 s | Absorbs one hit (no dog penalty, no stumble), then breaks | every 300–500 m, one at a time, none before 200 m |
| Coin Magnet | 7 s | Coins within 8 m fly to player | weighted random |
| Speed Boost | 5 s | +40% speed; invulnerable to nothing (still dodge!); dog gains no ground | weighted random |
| Coin ×2 | 10 s | Each coin counts 2 | weighted random |
| Bone (Dog Repel) | 4 s | Dog stops to chew → gap grows ~4·speed (capped) | weighted random, rarest |

Pickup applies instantly; timers shown as shrinking HUD chips; effects never stack (picking same refreshes duration).

### 6.9 Missions (daily)

- 3 missions/day from a catalog of 14, **seeded by UTC date hash** (deterministic across restarts).
- Reset check on app start **and** on date change while app is open (timer at local midnight).
- Catalog (type, goal range, reward): collect coins (300/500 → 100/150), distance in one run (600/1200/2000), jumps (30), slides (25), near-misses (15), power-ups used (3), survive with dog < 5 m cumulative 20 s (200), pass obstacles (150), coin streak 25 ×2, runs played (3), score in one run (6000), collect a specific power-up ×2, finish a run in Market theme ≥ 400 m, use both jump & slide ≥ 15× each in one run.
- Completing all 3 → bonus 300 coins. Rewards claimed manually on Missions screen (drives retention re-entry).

### 6.10 Cosmetics

- **Characters (8):** Bilal (free/default), Zara, Chacha Rustam (chai vendor), Ayesha (student), Coach Karim, Faisal (courier biker), Marina (Kemari fisher), DJ Shah. Prices 800–4,000 coins; 2 unlock via missions instead of coins.
- **Dogs (6):** Street Doggo (free), Tommy, Sheru, Shadow, Bully (Bully Kutta), Rickshaw Racer (mission unlock). Prices 600–3,500.
- All rigs share identical frame layout (run 6f / jump 4f / slide 4f / stumble 3f; dog run 6f / lunge 4f) — new skins are pure data; bark pitch varies per dog (free variety).
- Cosmetics are **visual only**. No gameplay stats — not pay-to-win by construction.

### 6.11 Camera, FX, Animation

- Camera per §4.2. FX: hit flash, dust puffs (pooled circles, max 24 Low / 60 Mid), coin sparkle, speed lines during boost, dog vignette. All FX are one pooled particle array drawn in a single canvas pass.
- Player/dog animations are Flame `SpriteAnimationComponent`s sharing per-cosmetic `SpriteAnimation` data (one player + one dog exist → safe to share frames).
- Pedestrians use shared 2-frame flip + code-driven bob (no per-entity animation objects).

### 6.12 Audio

- `flame_audio`: SFX via `FlameAudio.pool(name, maxPlayers: 4)` — hard voice cap 4 to bound decode/CPU. Critical SFX preloaded at splash (coin, jump, hit, bark, power-up, UI click); the rest lazy-load on first use.
- Music: one adaptive track per theme layer set (menu + 2 intensity layers), streamed OGG ~112 kbps — never fully decoded in RAM.
- Master mute when paused/backgrounded/ad showing; ducking on focus loss.
- SFX catalog (18): jump, land, slide, coin, streak, power-up ×5, shield break, stumble, dog bark (proximity loop), growl loop, catch, game-over jingle, mission complete, unlock, UI click, revive.

### 6.13 Save / Load

- `save_v1.json` via `path_provider`, written **atomically** (write `.tmp`, rename) on: run end, purchase, unlock, setting change, mission claim, daily reset. `pendingRun.json` every 5 s during play.
- Load path: read → verify checksum (CRC of content, catches corruption, not tampering) → **validate & clamp every field** → migrate if older version → on any failure, try `.bak` (previous good copy kept on every successful write) → else defaults (never crash on bad save).
- `shared_preferences` only: `firstLaunchDone`, `consentFlowDone`, `saveVersionHint` (fast boot decisions).

### 6.14 Ads & IAP

**Inventory:** 1 interstitial unit, 1 rewarded unit. No banners, no app-open ads (app-open interrupts re-entry flow; not worth the UX cost for this genre).

**Rewarded placements & rules:**

| Placement | Limit | Notes |
|---|---|---|
| Revive on death | 1 per run | 5 s decision window; if ad fails to load → button shows “unavailable”, run ends normally (reward never granted without completed ad) |
| ×2 coins at results | 1 per run | Applied to that run’s coins only |
| Home “Gift +150 coins” | 1 per 3 h | Cooldown persisted locally |

**Interstitial rules (ALL must hold):**

- Run duration ≥ 60 s (never punish short sessions),
- ≥ 2 runs since last interstitial,
- ≥ 180 s since last interstitial,
- First 3 runs of a fresh install are a grace period (no interstitials),
- No rewarded ad shown in the last 90 s,
- Shown **after the results screen is dismissed, before the next run starts** — never between death and the revive decision, never mid-run.

**Consent:** UMP flow (via `google_mobile_ads`) before first ad request; if no consent → ads disabled entirely, game fully functional. `tagForUnderAgeOfConsent = false`; child-directed setting exposed in Settings for guardians.

**IAP:** “Remove Ads” non-consumable (Phase 2, interface exists from day one). Local flag is **not trusted**: on every launch (and on “Restore Purchases”), query Play `queryPurchases` and derive entitlement from the store response; local flag is only a cache.

**Robustness:** every ad call wrapped for `PlatformException`; load failures retry with exponential backoff (max 3); ads never block game threads (all callbacks marshaled to main isolate).

### 6.15 Analytics (optional, OFF by default)

`AnalyticsService` interface + `NoopAnalytics` default. If enabled in a build, `FirebaseAnalytics` impl plugs in. Events: `run_start`, `run_end{distance, score, deaths_by_band}`, `tutorial_funnel`, `ad_shown/ad_rewarded{placement}`, `purchase`. No PII; respects consent flag.

---

## 7. Karachi Environments & Obstacle Catalog

### 7.1 Themes (6, sequential cycling with random order after first pass)

| Theme | Inspired by | Signature decor (non-colliding) | Palette | Traffic flavor |
|---|---|---|---|---|
| Residential | Nazimabad streets | Boundary walls, kites overhead, water tanks, banyan trees, street lights | Warm noon | Parked cars, cyclists |
| Urban Arterial | Shahrah-e-Faisal | Billboards, flyover pillars, gantries | Cool grey-blue | Buses, fast cars |
| Bazaar | Saddar/Tower | Awnings, Urdu signboards, stacked crates, decor crowds | Warm amber | Rickshaws, carts, pedestrians |
| Commercial | I.I. Chundrigar | Office blocks, bank facades, flags | Desaturated steel | Taxis, motorbikes |
| Coastal | Sea View/Clifton | Sea + railing on one side, palms, sunset horizon | Orange/pink dusk | Families on decor bikes, slow traffic |
| Dense Traffic | Old City / W-11 route | Hanging wires, close facades, chaos | Gritty green-grey | W-11 style buses, oncoming motorcycles |

Atlas per theme ≤ 1024² (Low) / 2048² (Mid+). Max 2 theme atlases resident (LRU).

### 7.2 Obstacle catalog (logical sizes, all meters)

| Obstacle | halfW | halfD | Height/gap | Severity (dog −m) | Clearable | Notes |
|---|---|---|---|---|---|---|
| Car | 0.85 | 2.0 | 1.4 | **6 major** | no | static or same-direction mover |
| Oncoming car | 0.85 | 2.0 | 1.4 | 6 | no | band ≥ 3, headlights telegraph |
| Bus (2 lanes) | 1.9 | 3.5 | 2.6 | 6 | no | never 2 buses in one gate |
| Truck | 0.95 | 3.0 | 2.8 | 6 | no | |
| Rickshaw | 0.6 | 1.2 | 1.5 | 4 | no | |
| Motorcycle | 0.45 | 1.0 | 1.1 | 4 | no | |
| Low barrier | 0.85 | 0.3 | clearHeight 1.0 | 6 if not jumped | **jump** | |
| Overhead barrier | 0.85 | 0.3 | gapBottom 1.0 | 4 if not slid | **slide** | scaffolding pipe |
| Pothole | 0.6 | 0.6 | ground | 4 trip | **jump** | |
| Cone | 0.25 | 0.25 | 0.6 | **3 minor** | — | |
| Street cart | 0.7 | 0.8 | 1.4 | 3 minor | — | |
| Construction debris | 0.6 | 0.8 | 0.9 | 3 minor | jump also works | |
| Pedestrian (crossing) | 0.3 | 0.3 | 1.6 | 3 minor | — | moves 0.6 lane/s, zebra decor telegraphs |

Only **vehicles and barriers are “hard”** (6/4) and are the fairness-critical set; everything soft is ≤ 4 and never wall-forming.

---

## 8. Data Models (Dart)

```dart
enum ThemeId { residential, urban, bazaar, commercial, coastal, traffic }
enum Severity { minor, trip, major }
enum PowerUpKind { shield, magnet, boost, coinX2, bone }
enum Phase { loading, ready, countdown, playing, paused, dying, gameOver, results }

class PlayerState {
  double px;            // -1..1 continuous
  int lane;             // 0,1,2 target lane
  double y, vy;         // jump physics
  bool sliding; double slideTimer;
  double invulnTimer, stumbleTimer;
}

class DogState {
  double gap;                 // meters, 0..24
  double timeSinceLastHit, nearMissBonusBudget;
  int coinStreak;
}

class WorldEntity {          // pooled POD base
  bool active; EntityKind kind;
  double px, y, z, prevZ, halfW, halfD, height, clearHeight, gapBottom;
  Severity severity; double vz;        // movers
  int variantId;                       // sprite variant
  double animTime;                     // bob/spin
}

class RunStats {
  double distance; int score, coins, nearMisses, obstaclesPassed, jumps, slides;
  int powerUpsUsed; double dogCloseTime; Map<PowerUpKind,int> powerUpsByKind;
}

class Mission { String id; MissionType type; int goal, progress; int reward; bool claimed; }

class CosmeticItem { String id; String name; int cost; CosmeticSource source; /* mission id or coins */ }

class SettingsData {
  bool sfxOn, musicOn, hapticsOn;
  QualityTier tier;         // auto, low, medium, high
  FpsTarget fps;            // auto, t60, t30
  bool buttonMode;
}

class SaveData {
  int version;
  int coins;                         // clamped 0..999_999_999
  int bestScore; double bestDistance;
  // lifetime stats
  int totalRuns; double totalDistance; int totalCoinsEarned;
  int totalNearMisses, totalPowerUps, totalObstaclesPassed, dogEscapes;
  List<String> unlockedCharacters; String selectedCharacter;
  List<String> unlockedDogs;         String selectedDog;
  List<Mission> dailyMissions; String dailyMissionsDate;
  bool tutorialComplete;
  SettingsData settings;
  AdLocalState adState;              // graceRunsLeft, lastInterstitialAt, giftCooldownAt
  PendingRun? pendingRun;            // interrupted-run recovery
}
```

---

## 9. Local Storage Schema

`save_v1.json` (fields = `SaveData` above). Rules:

- **Validation on load:** type-check every field; clamp `coins/bestScore` to `[0, 999_999_999]`; unknown character/dog ids → fall back to defaults; mission date ≠ today → reroll missions; `dailyMissions.length == 3` else reroll.
- **Atomic write + rolling `.bak`**; CRC32 in envelope `{crc, payload}`.
- Version field + `SaveMigrations` map (`1 → 2 → …`) so future fields never break old saves.
- What is stored: coins, high score/best distance, lifetime stats, unlocks + selections, daily missions + progress, tutorial flag, settings, ad pacing state, pending run. **Nothing else** — no tokens, no IDs, no PII.

---

## 10. Asset Management & Performance Engineering

### 10.1 Budgets

| Metric | Low tier (≤ 3 GB RAM) | Mid tier |
|---|---|---|
| Frame time | ≤ 33 ms (30–60 fps) | ≤ 16.6 ms |
| Sim (steps 1–16) | ≤ 6 ms | ≤ 4 ms |
| Draw calls | ≤ 90 | ≤ 140 |
| App RAM | ≤ 220 MB | ≤ 320 MB |
| Decoded textures | ≤ 14 MB (2×1024² + common) | ≤ 30 MB |
| Active entities | ≤ 90 (decor density reduced) | ≤ 140 |
| Particles | 24 | 60 |
| Cold start → Home | ≤ 4 s | ≤ 2.5 s |

### 10.2 Tiering

`TierService`: animated splash runs a 1.5 s frame-time probe on first launch; rolling average ≥ 24 ms → Low tier. Stored; manual override in Settings. Tier affects: atlas resolution (512/1024/2048), decor density, particles, shadows, run-bob. **Never affects gameplay rules** (speed, gaps, dog math identical everywhere).

### 10.3 Atlas & textures

- One atlas per theme + one common atlas (player rigs, dogs, coins, power-ups, FX, HUD icons). Power-of-two sizes, Linear filtering; low-contrast road textures to hide aliasing; far sprites culled at `scale < 0.06`.
- All sprites reference shared `Sprite`/source-rect data — never per-entity images.

### 10.4 Loading strategy

- Splash: load common atlas + default theme + critical SFX. Remaining themes load lazily **in advance** (prefetch next theme when 150 m before a planned switch, async over multiple frames — `Images.load` is already async; we also stagger decode). Loading never blocks `update`.
- Asset manifest (const) maps asset → tier variant (`theme_traffic_low.png` vs `theme_traffic.png`).

### 10.5 GC discipline

- Zero allocations per frame: pooled entities, ring-buffer input, reused scratch math, HUD strings only on change (and score text at 10 Hz), sprite batches prebuilt.
- No `Map` iteration in hot path; entity iteration over flat `List<WorldEntity>` with `active` flag.
- Periodic `debug` build check: allocation-tracking test asserts 0 B/frame steady-state over 600 frames.

### 10.6 Battery

- 30 fps mode (Settings) via frame limiter (skip render on alternating vsyncs).
- Rendering stops when paused/backgrounded (Flame `pauseEngine()`); music stops; no timers run.
- No network calls during gameplay (ads only load in menus/results, preloaded for death placements).

---

## 11. UI Specification

All screens are Flutter widgets; the game screen hosts `GameWidget` + HUD as a Flame **overlay**. Safe-area aware; portrait-locked; tablets letterbox to max aspect 21:9 (gameplay width fixed → fair). Strings centralized in `strings.dart` (English default; Urdu-ready keys).

| Screen | Contents / rules |
|---|---|
| **Splash** | Logo, tier probe, loading bar (real progress), consent flow gate (before any ad init), first-launch → Tutorial |
| **Home** | Big Play, coins pill, buttons: Shop, Missions (badge), Characters, Dogs, Settings; Gift +150 (rewarded, cooldown); daily mission peek cards |
| **HUD (gameplay)** | Top: score (10 Hz), coins, pause; left: dog chase meter (0–24 m bar + dog icon color); power-up chips with radial timers; combo flash. Granular `ValueNotifier`s → only changed widgets rebuild |
| **Pause** | Resume, Restart (confirm), Settings shortcut, Home (confirm; run voided, coins still banked) |
| **Game Over** | Dying cinematic → Revive card (5 s countdown, rewarded, 1/run, disabled if used/unavailable) → Results: score, best-badge if record, coins +N, ×2 (rewarded), mission progress chips, Run Again / Home |
| **Shop** | Tabs: Characters / Dogs; cards with cost, owned/selected state, buy-with-coins (double confirm), mission-locked items show requirement |
| **Missions** | 3 daily cards (progress bar, claim button), reset countdown, all-3 bonus banner |
| **Character/Dog select** | Preview (idle animation), stats = “Cosmetic only” note, select/buy |
| **Settings** | SFX, Music, Haptics, Graphics tier (Auto/L/M/H), FPS (Auto/60/30), Button mode, Restore purchases, Privacy policy, Reset progress (double confirm) |
| **Tutorial** | First-run forced scripted 300 m: dog disabled (gap locked 18), sparse single-lane obstacles, contextual prompts per gesture (swipe L/R/up/down), completes → +200 coins; restarts if app killed mid-tutorial |

**Interstitial insertion point:** after Results dismiss → optional interstitial → Home or next run. Never during death/revive windows.

---

## 12. Project Structure

```
dog_runner/
├── pubspec.yaml
├── assets/
│   ├── images/
│   │   ├── common/            # common atlas.png + atlas.json, rigs: player_*, dog_*
│   │   └── themes/{residential,urban,bazaar,commercial,coastal,traffic}/
│   │       ├── atlas_low.png  atlas_mid.png  atlas.json (frames/anchors)
│   ├── audio/sfx/*.ogg        audio/music/*.ogg
│   └── fonts/
├── lib/
│   ├── main.dart
│   ├── app/  (app.dart, router.dart)
│   ├── core/
│   │   ├── config/   game_config.dart  difficulty_config.dart  economy_config.dart
│   │   │             ad_config.dart    quality_config.dart
│   │   ├── logging/  logger.dart  error_handler.dart
│   │   └── utils/    pool.dart  ring_buffer.dart  rng.dart  math_utils.dart
│   ├── data/
│   │   ├── models/   save_data.dart  run_stats.dart  mission.dart  cosmetic.dart
│   │   │             world_entity.dart  player_state.dart  dog_state.dart
│   │   ├── catalogs/ segment_templates.dart  obstacle_catalog.dart
│   │   │             powerup_catalog.dart  mission_catalog.dart  cosmetics_catalog.dart
│   │   └── save/     save_repository.dart  save_validator.dart  save_migrations.dart
│   ├── services/
│   │   ├── audio_service.dart   ad_service.dart   iap_service.dart
│   │   ├── analytics_service.dart (+ noop_impl.dart)
│   │   ├── settings_service.dart  tier_service.dart  services.dart (bundle)
│   ├── game/
│   │   ├── dog_runner_game.dart          # FlameGame root
│   │   ├── session/ game_session.dart  session_phase.dart  revive_flow.dart
│   │   ├── world/  world_state.dart  entity_store.dart  road_projection.dart
│   │   │           theme_manager.dart
│   │   ├── systems/ input_system.dart  player_system.dart  dog_chase_system.dart
│   │   │             movement_system.dart  spawn_system.dart  collision_system.dart
│   │   │             powerup_system.dart  score_system.dart  mission_system.dart
│   │   │             fx_system.dart  audio_cue_system.dart  autosave_system.dart
│   │   │             difficulty_system.dart
│   │   ├── entities/ entity_kinds.dart  pools.dart
│   │   ├── render/  backdrop_renderer.dart  world_renderer.dart
│   │   │             player_renderer.dart  dog_renderer.dart  hud_bridge.dart
│   ├── ui/
│   │   ├── screens/ splash, home, game_screen, pause, game_over, shop,
│   │   │             missions, character_select, dog_select, settings, tutorial_overlay
│   │   ├── widgets/  swipe_gate.dart  coin_pill.dart  mission_card.dart
│   │   │             cosmetic_card.dart  dog_meter.dart  powerup_chip.dart
│   │   └── strings.dart
├── test/
│   ├── unit/  dog_chase_test.dart  validator_test.dart  difficulty_test.dart
│   │          economy_test.dart  save_test.dart  save_migration_test.dart
│   ├── fuzz/  generator_fuzz_test.dart   # perfect-bot survivability, 100k segs
│   ├── widget/ home_test.dart  game_over_test.dart  settings_test.dart
│   ├── integration/ run_flow_test.dart  lifecycle_test.dart  ad_flow_mock_test.dart
│   └── perf/  frame_budget_test.dart  allocation_test.dart
└── tools/  build_atlas.sh  validate_templates.dart   # CI gate
```

---

## 13. Error Handling & Logging

- `runZonedGuarded` + `FlutterError.onError` → `Logger` (in-memory ring buffer, 200 lines, dumped to rotating file `logs/` on error, capped 256 KB).
- Game loop: each system wrapped in a debug-asserted boundary; in release, a system exception is caught once → system disabled + logged → run continues degraded (e.g., FX off) rather than crashing.
- Asset load failure → placeholder sprite + report; never a hard crash.
- Ad/IAP failures degrade per §6.14.
- Save corruption → `.bak` → defaults, with a one-time “progress reset” notice.

---

## 14. QA Strategy

### 14.1 Automated tests (CI gate)

| Suite | What it proves |
|---|---|
| `validator_test` | Every authored template passes the fairness validator at its band |
| `generator_fuzz_test` | 100,000 random segments (all bands): perfect bot (450 ms reaction, 280 ms lane change) takes **0 hard hits** over 100 km; soft-hit rate < 0.5% (pedestrians only) |
| `dog_chase_test` | Gap math: penalties, regen caps per band, catch at ≤ 0.6, power-up interactions, no NaN/stuck states |
| `economy_test` | Coin flow: run → award once; revive doesn’t duplicate coins; ×2 applies once; clamps |
| `save_test` / migration | Round-trip, corrupted JSON, truncated file, wrong types, old version migration, atomic-write kill simulation |
| `run_flow_test` (flame_test) | Full session: start → stumble → dog catch → results → restart; revive flow |
| `ad_flow_mock` | Frequency caps hold across 20 simulated runs; revive unavailable when ad fails; no interstitial in grace/short runs |
| `lifecycle_test` | Background/foreground during play → paused state, audio off, pendingRun recovery; no double coin award |
| `frame_budget_test` / `allocation_test` | Headless 10-min run: sim ≤ 6 ms/tick; 0 B/frame steady-state allocations |

### 14.2 Device matrix (physical)

- Low: 2 GB RAM, Android 9 (e.g., Redmi 9A class) — must hold 30+ fps Low tier.
- Low-mid: 3 GB, Android 10–11.
- Mid: 4–6 GB, Android 12–14 (60 fps target).
- High: flagship (battery/thermal check over 30 min).
- Screen: 18:9, 19.5:9, 20:9, 21:9; notch/punch-hole (safe areas); one tablet (letterbox correctness).

### 14.3 Manual test checklist (key areas)

- **Gameplay:** every obstacle × every action (jump/slide/lane) behaves per severity table; combos; power-up edge cases (shield during boost; picking bone while dog at 1 m).
- **Dog:** visible at gap < 12; catch during invulnerability still possible (rule); dog never visible during results.
- **Procgen (exploratory):** 2 h play sessions × 5 testers look for “feel” issues (too dense/sparse) at bands 4–5.
- **Ads:** test IDs in debug; airplane mode (all ad entry points degrade, no crashes, no reward without completion); consent decline path.
- **Interruptions:** incoming call, screen off, split-screen, battery-saver on, force-kill mid-run (pendingRun recovery), clock change (mission reroll accepted), timezone travel, midnight crossing during session.
- **Offline:** full game loop playable with network off; ads never block menus.
- **Save:** kill app during write; edit save to absurd values (must clamp); delete save file.
- **Tutorial:** first-run flow, skip app mid-tutorial, replay via Reset.
- **UI:** all screens at smallest/largest supported sizes; RTL pseudo-locale; text scaling off (game uses fixed-scale strings, verify no overflow).

### 14.4 Release gates

All automated suites green on CI + Low device meets frame budget in a 15-min soak + crash-free sessions ≥ 99.5% in closed beta.

---

## 15. Security & Trust Boundaries

**Never trusted (validated or ignored):**
- **Local save values** — everything clamped; unlocks validated against catalog; a tampered `coins=10^12` is clamped, not crashed. Accepted risk: local coin inflation is possible; the game is single-player with no trading/leaderboards, so the only victim is the player’s own fun. If leaderboards are ever added → server-authoritative scores mandatory (documented as a hard future requirement).
- **Local ad pacing state** — used only for UX pacing; not security-relevant.
- **Local Remove-Ads flag** — cache only; real entitlement re-derived from Play `queryPurchases` on every launch.
- **Ad SDK callbacks** — reward amounts are **app-defined constants**, never read from ad payload; rewards granted only on verified completion callback.
- **Daily-mission clock manipulation** — accepted risk (coins only, no competitive value).

**Not present:** no PII collection, no credentials, no custom network protocol, no deep links. Consent (UMP) precedes any ad request; analytics off by default.

---

## 16. Self-Review Loop (performed; fixes integrated above)

### Pass 1 — full design audit

| # | Finding | Severity | Fix (already integrated) |
|---|---|---|---|
| F1 | Original design had one Flame component per entity → add/remove churn, per-frame priority sorting, GC pressure on low-end | **Critical** | Pooled POD entities + single custom renderer (§4.3, §10.5); only player/dog are Flame components |
| F2 | Dog projected on road plane at negative z → division blow-up / dog behind camera and invisible at large gap | **Critical** | Dog redefined as screen-space chaser visible < 12 m + HUD meter beyond (§6.3) — classic, cheap, readable |
| F3 | Moving obstacles that change lanes could form 3-lane walls → impossible situations | **Critical** | Movers fixed to lane + telegraph; only pedestrians (minor severity) cross lanes; validator + fuzz harness guarantee 0 unavoidable hard hits (§6.5, §14.1) |
| F4 | Fixed-timestep loop was unnecessary complexity | Major | Replaced with clamped variable `dt` + swept z-collision (§5, §6.4) |
| F5 | Dog could catch during post-hit invulnerability ambiguity | Major | Rule made explicit: invulnerability blocks obstacle penalties only; dog catch check runs every frame (§6.3) |
| F6 | Revive could re-award coins or re-hit the killing obstacle | Major | Revive contract: clear 25 m of obstacles **and coins**, reset dog, 2 s shield (§6.1) |
| F7 | Interstitial right after a rewarded revive = ad fatigue | Major | 90 s rewarded-ad suppression rule (§6.14) |
| F8 | App kill mid-run loses collected coins (bad feel) | Major | `pendingRun` snapshot every 5 s + recovery dialog; double-award prevented by clearing on award (§3.3, §6.13) |
| F9 | All 6 theme atlases resident → memory blow-up | Major | LRU, max 2 resident, async prefetch before theme switch (§7.1, §10.4) |
| F10 | `device_info` plugin added just for tiering | Minor | Replaced with first-launch frame benchmark + manual override — one less dependency (§10.2) |
| F11 | Score/coin double-count with ×2 multiplier ambiguous | Minor | Formula fixed: ×2 doubles coin pickups only (coins and their score contribution), applied once (§6.7) |
| F12 | Cross-template gate spacing could violate minGap at segment boundaries | Major | Runtime boundary check + `padStart` insertion (§6.5 rule 3) |

### Pass 2 — re-review of corrected design

| # | Finding | Severity | Fix |
|---|---|---|---|
| F13 | HUD rebuilding every frame (score text) would cause widget rebuilds | Minor | Granular `ValueNotifier`s + 10 Hz score throttle (§5 step 14, §11) |
| F14 | SFX voice explosion (many coins at once) on low-end | Minor | `maxPlayers: 4` pools (§6.12) |
| F15 | Unbounded entity z-precision / distance overflow concern | None | Double precision safe to 2⁵³; entities recycle at −20 m; documented (§4.1) |
| F16 | Coin values/mission rewards not tuned against cosmetic prices | Minor | Tuning table added: ~6–10 runs per cheap cosmetic (§6.7) |

Verified in this pass: all Pass-1 fixes consistent across sections; state machine covers every lifecycle path (background → pause → resume → pause menu; kill → pendingRun; ad fail → degrade); dog mechanic is closed-form (deterministic function of events + time, no emergent states); validator covers build-time and runtime paths.

### Pass 3 — final audit

- **Complexity:** 6 runtime dependencies, one JSON save file, no backend, no DI framework, no code generation. Minimal for the feature set. ✅
- **Implementability:** every algorithm (projection, dog math, validator, spawner, pooling) is concrete Dart with defined constants; all Flame APIs used are 1.x stable. ✅
- **Low-end feasibility:** flat arrays, O(n) near-sorted draw, ≤ 90 draw calls, 14 MB textures, zero per-frame allocation, 30 fps fallback. ✅
- **Impossible-pattern prevention:** proven by validator + 100 km bot fuzz (hard guarantee on hard obstacles; soft obstacles structurally non-fatal). ✅
- **Dog meaningfulness:** sole death condition, difficulty-scaled pressure cap, skill-based pushback, power-up synergy, full audio/visual feedback ladder. ✅
- **Storage/monetization/lifecycle/ads:** all edge cases enumerated with defined behavior. ✅

**Verdict: no critical architectural issues remain.** Residual known limitations (accepted, documented): local-save tampering possible (accepted for single-player), no cloud save, Urdu localization deferred, Remove-Ads IAP phased.

---

## 17. Build Order (implementation phases)

1. **P0 – Walking skeleton:** loop, projection, road, player, input, one obstacle, dog math + screen-space dog, collision, game over/restart, save v1.
2. **P1 – Full game:** template library + validator, 3 themes, coins, power-ups, difficulty, HUD, missions, cosmetics, tutorial, audio.
3. **P2 – Ship candidate:** all 6 themes, ads (rewarded + interstitial + UMP), perf pass on Low device, QA matrix, beta.
4. **P3:** Remove-Ads IAP, Urdu strings, optional analytics build.

This design is directly implementable: every system has defined data structures, algorithms, constants, state transitions, and test gates.