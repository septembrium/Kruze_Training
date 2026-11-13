### Story 9.1.1: Link Storefront Account to a Customer Record
**As a** Storefront member
**I want** to be recognised as the same customer I already am in the shop, from my first sign-in
**So that** every purchase I make while logged in is attributed to me and can earn points

**References:** UC-032, UC-026 (AF-6)
**Priority:** Should
**Estimate:** ~3 story points

**Acceptance Criteria:**
- [ ] At the first sign-in with a confirmed email, the account is linked to the existing customer record with the same email address (case and surrounding spaces ignored)
- [ ] With several matching records, the record with the most recent sale is linked
- [ ] With several matching records and none with a sale, the most recently added record is linked
- [ ] With no matching record, a new customer record is added and linked
- [ ] A matching record that is a B2B client is linked (it earns nothing, see Story 9.1.4)
- [ ] An account whose email is not yet confirmed is not linked and is not a member; it is linked at the first sign-in after confirmation
- [ ] Signing in again does not relink the account and does not create a duplicate customer record
- [ ] Two simultaneous first sign-ins result in one link and at most one new record
- [ ] An account that already saved its profile before go-live is linked to its existing record at its next sign-in, without a duplicate
- [ ] A confirmed member who never opened the profile page has a linked customer record at their first logged-in checkout
- [ ] Saving the profile for the first time does not create a second customer record for a linked account

**Technical Implementation:**
- Backend: link-or-create in `server/customerService.ts` (next to `updateAccountProfile`), invoked from the sign-in hooks in `server/supabaseAuth.ts` that call `storage.upsertUser` (~lines 88, 133, 304); email-confirmed flag comes from Supabase; most-recent-sale lookup on `sales`
- Frontend: none
- Database: stored user-to-customer link (shape in the spec); migration links existing accounts lazily at sign-in, no bulk backfill

**Dependencies:** None
