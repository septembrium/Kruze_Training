### Story 9.2.2: Identify a Member at the POS and Earn Points
**As a** cashier
**I want** to identify a loyalty member by scanning their QR code or typing their member code
**So that** the member earns points on their in-shop purchase

**References:** UC-032 (AF-1, AF-2, AF-3, EF-1, BR-5 to BR-7), UC-021 (AF-11), UC-007 (customer pricing)
**Priority:** Should
**Estimate:** ~5 story points

**Context:**
Identifying a member is equivalent to selecting the customer at UC-021 step 7. A POS sale is created as PAID (`server/posService.ts`), so points are awarded at creation using the earning logic and ledger from Story 9.1.3.

**Acceptance Criteria:**
- [ ] The cashier can enter a member code by scanning the QR code or by typing the 8 characters
- [ ] A valid code selects the matched customer and shows the customer's name so the cashier can confirm it
- [ ] The selected member is treated exactly like a customer selected through search: customer-specific prices (UC-007) and the existing pricing rules apply
- [ ] An unknown code shows an error; the cashier can retry or continue the sale without a member
- [ ] A code that was regenerated (Story 9.2.1) is treated as unknown
- [ ] Typed codes are accepted regardless of letter case and surrounding spaces
- [ ] When the sale is completed, one points entry is recorded for the member, linked to the sale, with the same earn basis, rate, rounding and stored rate as in Story 9.1.3 (products incl. VAT after discounts, excl. shipping)
- [ ] A member selected through the normal customer search (not the code) also earns, if the customer is a loyalty member
- [ ] A customer who is not a loyalty member (e.g., quick-registered at the POS without a Storefront account) can be selected, the sale proceeds, and no points are earned
- [ ] The default Kruze company customer earns nothing
- [ ] A B2B client can still be selected or identified by code, the sale proceeds as normal, and nothing is earned
- [ ] A sale with no customer earns nothing
- [ ] A sale that was completed without an identified member cannot be linked to a member afterwards
- [ ] A failure to record points does not fail or roll back the POS sale; the points are recorded later and never twice for the same sale
- [ ] Code lookup is limited to authenticated POS users and is rate limited, so codes cannot be guessed in bulk
- [ ] The lookup returns only what the cashier needs (name and customer ID), not other customer personal data

**Technical Implementation:**
- Backend: new `GET /api/pos/customers/by-member-code/:code` in `server/routes.ts` (isAuthenticated, next to `/api/pos/customers/search`; rate limited); lookup in `server/posService.ts` (next to `searchCustomers`, `getGuestCustomer`, `registerCustomer`) using the code lookup from Story 9.2.1; call `loyaltyService.awardPointsForSale` (Story 9.1.3) after the sale is created PAID in `POST /api/pos/sales` / `server/posService.ts` (~lines 404-464); membership and B2B checks live in the shared earning logic
- Frontend: member-code entry (typed and QR scan, camera access needed for scanning) in `client/src/components/pos/CustomerSelectionDialog.tsx`, feeding the same customer selection state used by `client/src/pages/pos/POSPage.tsx`; a QR scanner library is not in `package.json` today (new dependency, run `npm audit` before adding); typed entry works without a camera
- Database: none new (uses `member_code` from Story 9.2.1 and the ledger from Story 9.1.3)

**Dependencies:** Story 9.2.1 (member code), Story 9.1.3 (earning logic and ledger). Story 9.1.4 lets the member see the result but is not a blocker.
