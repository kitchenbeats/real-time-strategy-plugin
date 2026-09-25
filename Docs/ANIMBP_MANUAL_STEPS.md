# Custom Animation Blueprint

RealTimeStrategy does not require a particular skeleton, vendor, or Animation Blueprint. The
supported integration point is the unit definition in an `URTSContentSet` and the runtime
`URTSAnimComponent`. The interface does not depend on any character vendor; see
[What's Supported](SUPPORTED_PRODUCT_CONTRACT.md#characters-and-art).

## Choose the animation mode

Each skeletal unit definition exposes these fields to Blueprint and C++:

- `SkeletalMesh`: your mesh and its skeleton.
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

Create an Animation Blueprint for the unit's skeleton:

1. In **Event Blueprint Initialize Animation**, use **Get Owning Actor**, then **Get Component by
   Class** with `RTSAnimComponent`. Cache that component reference. Guard it with **Is Valid** during
   preview and initialization, where there may be no gameplay actor.
2. In **Event Blueprint Update Animation**, read the component's state into graph variables. Build
   your locomotion state machine or Blend Space from those variables.
3. Connect your base pose through a **Slot** node named `DefaultSlot` to **Output Pose**. An unused,
   disconnected Slot node does not provide action playback. Compile and save the graph.
4. For actions implemented inside your graph, bind **On Action Requested** once during initialization,
   not every animation update. Use the action enum to select your one-shot state.

| Read surface | Meaning and use |
| --- | --- |
| `GetLocoState()` | Idle, Walk, or Run. Work loops publish Idle while the action identifies Gather or Build. |
| `GetNormalizedSpeed()` | Planar speed divided by the Anim Set's Run Speed Threshold, clamped to 0..1. Use a 0..1 Blend Space axis, or read actor velocity separately for a centimeters-per-second axis. |
| `GetCurrentAction()` | Current action, including sustained Gather/Build and terminal Death. Use this for labor-loop state transitions; None leaves the work state. |
| `OnActionRequested` | A one-shot request such as Attack, Hit React, Death, or a custom Play Action call. Automatic Gather/Build transitions do not broadcast this delegate. |

When an action has a mapped clip, the component handles its montage on `DefaultSlot`. Do not start
a second copy from the event handler or rely on callback ordering to determine montage position. When the action has no
mapped clip, the event can start your own graph animation; let that graph own its completion. The
component's clipless one-shot action state is brief, so polling it is not a substitute for binding
the request event. Idle/Walk/Run and clipless Gather/Build still publish their gameplay state for a
custom graph.

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

The Content Set/import workflow currently requires a compatible Walk clip even when your graph
owns locomotion. The guided **Create Anim Set from My Clips...** mapper requires both Idle and Walk
and creates a set suitable for direct playback. For additive custom-graph actions, edit the saved
Anim Set and validate it in Custom Animation Blueprint mode. A fully clipless Anim Set is supported
by the runtime `Configure Animation Driver` state-publish path, but is not accepted by the current
Content Set importer.

## Map your own gameplay events

Attack, damage reactions, death, gathering, and construction have automatic runtime hooks. Heal,
Ability 1, and Ability 2 mappings are presentation slots for your own gameplay code. After your
ability accepts an activation, obtain the unit's `RTSAnimComponent`, call **Play Action** with the
matching enum, and check its Boolean result. This requests presentation; it does not heal a target,
spend resources, or execute an ability. Keep those effects in the ability's authoritative logic.

Use ordinary animation Sound or Niagara notifies for cosmetic timing. Attack damage or projectile
launch occurs before the attack animation request, so placing a notify at the impact frame does not
move gameplay damage to that frame. See the [mapping/event table](MULTIRIG_ANIM.md#which-events-play-the-mapped-clips)
for the complete built-in mapping.

## C++ authoring

Populate the same `FRTSUnitDefinition` properties from C++ or an editor commandlet, then invoke the
Content Set generator. To add behavior to a generated unit, use its **Custom Blueprint** (a child
of the generated Blueprint that Generate keeps using). To replace the unit with a class you built
yourself, set **Replace With Class** (`ExistingUnitActorClass`); that class must not inherit from the
generated Blueprint it replaces. Both keep the `URTSAnimComponent` state that Animation Blueprints
read. Never add components or presentation overrides directly to a generated Blueprint; Generate
replaces them.

## Acceptance checklist

- Test full-pose clips with `DirectAnimSet` first; validate additive clips through the custom graph.
- Confirm the custom Animation Blueprint compiles for the selected skeleton.
- Confirm idle, locomotion, actions, labor loops, hit reactions, and death presentation.
- Confirm distant units use appropriate animation LOD and update-rate optimization.
- Test standalone, listen server, and client views before shipping multiplayer content.

The plugin deliberately ships no vendor-specific character or project-bound Animation Blueprint.
A clean install therefore cannot acquire hidden `/Game` dependencies from the development project.
