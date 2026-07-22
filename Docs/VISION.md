# Vision and fog of war

The plugin ships a complete grid-based vision stack: team visibility, remembered terrain, frozen
last-seen building ghosts, observer reveal, a post-process fog renderer, and optional terrain-height
occlusion. The starter reference map already contains the required actors and material.

## Blueprint setup

1. Use an RTS GameMode/GameState. They automatically spawn the Vision Manager and one Vision Info per
   team; do not place duplicate manager or team-grid actors in the map.
2. Place one **RTS Vision Volume** around the playable map and set **Size In Tiles**. A 256 x 256 grid
   is a strong starting point for a full RTS map.
3. Place one **RTS Fog Of War Actor** and an unbound post-process volume, then assign that volume and
   the render material on the fog actor. The generated starter map does this already.
4. Add **RTS Vision Component** to every unit that reveals fog and **RTS Visible Component** to every
   actor whose visibility follows fog of war.
5. Assign the shipped `M_RTSFogOfWar` material to the fog actor, or start from it when authoring a
   custom visual treatment. Its runtime contract is `VisibilityMask`, `OneOverTileSize`, `WorldMin`,
   and `WorldSize`.

The volume exposes tile/world conversion, independent rectangular tile dimensions, grid origin,
height-grid readiness, and initialization progress to Blueprint. `On Height Grid Ready` is available
for loading screens or custom match-start orchestration.

For runtime display changes, use `Configure Fog Policy`, `Configure Post Process Renderer`, and
`Set Reveal All`. Each operation reports whether it committed a change. Renderer swaps are atomic:
invalid cross-world actors, partial volume/material pairs, and failed resource creation leave the
working fog texture and renderer untouched. `Set Reveal All` remains a local presentation feature;
it does not grant gameplay vision to a team or replicate hidden information.

Custom local presentation effects can call `Set Client Hide Reason` on an RTS Visible Component.
Use one valid gameplay tag per effect: reasons stack, clearing one reason never reveals an actor that
another effect still hides, and the function reports whether it changed the active reason set.

## Terrain-height occlusion and startup cost

**Sample Terrain Heights** makes cliffs and raised ground block vision. Sampling is incremental and
bounded by **Height Traces Per Frame** (1024 by default), preventing a production 256 x 256 grid from
issuing 65,536 collision traces in one frame. Vision casters wait for the complete authoritative grid
and refresh automatically when it becomes ready.

- Flat map: disable **Sample Terrain Heights**. The height grid is ready immediately and performs no
  collision traces.
- Loading screen: call **Complete Height Grid Immediately** when a synchronous bake is preferable;
  its return value confirms that the initialized grid is ready.
- Seamless match entry: leave incremental sampling enabled and use **Get Height Sampling Progress**
  to drive a loading indicator if desired.
- Flying, omniscient, or special units: enable **Ignore Height Levels** on their vision component.

Terrain that participates in height occlusion must block the volume's **Height Level Trace Channel**.
Use a dedicated project collision channel if other world-static decoration should not affect sight.
The vertical query interval follows the volume's authored Z bounds plus **Height Trace Padding**, so
translated worlds and elevated maps do not depend on hard-coded absolute heights.

## C++ extension points

`ARTSVisionVolume` owns the grid projection and height cache. `ARTSVisionInfo` owns one team's known
and visible tiles. `ARTSVisionManager` registers vision/visible actors and applies team state.
`ARTSFogOfWarActor` converts the local team grid into a clamped transient texture. These layers can be
subclassed independently; gameplay code does not need to depend on the shipped fog material.
