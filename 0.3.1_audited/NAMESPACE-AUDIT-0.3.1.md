# ECMO 0.3.1 — namespace and import-resolution audit (2026-10-07)

Scope: the 45 Turtle files of `ecmo-0.3.1-dl42-patched` (23 modules, 9 alignments, 13 unit-test files), all three
`catalog-v001.xml`, every IRI under `http://data.europa.eu/h8v/ecmo/`, every prefix declaration, and every prefixed
name inside the 384 SPARQL/SHACL query literals of the unit tests. Loading was reproduced with the OWL API (ROBOT
1.9.7) through each folder's catalog, offline, which is the situation of a user behind a proxy that blocks
`ontologydesignpatterns.org` and of any user opening a file from a sub-folder in Protégé.

## 1. The Biology module

`ecmo-bio.ttl` is clean. Its 28 entities all live in `…/ecmo/bio/`, all carry `rdfs:isDefinedBy`, every `bio:` IRI
used anywhere in the network (public-health, property-alignments, the Ebola case, EpiPulse, the frame model) is
declared in bio, no local name of bio is duplicated under another ECMO namespace, its only import (`core/foundation`)
is in the catalog, and the two modules that use it (public-health, core) import it. Two cosmetic points: the ontology
comment named a class that does not exist (`BiologicalEntityType`; fixed), and the individual `bio:Class` (a
taxonomic rank) has a local name that some tooling strips to the bare word *Class* — harmless in RDF, worth knowing.

The errors reported by users therefore do not originate in bio's own namespace. What reproduces, for a user who
opens `ecmo-bio.ttl`, is **import resolution**: bio imports foundation, foundation imports DUL and d0, and the root
catalog deliberately forced those two to the web (`uri="http://www.ontologydesignpatterns.org/..."`). Any user whose
network blocks that host — the sandbox reproduced it as HTTP 403 — gets *Could not load imported ontology … d0.owl*
on opening the smallest module of the network, which is exactly the "biology errors" one would hear about.

## 2. Findings (A = catalogs/imports, B = prefixes, C = dangling IRIs, D = versions)

| # | Finding | Effect for users | Fixed by the patch |
|---|---|---|---|
| A1 | DUL and d0 forced to the web by the root catalog | Every module fails to open offline / behind a proxy (403) | Yes: `dolce/DUL.ttl` (4.2, pinned) and `dolce/d0.owl` (to be dropped in; README) |
| A1 | The alignments catalog and the unit-tests catalog contain no `<uri>` entries (only Protégé's folder group) | Opening any alignment or unit-test file from its folder fails on `core/situation`, `core`, `cqs-schema`, `mcmc` (reproduced) | Yes: all three catalogs regenerated with one entry per ontology IRI (41 ECMO + DUL + d0), relative paths |
| A1 | Unit-tests catalog maps `hub/2.4` to an absolute path on the maintainer's machine | Warning/failure for anyone else | Yes: dropped |
| A1 | `ecmo-core-pattern.ttl` (`ecmo/main-pattern`) and `ecmo-phsm-alignments.ttl` (`ecmo/phsm/alignments`) missing from the catalog | Not resolvable except by folder scan | Yes |
| A2 | `ecmo-hub.ttl` has `owl:versionInfo "0.3.1"` but `owl:versionIRI hub/2.4` | Inconsistent provenance | Yes: `hub/0.3.1` |
| A3 | `ecmo-cqs.ttl` and `ecmo-cqs-extended.ttl` declare the same ontology IRI `ecmo/cqs` | Second file refuses to load in the same OWL API manager (*ontology already exists*) | Yes: extended → `ecmo/cqs-extended` |
| B1 | `idmp:` bound to the EDM Council IDMP ontology in public-health and to `ecmo/idmp/` in the alignment module; `athina:` to `example.org/agency/` in fixtures and to `ecmo/athina/` in the alignment | Prefix collisions when files are merged, queried or read side by side | Yes: `edmidmp:` in public-health, `athinaAgency:` in fixtures (prefixes only; no IRI changed) |
| B2 | `ex:` bound to three different example namespaces (EpiPulse, Hondius, PHSM) | Silent zero results in CQs unless prefixes are scoped per file (already handled by `run_cqs.py`) | No — documented; CQ files should declare their own prefixes |
| C1 | Property-alignments align four properties that do not exist: `foundation:coordinatesDRR`, `foundation:hasPolicyObjective`, `governance:hasObservationSource`, `impact:affects` (the last also in the Ebola case) | Alignments silently inert | Yes: retargeted to `governance:coordinatesDRR`, `governance:hasPolicyObjective`, `hazard:hasObservationSource`, `expvuln:affects` |
| C2 | `spec:specializes` declared only as `owl:TransitiveProperty` | Treated as undeclared by strict tools | Yes |
| C3 | The medical-countermeasure data in public-health types 105 products and 134 indications with EDM Council IDMP classes that are neither declared nor imported, and 239 of those individuals lack a declaration | Protégé shows undeclared classes; ROBOT reports missing declarations; two parallel IDMP vocabularies (EDMC and `ecmo/idmp/`) coexist unaligned | Partly: declaration stubs for the 2 classes and 4 properties, `owl:NamedIndividual` for the 239; the alignment of `ecmo/idmp/` to EDMC is a decision for the MCM extension |
| C4 | 99 further public-health IRIs (active ingredients `API_*`, producers, jurisdictions, indication targets) have a label but **no type at all** | Untyped nodes in every tool | Partly: declared as individuals; their classes are a design decision |
| C5 | `ecmo-ph.ttl` mints **3,078 IRIs in the `ecmo/eios/` namespace** (EIOS category terms, as `owl:sameAs` targets) that the EIOS alignment module never declares | The EIOS module "owns" a namespace it does not populate; terms exist only as sameAs objects | No — decision: declare them in a generated `ecmo-eios-categories.ttl`, or move them under a WHO/EIOS IRI scheme |
| C6 | `hera:ModerateToHigh_PopulationImmunity` used as a level but never declared; the PopulationImmunity scale contains both `LowToModerate_` and `ModerateToLow_` | One assessment points to a non-existent level | No — the scale needs a curator decision |
| C7 | Coreference CQs N02–N04 and PHSM CQs N03/N04/N06 reference SHACL shapes that do not exist (`situation:NoConflictingClassificationAtSameInstantShape` etc.) | Negative tests are placeholders | No — documented |
| D1 | Three unit-test ontologies still `versionInfo "0.3.0"` | Cosmetic | Yes |

Everything else checks out: no parse errors in 45 files; no ECMO IRI under an unknown namespace (apart from the
example and shape namespaces, which are intentional); no local name of any module duplicated under another module's
namespace; every `owl:imports` target is declared by a file in the package; every prefixed name inside the 384 query
literals resolves to an existing term except the shapes of C7.

## 3. Result

After the patch (`patch_namespace_audit.py`, applied as `ecmo-0.3.1-ns-audited`), all 45 files parse, and every
module, alignment and unit-test file opens through its own folder's catalog **offline** in the OWL API (tested on
twelve representative files, including bio, public-health, hub, the pattern, CECIS, IDMP, HIP and all CQ/shape files).
Named-triple changes are limited to declarations (338 `owl:NamedIndividual`, 6 external stubs, one object-property
declaration), five version strings, one ontology IRI, four retargeted alignment subjects and one comment; no axiom of
the models changed.

The one thing the package cannot contain is `d0.owl`, which must be copied into `dolce/` once (README inside).
