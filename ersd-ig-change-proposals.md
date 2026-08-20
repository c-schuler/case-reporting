# eRSD Section of the eCR FHIR IG — Proposed Changes

Draft list for review before conversion to Jira tickets.

**Scope:** the eRSD portion of the eCR FHIR IG (`hl7.fhir.us.ecr`, currently 3.0.0-ballot) — the
`ersd-plandefinition` and `ersd-supplemental-library` profiles, the "Electronic Reporting and
Surveillance Data (eERSD) Transaction and Profiles" page, the eRSD implementation-guidance pages,
and the eRSD examples.

**Reference point for "what we actually deliver":** eRSD v3.2.0 as produced from the
`aphl-ersd-specifications-v3` repository —
`input/resources/plandefinition/plandefinition-us-ecr-specification.json` and
`input/resources/bundle/eRSDv3_specification_bundle.json`.

**Framing note:** almost everything below is either (a) documentation of what is already being
distributed, (b) profile changes that *relax or extend* the profile so that the artifact we actually
ship becomes conformant, or (c) presentation cleanup within the existing HL7 IG template. Very little
of it adds new implementer obligations, and none of it touches the publisher template or introduces
custom styling. Items that are genuinely substantive, or where a scope decision is needed first, are
flagged **[DECISION NEEDED]** or **[LIKELY OUT OF PHASE]** so they can be split out or deferred.

**Groups at a glance.** 1 — PlanDefinition profile alignment · 2 — package structure and
distribution docs · 3 — CRMI alignment and pointers · 4 — supplemental spec and rule filters ·
5 — examples · 6 — small cleanups · 7 — page presentation and readability.

**How to read each item:** every item is written as a candidate ticket with *Context* (why this
matters), *Current state* (what the IG says today), *As delivered* (what v3.2.0 actually contains,
with evidence), *Proposed change*, *Acceptance criteria*, and *Risk / impact*. Copy-paste the block
into the ticket description.

---

# Group 1 — eRSD PlanDefinition profile alignment

**Group context.** The `ersd-plandefinition` profile in the IG was written against a workflow design
that has since evolved substantially through the v3.x releases. The profile fixes literal values
(`id`, `code`, `relatedAction.actionId`) at many points, so drift does not degrade gracefully — it
produces hard validation failures. The PlanDefinition we distribute today declares
`meta.profile = ["…/ersd-plandefinition", "…/us-ph-plandefinition"]` but would not pass validation
against the first of those. This group closes that gap.

**Suggested sequencing.** 1.1–1.5 are one coherent piece of FSH work and could reasonably be a
single ticket with five sub-tasks; 1.6–1.13 are independent and can be worked in parallel.

---

### 1.1 — Rename the "check suspected disorder" action slice to match delivery

**Type:** Profile change (breaking to the profile, non-breaking to consumers)
**File:** `input/fsh/profiles/ERSDPlanDefinition.fsh`; narrative in
`input/pagecontent/ersd_transaction_and_profiles.md`

**Context.** The action that runs immediately after `encounter-start` was renamed during the v3
triggering-optimization work. Its job broadened from "check the suspected-disorder value set" to
"check for anything immediately reportable, and schedule the follow-up checks" — hence the rename.
The profile was never updated. Because the profile fixes these ids with `(exactly)`, this is
currently the single largest source of validation failure in the delivered artifact.

**Current state (profile).**
* `* action[checkSuspectedDisorder].id = "check-suspected-disorder" (exactly)`
* `* action[encounterStart].relatedAction.actionId = "check-suspected-disorder" (exactly)`
* `* action[checkSuspectedDisorder].action[isEncounterSuspectedDisorder].id = "is-encounter-suspected-disorder" (exactly)`

**As delivered (v3.2.0).**
* `check-for-immediate-reporting` (code `execute-reporting-workflow`)
* `start-workflow.relatedAction.actionId = "check-for-immediate-reporting"`, offset 1 hour
* `is-encounter-immediately-reportable` (code `check-trigger-codes`), with the four inputs
  `suspectedDisorderConditions`, `suspectedDisorderEncounterDiagnosis`, `suspectedDisorderLabOrders`,
  `suspectedDiagnosticOrders` — note that it now covers lab and diagnostic orders against `lotc`,
  not only suspected disorders against `sdtc`, which is why the old name no longer describes it.

**Proposed change.**
* Rename the slices `checkSuspectedDisorder` → `checkForImmediateReporting` and
  `isEncounterSuspectedDisorder` → `isEncounterImmediatelyReportable`.
* Update the three fixed values above to the delivered ids.
* Update `^short` / `^definition` on both slices to describe the broadened behaviour.
* Update the walkthrough in the eRSD page, which currently narrates a `check-suspected-disorder`
  step that no longer exists by that name.

**Acceptance criteria.**
* The delivered `us-ecr-specification` PlanDefinition passes validation against
  `ersd-plandefinition` on all three of these elements.
* No occurrence of the string `check-suspected-disorder` or `is-encounter-suspected-disorder`
  remains in the profile, the eRSD page, or the examples.

**Risk / impact.** None for implementers — the profile is being corrected to describe the artifact
they already consume. Only risk is missing an occurrence; grep before closing.

---

### 1.2 — Add the missing `check-for-immediate-reporting` child actions

**Type:** Profile change (additive)
**Depends on:** 1.1 (slice rename)

**Context.** v3.x added explicit workflow-termination and completion tracking to the early branch of
the workflow, so an implementing system can stop scheduling checks for an encounter that has aged
out rather than looping indefinitely. Interestingly, the current IG page already anticipates this in
a note to implementers ("implementations may wish to extend this functionality to support…
the addition of an `is-encounter-completed` action") — we have since done exactly that, and the
profile and narrative should now reflect it as delivered behaviour rather than a suggestion.

**Current state (profile).** `* action[checkSuspectedDisorder].action 2..` with two slices
(`isEncounterSuspectedDisorder`, `continueCheckReportable`).

**As delivered (v3.2.0).** Four children:

| id | code | Purpose |
|---|---|---|
| `is-encounter-immediately-reportable` | `check-trigger-codes` | already sliced (see 1.1) |
| `continue-check-reportable` | `evaluate-condition` | already sliced |
| `terminate-late-encounter` | `terminate-reporting-workflow` | **missing** |
| `is-late-encounter-completed` | `complete-reporting` | **missing** |

Delivered logic for the two missing ones:
* `terminate-late-encounter` — fires when the encounter is still `in-progress`/`arrived` past
  `%encounterStartDate + 1 day * %normalReportingDuration`, or is `finished` and more than 72 hours
  past `%encounterEndDate`. CQL alternative: `Is Encounter Late`.
* `is-late-encounter-completed` — fires when the encounter is `finished` and past its reporting
  window, using the ambulatory window (`%ambulatoryReportingDuration`) for `AMB`/`VR`/`HH` class
  encounters and the normal window (`%normalReportingDuration`) for `IMP`/`EMER`/`OBSENC`.
  CQL alternative: `Is Encounter Complete`.

**Proposed change.** Add `terminateLateEncounter` and `isLateEncounterCompleted` slices, fixing
`id`, `code`, and `condition.kind = #applicability`; raise the minimum to `action 4..`.

**Acceptance criteria.** All four delivered children match a named slice; minimum cardinality
matches delivery; each slice has `^short`/`^definition` explaining when it fires.

**Risk / impact.** Additive to the profile. Confirm `complete-reporting` exists in the bound value
set (see 1.10) before closing.

---

### 1.3 — Add the missing `check-reportable` child actions

**Type:** Profile change (additive)

**Context.** The ambulatory reporting path is the significant gap here. v3.x split the in-progress
loop and the termination logic by encounter class, because ambulatory encounters have a much shorter
natural reporting window (delivered `ambulatoryReportingDuration` = 1 day) than inpatient encounters
(`normalReportingDuration` = 14 days). Continuing to re-check an ambulatory encounter on the
inpatient cadence generates avoidable load on the EHR for no reporting benefit. None of this is in
the profile or the narrative.

**Current state (profile).** `* action[checkReportable].action 4..` with four slices
(`isEncounterReportable`, `checkUpdateEicr`, `encounterInProgress`, `encounterComplete`).

**As delivered (v3.2.0).** Seven children:

| id | code | Sliced? | Purpose |
|---|---|---|---|
| `is-encounter-reportable` | `check-trigger-codes` | yes | main reportability check |
| `check-update-eicr` | `evaluate-condition` | yes | 72h refresh |
| `is-encounter-in-progress` | `evaluate-condition` | yes | inpatient loop |
| `is-amb-encounter-in-progress` | `evaluate-condition` | **no** | ambulatory loop |
| `terminate-encounter` | `terminate-reporting-workflow` | **no** | inpatient termination |
| `terminate-amb-encounter` | `terminate-reporting-workflow` | **no** | ambulatory termination |
| `is-encounter-completed` | `complete-reporting` | yes | completion |

The class split, which should be documented alongside: `AMB`, `VR`, `HH` take the ambulatory path;
`IMP`, `EMER`, `OBSENC` take the inpatient path.

**Proposed change.** Add `ambEncounterInProgress`, `terminateEncounter`, and `terminateAmbEncounter`
slices; raise the minimum to `action 7..`; document the class-based routing in the page narrative.

**Acceptance criteria.** All seven delivered children match a named slice; the ambulatory vs
inpatient split is described in the eRSD page.

**Risk / impact.** Additive to the profile. The narrative addition is genuinely useful new
information for implementers.

---

### 1.4 — Correct the `encounter-modified` branch structure

**Type:** Profile change (structural)

**Context.** The `encounter-modified` branch was restructured so that the reportability check is a
first-class, addressable action rather than an anonymous nested one. This matters because the
modified-encounter check carries its own 14 `modified*` inputs and its own large condition
expression, and implementations need to be able to reference it by id in logs and status tracking.

**Current state (profile).**
* `* action[encounterModified].relatedAction.actionId = "create-eicr" (exactly)`
* `* action[encounterModified].action.condition.kind = #applicability (exactly)` — i.e. the profile
  expects an unnamed *child* action carrying the condition.

**As delivered (v3.2.0).**
* `encounter-modified` (code `initiate-reporting-workflow`, trigger `encounter-modified`) has
  `relatedAction.actionId = "is-modified-encounter-reportable"` and **no child actions**.
* `is-modified-encounter-reportable` is a **top-level sibling action** (code `check-trigger-codes`),
  with `description` "This action represents the check for reportability to create the patients
  eICR.", 14 `modified*` inputs, a 4,068-character FHIRPath condition, and
  `relatedAction.actionId = "create-eicr"`.

**Proposed change.**
* Add a top-level `isModifiedEncounterReportable` slice.
* Change the fixed `action[encounterModified].relatedAction.actionId` to
  `is-modified-encounter-reportable`.
* Remove or relax the `action[encounterModified].action.condition` constraint.
* Update the page narrative, which currently says the `encounter-modified` action "specifies that if
  the encounter has extended beyond the normal reporting duration (E) `create-eicr` should be
  called" — the actual chain is `encounter-modified` → `is-modified-encounter-reportable` →
  `create-eicr`.

**Acceptance criteria.** Delivered `encounter-modified` and `is-modified-encounter-reportable` both
validate; the page's process diagram/list shows the two-step chain.

**Risk / impact.** Profile-side only.

---

### 1.5 — Raise the top-level action minimum

**Type:** Profile change (one line)
**Depends on:** 1.4

**Context.** Housekeeping consequence of 1.4 — but worth its own line so it isn't lost.

**Current state.** `* action 7.. MS`

**As delivered.** Eight top-level actions: `start-workflow`, `check-for-immediate-reporting`,
`check-reportable`, `create-eicr`, `validate-eicr`, `route-and-send-eicr`, `encounter-modified`,
`is-modified-encounter-reportable`.

**Proposed change.** `* action 8.. MS`.

**Acceptance criteria.** Cardinality matches the delivered artifact.

---

### 1.6 — Profile and document `action.output`

**Type:** Profile change + documentation

**Context.** The `output` element is how the delivered PlanDefinition expresses the hand-off of the
report between the last three workflow steps, and — importantly — it is the only place in the eRSD
artifact that formally connects the eRSD workflow to the eICR profile defined elsewhere in this same
IG. That linkage is worth surfacing explicitly, because it answers "what exactly is this workflow
supposed to produce?" with a canonical rather than prose.

**Current state.** The profile does not mention `action.output`. The page narrative describes
create → validate → route-and-send purely as prose.

**As delivered (v3.2.0).** All three outputs are typed `Bundle` and profiled to
`http://hl7.org/fhir/us/ecr/StructureDefinition/eicr-document-bundle`:

| Action | `output.id` |
|---|---|
| `create-eicr` | `eicrreport` |
| `validate-eicr` | `valideicrreport` |
| `route-and-send-eicr` | `submittedeicrreport` |

The corresponding `input` on each downstream action (`generatedeicrreport`, `validatedeicrreport`)
consumes the prior step's output.

**Proposed change.** Constrain `output` on the three slices (`1..1`, type `Bundle`, profile
`EICRDocumentBundle`), and add a short paragraph to the eRSD page explaining the
output → input chaining and the eICR profile linkage.

**Acceptance criteria.** Profile constrains all three outputs; page states that the eRSD workflow
produces an `eicr-document-bundle`.

**Risk / impact.** Low. Pinning the profile canonical is a genuine (small) new constraint — confirm
we are content to bind eRSD to the eICR bundle profile at this level.

---

### 1.7 — Profile and document the `create-eicr` input set

**Type:** Documentation (+ optional profile change)

**Context.** This is probably the highest-value documentation item in the whole list. The
`create-eicr` action is where the eRSD tells an implementing system *what data to gather to build
the eICR*, and it does so with 29 `input` DataRequirements each carrying a default FHIR query. For
an EHR team, this is the actual integration spec. It is currently invisible in the IG — an
implementer would have to read the raw JSON to find it.

**Current state.** Not profiled, not documented. The page's only mention of `create-eicr` is one
sentence: "involves the marshaling of FHIR resources needed to create the eICR profile."

**As delivered (v3.2.0).** 29 inputs. Grouped by what they contribute:

* **Core clinical** — `patientdata`, `encounterdata`, `conditiondata` (problem-list-item),
  `encounterDiagnosesData` (encounter-diagnosis), `procdata`
* **Medications** — `mrdata` (MedicationRequest, `intent=order`), `medAdmdata`, `medStatementdata`,
  `medDispensedata`
* **Immunizations** — `immzdata`
* **Labs / diagnostics** — `labOrderdata`, `labResultdata`, `diagnosticOrderdata`,
  `diagnosticResultdata`
* **Occupational data (ODH)** — `odhData-loinc` (LOINC 11295-3, 11341-5, 21843-8, 74165-2),
  `odhData-snomed` (SNOMED 224362002, 364703007)
* **Pregnancy** — `pregnancyObservations` (LOINC 90767-5), `pregnancyConditions` (SNOMED 77386006,
  146799005, 60001007), `pregnancy-status` (LOINC 82810-3), `lmp-data` (LOINC 8665-2),
  `postpartum-status` (SNOMED 249197004), `pregnancy-outcome` (SNOMED 17369002, 21243004, 237364002,
  282020008, 57797005)
* **Social / contextual** — `travelData-snomed` (9 SNOMED codes), `homeless-data` (SNOMED 32911000,
  105526001), `disability-data` (LOINC 69856-3 … 69861-3), `nationality-data` (SNOMED 186034007),
  `residency-data` (LOINC 77983-5), `vaccine-cred-data` (LOINC 11370-4)
* **Vitals** — `vitals-data`, scoped to the encounter with
  `&date=ge{{context.encounterStartDate}}`

**Proposed change.**
* Add a subsection to the eRSD page: "What `create-eicr` gathers", with the grouped table above and
  the statement that each input id is addressable as a `%` variable.
* State that these inputs align with the eICR Data Elements page, and cross-link the two.
* **[DECISION NEEDED]** whether to also slice these 29 inputs in the profile. Recommendation: **no**
  — document them, but keep the profile at `input 0..* MS`. Slicing 29 inputs makes the profile
  brittle against exactly the kind of drift this whole exercise is correcting, and every future
  eICR data element addition would become a profile change.

**Acceptance criteria.** The eRSD page enumerates the `create-eicr` inputs and their purpose; an
implementer can determine the required queries without opening the JSON.

**Risk / impact.** Documentation-only under the recommended option.

---

### 1.8 — Document the PlanDefinition-level `variable` extensions and the timing model

**Type:** Documentation (+ optional profile change)

**Context.** The IG's "Parameters" section still describes five abstract parameters labelled **A**
through **E** and never names the variables that actually implement them. Meanwhile the delivered PD
declares nine variables, three of which (`dxTimeboxDuration`, `labTimeboxDuration`,
`extendedTimeboxDuration`) implement the 3.2.0 timebox behaviour that has no counterpart in the A–E
scheme at all, and one of which (`negativeLabResultValueSet`) is not a duration. An implementer
reading the IG cannot map the guidance onto the artifact.

**Current state (profile).** `extension[variable] 0..*`, unsliced, with the generic definition
"Defines variables for the PlanDefinition." **Current state (page).** Parameters A–E, narrative only.

**As delivered (v3.2.0).** Nine `http://hl7.org/fhir/StructureDefinition/variable` extensions:

| Variable | Value | Maps to | Notes |
|---|---|---|---|
| `normalReportingDuration` | `14` | Parameter E | days; inpatient window |
| `ambulatoryReportingDuration` | `1` | *(new — no A–E equivalent)* | days; ambulatory window |
| `dxTimeboxDuration` | `30` | *(new)* | days; diagnosis/problem evidence age limit |
| `labTimeboxDuration` | `30` | *(new)* | days; lab evidence age limit |
| `extendedTimeboxDuration` | `365` | *(new)* | days; applies to `eltc` conditions |
| `negativeLabResultValueSet` | canonical | *(new)* | not a duration |
| `encounterStartDate` | `{{context.encounterStartDate}}` | context | supplied by the implementing system |
| `encounterEndDate` | `{{context.encounterEndDate}}` | context | " |
| `lastReportSubmissionDate` | `{{context.lastReportSubmissionDate}}` | context | " |

**Two implementation details that must be documented, because they are surprising:**

1. **The duration variables are plain integers, not Quantities.** Expressions coerce them with the
   `1 day * %variable` pattern — e.g.
   `%encounterStartDate + 1 day * %normalReportingDuration`. This pattern appears 106 times in the
   delivered PD. It is deliberate (Quantity-typed variables hit a parameter-binding failure in the
   CQL engine), but an implementer will not guess it.
2. **Parameters C and D are hardcoded, not variables.** The literal `72 hours` appears 19 times —
   as the post-encounter grace window (Parameter D) and in `check-update-eicr`'s
   `%lastReportSubmissionDate < now() - 72 hours` (Parameter C). The IG presents these as tunable
   parameters; in the delivered artifact they are not.

**Proposed change.**
* Rewrite the Parameters section around the delivered variable names, with a mapping table back to
  A–E for continuity with prior versions.
* Document the `1 day * %variable` coercion pattern explicitly.
* State plainly which timings are variable-driven and which are currently fixed at 72 hours.
* **[DECISION NEEDED]** whether to slice `extension[variable]` by name in the profile.
  Recommendation: slice the six configuration variables, leave the three context variables open.

**Acceptance criteria.** Every delivered variable is named and explained in the page; the A–E mapping
is preserved; the 72-hour literals are disclosed.

**Risk / impact.** Documentation-only unless the slicing option is taken. Item 2 may prompt a
follow-up question about whether C and D *should* become variables — worth anticipating, but out of
scope for this ticket.

---

### 1.9 — Reconcile the action code system and named-event canonicals **[DECISION NEEDED]**

**Type:** Decision, then profile and/or artifact change
**Blocks:** 1.10

**Context.** Several eRSD terminology and extension canonicals were moved from the eCR IG namespace
(`hl7.org/fhir/us/ecr/…`) into the US Public Health Library IG
(`hl7.org/fhir/us/ph-library/…`). The IG profile was updated to the new namespace; the delivered
artifact still emits the old one. This is not a documentation slip — it is an unresolved migration,
and resolving it in the wrong direction breaks either validation or every deployed consumer.

**The split, in full.**

| Artifact element | Profile / IG expects | Delivered v3.2.0 emits |
|---|---|---|
| Action codes | `…/us/ph-library/CodeSystem/us-ph-codesystem-plandefinition-actions` | `…/us/ecr/CodeSystem/us-ph-plandefinition-actions` |
| Named event codes | `…/us/ph-library/CodeSystem/us-ph-codesystem-triggerdefinition-namedevents` | `…/us/ecr/CodeSystem/us-ph-triggerdefinition-namedevents` |
| Named event type extension | `…/us/ph-library/StructureDefinition/us-ph-named-eventtype-extension` (also in invariant `epd-1`) | `…/us/ecr/StructureDefinition/us-ph-named-eventtype-extension` |

**Options.**
* **(a)** Update the delivered eRSD to the ph-library canonicals. Cleanest end state; requires
  coordination with eCR Now and any custom implementer that pattern-matches on the code system URL,
  and a release in which to land it.
* **(b)** Document the eCR canonicals as the ones in force for eRSD v3, with a stated migration
  path and target release. Lowest risk for this phase.
* **(c)** Have the profile accept both during a transition window.

**Recommendation.** (b) for this phase, with (a) tracked as a follow-up tied to a specific release —
it keeps the ballot honest without forcing a coordinated artifact change under time pressure.

**Acceptance criteria.** A decision is recorded; the profile, the invariant `epd-1`, and the page
examples all consistently reflect it.

**Risk / impact.** This is the highest-risk item in the list and the one most likely to attract
scrutiny. Recommend raising it as a decision ticket first, separate from any implementation ticket.

---

### 1.10 — Confirm every delivered action code exists in the bound value set

**Type:** Terminology verification, possibly terminology change
**Depends on:** 1.9

**Context.** The profile fixes `action.code` values on every slice. If a fixed code is not in the
code system it claims to come from, the profile is asserting something untrue and validators will
flag it. We should verify rather than assume, because the code set demonstrably grew during v3.

**Codes used across the delivered PD (9 distinct):**
`initiate-reporting-workflow`, `execute-reporting-workflow`, `terminate-reporting-workflow`,
`check-trigger-codes`, `evaluate-condition`, `complete-reporting`, `create-report`,
`validate-report`, `submit-report`.

**Known gap.** The eCR 2.1.1 `us-ph-plandefinition-actions` code system contains only four concepts:
`initiate-reporting-workflow`, `execute-reporting-workflow`, `terminate-reporting-workflow`,
`report-chronic-disease-surveillance`. The other six codes we use are not in it.

**Proposed change.**
* Verify against ph-library 2.0.0 (the version this IG depends on) whether the six missing codes are
  present.
* For any that are absent, raise the addition with the ph-library maintainers, or define them in
  this IG if that is the agreed home.
* Note that `report-chronic-disease-surveillance` is defined but unused by eRSD — confirm that is
  intentional.

**Acceptance criteria.** All nine codes resolve in the bound value set; any additions are tracked to
their owning IG.

**Risk / impact.** May generate a dependency on another IG's release cycle. Worth surfacing early.

---

### 1.11 — Reconcile the FHIR query pattern extension canonical

**Type:** Profile change + documentation
**Related to:** 1.9 (same namespace-migration root cause)

**Context.** Same story as 1.9, but for the extension that carries the default FHIR query on each
input. This one has a second, larger problem attached to it — see "undocumented aliasing" below,
which is arguably more important than the URL itself.

**Current state.**
* Profile: `action.input.extension contains http://hl7.org/fhir/StructureDefinition/cqf-fhirQueryPattern named fhirquerypattern 0..1 MS`
* `ersd_plandefinition_datarequirement_fhir_query.md` shows the same `cqf-fhirQueryPattern` URL, and
  uses `…/us/ecr/ValueSet/valueset-diagnosis-problem-triggers-example` in its example.

**As delivered (v3.2.0).**
* Extension URL is `http://hl7.org/fhir/us/ecr/StructureDefinition/us-ph-fhirquerypattern-extension`.
* Value sets referenced in `codeFilter` are the real groupers, versioned —
  e.g. `http://ersd.aimsplatform.org/fhir/ValueSet/dxtc|3.2.0`.

**Undocumented aliasing — needs its own paragraph in the page.** Not every `valueString` is a FHIR
query. Many are the *id of another input*, meaning "reuse that input's result set rather than
issuing a new query." Delivered examples:

| Input | `valueString` | Meaning |
|---|---|---|
| `labResults`, `labResultValues`, `negExemptLabResults`, `eltcLabResults` | `labTests` | filter the `labTests` result set |
| `diagnosticResults`, `diagnosticResultValues` | `diagnosticOrders` | filter the `diagnosticOrders` result set |
| `eltcProblems` | `conditions` | " |
| `eltcDiagnoses` | `encounterDiagnoses` | " |
| `encounters`, `encounterdata`, and all Encounter-typed inputs | `encounter` | the triggering encounter from context |
| `patientdata` | `patient` | the patient from context |

This is a real optimization — it prevents an implementing system from issuing a dozen redundant
Observation queries — but nothing in the IG tells an implementer that a bare token in this extension
means "alias", not "query". A naive implementation would issue `GET [base]/labTests` and fail.

**Proposed change.**
* Correct the extension URL in the profile and the guidance page (subject to the 1.9 decision).
* Replace the example in the guidance page with a real delivered input, including a versioned
  grouper canonical.
* Add an "Input aliasing" subsection documenting the reuse pattern, with the table above.

**Acceptance criteria.** Extension URL is consistent across profile, page, and delivered artifact;
the aliasing convention is documented with examples.

**Risk / impact.** The aliasing documentation is a straightforward win — it removes a real
implementation trap.

---

### 1.12 — Profile and document the alternative-expression extension on `action.condition`

**Type:** Profile change + documentation

**Context.** The IG currently frames FHIRPath and CQL as two *levels of implementation* — "triggering"
(FHIRPath) versus "supplemental" (CQL) — and presents the supplemental level as an opt-in extension
for sites doing richer rules processing. That is not how the delivered artifact works. Every single
condition in the delivered PD carries **both**: a FHIRPath expression as the primary, and a CQL
identifier as an alternative. The two are meant to be equivalent, not tiered.

**Current state.**
* Profile: says nothing about `condition.expression.extension`.
* Page: shows `http://hl7.org/fhir/StructureDefinition/cqf-alternativeExpression` and describes the
  CQL as belonging to the "supplemental level".

**As delivered (v3.2.0).** All 12 conditions carry
`http://hl7.org/fhir/us/ecr/StructureDefinition/us-ph-alternative-expression-extension` with
`language = text/cql-identifier`, an expression name, and
`reference = http://ersd.aimsplatform.org/fhir/Library/RuleFilters|3.2.0`. Delivered expression
names, which are a useful map of the logic:

* `Is Suspected Disorder?`
* `Is Encounter In Progress and Within Normal Reporting Duration or 72h or less after end of encounter?`
* `Is Ambulatory Encounter In Progress and Within Ambulatory Reporting Duration or 72h or less after end of encounter?`
* `Is Encounter Late`, `Is Encounter Complete`
* `Is Encounter Reportable and Within Normal Reporting Duration?`

**Proposed change.**
* Add the extension to the profile on `action.condition.expression`.
* Correct the URL in the page (subject to 1.9).
* Rewrite the framing: FHIRPath is primary and always present; CQL is an equivalent alternative for
  consumers with a CQL engine. Decouple this from the triggering/supplemental distinction, which is
  about *content*, not expression language.
* Cross-reference 4.2 — the referenced `RuleFilters` library is not shipped in the package.

**Acceptance criteria.** Profile constrains the extension; the page no longer implies CQL is only
for supplemental implementers.

**Risk / impact.** The reframing is the substantive part and is worth flagging to reviewers, since
it corrects a conceptual model that has been in the IG for several versions.

---

### 1.13 — Profile the PlanDefinition-level `relatedArtifact`, `effectivePeriod`, and release label

**Type:** Profile change (additive) + documentation

**Context.** Small housekeeping item, but it is what ties the PlanDefinition to the specific RCTC
version it was authored against — which matters for anyone reasoning about what a given eICR was
triggered by.

**As delivered (v3.2.0).**
* `relatedArtifact`: one `depends-on` entry, label "RCTC Value Set Library of Trigger Codes",
  resource `http://ersd.aimsplatform.org/fhir/Library/rctc|3.2.0` — note the **version pin**.
* `effectivePeriod.start`: `2026-12-01`.
* `http://hl7.org/fhir/StructureDefinition/artifact-releaseLabel` (see 3.3).
* `experimental: false`, `publisher`: "Association of Public Health Laboratories (APHL)".

**Proposed change.** Constrain `relatedArtifact` to require the versioned `depends-on` on the RCTC
library; document `effectivePeriod` semantics (when this specification takes effect) and the
version-pinning convention.

**Acceptance criteria.** Profile requires the RCTC dependency; page explains version pinning and
`effectivePeriod`.

**Risk / impact.** Low.

---

# Group 2 — eRSD page: package structure and distribution

**Group context.** This is the "docs are lacking, not just stale" half of the work. The current page
explains the *conceptual* model well but gives an implementer almost nothing about the artifact they
actually receive: what is in the bundle, how to navigate it, how big it is, or how versioning works
inside it. We already have most of this written internally — the
`eRSD_Specification_Bundle_Structure_v2.docx` developer navigation guide in the
`aphl-ersd-specifications-v3` repo — so this is largely a matter of updating it to 3.2.0 and moving
it into the IG.

---

### 2.1 — Add an "eRSD Specification Bundle Structure" section

**Type:** Documentation (new section)
**Source material:** `aphl-ersd-specifications-v3/docs/eRSD_Specification_Bundle_Structure_v2.docx`

**Context.** The single most common implementer question is "what am I looking at, and where do I
start?" The existing internal guide answers it well and should be promoted into the IG. Note it
needs updating before it lands: it documents six grouper value sets and we now ship nine (see 2.2).

**Content to include.**

*The five conceptual layers:*

| Layer | Resource | Canonical URL | Role |
|---|---|---|---|
| 0 | `Bundle` | none (transport wrapper) | `type = collection`; id follows `rctc-release-{version}-Bundle-rctc` |
| 1 | `Library` | `http://ersd.aimsplatform.org/fhir/Library/ersd-specification` | CRMI manifest; canonical entry point |
| 2a | `PlanDefinition` | `http://ersd.aimsplatform.org/fhir/PlanDefinition/us-ecr-specification` | workflow, timing, triggering logic |
| 2b | `Library` | `http://ersd.aimsplatform.org/fhir/Library/rctc` | index of grouper value sets |
| 3 | `ValueSet` (groupers) | `http://ersd.aimsplatform.org/fhir/ValueSet/{code}` | one per information-model concept |
| 4 | `ValueSet` (leaves) | `http://cts.nlm.nih.gov/fhir/ValueSet/2.16.840.1.113762.1.4.1146.*` | the actual trigger codes |

*Navigation rules:*
* Locate the root Library **by canonical URL** and traverse from it. Do **not** rely on Bundle entry
  ordering (this contradicts the current page — see 2.6).
* The root Library and the `rctc` Library are both `Library` with `type = asset-collection`.
  Distinguish them by canonical URL, not by type or position.
* `composed-of` on the root Library = the two direct components. `depends-on` = the flattened
  transitive closure of everything in the package, plus profile canonicals and pinned code system
  versions. Explain what each is useful for: `composed-of` for traversal, `depends-on` for
  membership checks and dependency auditing.
* **Versioned canonical references.** Where a reference carries a version suffix (`|3.2.0`), resolve
  on URL **and** version. Do not fall back to a different version that happens to be present. Where
  no suffix is present, match on URL alone.
* Grouper → leaf resolution happens through `compose.include.valueSet[]`; the trigger codes
  themselves are in `compose.include[].concept[].code` on the leaves, with the code system on the
  enclosing `compose.include[].system`.

*Scale, so implementers can size for it:* v3.2.0 ships **1,591 bundle entries** — 2 Libraries,
1 PlanDefinition, and 1,588 ValueSets.

*Root Library metadata worth calling out:* `status = active`, `approvalDate`,
`effectivePeriod.start = 2026-12-01`, `useContext` of `reporting = triggering` and
`specification-type = program`.

**Acceptance criteria.** A developer can parse the bundle correctly from the IG alone, without the
internal guide. The section includes the layer table, the navigation rules, and the versioned-
reference rule.

**Risk / impact.** Pure addition. Largest single writing effort in the list — budget accordingly.

---

### 2.2 — Document the full current grouper value set list

**Type:** Documentation

**Context.** The IG and the rule-filter guidance both describe **six** trigger categories. We ship
**nine**. The three additions are not cosmetic — they exist specifically to carry the 3.2.0
triggering-optimization behaviour, and without them the behaviour in 2.3 cannot be explained at all.

**As delivered (v3.2.0).**

| Code | Concept | Primary code systems | Role in the workflow |
|---|---|---|---|
| `dxtc` | Diagnosis / problem | ICD-10-CM, SNOMED CT | matched against `conditions`, `encounterDiagnoses`, `encounters.reasonCode` |
| `ostc` | Organism / substance | SNOMED CT | matched against result *values* (`labResultValues`) |
| `lotc` | Lab order test | LOINC | matched against `labOrders`, `labTests`, `diagnosticOrders` |
| `lrtc` | Lab observation result | LOINC, SNOMED CT | matched against `labResults`, `diagnosticResults` |
| `mrtc` | Medication | RxNorm | matched against request / administration / statement |
| `sdtc` | Suspected disorder | ICD-10-CM, SNOMED CT | drives the immediate-reporting check |
| `artc` | **All-results trigger codes** — *new* | — | conditions **exempt** from negative-result filtering (e.g. Gonorrhea, Hepatitis C) |
| `eltc` | **Extended timing threshold** — *new* | — | conditions using the 365-day window instead of 30 days |
| `iztc` | **Immunization triggers** — *new* | CVX | matched against `immunizations` |

**Proposed change.** Replace the six-category list with the table above, describing what each
grouper *does in the workflow* rather than only what it contains. Include the input-to-grouper
mapping from `is-encounter-reportable`, which is the clearest single illustration of how groupers are
consumed.

**Acceptance criteria.** All nine groupers documented with role; the six-category list no longer
appears in the eRSD page (see 4.3 for the rule-filter page).

**Risk / impact.** Documentation-only.

---

### 2.3 — Document the 3.2.0 triggering-optimization behaviours

**Type:** Documentation

**Context.** These shipped in 3.2.0 and are already visible to every implementer — they change which
encounters produce an eICR, which is the most consequential thing the eRSD does. They appear nowhere
in the IG. This is the clearest example of the "spec is out of sync with what we deliver" problem,
and probably the most defensible ticket in the whole list.

**Behaviours to document.**

1. **Negative lab result filtering.** Results coded or reported as negative are excluded from
   triggering. Delivered implementation checks `value.coding.code` and `interpretation.coding.code`
   for SNOMED CT `260385009` ("Negative") and `260415000` ("Not detected"), and string values
   containing "negative" or "not detected" (case-insensitive).
2. **The `artc` carve-out.** Conditions in the `artc` grouper are exempt from (1) — negative results
   remain reportable. Delivered via the `negExemptLabResults` input.
3. **Refuted / entered-in-error filtering.** `Condition.verificationStatus` matching
   `entered-in-error` or `refuted` is excluded.
4. **Evidence timeboxes.** Diagnosis/problem evidence older than `dxTimeboxDuration` (30 days) and
   lab evidence older than `labTimeboxDuration` (30 days) no longer trigger. Conditions in the
   `eltc` grouper use `extendedTimeboxDuration` (365 days) instead, for longer-latency conditions.
5. **Immunization triggering.** `Immunization.vaccineCode` is now checked against `iztc`.
6. **The ambulatory reporting path.** `ambulatoryReportingDuration` (1 day) with class-based routing
   — `AMB`/`VR`/`HH` ambulatory, `IMP`/`EMER`/`OBSENC` inpatient — and the matching
   `is-amb-encounter-in-progress` / `terminate-amb-encounter` actions (see 1.3).

**Proposed change.** New subsection on the eRSD page, "Triggering refinements", covering all six with
the specific codes and durations. Frame in terms of *intent* — these exist to reduce unnecessary
eICR volume and the associated EHR and public health system load.

**Acceptance criteria.** All six behaviours documented with their delivered codes and default
durations; an implementer can predict whether a given encounter triggers.

**Risk / impact.** Documentation-only, but high implementer value. The v3.2.0 release description in
`aphl-ersd-specifications-v3` already has most of this prose and can be adapted.

---

### 2.4 — Document provisional value sets

**Type:** Documentation

**Context.** Introduced in 3.2.0 to let us publish trigger codes for emerging conditions ahead of
formal VSAC review. It changes version-resolution behaviour, so consumers need to know about it.

**As delivered.** Provisional value sets carry the literal string `PROVISIONAL` as their `version`
rather than a date-stamped version. Grouper references to them were updated so they resolve without
special-case processing. The v3 repo currently carries provisional value sets for Ebolavirus and
Hantavirus, plus a provisional LOINC code system.

**Proposed change.** Document the convention, how a consumer should resolve a `PROVISIONAL` version,
and the expectation that provisional sets are replaced by date-versioned ones once VSAC review
completes.

**Acceptance criteria.** The convention and its lifecycle are documented.

**Risk / impact.** Low. Worth confirming with the content team that `PROVISIONAL` is the settled
convention before it goes into a balloted IG.

---

### 2.5 — Document the actual distribution mechanism and package variants **[DECISION NEEDED]**

**Type:** Documentation + scope decision

**Context.** The current page says the IG "is not prescriptive about the absolute mechanisms for
distribution", which is fine as a conformance statement but unhelpful as guidance. In practice there
is exactly one way implementers get the eRSD, and we should describe it. There are also two package
variants in production that the IG has never mentioned.

**As delivered.**
* JSON and XML bundles published to S3 and served through the `ersd.aimsplatform.org` polling API,
  alongside a release-description text file and a ZIP of the RCTC workbook and change log.
* Produced using the CRMI `$release` and `$package` operations against a FHIR server (see 3.4).
* **Cancer variant package** — `PlanDefinition/us-ecr-specification-cancer` and `Library/rctc-cancer`,
  a parallel distribution with its own canonical URLs and grouper set. Structurally identical
  workflow (same eight actions), different terminology. Not mentioned in the IG.
* **Change preview bundles** — generated to preview VSM-authored changes ahead of a release.

**Proposed change.**
* Add a short "How the eRSD is distributed in practice" subsection, clearly marked as informative
  rather than conformance-setting, so it does not conflict with the existing non-prescriptive stance.
* **[DECISION NEEDED]** whether the cancer variant is documented in the IG, mentioned as an example
  of the variant pattern, or left out entirely. Recommendation: document the *pattern* (a program-
  specific specification with its own canonicals) without making the IG responsible for the cancer
  package's content.
* **[DECISION NEEDED]** whether change preview bundles belong in the IG at all. Recommendation: no —
  they are a release-engineering artifact, not part of the specification.

**Acceptance criteria.** Decisions recorded; whatever is agreed is reflected in the page.

**Risk / impact.** The variant question is the one to settle before writing, since it determines
whether the whole section is about "the eRSD" or "eRSD specifications" plural.

---

### 2.6 — Correct the existing "Packaging and Distribution" section

**Type:** Documentation (correction)
**Related to:** 2.1

**Context.** Two statements in the current section are actively misleading — an implementer who
follows them will write code that breaks.

**Correction 1 — entry ordering.** The page states: "the expectation is that the Bundle would
include the Library as the first entry, followed by all the component resources as entries, and
finally all the referenced ValueSet resources." Consumers should resolve by canonical URL; the
shipped bundle should not be assumed to be ordered this way. Replace with the traversal guidance
from 2.1.

**Correction 2 — value set profiles. [DECISION NEEDED]** The page states that RCTC value sets are
distributed conforming to the CRMI `crmi-computablevalueset` and `crmi-expandedvalueset` profiles.
Delivered leaf value sets declare:
* `http://hl7.org/fhir/StructureDefinition/shareablevalueset`
* `http://hl7.org/fhir/us/cqfmeasures/StructureDefinition/computable-valueset-cqfm`
* `http://hl7.org/fhir/us/cqfmeasures/StructureDefinition/publishable-valueset-cqfm`

i.e. the CQF Measures profiles, not the CRMI ones. Options: change the delivered profiles to CRMI
(the stated direction of travel, but an artifact change affecting 1,588 resources), or correct the
documentation to match delivery and note CRMI as the intended future state. Recommendation: correct
the documentation now, track the profile migration separately.

**Acceptance criteria.** Neither misleading statement remains; the value set profile question has a
recorded decision.

**Risk / impact.** Correction 1 is safe and should be done regardless. Correction 2 needs the
decision first.

---

# Group 3 — CRMI alignment and pointers

**Group context.** The eRSD release process moved onto CRMI tooling — the bundle we ship is
literally the output of CRMI `$release` and `$package` operations — but the IG never says so. As a
result, several things an implementer can see in the artifact (the manifest library, the transitive
`depends-on` list, version pinning, expansion parameters, the release label) have no explanation
anywhere. Pointing at CRMI rather than re-documenting it is the cheapest fix and was one of the
explicit goals for this round.

---

### 3.1 — Describe the root Library as a CRMI manifest library

**Type:** Documentation

**Context.** The root Library declares two profiles and the IG only knows about one of them. The
CRMI one is the more useful of the two for an implementer, because "manifest" is what explains why a
single resource enumerates the exact version of every artifact in the release.

**Current state.** The page describes the eRSD specification as "an asset collection library (a
Library resource with a type of `asset-collection`) conforming to the US Public Health Specification
Library profile." True but incomplete.

**As delivered.** `meta.profile` = `[…/us/ecr/StructureDefinition/us-ph-specification-library`,
`http://hl7.org/fhir/uv/crmi/StructureDefinition/crmi-manifestlibrary]`.

**Proposed change.** State both profiles, link to `crmi-manifestlibrary`, and explain in one or two
sentences what the manifest gives the consumer: a single authoritative list of the exact versions
that constitute this release, usable for validation and for reproducing expansions.

**Acceptance criteria.** Both profiles named and linked; the manifest concept explained.

---

### 3.2 — Document the expansion parameters carried on the manifest

**Type:** Documentation

**Context.** This is how a consumer reproduces our value set expansions exactly rather than getting
subtly different results from their own terminology server. It is completely invisible in the IG
today, which means implementers who re-expand are silently at risk of divergence.

**As delivered.** The root Library carries
`http://hl7.org/fhir/StructureDefinition/cqf-expansionParameters` and
`http://hl7.org/fhir/StructureDefinition/cqf-inputParameters`, referencing a contained `Parameters`
resource that sets `activeOnly = false` and pins code system versions, including:
LOINC 2.81 · RxNorm 2026-01 · ICD-10-CM 2026 · CVX 20251119 · SNOMED CT US edition
(`http://snomed.info/sct/731000124108/version/20260301`) · ActCode 9.0.0 · ActMood 3.0.0 ·
NHSN 2023-04.

**Proposed change.** New subsection explaining the extension, the contained `Parameters`, what
`activeOnly = false` means for expansion behaviour, and why a consumer re-expanding value sets should
use these parameters. Link to CRMI's expansion-parameters guidance rather than restating it.

**Acceptance criteria.** An implementer can reproduce our expansions from the IG's instructions.

**Risk / impact.** Documentation-only; high value for anyone running their own terminology server.

---

### 3.3 — Document the `artifact-releaseLabel` extension

**Type:** Documentation

**Context.** Minor, but it appears on three resources and currently has no explanation, so it reads
as unexplained noise.

**As delivered.** `http://hl7.org/fhir/StructureDefinition/artifact-releaseLabel` on the root
Library, the `rctc` Library, and the PlanDefinition. In v3.2.0 its value is a date string
(`2026-06-26`) distinct from the artifact `version` (`3.2.0`).

**Proposed change.** One paragraph: what the release label is, how it relates to `version`, and its
relationship to the release label set in the RCTC workbook during release generation.

**Acceptance criteria.** The extension is explained wherever the bundle structure is described.

---

### 3.4 — Point at the CRMI artifact lifecycle and operations

**Type:** Documentation

**Context.** Explaining that the package is produced by CRMI operations retroactively explains
several otherwise-puzzling features of the artifact, and it is the natural place to satisfy the
"pointers to CRMI where they make sense" goal. Keep it to pointers — this IG should not re-specify
CRMI.

**What to cover.**
* The release is produced with the CRMI `$draft` → `$release` → `$package` operation sequence against
  a FHIR server.
* `$release` is what produces the versioned `relatedArtifact` entries and the flattened transitive
  `depends-on` list.
* `approvalDate` (and `lastReviewDate` where present) are prerequisites for `$release`.
* The `-draft` version suffix and `status = draft` appear during authoring; released artifacts are
  `status = active` with a clean version.
* Link CRMI's artifact versioning / version-policy guidance to explain the `|3.2.0` pinning
  convention used throughout the package.

**Proposed change.** Short informative subsection with links, not a restatement of CRMI.

**Acceptance criteria.** The lifecycle is described in outline with working links to CRMI; nothing in
the section duplicates normative CRMI content.

**Risk / impact.** Low. Confirm the CRMI version we cite matches the IG dependency (currently
`hl7.fhir.uv.crmi 1.0.0` in `sushi-config.yaml`, while the v3 artifacts reference CRMI profile
canonicals — worth checking these agree).

---

# Group 4 — Supplemental specification and rule filters

---

### 4.1 — The supplemental specification as documented is not currently distributed **[DECISION NEEDED]**

**Type:** Scope decision, then documentation

**Context.** This is the most significant honesty gap in the eRSD section. The IG devotes
substantial space — a profile, several page sections, and two dedicated implementation-guidance
pages — to a Supplemental eRSD specification. A reader would reasonably conclude it is something they
can obtain and implement. It is not currently part of what we ship.

**What the IG documents.** The `ersd-supplemental-library` profile; a supplemental library composed
of a CQL rule-filters library, a supplemental value set library, and the jurisdictions CodeSystem;
plus `ersd_jurisdictions_codesystem_query.md` and
`ersd_jurisdictions_codesystem_description.md`.

**What v3.2.0 actually contains.** Two Libraries (`ersd-specification`, `rctc`), one PlanDefinition,
1,588 ValueSets. No supplemental library, no CQL library, no supplemental value set library, no
jurisdictions CodeSystem.

**Options.**
* **(a)** Mark the supplemental content as *not currently distributed / future direction*, with a
  status note at the top of each affected page and section.
* **(b)** Keep the content as-is but add a single prominent note in the eRSD page.
* **(c)** Remove the supplemental content from the IG.

**Recommendation.** (a). It is honest, it is cheap, it preserves the design work for when supplemental
distribution resumes, and it avoids the larger conversation that (c) would open.

**Acceptance criteria.** A decision is recorded; every affected page carries a consistent status
statement.

**Risk / impact.** Low technically, but this is the item most likely to prompt discussion about
programme direction. Worth raising early and separately.

---

### 4.2 — Note that the referenced RuleFilters library is not shipped in the package

**Type:** Documentation
**Related to:** 1.12, 4.1

**Context.** Every condition in the delivered PD points at a CQL library that is not in the bundle.
An implementer with a CQL engine who tries to follow the alternative-expression reference will not
find it, with no explanation of why.

**As delivered.** All 12 conditions reference
`http://ersd.aimsplatform.org/fhir/Library/RuleFilters|3.2.0` — a versioned canonical that resolves
to nothing inside the distributed bundle. (The CQL source exists in the v3 repo as
`input/cql/RuleFilters.cql`, but it is not packaged.)

**Proposed change.** State plainly either where a consumer obtains the library, or that the CQL
alternative expressions are informative for v3 and the FHIRPath primary expression is the normative
one. Recommendation: the latter, consistent with 4.1.

**Acceptance criteria.** No dangling reference is left unexplained in the IG.

**Risk / impact.** Low, and removes a support question we will otherwise keep receiving.

---

### 4.3 — Update `rule_filter_generation.md`

**Type:** Documentation

**Context.** The page is built entirely around the six original trigger categories and `*-example`
value set canonicals, and describes a CQL-based rule generation approach tied to the supplemental
specification. Its opening CQL block declares exactly the six `valueset "Example …"` definitions.
With three new groupers and the supplemental status question open, it needs at minimum a consistency
pass.

**Proposed change.**
* Update the category list to the current nine groupers (see 2.2).
* Reconcile the example canonicals with the delivered `ersd.aimsplatform.org` groupers, or mark them
  clearly as illustrative.
* Apply whatever status statement comes out of 4.1.

**Acceptance criteria.** No reference to a six-category model remains; the page's status relative to
4.1 is clear.

**Risk / impact.** Scope depends on the 4.1 outcome — sequence this after that decision.

---

### 4.4 — Direction of travel toward CQL-primary expressions **[LIKELY OUT OF PHASE]**

**Type:** Optional documentation / future direction

**Context.** Internal analysis from Phase 1 integration testing recommends CQL become the *primary*
expression language for eRSD reportability logic, with FHIRPath retained for simple navigation and as
an alternative for consumers without a CQL engine. The motivation is concrete: FHIRPath has no named
bindings, which forces the delivered logic into single expressions of 4,195 and 4,068 characters
(`is-encounter-reportable` and `is-modified-encounter-reportable` respectively) that are effectively
untestable and un-reviewable, and which have already produced at least one class of engine
miscompilation.

**Proposed change.** If we want any hook for this in the current ballot, the cheapest version is a
one-paragraph "future direction" note. Full migration is clearly a later phase and should not be
proposed now.

**Recommendation.** Raise as a discussion item, not a ticket, unless there is appetite for signalling
direction in this ballot. Flagged here so the omission is a decision rather than an oversight.

**Risk / impact.** Raising the full migration in this phase would be a red flag. The one-paragraph
note would not.

---

# Group 5 — Examples

**Group context.** The examples are what implementers copy from, so stale examples propagate stale
implementations. All of the eRSD examples predate the v3 workflow.

---

### 5.1 — Regenerate the eRSD PlanDefinition example from what we actually ship

**Type:** Example update
**Depends on:** Group 1 (profile must be corrected first, or the example cannot validate)

**Context.** `plandefinition-ersd-instance-example` is the example an implementer reads first, and it
describes a workflow we stopped delivering some releases ago.

**Current state.** Carries `date = "2020-07-31"`, `effectivePeriod.start = "2020-12-01"`, the old
action ids (`check-suspected-disorder`, `is-encounter-suspected-disorder`), the `*-example` value set
canonicals, the `cqf-fhirQueryPattern` extension URL, commented-out condition blocks, and none of the
3.2.0 behaviour.

**Proposed change.** Regenerate from the delivered `us-ecr-specification` PlanDefinition, substituting
IG-appropriate example canonicals and dates, and retaining the full delivered action structure so the
example actually demonstrates the workflow described in the page.

**Acceptance criteria.** The example validates against the corrected profile and structurally matches
the delivered PD (same eight top-level actions, same child structure).

**Risk / impact.** Must land after Group 1. Consider whether the example should use the real
`ersd.aimsplatform.org` canonicals or IG example canonicals — the latter is conventional, but the
former is more useful; worth a quick decision.

---

### 5.2 — Consolidate the four near-duplicate PlanDefinition examples

**Type:** Example cleanup
**Depends on:** 5.1

**Context.** Four examples exist, three of them ~300 lines and largely identical, differing in ways
that are not obvious from their names. Maintaining four copies of a workflow that is already drifting
is the reason they all drifted.

**Current state.**
* `plandefinition-ersd-instance-example` (365 lines)
* `plandefinition-ersd-instance-simple-example` (304 lines)
* `plandefinition-ersd-instance-relateddata-extension-example` (304 lines)
* `plandefinition-ersd-instance-namedEvent-example` (31 lines)

**Proposed change.** Reduce to one complete example (5.1) plus small focused fragments that
demonstrate the specific feature each variant exists to show. Retire the rest, updating
`artifact-overview.md` and any inbound links.

**Acceptance criteria.** One canonical full example; each retained variant has a stated, distinct
purpose; no broken links.

**Risk / impact.** Removing published example ids may need a deprecation note rather than deletion —
check IG conventions before removing.

---

### 5.3 — Update the specification Library and Bundle examples

**Type:** Example update
**Depends on:** Groups 2 and 3

**Context.** These examples are supposed to illustrate the package structure documented in 2.1, and
currently illustrate a much simpler structure than the one we ship.

**Current state.**
* `library-ersd-specification-library-example` declares only
  `USPublicHealthSpecificationLibrary` — no CRMI manifest profile, no expansion parameters, no
  release label, and two unversioned `composed-of` entries.
* `bundle-ersd-specification-example` does not reflect the five-layer structure.

**Proposed change.** Update both to demonstrate the delivered pattern: dual profile declaration,
`cqf-expansionParameters` with a contained `Parameters`, `artifact-releaseLabel`, versioned
`composed-of` and `depends-on` entries, and a bundle showing all five layers (a representative
subset of leaf value sets is fine — the point is the structure, not the volume).

**Acceptance criteria.** The examples match the structure documented in 2.1 and 3.1–3.3.

**Risk / impact.** Sequence after Groups 2 and 3 so the examples and the prose land together.

---

# Group 6 — Small cleanups

**Group context.** Cheap, low-risk, independently landable. Good candidates for bundling into a
single housekeeping ticket if the process prefers fewer tickets.

---

### 6.1 — Fix the navigation label and acronym

**Type:** Documentation (trivial)
**File:** `sushi-config.yaml` lines 79 and 116

**Current state.** "Electronic Reporting and Surveillance **Data (eERSD)** Transaction and Profiles"
— appears twice, in the `menu` block and the `pages` block. Both the expansion and the acronym are
wrong; the page body itself correctly says "electronic Reporting and Surveillance Distribution
(eRSD)".

**Proposed change.** "electronic Reporting and Surveillance Distribution (eRSD) Transaction and
Profiles" in both places. Check `acronyms_and_abbreviations.md` for the same error while in there.

**Acceptance criteria.** The nav label matches the page title and the acronym is correct everywhere.

---

### 6.2 — Resolve the empty `specification.md` parent page

**Type:** Documentation

**Current state.** `input/pagecontent/specification.md` is a zero-byte file serving as the parent
page for the Specification menu section.

**Proposed change.** Either give it brief introductory content, or confirm the empty parent is
intentional (some IGs use empty parents purely as menu containers) and note that so it is not
repeatedly re-reported.

---

### 6.3 — Add the supplemental library to the eRSD page's Profiles list

**Type:** Documentation
**Related to:** 4.1

**Current state.** The `#### Profiles` list at the foot of the eRSD page includes
`ersd-plandefinition` but omits `ersd-supplemental-library`, which this IG defines and which
`artifact-overview.md` links.

**Proposed change.** Add it — or, if 4.1 lands as option (c), remove the profile and the stray links
consistently. Sequence after 4.1.

---

### 6.4 — Mark illustrative canonicals as illustrative

**Type:** Documentation

**Context.** The eRSD pages mix `hl7.org/fhir/us/ecr/…-example` canonicals into prose examples
without saying they are examples, while the real package uses `ersd.aimsplatform.org` and
`cts.nlm.nih.gov` canonicals. Readers have mistaken the example URLs for real ones.

**Proposed change.** Sweep the eRSD pages; either label example canonicals explicitly as
illustrative, or replace them with the real delivered canonicals where doing so is clearer. The
existing note about RCTC value sets in the IG being examples is a good model to follow consistently.

---

# Group 7 — Page presentation and readability

**Group context.** Separate from the accuracy problems above, the eRSD page is simply unpleasant to
read. It is 293 lines and 13 headings deep with no in-page navigation, its code samples are
hand-escaped HTML rather than fenced blocks, its link lists are raw `<ul>` markup, its images have no
alt text or captions, and its central Process section states the same workflow twice in two different
formats that have already drifted apart. None of this is anyone's fault — the page has accreted over
several versions — but it makes the content harder to use than it needs to be, and we are about to
add a significant amount of new material to it (Groups 2 and 3). Doing the structural cleanup first
means the new content lands in a page that can carry it.

**Deliberate constraint.** Everything below works *within* the HL7 IG template — structure, semantic
markup, tables, figures, alt text. No custom CSS, no branding, no template overrides. Publisher-level
styling changes would be both fragile across template updates and exactly the kind of thing that
attracts scrutiny at ballot. The goal is a well-structured page, not a restyled one.

**Sequencing.** 7.1, 7.2, and 7.5 are safe mechanical cleanups that can land immediately and
independently. 7.3, 7.4, and 7.6 should follow Group 1, since the diagrams and the Process narrative
have to be re-cut against the corrected workflow anyway — doing them twice would be wasted effort.

---

### 7.1 — Replace hand-escaped code samples with fenced code blocks

**Type:** Presentation (mechanical)
**File:** `input/pagecontent/ersd_transaction_and_profiles.md`

**Context.** The page's six code samples are written as `<pre><code>` blocks with every angle bracket
and quote hand-escaped as `&lt;` / `&gt;` / `&quot;`. They render as undifferentiated monospace with
no syntax highlighting, they are painful to edit (every change needs re-escaping, which is how errors
creep in), and they are inconsistent with the rest of this IG — `rule_filter_generation.md` and
`ersd_jurisdictions_codesystem_query.md` already use fenced blocks with language tags.

**Current state.** Six hand-escaped blocks, at lines 69, 90, 190, 204, 232, and 250.

**Proposed change.** Convert all six to fenced blocks with an explicit language tag (```` ```xml ````),
which the publisher's highlighter picks up. Consider converting the samples to JSON while in there —
the delivered artifact is distributed as JSON and XML, but implementers overwhelmingly work with the
JSON, and every sample on the page is currently XML only.

**Acceptance criteria.** No `&lt;`-escaped code remains on the page; all samples are syntax
highlighted; language tags are consistent with the other eRSD pages.

**Risk / impact.** None. Verify rendering once in the built output, since fenced blocks inside
markdown pagecontent occasionally interact badly with Jekyll templating if a sample contains `{{`.

---

### 7.2 — Replace raw HTML link lists with markdown

**Type:** Presentation (mechanical)

**Context.** The Profiles and Extensions lists at the foot of the page are hand-written
`<ul>` / `<li>` / `<a>` blocks. They bypass the template's list styling, so they sit inconsistently
against the markdown lists elsewhere on the page, and they carry stray trailing whitespace.

**Current state.** Raw `<ul>` blocks at lines 278 and 291; trailing whitespace on lines 282–283.

**Proposed change.** Convert to markdown lists using the existing `{{site.data.fhir.ver.*}}` link
variables. Strip trailing whitespace. While in there, add the missing supplemental library entry
(see 6.3).

**Acceptance criteria.** No raw list markup remains; lists render consistently with the rest of the
page.

**Risk / impact.** None.

---

### 7.3 — Give images alt text, captions, and sensible sizing

**Type:** Presentation + accessibility

**Context.** This one matters beyond aesthetics. All three images on the page are
`<img style="width:100%" src="…"/>` with **no `alt` attribute at all**. For a public health
specification, that is an accessibility gap worth closing on its own merits — screen reader users get
nothing, and the diagrams carry information that appears nowhere else in the text. Forcing
`width:100%` also upscales diagrams past their native resolution on wide viewports, which is a large
part of why the page looks soft and dated.

**Current state.**
* Line 7 — `ersd-transaction-system-overview.png`
* Line 47 — `eicr-triggering-and-transmission-guidance-components.png`
* Line 59 — `ersd-plandefinition-structure.png`

None has `alt`. All force `width:100%`. None has a caption or figure number, so the surrounding prose
cannot refer to them.

**Worth noting for consistency:** the eICR page wraps its images in `<table><tr><td>` and passes a
`caption="…"` attribute on the `<img>` — but `caption` is not a valid HTML attribute and renders
nothing. So neither page actually produces captions today. Whatever convention we adopt should be
applied to both.

**Proposed change.**
* Add descriptive `alt` text to every image.
* Adopt a real figure convention — `<figure>` / `<figcaption>`, or the template's own figure
  handling if it has one — with numbered captions ("Figure 1: …") that the prose can reference.
* Replace `width:100%` with `max-width:100%` so diagrams display at native size and scale down only
  when the viewport requires it.

**Acceptance criteria.** Every image has meaningful alt text; figures are numbered and captioned;
no image is upscaled beyond its native resolution.

**Risk / impact.** Low, and the accessibility improvement is independently defensible. Apply the same
convention to the eICR and RR pages for consistency, ideally as a follow-up ticket rather than
expanding this one.

---

### 7.4 — Re-cut the diagrams, and use the SVG that already exists

**Type:** Presentation (asset work)
**Depends on:** Group 1

**Context.** Two problems. First, the diagrams are stale in the same way the prose is — they depict
the old action names and the pre-v3 workflow, so they will need re-cutting once Group 1 lands
regardless. Second, they are PNGs of line-art diagrams, which is the format that blurs worst on
high-DPI displays; this is the single largest contributor to the page looking dated.

**Discovery worth flagging.** `input/images/ersd-processing.drawio.svg` is present in the repo and
**referenced by nothing**. It is a draw.io source SVG, meaning it is both vector (sharp at any size)
and editable. Someone did this work already. Worth reviewing whether it supersedes one of the PNGs or
can be the basis for the re-cut.

**Proposed change.**
* Review `ersd-processing.drawio.svg` and decide whether to adopt, adapt, or retire it.
* Re-cut the workflow diagram against the corrected eight-action structure (Appendix A), in SVG.
* Keep the draw.io source in the repo alongside the exported asset so the next person can edit rather
  than recreate.

**Acceptance criteria.** Diagrams match the delivered workflow; vector format where the source allows;
no orphaned image assets left in `input/images/`.

**Risk / impact.** Requires diagram authoring effort — the largest non-writing task in this group.
Sequence after Group 1 so it is done once.

---

### 7.5 — Replace `<u>` underlines with proper emphasis, and table the Parameters section

**Type:** Presentation
**Related to:** 1.8

**Context.** The Parameters section marks its example durations with `<u>` tags — `<u>1 hour</u>`,
`<u>72 hours</u>`, and so on, six occurrences. Underlined text reads as a hyperlink to essentially
every reader, so the section is visually noisy and mildly misleading. Underline for emphasis has been
discouraged since HTML 4.

**Current state.** Six `<u>` tags across the Parameter A–E examples (lines 160–176). The section
itself is five paragraphs of near-identical structure — parameter name, definition, worked example —
which is exactly the shape that wants to be a table.

**Proposed change.**
* Replace `<u>` with bold or plain text; the durations are already the subject of their sentences and
  do not need typographic marking at all.
* Restructure A–E as a table. This dovetails with 1.8, which has to rewrite this section anyway to
  introduce the real variable names — do them together, once.

**Acceptance criteria.** No `<u>` tags remain on the page; the parameters are presented as a scannable
table rather than five parallel paragraphs.

**Risk / impact.** None. Merge with 1.8 to avoid editing the same section twice.

---

### 7.6 — Restructure the Process section, which currently says everything twice

**Type:** Presentation + content
**Depends on:** Group 1

**Context.** The Process section is the heart of the page, and it is the hardest part to read. It
presents the workflow as a nested bullet outline, and then immediately restates the identical
information as eight consecutive paragraphs, each beginning "The `x` action …". The reader has to
work out that the second half adds nothing to the first.

The duplication has also already caused a factual divergence: the outline says `start-workflow`
triggers "check-suspected-disorder in 'A' hours", while the paragraph two screens later says it
"specifies that `check-reportable` should be called in 'A' hours". They cannot both be right, and
neither matches what we ship (`check-for-immediate-reporting`). This is a good illustration of why
the duplication is a maintenance problem and not only a presentation one.

**Proposed change.**
* Collapse to a single representation: one table of actions (id · code · trigger or condition · next
  action · delay), plus the re-cut diagram from 7.4.
* Keep prose only for the things a table cannot carry — the rationale for the loop structure, the
  ambulatory/inpatient split, the termination conditions.
* Fold in the corrected workflow from Group 1 while restructuring.

**Acceptance criteria.** Each action is described exactly once; the outline/prose divergence is gone;
the section is navigable without scrolling back and forth.

**Risk / impact.** This is a content rewrite as much as a presentation one — pair it with the Group 1
narrative updates rather than treating it as standalone.

---

### 7.7 — Add in-page navigation and flatten the heading hierarchy

**Type:** Presentation

**Context.** The page is 293 lines with 13 headings running four levels deep (`###` through
`######`), and no table of contents. A reader who wants the Parameters section has to scroll and
scan for it. The deepest headings also render at or below body-text size in the HL7 template, so
`###### Triggering eRSD Specification` — a significant section — is visually indistinguishable from
the paragraph beneath it. Once Groups 2 and 3 add the bundle structure and CRMI sections, the page
gets substantially longer and this gets worse.

**Current state.** Heading levels `###` (1) → `####` (5) → `#####` (5) → `######` (2), no TOC.

**Proposed change.**
* Add a table of contents at the top. Check whether the IG template offers one before hand-rolling
  it — several HL7 templates do.
* Flatten to three levels by promoting the two `######` headings and regrouping. "Triggering eRSD
  Specification" and "Supplemental eRSD Specification" are substantial sections and should not be
  the smallest headings on the page.
* **[DECISION NEEDED]** whether the page should be split. At its post-Group-2/3 length it is a
  candidate for splitting the package-structure and distribution material onto its own page under
  Implementation Guidance, leaving this page focused on the transaction and profiles its title
  promises. Recommendation: decide before writing 2.1, since it determines where that content lands.

**Acceptance criteria.** The page has working in-page navigation; no heading is deeper than three
levels; the split decision is recorded.

**Risk / impact.** The split decision affects Group 2 sequencing — settle it early.

---

### 7.8 — Establish a light style convention for the eRSD pages

**Type:** Documentation (process)

**Context.** Every item in this group is a case of the same underlying issue: there is no stated
convention, so each edit over the years picked its own approach. Writing down a handful of rules
costs almost nothing and is what stops the page drifting back.

**Proposed change.** A short conventions note — in the repo, in `CONTRIBUTING`, or as a comment block
at the top of the page source — covering: fenced code blocks with language tags; markdown rather
than raw HTML for lists and links; figures always with alt text and numbered captions; reference
material as tables rather than parallel prose; blockquote callouts formatted consistently
(the page currently has three "Note to implementers" blockquotes in slightly different styles);
maximum heading depth.

**Acceptance criteria.** A conventions note exists and is discoverable by the next person editing
these pages.

**Risk / impact.** None. Cheapest item in the list and the one most likely to prevent a repeat of
this whole exercise.

---

# Appendix A — Delivered v3.2.0 PlanDefinition action tree

For reference when writing the tickets. Codes shown are the `action.code` values; arrows are
`relatedAction` with `relationship = before-start`.

```
start-workflow                       initiate-reporting-workflow   trigger: encounter-start
  -> check-for-immediate-reporting (offset 1h)
check-for-immediate-reporting        execute-reporting-workflow
  is-encounter-immediately-reportable  check-trigger-codes        -> create-eicr
  continue-check-reportable            evaluate-condition          -> check-reportable
  terminate-late-encounter             terminate-reporting-workflow
  is-late-encounter-completed          complete-reporting
check-reportable                     execute-reporting-workflow
  is-encounter-reportable              check-trigger-codes         -> create-eicr
  check-update-eicr                    evaluate-condition          -> create-eicr
  is-encounter-in-progress             evaluate-condition          -> check-reportable
  is-amb-encounter-in-progress         evaluate-condition          -> check-reportable
  terminate-encounter                  terminate-reporting-workflow
  terminate-amb-encounter              terminate-reporting-workflow
  is-encounter-completed               complete-reporting
create-eicr                          create-report               -> validate-eicr
validate-eicr                        validate-report             -> route-and-send-eicr
route-and-send-eicr                  submit-report
encounter-modified                   initiate-reporting-workflow  trigger: encounter-modified
  -> is-modified-encounter-reportable
is-modified-encounter-reportable     check-trigger-codes          -> create-eicr
```

**Condition expression sizes** (characters of FHIRPath), useful context for 4.4:

| Action | Size |
|---|---|
| `is-encounter-reportable` | 4,195 |
| `is-modified-encounter-reportable` | 4,068 |
| `is-encounter-immediately-reportable` | 745 |
| `is-late-encounter-completed` | 429 |
| `is-amb-encounter-in-progress` | 324 |
| `is-encounter-in-progress` | 321 |
| `terminate-amb-encounter` | 312 |
| `terminate-encounter` | 309 |
| `continue-check-reportable` | 237 |
| `terminate-late-encounter` | 222 |
| `is-encounter-completed` | 47 |
| `check-update-eicr` | 44 |

# Appendix B — Profile vs. delivered, at a glance

| Profile slice / constraint | Profile says | We deliver | Item |
|---|---|---|---|
| `action` (top level) | `7..` | 8 actions | 1.5 |
| `checkSuspectedDisorder.id` | `check-suspected-disorder` | `check-for-immediate-reporting` | 1.1 |
| `encounterStart.relatedAction.actionId` | `check-suspected-disorder` | `check-for-immediate-reporting` | 1.1 |
| `isEncounterSuspectedDisorder.id` | `is-encounter-suspected-disorder` | `is-encounter-immediately-reportable` | 1.1 |
| `checkSuspectedDisorder.action` | `2..`, 2 slices | 4 children | 1.2 |
| `checkReportable.action` | `4..`, 4 slices | 7 children | 1.3 |
| `encounterModified.relatedAction.actionId` | `create-eicr` | `is-modified-encounter-reportable` | 1.4 |
| modified-encounter check | nested child action | top-level sibling action | 1.4 |
| action code system | `…/us/ph-library/CodeSystem/us-ph-codesystem-plandefinition-actions` | `…/us/ecr/CodeSystem/us-ph-plandefinition-actions` | 1.9 |
| named-event code system | `…/us/ph-library/CodeSystem/us-ph-codesystem-triggerdefinition-namedevents` | `…/us/ecr/CodeSystem/us-ph-triggerdefinition-namedevents` | 1.9 |
| named-event-type extension | `…/us/ph-library/StructureDefinition/us-ph-named-eventtype-extension` | `…/us/ecr/StructureDefinition/us-ph-named-eventtype-extension` | 1.9 |
| action codes in bound VS | 9 codes assumed present | only 4 confirmed in eCR 2.1.1 | 1.10 |
| FHIR query pattern extension | `…/StructureDefinition/cqf-fhirQueryPattern` | `…/us/ecr/StructureDefinition/us-ph-fhirquerypattern-extension` | 1.11 |
| input query aliasing | not documented | bare input ids used as aliases | 1.11 |
| alternative expression extension | not profiled; page shows `cqf-alternativeExpression` | `…/us/ecr/StructureDefinition/us-ph-alternative-expression-extension` | 1.12 |
| `action.output` | not profiled | 3 outputs, profiled to `eicr-document-bundle` | 1.6 |
| `create-eicr.input` | not profiled, not documented | 29 data requirements | 1.7 |
| PD `variable` extensions | `0..*`, unsliced, undocumented | 9 named variables + `1 day *` coercion | 1.8 |
| Parameters C and D | presented as tunable | hardcoded `72 hours`, 19 occurrences | 1.8 |
| grouper value sets | 6 categories | 9 groupers (`+artc`, `+eltc`, `+iztc`) | 2.2 |
| leaf ValueSet profiles | CRMI computable / expanded | `shareablevalueset` + cqfm computable / publishable | 2.6 |
| root Library | US PH Specification Library only | also `crmi-manifestlibrary` + expansion params + release label | 3.1–3.3 |
| supplemental specification | fully documented | not in the distributed package | 4.1 |
| `RuleFilters` CQL library | referenced by every condition | not in the distributed package | 4.2 |
