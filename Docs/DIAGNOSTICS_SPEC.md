# RTS Diagnostics

A diagnostic match has three consumers; each needs its own output:
dev watching live (overlay), agent/tooling iterating (structured events + verdicts),
CI (exit codes). This document defines their shared runtime contract.

## Event stream and developer diagnostics

1. **JSONL event stream** — `Saved/Diagnostics/match_<UTC-timestamp>.jsonl`, one object per
   event: `{"t": <sim-seconds float>, "type": "<Category.Event>", "player": "<name|null>", "data": {...}}`.
   Public API on the diagnostics subsystem: `EmitEvent(FName Type, AController* Player, TSharedRef<FJsonObject> Data)`
   (no-op unless the stream is enabled: `-RTSDiagEvents` or `rts.diag.events 1`).
2. **Event instrumentation** across gameplay systems (each behind its existing channel):
   - `Economy.Deposit` {resource, amount}, `Economy.Income` (10s rollup) {resource, perMin, bank, workers, saturationPct}
   - `Combat.Engagement` {phase: start|end, location, attackers, defenders, armyValueDelta}
   - `Combat.Kill` {victimClass, killerClass, victimPlayer, atHomeBase: bool}
   - `AI.BuildOrder` {phase: queued|started|completed, class, reason}
   - `AI.Decision` {kind: attackLatch|retreat|expand|draftWorkers, detail (e.g. strengthRatio)}
   - `AI.SquadState` {squad, from, to}
   - `Move.Stall` {unit, order, secondsStill} · `Move.Voided` {unit, order}
3. **Dev HUD overlay** (`rts.diag.hud 1`): per-player scoreboard, squad states, income
   sparklines, live watchdog status.
4. Later: HTML match report (Tools/ script over the JSONL), baseline diff tool.

## Match acceptance

1. **MatchCheck harness**: `-RTSMatchCheck=<simSeconds>` (optional `-RTSMatchCheckSpeed=<x>` and
   `-RTSMatchCheckRepeats=<count>`; add `-RTSHumanCheck` to require the local human runtime proof
   and `-RTSTransactionCheck` to require the public customer-transaction proof)
   runs a headless/windowed match, evaluates the watchdogs below, writes
   `Saved/Diagnostics/matchcheck_<UTC-milliseconds>_<run-id>_<role>_<pid>.json`, logs a
   `[MatchCheck]` verdict block,
   and exits the process with code 0 (pass) / 1 (alarms). A match becomes a test.
   Packaged automation can pass `-RTSMatchCheckOutput=<directory>` to select an explicit writable
   verdict directory; relative paths resolve from the project directory and the destination must be
   writable under the platform's application sandbox. For a sandbox-signed packaged macOS app,
   omit the override unless the destination is explicitly container-authorized; the default
   `ProjectSavedDir/Diagnostics` resolves inside the app container. Failure to create or write the
   verdict raises `VerdictWrite` and fails the process.
   Add `-RTSPerformanceCapture` to record fixed-memory wall-frame P50/P95/P99, average/worst
   frame time, >50/>100/>200 ms hitches, five-second sampled actor/pawn/RTS-unit/projectile/component/
   widget/order/production peaks, resident and Unreal-tracked memory, GC timing, sampled network
   rates/backlog, strategic-request work, and scoped replication/fog work counters. Add
   `-RTSPerformanceTier=100`, `500`, or `1000` to
   discover the installed content set's real worker and combat catalogs, establish the tier through
   authoritative gameplay spawns, and replenish battle losses. The 500- and 1,000-unit tiers
   require at least four or eight participant bases. Add
   `-RTSPerformanceBudget=Client100|Client500|Client1000|Server500|Server1000` to enforce the
   corresponding frame/hitch thresholds after at least ten sampled wall minutes; budget runs must
   explicitly pass `-RTSPerformanceSampleSeconds=600` or longer. An automatic client budget pass is
   a frame-threshold verdict, not full release qualification: the external qualification runner must
   also prove resolution, scalability, VSync, fog/presentation, hardware, and raw-capture metadata. Client
   profiles reject dedicated-server and `NullRHI` runs; server profiles require a dedicated server.
   Qualification invocations may also pass an absolute, previously nonexistent
   `-RTSPerformanceCsvOutput=<file>.csv` together with a bounded
   `-RTSPerformanceScenario=<slug>`. MatchCheck then owns the UE CSV Profiler capture, embeds the
   run ID, capture role, unit tier, and scenario as capture-local metadata, waits for asynchronous
   CSV finalization, publishes through a no-replace destination writer, and publishes the verdict
   only after the requested file exists. The release
   runner pairs this with an invocation-bound `-tracefile` and retains both artifacts. These short
   `NullRHI` captures prove that profiling evidence can be produced and bound to the functional
   workload; they do not establish a frame-rate, network, retained-memory, or soak claim.
   In networked runs, authority alone creates and replenishes the workload. A transient replicated
   coordinator publishes an immutable run ID, authoritative tier count, warm-up/sample durations,
   and future server-world capture epoch; remote clients synchronize capture without spawning actors or requiring hidden
   enemy actors to replicate. The 60-second warm-up cannot be reduced for a budget verdict.
   The configurable 60-second wall-clock warm-up is unaffected by simulation speed. This is
   foundational telemetry, not performance evidence: the installed-package hardware and scenario
   matrix in `PERFORMANCE_SOAK.md` remains required.
   A synchronized sample run remains active through its declared wall-clock end even if the match
   result arrives earlier. Other runs finish on a committed result because post-match AI
   intentionally stands down; repeat mode calls `ResetSkirmish` and requires multiple complete matches in the
   same process. A timeout without a committed result is a failure. The verdict records each match.
   Human mode requires a non-observer local controller and proves ownership, camera, HUD, the
   bundled Enhanced Input mapping and real camera movement, local team vision, public selection,
   a gather order, and real resource pickup proven by source depletion or worker cargo increase once
   per required match.
   Transaction mode discovers the local player's owned catalogs rather than assuming starter class
   names. It proves occupied placement rejection; construction order, payment, completion,
   cancellation, and refund; production order, payment, cancellation, and refund; targeted rally,
   spawning, movement, real supply consumption, and the supply-block rejection/event contract. To
   isolate those API transactions from enemy interruption, the harness temporarily applies the
   local player's maximum supported speed boost and God Mode, then restores both on completion,
   failure, reset, deinitialization, or verdict. This is not balance or survivability evidence.
2. **Watchdog framework + v1 watchdogs** (poll-based now; will consume the event stream for
   richer rules once it lands):
   - `EconomyLiveness` — a player with workers but zero income for 60s (the dead-economy class)
   - `BankRunaway` — at verdict, a bank remains above threshold after a full uninterrupted window
     without active production/construction or an observed resource debit. Activity later in the
     match clears the candidate because it proves the AI can spend. Evaluation occurs at every
     committed match boundary before repeat state is reset, and the verdict retains the longest
     rich-idle candidate across the complete run, including recovered candidates.
   - `TeleportDetector` — any pawn displaced farther than MaxSpeed × window (the builder-teleport class)
   - `WorkerSlaughter` — a roster with draft-capable workers suffers repeated worker deaths at
     home with no drafted-defense response (the harass-loop class). Unarmed worker definitions
     cannot legally be drafted, so their deaths remain in telemetry but do not arm this watchdog.
   - `StuckStorm` — no-progress order voids past threshold/min

## Shared conventions
- All thresholds `UPROPERTY(Config)` — rebalance via DefaultGame.ini, never code.
- Event `type` names are `Category.Event` PascalCase; add new ones freely, never rename existing.
- Timestamps are SIM seconds (GetWorld()->GetTimeSeconds()), not wall clock.
- Capture and file output ship disabled; flags/CVars enable them. Guarded instrumentation must remain
  bounded and negligible when unarmed. Gameplay analytics CSV
  snapshots are opt-in through a positive configured interval or
  `-RTSAnalyticsInterval=<seconds>`; `-RTSAnalyticsOutput=<absolute-file>.csv` selects a unique
  invocation-bound destination but does not enable capture by itself.
- One JSON verdict schema, versioned: `{"version": 2, "result": "pass|fail", "simSeconds": N,
  "matchEnded": bool, "winner": string, "completedMatches": N, "requiredMatches": N,
  "humanReadiness": {"required", "completedChecks", "requiredChecks", "controller", "ownership",
  "camera", "hud", "enhancedInput", "vision", "selection", "gatherOrder", "resourceChanged",
  "cameraMoved", "complete"}, "transactionReadiness": {"required", "completedChecks",
  "requiredChecks", "catalogDiscovered", "placementRejected", "constructionOrder",
  "constructionPayment", "constructionComplete", "constructionCancelOrder",
  "constructionCancelPayment", "constructionRefund", "productionQueue", "productionPayment",
  "productionCancel", "productionRefund", "rallyCommitted", "unitSpawned", "rallyFollowed",
  "supplyBlocked", "complete", "failureDetail"},
  "matches": [{"index", "winner", "simSeconds"}],
  "alarms": [{"watchdog", "player", "detail"}],
  "performance": {"enabled", "warmupWallSeconds", "sampleWallSeconds", "renderingMode",
  "requestedSampleWallSeconds", "frameSamples",
  "frameAverageMs", "frameP50Ms", "frameP95Ms", "frameP99Ms", "worstFrameMs",
  "hitchesOver50Ms", "hitchesOver100Ms", "hitchesOver200Ms", "sampledPeakActors",
  "sampledPeakPawns", "sampledPeakRTSUnits", "sampledPeakProjectiles",
  "sampledPeakComponents", "sampledPeakWidgets", "sampledPeakActiveOrders",
  "sampledPeakQueuedProduction", "initialUsedPhysicalBytes", "peakUsedPhysicalBytes",
  "finalUsedPhysicalBytes", "unrealTrackedMemoryAvailable", "initialUnrealAllocatedBytes",
  "peakUnrealAllocatedBytes", "finalUnrealAllocatedBytes", "garbageCollectionCount",
  "garbageCollectionTotalMs", "garbageCollectionWorstMs",
  "network": {"samples", "peakConnections", "peakConnectionInBytesPerSecond",
  "peakConnectionOutBytesPerSecond", "peakConnectionInPacketsPerSecond",
  "peakConnectionOutPacketsPerSecond", "peakConnectionQueuedBits", "latestInTotalBytes",
  "latestOutTotalBytes", "latestInTotalPackets", "latestOutTotalPackets",
  "latestInTotalPacketsLost", "latestOutTotalPacketsLost",
  "latestAcceptedStrategicRequestWorkUnits", "latestRejectedStrategicRequestWorkUnits"},
  "workCounters": {"available", "replicationGatherCalls", "replicationCandidates",
  "replicationAdmitted", "replicationStreamingLevelRejects", "replicationVisibilityRejects",
  "fogTextureUploads", "fogPixelsProcessed", "fogTextureUploadBytes"},
  "targetRTSUnits", "runId", "captureRole", "synchronized", "authoritativeReadyRTSUnits",
  "authoritativeFinalRTSUnits", "authoritativePopulationSamples",
  "authoritativeMinimumRTSUnits", "authoritativeMaximumRTSUnits",
  "authoritativeBelowTargetSamples", "authoritativeBelowTargetFraction",
  "workloadParticipantOwners", "workloadWorkerClass", "workloadCombatClass",
  "workloadInitialReadyWorkers", "workloadInitialReadyCombatUnits", "workloadSpawnedUnits",
  "workloadReplacementUnits", "workloadSpawnFailures",
  "budget": {"evaluated", "passed", "profile", "violations"},
  "csvCapture": {"requirement": "required|not-applicable",
  "status": "finalized|invalid|not-applicable", "filename"}},
  "stats": {per-player headline numbers}}`.
