# Editor Python, asset jobs, big graphs and editor traps

Adapted from the Stalker 2 modding harness by Noah Semus (github.com/noahsemus/stalker2-modding-harness, MIT).
How to start editor Python at all (headless runs, the log, what a headless run cannot save): stalker2-zonekit.

## Contents
- Editor Python: what works
- Headless and the open editor
- Kit scripts in this skill
- Asset jobs: the method
- Replacing a big graph
- Editor traps

## Editor Python: what works
- Graphs cannot be authored from Python, only pasted. `BlueprintEditorLibrary.add_member_variable` adds variables,
  `reparent_blueprint` reparents. Blueprint "Float" variables are doubles.
- CDO flags such as `AutoReceiveInput = Player0` on an actor Blueprint can be set from Python.
- A SoftClassProperty only accepts a real class object from Python; the saved value is its soft path.
- `unreal.new_object(cls, outer)` works for small sub-objects (such as `PlayerMappableKeySettings`). Setting properties on
  freshly created modifier or trigger objects fails with "cannot be edited on templates": MOVE the real objects from a
  duplicated source instead (`rename(outer=...)`).
- In the live editor, `EditorAssetLibrary.duplicate_asset` and `AssetTools.duplicate_asset` return None for `/Game`
  sources: run duplication headless, or edit existing mod assets in place.
- `delete_asset` on a never-saved temp asset crashes the commandlet.

## Headless and the open editor
- **Never edit an asset headless while the user's editor is running.** The editor keeps its loaded copy, and its next save
  overwrites the headless one. A variable added headless then shows on paste as "Could not find a variable named ..." /
  "The property associated with ... could not be found": add it again in the live editor (`add_member_variable`), then
  Refresh All Nodes and Compile. Make sure it is closed first (upstream asks the editor itself over remote execution
  instead of trusting a process name).
- A headless run mounts neither the mod plugin nor its assets into the asset registry. Upstream's scripts start with: if
  `EditorAssetLibrary.does_asset_exist("/<Mod>/<Mod>")` is false, call
  `unreal.GameFeaturesSubsystem.load_and_activate_game_feature_plugin("file:<kit>/Stalker2/Mods/<Mod>/<Mod>.uplugin",
  unreal.GameFeaturePluginLoadComplete())`; then
  `unreal.AssetRegistryHelpers.get_asset_registry().scan_paths_synchronous(["/<Mod>"], True)`.
- Python in the OPEN editor needs remote execution: Project Settings > Plugins > Python > Enable Remote Execution (not in
  Editor Preferences). It is a kit setting, so the user decides (stalker2-zonekit). It runs a script in seconds, and its
  interpreter persists between calls. Two traps: Unreal treats a command as a FILE path when its text contains
  `<something>.py` early (a docstring is enough): "Could not load Python file", so keep `.py` names out of a script's first
  lines. A script that raises leaves its previous output file in place: delete the output before the run, or you read
  stale results.
- Git Bash rewrites an argument that starts with `/` (`/Game/...`, `/<Mod>/...`) into a path under the Git install: use
  PowerShell or `MSYS_NO_PATHCONV=1`.

## Kit scripts in this skill
Both are editor Python: run them with stalker2-zonekit's headless runner (kit closed), inputs as environment variables set
in PowerShell. They only read. Each prints its output path as a `LogPython:` line.
- `scripts/export_t3d.py`: `S2_ASSET` = the asset's package path (`/Game/...` or `/<Mod>/...`), `S2_OUT` = output folder
  (default: the system temp folder). Writes `<AssetName>.t3d`: read a Blueprint or AnimBlueprint back after a paste, or
  feed `scripts/t3d_lift.py`.
- `scripts/dump_imc.py`: `S2_IMC` = one or more Input Mapping Context package paths separated by `;` (default
  `IMC_Exploration`), `S2_OUT` as above. Writes `imc_dump.json`: per row the action, key, `mappable`
  (`OverrideSettings/<Name>` on rows the player can rebind), modifiers, triggers and the action's own `action_triggers`.

## Asset jobs: the method
Upstream has a small editor script for each; they depend on its own runner, so write the few lines yourself.
- **Duplicate a game asset into the mod** (Blueprint graphs survive): headless, `EditorAssetLibrary.duplicate_asset(SRC,
  DST)`; if it returns None, fall back to `AssetToolsHelpers.get_asset_tools().duplicate_asset(name, folder,
  unreal.load_asset(SRC))`; then `save_asset(DST, only_if_is_dirty=False)` and check `does_asset_exist(DST)`. If DST exists
  already, delete it first. Duplicate, do not Move: a Move leaves a redirector at the old path, which would still act as
  an override.
- **Rename or move a mod asset** (editor closed): `EditorAssetLibrary.rename_asset(SRC, DST)`; if SRC still exists it is the
  redirector: `delete_asset(SRC)` (a redirector in the mod's Content gets cooked); then
  `save_directory("/<Mod>", only_if_is_dirty=True, recursive=True)`. List the referencers first
  (`asset_registry.get_referencers(SRC, unreal.AssetRegistryDependencyOptions())`). Afterwards open and Compile every
  referencer (compiled bytecode is stale until then) and re-check pins: a K2 pin default once failed to follow a rename.
- A string typed into a Blueprint node pin is compiled into the Blueprint's bytecode, so a plain export-string edit of
  the cooked `.uasset` does not change it. Edit it in the editor and recompile; an offline patch has to be a same-length
  byte swap (pad with spaces).
- **Set one class default on a Blueprint** (values awkward to enter in the UI, such as a soft class path):
  `cdo = unreal.get_default_object(unreal.load_class(None, "<BP path>.<BP name>_C"))`, `cdo.set_editor_property(PROP, v)`,
  `save_asset(BP, only_if_is_dirty=False)`. For `v` try `unreal.load_class(None, VALUE)`, then
  `unreal.SoftClassPath(VALUE)`, then the raw string.
- **A permanent compatibility marker** other mods can detect: an empty CurveFloat,
  `AssetToolsHelpers.get_asset_tools().create_asset(NAME, FOLDER, unreal.CurveFloat, unreal.CurveFloatFactory())`, then
  save. In the mod folder it lives at FOLDER/NAME; listed in OverridePackages.txt it cooks to `/Game/<same path minus the
  mod root>`, which is the path other mods load. Put an interface revision in the name (`XX_Interface_v1`), never the mod
  version, and never rename or remove it once released.

## Replacing a big graph
Never Ctrl+A / Delete a huge graph in the editor. Deleting an event graph of about 3,700 nodes / 14,000 pins in one go made
the editor reach about 118 GB of virtual memory and die ("Ran out of memory ... paging file is too small"), twice; at about
3,400 nodes it still went through. Splitting the graph over pages avoids the problem. For a graph that is already too big:
1. Keep a file copy of the old `.uasset`. The user closes that Blueprint's editor tab FIRST: an open tab follows the renamed
   asset, and the paste then lands in `<BP>_Old`.
2. With editor Python, `EditorAssetLibrary.rename_asset(<BP>, <BP>_Old)` (when its referencers are not loaded, no redirector
   is written and they keep pointing at the original path), then create a fresh Blueprint at the original path with your
   setup script (variables, defaults, class settings).
3. The user deletes its 3 default event nodes and pastes. Verify the graph, then `delete_asset(<BP>_Old)`.
4. If the paste landed in `_Old` anyway (it did three times: the editor kept serving the renamed asset when the original
   name was opened): delete the empty fresh asset, rename `_Old` back to the original name, run the setup script again
   (variables added after the rename are missing on `_Old`), recompile the referencers (their path references stay valid).

## Editor traps
- **Save before Compile** on big edits: a compiler crash loses unsaved work. Wait about 10 s after a save before cooking.
- After any asset rename, Compile every referencer and re-check pins. A redirector left in Content gets cooked: delete it.
- **Property Access nodes only inside AnimGraphs and transition rules**: one in an event graph crashed the compiler.
- The editor crashed when it auto-reopened an overridden `AnimBP_Player` at startup (compile-on-load before mod classes
  exist). Open such assets by hand. If a crash loop starts, delete the `OpenAssetsAtExit=` lines from
  `%LOCALAPPDATA%/Stalker2/Saved/Config/WindowsEditor/EditorPerProjectUserSettings.ini` (editor closed).
- Opening the pawn Blueprint once crashed while a modified `AnimBP_Player` sat in the same mod (the pawn editor previews
  the anim class): move the anim asset out temporarily if it recurs.
- **Overridden assets exist twice at runtime** (`/Game/...` and `/<Mod>/...`), and the editor redirects every `/Game` pick
  of an overridden asset to the mod path. Only the game-path object gets the game's runtime treatment (such as the
  player's rebinds): load it by path string when it matters.
- A Linked Anim Graph node only exposes an `In Pose` pin if the target's AnimGraph function has one; a pass-through
  AnimBlueprint never offered it (a dead end, see stalker2-animation).
- `Print String` is stripped in Shipping: trace with visible side effects.
