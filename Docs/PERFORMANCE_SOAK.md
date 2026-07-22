# Performance and Soak Release Contract

This document defines the evidence required before the Real-Time Strategy plugin may make scale,
frame-rate, memory, bandwidth, or long-session reliability claims. The numbers below are initial
release targets, not measured results. A target becomes a supported claim only after the installed
plugin passes the corresponding packaged test on declared hardware.

Internal and third-party benchmark comparisons may be used to find missing scenarios, but they are
not evidence for this implementation and must never be reported as plugin results. Only the
installed-plugin evidence defined below can authorize a customer-facing performance claim.

## Evidence rules

Every result must record the plugin commit and version, Unreal version, clean-project template,
map and scenario revision, executable configuration, platform, exact CPU/GPU/RAM, resolution and
scalability settings, RHI or `NullRHI`, process role, connection count, unit composition, sample
duration, warm-up duration, command line, and raw capture locations. Aggregated numbers without the
raw Unreal Insights/CSV, log, verdict JSON, and memory/network captures do not pass a release gate.

The benchmark must use an installed `BuildPlugin` artifact in a new project. Source-project runs
are diagnostic only. A rendered Manny client and a headless authoritative server are separate
products with separate budgets; a `NullRHI` result cannot support a client frame-rate claim.

The run fails on any crash, fatal error, assertion, ensure, unexpected error or warning, watchdog
alarm, invalid numeric value, connection failure, test timeout, incomplete match, or missing/truncated
capture. It also fails when a required percentile or peak exceeds its budget. Averages alone never
determine a pass.

## Initial qualification hardware

The first release qualification floor is deliberately fixed so results are comparable. It is not a
published customer minimum until the complete matrix passes:

- Win64: AMD Ryzen 7 5800X, NVIDIA GeForce RTX 3070 8 GB, 32 GB system RAM, and SSD storage.
- Mac: Apple M2 Pro with 12-core CPU and 19-core GPU, 32 GB unified memory, and internal SSD.

Rendered-client budgets must pass on the listed platform hardware or a demonstrably slower member
of the same architecture; a faster machine cannot establish the floor. Dedicated-server budgets use
the listed CPU with simultaneous multithreading enabled and no other workload. OS, driver, power
mode, thermal state, and clock behavior remain required metadata. A cloud instance may provide
supplemental evidence but cannot replace the fixed-CPU gate unless its physical CPU allocation and
performance variance are controlled and documented.

## Workload matrix

Unit counts are concurrent living RTS pawns, not cumulative spawn counts. Each tier uses at least
two teams, representative workers and combat units, active orders, combat, deaths, and replacement
spawns. Results must report the peak actor, pawn, projectile, widget, and replicated-actor counts.

| Tier | Required workload | Intended evidence |
| --- | --- | --- |
| Baseline | 100 units, two teams, complete economy-to-victory loop | Normal-match correctness and 60 Hz client baseline |
| Large battle | 500 units, four teams, simultaneous economy, movement, vision, production, and combat | Advertised release scale target |
| Stress | 1,000 units, eight teams, simultaneous army movement and combat | Graceful 30 Hz ceiling; not the default recommended match size |

Each tier must run all applicable movement cases: open ground, narrow/maze geometry, mixed-size
formation, dynamic building insertion/removal, repeated identical destinations, distinct nearby
destinations, unreachable destinations, and commands issued while a previous move is active. Stock
navigation is the shipping baseline. `Military Only` and `All Pawns` are reported separately and
remain experimental until every correctness and budget gate passes.

Network runs use a packaged dedicated server with 1, 2, 4, and 8 remote connections. They include
normal play, late join as observer, clean disconnect, reconnect policy, match restart, map travel,
100 ms round-trip latency with 10 ms jitter, and 1% packet loss. The unconditioned run establishes
the bandwidth budget; emulated adverse conditions establish correctness and recovery.

## Packaged scale harness

MatchCheck can establish and sustain the unit-count portion of a tier without test doubles. It
walks the installed content set's live construction and production catalogs, selects a real worker
and mobile combat unit, and uses `ARTSGameMode::SpawnActorForPlayer`. The tier counts the supported
unit boundary: owned, living, health-bearing `ACharacter` instances without a construction-site
component. Pawn-based buildings, camera pawns, and spectators therefore cannot inflate it. Engine ownership, replicated
RTS ownership, AI discovery, movement, vision, combat, death, and skirmish cleanup therefore use
shipping paths. Spawn work is bounded per frame and the first capture warm-up begins only after the
initial population is ready. Later deaths are replenished while capture remains continuous across
in-process skirmish resets. For multiplayer, authority publishes an immutable replicated run ID,
authoritative ready count, and future server-world capture epoch. Remote clients synchronize to
that contract without attempting authoritative spawning and without weakening fog-of-war
relevancy to count hidden actors.

| Tier | Skirmish URL option | MatchCheck flags |
| --- | --- | --- |
| 100 | `?NumBases=2` | `-RTSPerformanceCapture -RTSPerformanceTier=100 -RTSPerformanceBudget=Client100 -RTSPerformanceSampleSeconds=600` |
| 500 client | `?NumBases=4` | `-RTSPerformanceCapture -RTSPerformanceTier=500 -RTSPerformanceBudget=Client500 -RTSPerformanceSampleSeconds=600` |
| 500 server | `?NumBases=4` | `-RTSPerformanceCapture -RTSPerformanceTier=500 -RTSPerformanceBudget=Server500 -RTSPerformanceSampleSeconds=600` |
| 1,000 client | `?NumBases=8` | `-RTSPerformanceCapture -RTSPerformanceTier=1000 -RTSPerformanceBudget=Client1000 -RTSPerformanceSampleSeconds=600` |
| 1,000 server | `?NumBases=8` | `-RTSPerformanceCapture -RTSPerformanceTier=1000 -RTSPerformanceBudget=Server1000 -RTSPerformanceSampleSeconds=600` |

Budget runs need at least 660 wall seconds at 1x simulation speed: 60 seconds of warm-up plus ten
sampled minutes. The explicit sample flag is mandatory and the replicated coordinator owns the
shared end epoch, so neither authority nor clients may declare a result early. Use
`-RTSMatchCheck=900` as the standard watchdog allowance. A
tier-only run may omit `-RTSPerformanceBudget` for functional validation, but it is not budget
evidence. Qualification uses AI-only participant owners so all workload units receive autonomous
shipping orders; rendered clients join as observers. Once per wall second, authority and remote
client each retain a timestamped real population observation from the start through the terminal
measurement boundary; missed observations are never backfilled. The verdict records the bounded
series and its reconciled count/minimum/maximum/final/gap aggregates, local sampled peaks,
authoritative readiness, selected class paths, participant
count, initial composition, spawns/failures, run ID/role, budget profile, and every violation.

The in-process client budget evaluator proves only its declared frame and hitch thresholds. The
qualification runner must separately reject any run that did not use 1920x1080, High scalability,
native resolution scale, VSync off, fog and the bundled presentation, the declared hardware, and
the required raw capture set. Server and client verdicts count as one network run only when their
run IDs and immutable coordinator contract match.

## Initial performance budgets

These are release acceptance targets to validate and, if necessary, revise transparently after the
first complete capture. Revising a target requires a documented product decision; silently widening
a threshold to make a run pass is prohibited.

### Rendered client

The target client runs at 1920x1080, High scalability, native resolution scale, VSync off, with the
bundled Manny presentation and fog enabled. Frame time is measured after a 60-second warm-up.

| Tier | Game-frame P50 | Game-frame P95 | Game-frame P99 | Hitches |
| --- | ---: | ---: | ---: | ---: |
| 100 units | <= 16.67 ms | <= 20 ms | <= 25 ms | no frame > 100 ms; <= 1 frame > 50 ms per 10 minutes |
| 500 units | <= 16.67 ms | <= 25 ms | <= 33.33 ms | no frame > 150 ms; <= 3 frames > 50 ms per 10 minutes |
| 1,000 units | <= 33.33 ms | <= 40 ms | <= 50 ms | no frame > 200 ms; <= 6 frames > 50 ms per 10 minutes |

GPU, game-thread, render-thread, and RHI-thread times are reported independently. A test does not
pass by hiding a game-thread regression behind a slower GPU. Garbage collection pauses are included
in frame and hitch percentiles.

### Authoritative dedicated server

The server runs without rendering and must sustain a 30 Hz simulation envelope. Server frame P95
must be <= 33.33 ms and P99 <= 40 ms at 500 units with four connections. At 1,000 units and eight
connections, P95 must be <= 50 ms and P99 <= 66.67 ms. No authoritative frame may exceed 250 ms,
and no more than one frame may exceed 100 ms per 10 minutes.

CPU captures must separately expose AI heartbeat phases, order/controller work, navigation and flow
generation, avoidance, vision state, replication gathering, combat/projectiles, spawning, garbage
collection, and match diagnostics. A single undifferentiated plugin scope is insufficient.

### Memory and networking

- Record resident and Unreal-tracked memory after warm-up, at peak battle, after match teardown,
  after restart, and at run end. After five identical create/destroy cycles and an explicit garbage
  collection, retained resident growth must be <= 5% or 128 MiB, whichever is smaller.
- No workload may show monotonic retained growth across the final three cycles. GPU allocations and
  fog/minimap render resources are reported separately on rendered clients.
- Report per-connection payload bytes, packet bytes, packets, replicated actor candidates, admitted
  actors, dormancy/relevancy rejects, and visibility-policy rejects at P50/P95/P99 and peak. The
  initial 500-unit target is <= 256 Kibit/s average and <= 1 Mibit/s one-second peak per playing
  client, excluding connection handshake and map download. Observer results are reported separately.
- Packet loss, reliable-buffer overflow, saturation, queued bunch growth, or a security/relevancy
  violation is an unconditional failure even when the bandwidth number passes.

## Subsystem counters required before benchmarking

The release harness must capture low-overhead named trace/CSV scopes plus machine-readable counters:

- frame percentiles, >50/>100/>200 ms hitch counts, simulation time, and wall time;
- AI heartbeat and manager phase durations, processed/skipped agents, and deferred work;
- navigation request/result counts, flow cache hits/misses/evictions, generated/rejected fields,
  cells processed, generation duration, cache bytes, duplicate destinations, and retry/failure counts;
- vision casters/targets/team checks, dirty tiles, fog pixels processed, texture uploads, upload bytes,
  and vision/fog CPU duration;
- replication connections, candidates, admitted actors, vision rejects, bytes, packets, and backlog;
- live/peak actors, pawns, RTS units, projectiles, components, widgets, and queued production/orders;
- process/Unreal/GPU memory snapshots, garbage collections, collection duration, and retained delta;
- recovered no-progress orders by cause and age, plus all existing match-check alarms.

Counters must be available in Development captures and compiled safely out or remain negligible when
not armed. The capture path may observe behavior but must not substitute fake actors, mock navigation,
or synthetic success for the real packaged systems.

### Stable work-counter dictionary

`performance.workCounters` and the `RTS` CSV category use the following frozen names. JSON uses the
lower-camel spelling in the second column; CSV uses the exact PascalCase spelling in the first. Every
value is an unsigned cumulative count for the armed measurement window, saturates at `uint64` maximum,
and remains zero-cost apart from the capture-gate check when another world owns the capture or capture
is disabled.

| CSV custom stat | Verdict field | Exact accumulated meaning |
|---|---|---|
| `ControllerTicks` | `controllerTicks` | Player-controller tick executions. |
| `OrderRequests` | `orderRequests` | Individual order requests examined by the authoritative order path. |
| `OrdersAccepted` | `ordersAccepted` | Examined order requests admitted for execution. |
| `OrdersRejected` | `ordersRejected` | Examined order requests rejected before execution. |
| `OrderBatches` | `orderBatches` | Order batches processed; one batch may contain multiple requests. |
| `PawnAITicks` | `pawnAITicks` | Pawn-local AI tick executions. |
| `NavPathRequests` | `navPathRequests` | Stock-navigation path requests submitted. |
| `NavPathSuccesses` | `navPathSuccesses` | Submitted stock-navigation requests producing a usable path result. |
| `NavPathFailures` | `navPathFailures` | Submitted stock-navigation requests producing a failed/unusable result. |
| `AvoidanceTicks` | `avoidanceTicks` | Authoritative grounded movement observations satisfying UE's live RVO evaluation predicates. Exact engine-owned CPU duration remains `STAT_AI_ObstacleAvoidance`; this count is not a substitute timing scope. |
| `FlowMovementTicks` | `flowMovementTicks` | Flow-field movement-component tick executions. |
| `AttackTicks` | `attackTicks` | Attack-component tick executions. |
| `AttackAttempts` | `attackAttempts` | Attacks that reached an attempt decision after target/range/cooldown evaluation. |
| `AttackCommits` | `attackCommits` | Attempts committed to their configured damage/projectile execution path. |
| `ProjectilesSpawned` | `projectilesSpawned` | Spawned gameplay projectile actors that successfully completed authoritative `FireAt` arming. |
| `ProjectileTicks` | `projectileTicks` | Gameplay projectile tick executions. |
| `ProjectileImpacts` | `projectileImpacts` | Projectile impact resolutions, independent of whether damage was nonzero. |
| `ProductionTicks` | `productionTicks` | Production-queue component tick executions. |
| `ProductionQueuedItems` | `productionQueuedItems` | Queue-depth observations accumulated at production ticks; divide by `ProductionTicks` for the sampled mean. |
| `ProductionCompletions` | `productionCompletions` | Completed queue entries that successfully spawned and committed their output actor. |
| `ProductionSpawnFailures` | `productionSpawnFailures` | Permanent completed-entry failures that prevented a configured output actor from being committed; transient supply, admission, or congestion deferrals are excluded. |
| `MinimapTicks` | `minimapTicks` | Minimap widget tick executions. |
| `MinimapPaints` | `minimapPaints` | Minimap native-paint passes. |
| `MinimapUnitIconsDrawn` | `minimapUnitIconsDrawn` | Built-in unit-icon box draws emitted by `DrawUnits`; custom `NotifyOnDrawUnit` drawing is intentionally excluded because the widget cannot observe its primitives. |
| `MinimapVisionDrawCalls` | `minimapVisionDrawCalls` | Fog-of-war vision brush draws; the current renderer emits one brush draw, not one event per vision cell. |

The pre-existing stable counters remain `ReplicationCandidates`, `ReplicationAdmitted`,
`ReplicationStreamingLevelRejects`, `ReplicationVisibilityRejects`, `FogTextureUploads`,
`FogPixelsProcessed`, and `FogTextureUploadBytes`; verdicts additionally expose
`replicationGatherCalls`. A `Record*` publication is accepted only when its non-null capture scope is
pointer-identical to the active scope. Reset, coherent snapshot, and release transitions invalidate
the publication generation and drain admitted writers before touching the counter window.

### Stable CPU timing-scope dictionary

Every plugin CPU scope is paired across Unreal Insights (`RTS.<domain>.<scope>`) and the `RTS` CSV
category. These exact names form the reviewed v1 dictionary; additions or renames require the static
inventory test and this table to change together.

| Domain | Unreal Insights scopes | CSV timing stats |
|---|---|---|
| AI | `RTS.AI.Tick`, `RTS.AI.Workers`, `RTS.AI.Production`, `RTS.AI.Research`, `RTS.AI.Abilities`, `RTS.AI.Construction`, `RTS.AI.SquadsAndScouting`, `RTS.AI.PawnControllerTick` | `AITick`, `AIWorkers`, `AIProduction`, `AIResearch`, `AIAbilities`, `AIConstruction`, `AISquadsAndScouting`, `PawnAITick` |
| Orders/player | `RTS.Player.ControllerTick`, `RTS.Orders.ExecuteRequest`, `RTS.Orders.BatchDispatch` | `PlayerControllerTick`, `OrderRequestExecution`, `OrderBatchDispatch` |
| Navigation | `RTS.Navigation.StockPathRequest`, `RTS.Navigation.FlowFieldRequest`, `RTS.Navigation.FlowPathBuild`, `RTS.Navigation.FlowFieldGenerate`, `RTS.Navigation.FlowMovementTick` | `StockPathRequest`, `FlowFieldRequest`, `FlowPathBuild`, `FlowFieldGenerate`, `FlowMovementTick` |
| Vision/replication | `RTS.Vision.ManagerTick`, `RTS.Vision.FogTextureTick`, `RTS.Replication.VisionGather` | `VisionManagerTick`, `FogTextureTick`, `ReplicationVisionGather` |
| Combat | `RTS.Combat.AttackTick`, `RTS.Combat.AttackTransaction`, `RTS.Combat.ProjectileArm`, `RTS.Combat.ProjectileTick`, `RTS.Combat.ProjectileImpact` | `AttackTick`, `AttackTransaction`, `ProjectileArm`, `ProjectileTick`, `ProjectileImpact` |
| Production | `RTS.Production.Tick`, `RTS.Production.Finish` | `ProductionTick`, `ProductionFinish` |
| UI/minimap | `RTS.UI.MinimapTick`, `RTS.UI.MinimapPaint`, `RTS.UI.MinimapDrawUnits`, `RTS.UI.MinimapDrawVision` | `MinimapTick`, `MinimapPaint`, `MinimapDrawUnits`, `MinimapDrawVision` |
| Diagnostics | `RTS.Diagnostics.MatchCheckTick`, `RTS.Diagnostics.MatchCheckPoll` | `MatchCheckTick`, `MatchCheckPoll` |

UE's stock RVO calculation remains engine-owned and is timed by `STAT_AI_ObstacleAvoidance`; the
plugin's `AvoidanceTicks` counter reports eligible evaluations without wrapping a false proxy CPU
scope around the controller-side predicate check. Engine GC scopes likewise remain authoritative;
the verdict separately records explicit-GC lifecycle and duration evidence.

## Soak gates

The diagnostic gate is a 30-minute packaged Development run at the 500-unit tier with four remote
clients. It repeats complete matches, teardown, restart, late join, disconnect, and travel while
retaining continuous performance, memory, and network capture.

The release-candidate gate is a two-hour packaged Shipping run at the same tier and connection count,
followed by a 15-minute 1,000-unit stress segment. It must complete at least five full match cycles,
produce valid final captures, remain inside all budgets, and shut down cleanly. Platform release
evidence requires this gate independently on every advertised platform.

## Performance qualification receipt gate

Rendered-client and authoritative dedicated-server results are admitted only through the release
runner's explicit receipt verifier:

```text
release_rts_plugin.py --verify-only <installed BuildPlugin directory> \
  --performance-qualification-receipt <qualification.json> --engine-root <UE 5.8 root>
```

This is a verifier, not a benchmark generator. Unreleased schema 1 has been replaced by schema 2.
Schema 2 binds the receipt to the independently verified plugin tree, current platform, complete
host/UE identity, run/scenario/role, required rendered or dedicated-server presentation contract,
real workload, full command/configuration/project-revision provenance, role-appropriate thread
timings, and raw evidence. The receipt directory must contain exactly one individually size- and
SHA-256-bound Insights trace, CSV Profiler capture, runtime log, verdict JSON, memory capture,
network capture, population capture, lifecycle capture, and hardware inventory. JSON parsing rejects
duplicate keys, non-finite values, excessive size/depth/node counts, missing categories, unsafe
paths, and digest substitutions.

The verifier recomputes every summary from timestamped raw series instead of trusting asserted pass
fields. It requires continuous exact population and network coverage at least once per wall second,
stable connection identities and roles, four playing clients throughout the 500-unit segment, and
eight throughout the 1,000-unit stress segment. It derives connection percentiles, average and peak
rates, queue growth, packet loss, overflow, saturation, and security failures from cumulative raw
counters. Five contiguous create/destroy/explicit-GC cycles must cross-bind exactly to lifecycle
events and post-GC resident, Unreal-tracked, GPU, and fog/minimap snapshots. Dedicated-server
receipts use Server500/Server1000 budgets, game-thread-only timing, a headless presentation
contract, and zero rendered-resource accounting. A 30-minute run is labeled only
`diagnostic-qualified`; `release-candidate-qualified` additionally requires a continuously captured
two-hour primary segment, at least five completed matches, and a separately budgeted continuous
15-minute 1,000-unit/eight-client stress segment. Shortened, discontinuous, aggregate-only, or
internally inconsistent evidence fails closed.

Passing one role's verifier does not waive paired client/server evidence, the workload scenario
matrix, or per-platform repetition elsewhere in this contract.

## Current audit status

No workload in this document has passed qualification yet. Historical exact-artifact runs proved a
small packaged listen-server topology, complete two-match Manny behavior, and the 100-unit functional
baseline. The current source harness adds contract-tested wall-frame percentiles/hitches, one-second
timestamped exact population, bounded privacy-safe per-connection series, process-memory
start/peak/end values, and real post-GC snapshots. The warm-up boundary resets measured census,
memory, GC, sampled network peaks, and the scoped replication/fog work-counter window so those fields
cover the same interval as frame samples. Stable connection IDs and authority/remote roles retain
wrap-aware one-second byte/packet/loss deltas, computed and engine rates, and queued bits. Engine
driver and controller lifetime totals remain explicitly named `latest*` diagnostic context.
Unreal-tracked memory is recorded when Low Level Memory tracking is enabled.
`-RTSPerformanceGCCycles=<count>` schedules a bounded number of real explicit collections across the
measured interval and records post-GC resident/Unreal values plus lifecycle provenance; unavailable
GPU memory is explicitly marked unavailable rather than invented. One-second census samples also
retain peak component, live-widget, active pawn-order, and queued-production counts.

The process-wide work recorder permits exactly one MatchCheck owner. Overlapping performance worlds
fail the second harness instead of resetting or mixing captures; capture transitions invalidate a
generation and drain already-admitted writers before reset, snapshot, finalization, or ownership
transfer. These measurements are not qualification by themselves: the runtime series still lacks
payload-versus-packet separation, per-connection candidate/admission/rejection counters,
authoritative reliable-buffer overflow state, orchestrated identical create/destroy cycles, and
rendered GPU/fog/minimap allocation capture. It can generate and replenish real 100/500/1,000-unit
catalog workloads and automatically enforce the initial client/server frame and hitch budgets.

The release runner can reject or admit a complete externally produced rendered-client or
dedicated-server schema-2 qualification receipt without treating missing evidence as a pass, but the
repository contains no qualifying receipt. `performance_evidence_producer_v2.py` provides a
fail-closed Mac headless producer for exact-artifact Game/Editor/Server hosts, 2/4/8-team authority
workloads, 1/2/4/8 explicit observer peers, network emulation, late join/disconnect/client restart,
and invocation-bound raw artifacts. It truthfully marks observer-only output ineligible for the
playing-client bandwidth gate and emits a self-validated partial headless bundle when required raw
series are unavailable. The current UE 5.8 distribution advertises Mac Editor/Game targets but no
Mac Development Server target; a real package attempt confirmed UBT exit 6, and the producer now
fails that capability preflight before expensive packaging. A server-capable UE 5.8 distribution,
playing-client telemetry, authoritative travel acknowledgements, identical match lifecycle cycles,
full movement scenarios, rendered capture, and long-duration execution remain required.

Known measurement priorities from the source audit are the per-frame 256x256 fog-mask rebuild and
texture upload, visible-actor by team/member vision work, connection by visibility-protected-actor
replication gathering, synchronous flow-field generation, full-world AI scans, and continuously
ticking per-unit components. These are investigation targets, not confirmed bottlenecks, until the
required captures attribute their cost.
