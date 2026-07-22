# UI Authoring Qualification

This is the manual acceptance gate for the commercial claim that a Blueprint-only customer can
install the plugin, create a ruleset, reskin it, and reach a playable RTS without assistance. The
automated Blueprint-customer gate proves correctness and packaging; it does not prove that a new
customer can discover or understand the workflow.

## Qualified environment

Run against the exact release-candidate BuildPlugin artifact in a fresh UE 5.8 Blueprint project.
Record the artifact tree SHA-256, engine build, operating system, display resolution and scaling,
project template, participant experience, start/end timestamps, and an uninterrupted screen
recording. The participant must not have contributed to the plugin or read this repository's
implementation code.

No Python, terminal command, copied project content, Editor Utility Blueprint, source edit, or verbal
instruction is allowed. Product documentation may be opened only after the participant first tries
the visible editor workflow; every documentation lookup and search term is recorded.

## Ten-minute unassisted task

Start the timer with the fresh project closed and the plugin artifact available locally.

1. Install and enable the plugin, reopen the project, and locate the RTS Content Set asset in the
   Content Browser Add menu.
2. Create a Content Set, identify its starter units/buildings/resources in Details, and find the
   validate, safe-fix, imported-presentation, and generate actions without being told their menu.
3. Deliberately set production queue count to zero, run validation, understand the finding, apply
   the advertised safe fix, and confirm validation no longer reports that issue.
4. Apply a participant-owned static mesh or compatible skeletal mesh/Anim Set to one unit through
   **RTS Authoring > Apply Imported Unit Presentation...**. The participant must be able to cancel
   without mutation, then complete the transaction intentionally.
5. Generate customer-owned content, find the generated starter map, press Play, select units, issue
   movement, and observe AI-controlled opponents operating.
6. Locate one generated Blueprint and correctly identify it as generator-owned output: presentation
   is changed on the Content Set, while gameplay extensions belong in customer-owned child/composed
   classes outside the generated root. A generated-base child is selected from a customer-owned
   roster/map seam that keeps its parent expected; a complete stable project-owned class can instead
   be supplied through `Existing*Class`. The participant must not edit the generated base or make an
   `Existing*Class` depend on the generated output that field suppresses.

The timer stops only after the playable map is running and the participant has completed the final
explanation. Continue observation after ten minutes if unfinished so the blocking interaction is
captured rather than abandoned.

## Pass/fail rules

The run passes only when all tasks complete within ten minutes with:

- no crash, hang, data loss, invalid generated asset, Blueprint compiler warning, or unexpected log
  warning;
- no edit to plugin-owned content or C++ source;
- no hidden command, undocumented recovery, or intervention from the observer;
- no ambiguous destructive action, silent validation failure, or modal dialog that leaves the
  participant unable to state what to do next;
- successful cancel/undo semantics for imported presentation and safe-fix transactions; and
- a playable generated map whose selected unit visibly uses the participant's chosen presentation.

Any product defect fails the run. Participant hesitation is not automatically a product defect, but
the observer records every hesitation over ten seconds, wrong menu, abandoned path, unclear label,
documentation lookup, and recovery. A passed run with a repeated hesitation still creates a tracked
usability issue.

## Required coverage

Run at 1280×720, 1920×1080, and 3840×2160 (or the closest native display modes) at the operating
system's supported scaling settings. Complete at least:

- one Blueprint-focused Unreal user who has not built an RTS before;
- one experienced Unreal technical designer; and
- one C++-focused Unreal user following only the visible Blueprint workflow.

Repeat the complete study on every advertised desktop platform. One expert success is not sufficient
for the commercial usability claim.

## Evidence record

For each run retain:

- immutable screen-recording SHA-256 and duration;
- exact artifact, engine, project, platform, and display identities;
- task start/completion offsets;
- ordered hesitation, wrong-path, warning, error, crash, and observer-intervention events;
- final result and blocking task when failed; and
- linked issue identifiers for every product defect or repeated usability concern.

The release evidence row may say **manual UI ergonomics passed** only when all required participants,
resolutions, and advertised platforms pass against the same release-candidate artifact. Until then,
report the completed cells precisely and leave the gate open.
