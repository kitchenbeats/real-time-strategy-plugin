# Supported Product Contract

This is the normative scope and claims contract for the first commercial release of the
Real-Time Strategy plugin. Architecture documents explain how systems work. Release completion is
determined by the immutable candidate's validation receipts and the required platform, fixture,
usability, legal, and support approvals. When another document uses “supported” more broadly, this
contract controls what may be promised to a customer.

The plugin is not commercially released merely because a row below defines its intended v1 scope.
A target becomes a saleable claim only when its required qualification gates pass on the exact release
artifact. “Implemented,” “tested in source,” and “works on the development host” are not substitutes
for installed, packaged, Shipping, platform, or qualification evidence.

## Frozen v1 support target

| Area | Commercial v1 target | Required release evidence | Explicit boundary |
| --- | --- | --- | --- |
| Unreal Engine | Unreal Engine 5.8.x | Clean BuildPlugin, install, compile, cook, package, Shipping, and lifecycle matrix against the declared 5.8 build | Earlier or later engine versions are separate compatibility products; they cannot constrain or inherit the 5.8 claim without their own branch and matrix. |
| Host project | Blueprint-only and native C++ game projects | Independent clean-project install, authoring, extension, package, and runtime proofs for both project types | Customers do not need StarMaps source, content, configuration, or Python. Editor Utility Blueprints and commandlets are optional automation surfaces. |
| Platforms | Win64 x64 and macOS Apple Silicon | Immutable release artifact passes the complete platform matrix on each declared OS and qualification machine | Universal Mac binaries may contain x64 slices, but Intel Mac is not a supported v1 claim without its own runtime matrix. Linux, consoles, mobile, XR, and cloud/server distributions are out of v1 scope. |
| Rendering | UE 5.8 desktop renderer on the platform-default shipping RHI | Rendered Shipping starter/Manny and UI qualification at the declared resolution/scalability settings | `NullRHI` is valid for headless correctness and server diagnostics only; it never proves rendered frame rate or presentation. |
| Project content | Bundled vendor-neutral greybox starter, plugin UI/runtime assets, and customer-owned generated content | Blank-project dependency scan, cook/package/runtime proof, regeneration/migration proof | Bundled plugin content must never depend on `/Game`, StarMaps, Manny, Mixamo, Meshy, marketplace, or other optional vendor packages. |
| Presentation workflows | Greybox/static mesh, skeletal Direct Anim Set, and Custom Animation Blueprint | Each advertised mode passes validation, generation, cook, Shipping gameplay, regeneration, and platform coverage with its declared fixtures | A generic runtime interface is not proof that every imported character works. Skeleton compatibility, clips, sockets, scale, collision, LODs, materials, root motion, and animation policy remain customer-authored data validated by the workflow. |
| Character sources | Manny profile plus vendor-neutral customer import workflows | Manny passes from a fresh UE Third Person project. Mixamo, Meshy, marketplace, and custom/studio fixtures each require their own legal, repeatable qualification row before being advertised as proven. | Epic Manny and third-party characters are not redistributed by this plugin. Customers supply content they are licensed to use. |
| Gameplay modes | Standalone skirmish, listen-server multiplayer, and dedicated-server multiplayer | Complete economy, orders, construction, production, research, abilities, combat, vision, match, restart, lifecycle, and hostile-request gates in each advertised topology | The plugin is a gameplay framework, not a lobby, matchmaking service, backend, account system, anti-cheat product, campaign, persistence layer, or live-operations service. |
| Networking | UE legacy replication with the plugin Replication Graph policy | Packaged dedicated server/clients prove authority, private state, fog relevancy, late join, disconnect policy, travel, abuse resistance, and clean shutdown | Iris competitive fog filtering is not supplied in v1. Projects using Iris must implement and qualify an equivalent connection filter or disable Iris. |
| Navigation | Unreal stock Recast navigation | Packaged movement scenario matrix across representative unit sizes, formations, obstacles, unreachable goals, replacement commands, and supported maps | `Military Only` and `All Pawns` flow replacement policies remain experimental until separately qualified. World Partition, seamless multi-world campaigns, and unusually large/streamed maps are not v1 claims without added gates. |
| Input and display | Desktop keyboard/mouse at declared standard and ultrawide resolutions | Packaged input, focus, UI scaling, resolution, localization, and accessibility matrix | Controller, touch, split-screen/local multiplayer, VR, and console navigation are not v1 claims unless added by an approved scope decision and qualification plan. |
| Scale and performance | Only tiers that pass `PERFORMANCE_SOAK.md` on declared hardware | Installed rendered client and dedicated server, raw captures, budget verdicts, network/memory evidence, scenario matrix, and required soaks | 100/500/1,000 are workload and acceptance targets, not current performance claims. A short source, `NullRHI`, or development-host pass cannot be quoted as customer capacity. |
| Extensibility | Documented Blueprint and C++ seams for data, actors, orders, abilities, AI, match policy, UI, and gameplay components | Manual Blueprint usability audit plus external Blueprint-only and native-host compile/package/runtime recipes | Editing plugin source is never the supported extension workflow. Undocumented internals and implementation file paths are not stable APIs. |
| Updates | Versioned, non-destructive Content Set and plugin migration | N-to-N+1 fixture, preview, failure recovery, regeneration idempotence, customer-extension preservation, and the isolated install/disable/reinstall/uninstall workflow in `INSTALL_LIFECYCLE_QUALIFICATION.md` | Launcher/Marketplace transport and UX remain separately qualified; no release may infer them from a local artifact copy. |
| Downloaded archive intake | Source-independent verification, safe promotion, fresh-project install, package, and runtime qualification of the canonical release set | Publication binds the exact ZIP, checksum, detached receipt, schema-3 intake tool, and schema-3 downstream tool in a manifest published last; operators independently authenticate the one manifest hash and authenticate each executable from it before execution, then both tools recheck the same set | Schema 3 is Mac-only. Tooling and adversarial tests are not proof of a real Fab/customer download, independently observed trust channel, clean machine, live downstream qualification, or Win64 intake. |

## Truthful asset and workflow wording

Use these distinctions in documentation, store copy, support, and screenshots:

- **Included:** a redistributable vendor-neutral greybox RTS starter, native UI, framework runtime,
  authoring tools, diagnostics, and focused Blueprint/C++ extension examples.
- **Integrated from the customer project:** the Manny profile discovers compatible Epic content
  already present in a UE 5.8 Third Person project and generates customer-owned RTS assets under
  `/Game`. The plugin does not distribute Manny.
- **Customer imported:** Mixamo, Meshy, marketplace, studio, and other custom characters, meshes,
  animations, materials, sounds, and effects. The plugin provides a vendor-neutral integration
  contract; it does not grant content rights or guarantee an unqualified asset.
- **Showcase only:** StarMaps factions, maps, balance, art, VFX, audio, and marketing captures may
  demonstrate the plugin but are never an install, build, or runtime dependency.
- **Experimental:** a documented opt-in that is not covered by the commercial support promise. It
  must never be described as production-ready, recommended, or included in a scale claim.

Do not use “works with Manny/Mixamo/Meshy/custom characters” as one undifferentiated claim. The
permitted wording is “vendor-neutral static and skeletal presentation workflows,” followed only by
the specifically qualified fixture list. Do not imply that a third-party character, Epic project
content, or license is bundled.

## Customer-visible claim and evidence matrix

| Claim family | Permitted wording after its release gate passes | Current evidence | Gate that authorizes the wording |
| --- | --- | --- | --- |
| Install and play | “Install the plugin in a new UE 5.8 project and open a complete playable RTS starter.” | The r11 Mac candidate at commit `d87813430b4e74515aa98b4353cdc9d0e7e2de91` passed release run `20260721T100000.286819Z-d78c4b61-b2be-4222-b28d-300401afadbe`. Its exact 1,892-file artifact has tree SHA-256 `72d2515ecbcd42f9f4159a0e31ec63c867d804bb84a4c876d69cf03b421366d1`; source and installed automation, fresh-project generation and reopen, Manny Shipping qualification, and install/disable/re-enable/uninstall/reinstall lifecycle qualification passed. The source-built engine rebuilt plugin binaries/intermediates while preserving installed non-build inputs, and the lifecycle receipt binds both states. The later exact `c3ea648` artifact passed the focused content-only Blueprint customer Shipping run `20260721T230126.973852Z-c1a64166-cd72-4018-a47c-ba2c72803cff`: configured-map launch, exactly 100 ready customer Blueprint units in 6.46 seconds, and 2/2 semantic battles in 174.39 seconds with zero alarms. No-rebuild Launcher/installed-engine delivery, the timed unassisted workflow, a clean integrated lifecycle rerun, and Win64 remain open. | `EXT-001`, `HARNESS-002`, `LIFE-001`, `PLATFORM-MAC`, `PLATFORM-WIN` |
| No hidden game dependency | “The bundled starter and runtime require no StarMaps or project `/Game` content.” | Package parity/dependency policy and isolated blank-project cook pass on Mac. | `HARNESS-002`, both platform gates, release archive scan |
| Blueprint-only workflow | “Create, validate, generate, reskin, and extend an RTS without C++.” | In a fresh content-only project, the installed factory/editor-library path creates a Content Set, exposes one deliberate repairable production-queue diagnostic, applies exactly one public safe fix, and generates 10 customer-owned extension Blueprints across all 9 advertised categories. A second editor process reopens them without recreation; the result packages and completes two matches using the customer extension wiring. Manual UI-only discoverability remains open. | `EXT-001` through `EXT-003`, `API-003` |
| C++ workflow | “Use the same systems directly from a native game module without mandatory generated gameplay Blueprints.” | A fresh host builds Editor and Game targets against only the installed artifact and declared dependencies. Its customer-owned runtime module implements the promised resource, unit, building, order, AI, GameMode/win-condition, HUD, and native-panel seams without generated gameplay Blueprints. Mac Shipping package and authoritative two-match/reset play pass; Win64 remains open. | `API-004`, `API-005`, `EXT-004` through `EXT-006` |
| Complete RTS loop | “The starter gathers, builds, produces, fights, resolves a winner, and restarts.” | Fresh Mac Manny Shipping passes 2/2 matches with automated human-player workflow and transaction checks and zero alarms. Manual usability and rendered visual review remain separate gates. | Both platform gates and the immutable RC regression |
| Reskin and go | “Replace presentation through a validated static or skeletal workflow while gameplay dimensions remain authored data.” | The Manny baseline and focused vendor-neutral static baselines are Shipping-proven on Mac. The focused r18 Direct Anim Set gate also passes exact-artifact transport, customer-owned Anim Set creation, generation, regeneration, independent reopen, Shipping packaging, and 2/2 complete battles with every required Direct state observed. The Custom Anim Blueprint path is implemented and source-qualified but awaits the current clean exact-artifact Shipping lifecycle result. This is not generic rig-family or visual-quality evidence. The generic UI-only workflow, vendor/marketplace/custom fixtures, manual rendered review, and Win64 remain open. | `FLOW-003` through `FLOW-013`, both platform gates |
| Manny | “Generate Manny-based RTS units from content in a fresh UE 5.8 Third Person project.” | Shipping-proven on Mac from an installed artifact, with repeatable Mac release-runner coverage mandatory in the full gate; Win64 remains open. | `FLOW-006`, `HARNESS-004`, both platform gates |
| Mixamo/Meshy/marketplace/custom | Name only the individual fixture that passed; state that the customer imports and licenses it. | Runtime contract exists; no vendor fixture currently has complete commercial qualification. | Corresponding `FLOW-007` through `FLOW-010` plus `FLOW-013` |
| Extensible gameplay | “Blueprint and C++ extension seams cover resources, units, buildings, orders, abilities, AI policy, win conditions, and HUD panels.” | Ten Blueprint examples and matching compiled C++ examples exist. A fresh external Blueprint-only host creates, compiles, packages, and runs customer-owned derivatives across every advertised category; a fresh native host implements, links, packages, and runs the corresponding C++ seams in Mac Shipping. Manual Blueprint UI usability and Win64 proof remain open. | `API-003` through `API-006`, `EXT-003`, `EXT-005`, `EXT-006` |
| Multiplayer | “Supports authoritative [listen/dedicated] multiplayer” only with the topology explicitly named. | Mac packaged listen-server connection/assignment/observer baseline passes. Dedicated gameplay, security, lifecycle, bandwidth, and Shipping topology gates remain open. | `NET-001` through `NET-004`, `SEC-001` through `SEC-005`, both platform gates |
| Secure fog/private state | “Server-side per-connection relevancy keeps qualified hidden state off unauthorized clients.” | Replication Graph policy and focused security contracts exist; packaged hostile/privacy matrix remains open. | `SEC-001` through `SEC-005` |
| Unit scale/frame rate | Quote only the exact qualified tier, role, platform, settings, and hardware. | Harness-only 100-unit installed authority/client synchronization passes under `NullRHI`; no customer performance claim is authorized. | Corresponding `SCALE-*`, observation, scenario, bandwidth, memory, soak, and platform gates |
| AAA/commercial quality | Do not use as a substitute for concrete evidence. State the verified workflows, platforms, topologies, fixtures, and budgets. | Required platform, topology, fixture, performance, usability, legal, and support qualification remains incomplete. | Immutable-candidate release validation and legal/support signoff |

## Unsupported and customer-owned responsibilities

For v1, support does not include debugging modified engine source, undocumented plugin internals,
unlicensed assets, unsupported engine/platform branches, customer backend services, custom Iris
filters, arbitrary networking middleware, or project code that bypasses the documented authority,
ownership, validation, and lifetime contracts.

Customers own game-specific balance, faction design, campaign/save architecture, online service
integration, art/content licenses, retargeting quality, custom Animation Blueprints, accessibility
content, localization, platform-holder requirements, and any extension whose behavior changes the
documented security or performance envelope. The plugin must still fail clearly and preserve data;
“customer-owned” never excuses corruption, unsafe defaults, silent failure, or misleading docs.

## Change control

This v1 target is frozen before public API freeze. Expanding a supported engine, platform, input
mode, network stack, topology, rig fixture, navigation policy, or scale claim requires:

1. an explicit, reviewable product-scope decision;
2. new acceptance gates and reproducible fixtures;
3. complete evidence on the immutable candidate artifact;
4. updated documentation, support policy, legal/provenance review, and claim matrix.

Removing a target requires the same explicit decision and customer-facing migration/release-note
assessment. A test failure may block a release; it may not silently narrow or weaken this contract.
