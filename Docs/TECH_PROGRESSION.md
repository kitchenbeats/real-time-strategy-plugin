# Technology Progression Contract

Technology progression is server-authoritative, per-player, and data-driven. `RTSRequirementsComponent` defines
product prerequisites, `RTSResearchComponent` purchases and times research, and the replicated
player-level `RTSUpgradeComponent` stores completed levels. All runtime mutations are truthful
authority-only Blueprint/C++ transactions.

Requirement classes are polymorphic: owning a ready Blueprint child satisfies a requirement authored
against its parent class. This is the reskin-and-extend contract—customers can derive faction-specific
units and structures without duplicating the technology tree. Framework ownership and standard
Unreal controller ownership resolve to the same player identity, while missing identity and neutral
actors fail closed instead of borrowing unrelated world actors.

Research payment, cancellation refunds, and completion are guarded transactions. Refund callbacks
cannot reenter the queue, and a failed upgrade commit leaves paid research at 100% for a safe retry.
Unknown upgrades, invalid contexts, malformed catalogs, missing tiers, missing actors, and missing
upgrade levels produce structured `RTSTechRequirementResult` reasons for Blueprint UI and AI.
