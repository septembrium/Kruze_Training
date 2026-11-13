### Story 9.1.6: See My Points Balance and History
**As a** Storefront member
**I want** to see my points balance and history on my account page
**So that** I know how many points I have and where they came from

**References:** UC-032, UC-026 (AF-5)
**Priority:** Should
**Estimate:** ~3 story points

**Acceptance Criteria:**
- [ ] The account page shows the balance, equal to the sum of all the member's entries
- [ ] The history shows at most 50 entries per page, newest first, and older entries are reachable
- [ ] Each entry shows the date, the channel (webshop or POS), the reference, and the points
- [ ] The reference is the webshop order number for a webshop entry and the sale number for a POS entry
- [ ] Points are shown as positive numbers
- [ ] A member with 0 points sees an empty state explaining how points are earned
- [ ] The balance counts all entries, not only the visible page
- [ ] A member sees only their own balance and history

**Technical Implementation:**
- Backend: `GET /api/account/loyalty` in `server/routes.ts` (isAuthenticated, next to `/api/account/profile`); queries in `server/loyaltyService.ts`; the reference is not the invoice number, which is '0' for non-B2B
- Frontend: new Loyalty tab in `client/src/pages/storefront/Account.tsx` (`TABS` list)
- Database: none (reads the ledger from Story 9.1.3)

**Dependencies:** Story 9.1.3. Must ship in the same release as Story 9.1.3.
