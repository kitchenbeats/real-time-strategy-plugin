# Troubleshooting and Support Boundaries

Start with the smallest reproducible baseline: UE 5.8, an unchanged matching plugin package, a new
project, and the bundled reference map. Preserve the failing project and logs before applying fixes.

## Plugin is absent or will not load

Check these in order:

1. The descriptor exists at `<Project>/Plugins/RealTimeStrategy/RealTimeStrategy.uplugin`. An extra
   archive or version directory between `Plugins` and `RealTimeStrategy` prevents discovery.
2. **Real-Time Strategy** is enabled in **Edit > Plugins** and the editor was restarted.
3. The plugin artifact matches UE 5.8, the host platform, architecture, and engine distribution.
   “Missing modules” or “built with a different engine version” is an artifact/build mismatch, not
   a Content Set problem.
4. The project has not merged files from two plugin versions. Close Unreal, restore one complete
   artifact, and retry.
5. For C++ projects, build with the UE 5.8-supported compiler/SDK and inspect the first compiler or
   linker error. Do not diagnose from the final aggregate failure alone.

The plugin enables Enhanced Input, Niagara, and Replication Graph dependencies through its
descriptor. If a project deliberately changes those plugin states or its replication stack, restore
the supported baseline before reporting a framework defect.

## Plugin content or starter map is hidden

Enable **Show Plugin Content** in the Content Browser settings, then open
`/RealTimeStrategy/Starter/Reference/Generated/Maps/L_RTSStarter_Starter`. If the object path is
still absent, verify that the installed artifact contains its `Content` directory and was copied
completely. Never copy only `Binaries` and the descriptor.

## Content Set action is missing

Confirm that the editor loaded `RealTimeStrategyEditor`, then create the asset through
**Add > Gameplay > RTS > RTS Content Set**. The **RTS Authoring** actions appear on the context menu
for selected Content Set assets. Runtime/Shipping builds intentionally do not load editor authoring
tools.

## Validation or generation fails

- Open **Window > Developer Tools > Message Log** and select **RTS Authoring**. Fix the first
  blocking finding, validate again, and generate only after validation succeeds.
- Use **Apply Safe Authoring Fixes** only for deterministic repairs. It does not invent design data
  such as health, costs, ids, production relationships, or art choices.
- An asset/type conflict at a generated path means the generator cannot prove ownership. Do not
  delete or overwrite it blindly. Move customer-owned assets outside the generated root, restore
  the correct generation manifest, or choose a clean output root.
- Do not edit manifest-owned generated bases. Put project logic in customer-owned child/composed
  classes outside the generated output and select those through documented roster/configuration or
  complete `Existing*Class` seams.
- If source control makes an output read-only, check it out or correct permissions before retrying.

For structured runtime inspection, use `rts.diag.audit`; `rts.diag.audit.fix` applies only the
runtime repairs documented in [Skirmish Diagnostics](SkirmishDiagnostics.md).

## Manny action cannot find compatible assets

The Manny profile expects the matching UE 5.8 Third Person template/Characters feature content in
the project. Manny is not bundled with the plugin. Add that UE 5.8 content, save it, and rerun
**Apply UE 5.8 Manny Presentation**. Do not redirect the expected paths to assets from another
engine release and assume compatibility.

## Imported character is rejected or animates incorrectly

- A skeletal mesh, `RTSAnimSet`, every assigned sequence, and a Custom Animation Blueprint must
  resolve to the same intended skeleton.
- Direct Anim Set requires saved, playable, full-pose, in-place clips and required locomotion data;
  it rejects additive and root-motion clips because single-node playback has no AnimGraph base pose
  and RTS navigation owns movement.
- Use Custom Animation Blueprint for layered/additive animation, aim offsets, IK, motion matching,
  linked layers, or montage composition.
- Check mesh bounds, materials at every rendered LOD, import axes, `Visual Rotation`, `Visual
  Offset`, gameplay capsule size, and RTS-camera readability. Structural validation cannot judge
  retarget pose quality, skin weighting, foot plants, cloth, facial deformation, or art direction.
- Mixamo, Meshy, marketplace, and custom studio assets are not interchangeable “rig families.”
  Record and qualify the exact licensed fixture, import settings, target skeleton, retargeter, and
  animation mode.

See [Vendor-Neutral Skeletal Animation](MULTIRIG_ANIM.md) for the complete checklist.

## Units do not move, gather, build, or fight

Run `rts.diag.audit` in the loaded map and inspect Output Log entries in `LogRTS`.

- `NAV.NO_DATA`: add or enlarge a `NavMeshBoundsVolume`, ensure the floor has walkable collision,
  and rebuild navigation.
- `NAV.PAWN_OFF_MESH`: check spawn placement, nav-agent dimensions, collision, and bounds coverage.
- `NAV.PAWN_FALLING`: check spawn height, gravity, floor collision, and walkable geometry.
- For economy/production failures, verify stable resource ids, accepted resources, costs, supply,
  ownership, technology requirements, and that Content Set capabilities match any complete
  `Existing*Class` composition.
- For client commands, verify that requests route through the owning RTS player controller and that
  gameplay mutation executes on server authority. A local UI availability check does not authorize
  a transaction.

## Editor works but packaged build fails

Confirm the generated map is the **Game Default Map**, all required project assets are included by
the cook, and the plugin artifact matches the package target. Search the packaged log for the first
missing package, Blueprint compile error, module error, or `LogRTS` failure. Reproduce in a clean
packaging directory. Follow [Packaging](PACKAGING.md); a PIE-only success is not a Shipping result.

## Disable, update, or uninstall recovery

Do not resave dependent project assets while the plugin is absent. Restore the same or a documented
migration-compatible plugin version, enable it, restart, preview migration, validate, and only then
save or regenerate. Uninstalling plugin code does not convert or delete customer assets that depend
on its classes. Follow [Install Lifecycle Qualification](INSTALL_LIFECYCLE_QUALIFICATION.md).

## Preparing a support report

Provide:

- plugin version and whether the package was modified;
- exact UE 5.8 build/distribution, operating system, architecture, target configuration, and RHI;
- Blueprint-only or C++ project type and enabled replication mode;
- minimal reproduction steps starting from the bundled reference or a new Content Set;
- the first relevant editor/build/packaged log error plus surrounding context;
- Content Set validation findings and whether regeneration completed; and
- for character issues, asset source/license, import settings, skeleton, retargeter, animation mode,
  and the exact failing action.

Support covers the documented plugin runtime, authoring tools, shipped content, and stable extension
contracts on qualified targets. It does not include modified engine source, undocumented internals,
unlicensed assets, arbitrary retargeting/art correction, customer backends, custom Iris filters,
third-party networking middleware, or project code that bypasses documented authority and lifetime
rules. See [Supported Product Contract](SUPPORTED_PRODUCT_CONTRACT.md) for the normative boundary.

