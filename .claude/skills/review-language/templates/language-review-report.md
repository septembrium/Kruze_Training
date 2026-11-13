# Linguistic Review: [scope — e.g. UC-003 + related stories]

> **Scope Warning** *(only if applicable)*: No user stories trace to UC-NNN — only the use case was checked. Run `/review-stories UC-NNN` or `/slice-stories`.

## Summary
[1–2 sentences: overall clarity, the most important issue, and how many findings per severity]

## Verdict: [CLEAR | NEEDS CLARIFICATION | AMBIGUOUS]
- **CLEAR**: no Critical findings, ≤ 3 Warnings
- **NEEDS CLARIFICATION**: no Critical findings, > 3 Warnings
- **AMBIGUOUS**: at least one Critical finding

## Critical

### [L-rule]: [short title]
- **Location:** [file] → [section / step / BR / AC ID]
- **Text:** "[exact quote]"
- **Readings:** (1) [interpretation A] · (2) [interpretation B]
- **Rewrite:** "[concrete replacement]" *or* **Question for PO:** [decision needed]

## Warnings

### [L-rule]: [short title]
- **Location:** …
- **Text:** "…"
- **Readings:** …
- **Rewrite:** "…"

## Suggestions
| Rule | Location | Text | Rewrite |
|---|---|---|---|
| L14 | UC-NNN → Step 3 | "…" | "…" |

*Max 5 rows. If more were found: "+ N further suggestions omitted."*

## Terminology Report (L10)
| Term used | Glossary term | Locations | Issue | Fix |
|---|---|---|---|---|
| client | Customer | UC-NNN step 4, Story 2.3 AC-2 | synonym | use "customer" |

### Glossary Defects
- [Term] — [why the definition is ambiguous or overlapping] → Proposed definition: "…"

### Missing Glossary Entries
- [Term] — used in [locations] → Proposed definition: "…"

## Open Questions for the Product Owner

### From findings
1. [Every rewrite above that needs a business decision]

### Business-rule gaps (not linguistic — no severity)
1. [Things the document never addresses, e.g. "Which VAT class applies to shipping costs?"]
