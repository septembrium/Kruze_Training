### Story 9.2.4: Identify a Member by Scanning Their QR Code
**As a** cashier
**I want** to identify a member by scanning their QR code
**So that** I don't have to type the code

**References:** UC-032 (AF-1, EF-1), UC-021 (AF-11)
**Priority:** Should
**Estimate:** ~2 story points

**Acceptance Criteria:**
- [ ] Scanning a member's QR code with the device camera gives the same result as typing that code (Story 9.2.3)
- [ ] Scanning with a USB scanner that types like a keyboard gives the same result
- [ ] A scanned unknown code shows the same message and options as in Story 9.2.3
- [ ] A QR code that is not a member code is treated as an unknown code
- [ ] When camera access is denied, the cashier can still type the code

**Technical Implementation:**
- Backend: none (reuses the lookup of Story 9.2.3)
- Frontend: scan input in `client/src/components/pos/CustomerSelectionDialog.tsx`; a keyboard-wedge scanner may need no library; a camera scan needs a new library (none in `package.json` today, run `npm audit` first)
- Database: none

**Dependencies:** Story 9.2.3
