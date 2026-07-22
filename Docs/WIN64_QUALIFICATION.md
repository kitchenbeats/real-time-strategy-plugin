# Win64 Commercial Qualification

Win64 is part of the supported v1 product contract, but it is not qualified by a Mac build or by
the host-independent checks in this document. A release claim requires one uninterrupted gate on a
real 64-bit Windows qualification machine using UE 5.8, followed by review of the immutable
artifact, receipts, process logs, and runtime evidence produced on that machine.

## Mac-host readiness checks

The repository includes a standalone fail-closed verifier at
`Tools/maintenance/win64_release_contract.py`. It does not execute Windows code and always labels
its output as static readiness only.

Audit the current plugin tree for paths that cannot be installed safely on Windows:

```bash
python3 StarMapsRTS/Tools/maintenance/win64_release_contract.py \
  StarMapsRTS/Plugins/RealTimeStrategy --paths-only
```

Audit a real Win64 `BuildPlugin` output copied from the Windows qualification machine:

```bash
python3 StarMapsRTS/Tools/maintenance/win64_release_contract.py \
  /path/to/RealTimeStrategy-1.3.0-UE5.8-Win64 \
  --json-output /path/to/win64-static-readiness.json
```

The artifact check requires:

- an installed UE 5.8 descriptor with the exact three-module inventory;
- exact `UnrealEditor-<Module>.dll` membership and matching `UnrealEditor.modules` bindings;
- bounded native AMD64 PE32+ DLL structure, including complete COFF/optional headers, sane image
  alignment and sizes, nonoverlapping in-file sections, and at least one executable code section;
  header-shaped byte fixtures, ARM64, x86, malformed, truncated, or renamed non-PE files fail;
- one strict-JSON precompiled receipt for each runtime module in Development and Shipping, with
  every declared output resolving inside the matching artifact module/configuration build tree.
  Each `.obj` must be a bounded AMD64 COFF/bigobj object with real section data; each `.lib` must
  be a bounded COFF archive containing at least one such object. One-byte placeholders, archive
  signatures without objects, and merely nonempty files fail;
- no symlinks, Windows reparse points, special files, case-insensitive path collisions, reserved
  device names, forbidden Windows filename characters, trailing spaces/dots, or overlong relative
  paths;
- bounded JSON depth/node counts and strictly typed, duplicate-free platform allow/deny lists; and
- a digest-bound JSON summary that explicitly records `runtimeEvidence: false`. `--json-output`
  must be outside the artifact and is published by fsync plus atomic replacement.

Passing these checks means only that the artifact is structurally plausible for Win64. It is not
compile, cook, package, launch, gameplay, rendering, network, performance, memory, or soak evidence.

## Windows-host release lane

Run from a clean checkout with the Windows UE 5.8 source/installed build selected by
`--engine-root`. Do not reuse a prior package or evidence directory. Win64 currently has a
non-publishing qualification lane only; immutable archive promotion remains blocked until the
Windows install lifecycle is implemented and executed.

Before the full gate, run the host-independent tests and the runner's provenance suite:

```powershell
py -3 -m unittest `
  StarMapsRTS/Tools/maintenance/tests/test_win64_release_contract.py -v
py -3 -m unittest `
  StarMapsRTS/Tools/maintenance/tests/test_release_rts_plugin.py -v
```

The current non-publishing full qualification command is:

```powershell
py -3 StarMapsRTS/Tools/maintenance/release_rts_plugin.py `
  --project StarMapsRTS/StarMapsRTS.uproject `
  --plugin StarMapsRTS/Plugins/RealTimeStrategy `
  --engine-root C:/Epic/UE_5.8 `
  --output StarMapsRTS/Saved/PluginPackages/Win64Qualification
```

Do not lower the checked-in release minimum with an override. The current expected source inventory
is 64 tests and the Windows run must reproduce that exact inventory without warnings.

The runner rejects `--archive` and `--install-lifecycle` on Win64 because the Windows install
lifecycle and promotion path are not implemented. A passing non-publishing run remains engineering
qualification evidence, not a distributable customer release. The implementation has not yet been
executed on Windows, so its presence is not qualification. Do not use developer output or a Mac
static-verifier result as customer-release evidence.

## Required qualification evidence

The Windows run must prove the same customer contract as Mac, on the exact Win64 artifact:

1. Strict UE 5.8 Editor source compile with PCH, shared PCH, and unity disabled.
2. Full source automation with the exact expected inventory and zero warnings.
3. Win64 `BuildPlugin` for Editor plus Development and Shipping runtime targets.
4. Static artifact verification, including AMD64 PE identity and Windows-safe paths.
5. Exact installed-package automation in an isolated blank host.
6. Starter cook and migrated historical Content Set cook/package/play.
7. Fresh Blueprint-only author, reopen, safe-fix, regenerate, Shipping package/launch, and
   Development semantic two-match workflow.
8. Fresh native consumer Editor/Game compilation, all advertised headers/recipes, Shipping
   package, two authoritative matches, both winner paths, and one clean reset.
9. Packaged listen server, two clients, ownership/observer assertions, and clean process teardown.
10. Standalone and synchronized authority/client 100-unit functional captures.
11. Rendered starter and Manny presentation review on the Windows default Shipping RHI.
12. Win64-specific performance, bandwidth, retained-memory, and soak qualification under the
    published hardware/settings contract.
13. Complete durable command/process logs, host/tool/engine identity, package tree digest, schema-1
    runner-evidence-root manifest and checksum, release receipt, archive digest, clean commit, and exact
    version tag. The runner-evidence manifest covers only the finalized internal run root; package and
    archive trees remain separately verified external artifacts.

## Implemented Win64 runner contracts awaiting Windows proof

The runner now implements the Mac-developable portions without claiming that they have passed on
Windows:

- Every Windows child, including ordinary compiler/tool probes and long-lived runtime children, is
  created suspended, assigned before user code runs to a
  kill-on-close Job Object that does not permit breakaway, and only then resumed. A preflight probe
  records the descendant PID, observes that PID as active Job membership after the parent exits,
  observes output written through the inherited handle, and proves the entire tree can be closed.
  Missing APIs, failed assignment, an unexpected initial-thread set, failed resume, or failed tree
  termination/handle cleanup aborts the gate. Ordinary commands also have a six-hour hard bound.
- Host identity accepts only a strict JSON whitelist for Windows version/build, 64-bit
  architecture, CPU/core/RAM, GPU/driver, and PowerShell version. Release identity additionally
  binds hashes and versions for 64-bit Python, Git, PowerShell, one default Visual Studio 2022
  installation, x64 MSVC, Windows SDK `rc`/`signtool`, UE Build/Editor/UAT, and UE-bundled
  Subversion. Missing, ambiguous, malformed, or non-default toolchains fail closed.
- Every source compile, `BuildPlugin`, installed-consumer compile, and UAT build command must emit
  UBT's selected MSVC toolchain directory/version and Windows SDK directory/version in its own
  command logs. The runner binds each build command separately, rejects missing or inconsistent
  stage observations, hashes the selected `cl.exe` and `rc.exe`, and requires every observation to
  equal the preflight identities; merely recording the newest installed toolchain or the first
  compile is insufficient.
- Commercial identity collection and the runtime Job Object probe are implemented for Win64, but
  immutable archive promotion remains blocked until the Windows install lifecycle and publication
  path are implemented and pass on the same host.
- The Blueprint-only gate packages and hashes a real Shipping launch-smoke artifact and a separate
  Development semantic artifact on every supported host. Its durable receipt binds both exact
  package trees, executables, cooked-container counts, generated inventory, runtime verdicts, and
  installed plugin tree.
- Full installed-native and focused native-Shipping gates support Win64. Their receipt binds the
  Shipping package tree, executable, cooked containers, installed plugin tree, and two-match/reset
  runtime verdict.
- Main package verification directly invokes the standalone Win64 contract for Windows-safe paths,
  reparse/symlink exclusion, exact AMD64 editor DLL inventory, `.modules` binding, and Development
  plus Shipping precompiled receipts.
- Publication staging holds an exclusive lock, captures a stable private snapshot with descriptor-level
  file identity checks, reruns package-local semantic/binary gates on that exact snapshot, and
  archives only those snapshot bytes. Temporary archive/checksum files and promotion directory
  entries are fsynced. Promotion is restartable: a verified archive missing its checksum gets the
  checksum restored, an orphan checksum is removed before a deterministic rebuild, and a complete
  matching archive/checksum pair is an idempotent staging result. The runner does not report the archive
  as published or return success until the checksum-last detached final-evidence marker is written and
  reopened successfully. A dead publisher lock is reclaimed;
  live or malformed ownership remains fail-closed. Descriptorless payload directories never enter
  the semantic publication path. Artifact roots and their ancestors may not be junctions/reparse
  points.
- Packaged Windows launch validation requires the archive bootstrap and exactly one
  configuration-specific nested x64 executable. It rejects missing, ambiguous, malformed, DLL, or
  wrong-architecture payloads instead of searching arbitrary executables.

## Evidence that fundamentally requires Windows

The following cannot be closed honestly on macOS:

- execute the Job Object assignment/resume/descendant-termination path under the actual Windows
  account, antivirus, CI parent-job, and UE process behavior;
- validate the strict identity probes against the production Windows machine and UE 5.8 toolchain;
- confirm UE 5.8's real Development and Shipping archive names and `ProjectSavedDir` location match
  the fail-closed layouts above;
- compile, cook, package, and launch the exact plugin plus fresh Blueprint and native consumers;
- prove multiplayer teardown, source-control checkout, rendering/RHI, Manny presentation,
  performance, bandwidth, memory, and soak behavior; and
- review the resulting durable evidence and immutable archive before making a Win64 release claim.

Any mismatch discovered there must change the explicit contract and receive new tests. It must not
be handled by searching arbitrary paths, accepting multiple candidates, or relabeling Mac evidence.
