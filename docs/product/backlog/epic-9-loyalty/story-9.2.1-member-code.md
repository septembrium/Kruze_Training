### Story 9.2.1: See and Regenerate My Member Code
**As a** Storefront member
**I want** to see my member code and replace it with a new one
**So that** the shop can identify me, and I can replace a code someone else has seen

**References:** UC-032 (BR-8), UC-026 (AF-5 member code part, AF-7)
**Priority:** Should
**Estimate:** ~5 story points

**Acceptance Criteria:**
- [ ] Every member has a member code of 8 characters
- [ ] The code contains no 0, O, 1, I or L
- [ ] The code alphabet is case-insensitive
- [ ] Codes are unique across members
- [ ] The Loyalty tab shows the code as a QR code and as text, both carrying the same code
- [ ] Regenerating asks for confirmation and states that the current code stops working immediately
- [ ] Cancelling the confirmation leaves the code unchanged
- [ ] After confirming, the new code is shown as QR code and text
- [ ] After regeneration, the member code lookup does not find the old code and finds the new code (tested at service level)
- [ ] After two regenerations, only the last confirmed code works
- [ ] Points and balance are unchanged after regenerating
- [ ] A B2B account holder also has a member code
- [ ] A member sees only their own code

**Technical Implementation:**
- Backend: `GET /api/account/member-code`, `POST /api/account/member-code/regenerate` in `server/routes.ts`; generation (cryptographically secure random, automatic retry on collision) and lookup in `server/loyaltyService.ts`
- Frontend: code section in the Loyalty tab of `client/src/pages/storefront/Account.tsx` (tab from Story 9.1.6); QR rendering needs a new library (run `npm audit` first)
- Database: unique member code per member (shape in the spec)

**Dependencies:** Story 9.1.1, Story 9.1.6 (owns the Loyalty tab)
