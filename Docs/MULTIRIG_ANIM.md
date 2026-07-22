# Vendor-Neutral Skeletal Animation

The runtime animation contract is skeleton-specific but vendor-neutral. The plugin does not ship
an Anim Blueprint tied to StarMaps, Manny, Mixamo, Meshy, or another project's `/Game` content.

## Guided editor workflow

For a generated unit, select one `RTS Content Set` in the Content Browser, right-click, and choose
**RTS Authoring > Apply Imported Unit Presentation...**. The modal asset pickers provide an
Unreal-native, no-Python workflow:

1. Choose the unit id.
2. Assign exactly one customer-owned skeletal or static mesh.
3. For a skeletal mesh, assign an `RTSAnimSet` and choose Direct Anim Set or Custom Animation
   Blueprint. Custom mode also requires an Anim Blueprint compiled for the selected skeleton.
4. Optionally assign a portrait, then apply.
5. Run **Validate Content Set**, followed by **Generate Playable RTS Content**.

The apply operation is undoable and atomic. It rejects missing assets, mixed static/skeletal input,
animation clips from another skeleton, incompatible Anim Blueprints, and units backed by a complete
existing actor class. Customer assets are referenced in place and are never copied or modified.
Editor Utility Blueprints can use **Apply Imported Unit Presentation** for the same validated
transaction.

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
capsule; the visual component does not own collision. Skeletal and static meshes with valid bounds
are uniformly scaled to fit its authored diameter and height, then translated so the imported bounds
center sits on the capsule origin. Aspect ratio is preserved, so reskinning does not distort the character. Empty,
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
valid; triangle and LOD-count budgets belong to project performance qualification, not compatibility
validation. Engine-generated transient meshes used by automation are excluded from this persistent
asset contract.

Regeneration is safe across reskins. Switching static to skeletal presentation removes the
generator-owned static visual and creates the animation driver; switching back removes the owned
animation component and clears the native skeletal mesh. A customer component with the same reserved
name but the wrong type is reported as a conflict instead of being deleted. Each successful generation
reapplies the authored capsule and refreshes `CharacterMovement` navigation-agent dimensions.

## What “rig qualified” means

Vendor names are not compatibility rules. A rig family is qualified only for the exact
customer-owned fixture, import settings, target skeleton, engine/platform, and animation mode that
were exercised. Passing the read-only validator proves structural compatibility; it does not prove
retarget pose quality, foot plants, skin weighting, cloth, facial deformation, IK, or camera
readability.

For each licensed fixture a release candidate should retain this evidence outside the plugin:

1. Source/vendor asset identity, license owner, import settings, target skeleton, and retargeter
   revision.
2. A clean validator result for the exact mesh, Anim Set, mode, and optional Anim Blueprint.
3. Successful Content Set generation, Blueprint compile, cook, package, and isolated launch.
4. Rendered review at the intended RTS camera for idle, walk, run, turn, attack, labor, hit, death,
   selection, fog visibility, and at least one dense group movement scene.
5. Regeneration proof after swapping away from and back to the fixture, with no stale generated
   visual components.

The bundled vendor-neutral greybox baseline and engine-owned structural fixtures are automated.
The optional UE 5.8 Third Person presentation has its own clean-project acceptance path. No Mixamo,
Meshy, marketplace, or arbitrary studio rig is claimed as commercially qualified until a
customer-licensed fixture completes all five items above on the named release platform. The
repository References corpus contains architecture and gameplay references, not redistributable rig
fixtures, so it cannot substitute for that evidence.
