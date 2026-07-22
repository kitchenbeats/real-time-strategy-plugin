# Player Advantage Extension Contract

`RTSPlayerAdvantageComponent` is the replicated authority for per-player handicaps and controlled
test advantages. The bundled player controller includes it, so Blueprint-only projects can set god
mode, construction/production speed, and outgoing damage without replacing framework code.

All three setters are authority-only Blueprint/C++ transactions and return true only when a valid
value changes. Speed and damage multipliers must be finite and within 0–100; rejected values leave
the prior profile intact. Each commit wakes the controller, replicates all values before a revision
notification, and broadcasts `On Advantages Changed` with one coherent profile. Nested mutation
from that customer event is rejected.

God-mode commits immediately update every pawn currently owned by the controller. Newly transferred
pawns receive the same policy from `RTSGameMode::TransferOwnership`. Gameplay systems read speed and
damage factors on authority, while owning-client HUDs receive the replicated values for honest match
strip and handicap presentation.
