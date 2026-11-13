### Story 9.1.4: Only the Logged-In Member Earns at Checkout
**As a** Storefront member
**I want** my webshop order attributed to my own customer record whatever email I type at checkout
**So that** my points always reach me and never someone else

**References:** UC-032 (BR-5, BR-6), UC-025 (AF-5)
**Priority:** Should
**Estimate:** ~2 story points

**Acceptance Criteria:**
- [ ] The sale of a logged-in member is linked to the member's own customer record, whatever email is typed in the order details
- [ ] A guest checkout earns nothing, even when its email matches a member's customer record
- [ ] A guest sale is not linked to a member's record
- [ ] The webshop guest record (id 673) is never a member
- [ ] The sale of a B2B client (`customers.b2bClient`) earns nothing, even with a Storefront account

**Technical Implementation:**
- Backend: `findOrCreateCustomer` in `server/checkoutService.ts` (~570) already returns the guest id for `request.isGuest` (~572); the gap is that a logged-in checkout matches by email (~578) instead of using the linked record from Story 9.1.1; `POST /api/checkout/create` (`server/routes.ts`) has no auth middleware, so the session user must be read optionally; the member/B2B check goes in `server/loyaltyService.ts` (Story 9.1.3)
- Frontend: none
- Database: none

**Dependencies:** Story 9.1.1, Story 9.1.3
