# Dragon Claws — asset pack for a LostCity / 2004scape-style RS2 server

A complete Dragon claws weapon: item config, models, the "Slice and Dice"
special attack (4-2-1-1 damage spread), spec animation + graphic, and an
optional debug cheat for testing.

## What's in this pack

```
models/dclaws/
  dclaws_model_32784.ob2       inventory/ground model
  dclaws_model_29191.ob2       worn (manwear/womanwear) model
  dclaws_model_29207.ob2       spec spotanim model
  dclaws_animset_0.anim        animset containing the spec body frames  (dclaws_frame_10800_1..42)
  dclaws_animset_369.anim      animset containing the spotanim frames   (dclaws_frame_12742_1..32)

configs/
  dclaws.obj                   [dragon_claws] item definition
  dclaws.seq                   [dragon_claws_spec] and [dragon_claws_spec_spot_seq]
  dclaws.spotanim              [dragon_claws_spec_spot]

scripts/
  pvm_dragon_claws.rs2         the special attack (label pvm_dragon_claws_sa + helper procs)
  cheat_dclaws.rs2             optional: ::dclaws equips claws + maxes melee stats + fills spec bar;
                               ::dcspec loops the spec animation/graphic for visual testing
```

## Installation

### 1. Copy the files

- `models/dclaws/` → your content repo's `models/dclaws/`
- `configs/*` → anywhere your packer picks up configs, e.g. `scripts/skill_combat/configs/`
- `scripts/pvm_dragon_claws.rs2` → wherever your special attack scripts live,
  e.g. `scripts/skill_combat/scripts/player/specs/scripts/`
- `scripts/cheat_dclaws.rs2` (optional) → your debug/cheat scripts directory

### 2. Register names in the pack files

If your tooling auto-registers new names on build, skip this step. Otherwise
append the following names (with the next free IDs in each file):

- `obj.pack`: `dragon_claws`
- `seq.pack`: `dragon_claws_spec`, `dragon_claws_spec_spot_seq`
- `spotanim.pack`: `dragon_claws_spec_spot`
- `model.pack`: `dclaws_model_29207`, `dclaws_model_32784`, `dclaws_model_29191`
- `base.pack`: `dclaws_animset_0`, `dclaws_animset_369`
- `anim.pack`: all 74 frames — `dclaws_frame_10800_1` … `dclaws_frame_10800_42`
  and `dclaws_frame_12742_1` … `dclaws_frame_12742_32`, in the order they appear
  in `configs/dclaws.seq`

### 3. Wire up the special attack

In your special attack dispatcher (e.g.
`scripts/skill_combat/scripts/player/player_special_attack.rs2`), add a case to
the weapon switch:

```
switch_obj($weapon) {
    ...
    case dragon_claws : @pvm_dragon_claws_sa;
}
```

### 4. Level requirement (optional but standard)

Dragon claws require 60 Attack to wield. With a levelrequire-style system:

```
[opheld2,dragon_claws] @levelrequire_attack(60, last_slot);
```

Otherwise gate the wield however your server handles tiered equipment.

### 5. Rebuild

Run your normal content build/pack step and restart the server. Verify with the
cheat: `::dclaws` then attack any NPC with the special bar enabled, or
`::dcspec` to loop the animation and graphic with no target.

## Dependencies — things this pack assumes your server already has

**Item config (`dclaws.obj`) references:**
- Category `weapon_claws`
- Obj params: `stabattack`, `slashattack`, `crushattack`, `stabdefence`,
  `slashdefence`, `crushdefence`, `strengthbonus`, `attackrate`,
  `stabattack_anim`, `slashattack_anim`, `defend_anim`, `stab_sound`,
  `slash_sound`, `specwep`, `sa_energy`
- Seqs: `claws_punch`, `human_axe_chop`, `human_axe_block` (standard base-cache anims)
- Synths: `stabsword_stab`, `stabsword_slash`

**Spec script (`pvm_dragon_claws.rs2`) references:**
- Special attack framework: `~set_sa_vars`, the `%sa_energy` var, and the
  `sa_energy` param (the claws cost 500 on a 0–1000 bar, i.e. 50%)
- Combat framework: `~player_attack_roll_specific`, `~npc_defence_roll_specific`,
  `~give_combat_experience`, `~npc_retaliate`, `%com_maxhit`, `%damagetype`,
  `%damagestyle`, `%npc_combat_xp_multiplier`, and npc params `max_dealt`,
  `defend_anim`
- Synth: `impale` (the spec sound)

These all exist in the LostCity content repo this pack was extracted from; if
your server's combat/spec framework differs, `pvm_dragon_claws.rs2` is the only
file you should need to adapt — the configs and models are framework-agnostic.

## Spec mechanics ("Slice and Dice")

One accuracy roll is retried up to four times, and where it first connects
decides the spread (d = the rolled damage, max = ordinary max hit):

| First connecting roll | Hits               | Damage roll d              |
|-----------------------|--------------------|----------------------------|
| 1st                   | d, d/2, d/4, d/4+1 | between max/2 and max−1    |
| 2nd                   | 0, d, d/2, d/2+1   | between 3/8 and 7/8 of max |
| 3rd                   | 0, 0, d, d+1       | between 1/4 and 3/4 of max |
| 4th                   | 0, 0, 0, d         | between 1/4 and 5/4 of max |
| none                  | 0, 0, 1, 1         | two-thirds of the time; otherwise 0-0-0-0 |

Every roll has ordinary accuracy (no boost) against the NPC's slash defence
regardless of attack style, matching the OSRS wiki's description. Hits 1–2 land
on the spec tick, hits 3–4 on the next — the paired hitsplats the weapon is
known for. Details in the comments of `pvm_dragon_claws.rs2`.

## Known limitation

The spec's body animation frames were authored on a later (OSRS-era) player
skeleton. See the header comment in `pvm_dragon_claws.rs2` for the full story —
if the animation stretches on your revision's player rig, that is a framemap
mismatch, not a packing error. The spec **spotanim** is self-contained (rigged
to its own framemap) and ports cleanly to any revision.
