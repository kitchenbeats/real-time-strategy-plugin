# RTS Skirmish Diagnostics

The plugin includes a reusable skirmish harness for playing or validating a reskin or generated
content set in a live match. `HumanRole` selects a human-controlled first base or the full-map
AI-vs-AI observer/caster experience.

## Runtime Classes

- `URTSSkirmishDefinition`: data asset describing starting bases, resources, player classes,
  townhall, worker, mineral field, geyser, and starting wallets.
- `ARTSSkirmishGameMode`: plugin game mode that spawns the match from a skirmish definition or
  inline definition data.
- `URTSSkirmishSubsystem`: runtime control surface for start/reset/speed.

All three mutation surfaces return success and are authority-only in Blueprint and C++. Customer
data assets and URL overrides are normalized again at runtime: participant, worker, expansion, and
resource-node counts have hard ceilings; layout and time-dilation values must be finite; starting
wallet grants are non-negative and bounded. These guards remain active even when C++ constructs a
definition without passing through editor metadata.

The bundled faction-agnostic bot only executes on authority. Its configurable think rate, roster
targets, construction timeout, placement clearance, and synchronous path-test budget are bounded
again at runtime, and per-controller build/scout/squad memory is released when a bot leaves or a
skirmish resets. Clients receive replicated results; they never run a second strategic simulation.

StarMaps uses `ARTSMatchGameMode` as a thin subclass that only supplies StarMaps asset defaults.
Reusable match setup remains in the plugin.

## Console Commands

- `rts.skirmish.start`
- `rts.skirmish.reset`
- `rts.skirmish.speed <0.01-16>`
- `rts.diag.audit`
- `rts.diag.audit.fix`
- `rts.diag.draw 1`

## Authoring Validation

Validate the ruleset before generating or running a skirmish:

```bash
UnrealEditor-Cmd Project.uproject -run=RTSValidateContentSet -ContentSet=/Game/MyRTS/DA_MyRTSContentSet
```

Validation messages are structured as guided findings:

- `id`: stable identifier for CI, docs, and issue reports.
- `symptom`: what is wrong.
- `likely cause`: the most probable authoring/runtime cause.
- `impact`: why it matters in-game.
- `fix`: the recommended developer action.
- `auto-fix`: present only when the plugin can make a high-confidence safe correction.

To apply only conservative authoring fixes in commandlets, pass `-Fix`. Add `-SaveFixes` only when
you want the commandlet to persist those changes to the content set asset:

```bash
UnrealEditor-Cmd Project.uproject -run=RTSValidateContentSet -ContentSet=/Game/MyRTS/DA_MyRTSContentSet -Fix -SaveFixes
```

Safe authoring fixes are intentionally narrow: missing content set id, empty generated output paths,
negative/default-invalid scalar values, chase radius below acquisition radius, and assigned-art
entries with no mesh that can be switched to greybox primitive mode. The fixer does not invent
health, unit sizes, resource ids, production links, or costs because those define game design.

Editor tools can call:

- `URTSContentSetEditorLibrary::ValidateContentSet`
- `URTSContentSetEditorLibrary::ApplySafeAuthoringFixes`
- `URTSContentSetEditorLibrary::GenerateContentSet`

## Runtime Audit

Use runtime audit once the map is loaded and the skirmish has spawned:

```text
rts.diag.audit
```

`rts.diag.audit.fix` applies high-confidence runtime fixes such as spawning missing default AI
controllers on orderable pawns or registering wallet resource types discovered from live resource
sources. Navigation findings are reported with guidance instead of blindly editing the map, because
the right fix can be a missing NavMeshBoundsVolume, an incorrect floor/collision setup, or an actor
spawned outside navigable space.

Common navigation findings:

- `NAV.NO_DATA`: no active nav data exists. Add or enlarge a NavMeshBoundsVolume and rebuild nav.
- `NAV.PAWN_OFF_MESH`: an orderable pawn is not on navigable space. Check spawn placement, collision,
  nav agent settings, and map floor coverage.
- `NAV.PAWN_FALLING`: a character is falling near navmesh. Check spawn Z, floor collision, gravity,
  and whether the pawn is being spawned on the navmesh plane instead of a walkable floor.

## Validation Loop

1. Create or generate units, buildings, resources, and their RTS components.
2. Run `RTSValidateContentSet`; use `-Fix` only for safe authoring cleanup.
3. Generate content.
4. Create a `URTSSkirmishDefinition` asset pointing at those classes.
5. Set a map to use `ARTSSkirmishGameMode` or a subclass.
6. Run at `rts.skirmish.speed 2`, `4`, or `8`.
7. Run `rts.diag.audit`; use `rts.diag.audit.fix` for high-confidence runtime cleanup.
8. Watch economy, production, construction, scouting, combat, minimap, fog, and stuck-unit
   diagnostics.

The harness is intentionally visual and runtime-driven: if a faction cannot gather, build, scout,
fight, or recover from blocked commands under this mode, the reskin is not production-ready.
