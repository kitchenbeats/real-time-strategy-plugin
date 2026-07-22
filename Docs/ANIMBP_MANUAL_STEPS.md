# Custom Animation Blueprint Integration

RealTimeStrategy does not require a particular skeleton, vendor, or Animation Blueprint. The
supported integration point is the unit definition in an `URTSContentSet` and the runtime
`URTSAnimComponent`. The interface is vendor-neutral; `SUPPORTED_PRODUCT_CONTRACT.md` lists which
customer-supplied fixtures have completed commercial qualification.

## Choose the animation mode

Each skeletal unit definition exposes these fields to Blueprint and C++:

- `SkeletalMesh`: the customer-owned mesh and skeleton.
- `AnimSet`: locomotion and action clips compatible with that skeleton.
- `AnimationMode`: `DirectAnimSet` or `CustomAnimBlueprint`.
- `AnimationBlueprintClass`: required only for `CustomAnimBlueprint`.

`DirectAnimSet` is the zero-graph path. `URTSAnimComponent` drives full-pose, non-additive animation
sequences directly and is the fastest way to validate a reskin. Additive clips require
`CustomAnimBlueprint` so the AnimGraph can provide their base pose.

`CustomAnimBlueprint` lets the Animation Blueprint own the final pose. In this mode,
`URTSAnimComponent` publishes RTS gameplay state without replacing the Animation Blueprint's
playback.

## Build a custom Animation Blueprint

Create an Animation Blueprint for the unit's skeleton and read the owning actor's
`RTSAnimComponent`. The component provides Blueprint-accessible state including:

- `GetNormalizedSpeed()`
- `GetLocoState()`
- `GetCurrentAction()`
- `OnActionRequested`

Cache the values during the Animation Blueprint update and use them to drive the customer's own
state machine, Blend Space, motion-matching database, Control Rig, or procedural animation stack.
Bind `OnActionRequested` for one-shot and looping action presentation. The `URTSAnimSet` remains the
data source for action variants, so gameplay code does not depend on graph layout.

## Assign it through a Content Set

In the Content Set editor:

1. Set the unit's visual policy to assigned art and select its skeletal mesh.
2. Assign an Anim Set containing clips retargeted to that mesh's skeleton.
3. Set Animation Mode to Custom Animation Blueprint.
4. Select the Animation Blueprint class.
5. Validate, generate, save, and run the generated scenario.

The validator rejects skeletal definitions that omit an Anim Set and custom-graph definitions that
omit their Animation Blueprint class. Generated unit Blueprints receive the skeletal mesh,
`URTSAnimComponent`, Anim Set, playback ownership, and Animation Blueprint class consistently.

## C++ authoring

Populate the same `FRTSUnitDefinition` properties from C++ or an editor commandlet, then invoke the
Content Set generator. Runtime extensions can use a customer-owned subclass of the generated unit,
stored outside the generated output root and selected by a customer-owned roster/map/configuration
seam that keeps its parent expected. Alternatively, `ExistingUnitActorClass` can supply a complete
stable project-owned class that does not inherit from the generated class it suppresses. Both paths
retain the `URTSAnimComponent` state contract. Never add components or presentation overrides
directly to the manifest-owned generated base.

## Acceptance checklist

- Test full-pose clips with `DirectAnimSet` first; validate additive clips through the custom graph.
- Confirm the custom Animation Blueprint compiles for the selected skeleton.
- Confirm idle, locomotion, actions, labor loops, hit reactions, and death presentation.
- Confirm distant units use appropriate animation LOD and update-rate optimization.
- Test standalone, listen server, and client views before shipping multiplayer content.

The plugin deliberately ships no vendor-specific character or project-bound Animation Blueprint.
A clean install therefore cannot acquire hidden `/Game` dependencies from the development project.
