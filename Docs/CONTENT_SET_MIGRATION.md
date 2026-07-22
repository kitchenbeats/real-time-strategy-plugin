# Content Set Migration and Regeneration

Content Sets and generated bases are versioned independently. Always run **Preview Content Set
Upgrade (Dry Run)** before regeneration after updating the plugin. Preview is read-only: it does not
modify objects, dirty packages, save files, invoke source control, change the manifest generation
id, or rewrite its inventory.

Blueprint and C++ editor tools call
`URTSContentSetEditorLibrary::PreviewContentSetUpgrade` and receive an
`FRTSContentMigrationPlan`. A successful function return means the preview itself completed. Check
`bGenerationAllowed` separately. `GenerateContentSet` performs the same preview and fails before
asset mutation when any blocking condition remains.

When preview reports `SchemaUpgrade`, save any unrelated Content Set edits, review the plan, then
call **Apply Content Set Upgrade**. Blueprint and C++ editor tooling use
`URTSContentSetEditorLibrary::ApplyContentSetUpgrade`. The operation upgrades only the source
Content Set; it never regenerates bases in the same action. Run preview again, resolve any remaining
ownership/inventory blockers, and invoke **Generate Content Set** separately.

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
implementation and failure policy as the Blueprint node. Immutable historical inputs and reviewed
hashes are listed in `MIGRATION_FIXTURES.md`.

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
| Unowned | An expected path exists without the exact manifest ownership tuple. | Blocked and treated as customer-owned. |
| Incompatible version | Source schema/custom version or manifest generator version is newer than this plugin. | Blocked; use a compatible/newer plugin. Never downgrade-write it. |

Every change includes its semantic asset kind, stable id, current path, expected path, explanation,
and blocking status. Results are deterministically sorted with blockers first so UI, commandlets,
CI logs, and source-control review show the same plan.

## Ownership rule

The generator owns only assets listed by its provenance manifest. Customer graphs, components,
defaults, and presentation belong in child Blueprints or composed/complete classes outside the
generated roots. Reference complete customer classes from the Content Set's `Existing...Class`
fields. The generator registers those classes without modifying them.

Do not “fix” an Unowned result by editing the manifest. Provenance adoption is a migration action
and must first prove the asset is a known legacy generator output. Likewise, do not delete Stale or
Renamed output until reference analysis, source-control checkout, backup, and rollback gates pass.
Those mutation/recovery requirements are tracked separately from this read-only preview contract.

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
