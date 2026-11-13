### Story 9.1.1: Link Storefront Account to a Customer Record on First Sign-In
**As a** Storefront account holder
**I want** my account linked to my customer record the first time I sign in
**So that** I am a loyalty member from my first visit and my purchases can earn points

**References:** UC-032, UC-026 (AF-6)
**Priority:** Should
**Estimate:** ~3 story points

**Context:**
Today a customer record is only created on the first profile save (`updateAccountProfile` in `server/customerService.ts`, see UC-026 Implementation Notes), and the user and customer are linked by email address with no foreign key. Loyalty needs a durable link from the first sign-in onward.

**Acceptance Criteria:**
- [ ] At the first sign-in of an account, the system links the account to an existing customer record whose email matches the account email
- [ ] When several customer records match, the record with the most recent sale is linked
- [ ] When several records match and none has a sale, the most recently added record is linked
- [ ] When no record matches, a new customer record is added and linked
- [ ] A matched record that is a B2B client (`customers.b2bClient`) is linked, and the account holder is not blocked from signing in
- [ ] The link is stored on the account, so later steps (checkout, balance, member code) use the linked record and no longer depend on matching by email
- [ ] The link is idempotent: signing in again does not relink the account and does not create a duplicate customer record
- [ ] Two simultaneous first sign-ins of the same account result in one link and at most one new customer record
- [ ] An account that has already saved its profile before this story ships (customer record exists, matched by email) is linked at its next sign-in without creating a duplicate
- [ ] The first profile save no longer creates a second customer record when the account is already linked
- [ ] Email matching ignores letter case and surrounding spaces

**Technical Implementation:**
- Backend: link step in the sign-in/profile-load path (`server/supabaseAuth.ts` / `GET /api/account/profile` in `server/routes.ts`); new link-or-create method in `server/customerService.ts` (next to `updateAccountProfile`); lookup of most recent sale per matching customer; adjust `updateAccountProfile` to use the stored link
- Frontend: none (invisible to the user)
- Database: store the link between the user and `customers.id` (new column on `users` or equivalent; exact shape in the spec); migration backfills no points, only links on next sign-in

**Dependencies:** None
