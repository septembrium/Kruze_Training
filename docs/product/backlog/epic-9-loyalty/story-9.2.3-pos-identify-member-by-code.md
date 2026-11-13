### Story 9.2.3: Identify a Member by Typing Their Member Code
**As a** cashier
**I want** to identify a member by typing their member code
**So that** I select the right customer quickly and the sale earns their points

**References:** UC-032 (AF-1, EF-1), UC-021 (AF-11)
**Priority:** Should
**Estimate:** ~2 story points

**Acceptance Criteria:**
- [ ] A valid code selects the matched customer and shows the customer's name for confirmation
- [ ] The code is accepted in any letter case, and surrounding spaces are ignored
- [ ] The selected member is treated like a customer chosen through search, so customer-specific prices apply (UC-021 step 8)
- [ ] An unknown code shows a clear message, and the cashier can retry
- [ ] After an unknown code, the cashier can continue the sale without a member
- [ ] A regenerated (old) code gives the same message as an unknown code
- [ ] A member code of a B2B client selects that customer, and the sale proceeds without points
- [ ] Lookup is available to authenticated POS users only
- [ ] Lookup is limited to 10 per minute per user, and further attempts get a clear message
- [ ] The lookup returns only the customer's name and customer id

**Technical Implementation:**
- Backend: `GET /api/pos/customers/by-member-code/:code` in `server/routes.ts` (isAuthenticated, `express-rate-limit` already in the project); lookup in `server/posService.ts` using Story 9.2.1
- Frontend: code entry in `client/src/components/pos/CustomerSelectionDialog.tsx`
- Database: none

**Dependencies:** Story 9.2.1, Story 9.2.2
