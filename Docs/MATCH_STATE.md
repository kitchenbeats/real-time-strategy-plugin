# Replicated Match-State Contract

`RTSGameState` retains the authoritative elimination ledger and final result. Blueprint and C++
game modes commit defeats through `Commit Player Defeated` and finish a match through `Commit Match
Result`; both transactions reject client calls, invalid or cross-world player states, duplicates,
post-match changes, and an eliminated winner.

`Get Defeated Players`, `Is Player Defeated`, `Is Match Over`, and `Get Winning Player` expose the
same replicated truth to gameplay code and Blueprint-only HUDs. `On Player Defeated` is replayed for
previous eliminations when a client joins late. `On Match Ended` fires from the retained final result,
with a null winner representing an explicit draw. The result latch and winner replicate as one
record, so listeners always observe one coherent result even while actor references are resolving.

`Reset Match State` supports deliberate in-place rematches: it clears the retained ledger and result,
restarts the authoritative match clock, and broadcasts `On Match Reset`. The bundled skirmish reset
uses this transaction, so HUD and gameplay state cannot leak from one skirmish into the next.
