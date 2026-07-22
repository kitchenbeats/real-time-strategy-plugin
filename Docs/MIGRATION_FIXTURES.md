# Migration Fixtures

Migration fixtures are immutable historical inputs. Never resave them with the current editor or
replace their expected hash merely to make a test pass. Add a new versioned fixture when the schema
advances.

## Content Set schema 0

- Source asset: the original bundled `Starter/Reference/DA_RTSStarter` created before the Content
  Set custom-version GUID existed.
- Archived file:
  `Source/RealTimeStrategyEditor/Private/Tests/Fixtures/ContentSetSchema0/DA_RTSStarter_N0.uasset`
- SHA-256: `51272c6df87c6d45b7e24c36c7ae1ba39ba35eff25e53c36ca460288a4d6c288`
- Automation integrity hash (MD5, for UE's built-in file hash API):
  `A3B04AEC1D569E7C22096DDCE5D45706`
- Expected load contract: source schema `0`, absent archive custom version normalized to `0`, stable
  id `RTSStarter`, two unit definitions, and three building definitions.

`RTS.Authoring.ContentSet.ArchivedMigrationFixture` copies the unchanged package into an isolated
customer path, loads it through Unreal serialization, applies the public N-to-N+1 migration, proves
all authored properties survive, generates twice, and compiles a saved customer child Blueprint
after parent regeneration while preserving its custom default and inheritance.

The shipping `Content/Starter/Reference/DA_RTSStarter.uasset` is a separate schema-1 asset. It was
upgraded from the archived input through the public commandlet migration path; the historical bytes
remain only in the editor-test fixture above.

The exact universal BuildPlugin output carries and runs this fixture in isolated installed-package
automation. That gate migrates and regenerates twice, then compiles and reloads the customer child
while preserving its extension. The release gate also copies the immutable file into a clean
project, applies the production 0-to-1 commandlet upgrade, preserves its stable id, relocates output
under customer-owned `/Game` content, and generates the current starter. BuildCookRun then builds,
cooks, stages, packages, and archives the application. The packaged migrated map must reach exact
100-unit readiness using its regenerated project classes with zero spawn failures and a clean atomic
verdict. This closes MIG-008 without inferring migration compatibility from the separate schema-1
shipping starter.
