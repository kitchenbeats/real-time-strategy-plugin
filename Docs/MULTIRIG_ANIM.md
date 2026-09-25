# Vendor-Neutral Skeletal Animation

Animation data is tied to a skeleton but not to any character vendor. The plugin does not ship an
Anim Blueprint tied to Manny, Mixamo, Meshy, or any project's `/Game` content.

## Guided editor workflow

For a generated unit, right-click your Content Set in the Content Browser and choose
**Use My Model for a Unit...** (in the **RTS Game** section). The **Apply Imported Unit Presentation**
window opens:

1. Choose the unit id.
2. Assign exactly one skeletal or static mesh from your project.
3. For a skeletal mesh, click **Create Anim Set from My Clips...**. Map Idle and Walk, then any
   optional actions you have. The pickers filter for the selected skeleton. Set the walk/run clips'
   intended travel speeds in centimeters per second, then **Save New Anim Set...** in your project.
   The saved set is selected automatically. You can also select an existing `RTSAnimSet`.
4. Choose **Direct Anim Set (No Anim Blueprint)** for automatic playback, or **Custom Animation
   Blueprint** for your own graph. Custom mode requires a compatible compiled graph with an
   evaluated `DefaultSlot` node for action montages.
5. Optionally assign a portrait, then **Apply Presentation**.
6. Choose **Check for Problems**, then **Generate Game**, then **Play**.

Import and save the mesh and animation sequences in your project's Content Browser before opening
the mapper. Preview the clips on that mesh in Unreal first. An empty clip picker usually means the
clips belong to a different skeleton; the mapper does not import or retarget them. Walk/Run Clip
Speed describes the speed the original animation was authored for, not the unit's gameplay movement
speed. Start Running At selects the gait threshold. After applying, issue a move, gather, build,
and attack command in the generated map to check the mappings from the actual gameplay camera.

The apply operation is undoable and atomic. It rejects missing assets, mixed static/skeletal input,
animation clips from another skeleton, incompatible Anim Blueprints, and units backed by a complete
existing actor class. Customer assets are referenced in place and are never copied or modified.
Editor Utility Blueprints can use **Apply Imported Unit Presentation** for the same validated
transaction.

The clip mapper creates a new Anim Set asset in your project and leaves the imported mesh and clips
unchanged. It refuses to overwrite an existing asset. Edit a saved Anim Set directly to change a
mapping, add variants, or configure contextual deaths. If clips use another skeleton, retarget them
in Unreal before selecting them here.

### Which events play the mapped clips?

| Mapping | Trigger |
| --- | --- |
| Idle / Walk / Run | Actual movement state and speed. Run falls back to Walk when no Run clip is assigned in direct mode. |
| Attack | An attack commits damage or launches a projectile. |
| Gather | The worker harvests a resource. |
| Build | The worker contributes to construction. |
| Death | The unit dies; direct playback holds the final pose. |
| Hit React | Eligible damage reactions, subject to action priority and reaction cooldown. |
| Heal / Ability 1 / Ability 2 | Your Blueprint requests the corresponding action through **RTS Anim Component > Play Action**. |

Use standard animation Sound or Niagara notifies for cosmetic cues at particular frames. An attack
clip's notifies do not control damage timing: combat commits the hit or projectile before requesting
the animation. For custom presentation logic, bind `OnActionRequested`; combat also exposes
`OnAttackUsed`. Keep gameplay effects in the authoritative ability/combat logic so animation culling
cannot prevent an attack or change its result.

Studios can call **Validate Imported Unit Presentation** first when building a batch importer or
custom Editor Utility Blueprint. This read-only node accepts the same mesh, Anim Set, animation
mode, and optional Anim Blueprint tuple as the apply operation. It changes no package and returns
localized findings prefixed by stable codes such as `[SKELETON_MISMATCH]`,
`[MISSING_LOD_MATERIAL]`, or `[EMPTY_ANIMATION_CLIP]`. A saved animation with zero playable duration
is rejected; reimport or replace it rather than allowing a visually frozen generated unit.

## Fast path: no Anim Blueprint

Create an `RTSAnimSet` for the unit's skeleton and assign full-pose, non-additive clips:

- Idle, Walk, and optional Run sequences
- authored walk/run speeds for foot-speed matching
- optional Attack, Build, Gather, Heal, Death, HitReact, and ability variants

Set the unit definition's animation mode to `Direct Anim Set`. The generated Blueprint receives an
`RTSAnimComponent`, which drives the skeletal mesh through Unreal single-node playback. Gameplay
events automatically select actions, labor loops can ping-pong, action variants avoid immediate
repeats, and locomotion playback follows actual ground speed.

Direct mode rejects additive and root-motion clips during imported-presentation and Content Set
validation. Single-node playback has no AnimGraph base pose, while authoritative RTS navigation—not
animation—must move the gameplay capsule. Use `Custom Animation Blueprint` for additive recoil,
layered hit reactions, aim offsets, and other graph-composed presentation; keep locomotion in-place.

This path is intended for Blueprint-only customers and rapid reskins. Retarget or import clips onto
the chosen skeleton, fill one Data Asset, regenerate, and play.

## Advanced path: custom Anim Blueprint

Set animation mode to `Custom Animation Blueprint`, assign a compatible Anim Blueprint class, and
keep the same `RTSAnimSet` for action data and gait thresholds. `RTSAnimComponent` stops driving the
base pose and publishes:

- `GetLocoState`
- `GetNormalizedSpeed`
- `GetCurrentAction`
- `OnActionRequested`

The customer graph can use state machines, Blend Spaces, motion matching, Control Rig, IK,
inertialization, linked layers, and montages while RTS gameplay remains unchanged. Action clips are
played through the configured montage slot when appropriate.

Read `GetCurrentAction` for automatic Gather/Build loops; `OnActionRequested` handles one-shot
requests and does not announce those labor transitions. A mapped action already plays through the
component's slot montage, so an event handler should not start another copy. The custom graph can
also consume actions without mapped clips. The editor import path still requires a compatible Walk
clip; completely clipless sets are available through runtime configuration. Follow the
[Animation Blueprint wiring guide](ANIMBP_MANUAL_STEPS.md#build-a-custom-animation-blueprint) for
the owner reference, state variables, slot connection, and custom ability calls.

## Runtime and code extension

Class-default setup remains editable in Blueprint Details. Live changes use validated transactions:

- `Configure Animation Driver` atomically swaps the Anim Set, owning skeletal mesh, direct versus
  state-publish mode, and montage slot. Invalid sets, unrelated meshes, missing publish-mode slots,
  additive direct-playback clips, and attempts to reconfigure a dead unit fail without disturbing
  the current driver.
- `Configure Animation Timing` validates evaluation and hit-reaction cadence before committing it.
- `Play Action` returns whether the request was accepted. In direct mode an authored clip is required
  except for the terminal death transition; a custom Anim Blueprint can intentionally consume a
  clipless action through `OnActionRequested`.

C++ uses the same exported functions, so runtime faction skins, transformations, equipment-driven
movesets, and custom action systems do not need reflection or direct property mutation.

## Reskin checklist

1. Import or retarget the new skeletal mesh and clips.
2. Create or duplicate an `RTSAnimSet` for that skeleton.
3. Assign the mesh and Anim Set on the `RTSContentSet` unit definition.
4. Choose direct playback or assign a skeleton-compatible custom Anim Blueprint.
5. Validate the Content Set. Missing animation data or a missing custom Anim Blueprint is reported
   before generation.
6. Regenerate the unit Blueprint and test locomotion plus every authored action.

## Generated unit fit and gameplay authority

`Unit Size` is the authoritative gameplay envelope. Its radius and height generate the exact Pawn
capsule; the visual component does not own collision. Skeletal meshes with valid bounds are
uniformly scaled to the authored height, so outstretched bind-pose arms do not shrink the character.
Static meshes are uniformly fitted inside the authored diameter and height. Both are translated so
their imported bounds center sits on the capsule origin. Aspect ratio is preserved. Empty,
non-finite, or degenerate mesh bounds fail validation, as does a height smaller than the capsule's
diameter; generation never silently enlarges the authored gameplay shape.

Generated visual collision is disabled. Skeletal presentation uses visibility-based pose ticking and
Unreal's update-rate optimization; authoritative movement, collision, navigation, and combat do not
depend on evaluated bones. If an import uses different forward/up axes or needs a deliberate framing
adjustment, set the unit definition's Blueprint-writable `Visual Rotation` and `Visual Offset`.
Rotation participates in the automatic fit before the rotated bounds center is resolved; offset is
then applied in capsule-local centimeters. Both values regenerate deterministically and non-finite
values fail validation. Test every imported art set in the target gameplay camera and animation mode
before release.

Persistent imported meshes are also validated at every rendered LOD. Static and skeletal LODs must
contain sections, every section must resolve an effective material, and skeletal `LODMaterialMap`
entries must resolve to valid mesh material slots. A deliberately low-detail single-LOD asset is
valid; triangle and LOD-count budgets are a performance decision for your project, not part of this
compatibility check. Engine-generated transient meshes used by automation are excluded from this persistent
asset contract.

Regeneration is safe across reskins. Switching static to skeletal presentation removes the
generator-owned static visual and creates the animation driver; switching back removes the owned
animation component and clears the native skeletal mesh. A customer component with the same reserved
name but the wrong type is reported as a conflict instead of being deleted. Each successful generation
reapplies the authored capsule and refreshes `CharacterMovement` navigation-agent dimensions.

## Checking a character before you ship it

A vendor name is not a compatibility guarantee. A character is proven only for the exact asset,
import settings, skeleton, engine version, platform and animation mode you tested. Passing
**Check for Problems** proves the setup is structurally valid; it does not prove retarget pose
quality, foot contact, skin weighting, cloth, facial animation, IK or readability from the RTS camera.

For each character you ship, keep this record in your own project:

1. Source/vendor asset identity, license owner, import settings, target skeleton, and retargeter
   revision.
2. A clean validator result for the exact mesh, Anim Set, mode, and optional Anim Blueprint.
3. Successful Content Set generation, Blueprint compile, cook, package, and isolated launch.
4. Rendered review at the intended RTS camera for idle, walk, run, turn, attack, labor, hit, death,
   selection, fog visibility, and at least one dense group movement scene.
5. Regeneration proof after swapping away from and back to the fixture, with no stale generated
   visual components.

The plugin's own tests cover its placeholder art and engine-owned test meshes, and Unreal's Manny
from the UE 5.8 Third Person template. The plugin makes no compatibility claim for any particular
Mixamo, Meshy, marketplace or studio character; check each one with the steps above.
