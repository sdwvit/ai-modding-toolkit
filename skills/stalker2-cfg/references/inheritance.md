# refkey, refurl and the two kinds of mod

How a struct inherits from another, which refurl form a Zone Kit mod and a plain pak mod each need, the refurl and
refkey shapes that overflow the stack, array rules and DLC tables. Sections 2 and 3 come from a modder's field report
and from s2cfgtojson's notes (its readme, by sdwvit); the rest is adapted from the harness credited in the last line.

## Contents
1. refkey and refurl
2. Which refurl for which kind of mod
3. Shapes that overflow the stack
4. Arrays and bskipref
5. DLC tables
6. What cfg_verify reports

## 1. refkey and refurl
- `refkey=<Key>` inherits from a struct in the SAME file. A new struct added by a patch counts as being in the
  patched file, so it can `refkey` any vanilla struct of that file without `refurl`.
- `refurl` is only for another file. A relative `refurl` in a patch resolves against the PATCHED file, not the patch
  file's own folder: from `ItemPrototypes/DetectorPrototypes/DetectorPrototypes_patch_<Mod>.cfg`,
  `{refurl=../ItemPrototypes.cfg}` reaches `GameData/ItemPrototypes.cfg` (copy vanilla's value verbatim, no extra
  `../`; verified in game).
- Nested structs inherit too, from a root struct or from a sub-struct by dotted path:
  `Bandits : struct.begin {refkey=Bandits_X.Preset.PossibleInhabitantFactions.Bandits}`, then restate only what
  differs. Vanilla: `BoneDamageMap : struct.begin {refurl=../ref/refs.cfg; refkey=FleshBoneDamageMap}`
  (`ObjPrototypes/BoggyFlesh.cfg`). Keep path segments to named keys (no `[0]`). Whether a nested `refkey` inside a
  `{bpatch}` struct resolves (and to the patched or the vanilla donor) is not verified.
- Nested structs merge through `refkey`: a child restating only `VitalParams.MaxHP` keeps the parent's other vital
  params (vanilla zombies do this).
- `{bpatch}` only touches existing data, so a `refurl` on a `{bpatch}` struct does nothing.
- A patch on a base prototype does not reach children that restate the same field: they keep their own value.
  Check the children first (every weapon in `WeaponData/WeaponGeneralSetupPrototypes.cfg` redeclares its own arrays,
  so a template patch does not reach them).
- New structs, as the harness writes them: `refkey` the closest vanilla struct and restate only the changed fields
  (an ability variant restates only the changed `TriggeredCooldowns[n]`, not the list). A full copy is needed only
  when the new struct must drop inherited list entries. Copies bloat cfgs fast: one mod's lair file went from 92 MB
  to 4.7 MB by switching copies to bpatch + refkey. This pack's in-game result for one created-record shape
  differs: SKILL.md, patching rule 3.

## 2. Which refurl for which kind of mod
A refurl can start with a config path name: `@<ConfigPathName>/<path>`. The game takes the text before the first `/`
as the name and checks that this config path is active. If it is not, the log says
`refurl '...' references config path '...' which is not active` and the reference does not resolve. `BaseGame` gets
no special treatment. Only Zone Kit (Game Feature) mods turn config paths on (all of this from s2cfgtojson's notes).

| | Zone Kit (Game Feature) mod: ships through Steam Workshop, mod.io, Xbox | Plain pak in `<mods>` |
|---|---|---|
| Config paths | `BaseGame` (the game's `GameLite/GameData/`), `GameSettings`, and one per mod, named after the `ConfigPathName` in its GameFeatureData (the mod's name), at `ModGameData/<Mod>/` | none: the log says `No config paths are active, using default parser path '.../GameLite/GameData/'` |
| Write | `refurl=@BaseGame/<path below GameData>` | a relative refurl, as vanilla writes it (`../ItemPrototypes.cfg`) |
| The other form | relative refurls (`../X.cfg`) break on console since game 2.0, even when PC works | `@BaseGame/...` should fail with `references config path ... which is not active` (from s2cfgtojson's notes, not tested in game) |

- **A mod shipped both ways needs different refurls per build.** Generate the cfg once per target; never copy one
  build's files into the other.
- Zone Kit example: `{refurl=@BaseGame/ItemPrototypes/ConsumablePrototypes.cfg;refkey=TemplateConsumable}`.
  `@<ModName>/...` addresses a mod's own ModGameData tree.
- To see which case a build is in, search the log for `config path`. A Zone Kit mod logs
  `Activated config path 'BaseGame' at '.../Content/GameLite/GameData/'` and
  `Activated config path '<Mod>' at '.../Mods/<Mod>/Content/GameLite/ModGameData/<Mod>/'` when its Game Feature
  starts (s2cfgtojson's notes).
- Seen in a Zone Kit cook (harness): a relative refurl resolves only inside its own config path. From
  `ModGameData/<Mod>/EffectPrototypes/X.cfg`, `{refurl=../EffectPrototypes.cfg}` reached nothing, with no warning; the
  cook then failed with `FParserNode.EnsureNoEntryChildNotExist ... has missing child for Key '<name>'` once per
  unfilled key. `{refurl=@BaseGame/EffectPrototypes.cfg; refkey=...}` fixed it.
- Kept side by side: the harness also states that a Zone Kit mod's patches under `GameLite/GameData/` are already in
  `BaseGame`, so relative paths work there (observed in the cook, on PC). The field report says relative refurls in a
  Zone Kit mod break on console even when PC works, and does not separate the two folders. In a Zone Kit mod write
  `@BaseGame/` in every file.

## 3. Shapes that overflow the stack
Sign: `EXCEPTION_STACK_OVERFLOW` while cfg loads. Both shapes come from the field report.
- **A `@BaseGame` refurl to the same file that is being patched** (a struct in `ObjPrototypes/<patch>.cfg` with
  `refurl=@BaseGame/ObjPrototypes.cfg`) can loop in the Zone Kit until the stack overflows. Use `{bpatch}`, or write
  the struct out in full. Some files (EffectPrototypes) are reported fine; the cause is unknown.
- **A refkey left without its refurl** looks in the same file and can also overflow. When you remove or rewrite a
  refurl, check that every refkey that relied on it still finds its parent in the file.
- Kept side by side: for a plain (non-bpatch) replacement that must keep vanilla's fields, the harness and
  s2cfgtojson's notes give
  `CorpusMedlak : struct.begin {refurl=@BaseGame/ObjPrototypes/QuestObjPrototypes.cfg;refkey=CorpusMedlak}` plus the
  changed fields: a refurl into the same table (`QuestObjPrototypes.cfg` is itself merged into `ObjPrototypes.cfg`)
  and a refkey to the struct's own name. The harness marks it:
  whether a self-`refkey` resolves against the pre-replacement node is not verified. Prefer `{bpatch}`.

## 4. Arrays and bskipref
- `[*]` entries are numbered `[0]`, `[1]`, `[2]`... in file order and can be addressed by those indices.
- Add to a vanilla `[*]` list: bpatch the list with the NEXT FREE INDEX. A `[*]` inside a bpatch appends again on
  every apply.
- Change one entry of an inherited list: restate only that index.
- **`PreinstalledAttachmentsItemPrototypeSIDs`: never bpatch a single entry.** A partial bpatch replaced the whole
  list in game and stripped weapons' hidden built-in magazines and silencers (verified in game). Re-emit the full
  list without bpatch, keeping vanilla's `bskipref` entry.
- **Never put `X {bpatch}` and a struct with `refkey=X` in the same patch file.** The refkey resolved to the bpatch
  block (only its patched fields), not to the vanilla `X` (verified in game, 2.0). For an ObjPrototype that drops
  `AgentType` from the chain, and the game crashed with an access violation on load or when the NPC spawned, with no
  log line. Fix: give the new struct the vanilla `refurl`/`refkey` of `X` and restate the bpatched fields in it, or
  (untested) move the bpatch to another file.
- Some lists are parallel arrays indexed together (consumable `EffectPrototypeSIDs` / `EffectsDisplayTypes` /
  `ShouldShowEffects`): keep their indices aligned.
- `{bskipref}` appears to mark a struct or value that skips the inherited content (meaning inferred; vanilla
  `PreinstalledUpgrades { [0] = empty {bskipref} }` in `WeaponPrototypes.cfg`; 42 vanilla files use it). Keep it
  when re-emitting such a list.

## 5. DLC tables
- DLC cfgs (`DLC1`, `Deluxe`, `Ultimate`, `PreOrder`) are not in the Zone Kit's GameLite tree: extract
  them from the game paks (`Stalker2/Content/GameLite/DLCGameData/<DLC>/`; which pak: stalker2-paks).
- DLC trees are flat: one `ItemPrototypes.cfg` per DLC holds every item type. DLC items point back at the vanilla
  file through `refurl` (or inherit it by `refkey` from a DLC-local template), which is how to tell a DLC weapon
  from a DLC armor.
- A change to a whole prototype family (weapons, armors, attachments, consumables) needs the matching DLC patch too
  (`DLCGameData/{DLC1,Deluxe,PreOrder,Ultimate}/...`), or DLC variants behave differently.
- The pre-order backpack (`PreOrder/BackpackPrototypes.cfg`, `Backpack_PreOrder`) is the default backpack's mesh
  `SK_bpa_pla_01` with `MaterialPath = MI_bpa_sta_09_a_02_fov` (default: the mesh's own `MI_bpa_sta_09_a_01_fov`;
  both are base-game assets). A mod reproducing the player's backpack must copy the materials, not only the mesh.
  To test without owning the DLC, a dev pak can bpatch `Backpack_Default` / `Default` with that `MaterialPath`.

## 6. What cfg_verify reports
`node scripts/cfg_verify.mjs` tells the kind of mod from each file's path: a `ModGameData/` folder means a Zone Kit
mod, `GameData/` or `DLCGameData/` without it a plain pak. `--target pak|zonekit` overrides that for every file
given: use `--target zonekit` for a Zone Kit mod's patch files that sit under a plain `GameData/` path. Files in
neither folder get a NOTE asking for `--target`.

| WARN | Meaning |
|---|---|
| `refurl=... is relative in a Zone Kit mod` | section 2: use `@BaseGame/` (or `@<ModName>/`) |
| `refurl=@... in a plain ~mods pak` | section 2: no config path is active; use a relative refurl |
| `refurl=@BaseGame/... points at the file this one patches` | section 3: the loop shape (`<Table>/<file>.cfg` or `<Table>.cfg_patch_<Name>` pointing at `<Table>.cfg`) |
| `has refkey=... and no refurl, but ... is not a root of this file` | section 3; needs `--vanilla`: the parent is looked up in the file, in the vanilla table it patches and among records the mod's other files create there |

Adapted from the Stalker 2 modding harness (github.com/noahsemus/stalker2-modding-harness, MIT, copyright Noah Semus); this part was contributed there by sdwvit.
