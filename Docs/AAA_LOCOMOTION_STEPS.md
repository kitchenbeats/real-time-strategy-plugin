# Locomotion Polish

Start with the supported `DirectAnimSet` mode described in `MULTIRIG_ANIM.md`. Once the chosen mesh,
skeleton, and clips work together, switch the unit definition to `CustomAnimBlueprint` for a
project-specific high-fidelity locomotion stack.

## Recommended layers

1. Use a Blend Space, motion matching, or a state machine driven by
   `URTSAnimComponent::GetNormalizedSpeed()` and `GetLocoState()`.
2. Add sync markers and a shared sync group to compatible cyclic locomotion clips.
3. Use inertialization or dead blending for transitions where appropriate.
4. Apply stride or orientation warping only when the rig and source animation support it.
5. Layer action animation from `OnActionRequested` over locomotion, using the unit's `URTSAnimSet`
   as the action-data source.
6. Add foot placement, aim, and look-at passes behind significance, distance, and animation-LOD
   gates.

## Rig independence

Do not hard-code Manny, Mixamo, Meshy, or project-specific bone names in shared RTS gameplay code.
Keep bone mappings, IK definitions, retargeting, and Control Rig classes in your own animation
assets. `URTSAnimComponent` intentionally publishes semantic gameplay state rather than a skeleton
contract.

## Performance acceptance

An RTS animation setup must be measured with the intended army size, not only with a hero unit in an
empty map. Validate:

- animation budget allocation and update-rate optimization;
- skeletal-mesh LOD transitions and tick options;
- visibility and distance-based update suppression;
- Control Rig, IK, warping, and montage cost at representative unit counts;
- server, listen-server, and remote-client behavior.

Use Unreal Insights, Animation Insights, `stat anim`, and the target platform's GPU profiler. Treat
frame-time and memory budgets as content acceptance requirements.

## Shipping rule

The plugin supplies the animation state API and data-driven authoring path. Customers supply or
license the character meshes and compatible animation content. Plugin assets must never reference a
development project's `/Game` packages; the automated packaging suite enforces this boundary.
