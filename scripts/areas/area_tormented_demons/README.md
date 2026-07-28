# Tormented Demons

A port of the OSRS (Desert Treasure II) Tormented Demon onto rev 254.

The asset side — every npc, sequence, spotanim and model id, where it came from
and what it is for — lives in `3draster/docs/BOSS_ASSETS.md`, together with the
Inferno's. This file is about the fight.

## The fight

Four clocks, all read off the demon every tick from `ai_timer`:

| clock | period | what happens |
|---|---|---|
| attack | 6 ticks | melee, magic or ranged |
| defenceless | 30 ticks | the flames go out; every hit lands |
| fire bomb | 60 ticks | two bombs, and the style is forced to change |
| prayer | every 150 damage taken | swaps overhead, stands still 6 ticks |

None of them start at spawn. They start the first tick the demon has someone to
fight, which is what `%td_engaged` marks — a demon standing in an empty room is
not on any clock, and walking away and coming back does not hand you a fight
that is already half over.

**Style selection is positional, not random.** Adjacent means melee. Otherwise
it alternates magic and ranged rather than rolling, so a player who was paying
attention knows which is coming; a fresh roll every time would take that away.
A fire bomb clears the last style, so the attack after a bomb is a coin flip —
which is what the wiki describes.

**The fire shield is geometry.** The source cache ships two body models, one
with the shield and one without, so dropping it is an `npc_changetype_keepall`
between `td_demon` and `td_demon_unshielded` — not a spotanim. It has to be
`_keepall`: plain `npc_changetype` resets the npc, which would hand the demon
its 600 hitpoints back every time the player hit it.

The shield is up at spawn, comes off the first time the demon is struck, and
goes back on after every bomb. So it covers the opening and the beat after each
bomb, and nothing else. While up it takes 20% off anything that is not
demonbane.

**Both npc types carry every trigger**, each as a two-line stub jumping to a
shared label. The demon spends the fight swapping between the two forms, and a
trigger written for only one of them would go quiet for half the fight.

## Testing

```
::~td        spawn a demon next to you, already engaged
::~tdbomb    bring the fire bomb forward to the next tick
::~tdstate   print every clock the nearest demon is holding
::~tdclear   remove every demon around you
```

`::~td` engages the demon on your behalf. A real Tormented Demon is
unaggressive and does nothing until something hits it, so without that the whole
fight sits still waiting to be provoked.

Verified in play with the `torirs` client against a live server: the demon
spawns and renders, attacks on its 6-tick cycle, reports *"The flames on the
demon's back go out."* at 30 ticks and *"The Tormented Demon hurls a fire bomb!"*
at 60.

## Substitutions, and what is not right yet

- **The warded style is overhead text, not an icon.** A 254 npc has no overhead
  prayer icon — the headicon field belongs to players. `npc_say` puts the text
  in the same place over the demon's head and answers the same question, and it
  is re-asserted every 7 ticks because overhead text fades and an icon does not.
- **Demonbane is Silverlight.** The modern fight means Arclight and Emberlight;
  neither exists in 2004, and Silverlight is the era's demonbane weapon.
- **The prayer swap is full protection**, not a percentage. No source gives a
  number, and "you must bring two combat styles" describes immunity. `^td_style_*`
  and the branch in `td_damaged` are where to change it if that turns out wrong.
- **Not verified in play: the prayer swap and the shield model swap.** Both need
  150 damage dealt *to* the demon, and the headless test harness spawns the fight
  but does not fight it. The bomb's landing damage is likewise unverified — the
  cast fires and the markers appear, but nothing has stood in one yet.
- **There is no arena.** The Ancient Guthixian Temple is not in a 2004 map, so
  `::~td` fights on open ground at `^td_test_player_coord`. Porting the temple
  map square would be the next piece.
- Only one demon. The real fight pairs them, and the pair coordinate — one melees
  while the other uses a projectile, and they deliberately do not attack on the
  same tick so the player can prayer-flick.

## Re-running the export

```sh
cd <3draster>
./3rd/rscache/tools/port_lostcity/port_lostcity \
  --manifest 3rd/rscache/tools/port_lostcity/tormented_demon.ini --apply
```

**It overwrites `configs/td.npc`, and that is where the combat stats live.** The
exporter only writes the visual fields; hitpoints, the defence bonuses, the
params and the hunt mode are hand-authored. Back the file up before re-running,
same as `inferno.npc`.

Then, in this order — the server rewrites `engine/data/pack` while it runs, so a
cache copied out from under it is torn:

```sh
pkill -f "bun run src/app.ts"
cd <LostCity_Server>/engine && BUILD_VERIFY=false bun run build
cp engine/data/pack/main_file_cache.* <3draster>/cache.rs254_zuk/
cd <3draster> && python3 tools/jag_crc.py cache.rs254_zuk   # into manifest_rs254.ini jag_crc=
cd <LostCity_Server>/engine && bun run src/app.ts
```

Skip the CRC refresh and the client connects but never finishes logging in.
