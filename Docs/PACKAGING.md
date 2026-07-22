# Packaging a Project with Real-Time Strategy

Package only after the bundled reference and the project's generated map both work in Editor. A
successful compile or PIE session does not prove that a cooked Shipping build contains the right
map, project-owned presentation assets, platform binaries, or runtime configuration.

## Packaging preflight

1. Use UE 5.8.x and an installed plugin package built for the target platform and engine
   distribution. Do not reuse binaries from another engine version.
2. Save all project Content Sets, animation sets, custom Blueprints, maps, and configuration.
3. Run **Validate Content Set** and resolve every error. Review the **RTS Authoring** Message Log;
   do not package after a partially applied or failed presentation change.
4. Run **Generate Playable RTS Content** after the final data or presentation change. Confirm the
   generation manifest and generated assets are in the intended project-owned `/Game` folder.
5. Open the generated `L_<ContentSetId>_Starter` map and exercise selection, movement, gathering,
   construction, production, combat, death, match completion, and restart.
6. Set that map under **Project Settings > Maps & Modes > Game Default Map**. Generation does not
   change project-wide startup settings.
7. Make sure every custom mesh, material, texture, sound, `RTSAnimSet`, Animation Blueprint, and
   other soft-loaded asset is reachable from the project's cooked content policy. Explicitly add
   project maps or asset directories to the project's packaging settings when the project loads
   them only by name or custom soft-reference catalogs.
8. If project C++ includes plugin headers, add `RealTimeStrategy` to the consuming module's
   dependencies. Add `RealTimeStrategyExamples` only for code that uses those examples.
9. Package into a clean output directory, launch that output directly, and retain its logs. Do not
   treat a staged editor process or a source-project launch as packaged evidence.

## Blueprint-only packaging

A Blueprint-only project does not need a game module merely to use the plugin. Install and enable
the correct precompiled artifact, generate project-owned content, select the startup map, then use
Unreal's normal **Platforms > Package Project** workflow. If Unreal reports that plugin modules are
missing or were built for another engine version, do not convert the project to C++ as a workaround;
install a matching plugin artifact or use a supported source-build workflow.

## C++ packaging

Build the game target with the compiler and SDK required by UE 5.8 for the target platform before
packaging. Keep editor-only authoring code out of runtime modules: game runtime code depends on
`RealTimeStrategy`, while Content Set editor automation belongs in an editor module that may depend
on `RealTimeStrategyEditor`. Compile and package both Development and Shipping configurations when
qualifying a product; Shipping is the customer runtime result.

## Downloaded release archive intake

Release publication produces one canonical release set. `<version>` is the descriptor `VersionName`
and `<12-char-commit>` is the lowercase release commit prefix:

- `RealTimeStrategy-<version>-UE5.8-Mac-<12-char-commit>.zip`
- `RealTimeStrategy-<version>-UE5.8-Mac-<12-char-commit>.zip.sha256`
- `RealTimeStrategy-<version>-UE5.8-Mac-<12-char-commit>.release-verification.json`
- `RealTimeStrategy-<version>-UE5.8-Mac-<12-char-commit>.release-intake-schema3.py`
- `RealTimeStrategy-<version>-UE5.8-Mac-<12-char-commit>.downloaded-qualification-schema3.py`
- `RealTimeStrategy-<version>-UE5.8-Mac-<12-char-commit>.release-set-schema1.json`

The release-set manifest binds the canonical names, byte sizes, SHA-256 values, and contracts of
the ZIP, ZIP checksum, detached receipt, intake verifier, and downstream qualifier. It is published
last and is the sole publication-readiness marker. A directory without that manifest is incomplete;
an existing manifested set is immutable and is never repaired or upgraded in place.

Fab's **Project File Link points directly to the canonical ZIP**, whose archive contains exactly one
`RealTimeStrategy` plugin. The checksum, detached receipt, executable verification tools, and
release-set manifest are authenticated adjacent release evidence; they are not embedded in the ZIP
or substituted for the one-plugin Fab download. This separation preserves Fab's archive structure
while allowing operators to authenticate and qualify the exact downloaded bytes.

Do not manually extract a downloaded release archive. On macOS, intake schema 3 requires the exact
manifested release set. Obtain the expected release-set manifest SHA-256 through a publisher or
marketplace channel independent of the downloaded files. First authenticate the manifest, then use
its `intakeVerifier` record to authenticate the intake script *before* Python executes that script:

```text
shasum -a 256 /path/to/RealTimeStrategy-<version>-UE5.8-Mac-<12-char-commit>.release-set-schema1.json
shasum -a 256 /path/to/RealTimeStrategy-<version>-UE5.8-Mac-<12-char-commit>.release-intake-schema3.py
```

Only after both values match, run:

```text
python3 /path/to/RealTimeStrategy-<version>-UE5.8-Mac-<12-char-commit>.release-intake-schema3.py \
  --release-set-manifest /path/to/RealTimeStrategy-<version>-UE5.8-Mac-<12-char-commit>.release-set-schema1.json \
  --expected-release-set-manifest-sha256 <independently-trusted-lowercase-64-hex> \
  --work-root /fresh/path/RTSReleaseIntake \
  --json
```

No co-downloaded manifest or `.sha256` file proves origin by itself. The one expected manifest hash
must come from an independent authenticated channel. The trusted manifest then binds every member
as one release set and prevents mixing individually valid artifacts from different releases.

The source-independent intake tool does not import the release publisher or rely on the repository
source tree. It takes stable no-follow snapshots and fails closed on mutation, links and special
files, non-canonical ZIP metadata, path traversal, Unicode/case-fold/prefix collisions, Windows
path hazards, encrypted or unsupported entries, unexpected roots or order, and commercial package
contract drift. Schema 3 accepts only the canonical Mac product and verifies the exact UE 5.8
descriptor/version, commercial-content, commercial-archive, third-party-BOM, and public-header
manifests; documentation/config/resources; precompiled receipt closure; and universal arm64+x64
Mach-O products. `Config/CommercialArchive.v1.json` must classify every package file exactly once.
The independently recomputed Fab pre-ZIP gate is measured in decimal bytes with a maximum of
`15,000,000,000`, excluding only top-level `Saved` and `Intermediate` as specified by Fab; the full
package remains classified even where a role is outside that measurement.

`Config/ThirdPartyBom.v1.json` binds exact adapted-file mappings, exact non-basename lineage,
reproducible pinned basename-lineage inventories, component identities, and nested local acquisition
manifests. The UIKit acquisition inventory contains 120 repository assets:
117 are admitted to the commercial package and three Python authoring recipes remain repository-only.
That boundary is verified rather than inferred from filename extensions. The BOM deliberately keeps
human legal review open for the CC-BY credit/acquisition chain. It also records the
`crowd_pathfinder` reference as `NOASSERTION`, consulted-only, and mapped to zero shipped files.
These accurate flags are release inputs, not a claim of legal approval.

The inherited icon and renamed Unreal asset remain byte-bound. Descriptor lineage is instead bound
to an exact semantic projection because BuildPlugin deliberately reformats the descriptor and sets
`Installed=true`; the verifier rejects source-style or otherwise invalid packaged semantics.

Intake also enforces these hard resource limits: archive smaller than 4 GiB; at most 8,192 members;
at most 2 GiB per expanded member and 6 GiB expanded total; compression ratio at most 2,000:1;
paths at most 1,024 UTF-8 bytes and 32 components; checksum at most 256 bytes; detached receipt at
most 256 KiB; and central directory at most 8 MiB. Use a new, private work root. Any failure removes
that entire root. Success removes temporary snapshots and returns only
`<work-root>/RealTimeStrategy` plus `<work-root>/intake-receipt.json`; copy that promoted plugin
directory into the project.

This verifier, its immutable publication contract, and adversarial tests establish the intake
boundary. `Tools/maintenance/qualify_rts_downloaded_release.py` implements the separate downstream
Mac UE 5.8 gate without importing repository tools, reading Git, or consuming the unpublished
BuildPlugin producer receipt. Authenticate the manifest first and the downstream script against its
`downloadedQualifier` record before execution. The trusted script rechecks the one release-set
binding, revalidates the promoted tree, and copies it byte-for-byte into a fresh
content-only project; requires the exact 78-test installed RTS inventory; packages the unchanged
checked-in starter map in Development; and requires a natural-exit, headless packaged verdict with
exactly 100 ready units, the 20/80 worker/combat composition, at least two owners, zero spawn
failures, and zero alarms. Its receipt binds the staged map, pak/utoc/ucas bytes, cooked registry,
target receipt, Mach-O executable, command logs, runtime verdict, and build-root-only plugin delta,
then rereads and revalidates those artifacts.

The downstream schema-3 qualifier independently reclassifies the promoted and installed package,
revalidates the complete third-party BOM and nested inventories, and permits post-build package
differences only in Unreal-owned build roots. It requires the immutable archive-policy projection
and the BOM summary to remain identical across intake, installation, and final receipt sealing.

The downstream implementation and adversarial tests are not evidence that an actual customer/Fab
download, independent trust-channel lookup, clean-machine install, or live Unreal run has occurred.
No final authenticated release set currently exists for that live run. The internal lifecycle
qualifier still depends on source Git and an unpublished producer receipt and must not be cited as
downloaded-customer evidence. Launcher/Fab acquisition remains a separate external acceptance gate.

## Content and reskin checks

- Greybox content and the bundled reference must remain self-contained under the plugin and Engine
  content. They must not acquire a `/Game`, Manny, StarMaps, or third-party dependency.
- Customer-generated assets may reference customer assets under `/Game`; that is the supported
  reskin contract. Move or rename those assets through Unreal's Content Browser and fix redirectors
  before the final package.
- Direct Anim Set clips must be saved, playable, full-pose, in-place, and authored for the selected
  mesh skeleton. Test idle, locomotion, labor, attack, hit reaction, and death where authored.
- A Custom Animation Blueprint must compile for the selected skeleton and consume the published
  locomotion/action state as designed. A compatible class does not by itself prove pose quality,
  foot plants, materials, LODs, collision fit, or RTS-camera readability.
- Manny, Mixamo, Meshy, marketplace, and studio assets remain subject to their licenses. The plugin
  package does not grant redistribution rights for customer content.

### Shipped commercial-content inventory

The plugin's mounted `Content` payload is bound by
`Config/CommercialContent.v1.json`. Schema 1 lists all 230 shipped Unreal asset files and binds the
canonical Content-tree digest and the canonical manifest-record digest. Package verification fails
when a mounted asset is missing from the manifest, an entry names a nonexistent asset, an asset is
classified more than once, or the recorded count or digest no longer matches the exact Content
tree.

The manifest records a customer-facing **purpose** and an honest **retention basis**. A retention
basis identifies why the bytes remain in the commercial package; it is not a dependency-graph claim:

- `framework-runtime` / `framework-product-contract` identifies built-in framework content retained
  as part of the plugin product contract;
- `starter-vertical-slice` / `starter-reference-contract` identifies the reproducible Content Set,
  generated starter, map, and readable greybox presentation;
- `blueprint-extension-example` / `extension-example-contract` identifies small customer-facing examples
  that are valuable as extension entry points without pretending that the starter loads them; and
- `customer-reskin-library` / `documented-customer-library` identifies optional content intended
  for customers to browse, copy, replace, or use while reskinning.

Purpose or retention basis is not proof that an asset is loaded by the starter. In particular, the
69 fonts, icons, and panel textures under `UI/Kit` are an
optional customer-reskin library, not a dependency required for the default RTS runtime. Their
commercial justification is the documented, editable-source-backed reskin surface. Reviewers must
not relabel that library as runtime-required merely to satisfy an unused-asset review.

Every group declares `RTS.Packaging.CommercialContentManifest`. That installed automation contract
enumerates and loads every declared asset and is the common structural/load qualification; any
additional contracts record narrower purpose-specific checks. Neither contract kind implies that
the starter references an optional asset at runtime.

This inventory is an engineering classification and byte-integrity contract. It does not approve
copyright, trademark, font, source-art, or redistribution rights. Those remain part of the separate
legal/BOM review, and Fab acceptance remains external. Any addition, removal, rename, or purpose
change requires deliberate review of the manifest and its qualification contracts before another
release artifact is built.

## Multiplayer and dedicated-server projects

Package and exercise every topology the project intends to ship. Use server-authoritative request
paths and the documented Replication Graph policy; do not infer listen-server or dedicated-server
correctness from standalone play. Projects enabling Iris must supply and qualify an equivalent
per-connection fog/private-state filter or disable Iris. Lobby, matchmaking, accounts, backend
services, anti-cheat, campaigns, and persistence are integrations outside this plugin's v1 scope.

## Platform claim boundary

The plugin is UE 5.8-first. A local successful package proves only that exact engine build, host,
target, RHI, content fixture, and configuration. The current commercial target is Win64 x64 and
macOS Apple Silicon, but a target may be advertised only after the immutable candidate passes its
required platform qualification. Intel Mac, Linux, consoles, mobile, XR, and other engine branches
are not inherited claims. Check [Supported Product Contract](SUPPORTED_PRODUCT_CONTRACT.md) before
publishing compatibility wording.

## Package acceptance

On a clean machine or clean user account, verify that the packaged application:

- starts on the intended generated map without editor content or source-project paths;
- creates the HUD and expected human/AI participants;
- completes the project's economy and combat loop and can restart cleanly;
- renders the selected presentation in every required gameplay state;
- emits no fatal errors, missing-package messages, Blueprint compile failures, or repeated runtime
  warnings; and
- meets the project's declared performance, bandwidth, memory, and soak budgets on named hardware.

Use [Troubleshooting](TROUBLESHOOTING.md) for failures and
[Performance and Soak](PERFORMANCE_SOAK.md) for evidence requirements.
