### Story 9.1.2: Set the Loyalty Earn Rate
**As an** admin
**I want** to set the loyalty earn rate in system settings
**So that** I control how many points a member earns per euro spent

**References:** UC-032 (BR-2), UC-015 (AF-6, BR-7)
**Priority:** Should
**Estimate:** ~2 story points

**Acceptance Criteria:**
- [ ] The Settings page has a loyalty section showing the current earn rate in points per EUR 1
- [ ] The admin can save a new rate and sees the saved value after a reload
- [ ] A rate of zero, a negative value, or a non-number is rejected with a clear message, and the stored rate does not change
- [ ] A fractional rate (e.g., 0.5) is accepted
- [ ] The rate is seeded at 1 point per EUR 1, so the section shows 1 on a fresh system and earning (Story 9.1.3) works without this story
- [ ] A new rate applies only to sales that become paid after the change; points already awarded are not recalculated
- [ ] Only an admin can read or change the rate; the server validates the value, not only the form
- [ ] One global rate applies; there is no per-channel, per-product or per-customer-type rate

**Technical Implementation:**
- Backend: new `GET`/`POST /api/settings/loyalty-params` in `server/routes.ts` (isAdmin, Zod validation), following `/api/settings/shipping-params`; read/write through `parameterService` (`server/parameterService.ts`) using a new parameter such as `loyalty_earn_rate`; positive-number validation
- Frontend: new Loyalty tab in `client/src/pages/admin/Settings.tsx` (same pattern as the Shipping tab and its `saveParamsMutation`)
- Database: seed one row in `parameter_table` (`parameterName`/`parameterValue`) with value `1` through a migration

**Dependencies:** None (9.1.3 reads the seeded default and works without this story)
