# Content Set Migration

Content Sets and generated Blueprints are versioned independently. After updating the plugin,
choose **Check for Problems** and then **Generate Game** on each Content Set. Generate first runs a
read-only upgrade preview and stops, without changing anything, if the Content Set needs an upgrade
or another blocking condition is found. The **RTS Authoring** page of the Message Log explains why.

To see the preview on its own, call **Preview Content Set Upgrade (Dry Run)** from an Editor Utility
Blueprint (node category `RTS|Authoring|Migration`). Preview is read-only: it does not modify
objects, mark packages as changed, save files, use source control, or rewrite the generation
manifest.

Blueprint and C++ editor tools call
`URTSContentSetEditorLibrary::PreviewContentSetUpgrade` and receive an
`FRTSContentMigrationPlan`. A successful function return means the preview itself completed. Check
`bGenerationAllowed` separately. `GenerateContentSet` performs the same preview and fails before
asset mutation when any blocking condition remains.

When preview reports `SchemaUpgrade`, save any unrelated Content Set edits, review the plan, then
call **Apply Content Set Upgrade**. Blueprint and C++ editor tooling use
`URTSContentSetEditorLibrary::ApplyContentSetUpgrade`. The operation upgrades only the source
Content Set; it never regenerates bases in the same action. Run preview again, resolve any remaining
ownership/inventory blockers, and choose **Generate Game** separately.

```cpp
FRTSContentMigrationPlan Plan;
TArray<FText> Messages;
if (URTSContentSetEditorLibrary::PreviewContentSetUpgrade(ContentSet, Plan, Messages) &&
    Plan.bRequiresContentSetMigration)
{
    const bool bUpgraded =
        URTSContentSetEditorLibrary::ApplyContentSetUpgrade(ContentSet, Messages);
}
```

Apply is idempotent: a current asset returns success without dirtying or saving its package. Dirty
or unsaved legacy source packages are rejected so rollback cannot discard unrelated authoring work.
Future versions are never downgrade-written. Saving a legacy asset through another editor workflow
also does not silently change its visible schema; only the explicit apply path advances it.

Commandlet/CI users can add `-ApplyUpgrade` to `RTSGenerateContentSet`; add `-UpgradeOnly` to save the
source migration without regenerating in the same process. The commandlet uses the same public apply
implementation and failure policy as the Blueprint node. The plugin includes an archived schema-0
Content Set that its automated tests upgrade to prove this path.

## Schema 2: stable repair costs and complete progression

Schema 2 stores a unit's authored `Repair.CostPerSecond` as resource-id amounts, matching production
and research costs. Runtime `FRTSRepairData` remains class-keyed. Native authoring code uses
`FRTSRepairDefinition`; gameplay code continues to use `FRTSRepairData` and can inspect a component's
complete profile with `GetRepairData()`.

Loading a schema-1 asset retains its old repair rates and class-keyed costs for migration. Apply
Upgrade resolves those resource classes to the Content Set's resource ids. Resolve missing or
ambiguous resource definitions before retrying; never clear a paid repair profile to bypass an
upgrade error. Upgrade and regenerate in the editor before cooking old paid-repair authoring data.

Generator 5 also supports unit building prerequisites, authored building attacks, and research
prerequisites and cancellation refunds. These fields default to empty catalogs or their documented
defaults; migration does not invent a faction's missing tech tree. Existing research and resource
amount structs retain their reflected identities after moving to the economy/research headers.

## Generator 6: playable faction catalogs

Generator 6 resolves `StarterScenario.Factions` (**Match Setup > Factions**) into the generated skirmish definition. Each entry
uses stable unit/building ids from its own faction and carries its display name, description, and
optional crest. The existing default worker/town hall pair must match one complete catalog entry.
Invalid or duplicate factions fail validation before generation. Every initial participant receives
that default faction; the host can configure a different lineup before starting.

The added catalog and `bRequirePlayerSetup` fields default to empty/false, preserving automatic
startup for existing assets. No source schema upgrade is needed for this additive authoring change.
Regenerate owned outputs to adopt generator 6. This generator also builds distinct lightweight
worker, infantry, support, aircraft, armored, and siege silhouettes from native primitives.
Switching to assigned static or rigged art removes the generator-owned decorative parts; gameplay
capsules, collision, and navigation dimensions remain authored independently. `FRTSStarterScenarioDefinition` retains its reflected
identity; C++ users may include `Authoring/RTSContentSetScenario.h` directly or keep the umbrella
`Authoring/RTSContentSet.h` include.

## Generator 10: aircraft speed and network resource sources

Generator 10 applies a unit definition's `MoveSpeed` to generated `MaxWalkSpeed` and, for an air
unit, `MaxFlySpeed`. A value of zero restores the corresponding engine defaults. Removing the air
capability also restores the generated flying-speed default, so a previous aircraft speed does not
remain as a stale override.

Generated neutral resource-source actors now enable actor replication in addition to their
replicated resource components. The normal per-player visibility rules still determine which
connections receive them; they are neither always relevant nor restricted to an owning connection.

The Content Set schema remains 2. Review **Preview Content Set Upgrade (Dry Run)**, then regenerate
owned outputs to adopt these defaults and rebuild/cook the project. No source-schema migration is
required for this generator upgrade. The generator does not rewrite classes you supplied through
**Replace With Class** (`ExistingUnitActorClass` or `ExistingSourceActorClass`); configure movement
and replication on those classes yourself. Keep your own assets outside the generated folders.

## Generator 12: localized research presentation

Research options now accept optional **Display Name** and **Description** fields. Generator 12
copies these into each generated research catalog; command cards, cancellation, prerequisites,
and research-completion messages use the authored title. **Upgrade Key** remains the gameplay
identity. Empty titles retain the existing key-based display, and missing or conflicting ownership
metadata falls back to that key.

The source schema remains 2. Set the labels on your project-owned Content Set, review the upgrade
preview, then regenerate and cook your game. Existing custom research classes can configure the
same optional runtime fields directly. Regeneration does not modify classes you supplied through
**Replace With Class**.

## Reported conditions

| Condition | Meaning | Generation policy |
| --- | --- | --- |
| Create | An expected base is absent. | Allowed; regeneration creates it. |
| Update | Exact path, role, stable id, and manifest ownership match. | Allowed. |
| Schema upgrade | The Content Set predates the current supported schema. | Blocked until the supported migration is applied. |
| Generator upgrade | The manifest was produced by an older supported generator. | Allowed when no other blocker exists; regeneration updates owned bases and provenance. |
| Renamed | A manifest record has the expected role/id at a previous path. | Blocked for explicit move/removal review. |
| Stale | A manifest-owned asset still exists but its definition is no longer expected. | Blocked for explicit removal review. |
| Orphaned | The manifest records an obsolete asset that is already missing. | Reported; the next safe manifest rewrite can remove the record. |
| Stray | An untracked asset exists anywhere beneath selected generated roots. | Blocked; move customer work outside generated roots. |
| Redirector | A redirector remains beneath a generated root. | Blocked until fixed and rescanned. |
| Unowned | An expected path exists but the manifest does not record Generate as its owner. | Blocked; the asset is treated as yours and left alone. |
| Incompatible version | Source schema/custom version or manifest generator version is newer than this plugin. | Blocked; use a compatible/newer plugin. Never downgrade-write it. |

Every change includes its semantic asset kind, stable id, current path, expected path, explanation,
and blocking status. Results are deterministically sorted with blockers first so UI, commandlets,
CI logs, and source-control review show the same plan.

## Ownership rule

The generator owns only assets listed by its manifest. Your graphs, components, defaults and
presentation belong in a unit's or building's **Custom Blueprint** (a child of the generated
Blueprint, saved in `<Content Set folder>/Custom`), or in a complete class of your own set through
**Replace With Class**. The generator uses those classes without modifying them.

Do not “fix” an Unowned result by editing the manifest. Provenance adoption is a migration action
and must first prove the asset is a known legacy generator output. Likewise, do not delete Stale or
Renamed output until you have checked what references it, checked it out of source control and made
a backup.

## Persistence and recovery policy

Saving is allowed only after validation, migration preview, ownership, collision, and persistence
preflight all pass. Existing read-only generated packages are submitted to the active source-control
provider for checkout before generation. If checkout is unavailable or a target remains read-only,
generation fails before asset mutation. Provider-disabled and read-only behavior is covered by a
real saved-package automation test. Enabled-provider behavior is qualified from the exact installed
BuildPlugin artifact in a clean project backed by a real repository: all 11 generated outputs are
made read-only through `svn:needs-lock`, locked through Unreal's active provider, and regenerated.
Unattended authoring connects an enabled but not-yet-available provider on demand before checkout.

Immediately before any saved generation run mutates assets, the editor creates a private recovery
set under `Saved/RTSAuthoringRecovery/<RunId>`. Every existing expected output package is copied and
the copy is hash-verified. New expected filenames are also recorded. The recovery set is not plugin
content and must never be submitted or distributed.

If generation, compilation, or sequential package saving fails, the generator:

1. restores every changed existing package and verifies its original hash;
2. removes partially saved new packages and newly created in-memory assets;
3. reloads restored packages so the editor does not retain failed generated state; and
4. clears the result object so callers cannot consume a partial output set.

Verified recovery deletes its private recovery directory. If any restore, deletion, or reload step
fails, the run remains failed and retains the recovery directory with an actionable path for manual
repair. Successful generation deletes its snapshot only after the whole deterministic package batch
has saved. Automation saves one real package, injects a failure, proves exact hashes are restored,
proves no new recovery residue remains, and then regenerates successfully in the same editor process.

The source-schema apply path uses the same policy independently: it snapshots the saved Content Set,
injects a real post-save failure, restores and reloads the source, compares every reflected authored
property, retries successfully, and verifies that a repeated apply leaves package bytes unchanged.

This protects generator-owned bases only. It does not authorize overwriting Unowned, Stray, Renamed,
or Stale results and does not replace the customer-extension ownership rules above.

## Generator 7: starter ownership accents and arena presentation

Source schema remains 2. Regenerate project-owned outputs to add the opt-in team-color component to
generated placeholder units and buildings. Replacing their visuals with assigned art removes that
generated binding; custom art can add its own explicit component/slot/parameter bindings. Gameplay
IDs, costs, dimensions and prerequisite links are unchanged. Generated maps use the separate bundled
starter floor material. Customer-authored meshes, materials and extension Blueprints stay owned by
the project. Review the generation plan before applying it to an existing Content Set.
