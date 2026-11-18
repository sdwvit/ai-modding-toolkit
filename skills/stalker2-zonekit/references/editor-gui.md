# The Zone Kit editor GUI

## Contents
- Launching, and the startup mod selector
- One launch per sitting
- Checkout: two entries, /Game and /<mod>
- Saving
- Writing steps for the user
- Paste blocks
- Finding things in a big material
- Verifying a save before the kit closes
- Logs
- Opening the world map
- Sequencer (Level Sequences)

## Launching, and the startup mod selector
- The user starts `Stalker2ModEditor.exe` from the kit folder and picks the mod in the startup **mod selector**. A new mod is
  created first with `CreatePlainMod.bat <name>` (call it by its full path in `<kit>`); it scaffolds `<kit>/Stalker2/Mods/<name>`.
  Keep mod names to letters, digits and underscores (the pack's cook script refuses anything else).
- Launching from an agent shell: Smart App Control can block the launcher stub ("An Application Control policy has blocked this
  file"). Start the real binary instead, detached, for example with PowerShell `Start-Process`:
  `<kit>/Stalker2/Binaries/Win64/Stalker2ModEditor-Win64-Shipping.exe "<kit>/Stalker2/Stalker2.uproject"`. The mod selector
  appears as usual.
- Never launch the editor while a cook runs, and never cook while it is open (it locks the cook's cache files).
- Loading takes several minutes (the editor parses every gamedata cfg on start).

## One launch per sitting
- The open kit edits other mods' assets too. Do every edit of the sitting in one launch, for all mods. Never write "close the kit
  and restart" between parts of a guide; close it only at the end, when a cook needs it.
- To switch mod: in the main window's top toolbar, the dropdown right of "Audio Modding" shows the current mod's name. Click it and
  pick the other mod.

## Checkout: two entries, /Game and /<mod>
- Ctrl+P opens the asset picker. Once an asset is checked out, the picker lists TWO entries: `/Game/...` (GSC's original) and
  `/<mod>/...` (the mod's copy). Edit the `/<mod>/` copy. Opening `/Game/...` and checking it out again overwrites earlier work.
- The copy lives at `<kit>/Stalker2/Mods/<mod>/Content/<original package path>/<asset>.uasset`. It is the only copy: back it up
  into `<repo>` after every session.
- To revert an edit, delete the asset from the mod's Content folder (the game falls back to vanilla once the mod is rebuilt).

## Saving
1. In the asset's own window: Apply (materials), then Save from that window's toolbar.
2. Ctrl+S in the MAIN window is "Save Level As" (it offers NewMap): the wrong thing.
3. The kit shows **Check Out Assets** ("You cannot directly modify the original game content ..."): accept it. That writes the
   copy into the mod.
- A base asset cannot be saved over, and a headless editor cannot do the checkout (no mod is selected). Without the GUI, the
  uncooked file is copied into the mod folder and saved there instead (SKILL.md points to the headless Python page).

## Writing steps for the user
- Click by click, plain language, one action per line. Start a guide with a short "words used" block (node, pin, wire, how to
  drag a wire) so every later step is unambiguous.
- Name each node by the title it shows, where it sits (inside which comment box, top / middle / bottom) and what feeds it:
  "the Multiply fed by Particle Color". Never internal names like `Multiply_5`, which a save renumbers anyway.
- Write every wire as FROM (node, right-edge output pin) TO (node, left-edge input pin), ideally to an input of the same name.
  Give pasted Custom nodes input names that say what feeds them (`Layer1_UV`, not `U2`).
- Find things by what is ON SCREEN, never by counting: "the module whose first row is Spawn Rate", not "the 8th module".
- Dragging a new wire onto an input pin replaces the old wire (no detaching needed). Dropping a node onto an existing wire to
  splice it in did not work: insert with two wires instead.
- Long wires: "zoom out until both nodes show".
- If a step names a value, give the exact number and where to type it.

## Paste blocks
Big graph changes go in as a block of nodes the user pastes, then one or two wires:
- Write the nodes as material-editor clipboard text. Clone node templates from the user's own Ctrl+A copy of a graph; a node class
  is the same in every material, so a template (a CollectionParameter, a TextureObject) can come from another material's copy.
- Pin names in that text: a VectorParameter's alpha output is `Output5`; WorldPosition's output is `XYZ`.
- Give each Custom node a Description: it is the title the user sees, and how you find the node again later.
- Import textures as 8-bit PNGs. A 16-bit PNG imports as linear 16-bit and looks washed out in game. ImageMagick writes
  16-bit for some inputs (an `xc:` canvas) unless given `-depth 8` (verified in game, 2.0).
- A TextureObject's SamplerType must match the texture (an sRGB colour texture: leave the property out).
- A second parameter node with the same name (and default) shares the live value: a pasted block can read an existing dial.
- Replacing a block: the user deletes the old block's nodes first, then pastes. Leftover parameter nodes with the same names but
  other defaults are an untested risk.
- Removing a block: re-wire the pins to their old sources, then drag a selection rectangle around the whole comment box, Delete.
- Paste a large block BELOW the lowest existing nodes, a small one beside its target.
- The paste logs dozens of `unresolved reference ... MaterialGraphNode_N` warnings, and "Custom material X missing input N" lines
  while the user wires: both harmless. The saved asset is the judge.

## Finding things in a big material
- The material editor toolbar's Home button jumps to the result node. Ctrl+F searches node titles, material function calls and
  named reroutes.
- Some GSC materials feed their result pins through NAMED REROUTES (a reroute with a name such as "Roughness" or "Opacity") or a
  plain reroute dot, not straight from the math. Insert new nodes between the reroute and the pin.
- Material errors show in the Stats panel and as a red ERROR! banner on the node.

## Verifying a save before the kit closes
Check the saved file while the kit is still open, so a fix costs one more click instead of a relaunch:
- `python scripts/mat_uasset_tree.py <saved .uasset>` (reads through a copy) prints the root wiring by node title, every Custom
  input and its code, the parameter defaults.
- Compare each new Custom node's code with what you intended, byte for byte. Normalise line breaks first (the editor stores CRLF);
  do the comparison in a script file, not an inline shell string, because shells eat backslashes.
- Check every intended wire WITH its output index, the masks, the dial defaults, that the old block's names are gone (grep the
  .uasset), and that the StateId differs from the backup's (renewed = the cook will recompile).
- The editor log's "Applying material" / "Saving and Compiling material" lines mark the user's Apply and Save.

## Logs
- The editor (and a headless run) logs to `%LOCALAPPDATA%/Stalker2/Saved/Logs/Stalker2.log`; the previous run is rotated to
  `Stalker2-backup-*.log` on start. The kit folder's own `Saved/Logs` holds only UnrealPak.log.
- A CrashReportClient log at launch is the crash monitor starting, not a crash.

## Opening the world map
File > Open Level > `Content/_Stalker_2/maps/_Stalker2_WorldMap/WorldMap_WP`. It takes minutes and logs tens of thousands of
harmless LogActorPackSubsystem warnings.

## Sequencer (Level Sequences)
- Open the sequence with Ctrl+P and check it out into the mod shown in the main toolbar's dropdown. Save with the Save icon in
  the SEQUENCER's own toolbar.
- Niagara parameter tracks go on the COMPONENT row: `+` on the actor row, pick the Niagara component, expand it, then `+` on
  that component row and pick the parameter. `+` on the actor row offers the parameters too, but that gives a useless second
  pair: delete those. Details panel values cannot be keyed for user parameters.
- Python cannot set a Niagara parameter track's parameter, so these tracks need the GUI. Read the result from the saved file
  before the user closes the kit.
- In the saved file, float tracks sit on the Niagara component's binding (a child under the spawned actor), not on the actor.
- Binding names can mislead: two different bindings can carry the same name. Match a binding by its spawnable's
  name and template, never by the binding name or a zero GUID.
- A kit save rewrites the sequence with its spawnables moved into Possessables (custom spawnable bindings, no Spawnables array).
  That still runs in game.
