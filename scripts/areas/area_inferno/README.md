# TzKal-Zuk

The Inferno's final encounter, ported from OldSchool revision 239 and scripted
from Kronos' `TzKalZuk.java` / `JalTokJad.java` / `JalMejJak.java` /
`JalXil.java` / `JalZek.java`.

Start it in game with `::~zuk`. It teleports you into the arena at OSRS map
square 35_83 and spawns TzKal-Zuk with his Ancestral Glyph.

**Zuk hits for up to 251 through prayer and defence.** The only defence is the
glyph: it slides across the row in front of him, and a shot is nullified while
you are inside its band. Stand still and you die in one shot — that is the
fight, not a bug.

## Debug entry points

| Command | What it does |
| --- | --- |
| `::~zuk` | Teleport in, spawn Zuk and the glyph at full health |
| `::~zuk 470` | Same, but start Zuk at 470 hitpoints so the Jad phase fires at once |
| `::~zukhp 470` | Teleport to a fight already running and set Zuk's health |
| `::maxrange` | Max every stat, wear the black d'hide setup with 100k rune arrows, complete all quests. Lives in `scripts/_test/scripts/cheats/cheat_maxrange.rs2`; it ends by asking which side you took in Shield of Arrav and Temple of Ikov |

## The encounter

Sources: OSRS Wiki (TzKal-Zuk, Inferno), Kronos' `TzKalZuk.java` and
`Inferno.java`, and Jupiter's player guide. Where they disagree, Kronos wins —
it is what this port was written against.

### The opening

Zuk does not start the fight; he has to get out of the wall first. Three pieces
of scenery seal him in: the **Ancestral Glyph** loc at (2270, 5363) and two
rocks at (2268, 5364) and (2273, 5364). The glyph is removed outright; each
rock swaps to its "Falling rock" twin, plays the collapse two ticks later and
is cleared two ticks after that, as the animation ends. All of it happens
behind a camera pan and a line of dialogue from TzHaar-Ket-Rak in Kronos, with
Zuk **locked** for the duration and unlocked ten ticks in.

Over the same ticks the glyph npc, which spawns embedded in the seal at
(2270, 5363), pauses three ticks and then force-walks two tiles south onto its
running row, z 5361.

Nothing is on the clock until Zuk is out. He then still holds fire until the
glyph has completed its first run to one end, so the earliest possible shot is
roughly twenty ticks after `::~zuk`.

### The steady state

- **Zuk** — 1200 hitpoints, never moves. Fires every **10 ticks** and cannot
  miss: the shot ignores prayer and defence and can kill from full. The glyph
  is the only defence there is.
- **Enrage** — at **240 hitpoints** he fires every **7 ticks** instead. This is
  the same threshold that spawns his healers, and the speed-up is what closes
  the two middle safespots: there is no longer time to cross behind the glyph
  between shots, so the player has to stay level with it.
- **Ancestral Glyph** — 600 hitpoints, three by three. Slides one tile a tick
  between x 2257 and x 2283 on z 5361, pausing four ticks at each end. It
  neither turns nor fights back. A shot is nullified while the player is inside
  its band; killing it removes the safespot permanently.
- **The band** is the glyph's own three tiles widened by where it is heading —
  a moving glyph covers more ground behind it than in front, and one parked at
  an end covers one extra tile on its outward side. Offsets are Kronos'
  `isTargetSafe()`.
- **Add waves** — a Jal-Xil and a Jal-Zek — 60 ticks after the glyph is ready,
  then no sooner than every 350 ticks; Jad's arrival pushes that out by 175.
  Suspended while Zuk sits between 479 and 599 hitpoints, which is the pause
  players farm by leaving the mager alive to buy time for Jad.
- **JalTok-Jad** at 480 hitpoints: nine-tick cycle, melee when adjacent,
  otherwise an even split of the magic tri-projectile and the ranged ground
  graphic. Calls three Yt-HurKot to heal it at half health.
- **Jal-MejJak** ×4 at 240 hitpoints: they heal Zuk from range until you hit
  one, then turn their backs on him and bombard the ground behind them.

### Everything targets the glyph first

The waves and Jad arrive shooting the **glyph**, not the player. That is what
makes a wave urgent: 600 hitpoints of safespot are being chewed through while
you decide what to do about it, and the fight is about tagging them off it
rather than out-damaging them. Left alone, a Jal-Xil and a Jal-Zek take the
glyph from full to nothing in about forty ticks.

An add turns on the player when the player gives it a reason to — a hit lands
through `ai_queue2`, which is where every player attack arrives. Until then it
does not hunt at all, which is why neither wave add carries a `huntmode` in
`inferno.npc`: being provoked is what hands it one. Killing the glyph releases
them too, because there is nothing else in the arena to shoot.

The Jal-MejJak barrage is aimed the same way. Kronos centres it on the healer's
own column dropped to the player's row — the ground directly behind a healer
that has turned around — scatters three tiles across the seven-by-seven square
there, and hits only a player still standing on one of them three ticks later,
ignoring prayer and defence. Moving at all is what takes nothing.

## Where the assets came from

Everything except the scripts and the gameplay fields of `inferno.npc` is
generated by `3rd/rscache/tools/port_lostcity` in the `3draster` repo. Re-run it
with:

```sh
cd <3draster>
make -C 3rd/rscache/tools port_lostcity
./3rd/rscache/tools/port_lostcity/port_lostcity \
  --rev osrs239 cache.osrs239 \
  --content <LostCity_Server>/content \
  --area areas/area_inferno --prefix inferno \
  --npc 7706=inferno_zuk --npc 7707=inferno_zuk_shield --npc 7708=inferno_mejjak \
  --npc 7700=inferno_jad --npc 7701=inferno_hurkot --npc 7702=inferno_xil --npc 7703=inferno_zek \
  --seq 7562=inferno_zuk_death --seq 7563=inferno_zuk_spawn --seq 7565=inferno_zuk_defend --seq 7566=inferno_zuk_attack \
  --seq 7568=inferno_shield_defend --seq 7569=inferno_shield_death \
  --seq 2863=inferno_mejjak_defend --seq 2864=inferno_mejjak_spawn --seq 2865=inferno_mejjak_death --seq 2868=inferno_mejjak_attack \
  --seq 7590=inferno_jad_melee --seq 7591=inferno_jad_defend --seq 7592=inferno_jad_magic --seq 7593=inferno_jad_range --seq 7594=inferno_jad_death \
  --seq 2639=inferno_hurkot_heal \
  --seq 7604=inferno_xil_melee --seq 7605=inferno_xil_range --seq 7606=inferno_xil_death --seq 7607=inferno_xil_defend \
  --seq 7610=inferno_zek_magic --seq 7612=inferno_zek_melee --seq 7613=inferno_zek_death \
  --seq 7561=inferno_seal_collapse \
  --loc 30343 --loc 30344 \
  --spotanim 1375=inferno_zuk_proj --spotanim 1376=inferno_zek_proj --spotanim 1377=inferno_xil_proj \
  --spotanim 660=inferno_heal_proj --spotanim 659=inferno_lava_splash \
  --spotanim 447=inferno_jad_magic_gfx --spotanim 448=inferno_jad_proj1 --spotanim 449=inferno_jad_proj2 --spotanim 450=inferno_jad_proj3 \
  --spotanim 451=inferno_jad_range_gfx --spotanim 157=inferno_jad_hit --spotanim 444=inferno_hurkot_heal_gfx \
  --map 35_83 --apply
```

It writes `configs/inferno.{npc,seq,spotanim,loc,flo}`, the models and animsets
under `content/models/inferno/`, `content/maps/m35_83.jm2`, and the id lines in
`content/pack/*.pack`. It is idempotent: names already registered keep their ids,
and re-running the command above reproduces every file byte for byte.

**Always run the whole command.** Each config is rewritten from the assets that
run asked for, not merged into — so exporting one extra loc on its own empties
`inferno.loc` of the sixty-odd the map pulls in, and the same for `inferno.seq`.
The damage is quiet: the build still succeeds under `BUILD_VERIFY=false` and
only the server's own startup refuses it, with a stray
`inferno_loc_6926 was not found in any .loc files`. To add an asset, add its
flag to the command above and run all of it.

**`inferno.npc` is half generated and half hand-authored.** The exporter writes
the asset lines (name, models, size, animations, vislevel, resize, ambient,
ops); the combat stats, params and hunt modes below them come from the Kronos
encounter data and are *not* regenerated. Re-running the exporter overwrites
that file, so those lines have to be merged back afterwards.

`configs/inferno.{constant,varn,hunt}` and everything in `scripts/` are
hand-written and untouched by the exporter.

## Why the arena first came out green

Worth writing down, because nothing about it looked broken.

A map tile does not store a floor's config id — it stores **id + 1**, with 0
meaning "no floor on this tile". Both the OSRS client and LostCity's do the
`- 1` when they resolve it (`FloType.list[id - 1]`). rscache's terrain decoder
keeps the raw stored value in `underlay_id` / `overlay_id`, faithfully, so the
exporter has to subtract the one itself — and the first version did not.

Every tile in the square was therefore exported carrying its *neighbour's*
colour: consistently one record off, and every id still valid, so it read as a
colour problem rather than a decode failure. The Inferno's floor ids sit in a
run where the record above each one is an ordinary outdoor floor, so a lava bowl
came out grass green (underlay 1 instead of 0) over yellow-brown. The lava moat
ringing the arena was the clearest tell: its real overlay is the bright
`0xF6CF0E`, but the record one above it carries the client's `0xFF00FF`
"no colour of my own" sentinel, which the exporter resolved to that record's
dull `0x544D37` secondary — so the ring rendered as a flat empty band instead of
lava.

Correction, from a later look at the actual records: 151 *is* the square's real
overlay and the only one it has, and 151 is the magenta-sentinel record —
`05 01 FF00FF 07 F6CF0E 00`, so `hideUnderlay=false`, primary magenta, secondary
`0xF6CF0E`, texture `-1`. The lava yellow is its *secondary* colour, which both
clients use for the minimap and never for the 3D tile. So the moat is not a
coloured overlay at all: the sentinel resolves to "draw nothing" and the visible
lava is loc geometry standing on tiles that have no underlay either. See the
black-bands note below for what the exporter was doing with that instead.

The fix is one subtraction in `lc_export_map`. With the right records the square
exports the palette it should have: dark reds (`0x752222`, `0x300000`),
near-blacks, and the lava yellow.

## Why there were black bands between the lava tiles

The sentinel again, one layer down. Rev 254 resolves an overlay exactly the way
OSRS does — texture first, then `flo.rgb === 0xFF00FF`, then the colour — and
its magenta branch sets the overlay colour to `-2`, which `getOCol` turns into
the `12345678` "skip this face" value. The tile draws nothing and you see the
lava locs behind it.

The exporter used to *omit* `colour=` for a sentinel overlay, on the belief that
LostCity reads rgb 0 as "no overlay colour". It does not: 0 is the colour black.
`FloType.rgb` stayed 0, the record fell through to the third branch, and every
one of the 872 tiles painted a flat black quad across the lava. Writing
`colour=0xFF00FF` through verbatim puts the record back on the branch the
reference takes.

## Known gaps

- **Zuk's footprint is 5, not 7.** LostCity caps npc `size` at 5. His model
  renders at full size; only the tiles he occupies are smaller. A model is
  drawn *centred on its footprint*, so his coord is not Kronos' — it is Kronos'
  (2268, 5364) shifted a tile north-east, which is what puts the centre of a
  five-tile footprint where the centre of a seven-tile one would have been.
  Anything else leaves him visibly off to one side of his alcove.
- **The opening has no camera work or dialogue.** Kronos fades out, pans the
  camera onto Zuk, shakes it, and has TzHaar-Ket-Rak speak a line while he
  breaks out. The port keeps the timing — Zuk is inert for the same ten ticks —
  but the player just watches it happen from where they are standing.
- **The arena is untextured, and so is the source.** Not a porting gap: the
  exporter carries materials across now, but there is nothing here to carry.
  Every one of the seven npc models decodes with no face-texture section at
  all, all 69 loc types on 35_83 come to 19,890 faces with zero textured ones,
  and the square's single overlay (151) has `texture = -1`. The client agrees —
  loading 35_83 reports `0 textures`. The Inferno is authored in flat colour.
- **No sounds.** Sound 163 (Jad's hit) and the rest have no `.synth` equivalent.
- **Jad's Yt-HurKot healers are scripted but were not exercised in testing** —
  reaching them needs Jad taken to half health in combat.
- **The Jal-MejJak barrage was not exercised either.** It only starts once a
  player has hit a healer, which the headless harness cannot stage; the aiming
  and the three-tile scatter are transcribed from Kronos but unproven in play.
- **Nothing rolls for accuracy.** Every attack in the encounter, against the
  player or against the glyph, rolls damage straight out of its max hit. That
  matches how the player-facing attacks were already written, but it makes the
  adds strictly harsher on the glyph than Kronos, where they have to get past
  its defence first.
- **The fight stops while nobody is in the arena.** LostCity only ticks npcs in
  zones that hold a player, so logging out (or dying and respawning in Lumbridge)
  freezes the glyph mid-run and stops the wave timer. `::~zukhp` teleports you
  back in, which starts it moving again.
