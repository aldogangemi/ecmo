# ECMO 0.3.1 — release notes (2026-09-28)

Built from 0.3.0 by `build_0_3_1.py` (textual surgery, comments preserved; every change logged in
`BUILD-LOG-0.3.1.md`; rename map in `rename-map-0.3.1.tsv`). The zip contains the 0.3.1 tree
(modules, `0.3.1-alignments/`, `0.3.1-unittests/`, `0.3.1-harness/`), plus `reasoned-0.3.1.ttl`
(HermiT classification of the merged, datatype-sanitised network).

## What changed

| Area | Change |
|---|---|
| New module | `ecmo-situation.ttl` (`…/core/situation`, prefix `situation:`; imports foundation only). Contents: the whole 0.3.0 coreference pattern; from foundation: `contributesTo`, `includesEvent`, `isEvolvedAs`, `Signal`, `EventValidation` (+10 properties), `RiskAssessmentReport` (+5 properties). New generic `situation:classifiesAs`. |
| Relocations | `hazard:HazardousEvent ⊑ situation:Occurrence` → hazard module. `HazardManifestationClassification`, `hasManifestedHazard` → hazard module (as `hazard:`), `hasManifestedHazard ⊑ situation:classifiesAs`. 191 `hera:` assessment individuals → `ecmo-hera.ttl`. |
| Retargets | `pertainsToEvent`/`confirmsEvent` range and `hasRiskAssessmentReport` domain: `hazard:HazardousEvent` → `situation:Occurrence`. `generatesSignal` domain → `dul:InformationObject`. `concernsOccurrence` domain → `dul:Situation ∪ dul:InformationObject`. |
| Defects fixed | Cyclic property chains on `hasManifestedHazard`/`isManifestationOf` (OWL 2 DL regularity; HermiT refused 0.3.0). IRI bug `ecmocore:hasJustification` → `foundation:hasJustification` (158). `hasJustification` range `xsd:string` → `rdfs:Literal` (values are `@en`; corrected IRI made the network inconsistent). `includesEvent` declared `owl:ObjectProperty`. Copy-pasted comment on `accordingTo`. |
| Annotations | `rdfs:isDefinedBy <declaring module>` on 4,857 classes/properties/individuals, appended as a generated block per file. |
| Imports/catalog | `coref` → `core/situation` everywhere; direct `owl:imports situation` added to 10 modules/alignments; catalog updated. |
| Versions | `owl:versionInfo "0.3.1"` on all 30 modules and alignments (was 0.1 / 0.2 / 0.3 / 2.0 / 0.1-proposal). |

## Verification

- Semantic diff (rdflib, whole network, old mapped through the rename map): 33,088 → 37,962 named triples; **0 unexplained** added or removed; every difference falls into a logged category.
- All 46 Turtle files (modules, alignments, unit tests) parse; renames applied inside SPARQL literals, CQ expected-binding strings and fixture namespaces; `PREFIX situation:`/`hazard:` injected into query literals where needed.
- HermiT (ROBOT 1.9.7) on the merged network with `xsd:date`/`xsd:duration` literals sanitised: 0.3.0 cannot be reasoned (irregular property hierarchy); **0.3.1 is consistent, no unsatisfiable classes**, 16 inferred named subsumptions, 4,525 inferred type assertions.
- Competency-question regression (`run_cqs.py`): coref suite P01–P09/N01 and the 158-CQ cross-module suite on the Ebola + Hantavirus cases give **identical results on 0.3.0 and 0.3.1**. Pre-existing: P09 returns 0 rows on both; N02–N04 are shape references, not queries; CQ-RES-N02 has a syntax error.
- The network's own SHACL consistency shapes: the same 22 frame-role-coherence violations on both versions (pre-existing).

## Findings not fixed (design decisions)

1. `risk:assessesRisk ⊑ risk:addressesRisk` (domain `d0:Eventuality`) entails `situation:RiskAssessmentReport ⊑ d0:Eventuality` and `governance:RiskAssessment ⊑ d0:Eventuality` — an information object becomes an event/situation. With the real DUL disjointness axioms loaded this is likely to become an inconsistency. Options: drop the sub-property axiom, or express the report's restriction with `dul:isAbout` instead of `assessesRisk`.
2. Seven `owl:imports` cycles in 0.3.0 (core↔epipulse, core↔frames, disaster↔impact↔instruments↔risk↔capacity, …). Legal, but ROBOT/WIDOCO/Protégé loading order becomes unpredictable.
3. `situation` (inherited from foundation/coref) references `risk:Risk`, `risk:assessesRisk`, `response:MeasureType`, `ph:*` examples without importing those modules; resolved only through the hub.
4. `phsm:deploysCountermeasure` is declared in `ecmo-phsm-alignments.ttl` but namespaced in `phsm`.
5. Frame model: `COREF.Occurrence` has `fschema:ontoType dul:Situation` while `Occurrence ⊑ d0:Eventuality` (one of the 22 shape violations).
6. Unit-test files declare no `PREFIX` for `ex:` and several modules bind `ex:` to different namespaces; the runner therefore scopes prefixes CQ-file > fixture > network. Each CQ file should carry its own prefix map.
7. `xsd:date` and `xsd:duration` are outside HermiT's datatype map; the harness should reason on a sanitised copy (as done here) or switch to Openllet/Pellet for the datatype layer.
