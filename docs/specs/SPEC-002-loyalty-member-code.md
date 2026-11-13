# SPEC-002: Loyalty Member Code - Display, Regenerate, POS Lookup by Code and QR

Status: Draft for implementation (file plan confirmed by the product owner).
Depends on SPEC-001 (`docs/specs/SPEC-001-loyalty-earn-points.md`): the account-to-customer link (`users.customer_id`), `loyaltyService`, the Loyalty tab in `Account.tsx`, `GUEST_CUSTOMER_ID`, the POS award hook (story 9.2.2, which is in SPEC-001) and the test tooling.

## References
- **Use case:** UC-032 (BR-5, BR-6, BR-8, AF-1, AF-2, AF-3, EF-1), UC-026 AF-5 (member code part) and AF-7, UC-021 AF-11.
- **Epic:** `docs/product/backlog/epic-9-loyalty/epic.md`
- **Stories (all review-PASSED):**
  - 9.2.1 `docs/product/backlog/epic-9-loyalty/story-9.2.1-member-code.md`
  - 9.2.3 `docs/product/backlog/epic-9-loyalty/story-9.2.3-pos-identify-member-by-code.md`
  - 9.2.4 `docs/product/backlog/epic-9-loyalty/story-9.2.4-pos-identify-member-by-qr.md`
- **Reference implementation:** `server/roastService.ts`.

## Scope

### In scope
- `users.member_code` column, generation (cryptographically secure, collision retry), regeneration and lookup (9.2.1).
- Member-code section in the Storefront Loyalty tab: QR code and text, regenerate with AlertDialog confirmation (9.2.1).
- `POST /api/pos/customers/by-member-code` and the code entry in the POS customer dialog (9.2.3).
- USB keyboard-wedge scanning and camera QR scanning in the POS customer dialog (9.2.4).
- Per-user rate limiters for the lookup (10 per minute) and regenerate (5 per minute).
- Excluding the member-code routes from response-body logging in `server/index.ts`.
- New dependencies `qrcode.react` and `qr-scanner`.

### Out of scope
- Earning, ledger, retry, link logic: SPEC-001.
- A distributed rate-limit store (the in-memory store is per instance; see Risks).
- Printing a card or wallet pass; emailing the code; admin viewing or resetting a member's code.
- Member code for customers without a Storefront account (they have none by definition, BR-6).

## Decisions fixed by the product owner
- Lookup is `POST /api/pos/customers/by-member-code` with the code in the JSON body, not in the URL. Story 9.2.3's technical pointer said `GET /api/pos/customers/by-member-code/:code`; that pointer is updated to the POST route (follow-up task T-1 below). Reason: a path parameter would put the code into access logs (`server/index.ts:30,42` logs `req.path`).
- Member-code routes are excluded from response-body logging in `server/index.ts`.
- Rate limiter 10 per minute per user, keyed on `req.user.id`, mounted after `isAuthenticated`. Regenerate 5 per minute. The in-memory store is per instance.
- Libraries `qrcode.react` (rendering) and `qr-scanner` (camera), both subject to `npm audit` and a licence check before install: **unverified, verify before install**.
- Alphabet and length: 8 characters, no 0, O, 1, I, L.

## Schema Changes

### Checklist
- [ ] `users.memberCode` (varchar 8, unique, nullable) in `shared/schema.ts`
- [ ] Member-code constants and Zod schema in `shared/schema.ts`
- [ ] `npm run db:push` (no setup SQL needed; the unique constraint is expressible in Drizzle)

### Drizzle definition (`shared/schema.ts`, `users` table)
SPEC-001 already changed `users`; add one column:
```ts
export const users = pgTable("users", {
  // ... existing columns and customerId (SPEC-001) ...
  // Loyalty (SPEC-002): 8-character member code shown as QR and text. Null until first requested.
  memberCode: varchar("member_code", { length: 8 }).unique(),
  // ... createdAt, updatedAt ...
});
```
PostgreSQL allows many `NULL` values in a unique column, so members without a code yet do not conflict. The code lives on `users` (not `customer`) because only a customer with a Storefront account has a code (BR-6), and the column rides with the existing link (`users.customer_id`, SPEC-001). `storage.upsertUser` does not overwrite it (`storage.ts:435-446` sets only name, image and role).

Same primary-key convention note as SPEC-001: no new table is added here, so no new PK.

### Member-code constants and Zod schema (`shared/schema.ts`)
```ts
// 23 letters (A-Z without I, L, O) + 8 digits (2-9) = 31 characters. 31^8 is about 8.5e11 combinations.
export const MEMBER_CODE_ALPHABET = "ABCDEFGHJKMNPQRSTUVWXYZ23456789";
export const MEMBER_CODE_LENGTH = 8;
export const MEMBER_CODE_PATTERN = /^[ABCDEFGHJKMNPQRSTUVWXYZ2-9]{8}$/;

export function normalizeMemberCode(raw: string): string {
  return raw.trim().toUpperCase();
}

export const memberCodeLookupSchema = z.object({
  code: z.string().max(64, "Member code is too long"), // length capped only to bound the input; shape is checked after normalising
});
export type MemberCodeLookupInput = z.infer<typeof memberCodeLookupSchema>;
```
Anything that does not match `MEMBER_CODE_PATTERN` after normalising (too short, containing 0, O, 1, I or L, a URL from a foreign QR code) is treated exactly like an unknown code (story 9.2.4 AC 4).

### Migration procedure
`npm run db:push` and review the statements: they should only add `users.member_code` and its unique constraint (plus the SPEC-001 changes if not yet applied). No setup SQL. There is no data to backfill: codes are generated lazily on the member's first view of the Loyalty tab (see Backend).

## Backend Changes

### Storage methods (`server/storage.ts`)
```ts
export interface MemberLookupRow { customerId: number; firstName: string; lastName: string; b2bClient: boolean; b2bPrices: boolean }

getMemberCode(userId: string): Promise<{ customerId: number | null; memberCode: string | null } | undefined>;
setMemberCodeIfNull(userId: string, code: string): Promise<string | undefined>; // UPDATE ... SET member_code = $code WHERE id = $userId AND member_code IS NULL RETURNING member_code
replaceMemberCode(userId: string, code: string): Promise<string>;               // UPDATE users SET member_code = $code, updated_at = now() WHERE id = $userId RETURNING member_code
findMemberByCode(code: string): Promise<MemberLookupRow | undefined>;
```
`setMemberCodeIfNull` and `replaceMemberCode` let the unique-violation error (`pg` code `23505`) propagate; the service retries. `findMemberByCode` is one SQL query:
```sql
SELECT c.id, c.first_name, c.last_name, c.b2b_client, c.b2b_prices
FROM users u JOIN customer c ON c.id = u.customer_id
WHERE u.member_code = $1 AND u.customer_id IS NOT NULL AND c.id <> 673
```
(673 comes from `GUEST_CUSTOMER_ID`). A customer without an account has no `users` row, so it cannot match. The old code stops matching the moment `replaceMemberCode` commits because the column holds exactly one code per member.

### `server/loyaltyService.ts` additions
```ts
export class MemberCodeError extends Error {
  constructor(public readonly code: "NOT_A_MEMBER" | "CODE_GENERATION_FAILED", message: string) { super(message); }
}
export interface MemberLookupResult { id: number; firstName: string; lastName: string; b2bClient: boolean; b2bPrices: boolean }

generateMemberCode(): string;                                          // pure apart from crypto
async getOrCreateMemberCode(userId: string): Promise<string>;
async regenerateMemberCode(userId: string): Promise<string>;
async findMemberByCode(rawCode: string): Promise<MemberLookupResult | undefined>;
```
- `generateMemberCode()` builds 8 characters, each `MEMBER_CODE_ALPHABET[crypto.randomInt(0, MEMBER_CODE_ALPHABET.length)]`. `crypto.randomInt` is unbiased and cryptographically secure (`import { randomInt } from "crypto"`); `Math.random` is never used.
- `getOrCreateMemberCode(userId)`: `getMemberCode`; if the user has no `customerId` throw `MemberCodeError("NOT_A_MEMBER", "Only members with a confirmed email can have a member code")`; if a code exists return it; otherwise generate and `setMemberCodeIfNull`, retrying on unique violation up to 5 times (`CODE_GENERATION_FAILED` afterwards). When `setMemberCodeIfNull` returns `undefined` (another request set it first) re-read and return the stored code. A B2B account holder gets a code like any member (story 9.2.1 AC 13).
- `regenerateMemberCode(userId)`: requires a link (same `NOT_A_MEMBER` rule); generates and `replaceMemberCode`, retrying on unique violation up to 5 times. Points, ledger and balance are untouched because nothing in the ledger references the code (stories AC 12). After two regenerations only the last code resolves (single column).
- `findMemberByCode(rawCode)`: `normalizeMemberCode`, return `undefined` when `MEMBER_CODE_PATTERN` fails (no database access, so malformed input does not cost a query), else `storage.findMemberByCode`.
- No transactions are needed: each operation is a single atomic statement; races are resolved by the unique constraint and the `member_code IS NULL` guard.

### `server/posService.ts` addition
```ts
async lookupMemberByCode(rawCode: string): Promise<MemberLookupResult | undefined> {
  return loyaltyService.findMemberByCode(rawCode);
}
```
Per story 9.2.3 the lookup lives in `posService` and uses story 9.2.1. A B2B member is returned like any member (the sale proceeds without points through the SPEC-001 qualification rule). Pricing: the POS looks products up with a `customerId` (`posService.ts:202-220` uses customer-specific prices by `customerId`), so selecting the matched customer gives the same prices as selecting through search. Unverified: how the variants route derives `isB2B` for the selected customer (the lines above line 200 and the route at `routes.ts:2436` were not read).

### Rate limiters (`server/security.ts`)
`express-rate-limit` is already a dependency (`package.json`: `^8.2.1`) and used in `server/index.ts:118-131`.
```ts
import rateLimit from "express-rate-limit";

export const memberCodeLookupLimiter = rateLimit({
  windowMs: 60 * 1000,
  limit: 10,                                    // v8 option name; index.ts uses the older alias `max`
  keyGenerator: (req) => req.user?.id ?? "anonymous",
  standardHeaders: true,
  legacyHeaders: false,
  message: { error: "Too many member code lookups. Wait a minute and try again.", code: "RATE_LIMITED" },
});

export const memberCodeRegenerateLimiter = rateLimit({
  windowMs: 60 * 1000,
  limit: 5,
  keyGenerator: (req) => req.user?.id ?? "anonymous",
  standardHeaders: true,
  legacyHeaders: false,
  message: { error: "Too many member code changes. Wait a minute and try again.", code: "RATE_LIMITED" },
});
```
Mounted after `isAuthenticated` so `req.user.id` exists. Keying on the user id (not the IP) means a shared shop IP does not pool all cashiers. Every attempt counts, including unknown codes. The store is in memory: with several Cloud Run instances each instance counts separately, so the real ceiling is 10 per minute per instance, and counters reset on a cold start. Unverified: the exact v8 `keyGenerator` and option names; check the installed package types.

### Routes (`server/routes.ts`)
| Method and path | Middleware | Validation | Behaviour |
|---|---|---|---|
| `GET /api/account/member-code` | `isAuthenticated` | none | `{ memberCode: string }`; 403 `{ error, code: "NOT_A_MEMBER" }` when the account is not linked |
| `POST /api/account/member-code/regenerate` | `isAuthenticated`, `memberCodeRegenerateLimiter` | `z.object({}).strict()` | `{ memberCode: string }` (the new code); 403 when not a member |
| `POST /api/pos/customers/by-member-code` | `isAuthenticated`, `memberCodeLookupLimiter` | `memberCodeLookupSchema` | 200 `{ id, firstName, lastName, b2bClient, b2bPrices }`; 404 `{ error: "Member code not found. Check the code and try again, or continue without a member.", code: "MEMBER_CODE_NOT_FOUND" }` for unknown, old (regenerated), malformed and non-member codes alike (identical response); 400 on a body without `code`; 429 from the limiter |

Route registration order: `POST /api/pos/customers/by-member-code` does not collide with the existing `GET /api/pos/customers/search` and `/guest` and `POST /register` routes (`routes.ts:2450,2466,2478`).

The response of the lookup is deliberately limited to the identifiers the POS needs to behave as with a searched customer: `id` and name (as required by story 9.2.3), plus `b2bClient` and `b2bPrices` booleans because the existing `POSCustomer` type requires them (`client/src/pages/pos/POSPage.tsx:31-39`). It never includes email, phone, address, birthday, the member code or points. This adds two non-PII flags to the literal wording "only name and customer id"; see Open Item OI-1.

### `server/index.ts`: exclude member-code routes from response-body logging
The logger at `server/index.ts:28-52` captures every JSON response and appends it to the log line (truncated at 80 characters, lines 43-47). `GET /api/account/member-code` returns `{"memberCode":"ABCD2345"}`, which would be logged in full, and the lookup response carries a customer name. Add before the middleware:
```ts
const NO_BODY_LOG_PREFIXES: readonly string[] = ["/api/account/member-code", "/api/pos/customers/by-member-code"];
```
and capture the body only when `!NO_BODY_LOG_PREFIXES.some((p) => path.startsWith(p))`. The request line (method, path, status, duration) is still logged. The member code never appears in a URL (the lookup uses the body), so the path logging is safe.

## Frontend Changes

Design: `docs/design/design_guidelines.md` (touch targets: large buttons `min-h-16` at line 146 and minimum 44x44 px on POS at line 234, found by targeted search; the full file was not read), `.claude/rules/frontend.md`. Dutch copy on the Storefront, English copy in the POS (the existing POS dialog is English, `CustomerSelectionDialog.tsx:106,122,131`). Delete-style confirmations use the shadcn `AlertDialog` with an `AlertTriangle` icon (`.claude/skills/_shared/references/coding-standards.md`). Unverified: that the shadcn component file `client/src/components/ui/alert-dialog.tsx` exists (the Radix package `@radix-ui/react-alert-dialog` is in `package.json`).

### `client/src/components/storefront/MemberCodeSection.tsx` (new) and `Account.tsx` (story 9.2.1)
- `MemberCodeSection` is rendered inside the `LoyaltyTab` created by SPEC-001, below the balance.
- `useQuery<{ memberCode: string }>` on `["/api/account/member-code"]`. Shows a skeleton while loading; shows a friendly message on 403 ("Je hebt een bevestigd e-mailadres nodig om een ledencode te krijgen.").
- Shows the code as a QR code and as text, both from the same `memberCode` string:
  - `<QRCodeSVG value={memberCode} size={192} level="M" />` from `qrcode.react` (unverified: API names, check the installed types). The QR content is the raw 8-character code, not a URL, so a POS wedge scanner types exactly the code.
  - Text in `font-mono` (JetBrains Mono, data font) with letter-spacing, upper case, `data-testid="member-code-text"`. A "Kopieer" button is not included in this spec.
- **Regenerate** button (`data-testid="button-regenerate-member-code"`) opens an `AlertDialog` with an `AlertTriangle` icon (`text-destructive`), title "Nieuwe ledencode aanmaken?", body "Je huidige code werkt meteen niet meer. Je punten blijven behouden.", actions "Annuleren" (cancel, closes with no request and the code unchanged) and "Nieuwe code aanmaken" (destructive variant). On confirm: `useMutation` `POST /api/account/member-code/regenerate`, then the query cache is set to the returned code so the new QR and text appear. Errors (including 429) show a toast with the server message.
- The same component is used for B2B account holders (no B2B condition).

### `client/src/components/pos/CustomerSelectionDialog.tsx` (stories 9.2.3 and 9.2.4)
Add a "Member code" section between "Guest Checkout" (line 111) and "Search Existing Customer" (line 129):
- Input (`data-testid="input-member-code"`, `autoCapitalize="characters"`, `autoComplete="off"`, `spellCheck={false}`, class `uppercase font-mono`, height at least `min-h-11`) plus "Find member" button (`min-h-16`, `data-testid="button-find-member"`). Enter in the field submits, which is how a USB keyboard-wedge scanner (types the code, then Enter) works with no library (story 9.2.4 AC 2).
- Submit: `normalizeMemberCode` (trim, uppercase). If the result does not match `MEMBER_CODE_PATTERN`, show the unknown-code message without calling the API (this also covers a QR that is not a member code, story 9.2.4 AC 4). Otherwise `POST /api/pos/customers/by-member-code` with `{ code }`.
- **Match:** a confirmation card shows the matched name ("Member found: Jan Peeters") with "Select" (`min-h-16`) and "Cancel". "Select" calls `onSelectCustomer` with a `POSCustomer` built from the response (`id`, `firstName`, `lastName`, `b2bClient`, `b2bPrices`), which is treated like a customer chosen through search, so the existing customer-specific pricing applies. The dialog closes through the existing parent handler (`POSPage.tsx:294-295`).
- **Unknown, old or malformed code:** message "Member code not found. Check the code and try again, or continue without a member." with the input kept focused and a "Continue without member" button (`min-h-11`) that selects nothing and returns to the dialog list (the guest card remains available). A regenerated old code shows exactly the same message (the server returns the identical 404).
- **429:** show the server message "Too many member code lookups. Wait a minute and try again." and disable the button for the remainder of the window is not required; the message is enough. Unverified: how `apiRequest` in `@/lib/queryClient` formats thrown errors (the file was not read; the existing dialog shows `error.message`, `CustomerSelectionDialog.tsx:64-69`). Verify and parse the status accordingly.
- State handled with `useState` and `useMutation`; no new global state.

### `client/src/components/pos/MemberCodeScanner.tsx` (new, story 9.2.4)
- Props: `onCode: (code: string) => void`, `onUnavailable: (reason: "denied" | "no-camera") => void`.
- Starts the camera with `qr-scanner` (`new QrScanner(videoEl, (result) => onCode(result.data), { returnDetailedScanResult: true, highlightScanRegion: true })`, `await scanner.start()`, `scanner.stop()` and `scanner.destroy()` on unmount; unverified: API names, check the installed types). The library is loaded with a dynamic `import("qr-scanner")` inside the component so it stays out of the main POS bundle (this is client code; the "no dynamic imports" rule applies to routes).
- A decoded value goes through the same submit path as typed input (normalise, pattern check, lookup), so a scan gives the same result as typing (story 9.2.4 AC 1) and a scanned unknown code shows the same message and options (AC 3).
- If `start()` rejects (permission denied) or `QrScanner.hasCamera()` is false, the scanner is replaced by the text "Camera not available. Type the member code instead." and the typed input stays usable (AC 5). The camera is a toggle button "Scan with camera" (`min-h-16`) so the device camera is not opened unless the cashier asks.

### Dependencies (`package.json`)
- `qrcode.react` (QR rendering) and `qr-scanner` (camera decoding). **Unverified - verify before install:** run `npm audit`, check the current maintained version, licence (both are expected to be MIT, from general knowledge, not verified in this session) and bundle size. If `qr-scanner` fails the audit, fall back to the keyboard-wedge-only approach for 9.2.4 and raise a change request (the camera acceptance criteria would then not be met).

## Security Considerations

- **Auth and roles.**
  - Member code view and regenerate: authenticated Storefront user, only for the caller's own account (`req.user.id`); no user id is accepted from the client, so a member sees only their own code.
  - POS lookup: any authenticated user (the POS uses `isAuthenticated`, as the other `/api/pos/*` routes, `routes.ts:2423-2579`); there is no separate cashier role in the code I read.
- **Code strength and brute force.** About 8.5e11 codes (31^8), generated with `crypto.randomInt`. At 10 attempts per minute per user a guess has negligible chance. The lookup returns only a name and non-PII flags, never email, phone, address, member code or points.
- **Enumeration.** Unknown, old and malformed codes return an identical 404 body, so the response does not reveal whether a code once existed.
- **Input validation (Zod on every POST).** `memberCodeLookupSchema` on the lookup; `z.object({}).strict()` on regenerate; shape check against `MEMBER_CODE_PATTERN` before any query.
- **Rate limiting.** Lookup 10 per minute per user, regenerate 5 per minute per user, both keyed on `req.user.id`, mounted after `isAuthenticated`. The in-memory store is per instance (see Risks). The global `/api` limiter stays (`server/index.ts:133`).
- **Logging.** The code is never in a URL; response bodies of the member-code routes are excluded from the logger. No PII in logs.
- **Customer 673 and non-members.** `findMemberByCode` joins through `users.customer_id` and excludes 673, so those can never be found even if a code string were set.
- **Camera.** Camera access is requested only when the cashier taps "Scan with camera"; no frames leave the browser (decoding is client-side in `qr-scanner`; unverified from general knowledge).
- **Dependencies.** `npm audit` and licence check before adding `qrcode.react` and `qr-scanner` (`.claude/rules/security.md`).

## Test Plan (TDD - tests are written BEFORE implementation)

Conventions as in SPEC-001 (Vitest, `happy-dom`, mocked `./db` and `./storage`; supertest and `@testing-library/react` are added by SPEC-001; annotate tests with `// Story: 9.2.x | UC: UC-032`). Camera and QR rendering libraries are mocked in component tests (`vi.mock("qr-scanner")`, `vi.mock("qrcode.react")`).

Test files (exact paths):
- `server/memberCode.test.ts` (new): generation and service logic
- `server/posService.test.ts` (created in SPEC-001; add `lookupMemberByCode` cases)
- `server/routes.memberCode.test.ts` (new, supertest; same harness caveat as SPEC-001 OI-3)
- `server/security.test.ts` (existing; add limiter configuration cases)
- `server/loyalty.integration.test.ts` (created in SPEC-001, gated on `TEST_DATABASE_URL`; add uniqueness cases)
- `client/src/pages/storefront/MemberCodeSection.test.tsx` (new)
- `client/src/components/pos/CustomerSelectionDialog.test.tsx` (new)
- `client/src/components/pos/MemberCodeScanner.test.tsx` (new)

### Acceptance Criteria to Test Cases

#### Story 9.2.1: See and Regenerate My Member Code (14 criteria)
| # | Acceptance Criterion | Type | Test Description | File |
|---|---|---|---|---|
| 1 | Every member has a code of 8 characters | service | `getOrCreateMemberCode` for a linked user without a code stores and returns a string of length 8; a second call returns the same code | `server/memberCode.test.ts` |
| 2 | Code contains no 0, O, 1, I or L | service | Generate 5,000 codes: none contains `0`, `O`, `1`, `I`, `L`; all characters are in `MEMBER_CODE_ALPHABET`; every code matches `MEMBER_CODE_PATTERN` | `server/memberCode.test.ts` |
| 3 | Codes are shown in upper case | service + component | Stored code equals its upper-case form; component renders the text in upper case | `server/memberCode.test.ts`, `client/src/pages/storefront/MemberCodeSection.test.tsx` |
| 4 | Customer without account and guest 673 have no code and are never found | service | `findMemberByCode` with a code that only exists for a row whose `customerId` is null or 673 returns `undefined`; `getOrCreateMemberCode` for an unlinked user throws `NOT_A_MEMBER`; route returns 403 | `server/memberCode.test.ts`, `server/routes.memberCode.test.ts` |
| 4b | (same, query) | integration | The lookup query excludes `customer_id IS NULL` and `customer.id = 673` against a real database | `server/loyalty.integration.test.ts` |
| 5 | Codes are unique across members | service + integration | First `setMemberCodeIfNull` throws unique violation `23505`, generation retries and succeeds; after 5 failures `CODE_GENERATION_FAILED`; integration: the unique constraint rejects a duplicate | `server/memberCode.test.ts`, `server/loyalty.integration.test.ts` |
| 6 | Loyalty tab shows code as QR and as text, both the same code | component | The mocked `QRCodeSVG` receives `value === memberCode` and `member-code-text` shows the same string | `client/src/pages/storefront/MemberCodeSection.test.tsx` |
| 7 | Regenerating asks for confirmation and states the current code stops working immediately | component | Clicking the button opens the AlertDialog with the `AlertTriangle` icon and the text "Je huidige code werkt meteen niet meer"; no request is sent yet | `client/src/pages/storefront/MemberCodeSection.test.tsx` |
| 8 | Cancelling leaves the code unchanged | component | "Annuleren" closes the dialog, no `POST` is made, the displayed code is unchanged | `client/src/pages/storefront/MemberCodeSection.test.tsx` |
| 9 | After confirming, the new code is shown as QR and text | component | Confirm sends `POST /api/account/member-code/regenerate`; the QR `value` and the text switch to the returned code | `client/src/pages/storefront/MemberCodeSection.test.tsx` |
| 10 | After regeneration the lookup does not find the old code and finds the new code (service level) | service | `regenerateMemberCode` then `findMemberByCode(old)` returns `undefined` and `findMemberByCode(new)` returns the member (stateful in-memory fake of the storage methods) | `server/memberCode.test.ts` |
| 11 | After two regenerations only the last confirmed code works | service | Regenerate twice; the first and second codes do not resolve, the third does | `server/memberCode.test.ts` |
| 12 | Points and balance unchanged after regenerating | service | `regenerateMemberCode` calls only the member-code storage methods; ledger methods never called; `getLedgerBalance` before equals after | `server/memberCode.test.ts` |
| 13 | A B2B account holder also has a member code | service | A linked user whose customer has `b2bClient=true` receives a code | `server/memberCode.test.ts` |
| 14 | A member sees only their own code | route | `GET` and `POST regenerate` return 401 without a session; the user id comes from `req.user.id`; a `userId` in the query or body is ignored | `server/routes.memberCode.test.ts` |
| extra | Regenerate limiter | route + service | 6th regenerate within a minute returns 429 `RATE_LIMITED`; limiter config is `limit 5`, `windowMs 60000`, keyed on `req.user.id` | `server/routes.memberCode.test.ts`, `server/security.test.ts` |

#### Story 9.2.3: Identify a Member by Typing Their Member Code (10 criteria)
| # | Acceptance Criterion | Type | Test Description | File |
|---|---|---|---|---|
| 1 | Valid code selects the matched customer and shows the name for confirmation | route + component | Route returns 200 with the member; the dialog shows "Member found: <name>" and "Select" calls `onSelectCustomer` with the matched customer | `server/routes.memberCode.test.ts`, `client/src/components/pos/CustomerSelectionDialog.test.tsx` |
| 2 | Any letter case accepted, surrounding spaces ignored | service + component | `findMemberByCode("  abcd2345 ")` normalises to `ABCD2345` and matches; typing `abcd2345` sends `ABCD2345` | `server/memberCode.test.ts`, `client/src/components/pos/CustomerSelectionDialog.test.tsx` |
| 3 | Selected member treated like a searched customer (customer-specific prices apply) | component | The object passed to `onSelectCustomer` has the `POSCustomer` shape (`id`, `firstName`, `lastName`, `b2bClient`, `b2bPrices`) identical to a search result, so `POSPage` uses `customer.id` for pricing | `client/src/components/pos/CustomerSelectionDialog.test.tsx` |
| 4 | Unknown code shows a clear message and the cashier can retry | route + component | 404 body has code `MEMBER_CODE_NOT_FOUND`; the dialog shows the message and keeps the input focused and editable | `server/routes.memberCode.test.ts`, `client/src/components/pos/CustomerSelectionDialog.test.tsx` |
| 5 | After an unknown code the cashier can continue without a member | component | "Continue without member" returns to the dialog list; the guest card still works and nothing is selected | `client/src/components/pos/CustomerSelectionDialog.test.tsx` |
| 6 | A regenerated (old) code gives the same message as an unknown code | route | After regeneration the old code and a random valid-shape code return byte-identical 404 bodies | `server/routes.memberCode.test.ts` |
| 7 | B2B member code selects that customer, sale proceeds without points | route + service | Lookup returns the B2B customer with `b2bClient: true`; the SPEC-001 qualification rule returns `NOT_QUALIFIED` for it | `server/routes.memberCode.test.ts`, `server/posService.test.ts` |
| 8 | Lookup available to authenticated POS users only | route | 401 without a session | `server/routes.memberCode.test.ts` |
| 9 | Limited to 10 per minute per user; further attempts get a clear message | route + service | 11th request within a minute returns 429 with the message "Too many member code lookups. Wait a minute and try again."; a different user is not limited; limiter config asserted | `server/routes.memberCode.test.ts`, `server/security.test.ts` |
| 10 | Lookup returns only the customer's name and customer id | route | Response keys are exactly `id`, `firstName`, `lastName`, `b2bClient`, `b2bPrices`; no email, phone, address, birthday, member code or points (see OI-1) | `server/routes.memberCode.test.ts` |
| extra | Code never in a URL or body log | route | Request path contains no code; the logger skips response bodies for `/api/account/member-code*` and `/api/pos/customers/by-member-code` | `server/routes.memberCode.test.ts` |
| extra | Malformed input does not hit the database | service | A code of the wrong length or with `0`/`O` returns `undefined` without calling storage | `server/memberCode.test.ts` |

#### Story 9.2.4: Identify a Member by Scanning Their QR Code (5 criteria)
| # | Acceptance Criterion | Type | Test Description | File |
|---|---|---|---|---|
| 1 | Camera scan gives the same result as typing the code | component | The mocked `qr-scanner` emits `ABCD2345`; the dialog sends exactly the same `POST` and shows the same confirmation card as typing it | `client/src/components/pos/MemberCodeScanner.test.tsx`, `client/src/components/pos/CustomerSelectionDialog.test.tsx` |
| 2 | USB keyboard-wedge scanner gives the same result | component | Simulate rapid keystrokes of the code then `Enter` in the focused input: one lookup, same result as typing | `client/src/components/pos/CustomerSelectionDialog.test.tsx` |
| 3 | Scanned unknown code shows the same message and options | component | Scanner emits an unknown valid-shape code: 404 message and "Continue without member" appear, identical to the typed case | `client/src/components/pos/CustomerSelectionDialog.test.tsx` |
| 4 | A QR that is not a member code is treated as unknown | component | Scanner emits `https://example.com/x`: the unknown-code message appears and no API request is made | `client/src/components/pos/CustomerSelectionDialog.test.tsx` |
| 5 | Camera access denied: the cashier can still type the code | component | `start()` rejects: the fallback text "Camera not available. Type the member code instead." appears and the input still works | `client/src/components/pos/MemberCodeScanner.test.tsx` |

Totals mapped for this spec: 9.2.1 = 14, 9.2.3 = 10, 9.2.4 = 5, total 29 acceptance criteria, each with at least one test.

### Service Tests (Vitest, mock storage)
- `generateMemberCode`: length 8, alphabet only, no excluded characters over a large sample, uses `randomInt` (spy), distinct values across calls.
- `getOrCreateMemberCode`: no link, existing code, race (`setMemberCodeIfNull` returns `undefined`, then re-read), collision retries.
- `regenerateMemberCode`: replaces, retries on collision, not-a-member.
- `findMemberByCode`: normalisation, pattern guard, hit and miss.

### Route Tests (supertest)
- 401 without auth on all three routes; 403 `NOT_A_MEMBER`; 400 on a lookup body without `code`; 404 identical body for unknown, old and malformed; 200 shape; 429 on the 11th lookup and the 6th regenerate.

### Component Tests (React Testing Library)
- `MemberCodeSection`: renders QR and text, dialog flow, cancel, confirm, 403 message.
- `CustomerSelectionDialog`: typed lookup, Enter submit, confirmation card, unknown message, continue without member, 429 message, touch-target class `min-h-11` or `min-h-16` on the new buttons.
- `MemberCodeScanner`: starts and stops the scanner, decodes, denied fallback.

### Implementation Order
1. Complete SPEC-001 steps 1-5 (test tooling, schema, `loyaltyService`).
2. Run `npm audit` and the licence check for `qrcode.react` and `qr-scanner`; install only if they pass (unverified until then).
3. Write all failing tests above.
4. Schema: add `users.memberCode`, `MEMBER_CODE_*` constants, `memberCodeLookupSchema`; `npm run db:push`.
5. Storage methods; `loyaltyService` member-code methods; `posService.lookupMemberByCode`; make the service tests pass.
6. `security.ts` limiters; routes; `server/index.ts` log exclusion; make the route tests pass.
7. Frontend: `MemberCodeSection` in the Loyalty tab, then the dialog code entry, then `MemberCodeScanner`.
8. Refactor with tests green; run `npm run check` and `npm test`.
9. Follow-up T-1: update the technical pointer in `docs/product/backlog/epic-9-loyalty/story-9.2.3-pos-identify-member-by-code.md` from `GET /api/pos/customers/by-member-code/:code` to `POST /api/pos/customers/by-member-code` (code in the body), and its mention of "only name and customer id" per OI-1.

## Go-Live Checklist
- [ ] SPEC-001 released first (link, ledger, Loyalty tab).
- [ ] `npm run db:push` applied; `users.member_code` exists with its unique constraint.
- [ ] `npm audit` clean and licences accepted for `qrcode.react` and `qr-scanner`.
- [ ] A real printed or on-screen QR scanned with the shop's USB scanner and a tablet camera (the camera path needs HTTPS and a user gesture; unverified in this environment).
- [ ] Log check: no member code in the Cloud Run logs after a lookup and a code view.

## Risks and Residual Issues
1. **Per-instance limiter.** The in-memory store counts per Cloud Run instance and resets on a cold start; the effective ceiling with several instances is higher than 10 per minute. A shared store (for example Redis) is out of scope.
2. **Lookup response fields.** The response includes `b2bClient` and `b2bPrices` beyond the literal "name and id" (OI-1).
3. **Lazy code creation.** A member who never opens the Loyalty tab has no code until they do, so a POS lookup of their (nonexistent) code cannot happen; this matches the observable behaviour of "every member has a code" because the code appears on first view. If the PO wants codes created at link time, call `getOrCreateMemberCode` from `linkAccountToCustomer` (small change).
4. **Look-alike characters.** The alphabet excludes only 0, O, 1, I, L per BR-8; B/8, S/5 and Z/2 remain. Accepted by the UC.
5. **Camera on POS hardware.** Camera support depends on the POS device and requires a secure context; the fallback to typing is always available.
6. **Unverified items:** `qrcode.react` and `qr-scanner` API names, versions, licences and audit status; `express-rate-limit` v8 option names; `apiRequest` error formatting; `alert-dialog.tsx` presence; how the POS variants route derives B2B pricing for the selected customer.

## Open Items
- OI-1: Story 9.2.3 AC 10 says the lookup returns only name and customer id, but the POS `POSCustomer` type needs `b2bClient` and `b2bPrices`. The spec returns those two flags too. PO to confirm, or accept an extra follow-up request by id to fetch the flags.
- OI-2: Verify and install the two libraries (audit, licence).
- OI-3: Update story 9.2.3 pointer (T-1).
- OI-4: Whether a member code should be created at link time instead of lazily (Risk 3).

## Quality Checklist (run before saving)
- [x] References link the stories (9.2.1, 9.2.3, 9.2.4) and UC-032 with the related UCs.
- [x] Scope states what is and is not included.
- [x] 3-layer architecture respected: routes only validate, authenticate and respond; no transactions needed (single atomic statements); storage holds queries only.
- [x] Every file to create or modify has an exact path (consolidated list below).
- [x] Schema change includes the Drizzle definition, constants and Zod schema, and the migration step (`db:push`).
- [x] Security section covers roles, validation, rate limiting, logging, enumeration and dependencies.
- [x] Test Plan maps all 29 acceptance criteria to tests with type, description and file; implementation order is tests first.
- [x] No `any` in the specified types.
- [x] Design guidelines referenced (partial read, noted); AlertDialog with AlertTriangle; POS targets at least 44 px.
- [x] `roastService.ts` pattern referenced; existing patterns followed.

## Consolidated File List
Create: `client/src/components/storefront/MemberCodeSection.tsx`, `client/src/components/pos/MemberCodeScanner.tsx`, `server/memberCode.test.ts`, `server/routes.memberCode.test.ts`, `client/src/pages/storefront/MemberCodeSection.test.tsx`, `client/src/components/pos/CustomerSelectionDialog.test.tsx`, `client/src/components/pos/MemberCodeScanner.test.tsx`.

Modify: `shared/schema.ts`, `server/storage.ts`, `server/loyaltyService.ts`, `server/posService.ts`, `server/security.ts`, `server/routes.ts`, `server/index.ts`, `client/src/pages/storefront/Account.tsx`, `client/src/components/pos/CustomerSelectionDialog.tsx`, `package.json`, `server/security.test.ts`, `server/posService.test.ts`, `server/loyalty.integration.test.ts`, `docs/product/backlog/epic-9-loyalty/story-9.2.3-pos-identify-member-by-code.md` (pointer update, T-1).
