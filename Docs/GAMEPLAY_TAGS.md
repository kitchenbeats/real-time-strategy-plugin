# Gameplay Tags

`RTSGameplayTagsComponent` is the replicated source of truth for runtime actor tags used by orders,
requirements, construction, harvesting, combat, abilities, and UI. Author default tags on the
component or through `SetInitialTags` before registration. At BeginPlay, authority combines those
defaults with tags contributed by the actor's `RTSGameplayTagsProvider` implementations and commits
the result as one runtime transaction. Clients consume the replicated container and its RepNotify.

`Add Gameplay Tag(s)` and `Remove Gameplay Tag(s)` are authority-only Blueprint/C++ transactions.
They return true only if the explicit tag set changes, wake the owning actor for replication, and
broadcast `Current Tags Changed` after the new set is stable. Invalid tags, missing actors or
components, repeated operations, client mutation, and nested mutation from customer callbacks are
rejected without broadcasting. The matching gameplay-tag library helpers use the same contract and
safely return false for null or untagged actors.
