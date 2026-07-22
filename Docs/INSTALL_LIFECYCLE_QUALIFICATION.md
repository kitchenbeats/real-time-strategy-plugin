# Install, Update, Disable, and Uninstall Qualification

The supported lifecycle preserves customer content and makes dependencies visible. Disabling or
uninstalling a code plugin cannot make assets based on its classes usable; it must never silently
rewrite or delete those assets. Back up and inventory the project before changing plugin state.

## Automated isolated workflow

Repository maintainers normally run `Tools/maintenance/release_rts_plugin.py --install-lifecycle`
from a clean committed revision. That non-publishing full gate builds the exact UE 5.8 artifact,
writes a producer receipt binding the BuildPlugin argv, source tree, engine, commit, runner, run ID,
and artifact bytes, then invokes `qualify_rts_install_lifecycle.py`. The parent runner predeclares the
complete release-harvest contract before producers launch and requires the resulting manifest to pass
semantic and cardinality validation before lifecycle success. `--archive` implies the same lifecycle
gate and cannot publish without its passing receipt. The qualifier refuses a source-tree
plugin, an uninstalled descriptor, a wrong engine version, staging directories, overlapping
engine/artifact/work paths, linked or
special filesystem entries, multi-link files, oversized trees, or a changed archived migration
fixture. It rejects pre-existing linked ancestors and verifies the newly created work root by inode.
Before artifact acceptance, a durable receipt binds a clean Git
commit and source manifest plus exact UE 5.8 Build.version, Editor, and UAT hashes. It invokes the same
commercial package verifier used by the release gate, including exact Source/Content/Resources
parity, distributable-document inventory, universal Mac binaries, parsed precompiled receipts and
their real dual-architecture outputs, and the Mac module receipt. It creates a
new content-only host beneath the caller's work directory and never edits the source project or
source artifact.

The workflow uses separate real Unreal processes to prove:

1. The complete artifact survives a local transport copy byte-for-byte.
2. A fresh project explicitly enables it and generates customer-owned starter content.
3. A new editor process reopens and validates that Content Set.
4. The immutable schema-0 fixture is copied unchanged into `/Game`, explicitly migrated to the
   current schema, generated, reopened, and regenerated.
5. Unreal's loaded Asset Registry inventories every `/Game` package with a direct plugin script or
   content dependency before disable/uninstall.
6. The project descriptor is atomically disabled, a new editor process opens without the plugin,
   and the complete customer Content tree remains byte-identical.
7. Re-enable and exact-artifact reinstall each reopen and validate both current and migrated
   Content Sets without changing their package bytes.
8. The reinstalled project builds, cooks, stages, packages, and archives the migrated starter map.

Every child process runs in a dedicated process group with a hard timeout. Success, failure,
timeout, and interruption all probe and terminalize the complete process group. A one-millisecond
macOS libproc sampler additionally retains every descendant PID it observes, including observed
children that later detach into another session. macOS does not provide this runner an atomic
job-object-style descendant boundary: a process that reparents between libproc samples is outside
the claimed evidence. Receipts therefore state `process-group-plus-1ms-libproc-observed-descendants`
instead of claiming unknowable complete descendant containment. SIGINT and SIGTERM become terminal
lifecycle failures instead of bypassing the receipt. Teardown has bounded TERM and KILL phases. Every process has a separate
hashed log. Fatal/error output
and any warning that has not been narrowly reviewed fail the run even when the process returns
zero. Timeout values are recorded and can be overridden with `--editor-timeout-seconds`,
`--package-timeout-seconds`, and `--termination-grace-seconds`.
Log byte and physical-line bounds prevent an output flood from being treated as auditable evidence.

The tool atomically checkpoints `lifecycle-receipt.json` from initialization onward. A successful
receipt includes the runner and commercial-verifier digests; exact source, transport, descriptor,
module-receipt, and binary identities; every dependency-inventory row; authored-script hashes; the
final enabled project and complete installed-plugin tree; and the exactly named Mac app, Info.plist, every Mach-O
fat slice, exact requested-map row in UAT's UFS staging manifest, byte-identical staged/archive
containers, cooked registry, and Unreal target receipt. The receipt then
self-validates all project/plugin/app/script/inventory/log bindings from disk, reruns every log
audit and package check, and replays the exact command argv, working directory, timeout, and
descendant policy. The reinstall is byte-identical before compilation. After `BuildCookRun`, the
receipt independently rebinds the complete installed tree, requires every non-build input to remain
byte-identical, permits differences only below Unreal-owned `Binaries/` and `Intermediate/` roots,
and records a deterministic count and SHA-256 over every rebuilt build-product delta. Mach-O
acceptance parses the real arm64 slice and every load-command boundary;
a magic number alone is insufficient. After work-root and receipt initialization, a failure receipt
identifies the phase and retained work paths; the tool never cleans lifecycle work products on failure. A command
exit code alone is not lifecycle evidence; all log, state, and hash assertions must also pass.

The implementation and its focused Python contract tests pass. The latest uninterrupted exact-artifact
Mac lifecycle evidence is r11 run
`20260721T100000.286819Z-d78c4b61-b2be-4222-b28d-300401afadbe` at clean commit
`d87813430b4e74515aa98b4353cdc9d0e7e2de91`; its schema-3 lifecycle receipt SHA-256 is
`7a6b1ed35db45128116caf14038963b0caae09bb74bb14a11bd229b240e23449`. A later focused
presentation qualification does not supersede that full lifecycle evidence. Native-host lifecycle,
Launcher/no-rebuild loading, Marketplace/Vault transport, manual dialogs, and Win64 remain open.

Preferred non-publishing pre-release qualification (no release tag required):

```text
python3 Tools/maintenance/release_rts_plugin.py \
  --engine-root=/path/to/UE_5.8 \
  --install-lifecycle
```

Low-level qualifier invocation requires the producer receipt created by that runner:

```text
python3 Tools/maintenance/qualify_rts_install_lifecycle.py \
  --engine-root=/path/to/UE_5.8 \
  --artifact=/path/to/RealTimeStrategy-BuildPlugin \
  --producer-receipt=/path/to/RealTimeStrategy-BuildPlugin.build-producer.json \
  --work-root=/new/path/RTSInstallLifecycle
```

## Source-independent downloaded archive intake

Downloaded bytes enter the lifecycle only after the separate schema-3 Mac intake described in
[Packaging](PACKAGING.md#downloaded-release-archive-intake). Publication emits the canonical ZIP,
checksum, detached schema-10 release receipt, schema-3 intake and downstream tools, then publishes the schema-1
release-set manifest last. Operators obtain only the expected manifest SHA-256 out of band,
authenticate the manifest, and authenticate each executable tool against its exact manifest record
before execution. Both tools are standard-library-only and have no publisher or source-tree dependency.
On success, `<work-root>/RealTimeStrategy` is the only plugin directory handed to a fresh project,
and `<work-root>/intake-receipt.json` binds the verified inputs, intrinsic package checks, extracted
tree, verifier, and timestamps. On failure it removes the entire fresh intake work root.

That source-independent promotion is the boundary between downloaded bytes and install/lifecycle
qualification. The lifecycle gate documented earlier is intentionally an internal pre-publication
gate: it requires source Git and an unpublished BuildPlugin producer receipt, so it cannot prove a
customer download. The standalone downstream implementation is:

```text
python3 /downloaded/RealTimeStrategy-<version>-UE5.8-Mac-<commit>.downloaded-qualification-schema3.py \
  --engine-root /path/to/UE_5.8 \
  --release-set-manifest /downloaded/RealTimeStrategy-<version>-UE5.8-Mac-<commit>.release-set-schema1.json \
  --expected-release-set-manifest-sha256 <independently-trusted-lowercase-64-hex> \
  --intake-work-root /fresh/path/RTSReleaseIntake \
  --work-root /fresh/path/RTSDownloadedQualification
```

It installs only the promoted tree into a fresh content-only project, runs the exact 78-test RTS
inventory, packages the unchanged plugin starter map in Development, and requires a naturally
exiting headless packaged workload with 100 ready real units, the 20/80 composition, at least two
owners, zero spawn failures, and zero alarms. It verifies the staged plugin map and content
containers, target receipt, cooked registry, and packaged Mach-O; permits post-build changes only
under plugin `Binaries/` and `Intermediate/`; and emits a self-validating schema-3 receipt. Before
and after that build delta it independently applies `Config/CommercialArchive.v1.json` to the whole
package, enforces Fab's decimal `15,000,000,000`-byte pre-ZIP measurement excluding top-level
`Saved` and `Intermediate`, and verifies `Config/ThirdPartyBom.v1.json` plus its nested acquisition
inventories. The BOM preserves the exact UIKit boundary of 117 shipped assets and three
repository-only authoring recipes. Its CC-BY credit/acquisition-chain review remains human, and its
`crowd_pathfinder` `NOASSERTION` component is consulted-only with zero shipped mappings; no automated
receipt claims legal approval. The
runtime workload is functional install evidence, not a rendered-performance claim or a new battle
completion claim.

The implementation and adversarial tests pass, but no live receipt is recorded because no final
authenticated release set exists yet. Intake success alone is not an install, launch, gameplay, or
Fab transport pass, and implementation alone does not close that evidence gap.

## Customer disable and uninstall policy

Before disabling:

1. Commit or back up the project, including Config and Content.
2. Inventory `/Game` assets that depend on `/Script/RealTimeStrategy` or
   `/RealTimeStrategy/...`. Generated Blueprints, maps, Content Sets, and customer subclasses are
   expected to appear.
3. Close the editor. Disable the plugin in the project descriptor and reopen only after the
   descriptor is valid JSON.

Disabled dependencies may show missing-class or unloadable-package diagnostics. Do not resave,
redirect, consolidate, or delete those packages while their defining plugin is absent. To recover,
restore the same or a migration-compatible plugin version, explicitly re-enable it, restart the
editor, preview Content Set migration, and validate before saving or regenerating.

For uninstall, keep the plugin disabled, close the editor, retain the dependency inventory, then
remove the plugin directory. Uninstalling removes plugin code and plugin-owned content only. It does
not authorize deletion or conversion of project-owned assets. Reinstall the exact artifact first
when project content must be recovered or migrated.

## What local automation does not prove

The local copied-artifact gate and the source-independent intake tooling are not evidence of an
actual Epic Marketplace/Fab download. No independently acquired public artifact, independently
observed trust-channel values, or clean-machine Unreal run is implied by their implementation or
contract tests. Epic Games Launcher enable/install UX, entitlement, CDN/Vault transport, no-rebuild
Launcher-engine binary loading, manual Content Browser dialogs, other operating systems, and other
architectures require their own recorded qualification. Never label those surfaces passed based on
these workflows.
