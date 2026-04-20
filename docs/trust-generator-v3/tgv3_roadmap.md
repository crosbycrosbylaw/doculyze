# Trust Generator v3 — Roadmap

Session tracker for the remaining design sessions leading up to Claude Code implementation. Ordered by the tradeoff-analysis weighting from this session's sequential-thinking pass, grouped into execution tiers, with session-specific insights inlined.

**Status legend:**

- `[ ]` not started
- `[~]` in progress
- `[x]` complete
- `[-]` skipped or deferred

---

## Where we are

- [x] **Schema consolidation (TGv3 schema.py)** — Complete. File delivered at `/mnt/user-data/outputs/schema.py`, 10 integration tests pass, all 17 TGv3 memory commitments reflected in code.

## What comes next — the critical path

The critical path through remaining design work, derived from memory 19 and refined by the tradeoff-analysis. Items on the same tier can run in parallel; tier-to-tier transitions are strict dependencies.

### Tier 1 — Schema-adjacent shape work (parallel)

Three sessions that can run in any order or simultaneously, each depending on schema.py but not on each other.

- [ ] **Firm config module** (burden rank 13/14, lightest tier)
    - **Scope:** File-backed config loader with hierarchy (firm-wide, office, user). Keys: estate thresholds, trustee catalog radius, audit log path, office location, template overrides, default guardianship policy.
    - **Why lightest:** dependencies are clear, no unknowns, deliverable is straightforward Python plus a YAML or TOML schema.
    - **Insight:** do this one first among the Tier 1 sessions. It establishes the config keys that diagnostics and promote_seed will reference; resolving it early prevents rework.
    - **Library recon before starting:** `pydantic-settings` (Pydantic-native config), `ruamel.yaml` (preserves comments for paralegal-editable files), `tomli` (stdlib TOML reader in 3.11+).

- [ ] **promote_seed refinement** (burden rank 14/14, lightest)
    - **Scope:** The stub in schema.py handles trust_type × marital_status caption resolution and co_grantor presence. What remains: integration with firm config, handling of accessibility overrides at the generator boundary, conflict resolution for when seed data disagrees with later-filled data (rare but possible).
    - **Why lightest:** bounded function, testable in isolation.
    - **Insight:** the current stub passes all happy-path tests. This session is really about edge cases and test coverage, not new design. Could be folded into the generator-migration session if short on budget.

- [ ] **Diagnostics engine** (burden rank 5/14, medium)
    - **Scope:** Rule loading from YAML files in firm-config directory, per-rule toggle, audit log writer, integration of `rule-engine` library, test harness. GUI integration points (rule list, edit form, test runner) designed but deferred to the GUI rules pane session.
    - **Why mid-tier:** cross-cutting (touches schema, config, GUI, audit). Higher legal risk since rules encode legal judgments.
    - **Insight:** this is the session where the "dangerously useful" framing matters most. A well-designed diagnostics engine catches cliff-crossings, missing trustees, and pretermitted-heir risks before generation. A poorly-designed one gives paralegals false confidence. Invest in the rule library and test harness; the code wrapping rule-engine can be minimal.
    - **Library recon before starting:** confirm `rule-engine` package version, review its type-resolver pattern for handling Pydantic models as rule context.

### Tier 2 — Migration (serial)

Parser and generator migration must run after Tier 1 (because they consume config and emit diagnostics) and must run in order (generators consume parsed data).

- [ ] **Parser migration** (burden rank 8/14, medium-high)
    - **Scope:** Adapt `docx_parser.py`, `pdf_parser.py`, `json_parser.py` to the new schema. Permissive date coercion for handwritten paper inputs ("3/15/47"). Decimal coercion for currency fields. Structured Address parsing from free-text when OSM lookup is unavailable.
    - **Why medium-high:** three files, pre-existing heuristic complexity in `docx_parser.py`. The v2.2 `_HINTS`, `_CHECKBOX_MAP`, and `_KEY_MAP` scaffolding largely dissolves once the schema is the source of truth.
    - **Insight:** this session is an excellent place to establish a regression test corpus. Collect every v2.2 questionnaire artifact the firm has, run both old and new parsers against it, and verify the TrustData outputs are structurally equivalent (accounting for schema rename/collapse). This becomes the safety net for the rest of v3.

- [ ] **Generator migration** (burden rank 7/14, medium-high)
    - **Scope:** Adapt `trust_document.py`, `printable_questionnaire.py`, `pdf_questionnaire.py` to consume new TrustData. Update references to `grantor`/`co_grantor` fields, format Decimals as currency, format dates as `MM/DD/YYYY`, resolve ref-by-id recipients from beneficiary lists.
    - **Why medium-high:** three files, but output formatting is straightforward. Legal risk is high because generator output is the firm's visible product.
    - **Insight:** take the opportunity to add diagnostic hooks. Every generator should invoke `diagnose()` before emitting output and respect the blocking/override contract. This is the integration moment for the diagnostics engine; do not postpone it to a later session.

### Tier 3 — New-surface production (parallel after Tier 2)

Four sessions that can run simultaneously; each depends on Tier 2 but not on each other.

- [ ] **Printable accessibility redesign** (burden rank 6/14, medium)
    - **Scope:** 14-16pt Arial minimum, generous line-height, high-contrast rendering, larger checkboxes and input lines. Conditional rendering logic for the 18 variants (trust_type × marital_status × estate_value × child_count). Overlay handling for pets and explicit guardianship.
    - **Why mid-tier:** iterative visual proofing is slower than pure code, but the variant logic is structured — not 18× the code, maybe 20% over the six-variant base.
    - **Insight:** the variant count (18) does not scale the work linearly; most sections render identically across variants, and the variant-specific logic lives in a handful of conditional blocks. The heaviest work is the accessibility pass itself — font sizing, tab-stop layout, paper-fold mechanics. Get one variant right before expanding.
    - **Library recon before starting:** `python-docx` for basic authoring, `docxcompose` if assembling from templates. Consider `reportlab` platypus for higher layout control if docx accessibility proves limiting.

- [ ] **docxtpl + SDT Word template** (burden rank 1/14, HEAVIEST session)
    - **Scope:** New `generators/fillable_word.py` plus a master `templates/intake.docx` authored in Word with Jinja2 tags for static content and tagged content controls (SDT `w:tag` = dotted schema path) for user inputs. Repeating sections for children, trustees, assets. Date picker SDTs, checkbox SDTs, dropdown SDTs. Plus a matching `parsers/fillable_word_parser.py`.
    - **Why heaviest:** this session carries the most residual design risk in v3. Word's SDT behavior has genuine unknowns around round-trip fidelity, repeating-section compatibility across Word versions, and lock semantics. `docxtpl` documentation is good but SDT authoring in Word itself is slow and iterative.
    - **Insight:** run library reconnaissance *before* the design session proper, not as part of it. Specifically: (a) verify `docxtpl`'s support for SDT tags via a minimum-viable template, (b) test repeating sections on both Word 2019 and 365, (c) confirm lock semantics prevent users from deleting SDT containers. If recon reveals dealbreakers, pivot to a simpler fillable-Word approach (pre-allocated rows, no SDTs) before committing session budget.
    - **Recommend:** split this into two sessions — "SDT recon and approach decision" (light) and "template authoring and parser" (heavy). The recon decision gates the design.

- [ ] **PDF completion (lists and elections)** (burden rank 11/14, medium)
    - **Scope:** Extend `pdf_questionnaire.py` + `pdf_parser.py` to cover the asset lists (children, trustees, beneficiaries) and all election checkboxes. AcroForm does not support repeating sections natively; implement as pre-allocated rows (e.g., 5 children, 6 accounts) with parser skipping blanks.
    - **Why mid-low:** the scaffolding already exists in v2.2; the work is extending it. Moderate complexity in AcroForm's repeating-section workaround.
    - **Insight:** this is the simplest of the three format surfaces. Treat it as a tight deliverable, not a design-heavy session. The pre-allocated-row pattern is well-trodden and matches the printable convention.
    - **Library recon before starting:** `reportlab` AcroForm primitives, `pypdf` for parser-side field extraction.

- [ ] **GUI questionnaire tab** (burden rank 10/14, medium)
    - **Scope:** New tab in the existing GUI for the paralegal's seed-entry surface. Inputs: trust_type, marital_status, estate_value_estimate, child_count_tier, preliminary_trust_name, has_pets, consultation metadata. Action: generate tailored printable for this seed.
    - **Why mid-tier:** a new UI component, but structured as form widgets plus a single "generate" action. Framework-dependent — existing v2.2 GUI framework (confirm which) shapes the effort.
    - **Insight:** this tab is directly visible to the paralegal and will likely be their most-used v3 entry point. Prioritize input validation and clear feedback ("variant joint_married_above_threshold_one_to_five will be generated") over aesthetic polish.

### Tier 4 — GUI surfaces and OCR (after Tier 3)

Two sessions, each depending on earlier tiers. These can run in parallel but are heaviest and benefit from serialization when budget is constrained.

- [ ] **GUI rules pane** (burden rank 9/14, medium-high)
    - **Scope:** New tab listing all diagnostic rules with toggle switches, descriptions, and level indicators. Guided form for authoring new rules; raw-YAML editor as advanced mode. Test-rule button that evaluates the rule against a chosen TrustData sample.
    - **Why mid-tier:** the test harness is the non-trivial component — rendering the rule, its description, and letting a paralegal run it against a dummy TrustData requires thoughtful UX.
    - **Insight:** the GUI framework choice from the questionnaire tab session carries over here. Focus on making rule descriptions legible — paralegals will toggle rules based on the description, not the expression. Well-written descriptions are the product.

- [ ] **GUI transcription surface** (burden rank 2/14, second-heaviest)
    - **Scope:** Split-view scan viewer and form. Per-field confidence indicators (green/yellow/red from extraction backend). Scan-to-field anchor (click field → highlight scan region). "Next unfilled" keyboard shortcut. Autosave on every field change.
    - **Why second-heaviest:** this is the most complex new UI surface in v3. Confidence indicators, anchor highlighting, and autosave each carry UX complexity.
    - **Insight:** this session depends on OCR integration being at least in-scope, because the confidence indicators come from the extraction backend. Plan this session after or concurrent with OCR integration.
    - **Consider splitting:** "scan-viewer and form layout" (heavy) and "confidence indicators and anchors" (medium) as two sessions. The layout session produces a usable surface without OCR; the indicators session adds the AI-assist layer.

- [ ] **OCR pre-fill integration** (burden rank 3/14, heavy)
    - **Scope:** Extraction backend Protocol in `doculyze_core.extract`. Two implementations: `OllamaBackend` (dev/local) and `AnthropicBackend` (firm production). Confidence scoring from extraction output. Integration with the GUI transcription surface.
    - **Why heavy:** model selection, prompt engineering for legal-domain extraction, confidence calibration — each carries genuine unknowns.
    - **Insight:** start with the Protocol definition and test against synthetic TrustData. Prompt engineering iterates faster when the surrounding scaffolding is stable. Consider splitting into sub-sessions: "Protocol + Ollama backend" (medium), "Anthropic backend" (medium), "confidence calibration and GUI integration" (medium).

### Tier 5 — Cross-repo extraction

- [ ] **doculyze-core extraction** (burden rank 12/14, light)
    - **Scope:** Once OCR integration lands and medscan becomes a second consumer, extract shared primitives into `doculyze-core`: `doculyze_core.ocr` (PyMuPDF wrapper), `doculyze_core.extract.backend` (Protocol + implementations), `doculyze_core.confidence` (scoring), `doculyze_core.auth` (API key helpers).
    - **Why light:** by the time this runs, all patterns are established. It is a refactor, not a design.
    - **Insight:** defer this until after v3 ships. Premature extraction across one consumer (trust-generator only) is worse than delayed extraction across two.

---

## Tradeoff-analysis summary (for session budgeting)

Weighted rank-sum from the Phase 4 output, lower = heavier burden:

| Rank | Session | Rank-sum | Tier |
|---|---|---|---|
| 1 | docxtpl+SDT | 1.62 | Tier 3 |
| 2 | GUI transcription | 2.12 | Tier 4 |
| 3 | OCR integration | 2.38 | Tier 4 |
| 4 | schema.py | 2.42 | **Complete** |
| 5 | Diagnostics engine | 2.72 | Tier 1 |
| 6 | Printable redesign | 3.06 | Tier 3 |
| 7 | Generator migration | 3.18 | Tier 2 |
| 8 | Parser migration | 3.58 | Tier 2 |
| 9 | GUI rules pane | 3.86 | Tier 4 |
| 10 | GUI questionnaire tab | 3.88 | Tier 3 |
| 11 | PDF completion | 4.28 | Tier 3 |
| 12 | doculyze-core | 4.60 | Tier 5 |
| 13 | Firm config | 4.96 | Tier 1 |
| 14 | promote_seed refinement | 5.28 | Tier 1 |

**Convergence status: refinement-needed.** Specifically, S1 (schema.py) vs S13 (OCR integration) rank-sums are within 0.04 — effectively tied. If schema composition proves more complex than estimated during later implementation work, expect schema.py's effective burden to be higher than rank 4 suggests.

---

## Process commitments

- [ ] **Library reconnaissance precedes each design session for components with plausible existing libraries.** Documented outcome per component: adopt / wrap / build-custom with rationale. Recon targets listed in each Tier above.

- [ ] **New design decisions are persisted to memory during their originating session**, with `TGv3:` prefix for trust-generator-specific items and no prefix for generalizable ones.

- [ ] **Diagnostic rules for the rule engine are authored declaratively in YAML**, not in Python. Each session that surfaces a new diagnostic should add its rule to the rule file rather than hardcoding the check.

---

## Milestones for session checkpoints

After each tier, a demonstrable test confirms forward progress:

- [x] After schema.py: can validate a TrustData instance programmatically — 10 tests pass.
- [ ] After Tier 1: diagnose() runs against TrustData and produces a `list[Diagnostic]`; firm config loads from disk; promote_seed round-trips through all 18 variants.
- [ ] After Tier 2: every v2.2 test questionnaire can be re-ingested to new TrustData and re-emitted as v2.2-equivalent artifacts.
- [ ] After Tier 3: paralegal can generate a tailored printable from a seed via the GUI; the fillable Word and PDF produce round-trip-parseable documents.
- [ ] After Tier 4: paralegal can scan a completed paper questionnaire and receive a pre-filled TrustData with confidence indicators in under 60 seconds.
- [ ] After Tier 5: medscan consumes the same `doculyze_core.extract` protocols used by trust-generator.

---

## Cleanup after v3 ships

- [ ] Confirm v3 implementation is complete and in firm use.
- [ ] Review each `TGv3:`-prefixed memory with Claude; promote any findings of lasting value (e.g., `rule-engine` library suitability) to generalizable memories without the prefix.
- [ ] Remove all remaining `TGv3:`-prefixed memories per the stored cleanup instruction.
- [ ] Retain generalizable memories (Python conventions, library reconnaissance process).
