# CDIF Core

Core interoperability specification for CDIF (Cross-Domain Interoperability Framework). The CDIF Core profile defines the mandatory and optional base properties for any CDIF metadata record, implemented using the schema.org vocabulary.

## Specification

- **[CDIFCoreImplementationGuide.md](CDIFCoreImplementationGuide.md)** — Complete documentation of all classes and properties for the CDIF Core profile, including required/optional properties, data types, and JSON-LD implementation guidance.
- **[cdifCoreStructuredSchema.json](cdifCoreStructuredSchema.json)** — JSON Schema (Draft 2020-12) for validating CDIF Core profile instances. Generated from the [metadataBuildingBlocks](https://github.com/Cross-Domain-Interoperability-Framework/metadataBuildingBlocks) source schemas.
- **[coreRules.shacl](coreRules.shacl)** — SHACL validation shapes for CDIF Core. Copied from [`metadataBuildingBlocks/_sources/cdifProperties/cdifCore/rules.shacl`](https://github.com/Cross-Domain-Interoperability-Framework/metadataBuildingBlocks/blob/main/_sources/cdifProperties/cdifCore/rules.shacl) and should be updated whenever the source changes.

## Required properties

Every valid CDIF Core instance must include:

- `@id` — Resource identifier (IRI)
- `@type` — Must include `schema:Dataset`
- `@context` — JSON-LD context with `schema`, `dcterms`, `dcat`, `prov` prefixes
- `schema:name` — Descriptive name
- `schema:identifier` — Primary identifier (string or PropertyValue)
- `schema:dateModified` — Last update date (ISO 8601)
- `schema:license` OR `schema:conditionsOfAccess` — Rights/access information
- `schema:url` OR `schema:distribution` — Access to the resource
- `schema:subjectOf` — CatalogRecord with `dcterms:conformsTo` including `https://w3id.org/cdif/core/1.1`

## Examples

The `examples/` directory contains 40+ validated JSON-LD dataset examples that conform to the CDIF Core profile. These are records that do NOT use Discovery-level properties (spatialCoverage, temporalCoverage, variableMeasured) — records with those properties are in the [Discovery repository](https://github.com/Cross-Domain-Interoperability-Framework/Discovery/tree/main/examples).

Sources include:

| Source | Count | Description |
|--------|-------|-------------|
| **CDIF/ESIP/ODIS** | 9 | Minimal, simple, and template examples |
| **ADA** | 10 | Core-stripped records from ADA test corpus |
| **Kaggle** | 10 | Iris, Netflix, Titanic, etc. — harvested from landing page JSON-LD |
| **Dataverse** | 7 | Harvard, Borealis, DANS, JHU — via Dataverse schema.org export API |
| **PSDI DCAT** | 5 | CSD, ChASe, AFLOW, etc. — converted from DCAT via `DCAT/dcat_to_cdif.py` |
| **FEDEO** | 1 | Satellite data collection with WebAPI distribution (OpenSearch) |

All examples pass cdifCore JSON Schema validation.

## JSON-LD Framing and Validation

**`FrameAndValidate.py`** frames a JSON-LD document against the CDIF Core schema and optionally validates it:

```bash
# Frame and validate
python FrameAndValidate.py examples/exampleCdifCoreMinimal.json --validate

# Frame and save output
python FrameAndValidate.py examples/exampleCdifCoreMinimal.json -o framed.json

# Use a different schema
python FrameAndValidate.py input.jsonld --validate --schema my-schema.json
```

The script uses **`cdifCore-frame.jsonld`** to frame JSON-LD documents into the expected property structure. Context prefixes from the input document are automatically merged into the frame, so domain-specific prefixes (e.g. `ada:`, `xas:`) work without being pre-declared in the frame.

**Requirements:** `pyld`, `jsonschema` (`pip install pyld jsonschema`)

## SHACL Validation

**`coreRules.shacl`** contains SHACL shapes for validating CDIF Core instances. This file is copied from [`metadataBuildingBlocks/_sources/cdifProperties/cdifCore/rules.shacl`](https://github.com/Cross-Domain-Interoperability-Framework/metadataBuildingBlocks/blob/main/_sources/cdifProperties/cdifCore/rules.shacl) and should be updated whenever the source changes.

## Repository structure

```
├── CDIFCoreImplementationGuide.md  Classes and properties documentation
├── cdifCoreStructuredSchema.json    JSON Schema for validation
├── cdifCore-frame.jsonld           JSON-LD frame for document framing
├── FrameAndValidate.py             JSON-LD framing and JSON Schema validation
├── coreRules.shacl                 SHACL validation shapes (synced from metadataBuildingBlocks)
├── examples/                       40+ validated Core-only JSON-LD examples
├── ODIS/                           Archived ODIS type-specific templates and examples
└── LICENSE
```

## Relationship to other CDIF repositories

- **[Discovery](https://github.com/Cross-Domain-Interoperability-Framework/Discovery)** — Extends Core with spatialCoverage, temporalCoverage, variableMeasured, measurementTechnique, quality
- **[metadataBuildingBlocks](https://github.com/Cross-Domain-Interoperability-Framework/metadataBuildingBlocks)** — Source building block schemas, SHACL rules, and profile definitions
- **[validation](https://github.com/Cross-Domain-Interoperability-Framework/validation)** — Framing, batch validation, conformance checking, harvesting, and DCAT conversion tools

## Changelog — v1.1.0

Released 2026-09-10 as `v1.1.0`. Content synced from the CDIF
**metadataBuildingBlocks** source; see the
[release](../../releases/tag/v1.1.0) for the tagged snapshot and
`git log v1.1.0` for the per-commit history:

- **Populated from metadataBuildingBlocks** — `*StructuredSchema.json`, merged SHACL,
  JSON-LD frame, examples, and the normative `FrameAndValidate.py` generated from the
  building-block source; `Examples/` renamed to `examples/`.
- **CDIF v1.1** — profile conformance URIs migrated `/1.0` → `/1.1`.
- **License** standardized on CC-BY-4.0.
- **`@id`-reference tightening** — bare `{@id}` reference slots sealed
  (`additionalProperties: false` + `required: ['@id']`); a canonical `objectReference`
  building block introduced as the strict node reference.
- **`prov:used` wrapper reconciliation** — the base `generatedBy.prov:used` accepts
  role-keyed wrappers (`schema:instrument` / `bios:computationalTool` / `prov:reagent`)
  alongside string / `{@id}` / inline `prov:Entity`; profiles pin a wrapper's shape via
  a constraint-only `if/then` (never a narrowed `anyOf`).
- **`skos:notation` → single string** at concept level (consistent with the codelist
  single-notation design).
- **`FrameAndValidate.py`** (normative, drift-checked against
  `Cross-Domain-Interoperability-Framework/validation`) — two-frame root-`@type`
  selection, context-aware `schema:about`, `--conformance` detection, `cdif:`-`@id`
  re-expansion, and (2026-08) reference-collapse on all document types + blank-node
  dedupe + agent `schema:identifier` unwrap, so `@embed:@always`-framed documents
  validate against the tightened schemas.
- **Examples** conformed to the tightened schemas throughout (PrimaryKey →
  `cdi:ComponentPosition`, reference slots → `{@id}`, CVE `hasIntendedDataType` →
  string, `skos:notation` → string, `schema:additionalType` URI → `{@id}`).


## Branches

`main` is the **current release** — GitHub Pages serves it, so the published
URLs always show the newest release. It is protected: changes reach it only by
pull request, which means the merge *is* the release.

New work goes on the **`updates`** branch and is merged to `main` when a release
is cut, then tagged `v1.1.n`. The former `reviewRevision202606` branch is retained
as **`archive202609`**.


## License

See [LICENSE](LICENSE).
