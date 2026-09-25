# API Change Policy

This page explains how the plugin's Blueprint and C++ API may change between versions, and what you
need to do when it does.

## What this means for you

- **Before the first Fab release (now):** names, pins and C++ signatures may still change between
  versions. Every intentional change, with what you would need to do about it, is listed in the
  [API Change Log](API_CHANGE_LEDGER.md). Read it before you update.
- **From the first Fab release (version 1.0.0):** patch releases keep your assets and code working;
  breaking changes wait for a new major version, and renamed types keep working through Unreal
  redirects.
- The supported C++ headers are listed in [C++ API Surface](CPP_API_SURFACE.md). Other headers under
  `Public` compile, but are not a promise.
- Your Content Set data is upgraded through [Content Set Migration](CONTENT_SET_MIGRATION.md), not
  by this policy.

## How changes are made

The rest of this page is the policy the plugin's maintainers follow.

| Stage | Applies when | Rule |
| --- | --- | --- |
| **Pre-release (current)** | No published Fab listing exists | Breaking changes are permitted and expected. Every intentional change is recorded in `Docs/API_CHANGE_LEDGER.md`. |
| **Released** | From the first published Fab listing, tagged `v1.0.0` | The compatibility rules below apply in full; breaking changes require a major version. |

UE 5.8 is the primary engine target. `Config/PublicHeaders.v1.txt` is the authoritative public-header
path manifest; `RTS.Packaging.PublicApiFreeze` verifies that manifest, a normalized fingerprint of
every public C++ header, and a canonical fingerprint of the reflected Unreal contract. Both
fingerprints are change detectors and they operate in **both** stages. What the stage decides is what
you may do once one of them moves.

## What the fingerprints cover

The reflection fingerprint covers plugin and examples classes, structs, enums, Blueprint-visible
functions, events, properties, parameter names/types/flags/defaults, and customer-relevant metadata
such as categories, display names, world context, advanced pins, auto-created references, and
request-result labels.

The two fingerprints are independent, and their disagreement is diagnostic:

- **Header hash moves, reflected hash does not** — header text changed, contract did not: a comment,
  formatting, or a private member. Safe.
- **Both move** — a real contract change. Requires a ledger entry.
- **Reflected hash moves, header hash does not** — should be impossible. Investigate before
  proceeding: it implies a contract change arriving from somewhere other than the public headers.

Before release, a separate test project that contains only the installed plugin compiles every
header listed in `CPP_API_SURFACE.md` on its own, plus a game module with real gameplay and HUD
extensions, and packages and plays it. This catches a header that no longer compiles by itself or an
extension point that no longer works.

## Pre-release rules (current stage)

1. Breaking changes are allowed. Prefer making the structure right over preserving a signature that
   no customer has ever compiled against.
2. Every intentional public API change gets a `Docs/API_CHANGE_LEDGER.md` entry before the fingerprint is
   re-baselined: what changed, why, and what a customer would have had to do about it.
3. Re-baseline a fingerprint only in the same commit as the change that justifies it, with the reason
   in the commit message.
4. Obsolete experiments are removed, not deprecated. There is no installed base to protect, and
   carrying dead API into `v1.0.0` makes it permanent.
5. Renaming a reflected type still requires an Unreal core redirect. Saved assets reference these
   types, so **asset** compatibility matters even where **source** compatibility does not.

## Released rules (from v1.0.0)

- Patch releases may fix implementation, performance, security, diagnostics, documentation, or
  defaults only when existing serialized assets and source integrations remain compatible.
- Additive APIs require a deliberate manifest/fingerprint update, a customer-use example, external
  consumer compilation, Blueprint metadata audit, cook/package proof, and release-note entry.
- Renaming, moving, removing, changing parameter order/type/default, narrowing availability,
  changing authority or purity, or changing a return contract is breaking. It requires a major API
  version unless an Unreal redirect and migration preserve both assets and supported source use.
- Deprecated APIs remain functional for the documented deprecation window after release.
- Public paths are case-sensitive contract values even on case-insensitive development hosts.
- Native-only signature changes that are not represented by reflection still require review against
  `CPP_API_SURFACE.md`, the external consumer recipes, and all feature extension examples.

## Intentional change procedure

1. Explain the customer need and classify the change as compatible, additive, or breaking.
2. Update the implementation, relevant feature contract, and extension example together.
3. Regenerate the header manifest only for an intentional path addition/removal.
4. Update the reflected fingerprint only after reviewing its canonical API diff.
5. Run strict UE 5.8 compilation, the external consumer module, Blueprint API audit, complete source
   automation, BuildPlugin, installed automation, cook, and supported packaged-play gates required by
   the release tracker.
6. Record migration and release notes before publishing the release candidate.

Never update a baseline merely to make the freeze test green. A changed fingerprint is a review
trigger, not generated noise — in either stage.

## Transition to the released stage

At first Fab publication: tag `v1.0.0`, record the shipped fingerprints in `Docs/API_CHANGE_LEDGER.md` as
the v1 baseline, and replace the pre-release section above with a pointer to that tag. From that
moment the released rules are the only rules.
