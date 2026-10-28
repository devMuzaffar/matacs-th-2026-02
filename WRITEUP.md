# AI Proposal Triage System Design (MATACS-TH-2026-02)

## 1. AI vs. Rules
- **AI (LLM) Scope:** Parses unstructured proposal PDFs into a dynamically generated JSON schema based on the active project category. It extracts raw facts (counts, scope capacities, date ranges, prices, currencies, durations, and checklist booleans), verbatim quotes, and adversarial prompt injection flags. The AI is restricted to text interpretation; it never performs arithmetic, assigns points, or determines whether a contractor qualifies.
- **Plain Code / Rule Engine Scope:** Validates that extracted values match verbatim quotes, enforces category-configured mandatory pass/fail gates (boolean presence or minimum numeric thresholds), verifies whether stated experience meets configured scope and date windows, and calculates weighted scores using reusable mathematical methods.
- **Why Draw the Line Here:** LLMs are probabilistic, prone to calculation errors, and vulnerable to prompt injection attacks embedded in documents. Confining AI to raw evidence extraction while executing eligibility checks and scoring formulas in deterministic code ensures calculations are reproducible for the same validated inputs and configuration across any project domain.

## 2. Consistency and Explainability
- **Repeatability:** Extraction calls use `temperature: 0`, a fixed `seed`, and strict JSON schema validation to maximize extraction consistency, though minor variations can still occur across model updates. Once extracted facts are validated and stored, deterministic code evaluates them against versioned admin rules, making scoring completely repeatable.
- **Generic Scoring Engine & Sample Solar Configuration:** The code engine supports four category-agnostic calculation methods (`capped_count`, `linear_range`, `capped_ratio`, `checklist`). For the **Commercial Solar (150 kW) sample set**, the admin configuration maps criteria to these methods as follows:
  - **`capped_count` — Relevant Experience (30%):** Configured for installs >= 100 kW within the 5-year window (2021–2026) with a target anchor of `15`. If a proposal provides a summary spanning outside the window without an itemized yearly breakdown, code flags the ambiguity rather than guessing. For verified counts: `min(30, (qualifying_installs / 15) * 30)`.
  - **`linear_range` — Price (25%) & Timeline (15%):**
    - Price: Configured with $180k floor anchor and $260k ceiling: `min(25, max(0, ((260000 - price) / (260000 - 180000)) * 25))`.
    - Timeline: Configured with 8-week floor anchor and 16-week ceiling: `min(15, max(0, ((16 - weeks) / (16 - 8)) * 15))`.
  - **`capped_ratio` — Warranty (15%):** Configured with 25-year panel and 12-year workmanship targets: `min(7.5, (panel_years / 25) * 7.5) + min(7.5, (workmanship_years / 12) * 7.5)`.
  - **`checklist` — Team & Approach (15%):** Configured with 4 required items (`3.75` points each for named lead, specified crew, site safety plan, and monitoring plan): `(confirmed_items / 4) * 15`.
- **Missing Data, Limit, and Precision Policy:**
  1. *Mandatory gate failure:* Status = `ineligible`, `rank: null`, `score: null`.
  2. *Critical evidence missing/ambiguous or security alert:* Status = `flagged`, `rank: null`, `score: null` (holds official scoring and ranking for human review).
  3. *Confirmed weighted value exceeds ceiling:* Award `0` points for that criterion.
  4. *Missing checklist sub-item:* Award `0` points for that sub-item without pausing evaluation.
  5. *Ranking precision:* Code sums unrounded criterion floats, sorts proposals by unrounded total, assigns ranks, and rounds final scores to integers (0–100) for display.
- **Sample Set Evaluation:**
  - **Proposal A (`ranked`, Rank 1, Score 71):** Passes mandatory gates. Experience `28.00` + Price `6.875` + Timeline `7.50` + Warranty `13.75` + Team `15.00` = `71.125` -> rounds to `71`.
  - **Proposal E (`ranked`, Rank 2, Score 57):** Passes mandatory gates. Experience `16.00` + Price `5.46875` + Timeline `5.625` + Warranty `15.00` + Team `15.00` = `57.09375` -> rounds to `57`.
  - **Proposal D (`flagged`, Rank null, Score null):** Passes mandatory gates, but the installation timeline is deferred ("determined after a detailed site assessment"), preventing verification of the 16-week ceiling. Official scoring and ranking are held for human review.
  - **Proposal C (`flagged`, Rank null, Score null):** Flagged and held for human review due to an adversarial prompt injection in Appendix C and an unverified experience window ("9 commercial installations... since 2020" spans outside 2021–2026 without an itemized breakdown to count qualifying installs).
  - **Proposal B (`ineligible`, Rank null, Score null):** Fails mandatory gate because the bidder does not hold an Ontario electrical licence directly, deferring to an unconfirmed subcontractor.
- **Explainability UI:** Clicking any proposal row opens a split-screen view displaying the formula trace and missing-data flags next to highlighted verbatim quotes in the source PDF.

## 3. Human in the Loop
- **Checkpoints:**
  1. **Triage Hold Queue:** Flagged (C, D) and ineligible (B) proposals pause in a review queue. For Proposal C, the reviewer inspects the injection attempt and requests an installation breakdown by year without automatically disqualifying the contractor for an extraction flag.
  2. **Manual Override:** Reviewers can update verified values or release a held proposal after vendor clarification by submitting a mandatory justification.
  3. **Final Award:** A human procurement officer makes the final contract award decision.
- **Audit Log:** An append-only audit table records `timestamp`, `actor_id`, `file_sha256`, `criteria_version_id`, `raw_ai_json`, `validation_trace`, `override_diff`, and `justification`.

## 4. Changing Criteria
- **Decoupled Architecture & Versioned Rules:** Admin criteria configurations are versioned (`v1`, `v2`), and extracted proposal JSON is stored separately from calculated scores.
- **Rescoring:** When an admin updates weights, ceilings, or formula anchors, cached facts allow deterministic rescoring without another AI extraction call whenever existing data covers the updated criteria. If a new qualitative field is added, existing proposals display a `Stale Criteria (v1)` indicator and queue a targeted extraction for the new field while retaining historical `v1` audit records.

## 5. Risks and Safeguards
- **Prompt Injection (Proposal C):** Attackers embed instructions to manipulate evaluation. *Safeguard:* Isolate document text from system instructions, execute scoring outside the model in deterministic code, and flag suspicious text for human review.
- **Hallucinated Facts:** AI fabricates credentials or figures. *Safeguard:* Require a `verbatim_quote` per field and run code checks confirming the quote and extracted numbers/currencies exist in the raw PDF.
- **Ambiguous or Missing Evidence (Proposals C & D):** Deferred schedules or multi-year summaries. *Safeguard:* Extract `null` or flag unverified date ranges without guessing, holding official ranking (`flagged`) until human verification.
- **Cost & Latency:** Repeated parsing of large documents. *Safeguard:* Deduplicate uploads via SHA-256 hashing and cache structured JSON extractions.