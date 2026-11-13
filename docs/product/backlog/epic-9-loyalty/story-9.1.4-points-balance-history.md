### Story 9.1.4: See My Points Balance and History
**As a** Storefront member
**I want** to see my points balance and points history on my account page
**So that** I know how many points I have earned and where they came from

**References:** UC-032, UC-026 (AF-5)
**Priority:** Should
**Estimate:** ~3 story points

**Acceptance Criteria:**
- [ ] The account page shows the member's current balance, equal to the sum of all their points entries
- [ ] The history lists every entry, newest first
- [ ] Each history row shows the date, the channel (webshop or POS), the sale reference, and the points earned
- [ ] Points are shown as positive numbers in v1
- [ ] A member with no entries sees a balance of 0 and an empty state explaining how points are earned (instead of an empty table)
- [ ] After a webshop order becomes paid, the new entry and the updated balance appear on the next page load, with no notification sent
- [ ] A member sees only their own balance and history; the endpoint takes the member from the session, not from a client-supplied customer ID
- [ ] A B2B account holder sees a balance of 0 and the empty state
- [ ] A long history is paginated or limited (e.g., 50 entries per page) and the balance is still the total of all entries, not of the visible page
- [ ] The labels follow the page language (the account page is Dutch)

**Technical Implementation:**
- Backend: new `GET /api/account/loyalty` in `server/routes.ts` (isAuthenticated, next to `/api/account/profile`); balance and history queries in `server/loyaltyService.ts` (new, created in Story 9.1.3), joining the ledger to sales to get channel and sale reference
- Frontend: new Loyalty tab in `client/src/pages/storefront/Account.tsx` (add to the `TABS` list next to profile, orders and top-products; new tab component using `useQuery`)
- Database: no new tables (reads the ledger from Story 9.1.3)

**Dependencies:** Story 9.1.3 (ledger and earning). Must ship in the same release as Story 9.1.3: without it members earn points they cannot see; without 9.1.3 the page has nothing to show.
