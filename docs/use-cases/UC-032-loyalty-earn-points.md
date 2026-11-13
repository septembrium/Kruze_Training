# UC-032: Earn Loyalty Points

A registered B2C customer earns loyalty points on every paid purchase made in the webshop or in the shop. The points are recorded against the sale that earned them.

**Scope of v1:** earning points only. Redeeming points (UC-034), a separate loyalty login and email verification (UC-031), badges (UC-033) and challenges (UC-035) are out of scope and remain Draft.

## Metadata
- **Domain:** Customer / Loyalty / Sales
- **Primary Actor:** Registered Customer (loyalty member)
- **Secondary Actors:** Payment Provider (Mollie), POS User / Cashier (identifies the member in the shop).
- **Priority:** Should
- **Status:** Reviewed
- **Interface:** Storefront (checkout), POS Terminal (member identification)

## Preconditions
- The customer has a Storefront account with a linked customer record (see UC-026 AF-6). Every account holder is a loyalty member; B2B clients earn no points (BR-5).
- An earn rate has been configured by the admin (see UC-015 AF-6).
- For a webshop purchase: the member is logged in during checkout (see UC-025).
- For a shop purchase: the cashier is authenticated (see UC-021).

## Main Flow
1. The logged-in member places a webshop order and pays it through the payment provider (UC-025).
2. The system links the resulting sale to the member's customer record.
3. The payment provider confirms the payment, and the system marks the sale as paid.
4. The system determines the earn basis of the sale (BR-3), applies the configured earn rate, and rounds the result down to whole points (BR-4).
5. The system records the points as a new entry for the member, linked to the sale.

## Alternative Flows
- **AF-1: Earn Points on a POS Purchase** — At step 1, instead of buying in the webshop, the member buys in the shop. Before completing the sale, the cashier identifies the member by scanning the member's QR code or typing the member code. This is a faster way to select the customer and is equivalent to selecting a customer in UC-021 step 7, including the existing pricing rules (UC-021 step 8). The system shows the matched customer's name so the cashier can confirm it. The cashier completes the sale (UC-021), and the system records it as paid. The flow continues at step 4.
- **AF-2: Sale Without a Member** — At AF-1, the cashier does not select a customer, or selects a customer who is not a loyalty member (for example a customer quick-registered at the POS without a Storefront account, see UC-002 AF-6). The sale is completed as normal but earns no points, and the use case ends.
- **AF-3: B2B Customer** — At step 4, the customer is registered as a B2B client. No points are earned and the use case ends. At the POS, a B2B customer can still be selected and the sale proceeds as normal, but earns nothing.

## Exception Flows
- **EF-1: Member Code Not Found** — At AF-1, when the cashier scans or types a code, the system detects that it does not match any member. The system informs the cashier. The cashier can retry, or continue the sale without a member (AF-2).
- **EF-2: Sale Not Paid** — At step 3, the payment fails or stays pending (e.g., bank transfer, the default payment method, is not yet received). No points are awarded. When the payment is confirmed later, the flow continues at step 3.
- **EF-3: Points Cannot Be Recorded** — At step 5, the system cannot record the points. The sale itself is not affected. The points are still recorded once the problem is resolved, and never twice for the same sale (BR-10).

## Postconditions
- For a paid sale of a B2C member: one points entry exists for the sale, linked to it.
- For a sale without a member, or of a B2B client: no points entry exists.

## Business Rules
- **BR-1:** Points are awarded only for sales that are paid.
- **BR-2:** One global earn rate applies to all sales, set by the admin as points per EUR 1 (see UC-015). There are no different rates per channel, product or customer type. A new rate applies only to later sales.
- **BR-3:** The earn basis is the amount paid for products, including VAT, after discounts (including a sale-level discount), and excluding shipping cost. Example: a paid basket of EUR 42,50 incl. VAT plus EUR 5,00 shipping, at 1 point per EUR 1, earns 42 points.
- **BR-4:** Points are rounded down to whole points per sale.
- **BR-5:** Only customers who are not registered as B2B clients earn points. A B2B client earns nothing, even with a Storefront account. A B2B client who has a Storefront account still has a member code, but earns no points.
- **BR-6:** Only a logged-in member earns points online. The sale of a logged-in member is always linked to the member's own customer record, whatever email address is entered in the checkout order details. A guest checkout earns no points, even when its email address matches a member's customer record. At the POS, only a customer who is a loyalty member earns points, whether selected through the member code or the normal customer selection; a selected customer who is not a member earns no points and has no member code. The default Kruze company customer is never a loyalty member.
- **BR-7:** A sale without an identified member earns nothing and cannot be linked to a member afterwards in v1.
- **BR-8:** Each member has a unique member code of 8 characters that identifies them at the POS. The code contains no look-alike characters (no 0 or O, no 1, I or L). A member can regenerate the code (UC-026 AF-7); the old code stops working immediately, and points already earned stay with the member.
- **BR-9:** Points entries are never changed or removed, and every entry traces to a sale.
- **BR-10:** A sale awards points at most once.
- **BR-11:** Points awarded for a sale are final. Because a paid sale cannot be edited, removed or changed to another status (UC-003 BR-8, UC-004 AF-3 and BR-4), no reversal is needed in v1.

## Out of Scope (v1)
- Showing the balance, history and member code to the member (UC-026 AF-5) and configuring the earn rate (UC-015 AF-6) are described in those use cases.
- Redeeming points (UC-034), loyalty login, email verification and multiple emails (UC-031), badges (UC-033), challenges and bonus points from challenges (UC-035).
- Bonus or promotional multipliers (e.g., double-points weekends).
- Partial refunds and credit notes (the system has no refund flow), and points reversal (needed once refunds or credit notes exist).
- Admin viewing or adjusting a member's points.
- Linking a sale to a member after it is completed, and backfilling points for purchases made before go-live (every member starts at zero).
- Notifications. The member sees the updated balance in their Storefront account.

## Open Questions
None.

## Implementation Notes
- The legacy `loyalty` table is unused and is replaced by an append-only points ledger with one entry per award. A member's balance is the sum of their entries. The ledger structure should allow reversal entries later (see Out of Scope), but none are written in v1.
- The earn rate in force is stored with each entry, so history stays explainable after a rate change.
- The earning event is the sale becoming paid, not the webshop order record, which has its own separate status. Sale statuses are a reference table: NEW=1, SHIPPED=2, INVOICED=3, PAID=4 (`server/salesService.ts`, verified). POS sales are created as PAID. A webshop sale goes from NEW to PAID through the Mollie webhook. The unused `salesStatusEnum` in `shared/schema.ts` is not relevant.
- A paid sale can no longer be edited, have its status changed, or be removed (`server/salesService.ts` refuses all three), which is why awarded points are final.
- Shipping is stored as a separate sales line (`server/checkoutService.ts`, verified), so the earn basis can exclude it. Sale-level discounts must be taken into account.
- B2B is determined by the customer's B2B client flag, not the B2B prices flag.
- Award must be idempotent per sale: a retried award, or a repeated payment webhook, must not add a second entry.
- Membership: at first sign-in the account is matched to a customer record by email address (rules in UC-026 AF-6). Today the customer record is only created on first profile save, and the user and customer are linked by email address with no foreign key (see UC-026 Implementation Notes). The webshop sale of a logged-in member uses the linked customer record, not the email typed at checkout. Guest checkouts are never linked to a member, even when the email matches.
- The member code is exposed as a QR code and as text; both resolve to the same customer lookup at the POS (equivalent to the customer search in UC-021).

## Implemented By
- Status: **Not yet implemented** — v1 scope is defined but no code exists.
- Related use cases: UC-002 (customer type; AF-6 quick-register at the POS), UC-007 (customer pricing at the POS), UC-015 (earn rate), UC-021 (POS sale and customer selection), UC-025 (webshop checkout), UC-026 (member sees balance, history, member code), UC-003 and UC-004 (a paid sale cannot be changed or removed).
