# UC-032: Earn Loyalty Points

A customer with a Storefront account automatically earns loyalty points on every completed and paid purchase, in the webshop or in the shop at the POS. The customer sees their points balance, points history, and personal member code in their Storefront account. This drives repeat visits and rewards customers for their spending with KRUZE.

**Scope of v1:** earning points only. Redeeming points (UC-034), a separate loyalty login and multi-email handling (UC-031), badges (UC-033), and challenges (UC-035) are out of scope and remain Draft.

## Metadata
- **Domain:** Customer / Loyalty / Sales
- **Primary Actor:** Registered Customer (loyalty member)
- **Secondary Actors:** POS User / Cashier (identifies the member at the POS), Admin (configures the earn rate)
- **Priority:** Should
- **Status:** Draft (v1 scope, pending review)
- **Interface:** POS Terminal (identify member), Storefront (checkout and customer account page), Admin Dashboard (configuration)

## Preconditions
- The customer has a Storefront account linked to a customer record (see UC-026). Having such an account makes the customer a loyalty member; there is no separate enrolment.
- An earn rate is configured by the admin (AF-4).
- For POS purchases: the cashier is authenticated (see UC-021).
- For webshop purchases: the customer is logged in during checkout (see UC-025).

## Main Flow
1. The member makes a purchase, either in the shop at the POS (UC-021) or in the webshop (UC-025).
2. The system links the purchase to the member:
   - **POS:** before completing the sale, the cashier identifies the member by scanning the member's QR code or typing the member code (AF-1). The sale is registered to that customer.
   - **Webshop:** the logged-in customer is automatically the purchaser.
3. The sale becomes completed and paid (POS: when the sale is confirmed; webshop: when the payment provider reports the order as paid).
4. The system calculates the points earned from the sale amount and the configured earn rate, applying the rounding rule (BR-4).
5. The system records the points as a new entry in the member's points history, linked to the sale.
6. The member opens their Storefront account page (UC-026) and sees the updated points balance, the new entry in the points history (date, channel, sale reference, points), and their personal QR code / member code.

## Alternative Flows
- **AF-1: Identify Member at POS** — At step 2, the cashier identifies the member either by scanning the member's QR code (shown on the member's phone or printed) or by typing the member code. The system shows the matched customer's name so the cashier can confirm it is the right person. The flow continues at step 3. The cashier can remove or change the identified member before the sale is completed.
- **AF-2: View Points History** — The member opens the points history on their account page. The system lists every entry, newest first, with date, channel (POS or webshop), sale reference, and points (positive for earned, negative for reversed).
- **AF-3: Sale Without Member Identification** — At step 2 (POS), the cashier completes the sale without identifying a member, or a guest checks out in the webshop. The sale earns no points and the use case ends without a points entry (see BR-6 and Open Question 5).
- **AF-4: Configure Earn Rate** — The admin opens the loyalty configuration in the Admin Dashboard and sets the earn rate (e.g., points per EUR spent). The new rate applies to sales completed after the change; points already awarded are not recalculated.

## Exception Flows
- **EF-1: Member Code Not Found** — At AF-1, the scanned or typed code does not match any member. The system informs the cashier. The cashier can retry, or continue the sale without a member (AF-3).
- **EF-2: Sale Not Paid** — The sale is not completed or the payment fails or stays pending (e.g., delayed bank transfer). No points are awarded. If the payment is later confirmed, the flow resumes at step 3.
- **EF-3: Refund or Cancellation** — A sale that earned points is refunded or cancelled. The system adds a reversing entry (negative points) linked to the same sale, so the points earned for it are removed from the balance. The original entry is never edited or deleted. A partial refund reverses the proportional number of points (see Open Question 4).
- **EF-4: Points Award Failure** — The system cannot record the points when the sale becomes paid. The sale itself is not affected. The award is retried so no earned points are lost, and it is never awarded twice for the same sale.

## Postconditions
- For an identified member and a completed, paid sale: exactly one earn entry exists for that sale, and the member's balance has increased by the points earned.
- For a refunded or cancelled sale: a reversing entry exists and the balance no longer includes the points of that sale.
- The member sees the updated balance and history in their Storefront account.
- For a sale without a member: no points entry exists.

## Business Rules
- **BR-1:** A customer with a Storefront account (linked customer record, see UC-026) is a loyalty member. There is no separate enrolment or loyalty login.
- **BR-2:** Points are awarded only on completed and paid sales.
- **BR-3:** Points are earned at an admin-configurable earn rate (e.g., 1 point per EUR 1 spent). Whether the amount includes or excludes VAT is an open question.
- **BR-4:** Points are rounded down to a whole number (assumption, see Open Question 2).
- **BR-5:** A refund or cancellation of a sale reverses the points awarded for it. Whether the balance may become negative is an open question.
- **BR-6:** Points are awarded only to an identified member. A POS sale without member identification and a guest webshop checkout earn no points. Points are not held pending and are not linked afterwards in v1 (see Open Question 5).
- **BR-7:** The points balance is the sum of all points entries for the member. Every change to the balance is a separate, permanent entry that is traceable to a sale; entries are never edited or deleted.
- **BR-8:** A sale awards points at most once.
- **BR-9:** Each member has a personal, unique QR code / member code that identifies them at the POS. It is shown in the Storefront account page.
- **BR-10:** Redemption of points is not part of v1. Points earned in v1 accumulate in the balance and can be used once redemption (UC-034) is introduced.

## Out of Scope (v1)
- Redeeming points for rewards (UC-034).
- A separate loyalty app, loyalty login, email verification, and multi-email linking (UC-031).
- Badges (UC-033) and challenges, including bonus points from challenges (UC-035).
- Bonus or promotional point multipliers (e.g., double-points weekends).
- Partial refunds and credit notes (the system has no refund flow today). Partial reversal of points is future work.
- Different earn rates per channel, product type or customer type.
- Retroactive linking of a sale to a member after completion.
- Backfilling points for purchases made before go-live.
- Push or in-app notifications. The member sees the updated balance in their Storefront account instead.

## Open Questions
1. **Rounding:** Assumed: round down (floor) to whole points per sale. Confirm, or choose round-to-nearest or fractional points.
2. **Negative balance:** If a cancelled sale's points were already spent (once redemption exists), may the balance become negative, or is the reversal capped at zero? Not relevant for v1 until redemption exists, but must be decided before UC-034.
3. **Rate stored per entry:** Should the earn rate be stored with each entry so history stays explainable after a rate change? (Assumed yes; the new rate applies only to later sales.)
4. **Member code format:** Is the typed member code short enough to type reliably, and may a member regenerate it if shared or lost?
5. **POS-registered customers:** A customer created at the POS by quick-register (UC-002 AF-6) may not have a Storefront account. Do they earn points once they register later, and does a customer without an account have a member code?
6. **Admin visibility:** Should admins see a member's balance and history (e.g., in the customer details, UC-039) or adjust points manually? Not included in v1 unless confirmed.

## Implemented By
- Status: **Not yet implemented** — v1 scope is defined but no code exists. The legacy `loyalty` table is unused and is replaced by a points ledger (one entry per earn or reversal).
- Related use cases: UC-021 (POS sale, member identification), UC-025 (webshop checkout, paid order), UC-026 (account page, balance and history), UC-015 (system settings, earn rate configuration may be placed here or in a dedicated loyalty settings page).
