# Install and First Play

This guide is the shortest supported path from a Real-Time Strategy plugin package to a working
UE 5.8 project. Keep the plugin package unchanged until the baseline has opened and played once.

## Before installation

- Use Unreal Engine 5.8.x and a plugin package built for the host operating system and engine
  distribution. A package built for another engine branch or platform is not interchangeable.
- Close Unreal Editor and back up or commit the project.
- If starting from the canonical Mac release set, run the source-independent intake command in
  [Packaging](PACKAGING.md#downloaded-release-archive-intake). Obtain the one independently trusted
  release-set manifest SHA-256, authenticate that manifest, then authenticate the versioned intake
  tool against its manifest record before executing it. A co-downloaded manifest or checksum is not
  proof of authenticity. Do not manually extract or copy an unverified downloaded ZIP.
- Confirm that the promoted package contains one top-level `RealTimeStrategy` directory with
  `RealTimeStrategy.uplugin` directly inside it. Copy only the intake tool's
  `<work-root>/RealTimeStrategy` output, not the archive, intake work root, or staging directory.
- Remove or move any older `<Project>/Plugins/RealTimeStrategy` installation before copying the
  replacement. Do not merge two plugin versions.

## Install and enable

1. Create `<Project>/Plugins` if it does not exist.
2. Copy the complete packaged directory to `<Project>/Plugins/RealTimeStrategy`.
3. Open the project in UE 5.8. If Unreal asks to build project modules, allow it only when this is a
   C++ project using a source-capable plugin distribution and a supported compiler toolchain.
4. Open **Edit > Plugins**, search for **Real-Time Strategy**, enable it, and restart the editor.
5. In the Content Browser settings, enable **Show Plugin Content**.
6. Open `/RealTimeStrategy/Starter/Reference/Generated/Maps/L_RTSStarter_Starter` and press Play.

The reference map is the installation smoke test: it is a complete vendor-neutral greybox RTS and
does not depend on Manny, StarMaps, third-party art, project Python, or project configuration. If it
does not load and play, stop and use [Troubleshooting](TROUBLESHOOTING.md) before authoring content.

## Blueprint-only project

1. In the Content Browser choose **Add > Gameplay > RTS > RTS Content Set**.
2. Name and save the asset in a project-owned folder such as `/Game/RTS`.
3. Right-click it and choose **RTS Authoring > Validate Content Set**, followed by
   **RTS Authoring > Generate Playable RTS Content**.
4. Open the generated `L_<ContentSetId>_Starter` map and press Play.

New Content Sets contain a working economy/combat ruleset and can use plugin-owned greybox visuals,
so no art assignment is required. Generated assets belong to the project under `/Game`. Keep custom
Blueprints outside the generated output root; regeneration owns the generated bases and manifest.
For the complete ownership and extension rules, see [Starter Ruleset](STARTER_RULESET.md) and
[Extension Guide](EXTENSION_GUIDE.md).

## C++ project

Enable the plugin as above, then add the runtime module to the game module's `.Build.cs`:

```csharp
PublicDependencyModuleNames.Add("RealTimeStrategy");
```

Add `RealTimeStrategyExamples` only when project code intentionally derives from or includes the
optional compiled examples. The editor authoring module is not a runtime dependency. Regenerate
project files when the IDE or build workflow requires it, compile the project, and run the same
reference-map smoke test. Start from the exported headers and stable extension seams listed in
[C++ API Surface](CPP_API_SURFACE.md) and obey the authority, lifetime, and teardown rules in
[C++ Extension Contract](CPP_EXTENSION_CONTRACT.md).

## Choose a presentation workflow

The gameplay contract is independent of a character vendor. Validate after every presentation
change, regenerate, and test every authored action in the target gameplay camera.

| Goal | Customer action | Boundary |
| --- | --- | --- |
| Immediate playable baseline | Keep `UseGreyboxPrimitive`, or play the bundled reference map. | This is the guaranteed blank-project presentation. |
| UE 5.8 Manny | Start with the UE 5.8 Third Person template or add its matching Characters feature content. Right-click the Content Set and choose **Apply UE 5.8 Manny Presentation**. | Manny remains Epic/project content and is not redistributed by the plugin. The action validates the exact discovered assets before changing the Content Set. |
| Skeletal character without an Anim Blueprint | Import or retarget a mesh and full-pose, in-place clips for one skeleton; create an `RTSAnimSet`; use **Apply Imported Unit Presentation...** and select **Direct Anim Set (No Anim Blueprint)**. | Direct playback rejects incompatible skeletons, additive clips, root-motion clips, and missing required locomotion data. |
| Advanced custom graph | Import a mesh, compatible `RTSAnimSet`, and Animation Blueprint; use **Apply Imported Unit Presentation...** and select **Custom Animation Blueprint**. | The graph owns visual composition; RTS movement and gameplay authority remain in the plugin runtime. |
| Mixamo, Meshy, marketplace, or studio art | Use either imported-presentation mode after the project imports, retargets, and licenses the assets. | The workflow is supported, but a vendor name is not a compatibility guarantee. Each exact fixture, import, skeleton, and animation setup must be validated and visually qualified by the project. |

Detailed skeletal setup and validation rules are in
[Vendor-Neutral Skeletal Animation](MULTIRIG_ANIM.md).

## Before the first packaged build

Set the generated starter map as the project's **Game Default Map**, package for the intended
platform, and test the packaged executable rather than relying only on PIE. Follow
[Packaging](PACKAGING.md) for the complete preflight. The currently saleable platform and fixture
claims are controlled by [Supported Product Contract](SUPPORTED_PRODUCT_CONTRACT.md), not by an
editor success on one machine.
