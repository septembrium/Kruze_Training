### Story 9.1.2: Set the Loyalty Earn Rate
**As an** admin
**I want** to set the loyalty earn rate in system settings
**So that** I control how many points members earn per euro spent

**References:** UC-032 (BR-2), UC-015 (AF-6, BR-7)
**Priority:** Should
**Estimate:** ~2 story points

**Acceptance Criteria:**
- [ ] The Settings page shows the current earn rate in points per EUR 1; a fresh system shows 1
- [ ] The admin can save a new rate and sees it after a reload
- [ ] A rate of 0.5 is accepted
- [ ] A rate with more than 2 decimals is rejected with a clear message
- [ ] A rate of zero, a negative value, a non-number or a value above 100 is rejected with a clear message, and the stored rate is unchanged
- [ ] Only an admin can read or change the rate, and the server validates the value

**Technical Implementation:**
- Backend: `GET`/`POST /api/settings/loyalty-params` in `server/routes.ts` (isAdmin, Zod), modelled on `/api/settings/shipping-params`; stored via `parameterService` in `parameter_table`
- Frontend: Loyalty tab in `client/src/pages/admin/Settings.tsx`, modelled on the Shipping tab
- Database: seed `loyalty_earn_rate` = 1 following the `INSERT ... ON CONFLICT DO NOTHING` pattern in `server/migrate.ts` (~line 68)

**Dependencies:** None. Story 9.1.3 reads the seeded default and does not need this story; which sales a new rate applies to is owned by Story 9.1.3.
