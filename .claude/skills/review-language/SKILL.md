---
name: review-language
description: Linguistic check of requirements — ambiguity, vague or untestable wording, weak modals, terminology drift against the glossary, unclear references, and grammar. Use after writing or updating use cases, stories, or specs, or on demand.
user-invocable: true
argument-hint: "[UC-NNN, Story ID, SPEC-NNN, file path, or 'all']"
allowed-tools: Read, Grep, Glob
agent: linguist
---

Run a linguistic check on the requirements specified by `$ARGUMENTS`.

---

## Step 0 — Read References
Read the following before starting:
- `.claude/skills/review-language/references/linguistic-rules.md`
- `docs/product/backlog/GLOSSARY.md`

## Step 1 — Gather Documents

- **UC-NNN**: the use case in `docs/use-cases/`, plus the stories in `docs/product/backlog/` that reference it
- **Story ID** (e.g. "1.1"): the story file in `docs/product/backlog/epic-N-*/`
- **SPEC-NNN**: the spec in `docs/specs/`. Check only the requirement-bearing parts (acceptance criteria, behaviour descriptions, test plan); skip code blocks and file lists.
- **File path**: that file
- **"all"**: all use cases and story files. Warn that this is a large run and report per document.
- **"glossary"**: `GLOSSARY.md` itself (rule L10 only, plus definitions that use vague terms)

Skip `## Implementation Notes` and `## Implemented By` sections: they're technical by design.

**Pre-check — traceability:** For a UC-NNN scope, if no story file references the use case, add a **Scope Warning** at the top of the report ("No user stories trace to UC-NNN — only the use case was checked; run `/review-stories UC-NNN` or `/slice-stories`"). This is not a linguistic finding and does not count towards the verdict.

## Step 2 — Scan for Candidates

For each document, use the trigger lists in the rules reference to find candidate sentences (Grep is fine for this). Candidates are **not** findings yet.

## Step 3 — Judge Each Candidate in Context

For each candidate, apply the rule's "Ask" question:
- Can you name **two distinct readings** a developer or tester could act on? If not, discard it.
- Would the difference change **what gets built or how it gets tested**? If not, downgrade it to Suggestion or discard it.
- Is the problem the **wording** or the **missing business content**? If no rewrite can fix it without the product owner deciding something that the document never addresses (e.g. which VAT class applies to shipping, whether green coffee is sellable), it is a business-rule gap, not a linguistic finding. Put it under **Open Questions** only — no finding, no severity, no effect on the verdict. A finding is linguistic when the text *does* say something, but says it ambiguously.

## Step 4 — Terminology Pass (L10)

Across all documents in scope:
1. List every domain noun used for actors, entities, states and channels.
2. Map each one to a glossary term. Flag synonyms, case/spelling variants, terms used with a meaning that differs from the glossary, and domain terms missing from the glossary.
3. Flag glossary entries that are themselves ambiguous (one term, two meanings) or overlap with another entry.

## Step 5 — Produce Report

Use `.claude/skills/review-language/templates/language-review-report.md`.

**Severity levels:**
- **Critical**: the ambiguity changes the testable behaviour (two readings → two different implementations), or a key term conflicts with the glossary
- **Warning**: readers will probably converge on the same meaning, but the text invites questions or isn't verifiable as written
- **Suggestion**: clarity, grammar, or consistency improvements with no change in meaning

**Suggestions are capped at 5.** Keep the five with the most value to a reader, and end the table with "+ N further suggestions omitted" if there were more. Critical findings and Warnings are not capped.

Group repeated instances of the same issue into one finding with a list of locations. Every finding needs: location, quoted text, the competing readings (for ambiguity rules), and a concrete rewrite or a product-owner question.

---

## Retrospective

After completing this task (or after a review session with the user), review the conversation for any corrections or feedback the user gave. If the user pointed out false positives, missed issues, or preferred phrasing that is **not already captured** in this skill or its rules reference:

1. List each lesson learned with a brief explanation of what was wrong and what the correct approach is.
2. Propose specific changes to this skill file or `references/linguistic-rules.md` (new trigger words, new exceptions, adjusted severity).
3. Propose any user-specific preferences to save to memory (feedback type).

**Do not apply changes.** Present the proposals and wait for the user to approve before modifying anything.
