### Story 9.2.2: A POS Sale for a Selected Member Earns Points
**As a** cashier
**I want** the sale of a customer I select to earn that customer's points when the customer is a Storefront member
**So that** members are rewarded for shop purchases

**References:** UC-032 (AF-1, AF-2, AF-3, BR-5, BR-6), UC-021 (AF-11)
**Priority:** Should
**Estimate:** ~3 story points

**Context:**
Uses the existing customer search at UC-021 step 7. Member codes are not needed (see Stories 9.2.3 and 9.2.4).

**Acceptance Criteria:**
- [ ] When the cashier selects a member through the existing customer search, one points entry is recorded on the PAID POS sale
- [ ] Basis, rate, rounding and stored rate follow Story 9.1.3; a POS sale-level discount reduces the basis, and there is no shipping
- [ ] A selected customer without a Storefront account (e.g., quick-registered at the POS) earns nothing (qualification rule of Story 9.1.4)
- [ ] The default Kruze guest customer (id 673) earns nothing
- [ ] A selected B2B client earns nothing (qualification rule of Story 9.1.4), and the sale proceeds as normal
- [ ] A sale without a customer earns nothing
- [ ] A recording failure does not fail the sale
- [ ] A failed award is retried automatically (Story 9.1.5)

**Technical Implementation:**
- Backend: call the award from the POS sale creation in `server/posService.ts` (~404-464, sale created PAID) via `POST /api/pos/sales`; guest customer from `getGuestCustomer` (~285)
- Frontend: none (existing `client/src/components/pos/CustomerSelectionDialog.tsx`)
- Database: none

**Dependencies:** Story 9.1.3, Story 9.1.4, Story 9.1.5. Not dependent on member codes.
