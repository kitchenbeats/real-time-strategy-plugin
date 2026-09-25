# Technology Progression

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

Research presentation is separate from gameplay identity. Author optional localized `DisplayName`
and `Description` on each Content Set research option, then regenerate; generated runtime
`FRTSResearchOption` values carry the same fields. Keep `UpgradeKey` unchanged when renaming a line
so its weapon/armor categories, costs and prerequisites still refer to the same upgrade.

Command labels and descriptions use the selected provider's metadata. Cancellation identifies its
authored title. Cross-key prerequisites and completion messages resolve the current owner's
research definitions, with construction-catalog fallback when no live provider defines the key.
Missing or conflicting names fall back to the stable key; empty descriptions stay empty. Completed
HUD events retain their copied text after a provider is removed. Runtime configuration remains
server-authoritative; these authored catalog fields add no separate metadata replication channel.
For client display, include the same authored definitions in the cooked project assets.
