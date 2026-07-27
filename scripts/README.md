# RuneScript (rs2) architecture

How the content in this tree actually runs: what a tick does, what fires a
script, how combat gets from a click to a hitsplat, and where the sharp edges
are. Written from reading the engine, not from the docs, so where the two
disagree this follows the engine.

Engine paths below are relative to `engine/src/`.

---

## 1. The shape of the thing

Nothing in this tree is called by name from the engine except through
**triggers**. A trigger is a (kind, subject) pair — "someone clicked option 2 on
this npc", "this npc's timer came round", "this queued script's delay ran out".
The engine raises them; scripts handle them. There is no main loop in content.

Everything else — `proc`, `label`, `command` — is just factoring: code that runs
because a trigger script called it.

### File types

| Extension | What it is |
| --- | --- |
| `.rs2` | Scripts. The only executable file type |
| `.constant` | `^name = value` compile-time substitutions. No runtime cost, no storage |
| `.varp` `.varn` `.vars` `.varbit` | Player / npc / shared / bit-packed variables |
| `.obj` `.npc` `.loc` `.seq` `.spotanim` `.flo` `.idk` | Config for the things in the world |
| `.param` | Named typed fields you can hang off obj/npc/loc configs |
| `.inv` | Inventory definitions (size, scope, stack rules) |
| `.enum` `.struct` `.dbtable` `.dbrow` | Lookup tables |
| `.hunt` | Npc target-acquisition profiles |
| `.if` | Interfaces |
| `.pack` (in `../pack/`) | `id=name` registries. Ids are assigned here, not in the config |

Layout is by subject: `scripts/areas/`, `scripts/quests/`, `scripts/minigames/`,
`scripts/skill_combat/`, `scripts/player/`, `scripts/npc/`. `scripts/_test/`
holds dev cheats, `scripts/_unpack/` holds decompiled reference config.

`scripts/engine.rs2` is the command surface — every builtin's signature and
return type. **It is the reference for what an op is called and what it takes.**

---

## 2. Ticks

A tick is 600ms. `World.cycle()` (`engine/World.ts`) runs a fixed pipeline
every tick, and the order matters more than anything else in this document:

```
processWorld()        world queue, npc hunt
processClientsIn()    packets, pathfinding requests
processNpcEventQueue()spawn / despawn triggers
processNpcs()         per npc: resume script, regen, TIMER, QUEUE, movement/modes
processPlayers()      per player: resume script, QUEUE, TIMERS, engine queue, interactions, movement
processLogouts() / processLogins()
processZones()        loc/obj respawn, zone events
processInfo()         player & npc update masks
processClientsOut()   flush
processCleanup()      reset masks
```

Two consequences worth internalising:

- **Npcs are processed before players.** An npc that reads the player's position
  sees where they were at the end of last tick.
- **Within an npc, `ai_timer` runs before `ai_queue`.** A queue you raise this
  tick lands after the timer has already run — and if the timer deletes the npc,
  the queue never runs at all. This is a real trap; see §11.

`map_clock` is the current tick number. The idiom for "not again for N ticks"
is a var holding a deadline:

```
if (%npc_action_delay > map_clock) { return; }
%npc_action_delay = add(map_clock, 4);
```

---

## 3. Script kinds

```
[proc,name](args)(returns)     called with ~name
[label,name](args)             jumped to with @name  — tail call, never returns
[command,name]                 engine builtin declaration (engine.rs2 only)
[debugproc,name](args)         a cheat: ::name
[<trigger>,<subject>]          raised by the engine
```

`~proc` **calls** — execution comes back. `@label` **jumps** — nothing after it
runs. That is the whole difference, and it is the reason a `@`-jump is always
the last line of a script.

### Trigger subjects and the fallback chain

`[opnpc2,goblin]` is specific. `[opnpc2,_]` is the global default. There is also
a category tier. `ScriptProvider.getByTrigger` resolves:

```
(trigger, type)  →  (trigger, category)  →  (trigger)
```

First hit wins, and **a specific script replaces the default entirely** — it
does not extend it. This is why so many scripts end with an explicit call to the
default they displaced:

```
[ai_queue2,inferno_jad]
... my phase logic ...
~npc_default_damage($damage);      // or the damage never gets applied
```

Forget that line and your npc becomes invulnerable, silently.

---

## 4. Active slots and the `.` prefix

Scripts do not take "the npc" as an argument. There is an **active** slot per
entity kind (player, npc, loc, obj) and a **secondary** slot, and ops read
whichever the prefix selects:

```
npc_coord        the active npc's coord
.npc_coord       the secondary npc's coord
%my_var          a var on the active npc
.%my_var         a var on the secondary npc
```

Ops that find something set the slot: `npc_find` sets the active npc,
`.npc_find` sets the secondary. `npc_add` leaves the npc it created active.

**The compiler tracks slot liveness per branch.** You cannot hide a `.npc_find`
inside a proc and use `.npc_coord` at the call site — the caller's branch does
not know the slot is live, and you get *"Attempt to access uninitialized
pointer"*. The lookup has to be written out at every site that uses the result:

```
if (.npc_find(npc_coord, some_npc, 16, ^vis_off) = true) {
    npc_facesquare(.npc_coord);      // fine: inside the branch that succeeded
}
```

---

## 5. Op and Ap

The two ways to interact with a target.

- **op** — "operate". Requires being *adjacent/on* the target. `opnpc1..5`
  correspond to the right-click options in order.
- **ap** — "approach". Fires from a distance, at whatever range the script asks
  for. Ranged and magic live here.

`Player.tryInteract` (`engine/entity/Player.ts`) runs op first if the player is
in operable distance, otherwise ap if in approach distance. `processInteraction`
tries once *before* movement and once *after*, so a player who steps into range
this tick still acts this tick.

`p_aprange(n)` inside an ap script means "I need to be within n — go closer and
try again". The engine sees `apRangeCalled`, restores the waypoints it had
stashed, and does **not** count the interaction as having happened. That is how
a ranged attack pulls you into range without consuming the click.

If nothing interacts, the path is empty, and no step was taken, the engine emits
*"I can't reach that!"* and drops the interaction.

The `u` and `t` suffixes (`opnpcu`, `applayert`) are use-item-on and cast-spell-on.

### Npc modes

Npcs get the mirror image, as **modes** rather than clicks. `NpcMode`
(`engine/entity/NpcMode.ts`):

```
NONE  WANDER  PATROL                     targetless
PLAYERESCAPE  PLAYERFOLLOW  PLAYERFACE  PLAYERFACECLOSE
OPPLAYER1..5   APPLAYER1..5              → [ai_opplayer2,x] / [ai_applayer2,x]
OPNPC1..5      APNPC1..5
OPLOC1..5      APLOC1..5
OPOBJ1..5      APOBJ1..5
```

`npc_setmode(applayer2)` targets whatever is in the relevant active slot —
`_activePlayer` for the player modes, `_activeNpc2` for the npc modes. **If that
slot is empty the call silently does nothing.** A queued npc script has no
active player (§7), so `npc_setmode(applayer2)` there is a no-op.

---

## 6. Default npc behaviour

An npc with no scripts still does things, driven by its config:

| Field | Effect |
| --- | --- |
| `defaultmode` | `none` / `wander` / `patrol`. Defaults to **wander** |
| `wanderrange` | Wander radius. Default 5 |
| `moverestrict` | `normal` / `blocked` / `blocked+normal` / `indoors` / `outdoors` / `nomove` / `passthru` |
| `blockwalk` | Whether it blocks others: `none` / `NPC` / `all` |
| `huntmode` | A `.hunt` profile. Default −1 = never hunts |
| `huntrange` | Hunt radius. Must be ≥ 1 or hunting is skipped |
| `timer` | Ticks between `ai_timer` firings |
| `maxrange` | How far from spawn it will chase before giving up |
| `respawnrate` | Ticks before a killed static npc returns |
| `attackrange` `attackrate` | Reach and speed |

`Npc.processMovementInteraction` runs each tick: targetless modes call
`noMode()` / `wanderMode()` / `patrolMode()`; targeted modes validate the target
(same floor, within `maxrange`, still the same type) and drop it if it fails.

**`moverestrict=blocked` is an inversion, not a restriction.** rsmod's BLOCKED
strategy requires `CollisionFlag.FLOOR` to be *present* and drops the npc-block
flag. It is for things that live on terrain the map marks impassable — lava,
water. An npc standing on such terrain with the default `normal` cannot move at
all, and fails silently: the waypoint stays queued forever.

`wanderMode` also re-queues a waypoint back to the spawn tile 1 tick in 8, and
teleports the npc home after 500 idle ticks. If a script is driving movement,
set `defaultmode=none` or the wander will fight it.

### Hunt

`.hunt` profiles decide what an npc goes looking for:

```
[my_hunt]
type=player            player | npc | obj | scenery | off
check_vis=lineofsight
check_notcombat=%lastcombat
find_keephunting=on
find_newmode=applayer2   ← the mode it switches to on finding something
rate=11                  ← throttle, in ticks
```

Hunting runs in `processWorld` (npcs) and per-npc in `turn()`. `npc_sethuntmode`
and `npc_sethunt` change it at runtime — which is the practical way to give an
npc a target when you have no active player to hand it.

---

## 7. Queues

Deferred work. The delay is in ticks; **delay 0 means "next tick", not "now"**,
because the counter is decremented before it is tested.

### Player queues

```
queue(script, delay, arg)          normal
queue*(script, delay)(args...)     normal, typed args
weakqueue / strongqueue / longqueue / softqueue
```

| Type | Behaviour |
| --- | --- |
| NORMAL | Runs when the player `canAccess()` |
| WEAK | Cleared when a modal opens |
| STRONG | Closes any open modal before running |
| LONG | Like normal, with explicit logout behaviour |
| SOFT | Runs even while busy |
| ENGINE | Internal, delay forced to 0 |

`canAccess()` is `!protect && !busy()` — busy meaning a modal is open or the
player is mid-delay. Queues stall rather than drop while inaccessible.

### Npc queues

```
npc_queue(n, arg, delay)     → [ai_queue<n>,type]      n = 1..20
.npc_queue(n, arg, delay)    → same, on the secondary npc
```

The number carries no engine meaning. What gives 1, 2 and 3 their meaning is
`scripts/skill_combat/scripts/npc/npc_combat.rs2`:

```
[ai_queue1,_] gosub(npc_default_retaliate);
[ai_queue2,_] ~npc_default_damage(last_int);
[ai_queue3,_] gosub(npc_default_death);
```

So by convention across all content:

- **ai_queue1 = "a player attacked me"** — raised by `~npc_retaliate`, which also
  records the attacker in `%npc_aggressive_player`.
- **ai_queue2 = "take this much damage"** — every player attack lands here.
- **ai_queue3 = "you died"** — raised by `~npc_default_damage` when hp hits 0.

4 and up are free for your own use.

> **A queued npc script has no active player.** `ScriptRunner.init(script, this,
> null, args)` — the player slot is `null`. `mes`, `npc_setmode(applayer2)` and
> anything else needing a player will do nothing. If you need to know who hit
> you, read `%npc_aggressive_player`, which the *player-side* retaliate path set
> for you. To talk to somebody, walk the arena for one with `huntall`.

`last_int` is the queue's argument; `last_int` from `queue*` forms is the first
typed arg.

---

## 8. Timers

```
[timer,name]       player, does not run while busy
[softtimer,name]   player, runs while busy
[ai_timer,type]    npc, interval from the npc config's `timer=` field
```

Player timers are set with `settimer`/`cleartimer` and keyed by script, so
setting the same timer twice replaces it. Npc timers come from config; the
interval can be changed at runtime with `npc_settimer`.

`ai_timer` is the workhorse for bespoke npc behaviour — bosses in this codebase
are written as one `ai_timer` plus a handful of `ai_queue`s, deliberately
bypassing the stock combat modes.

---

## 9. Delays and suspension

`p_delay(n)` / `npc_delay(n)` suspend the *script* and mark the entity delayed
until `currentTick + 1 + n`. The script resumes where it left off — the engine
re-enters it from `activeScript` at the top of the next eligible tick.

While delayed:

- the entity fails `isValid()`,
- **it is invisible to `npc_find` / `npc_findallany`**,
- its queues do not decrement.

That last pair is the reason you cannot poll for "has this npc finished dying
yet" — a dying npc is delayed, so it reads as already gone from the first tick
of its death.

`p_arrivedelay` / `npc_arrivedelay` suspend until movement finishes (up to 2
ticks).

---

## 10. Combat

### Player attacks npc

1. Player clicks Attack → `opnpc2` → `[opplayer2,_]` in `player_combat.rs2`, or
   an ap trigger for ranged/magic.
2. Style and weapon data come from `combat.rs2` — `~combat_get_weapon_style_data`
   maps the weapon's category to a `dbrow` of attack styles.
3. `player_melee.rs2` / `player_ranged.rs2` / `player_magic.rs2` roll accuracy
   and damage, then deliver:

   ```
   npc_queue(2, $damage, <flight ticks>);   // the hit
   ~npc_retaliate(<delay>);                 // ai_queue1 + %npc_aggressive_player
   ```

   Melee delivers at delay 0; ranged and magic delay by projectile flight time,
   `calc($duration / 30)` — **30 client cycles per tick**.
4. `[ai_queue2,...]` applies it via `~npc_default_damage`, which calls
   `npc_damage(^hitmark_damage, n)` and raises `ai_queue3` at 0 hp.

Specials live in `player/specs/scripts/`, spells in `player/spells/scripts/`,
and each has a `pvp/` mirror for player-versus-player.

### Npc attacks player

Either the stock path — `[ai_opplayer2,_] gosub(npc_default_attack)` in
`npc_combat.rs2`, dispatching to `npc_combat_melee/ranged/magic.rs2` — or a
scripted one from `ai_timer`. The scripted shape is:

```
%npc_action_delay = add(map_clock, npc_attackrange_or_rate);
npc_anim(npc_param(attack_anim), 0);
def_int $d = ~player_projectile(npc_coord, coord, uid, spotanim, ...);
queue(combat_damage_player, calc($d / 30), $damage);
queue*(playerhit_n_retaliate, calc($d / 30))(npc_uid);
~npc_set_attack_vars;
```

`~player_projectile` returns flight time in client cycles; divide by 30 for
ticks. `queue(combat_damage_player, ...)` applies it and handles ring of recoil.
`playerhit_n_retaliate` is the *player's* auto-retaliate, not the npc's.

### Multi-combat

`~npc_check_notcombat` calls `map_multiway(npc_coord)` — **the npc's tile, always**.
Outside multi, an npc will not attack a player already in combat with something
else. Multi zones are listed per 8×8 zone in `maps/multiway.csv`.

---

## 11. Death

**Npc.** `~npc_default_damage` → hp 0 → `npc_queue(3, 0, 0)` → `[ai_queue3,_]` →
`~npc_default_death` → `~npc_death`, which walks the npc, `npc_arrivedelay`s,
plays `death_anim`, `npc_delay(1)`, then `npc_del`. Measured end to end that is
about 8 ticks from the trigger, and it varies with the arrive delay.

Override `[ai_queue3,yourtype]` for bespoke death, and remember to
`gosub(npc_default_death)` at the end or the npc never leaves.

**Player.** `~damage_self` → hp 0 → `queue(player_death, 0, 0)`.

---

## 12. Variables

| Kind | Scope | Notes |
| --- | --- | --- |
| `%name` varp | Player | `transmit=yes` syncs it to the client for interface scripts |
| `%name` varn | Npc | Per npc *instance*. Same name on two npc types is two independent values |
| `%name` vars | Shared | World-wide |
| varbit | Packed | Bits inside a varp |

Types are declared in the config (`type=player_uid`, `type=coord`,
`type=boolean`, …); an undeclared var is an int.

> **Non-int npc vars reset to −1**, not to 0 or false. A `type=boolean` npc var
> therefore matches neither `= true` nor `= false` after a respawn, and every
> comparison against it silently fails. Use plain ints as 0/1 for npc flags.

`^name` constants are compile-time text substitution — no storage, no runtime
lookup, and they can hold coords, ids, anything.

---

## 13. Params vs fields

Config entries have **fields** (known to the engine) and **params** (arbitrary
named values you define in a `.param`). They are read differently:

```
npc_attackrange          ← the `attackrange=` FIELD
npc_param(attackrange)   ← a PARAM that happens to share the name
```

Both compile. If the param exists with a default and nothing sets it, the second
returns that default — usually **0** — and does so silently. In this codebase
that mistake once left an entire boss mechanic dead: `huntall(coord, 0, ...)`
finds nobody, and `npc_range(coord) > 0` is always true.

When a config line looks like `attackrange=6`, it is a field. Use the accessor.

---

## 14. Interfaces

`.if` files define components; `if_openmain` / `if_openside` / `if_openoverlay` /
`if_openchat` mount them. `if_settext`, `if_sethide`, `if_setcolour`,
`if_setmodel`, `if_setnpchead`, `if_setanim` push updates. **There is no
`if_setsize`** — a component's dimensions are fixed at build time, so a
continuously-shrinking bar is not expressible; a row of segments toggled with
`if_sethide` is the period-correct substitute.

Components can also carry **client-side scripts**, which cost no packets at all:

```
[com_2]
type=text
script1op1=pushvar,xplamp
text=%1%                    ← substitutes the varp's value

[com_5]
type=rect
script1op1=pushvar,my_varp
script1=gt,50               ← only drawn while the varp exceeds 50
```

Comparators are `eq`, `lt`, `gt`, `neq`. A `transmit=yes` varp plus these is a
live readout that updates itself. See `interfaces/mazetimer.if` for the pattern.

---

## 15. Gotchas, collected

- A specific trigger **replaces** the default. Call the default explicitly.
- Queued npc scripts have **no active player**.
- `ai_timer` runs **before** `ai_queue` in the same tick.
- Delayed entities are **invisible to find ops** and fail `isValid()`.
- Queue delay 0 = next tick.
- The secondary slot's liveness is tracked **per branch**; a proc cannot vouch
  for it.
- `npc_param(x)` and the field `x` are different things.
- Non-int npc vars reset to −1.
- `moverestrict=blocked` is inverted — it *requires* blocked terrain.
- Anything after `@label` is unreachable.
- `~npc_default_damage` can return early, so write state changes *before* the
  call, never after.
- 30 client cycles = 1 tick, everywhere projectile timings appear.
- An npc coord is its **south-west** tile; a size-N npc extends north-east from
  it, and its model draws centred on that footprint.
