# Linguistic Rules for Requirements

Based on ISO/IEC/IEEE 29148 (characteristics of well-formed requirements) and the INCOSE *Guide to Writing Requirements*, adapted to KruzeApp's use case and story format.

**Trigger lists find candidates, not violations.** Every rule has an "Ask" question; only report a finding if the answer exposes a real second reading or an unverifiable statement.

---

## L1 — Vague or Subjective Terms
Words whose meaning depends on the reader's opinion.

**Triggers:** appropriate, relevant, adequate, sufficient, reasonable, suitable, user-friendly, intuitive, easy, simple, clear, efficient, flexible, robust, seamless, nice, good, proper, correct(ly), normal(ly), typical(ly), significant, basic, standard (when not referring to a defined "standard price"), etc.

**Ask:** Could a tester write a pass/fail check for this word as it stands?

| Vague | Rewrite |
|---|---|
| The system shows the relevant sales. | The system shows the sales of the selected customer from the current fiscal year. |
| The system displays basic statistics. | The system displays the number of sales and the total revenue (excl. VAT). |

## L2 — Weak Modals and Escape Clauses
Words that make it unclear whether something is required.

**Triggers:** may, might, could, should, can, optionally, if possible, as appropriate, where applicable, if needed, if necessary, when relevant, as required, ideally, preferably, try to, attempt to.

**Ask:** Who has the choice: the **actor** or the **system**?
- Actor permission is **fine**: "The admin may override the price." / "The admin optionally adds a discount."
- System behaviour must be **definite**: "The system may send a confirmation" → does it or doesn't it, and under which condition?

| Weak | Rewrite |
|---|---|
| Stock movements may be created depending on the sale status workflow. | When the sale status changes to "delivered", the system records a stock movement for each line item. *(confirm trigger status with PO)* |
| The system should notify the customer. | The system notifies the customer by email. |

## L3 — Open-Ended Lists
Lists that can't be checked for completeness.

**Triggers:** etc., and so on, and more, such as, including, e.g., for example, among others, like (as "such as"), and other …

**Ask:** Is this list an *example* (fine in descriptions) or the *definition of scope* (must be complete)?
- In flows, business rules and acceptance criteria, scope lists must be closed.
- "e.g." pointing to an example UC or vendor ("Payment Provider (e.g., Mollie)") is fine.

| Open | Rewrite |
|---|---|
| The system asks for customer details such as name, email, etc. | The system asks for the customer's name, email address, phone number and VAT number. |

## L4 — Ambiguous Conjunctions and Slashes
**Triggers:** and/or, `/` between words (unit/weight, add/edit), "or" in a rule or criterion, "and" joining two conditions.

**Ask:** Inclusive or exclusive? Both at once, or either one? One field or two?

| Ambiguous | Rewrite |
|---|---|
| The admin selects the unit/weight. | The admin selects the unit and enters the weight. *(or: selects the unit, which determines the weight; whichever is true)* |
| Discounts can be a percentage or fixed amount. | A discount is either a percentage or a fixed amount, not both. |

## L5 — Unclear References
Pronouns and demonstratives with more than one possible antecedent.

**Triggers:** it, its, this, that, these, those, they, them, the former, the latter, the above, the same, the item, the record.

**Ask:** Does the pronoun have exactly one candidate noun in the same or preceding sentence?
- Singular "they" for a single actor ("the admin indicates they want…") is **correct**; don't flag it.

| Unclear | Rewrite |
|---|---|
| The system links the roast to the green coffee and updates its stock. | The system links the roast to the green coffee and reduces the green coffee stock. |

## L6 — Passive Voice Hiding the Actor
**Triggers:** is/are/was/were/be/been + past participle, with no "by …".

**Ask:** Does it matter *who* does this, and can the reader tell?
- Postconditions describing a resulting state are fine in passive ("The sale is recorded").
- Flow steps and business rules must name the actor.

| Passive | Rewrite |
|---|---|
| The price is overridden. | The admin overrides the price. |
| The invoice number is assigned. | The system assigns the next invoice number. |

## L7 — Unquantified Terms and Comparatives
Words implying a measure without giving it.

**Triggers:** fast, quick(ly), slow, real-time, instant(ly), immediately, large, small, many, few, several, some, most, recent, frequently, often, regularly, high, low, more, less, better, faster, minimal, maximal, limited.

**Ask:** What number, threshold or reference point makes this testable?

| Unquantified | Rewrite |
|---|---|
| The system calculates totals in real-time. | The system recalculates the totals each time a line item, discount or shipping cost changes. |
| The dashboard shows recent sales. | The dashboard shows the 10 most recent sales by sale date. |

## L8 — Absolutes and Universal Quantifiers
**Triggers:** all, every, each, any, always, never, none, nothing, only, no (as quantifier), completely, entire.

**Ask:** Is the absolute really intended, and are there exceptions documented elsewhere (a BR or AF) that contradict it?

| Risky | Rewrite |
|---|---|
| All sales can be edited. *(BR-8 says paid sales can't)* | Sales that are not marked as paid can be edited. |

## L9 — Negatives and Double Negatives
**Triggers:** not, no, never, cannot, unless, except, without, un-/in-/non- prefixes combined with another negation.

**Ask:** Can it be stated positively? Is the negative a real constraint (fine) or a roundabout positive?

| Negative | Rewrite |
|---|---|
| The system does not allow sales without at least one line item to not be saved. | The system requires at least one line item before it records a sale. |

## L10 — Terminology Consistency (against GLOSSARY.md)
The same concept must use the same word everywhere, and every word must mean what the glossary says.

**Check for:**
- **Synonyms**: customer / client / buyer; sale / order / transaction; remove / delete; product / item / article
- **Spelling and case variants**: Webshop / webshop / web shop; B2B client / B2B customer
- **Meaning drift**: a glossary term used with a different meaning (e.g. "Webshop Order" used for a confirmed Sale)
- **Undefined domain terms**: business nouns that aren't in the glossary (statuses, flags, channels)
- **Glossary defects**: one term with two meanings, or two terms with overlapping definitions. These are reported as Critical, because they affect every document.

**Note:** Rule 5 of the use case writing rules (business verbs: *remove* not *delete*) belongs to the review agent; only flag it here if the *same document* mixes both.

## L11 — Sentence Complexity
**Triggers:** more than ~30 words, more than one "and"/"or" joining requirements, nested conditions ("if … unless … when …"), parentheses that contain a requirement.

**Ask:** Does this sentence carry more than one requirement or condition? Can a tester trace a single pass/fail to it?

| Complex | Rewrite |
|---|---|
| The admin can edit the sale and its line items unless it is paid, in which case only the notes can be changed (if the stock was not updated). | Split into one rule per condition, each with its own ID. |

## L12 — Undefined Abbreviations
**Triggers:** acronyms not in the glossary and not expanded at first use in the document.

Known and allowed without expansion: VAT, BTW, B2B, B2C, POS, SKU, UC, SPEC, PO.

## L13 — Temporal Ambiguity
**Triggers:** after, before, when, once, until, during, periodically, daily, at the end of, later, then, soon, eventually.

**Ask:** Is the trigger event or time anchor explicit? Whose timezone or which calendar (fiscal year = calendar year)?

| Ambiguous | Rewrite |
|---|---|
| The report is generated at the end of the day. | The system generates the report when the admin closes the POS register for the day. |

## L14 — Grammar, Spelling and Punctuation
- Spelling errors and typos
- Subject–verb agreement, missing articles, dangling modifiers
- **English variant**: don't impose British or American spelling, but flag a document that mixes both (e.g. "behaviour" and "behavior" in the same file)
- Inconsistent capitalisation of glossary terms within a document
- Always report as **Suggestion** unless the error changes the meaning
