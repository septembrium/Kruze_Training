### Story 9.2.1: See and Regenerate My Member Code
**As a** Storefront member
**I want** to see my personal member code and generate a new one
**So that** the shop can identify me at the POS and I can replace a code that someone else has seen

**References:** UC-032 (BR-8), UC-026 (AF-5 member code part, AF-7)
**Priority:** Should
**Estimate:** ~5 story points

**Acceptance Criteria:**
- [ ] Every account holder with a linked customer record has a member code, issued without any action
- [ ] The code is exactly 8 characters from an alphabet without look-alikes: no 0, O, 1, I or L
- [ ] The code is unique across all members; a collision at generation is retried automatically and the member never sees an error
- [ ] The account page shows the code as a QR code and as text, and both carry the same code
- [ ] Regenerating asks for confirmation and states that the current code stops working immediately
- [ ] Cancelling the confirmation leaves the current code unchanged
- [ ] After confirming, a new 8-character code is shown as QR code and text
- [ ] The old code stops working immediately (Story 9.2.2 treats it as not found); the new code works
- [ ] Points already earned are kept and the balance is unchanged after regenerating
- [ ] Two quick regenerate requests leave exactly one valid code, and the one shown is the one that works
- [ ] A B2B account holder also has a member code (it identifies them, but earns nothing)
- [ ] A customer without a Storefront account (e.g., quick-registered at the POS) has no member code
- [ ] The code is returned only to its own member (taken from the session)
- [ ] The code is generated with a cryptographically secure random source

**Technical Implementation:**
- Backend: new `GET /api/account/member-code` and `POST /api/account/member-code/regenerate` in `server/routes.ts` (isAuthenticated, Zod where there is a body); code generation, uniqueness retry and lookup by code in `server/loyaltyService.ts` (new, created in Story 9.1.3) or `server/customerService.ts`; issue the code when the account is linked (Story 9.1.1) and lazily for already-linked accounts
- Frontend: member code section in the Loyalty tab of `client/src/pages/storefront/Account.tsx`; confirmation via the existing AlertDialog pattern; QR rendering needs a QR library, which is not in `package.json` today (new dependency, run `npm audit` before adding)
- Database: nullable unique `member_code` column on `customers` in `shared/schema.ts` (or a separate member table, decided in the spec), with a unique index; migration via `npx drizzle-kit generate`

**Dependencies:** Story 9.1.1 (linked customer record). Story 9.1.4 is the natural home of the page section but is not a technical blocker.
