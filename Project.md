# Project — GZDoom Playsim World-Machine Specification

## WORLD

### Purpose

`playsim` is the gameplay simulation component of GZDoom. It advances the state of the game world independently of rendering, audio, input hardware, network transport, and wall-clock time. It is the part of the engine responsible for actors, movement, collision, damage, scripts, sector effects, AI, line specials, and other gameplay-state changes.

The simulation runs at a fixed rate of **35 tics per second**. A tic is the indivisible logical time unit of the simulation.

### World Assumptions

The world that `playsim` operates on has the following properties.

#### Map Geometry Is Loaded Before Simulation

Map geometry is loaded from WAD/map data before the first tic runs. This includes:

- `sector_t`
- `line_t`
- `side_t`
- `vertex_t`
- blockmap data
- level assets and metadata

The WAD is not reloaded during normal simulation. Runtime logic reads and mutates the in-memory level structures.

#### Map Geometry Is Mostly Static

`line_t`, `vertex_t`, and `side_t` records generally do not change shape after level load.

The main mutable sector properties are:

- `sector_t::floorplane`
- `sector_t::ceilingplane`
- `sector_t::lightlevel`
- active sector-effect pointers such as `floordata`, `ceilingdata`, and `lightingdata`

Polyobjects are the exception: they move by modifying the positions of referenced `vertex_t` records in-place.

#### Actors Occupy 2.5D Space

Actors are represented as vertical cylinders.

An actor has:

- an XY position
- a radius
- a height
- a floor contact point
- floor and ceiling constraints

Collision is not true volumetric 3D collision. Actors can visually overlap at different heights, but collision logic is based primarily on 2D cylinder overlap plus floor/ceiling limits.

#### Time Is Integer Tic-Based

All simulation durations are measured in integer tics.

Examples:

- actor state durations
- AI wait times
- ACS delays
- weapon animation timing
- sector mover waits
- countdown timers

Floating-point seconds and wall-clock time are outside the simulation boundary.

#### Randomness Is Deterministic

Simulation randomness comes from deterministic PRNG instances, such as `M_Random` / named `FRandom` streams. Randomness is order-dependent: changing thinker execution order changes the sequence of random draws and therefore changes the future state.

#### Inventory Is Actor-Based

Inventory items are also `AActor` instances. There is no completely separate item object model. A weapon, key, powerup, monster, corpse, or player pawn is distinguished by class type, state, flags, and statnum.

#### Sector Effects Are Sector-Owned

A sector owns pointers to active floor, ceiling, and lighting effects.

Important consequence:

- a sector can normally have only one active floor mover at a time
- a sector can normally have only one active ceiling mover at a time
- starting a conflicting mover is ignored or consumed rather than allowing two movers to fight over the same sector plane

---

## MACHINE

### System Boundary

`playsim` is the machine that consumes per-tic inputs and advances the in-memory game state.

It does **not** directly render frames, mix audio, save files, read keyboard hardware, or transmit network packets.

```text
                         ┌─────────────────────────────────────┐
usercmd_t per player ───►│                                     │
                         │              playsim                │
map geometry ───────────►│                                     │
                         │  thinkers + actors + map state      │
actor definitions ──────►│  state machines + physics + scripts │
                         │                                     │
random seed ────────────►│                                     │
                         └─────────────────────────────────────┘
                                           │
                      ┌────────────────────┼────────────────────┐
                      ▼                    ▼                    ▼
                renderer reads       demo system records   save system
                game state           input stream          serializes state
```

### Inputs

| Input | Representation | Frequency | Source |
|---|---:|---:|---|
| Player commands | `usercmd_t` | Every tic, per player | Game loop / netplay / demo playback |
| Map geometry | `sector_t`, `line_t`, `side_t`, `vertex_t` arrays | Once at level load | WAD/map loader |
| Actor and behavior definitions | ZScript / DECORATE class registry | Startup / load-time | Scripting and actor-definition layer |
| Random seed | integer / PRNG state | Level start / save-load restore | Game initialization or serialized state |

### Outputs

`playsim` produces no explicit return object. Its output is the mutated in-memory game state.

External systems read that state independently.

| Consumer | Reads | Timing |
|---|---|---|
| Renderer | actor positions, actor angles, sector planes, light levels, player camera actor, HUD weapon sprites | every rendered frame |
| Audio system | actor/sector positions and sound events | as needed by the audio layer |
| Demo system | the `usercmd_t` stream | every tic while recording |
| Save system | full thinker and actor state | on player save |
| Network/game sync systems | deterministic simulation state implied from identical inputs | per tic / protocol-dependent |

### Core Machine Loop

`P_Ticker()` is the per-tic entry point.

A simplified sequence is:

```text
P_Ticker():
    1. update particle thinkers / particle-side simulation hooks
    2. for each in-game player:
           P_PlayerThink(player)
    3. run world-level event hook
    4. run level-level bookkeeping
    5. run all registered thinkers
    6. update map specials
```

The important ordering rule is that player commands are processed before normal thinkers. Enemies, physics helpers, sector movers, scripts, and other thinkers therefore observe the player state after player input has been applied for that tic.

### Thinkers

A thinker is a registered simulation object that can participate in ticking, serialization, or lifecycle management.

Thinking objects receive per-tic execution through `Tick()`.

Non-thinking objects are still registered so they can be serialized and managed, but they do not perform meaningful per-tic work.

Examples of thinking systems:

- actors
- monsters
- projectiles
- sector movers
- ACS scripts
- bots
- scrollers
- light effects
- earthquakes
- visual thinkers

Examples of non-thinking or mostly static registered state:

- decals
- corpse references
- travelling actors
- persistent objects needed by save/load

### Thinker Collection

`FThinkerCollection` maintains statnum-indexed intrusive lists.

Conceptually:

```cpp
FThinkerList Thinkers[MAX_STATNUM + 2];
FThinkerList FreshThinkers[MAX_STATNUM + 1];
```

Every thinker belongs to a statnum bucket.

A thinker stores:

- `_statNum`
- `NextThinker`
- `PrevThinker`
- pointer to owning `FLevelLocals`

New thinkers are first inserted into `FreshThinkers`, not directly into active `Thinkers`. This prevents a newly spawned thinker from ticking in the same tic in which it was created.

### Thinker Execution

Thinkers execute by ascending statnum order.

The thinking range begins at `STAT_FIRST_THINKING` and continues to `MAX_STATNUM`.

Important statnums:

| Statnum | Constant | Meaning |
|---:|---|---|
| 32 | `STAT_SCROLLER` | scrolling floors, ceilings, walls |
| 33 | `STAT_PLAYER` | player actors |
| 35 | `STAT_LIGHTNING` | sky lightning |
| 36 | `STAT_DECALTHINKER` | animated wall decals |
| 37 | `STAT_INVENTORY` | inventory actors |
| 38 | `STAT_LIGHT` | sector lighting effects |
| 39 | `STAT_LIGHTTRANSFER` | light transfer effects |
| 40 | `STAT_EARTHQUAKE` | camera/sector earthquake effects |
| 70–90 | `STAT_USER` range | user-defined ZScript thinkers |
| 100 | `STAT_DEFAULT` | default actors |
| 101 | `STAT_SECTOREFFECT` | floor, ceiling, door, platform movers |
| 102 | `STAT_ACTORMOVER` | actor movement helpers |
| 103 | `STAT_SCRIPTS` | ACS script thinker |
| 104 | `STAT_BOT` | bot AI |
| 105 | `STAT_VISUALTHINKER` | visual-only thinkers |

Simplified algorithm:

```text
RunThinkers():
    for statnum from STAT_FIRST_THINKING to MAX_STATNUM:
        for thinker in Thinkers[statnum]:
            call PostBeginPlay if needed
            call Tick

    repeat:
        moved = false
        for statnum from 0 to MAX_STATNUM:
            move all FreshThinkers[statnum] into Thinkers[statnum]
            moved = true if anything moved
    until moved == false
```

The fresh-thinker drain can repeat because `PostBeginPlay()` may spawn additional thinkers.

### Actor State Machine

Actor behavior is driven by linked `FState` records.

A state contains:

```cpp
struct FState {
    FState      *NextState;
    VMFunction  *ActionFunc;
    int32_t      sprite;
    uint8_t      Frame;
    int16_t      Tics;
    uint16_t     TicRange;
    uint16_t     StateFlags;
};
```

Per-tic actor state advancement:

```text
AActor::Tick():
    if tics > 0:
        decrement tics
        remain in current state

    if tics == 0:
        follow state->NextState

        if NextState is null:
            destroy actor

        otherwise:
            set new state
            load new tic duration
            call state's action function
```

State labels such as `Spawn`, `See`, `Melee`, `Missile`, `Pain`, `Death`, and `Raise` are resolved to `FState*` pointers. Runtime execution follows pointers rather than repeatedly performing string lookup.

### Input Pipeline

The simulation receives one `usercmd_t` per player per tic.

```cpp
struct usercmd_t
{
    uint32_t buttons;
    short    pitch;
    short    yaw;
    short    roll;
    short    forwardmove;
    short    sidemove;
    short    upmove;
};
```

The command is stored in `player_t::cmd`.

`P_PlayerThink()` reads this command and applies it to the player actor.

Simplified sequence:

```text
P_PlayerThink(player):
    1. read player->cmd
    2. preserve original command if needed
    3. run PlayerPawn.PlayerThink()
    4. convert movement command into velocity / movement attempt
    5. update view height, bobbing, crouch, death/respawn logic
```

Bots synthesize commands into the same `player_t::cmd` structure. From the rest of the simulation's perspective, bot input and human input are equivalent.

### Physics and Collision

Movement and collision use map geometry, blockmap acceleration, and actor cylinder checks.

Primary responsibilities:

- test actor movement against walls
- test actor movement against other actors
- compute floor and ceiling constraints
- allow sliding along blocking lines
- trigger line specials when crossed
- handle portals and cross-sector movement
- update blockmap links after movement

Simplified movement:

```text
P_TryMove(actor, newX, newY):
    check position against lines and actors
    compute floorz and ceilingz
    if blocked:
        attempt slide movement
    if accepted:
        unlink from old blockmap cells
        update position
        relink into new blockmap cells
        update sector/floor/ceiling context
        activate crossed line specials
```

### Blockmap

The blockmap is a grid acceleration structure with 128×128 map-unit cells.

It is used for:

- collision
- sight checks
- line traces
- hitscan attacks
- radius damage
- actor proximity checks

Instead of searching the whole map, spatial queries inspect only the cells overlapping the query region.

### Damage and Interaction

The simulation resolves:

- hitscan traces
- projectile impacts
- splash damage
- melee attacks
- actor contact interactions
- special line activation
- sector damage and environmental effects

Damage modifies `AActor::health`.

If health reaches zero or below, the actor transitions into death behavior.

### Line Specials

A line special is activated when an actor interacts with a tagged `line_t`.

Activation types include:

| Activation | Trigger |
|---|---|
| Cross | actor crosses the line |
| Use | player presses use against the line |
| Shoot | hitscan/projectile strikes the line |
| Push | actor physically pushes the line |

These converge on line-special dispatch, which reads:

- `line_t::special`
- `line_t::args[5]`

The dispatcher often constructs a sector-effect thinker such as:

- `DDoor`
- `DFloor`
- `DPlat`
- `DCeiling`
- `DAnimatedDoor`

### Sector Effects

Sector movers follow a repeated machine pattern.

```text
constructor:
    register sector-effect thinker
    attach to sector pointer
    record start plane and target plane
    register interpolation if needed

Tick():
    move plane toward target
    if target reached:
        wait, reverse, change state, or destroy self
```

Typical hierarchy:

```text
DSectorEffect
  └── DMover
        ├── DMovingFloor
        │     ├── DFloor
        │     └── DPlat
        └── DMovingCeiling
              ├── DCeiling
              ├── DDoor
              └── DAnimatedDoor
```

### Scripting

`playsim` integrates ACS and ZScript.

#### ACS

ACS scripts are managed by `DACSThinker`, which runs under `STAT_SCRIPTS`.

A running ACS script stores:

- program counter
- stack
- locals
- wait condition
- script number / identity

Each tic, non-waiting scripts execute bytecode until they reach a delay, wait, terminate, or blocking condition.

#### ZScript

ZScript functions called by actors or events run synchronously inside the current tic. They return before the next thinker executes unless they explicitly create future work through a thinker, state change, delay, or script mechanism.

### Serialization Machine

The save system serializes the full thinker graph.

Each serializable object implements:

```cpp
void Serialize(FSerializer &arc);
```

The same method handles writing and reading depending on archive mode.

Simplified save:

```text
SerializeThinkers():
    for statnum from 0 to MAX_STATNUM:
        for thinker in Thinkers[statnum]:
            write class identity
            write serialized fields
```

Simplified load:

```text
LoadThinkers():
    clear existing thinkers
    read archive entries
    construct object by class identity
    deserialize fields
    restore thinker list membership
    resolve saved object references
```

Non-thinking thinkers are serialized because they may contain persistent gameplay or visual state even if they do not tick.

### Renderer Relationship

The renderer is not driven by messages from `playsim`.

It reads the memory that `playsim` mutates, including:

- actor position
- actor angle
- actor sprite state
- sector floor and ceiling planes
- sector light level
- player camera actor
- player HUD weapon sprites

The renderer may run more often than the simulation. When it does, it interpolates between previous and current tic states. Interpolation is a presentation concern, not a simulation concern.

### Demo Relationship

The demo system records the input stream, not the full world state.

During recording, it stores `usercmd_t` values.

During playback, it feeds the same command sequence back into `playsim`. Determinism makes the same world state emerge from the same initial state and same input sequence.

### Save Relationship

Unlike demos, saves store complete state.

A save captures:

- thinkers
- actors
- positions
- sectors
- scripts
- inventory
- doors
- platforms
- lights
- object references
- other live simulation state

A loaded save resumes from the serialized state rather than replaying input history.

### 3D Floors

A sector can reference an extended sector structure containing 3D floor data.

Conceptual structure:

```cpp
struct extsector_t {
    struct xfloor {
        TDeletingArray<F3DFloor*> ffloors;
        TArray<lightlist_t>       lightlist;
        TArray<sector_t*>         attached;
    } XFloor;
};
```

An `F3DFloor` references:

- a defining line
- a model sector
- a target sector
- top and bottom planes

The 3D floor renders inside the target sector but gets its geometry from the model sector. Moving a 3D floor is usually implemented by moving the model sector's planes.

### Polyobjects

A polyobject owns references to existing level geometry arrays.

It stores pointers to:

- `side_t`
- `line_t`
- `vertex_t`

It moves by mutating referenced vertex positions in-place and relinking the geometry into the blockmap.

Simplified movement:

```text
MovePolyobj(delta):
    check actor blocking
    move referenced vertices
    update bounding box
    relink in blockmap
    crush or block actors if required
```

---

## REQUIREMENTS

### Functional Requirements

#### FR-01 — Tick Advancement

The simulation shall advance by exactly one tic per call to `P_Ticker()`.

A single tic shall be treated as an indivisible unit of simulation time.

#### FR-02 — Fixed Tick Rate

The simulation shall be designed around a fixed logical rate of 35 tics per second.

The outer game loop may decide when to call `P_Ticker()`, but the simulation itself shall measure time only in tics.

#### FR-03 — Player Input Consumption

For every in-game player, the simulation shall consume one `usercmd_t` per tic.

The command shall be read from `player_t::cmd`.

#### FR-04 — Player-First Ordering

Player thinking shall run before ordinary thinker execution in the same tic.

This ensures that AI, scripts, physics helpers, and other thinkers observe the player's updated state for that tic.

#### FR-05 — Deterministic Thinker Execution

Thinkers shall execute in ascending statnum order.

Thinkers in the same statnum bucket shall execute in stable list order for that tic.

#### FR-06 — Fresh Thinker Deferral

A thinker created during a tic shall not receive its first ordinary tick until a later tic.

Fresh thinkers shall be held separately before being merged into active thinker lists.

#### FR-07 — Non-Thinking Thinker Persistence

Non-thinking thinkers shall be registered for lifecycle and serialization purposes even if they do not receive ordinary per-tic `Tick()` calls.

#### FR-08 — Actor State Execution

Every actor shall maintain a current state pointer.

The simulation shall decrement the actor's state tic counter and transition to the next state when the counter expires.

When entering a new state, the actor shall execute that state's action function.

#### FR-09 — Actor Lifecycle

The simulation shall support actor spawning, destruction, morphing, teleportation, inventory ownership, and state transitions.

Actor destruction shall remove the actor from active simulation participation while respecting the engine's object lifetime model.

#### FR-10 — Movement Resolution

The simulation shall resolve actor movement against map geometry and actor collision.

Movement resolution shall include:

- wall collision
- actor collision
- floor and ceiling clamping
- slide response
- sector transition
- blockmap relinking

#### FR-11 — Spatial Acceleration

The simulation shall use the blockmap as the primary spatial acceleration structure for collision, sight, hitscan, and radius queries.

#### FR-12 — Damage Processing

The simulation shall process hitscan, projectile, melee, and splash damage.

Damage shall modify actor health and trigger death-state behavior when health reaches zero or below.

#### FR-13 — Line Special Dispatch

The simulation shall dispatch line specials for cross, use, shoot, and push interactions.

Line-special dispatch shall read line special identifiers and arguments from `line_t`.

#### FR-14 — Sector Effect Execution

Sector effects such as doors, floors, ceilings, platforms, scrollers, and lights shall execute through registered thinkers.

Sector movers shall mutate sector planes over time until they reach their target state.

#### FR-15 — Sector Mover Conflict Prevention

The simulation shall prevent incompatible sector movers from controlling the same sector plane at the same time.

#### FR-16 — Monster AI

Non-player actors shall support autonomous AI through actor state functions and native/ZScript behavior.

AI shall include target acquisition, chase movement, attack selection, pain response, and death response.

#### FR-17 — Bot Input Equivalence

Bot actors shall produce player-command-equivalent input.

Once written into `player_t::cmd`, bot commands shall be consumed by the same player-thinking path as human commands.

#### FR-18 — ACS Script Execution

The simulation shall execute ACS scripts through a script thinker.

ACS scripts shall maintain their own program counters, stacks, local variables, and wait conditions.

#### FR-19 — ZScript Integration

ZScript called by actors, states, and world events shall execute synchronously within the current tic unless it explicitly schedules future behavior.

#### FR-20 — Serialization

The simulation shall support full serialization and deserialization of live thinker state.

Serialization shall include all gameplay-relevant actor, sector, script, and thinker state needed to resume the simulation exactly.

#### FR-21 — Demo Reproduction

Given the same initial state and the same recorded `usercmd_t` sequence, demo playback shall reproduce the same simulation result.

#### FR-22 — Renderer Decoupling

The simulation shall not depend on renderer state or renderer APIs.

The renderer shall be a passive consumer of simulation state.

### Non-Functional Requirements

#### NFR-01 — Determinism

Given the same initial state, same map data, same actor definitions, same random seed, and same input sequence, the simulation shall produce the same resulting state.

#### NFR-02 — No Wall-Clock Dependency

The simulation shall not use wall-clock time internally for gameplay decisions.

#### NFR-03 — Platform Independence

Gameplay results shall not vary across supported platforms.

Platform-specific behavior shall be excluded from deterministic gameplay paths.

#### NFR-04 — Stable Execution Ordering

Statnum ordering and intra-list execution behavior shall remain stable.

Changes to ordering shall be treated as determinism-affecting changes.

#### NFR-05 — Complete Save State

Any field that affects future gameplay shall be serializable and restorable.

Partially serialized gameplay state shall be treated as a defect.

#### NFR-06 — Renderer Isolation

The simulation shall not allocate GPU resources, call rendering APIs, or depend on frame interpolation.

#### NFR-07 — Replay Safety

The simulation shall be suitable for demo replay and lockstep-style synchronization by ensuring that input streams are sufficient to reproduce world evolution.

#### NFR-08 — Mod Compatibility

The simulation shall support actor and behavior definitions loaded from mod-provided ZScript and DECORATE data while preserving deterministic execution semantics.

---

## SPECIFICATIONS

### `P_Ticker()` Specification

#### Responsibility

`P_Ticker()` advances the entire gameplay simulation by one tic.

#### Inputs

- current `FLevelLocals`
- per-player `player_t::cmd`
- current thinker lists
- loaded map geometry
- active scripts
- active actor states
- deterministic random state

#### Output

No explicit return value.

It mutates the current level state.

#### Required Order

```text
1. Particle/simulation-side particle update
2. Player thinking for each active player
3. World event tick
4. Level bookkeeping tick
5. Thinker execution
6. Map special update
```

### `usercmd_t` Specification

#### Structure

```cpp
struct usercmd_t
{
    uint32_t buttons;
    short    pitch;
    short    yaw;
    short    roll;
    short    forwardmove;
    short    sidemove;
    short    upmove;
};
```

#### Semantics

| Field | Meaning |
|---|---|
| `buttons` | action bitfield: attack, use, jump, crouch, etc. |
| `pitch` | vertical look delta |
| `yaw` | horizontal turn delta |
| `roll` | roll/tilt delta |
| `forwardmove` | forward/backward movement magnitude |
| `sidemove` | strafe movement magnitude |
| `upmove` | vertical movement for flight/swimming/noclip contexts |

### `P_PlayerThink()` Specification

#### Responsibility

Apply one player's command to that player's simulation state.

#### Algorithm

```text
1. Read player_t::cmd.
2. Preserve original command if required.
3. Run player pawn logic.
4. Apply movement command.
5. Update camera/view height state.
6. Process death/respawn logic if applicable.
```

### Thinker Collection Specification

#### Active Lists

```cpp
FThinkerList Thinkers[MAX_STATNUM + 2];
```

Active thinkers live here.

#### Fresh Lists

```cpp
FThinkerList FreshThinkers[MAX_STATNUM + 1];
```

Newly created thinkers live here until they are merged into active thinker lists.

#### Invariant

A thinker shall be in exactly one thinker list at a time, unless destroyed or not yet registered.

### `RunThinkers()` Specification

#### Responsibility

Call the tick logic of all active thinking thinkers in deterministic order.

#### Algorithm

```text
for statnum = STAT_FIRST_THINKING to MAX_STATNUM:
    for each thinker in Thinkers[statnum]:
        if thinker has not received PostBeginPlay:
            call PostBeginPlay
        call Tick

repeat:
    moved = false
    for statnum = 0 to MAX_STATNUM:
        while FreshThinkers[statnum] is not empty:
            move first fresh thinker to Thinkers[statnum]
            moved = true
until moved == false
```

### `FState` Specification

#### Structure

```cpp
struct FState {
    FState      *NextState;
    VMFunction  *ActionFunc;
    int32_t      sprite;
    uint8_t      Frame;
    int16_t      Tics;
    uint16_t     TicRange;
    uint16_t     StateFlags;
};
```

#### State Transition Rule

```text
if actor.tics > 0:
    actor.tics--

if actor.tics == 0:
    next = actor.state->NextState

    if next == null:
        actor.Destroy()
    else:
        actor.state = next
        actor.tics = next.Tics plus deterministic random TicRange adjustment
        call next.ActionFunc
```

### Movement Specification

#### `P_TryMove()`

Responsibility: attempt to move an actor to a new XY position.

```text
P_TryMove(actor, newX, newY):
    result = P_CheckPosition(actor, newX, newY)

    if result is blocked:
        attempt P_SlideMove(actor)

    if final movement accepted:
        unlink actor from old blockmap cells
        update actor position
        relink actor into blockmap
        update sector/floor/ceiling data
        activate crossed line specials
```

### `P_CheckPosition()` Specification

#### Responsibility

Evaluate whether an actor can occupy a requested position.

#### Checks

- line blocking
- actor blocking
- floor height
- ceiling height
- dropoff constraints
- 3D floor constraints
- portal/cross-sector constraints
- special-line crossing candidates

### Blockmap Specification

#### Cell Size

Each blockmap cell represents a 128×128 map-unit region.

#### Contents

Each relevant cell references:

- line segments touching that cell
- actors whose bounding boxes overlap that cell

#### Required Uses

The blockmap shall be used for:

- movement collision
- hitscan line traces
- sight checks
- explosion radius checks
- actor proximity iteration

### Line Special Specification

#### Activation Entry Points

| Trigger | Entry |
|---|---|
| actor crosses line | `P_CrossSpecialLine()` |
| player uses line | `P_UseLines()` |
| projectile/hitscan shoots line | `P_ShootSpecialLine()` |
| actor pushes line | `P_PushSpecialLine()` |

All activation paths converge on line-special dispatch.

#### Required Data

Line-special dispatch reads:

- `line_t::special`
- `line_t::args[5]`
- sector tag references
- activating actor
- activation side/context

### Sector Effect Specification

#### Constructor Behavior

A sector-effect thinker shall:

1. register itself under the correct statnum
2. attach itself to the affected sector
3. store start and target state
4. set sector back-pointer such as `floordata` or `ceilingdata`
5. initialize interpolation metadata if needed

#### Tick Behavior

A sector-effect thinker shall:

1. move its controlled plane or effect toward the target state
2. update sector state
3. transition internal phase if necessary
4. destroy itself when complete
5. clear the sector back-pointer when destroyed

### ACS Specification

#### `DACSThinker`

`DACSThinker` manages running ACS scripts.

Each script stores:

- program counter
- stack
- local variables
- wait state
- script identifier

#### Tick Behavior

Each tic, the ACS thinker executes runnable scripts until each script:

- delays
- waits
- terminates
- blocks on a condition

### ZScript Specification

ZScript functions execute synchronously when called from:

- actor state actions
- actor methods
- event handlers
- line specials
- world events
- sector or gameplay systems

They execute within the current tic and return before control continues to the next scheduled simulation step.

### Serialization Specification

#### Method

Serializable thinkers implement:

```cpp
void Serialize(FSerializer &arc);
```

#### Save Algorithm

```text
for statnum = 0 to MAX_STATNUM:
    for thinker in Thinkers[statnum]:
        write class identity
        write serialized fields
```

#### Load Algorithm

```text
clear existing thinker state
read saved thinker entries
for each entry:
    construct class by saved identity
    deserialize fields
    insert into thinker list
resolve object references
resume simulation
```

#### Required Coverage

Serialization shall cover:

- actors
- actor positions and velocities
- actor state pointers / state identity
- inventory
- health
- targets and object references
- sector movers
- sector planes and light state
- scripts
- script variables and wait states
- random state
- persistent non-thinking thinkers

### Demo Specification

#### Recording

Record per-tic `usercmd_t` input.

Do not record every actor position or every sector state.

#### Playback

Replay by feeding recorded `usercmd_t` values into the same simulation from the same initial state.

Correctness depends on determinism.

### Renderer Consumption Specification

The renderer may read:

- `actor->Pos()`
- `actor->Angles`
- sector floor and ceiling planes
- sector light levels
- `player->mo`
- `player->psprites`

The renderer may interpolate presentation state between tics.

The simulation shall not depend on this interpolation.

### Randomness Specification

#### Source

The simulation uses deterministic PRNG streams.

#### Determinism Inputs

Random results are determined by:

1. initial seed
2. named PRNG state
3. number of calls already made
4. order of calls

#### Consequence

Changing thinker order or introducing extra random calls in gameplay paths can change future simulation state.

### 3D Floor Specification

#### Storage

3D floors are stored through extended sector data rather than directly on the base sector.

#### `F3DFloor` References

A 3D floor references:

- defining line
- model sector
- target sector
- top plane
- bottom plane

#### Movement Rule

Moving a 3D floor is implemented by moving the sector planes that the 3D floor references.

### Polyobject Specification

#### Geometry Ownership

A polyobject does not own independent copies of map geometry.

It owns pointers into level geometry arrays.

#### Movement Rule

A polyobject moves by:

1. checking whether movement is blocked
2. mutating referenced vertex positions
3. updating its bounding box
4. relinking itself in spatial structures
5. optionally crushing actors or reversing movement

### Coherence Rules for Future Changes

Any future modification to this document or implementation should preserve the following distinctions.

| Section | Contains | Does Not Contain |
|---|---|---|
| WORLD | domain facts, assumptions, external constraints | implementation algorithms |
| MACHINE | system boundary, dataflow, major mechanisms | testable shall-statements |
| REQUIREMENTS | what the system must do | low-level implementation detail unless required |
| SPECIFICATIONS | concrete structures, algorithms, ordering, interfaces | broad project motivation |
