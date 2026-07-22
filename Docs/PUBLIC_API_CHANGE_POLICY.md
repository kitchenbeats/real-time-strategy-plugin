# Public API Change Policy

The Real-Time Strategy plugin's commercial v1 API is frozen at API contract version 1. UE 5.8 is
the primary engine target. `Config/PublicHeaders.v1.txt` is the authoritative public-header path
manifest; `RTS.Packaging.PublicApiFreeze` verifies that manifest, a normalized fingerprint of every
public C++ header, and a canonical fingerprint of the reflected Unreal contract.

The reflection fingerprint covers plugin and examples classes, structs, enums, Blueprint-visible
functions, events, properties, parameter names/types/flags/defaults, and customer-relevant metadata
such as categories, display names, world context, advanced pins, auto-created references, and
request-result labels. The external `RTSConsumerCompile` host module independently compiles every
advertised entry header and the documented native override recipes without PCH or unity. The release
runner copies that module into a fresh native project containing only the installed BuildPlugin
artifact, cross-checks its single-header units against `CPP_API_SURFACE.md`, restricts dependencies
to the published runtime contract, and builds both Editor and Game targets. The same fresh host
also compiles a customer-owned runtime module containing real native gameplay and HUD extensions;
this prevents signature-only coverage from standing in for usable implementation seams. That same
fresh host packages Mac Shipping and proves two authoritative objective matches plus a clean reset
through a structured runtime receipt, without generated customer gameplay Blueprints.

## Compatibility rules

- Patch releases may fix implementation, performance, security, diagnostics, documentation, or
  defaults only when existing serialized assets and source integrations remain compatible.
- Additive APIs require a deliberate manifest/fingerprint update, a customer-use example, external
  consumer compilation, Blueprint metadata audit, cook/package proof, and release-note entry.
- Renaming, moving, removing, changing parameter order/type/default, narrowing availability,
  changing authority or purity, or changing a return contract is breaking. It requires a major API
  version unless an Unreal redirect and migration preserve both assets and supported source use.
- Deprecated APIs remain functional for the documented deprecation window after release. Before
  the first public release, obsolete experiments may be removed instead of preserved.
- Public paths are case-sensitive contract values even on case-insensitive development hosts.
- Native-only signature changes that are not represented by reflection still require review
  against `CPP_API_SURFACE.md`, the external consumer recipes, and all feature extension examples.

## Intentional change procedure

1. Explain the customer need and classify the change as compatible, additive, or breaking.
2. Update the implementation, relevant feature contract, and extension example together.
3. Regenerate the header manifest only for an intentional path addition/removal.
4. Update the reflected fingerprint only after reviewing its canonical API diff.
5. Run strict UE 5.8 compilation, the external consumer module, Blueprint API audit, complete source
   automation, BuildPlugin, installed automation, cook, and supported packaged-play gates required
   by the release tracker.
6. Record migration and release notes before publishing the release candidate.

Never update the baseline merely to make the freeze test green. A changed fingerprint is a review
trigger, not generated noise.
