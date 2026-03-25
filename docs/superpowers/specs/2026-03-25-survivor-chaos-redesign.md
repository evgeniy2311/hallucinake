# Survivor-Chaos Redesign

## Problem

Hallucinake's pacing rewards survival length but punishes short runs. Tier-2 powers unlock at snake length >= 8, special spawn rates are low early, and obstacles scale with trip level — meaning most runs end before the interesting systems kick in. The first minute should be the hook, not a slow ramp to fun.

## Approach

All changes go into the existing `index.html` monolith. A new `RunDirector` object manages time-based pacing, bonus economy, field events, and rescue drops. Estimated addition: ~700-900 lines.

**Timer convention:** All Director timers use wall-clock milliseconds (`Date.now()` deltas), not game ticks. Existing game systems (obstacle timers, effect durations) remain tick-based. The Director bridges these worlds by converting ms thresholds into spawn calls within `step()`.

## System 1: Chaos Director (`RunDirector`)

A state object created at `init()`, ticked every `step()`.

### State
All timers are wall-clock milliseconds (Date.now deltas), not game ticks.
```
startMs              — Date.now() at run start
elapsedMs            — real time since run start
phase                — 'warmup' | 'rampup' | 'chaos'
prevPhase            — previous phase (for transition alpha)
phaseTransitionAlpha — 0-1, for HUD fade
eventTimerMs         — ms until next field event
activeEvent          — null | {type, phase:'telegraph'|'active', startMs, data}
pityMs               — ms since last special pickup
guaranteedT1Ms       — ms at which a tier-1 special is force-spawned
guaranteedT2Ms       — ms at which a tier-2 special is force-spawned
t1Spawned            — boolean, has guaranteed T1 been spawned
t2Spawned            — boolean, has guaranteed T2 been spawned
eventCharge          — 0-5, accumulated from surviving events
lastEventMs          — ms timestamp of last event end (for charge decay)
eventsWithoutDrop    — count of events survived without getting a special
```

### Phase Transitions
| Phase | Time | Trigger |
|-------|------|---------|
| warmup | 0-25s | Run start |
| rampup | 25-60s | elapsedMs >= 25000 |
| chaos | 60s+ | elapsedMs >= 60000 |

### Per-Tick Logic
1. Update `elapsedMs` from `Date.now() - startMs`, determine `phase`, update transition alpha
2. Check guaranteed drop timers (ms-based), force-spawn if due and not yet spawned
3. Update `pityMs`, force-spawn if > `PITY_MS`
4. If no `activeEvent`: decrement `eventTimerMs`, trigger new event if <= 0 and phase != warmup
5. If `activeEvent`: tick event phases (telegraph → active → resolve → null)
6. Decay `eventCharge` if `Date.now() - lastEventMs > CHARGE_DECAY_MS`

## System 2: Early Game Softening

### Speed Curve
The multiplier is applied to the **delay interval** returned by `spd()`. Higher multiplier = longer delay = slower snake.
- `warmup`: delay multiplied by 1.2 (snake moves ~17% slower)
- `rampup`: linear interpolation from 1.2x to 1.0x over 35s
- `chaos`: no multiplier (existing formula)

Applied inside `spd()` as: `return Math.floor(baseResult * director.speedMultiplier())`.

### Obstacle Delay
- `spawnObs()` returns empty array during `warmup` phase
- `rampup`: obstacle count = existing formula * `rampProgress` (0.0 to 1.0)
- `chaos`: unchanged

### No Lethal Pressure Before First Special
- During `warmup`, obstacles are suppressed entirely
- First guaranteed special arrives at 8-12s (random, set at init)

## System 3: Bonus Economy 2.0

### Guaranteed Drops
- **Tier-1**: Force-spawn at 8-12s (randomized at run start). Bias toward survival powers: ghost (30%), portal (25%), crystal (20%), others (25%)
- **Tier-2**: Force-spawn at 35-45s (randomized) OR when snake.length >= 8, whichever comes first
- Hard cap: max 4 specials on screen at any time (including guaranteed, rescue, and normal spawns). If cap reached, guaranteed drops replace the oldest existing special.

### Tier-2 Unlock
Current: `snake.length >= 8` only.
New: `snake.length >= 8` OR `elapsedMs >= 35000` — whichever comes first.

### Pity System
- `pityMs` tracks wall-clock milliseconds since last special pickup (consistent with Director's ms-based timers)
- At `pityMs > 20000` (20 seconds real time): force-spawn a random tier-1 special
- Counter resets on any special pickup

### Early Drop Bias
During `warmup` and first half of `rampup`, special spawn pool weighted:
- ghost: 30%, portal: 25%, crystal: 20% (survival)
- mushroom, rainbow, fire, speed, eye: split remaining 25%

## System 4: Field Events

### Scheduling
- First event: 25-30s (start of rampup)
- Subsequent events: every 12-18s (random interval)
- `warmup`: no events
- Each event has a telegraph phase (visual warning) before becoming lethal

### Event Types

#### Pulse Walls
- **Telegraph (2s):** Border cells flash magenta
- **Active (5s):** Playfield shrinks by 2 cells on each side (effective grid: COLS-4 x ROWS-4). Outer cells become lethal (unless ghost). Gaps (2-cell wide passages) visible on each wall.
- **Reward:** +30 score for surviving
- **Visual:** Pulsing magenta border closing in, gaps glow cyan

#### Rift Bloom
- **Telegraph (1.5s):** 3-5 circular zones glow with expanding rings
- **Active (4s):** Zones become lethal (2x2 cell areas). Center of each zone has a bonus food worth 3x points.
- **Reward:** Bonus food if collected (risk/reward)
- **Visual:** Purple/red pulsing circles, gold center dot

#### Phantom Sweep
- **Telegraph (2s):** Ghost line appears at edge, semi-transparent
- **Active (3s):** Horizontal or vertical line sweeps across field at ~2 cells/second. Lethal on contact (unless ghost).
- **Reward:** +20 score, eventCharge += 1
- **Visual:** Cyan laser line with particle trail

### Lethality Rules
- All events telegraph before becoming lethal
- Ghost effect grants immunity to all event damage
- Pill invincibility grants immunity to all event damage
- During `rampup`: events deal damage but don't kill (reduce to 1-cell snake instead)

### Damage Resolution (rampup truncation)
When an event truncates the snake to 1 cell during rampup:
- **Active effects:** preserved (timers continue ticking)
- **Trip level:** preserved (no reset)
- **Score:** preserved
- **Expanded field (portal):** if head is within base grid, keep expanded. If head is outside base grid, warp head to `head % BASE` before truncation.
- **Visual feedback:** screen flash + shake to signal damage

### Event Overlap Rules
- Only one event can be active at a time
- `eventTimer` does not decrement while an event is active (telegraph or active phase)
- Next event scheduling begins after current event fully resolves

### Rift Bloom Placement
- Zones must be >= 5 cells from snake head (consistent with obstacle spawn buffer)
- Zones must not overlap each other
- If valid placement cannot be found in 50 attempts, reduce zone count

### Phantom Sweep Speed
- Sweep line crosses the full field width/height during the 3s active window
- Position is time-interpolated in `draw()` using `(Date.now() - sweepStartMs) / 3000 * fieldSize`
- Not stepped — smooth continuous movement

## System 5: Rescue Drops & Event Charge

### Rescue Drops
- Trigger: `eventsWithoutDrop >= 3` after an event ends
- Spawns a utility special: ghost (50%) or portal (50%)
- Resets `eventsWithoutDrop` to 0

### Event Charge (0-5)
- +1 per event survived
- Decay: -1 every 20s without an event (minimum 0)
- Bonuses:
  - Score multiplier: `1 + eventCharge * 0.1` (up to +50%)
  - Special spawn chance formula: `Math.random() < 0.4 + trip*0.3 + director.eventCharge * 0.15`

## System 6: HUD & Readability

### Phase Label
- Position: top-left, below score
- Font: Orbitron, 10px
- Colors: warmup=#0f0, rampup=#ff0, chaos=#f00
- Text-shadow matching color
- Fades between phases using `phaseTransitionAlpha` (0→1 over 1s, tracked in Director, interpolated in draw)

### Event Warning Banner
- Position: center screen, above play field
- Shows 2s before event activates
- Text: event name in caps (e.g., "PULSE WALLS")
- Style: large Orbitron, flashing opacity (pulse animation), color matching event
- Fades out when event activates

### Guaranteed Drop Hint
- When guaranteed drop timer is within 3s of firing:
- Subtle pulsing glow on screen border (cyan, 2% opacity sine pulse)
- No text — just ambient anticipation signal

### Event Charge Display
- Rendered in HTML HUD, appended to `#hE` span as colored unicode dots (consistent with existing effects display)
- Each dot = 1 charge, colored by intensity (green → yellow → red)
- Appears only when eventCharge > 0

### Help Panel Updates
- Add section for field events (Pulse Walls, Rift Bloom, Phantom Sweep) with brief descriptions
- Add note about phases (warmup → rampup → chaos)

### Touch Input Fix
- Apply `invertControls` check to touch handler (pre-existing bug, fixed as part of this work since pill effect is redesigned)

## Integration Points

### `init()` Changes
- Create `RunDirector` instance with randomized timers
- Reset all director state

### `step()` Changes
- Call `director.tick()` at start of step
- Replace obstacle spawn check with director-gated version
- Replace special spawn logic with director-enhanced version
- Add event tick logic

### `spd()` Changes
- Apply `director.speedMultiplier()` to final speed

### `draw()` Changes
- Add phase label rendering
- Add event warning banner rendering
- Add event visual effects (pulse walls, rift zones, sweep line)
- Add guaranteed drop hint glow
- Add event charge dots

### Collision Changes (line 364-366)
- Add event collision checks (pulse walls, rift zones, sweep line)
- During `rampup`: event damage truncates snake instead of killing

## Constants

```javascript
const DIRECTOR = {
  WARMUP_MS: 25000,
  RAMPUP_MS: 60000,
  WARMUP_SPEED_MULT: 1.2,
  GUARANTEED_T1_MIN: 8000,
  GUARANTEED_T1_MAX: 12000,
  GUARANTEED_T2_MIN: 35000,
  GUARANTEED_T2_MAX: 45000,
  PITY_MS: 20000,
  EVENT_MIN_INTERVAL: 12000,
  EVENT_MAX_INTERVAL: 18000,
  EVENT_FIRST_DELAY: 25000,
  RESCUE_THRESHOLD: 3,
  CHARGE_MAX: 5,
  CHARGE_DECAY_MS: 20000,
  CHARGE_SCORE_BONUS: 0.1,
  CHARGE_SPAWN_BONUS: 0.15
};
```
