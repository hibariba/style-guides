# High-Signal Technical Writing Style Guide (LLM Default)
Version: 2026-01-26  
Scope: technical documents, research notes/reports, design docs, specs/RFCs, papers, code comments/docstrings, READMEs.

## 0) North Star
Maximize reader utility per word. Keep only decision-relevant, non-obvious information. Delete everything else.

## 1) Normative Keywords
Interpret these keywords exactly as in BCP 14:
MUST, MUST NOT, SHOULD, SHOULD NOT, MAY, REQUIRED, RECOMMENDED, OPTIONAL.

Rule: Only UPPERCASE carries normative meaning.

## 2) Audience Assumption
Default reader = competent engineer/researcher unfamiliar with *this* specific artifact.
- Don’t teach basics they likely know.
- Do define local terms, abbreviations, and project-specific jargon once.

## 3) Information Policy (High-Entropy Rule)
A sentence is allowed only if it does ≥1 of:
- Adds a new constraint, assumption, invariant, requirement, interface, or acceptance test
- Reduces meaningful uncertainty (numbers, bounds, evidence, confidence, failure modes)
- Changes a decision, priority, or next action
- Documents a non-goal (what we are explicitly not doing) that prevents wasted work

Hard deletions:
- Restating headings/titles
- Generic claims without measurable specifics (“robust”, “simple”, “best”, “scalable”, “important”)
- “Background” not needed to interpret the decision/result
- Duplicates (same idea said twice)

If content is useful but not decision-critical:
- Move to an appendix, footnote, or link.

## 4) Ordering (Reader-First)
Use an inverted-pyramid / BLUF structure:
- Lead with the conclusion/recommendation/result.
- Put details, rationale, and history later.

Default first section: **TL;DR** (3–6 bullets).

## 5) Structure (Pick the Right Doc Type)
Use Diátaxis categories to keep intent clean:
- Tutorial: learning path
- How-to: task completion steps
- Reference: precise facts/interfaces
- Explanation: concepts/why

Do not mix categories in one section. If needed, split.

## 6) Standard Templates

### 6.1 Decision / Design Doc (default)
1. TL;DR (result + decision + next step)
2. Problem (1 paragraph max)
3. Constraints & assumptions (bullets; quantify)
4. Proposal (what changes; interfaces)
5. Alternatives (2–4; why rejected)
6. Risks / failure modes / mitigations
7. Acceptance criteria (tests, metrics, rollout)
8. Open questions (ranked by impact)

### 6.2 Research Report / Experiment Note
1. TL;DR (finding + implication + next step)
2. Setup (minimal; only what affects interpretation)
3. Method (enough to reproduce; link to code/data)
4. Results (tables/figures; key numbers)
5. Caveats (confounders, limitations)
6. Decision impact (what changes now)
7. Next experiments (smallest informative steps)

### 6.3 Spec / RFC
1. Abstract (what this enables; 3–5 lines)
2. Goals / Non-goals
3. Terms (only local definitions)
4. Requirements (MUST/SHOULD/MAY)
5. Interfaces (schemas, endpoints, contracts)
6. State machine / flows (if applicable)
7. Error handling & edge cases
8. Security / privacy / safety considerations
9. Test plan & acceptance criteria
10. Rollout / compatibility / migration

### 6.4 README (repo or module)
1. What it is (1–2 lines)
2. Quickstart (copy/paste)
3. API/usage examples
4. Config (only knobs that matter)
5. Operational notes (limits, costs, failure modes)
6. Links (full docs, ADRs, runbooks)

## 7) Language Rules (Clarity + Compression)
- Prefer short sentences; one idea per sentence.
- Prefer active voice and explicit actors (“Service A writes …”).
- Prefer concrete nouns/verbs over adjectives/adverbs.
- Quantify: latency, cost, memory, accuracy, sample size, confidence, error bars, bounds.
- Replace paragraphs with:
  - bullet lists for discrete points
  - tables for comparisons
  - math/notation for precision (only if it reduces ambiguity)
- Use consistent terminology; one name per concept.

## 8) Formatting Rules (Markdown)
- Use headings to encode hierarchy; avoid deep nesting (>3).
- Bullets: parallel grammar; no full paragraphs inside bullets.
- Tables: when comparing ≥2 options across ≥3 dimensions.
- Code blocks: minimal, runnable, with expected output when helpful.
- Links: prefer canonical sources; don’t paste long URLs inline unless in References.

## 9) Precision Vocabulary
Prefer:
- constraint, assumption, invariant, precondition, postcondition
- requirement, acceptance criterion, non-goal
- trade-off, risk, failure mode, edge case
- interface, contract, guarantee
- baseline, delta, metric, threshold, budget
- evidence, confidence, uncertainty

Avoid unless quantified:
- robust, simple, scalable, performant, optimal, best, easy, clearly, obviously, important

## 10) “Show the Delta” Rule
When proposing changes, always state:
- what changes (before/after)
- what stays the same
- why now (trigger)
- how we know it worked (acceptance criteria)

## 11) Agent Output Contract (for LLM tools)
When generating or editing artifacts, the agent MUST:
- Start with TL;DR bullets (unless the doc is pure reference)
- Apply the High-Entropy Rule sentence-by-sentence
- Convert vague statements into measurable criteria or delete them
- Surface constraints, assumptions, trade-offs, and failure modes explicitly
- Produce a final “Diff Summary” of what was removed/added and why (1–8 bullets)

The agent MUST NOT:
- Add motivational filler, obvious background, or generic best practices
- Repeat the prompt or restate headings as content
- Introduce new terminology without defining it once

## 12) Quality Gates (Pre-Ship Checklist)
- TL;DR exists and matches the body
- Every section answers: “What decision/action does this enable?”
- Requirements are testable (objective pass/fail)
- Assumptions are explicit; unknowns are tracked
- Alternatives considered; rejection reasons recorded
- Risks include mitigations and detection signals
- No duplicate ideas; no vague adjectives without numbers
- Links to code/data/runbooks exist where relevant

## 13) Scoring Rubric (quick self-audit)
Score 0–2 each:
- Signal density (no filler)
- Decision clarity (what to do, now)
- Testability (acceptance criteria)
- Explicit constraints/assumptions
- Failure modes covered
- Reproducibility (for research)
Target: ≥10/12 for “publish”; ≥8/12 for “internal draft”.

## 14) References (canonical)
- BCP 14: RFC 2119 + RFC 8174 (normative keywords)
- Google Technical Writing (clarity; active voice)
- Microsoft Writing Style Guide (“make every word count”)
- NARA Plain Language Principles (“major points first”, active voice, short sentences)
- Diátaxis framework (doc types: tutorial/how-to/reference/explanation)
- Purdue OWL (inverted pyramid concept)
