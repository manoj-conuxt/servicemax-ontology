# Prompt: Write the requirements for the ServiceMax Revisit Ontology project

You are working in `servicemax-ontology/`. Your task is to **write the project requirements** as Markdown documents in a new `requirements/` folder.

You are **not** building the project. Do not write pipeline code, ontology/schema files or graph files. Your output is documentation only. Every factual claim in it must be backed by evidence you verified against the org with read-only queries.

---

## 0. Hard constraints

1. **Read-only against Salesforce.** Use only `sf sobject describe`, `sf sobject list`, `sf data query` (SOQL) and REST GET calls. Never create, update, delete, import, deploy or run Apex.
2. **Always pass `-o svmx-failure-onthology`** (Org Id 00Diw000004BagzEAC). The CLI default org is a different sandbox; never rely on it.
3. **Do not design the ontology.** The requirements must say *what* the ontology has to represent and answer, written as competency questions and required concepts. They must not lay out a final class/property schema. Any concept list must be labelled **"Candidate, for review"**.
4. **Do not invent facts.** Re-verify every number below before writing it down.
   - If a number differs, record the new value and the difference in the decisions log. Do not quietly overwrite it.
   - If you can't verify something, mark it `UNVERIFIED`.
5. **Do not invent tool capabilities.** Where a requirement depends on Semantica, phrase it as a capability need, e.g. "must export the graph in a standard format". Do not name specific Semantica APIs.
6. **Never print or store secrets** (tokens, auth URLs).
7. Platform is Windows. Run `sf` from PowerShell. Put scratch query scripts in a temporary folder, not in `requirements/`.

---

## 1. Project background (give this to the requirements as input)

**Goal.** Build an ontology and knowledge graph, using **Semantica**, over ServiceMax Core data. It must identify technician **revisits**:
- revisits that carry the explicit link to the earlier visit, **and**
- revisits that have no link,

and explain them through failure, asset, technician and fix context.

**Source org.** ServiceMax Core (`SVMXC__` namespace), API v68.0. The data load is complete.

**Baseline facts.** These were verified on 2026-11-18. Re-verify each one; the SOQL used becomes the proof in `requirements/evidence/`.

| Fact | Value |
|---|---|
| Work Orders (`SVMXC__Service_Order__c`) | 5,210 total; 5,000 in scope (symptom code starts `TC-`/`QP-`/`DD-`/`CH-`/`GEN-`); 210 legacy, excluded |
| In-scope order types | Field Service 4,250; Preventive Maintenance 550; Installation 200 |
| WO Lines / Cases / Installed Products | 9,442 (Labor 4,842, Parts 4,530) / 3,379 / 938 |
| Sites / Accounts / Products / Technicians | 277 / 88 / 95 / 140 |
| Explicit link field | `SVMXC__Related_Work_Order__c` (label "Primary Work Order"), self-lookup on WO |
| Explicit revisits | 701 WOs linked to 701 distinct earlier WOs; chains: 563 × 2 visits, 69 × 3 visits |
| Linked-pair agreement | same Case 701; same Installed Product 679; same root cause 548; same symptom 506; same technician 267 |
| Linked-pair gap | 2–23 days (median 9), from earlier visit completed to later visit scheduled |
| Unlinked, same Case + same asset (≤30 days) | 215. All 215 earlier WOs are `Follow-Up WO Required` (161 Unresolved / 54 Resolved); they are exactly the follow-up-flagged WOs with no linked child |
| Unlinked, same asset, different Case (≤30 days) | 335. Evidence: same root cause 174, same subsystem 197, same symptom 140, prior Unresolved 145, prior unknown cause / no-details symptom 65; at least one of (same symptom, same root cause, prior unresolved) 207 |
| Status unreliable | 357 WOs that got a revisit are `Closed` (251 of them `Unresolved`); `Follow-Up WO Required` appears on 559 WOs |
| Failure taxonomy | `SVMXC__Symptom__c`, `SVMXC__Root_Cause__c` (`<FAMILY>-<SUBSYSTEM>-<COMPONENT>-<MODE>: Subsystem / Component / Failure mode`) and `SVMXC__How_Fixed__c` are structured picklists. Ignore `Sample Value1`/`Sample Value2` |

**Decisions already made by the user.** Record these in the decisions log as **Accepted**.

| ID | Decision |
|---|---|
| D1 | Revisit window = **30 days**, from earlier visit `SVMXC__Completed_Date_Time__c` to later visit `SVMXC__Scheduled_Date_Time__c` |
| D2 | Revisit candidates on the same asset but under a **different Case** are kept as a **scored "suspected" tier**, separate from confirmed revisits. Default KPIs count confirmed revisits only |
| D3 | The 210 legacy demo WOs (non-coded symptoms) are **excluded** |
| D4 | Data load is complete; the snapshot above is the baseline |
| D5 | The project must never write to the org |
| D6 | The ontology is designed only after the requirements are reviewed and approved |

---

## 2. What to produce in `requirements/`

| File | Contents |
|---|---|
| `README.md` | Purpose of the folder, document index, status (Draft), how to review, glossary (revisit, confirmed / suspected, corrective visit, window, asset = Installed Product, etc.) |
| `01-business-requirements.md` | Problem statement, objectives, stakeholders and users (service ops, quality / reliability engineering, technician managers), in-scope / out-of-scope, success measures. IDs: `BR-xxx` |
| `02-data-requirements.md` | Source objects and required fields (API name, label, type, fill rate, purpose), scope filter, relationships between objects, data quality findings, and how the failure codes are parsed. IDs: `DR-xxx` |
| `03-functional-requirements.md` | Extraction; scope filtering; revisit detection rules (explicit link, same-Case inference, suspected tier) and the order they apply in; evidence attributes each revisit must carry; scoring for the suspected tier, with weights and thresholds marked as configurable and to be confirmed; chains of revisits; reporting. IDs: `FR-xxx` |
| `04-ontology-requirements.md` | Competency questions the ontology / graph must answer (see section 3), required concepts and relationships as **Candidate, for review**, how confirmed vs suspected revisits and their evidence must be representable, what the failure hierarchy must express. IDs: `OR-xxx` |
| `05-non-functional-requirements.md` | Read-only access, secrets handling, reproducibility (snapshot + manifest), configurability (window, weights), traceability back to source record Ids, performance at the current data volume, Windows / PowerShell environment, data kept out of git. IDs: `NFR-xxx` |
| `06-solution-structure.md` | Proposed project folder structure and how the pipeline layers separate (extract → transform → infer → ontology / graph → queries → tests), and where Semantica fits. Describe responsibilities, not code |
| `07-acceptance-criteria.md` | Testable criteria tied to the baseline numbers (e.g. "explicit rule yields 701 links"), validity checks on the graph (each revisit has one earlier and one later visit, earlier comes before later, no pair counted twice), and a no-write audit. IDs: `AC-xxx`, each linked to the requirement IDs it verifies |
| `08-decisions-and-open-questions.md` | Decisions log (D1–D6 plus any you add as **Proposed**), open questions for the user (e.g. final scoring weights, whether Preventive Maintenance / Installation visits can start a revisit, how to treat the 22 explicit links whose two visits are on different assets) |
| `09-traceability-matrix.md` | Table: requirement ID → source evidence → acceptance criterion |
| `evidence/proof-queries.md` | Every SOQL query you ran to verify a fact, the date run, and the result (counts / aggregates only, no customer personal data) |

**Writing rules.**
- Every requirement has: ID, statement ("The system shall …"), MoSCoW priority, rationale, source (evidence or decision ID), and acceptance criterion ID.
- Requirements must be atomic and testable. No vague words such as "fast", "user-friendly" or "robust" without a measure.
- Use exact Salesforce API names in backticks.
- Use counts and aggregates only. Do not copy record-level personal data (contact names, emails, phone numbers) into the documents. WO numbers are fine as examples.

---

## 3. Competency questions to include as a minimum (in `04`)
1. Which revisits exist, by tier (confirmed: explicit / same-Case; suspected: high / medium / low), and what evidence supports each?
2. Which revisits had no link, and on which assets, customers, sites and technicians?
3. What is the revisit rate by instrument family, root cause, subsystem and fix action?
4. Which repeat-failure chains have 3 or more visits, and how does the root cause change along each chain?
5. Which visits were closed as Resolved but still led to a revisit?
6. Which replaced parts were followed by a revisit with the same root cause?

Add further questions you find in the data, and mark them **Proposed**.

---

## 4. Steps
1. Re-verify every baseline fact (section 1) with read-only queries and save them to `evidence/proof-queries.md`.
2. Run `sf sobject describe` on each source object to confirm field API names, types and picklist values.
3. Write the documents in section 2, in order.
4. Cross-check: every requirement appears in the traceability matrix, and every acceptance criterion maps to at least one requirement.
5. **Stop and report:**
   - files created
   - any baseline differences
   - open questions that need the user's decision
   - the full list of `sf` commands run, to confirm none were writes

Do not start building the ontology or pipeline. Wait for the user to review the requirements.