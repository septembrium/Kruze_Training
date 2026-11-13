# SPEC-001: Loyalty Points - Earning, Ledger, Balance and History

Status: Draft for implementation (file plan confirmed by the product owner).
Companion spec: SPEC-002 (member code, POS lookup by code and QR). SPEC-002 depends on this spec.

## References
- **Use case:** UC-032 (`docs/use-cases/UC-032-loyalty-earn-points.md`, Reviewed). Loyalty alternative flows in UC-003 AF-1, UC-004 AF-3, UC-015 AF-6 and BR-7, UC-021 AF-11, UC-025 AF-5, UC-026 AF-5 and AF-6.
- **Epic:** `docs/product/backlog/epic-9-loyalty/epic.md`
- **Stories (all review-PASSED):**
  - 9.1.1 `docs/product/backlog/epic-9-loyalty/story-9.1.1-link-account-customer.md`
  - 9.1.2 `.../story-9.1.2-earn-rate-setting.md`
  - 9.1.3 `.../story-9.1.3-earn-webshop-points.md`
  - 9.1.4 `.../story-9.1.4-checkout-member-attribution.md`
  - 9.1.5 `.../story-9.1.5-points-survive-payment-delays.md`
  - 9.1.6 `.../story-9.1.6-points-balance-history.md`
  - 9.1.7 `.../story-9.1.7-admin-makes-sale-paid.md`
  - 9.2.2 `.../story-9.2.2-pos-sale-earns-points.md` (kept in this spec because it needs only 9.1.3, 9.1.4 and 9.1.5, not member codes)
- **Reference implementation:** `server/roastService.ts` (class plus exported singleton, transactions in the service, storage for data access).

## Scope

### In scope
- Link a confirmed Storefront account to a customer record on every sign-in (9.1.1).
- Admin setting for the global earn rate (9.1.2).
- Append-only points ledger, earn math and the single qualification rule (9.1.3, 9.1.4).
- Awarding on all five paths to a paid sale: Mollie webhook, POS sale, admin status change, admin creating a paid sale, admin editing a sale to paid (9.1.3, 9.1.7, 9.2.2).
- Server-side checkout attribution to the logged-in member's linked customer (9.1.4). This closes an existing attribution bypass (see Security Considerations).
- Failure marker and scheduled retry (9.1.5).
- Balance and history endpoint and the Loyalty tab in the Storefront account (9.1.6).
- Aligning `/api/account/profile`, `/api/account/top-products` and `/api/checkout/customer-data` to the linked customer.
- Blocking deletion of a customer that is linked to an account or has ledger entries.
- Removing the legacy `loyalty` table.

### Out of scope (for this spec)
- Member code, QR code, POS lookup by code, camera scanning: SPEC-002.
- Redeeming points, badges, challenges, reversal entries, refunds, admin viewing or adjusting points, backfill of past sales, linking a sale to a member after completion, notifications, promotional multipliers (UC-032 Out of Scope).
- Rewriting the existing manual validation in `POST /api/pos/sales` (`server/routes.ts:2509-2560`) to Zod. Only the award hook is added.
- Changing `/api/account/orders` (`server/routes.ts:3326`), which lists by `webshop_order.email`.
- Syncing `users.email` when a Supabase email changes.

## Decisions fixed by the product owner
1. Supabase "Confirm email" is ON. The code also refuses to link when `email_confirmed_at` is empty. Go-live checklist item added.
2. Migrations use `drizzle-kit push` (`npm run db:push`). The `db/migrations/` auto-runner in `server/index.ts:161-190` is not used. What Drizzle cannot express goes in `scripts/loyalty-setup.sql`, run once after `db:push`.
3. The legacy `loyalty` table is empty and unused and is dropped.
4. Retry: Cloud Scheduler every 5 minutes calls `POST /api/internal/loyalty/retry-awards`, protected by `LOYALTY_RETRY_SECRET`.
5. Split: this spec covers 9.1.1-9.1.7 and 9.2.2. SPEC-002 covers 9.2.1, 9.2.3 and 9.2.4.

## Schema Changes

### Checklist
- [ ] `users.customer_id` (unique, nullable FK to `customer.id`) in `shared/schema.ts`
- [ ] `users.member_code` column is added by SPEC-002 (not here)
- [ ] `loyaltyEntryTypeEnum`, `loyaltyChannelEnum`, `loyaltyPointsLedger`, `loyaltyAwardFailures` in `shared/schema.ts`
- [ ] Remove legacy `loyalty` table, `insertLoyaltySchema`, `Loyalty` and `InsertLoyalty` types
- [ ] Zod insert schemas and types
- [ ] `earnRateInputSchema` Zod schema (shared by route and client)
- [ ] `scripts/loyalty-setup.sql` and `scripts/loyalty-setup.ts`, npm script `db:loyalty-setup`
- [ ] `npm run db:push` then `npm run db:loyalty-setup` (see Migration Procedure)

### Primary key convention (deviation noted)
`.claude/rules/database.md` says "UUID primary keys". The actual schema uses `serial` integer PKs for every domain table (`shared/schema.ts`: `customer` line 117, `sale` line 180, `parameter_table` line 416) and only `users.id` is a UUID varchar (line 30). The new tables use `serial` PKs because their foreign keys target integer `customer.id` and `sale.id`. This is a deliberate deviation from the rule file, matching the existing schema.

### Drizzle definitions (`shared/schema.ts`)

Import line 2 must add `uniqueIndex`:
```ts
import { pgTable, text, varchar, integer, decimal, timestamp, boolean, pgEnum, jsonb, index, uniqueIndex, date, smallint, serial } from "drizzle-orm/pg-core";
```

Change `users` (currently lines 29-38). `customers` is declared later in the file; the arrow function reference is lazy, so ordering is fine.
```ts
export const users = pgTable("users", {
  id: varchar("id").primaryKey().default(sql`gen_random_uuid()`),
  email: varchar("email").unique(),
  firstName: varchar("first_name"),
  lastName: varchar("last_name"),
  profileImageUrl: varchar("profile_image_url"),
  role: userRoleEnum("role").notNull().default("customer"),
  // Loyalty (SPEC-001): the customer record this Storefront account is linked to.
  // Null until the first sign-in with a confirmed email. Unique: one customer, at most one account.
  customerId: integer("customer_id").references(() => customers.id).unique(),
  createdAt: timestamp("created_at").defaultNow(),
  updatedAt: timestamp("updated_at").defaultNow(),
});
```
`UpsertUser` (`typeof users.$inferInsert`, line 40) gains an optional `customerId`. Existing `storage.upsertUser` callers (`supabaseAuth.ts:88,133,304`) do not pass it, so they are unaffected. `storage.upsertUser` rewrites `users.id` on an email collision but only sets name, image and role (`storage.ts:435-446`), so the link is preserved.

Replace the legacy `loyalty` table (lines 391-398) with:
```ts
// Loyalty points ledger - append-only. One earn entry per sale. Balance = SUM(points).
export const loyaltyEntryTypeEnum = pgEnum("loyalty_entry_type", ["earn", "reversal"]); // "reversal" is reserved, never written in v1
export const loyaltyChannelEnum = pgEnum("loyalty_channel", ["webshop", "pos"]);

export const loyaltyPointsLedger = pgTable("loyalty_points_ledger", {
  id: serial("id").primaryKey(),
  customerId: integer("customer_id").notNull().references(() => customers.id),
  saleId: integer("sale_id").notNull().references(() => sales.id),
  entryType: loyaltyEntryTypeEnum("entry_type").notNull().default("earn"),
  points: integer("points").notNull(), // signed; earn entries are always > 0
  earnRate: decimal("earn_rate", { precision: 5, scale: 2 }).notNull(), // points per EUR 1 in force when awarded
  basisAmount: decimal("basis_amount", { precision: 10, scale: 2 }).notNull(), // earn basis in EUR (explainability)
  channel: loyaltyChannelEnum("channel").notNull(),
  createdAt: timestamp("created_at").notNull().defaultNow(),
}, (table) => [
  // Idempotency (BR-10): at most one earn entry per sale. Reversal entries may share the sale later.
  uniqueIndex("uq_loyalty_earn_per_sale").on(table.saleId).where(sql`${table.entryType} = 'earn'`),
  index("idx_loyalty_ledger_customer_created").on(table.customerId, table.createdAt, table.id),
]);

// Failure marker: one row per sale whose award was attempted and threw. Pending while resolvedAt is null.
export const loyaltyAwardFailures = pgTable("loyalty_award_failure", {
  id: serial("id").primaryKey(),
  saleId: integer("sale_id").notNull().unique().references(() => sales.id),
  attempts: integer("attempts").notNull().default(1),
  lastErrorCode: text("last_error_code"), // short code, never PII
  earnRateHundredths: integer("earn_rate_hundredths"), // rate snapshot taken at the first attempt (nullable)
  firstFailedAt: timestamp("first_failed_at").notNull().defaultNow(),
  lastAttemptAt: timestamp("last_attempt_at").notNull().defaultNow(),
  resolvedAt: timestamp("resolved_at"),
}, (table) => [
  index("idx_loyalty_award_failure_pending").on(table.lastAttemptAt).where(sql`${table.resolvedAt} IS NULL`),
]);
```
Unverified: that Drizzle 0.39's index builder accepts `.where(sql...)` for `uniqueIndex` and `index`. The safety net in `scripts/loyalty-setup.sql` creates both indexes with `IF NOT EXISTS` and the same names. If `.where` does not compile, define the two partial indexes only in the setup SQL and note that `drizzle-kit push` may then propose dropping them; do not accept that proposal.

Why the snapshot column: a retried award must use the rate in force when the sale became paid (AC "after a rate change only sales paid afterwards use the new rate"), not the rate at retry time. The snapshot is nullable because the failure may be the rate read itself; the retry then uses the current rate.

Zod insert schemas and types (replace lines 821 and 952-953):
```ts
export const insertLoyaltyLedgerSchema = createInsertSchema(loyaltyPointsLedger).omit({ id: true, createdAt: true });
export const insertLoyaltyAwardFailureSchema = createInsertSchema(loyaltyAwardFailures).omit({
  id: true, firstFailedAt: true, lastAttemptAt: true, resolvedAt: true,
});

export type LoyaltyLedgerEntry = InferSelectModel<typeof loyaltyPointsLedger>;
export type InsertLoyaltyLedgerEntry = z.infer<typeof insertLoyaltyLedgerSchema>;
export type LoyaltyAwardFailure = InferSelectModel<typeof loyaltyAwardFailures>;
export type InsertLoyaltyAwardFailure = z.infer<typeof insertLoyaltyAwardFailureSchema>;
```
`InferSelectModel` comes from `drizzle-orm` (the file imports only `sql` and `relations` today, lines 1 and 3; add the import).

Earn-rate input schema (also used by the client form):
```ts
export const EARN_RATE_MAX = 100;
export const earnRateInputSchema = z
  .union([z.string(), z.number()], { errorMap: () => ({ message: "Earn rate must be a number" }) })
  .transform((v) => (typeof v === "number" ? String(v) : v.trim().replace(",", ".")))
  .refine((s) => /^\d+(\.\d+)?$/.test(s), "Earn rate must be a positive number")
  .refine((s) => !s.includes(".") || s.split(".")[1].length <= 2, "Earn rate can have at most 2 decimals")
  .refine((s) => Number(s) > 0, "Earn rate must be greater than 0")
  .refine((s) => Number(s) <= EARN_RATE_MAX, `Earn rate cannot be higher than ${EARN_RATE_MAX}`);
```

### Shared constant (new file `server/customerConstants.ts`)
```ts
export const GUEST_CUSTOMER_ID = 673; // the default company customer "Kruze"; never a loyalty member
```
Replace the hard-coded `673` in `posService.ts:289,644,722` and `checkoutService.ts:563` with this constant (behaviour unchanged).

### Migration Procedure (drizzle-kit push)
`drizzle.config.ts` reads `DATABASE_URL`, schema `./shared/schema.ts`, dialect postgresql. Steps, in order:
1. Confirm the legacy table is empty: `SELECT count(*) FROM loyalty;` must return 0. If not, stop and ask the product owner.
2. `npm run db:push` (script exists, `package.json:11`). Read every statement it proposes. Abort if anything beyond the loyalty changes appears.
3. **Prompt handling.** The legacy `loyalty` table is removed from `schema.ts`, and the new tables are added. `drizzle-kit push` may ask whether `loyalty_points_ledger` (or `loyalty_award_failure`) is "created or renamed from another table". Answer "create table" (the first option, `+ create`), never "rename from loyalty". It may then show a data-loss warning for dropping `loyalty`. Accept the drop only because step 1 proved the table empty. Do not run push with `--force` in production.
4. `npm run db:loyalty-setup` (see below). It is idempotent and safe to re-run. It also runs `DROP TABLE IF EXISTS loyalty`, so the table is gone even if the push prompt was declined.

Add to `package.json` scripts:
```json
"db:loyalty-setup": "tsx scripts/loyalty-setup.ts"
```
Chosen command: `npm run db:loyalty-setup` (a small `tsx` script, matching the existing `tsx scripts/...` convention in `package.json:12-13`). It needs no `psql` binary. The `psql "$DATABASE_URL" -v ON_ERROR_STOP=1 -f scripts/loyalty-setup.sql` form is not used.

`scripts/loyalty-setup.ts`:
```ts
import { readFileSync } from "fs";
import { join } from "path";
import { pool } from "../server/db"; // `pool` is exported by server/db.ts (imported in server/migrate.ts:1)

async function main(): Promise<void> {
  const sqlText = readFileSync(join(process.cwd(), "scripts", "loyalty-setup.sql"), "utf-8");
  await pool.query(sqlText); // single multi-statement script wrapped in BEGIN/COMMIT
  console.log("loyalty-setup.sql applied");
}

main()
  .then(() => pool.end())
  .catch(async (error: unknown) => {
    console.error("loyalty-setup failed:", error instanceof Error ? error.message : String(error));
    await pool.end();
    process.exit(1);
  });
```
Unverified: that `server/db.ts` `pool` supports a multi-statement simple query (the file was not read; `migrate.ts` uses `pool.connect()` and `client.query`). If it does not, split the SQL file on a `-- STATEMENT` marker and run the pieces inside one transaction.

`scripts/loyalty-setup.sql` (idempotent):
```sql
BEGIN;

-- 1. Seed the earn rate (UC-015 BR-7, story 9.1.2). parameter_name is unique (shared/schema.ts:417).
INSERT INTO parameter_table (parameter_name, parameter_value)
VALUES ('loyalty_earn_rate', '1')
ON CONFLICT (parameter_name) DO NOTHING;

-- 2. Safety net: partial unique index (same name as in schema.ts; no-op if push created it).
CREATE UNIQUE INDEX IF NOT EXISTS uq_loyalty_earn_per_sale
  ON loyalty_points_ledger (sale_id) WHERE entry_type = 'earn';

-- 3. Append-only ledger (BR-9): no UPDATE, DELETE or TRUNCATE.
CREATE OR REPLACE FUNCTION loyalty_ledger_append_only() RETURNS trigger AS $$
BEGIN
  RAISE EXCEPTION 'loyalty_points_ledger is append-only: % is not allowed', TG_OP;
END;
$$ LANGUAGE plpgsql;

DROP TRIGGER IF EXISTS trg_loyalty_ledger_no_update_delete ON loyalty_points_ledger;
CREATE TRIGGER trg_loyalty_ledger_no_update_delete
  BEFORE UPDATE OR DELETE ON loyalty_points_ledger
  FOR EACH ROW EXECUTE FUNCTION loyalty_ledger_append_only();

DROP TRIGGER IF EXISTS trg_loyalty_ledger_no_truncate ON loyalty_points_ledger;
CREATE TRIGGER trg_loyalty_ledger_no_truncate
  BEFORE TRUNCATE ON loyalty_points_ledger
  FOR EACH STATEMENT EXECUTE FUNCTION loyalty_ledger_append_only();

-- 4. Validate rows on insert: earn entries must have points > 0; any entry must have points <> 0.
CREATE OR REPLACE FUNCTION loyalty_ledger_validate() RETURNS trigger AS $$
BEGIN
  IF NEW.points = 0 THEN
    RAISE EXCEPTION 'loyalty_points_ledger.points must not be 0';
  END IF;
  IF NEW.entry_type = 'earn' AND NEW.points < 0 THEN
    RAISE EXCEPTION 'loyalty_points_ledger earn entries must have positive points';
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

DROP TRIGGER IF EXISTS trg_loyalty_ledger_validate ON loyalty_points_ledger;
CREATE TRIGGER trg_loyalty_ledger_validate
  BEFORE INSERT ON loyalty_points_ledger
  FOR EACH ROW EXECUTE FUNCTION loyalty_ledger_validate();

-- 5. Retire the legacy, unused loyalty table (empty, confirmed by the product owner).
DROP TABLE IF EXISTS loyalty;

COMMIT;
```
Note for tests that need to clean the ledger: the trigger blocks `DELETE` and `TRUNCATE`. Integration tests use a throwaway database and drop the trigger in their own setup, or recreate the schema.

## Backend Changes

Layering follows `server/roastService.ts`: routes parse and respond, services own business rules and `db.transaction`, storage owns queries and holds no transaction and no business logic. Existing code in `customerService.ts` already runs queries on `tx` inside its service transaction (`customerService.ts:163-218`); the link transaction follows that same pattern.

### Storage methods (`server/storage.ts`, add to `IStorage` and `DatabaseStorage`)
```ts
export interface EarnBasis {
  saleId: number;
  customerId: number;
  salesStatus: number;
  webshopSale: boolean;
  basisCents: number; // integer cents, see "Earn math"
}
export interface MemberQualification { customerId: number; b2bClient: boolean; hasAccount: boolean }
export interface LoyaltyHistoryRow {
  id: number; createdAt: Date; channel: "webshop" | "pos"; reference: string; points: number;
}
export interface UserLinkState {
  id: string; email: string | null; firstName: string | null; lastName: string | null; customerId: number | null;
}

// Account link (9.1.1, 9.1.4)
getUserLinkState(userId: string): Promise<UserLinkState | undefined>;
getLinkedCustomer(userId: string): Promise<Customer | undefined>;          // users.customer_id -> customer
checkCustomerLinkedToAccount(customerId: number): Promise<boolean>;         // for deletion guard
checkCustomerHasLedgerEntries(customerId: number): Promise<boolean>;        // for deletion guard

// Earning (9.1.3, 9.1.4, 9.1.7, 9.2.2)
getEarnBasis(saleId: number): Promise<EarnBasis | undefined>;
getMemberQualification(customerId: number): Promise<MemberQualification | undefined>;
insertEarnEntry(entry: InsertLoyaltyLedgerEntry): Promise<LoyaltyLedgerEntry | undefined>; // undefined = conflict

// Balance and history (9.1.6)
getLedgerBalance(customerId: number): Promise<number>;
getLedgerPage(customerId: number, page: number, pageSize: number): Promise<{ rows: LoyaltyHistoryRow[]; totalEntries: number }>;

// Failure marker (9.1.5)
upsertAwardFailure(saleId: number, errorCode: string, earnRateHundredths: number | null): Promise<void>;
listPendingAwardFailures(limit: number): Promise<LoyaltyAwardFailure[]>;
resolveAwardFailure(saleId: number): Promise<void>;
recordAwardRetryFailure(saleId: number, errorCode: string): Promise<void>; // attempts + 1, lastAttemptAt = now
```
No `updateLedgerEntry` or `deleteLedgerEntry` exists (BR-9); a test asserts the storage surface has none.

Query specifications:
- `getEarnBasis` returns one SQL row. The sums use SQL `numeric`, not JS floats:
```sql
SELECT s.id AS sale_id, s.customer_id, s.sales_status, s.webshop_sale,
  (GREATEST(ROUND(
     COALESCE(SUM(sl.total_with_vat) FILTER (WHERE sl.product_id NOT IN (145, 146)), 0)
       - s.discount_with_vat, 2), 0) * 100)::bigint AS basis_cents
FROM sale s LEFT JOIN salesline sl ON sl.sale_id = s.id
WHERE s.id = $1
GROUP BY s.id
```
The product ids come from the exported `SHIPPING_PRODUCT_IDS` (parameterised, not string-built). `ROUND(numeric, 2)` rounds half away from zero, which equals half up for the non-negative values here. `sale.shipping_cost_with_vat` is deliberately not read: for admin-created sales it is a separate field and not part of the lines (`salesService.ts:140`), and for webshop sales shipping is a line (`checkoutService.ts:183-196`), so excluding the shipping product lines covers both.
- `getMemberQualification`: `customer` left-joined with `EXISTS (SELECT 1 FROM users WHERE users.customer_id = customer.id)` as `hasAccount`.
- `insertEarnEntry`: `INSERT ... ON CONFLICT DO NOTHING RETURNING *`. Returns `undefined` when a row already exists. The plain form (no conflict target) is used on purpose so it works with the partial unique index without depending on Drizzle's partial-target syntax (unverified).
- `getLedgerPage`: join `webshop_order` on `webshop_order.sale_id = ledger.sale_id`; `reference = COALESCE(webshop_order.order_number, '#' || ledger.sale_id)`; `ORDER BY ledger.created_at DESC, ledger.id DESC LIMIT $pageSize OFFSET ($page - 1) * $pageSize`; `totalEntries` from `COUNT(*)`.
- `getLedgerBalance`: `COALESCE(SUM(points), 0)` over all entries of the customer (not only the page).
- `getLinkedCustomer`: `users JOIN customer ON customer.id = users.customer_id`.

### `server/shippingService.ts`
Export the shipping product ids (currently private constants at lines 19-20):
```ts
export const SHIPPING_PRODUCT_IDS: readonly number[] = [145, 146];
```
(`SHIPPING_PRODUCT_6_PCT`/`_21_PCT` stay and are reused to build the array.)

### `server/loyaltyService.ts` (new, follows `roastService.ts`)
```ts
export class LoyaltyError extends Error {
  constructor(public readonly code: "INVALID_EARN_RATE" | "SALE_NOT_FOUND", message: string) { super(message); }
}
export type AwardResult =
  | { awarded: true; points: number; entry: LoyaltyLedgerEntry }
  | { awarded: false; reason: "NOT_PAID" | "NOT_QUALIFIED" | "BELOW_ONE_POINT" | "ALREADY_AWARDED" };

export class LoyaltyService {
  async getEarnRateSetting(): Promise<string>;                 // parameter 'loyalty_earn_rate', default "1" when missing
  async setEarnRate(input: unknown): Promise<string>;          // validates via earnRateInputSchema; stores canonical string
  async getEarnRateHundredths(): Promise<number>;              // Math.round(Number(rate) * 100)
  computeEarnPoints(basisCents: number, rateHundredths: number): number; // pure
  async isQualifyingMember(customerId: number): Promise<boolean>;
  async awardPointsForSale(saleId: number, options?: { rateHundredths?: number }): Promise<AwardResult>;
  async awardPointsSafely(saleId: number): Promise<void>;      // never throws
  async retryFailedAwards(limit?: number): Promise<{ processed: number; resolved: number; stillFailing: number }>;
  async getBalanceAndHistory(userId: string, page: number): Promise<LoyaltyHistoryResponse>;
}
export const loyaltyService = new LoyaltyService();
```
Behaviour:
- **Earn math (BR-3, BR-4).** `computeEarnPoints = Math.floor((basisCents * rateHundredths) / 10000)` using integers only. Examples: basis EUR 42,50 = 4250 cents at rate 1 (100): `4250*100/10000 = 42.5` gives 42. At rate 0.5 (50): `4250*50/10000 = 21.25` gives 21. The basis is already rounded to cents (half up) by SQL. The earn rate has at most 2 decimals (validated), so `rateHundredths` is an exact integer.
- **`isQualifyingMember(customerId)` is the single qualification rule** (story 9.1.4, BR-5, BR-6): true only if the customer exists, `customerId !== GUEST_CUSTOMER_ID`, `b2bClient === false` (evaluated now, at award time), and `hasAccount === true`. A linked account implies a confirmed email because linking requires it. All five paths call `awardPointsSafely`, which uses this rule; no path re-implements it.
- **`awardPointsForSale(saleId, options)`:** load `getEarnBasis` (throw `LoyaltyError("SALE_NOT_FOUND")` when missing); return `NOT_PAID` unless `salesStatus === 4` (`const PAID_STATUS_ID = 4`, same value as `salesService.ts:37` and `checkoutService.ts:31`); return `NOT_QUALIFIED` when not a qualifying member; `rateHundredths = options?.rateHundredths ?? await getEarnRateHundredths()`; `points = computeEarnPoints(...)`; return `BELOW_ONE_POINT` when `points < 1` (no entry, BR: a basket under 1 point records nothing); insert the entry (`channel = webshopSale ? "webshop" : "pos"`, `earnRate = rateHundredths / 100` as a 2-decimal string, `basisAmount = basisCents / 100` as a 2-decimal string); when the insert returns `undefined` return `ALREADY_AWARDED`; on success or `ALREADY_AWARDED`, resolve any pending failure marker for the sale.
- **`awardPointsSafely(saleId)`:** reads the rate snapshot first (inside the try block), calls `awardPointsForSale`, and on any thrown error writes the failure marker with a short `errorCode` (the error class name, never the message and never PII) and the snapshot (null if the rate read failed). The marker write is itself wrapped in try/catch; if it also fails it logs `Loyalty award failed for sale <saleId>` (sale id only, no customer data) and returns. It never throws, so the sale and the customer are never affected (UC-032 EF-3, story 9.1.5).
- **`retryFailedAwards(limit = 50)`:** `listPendingAwardFailures(limit)` (rows with `resolved_at IS NULL`, oldest `last_attempt_at` first). For each: `awardPointsForSale(saleId, { rateHundredths: marker.earnRateHundredths ?? undefined })`; any non-throwing result (including `NOT_QUALIFIED` and `ALREADY_AWARDED`) resolves the marker; a throw calls `recordAwardRetryFailure`. It touches only marker rows, so it never sweeps PAID sales without an entry (story 9.1.5 AC 7). Concurrent runs are safe because the award is idempotent through the unique index, so no row locking is needed.
- **`getBalanceAndHistory(userId, page)`:** resolve `users.customer_id` server-side. A user without a link gets `{ isMember: false, balance: 0, entries: [], ... }`. Otherwise balance from `getLedgerBalance` and a page of 50 from `getLedgerPage`. The customer id never comes from the client.

### `server/customerService.ts` changes (9.1.1, 9.1.4)
```ts
export interface LinkAccountInput { userId: string; email: string | null; emailConfirmed: boolean }
export interface LinkAccountResult { linked: boolean; customerId: number | null; created: boolean }

async linkAccountToCustomer(input: LinkAccountInput): Promise<LinkAccountResult>;
async ensureLinkedCustomer(userId: string, email: string | null, emailConfirmed: boolean): Promise<Customer | undefined>;
async getLinkedCustomer(userId: string): Promise<Customer | undefined>;   // read-only, delegates to storage
async deleteCustomer(customerId: number): Promise<boolean>;               // guarded delete
```
`linkAccountToCustomer` (one `db.transaction`):
1. Return `{ linked: false, customerId: null, created: false }` when `!emailConfirmed` or the email is empty. This is the code-level refusal when `email_confirmed_at` is empty (PO decision 1).
2. `SELECT ... FROM users WHERE id = $userId FOR UPDATE`. This serialises simultaneous first sign-ins of the same account.
3. If `users.customer_id` is already set, return it (no relink, no new record).
4. Match candidates: `lower(btrim(customer.email)) = lower(btrim($email)) AND customer.id <> 673`, ordered by most recent sale date desc (nulls last), then most recent sale id desc (nulls last), then `creation_date` desc, then `id` desc, `LIMIT 1`. Customer 673 is excluded from matching.
5. A B2B match is linked as any other.
6. If there is no match, insert a customer (`firstName`/`lastName` from the user or `""` as `updateAccountProfile` does at `customerService.ts:201-202`, `email` as stored on the user, `b2bClient: false`, `b2bPrices: false`) and link it (`created: true`).
7. `UPDATE users SET customer_id = $id, updated_at = now() WHERE id = $userId AND customer_id IS NULL`.

`ensureLinkedCustomer` returns the existing link, or runs `linkAccountToCustomer` when unlinked and confirmed. Used by checkout and profile save. `getLinkedCustomer` never writes and is used by the three read routes.

`updateAccountProfile` (currently `(userId, email, data)`, lines 162-219) becomes `(userId, email, emailConfirmed, data)`: inside its transaction it resolves the linked customer (linking first when needed and confirmed). It then updates that record by id, never by email, and never inserts a second record for a linked account (story 9.1.1 AC 11). If the account cannot be linked (unconfirmed), it updates only the `users` names and throws `AccountNotLinkedError`, which the route maps to HTTP 409 with `"Bevestig eerst je e-mailadres voordat je je gegevens kunt opslaan."`.

`deleteCustomer(customerId)`: refuses with `CustomerDeletionBlockedError` (a `message` explaining why) when the customer has sales (existing rule, `routes.ts:651-657`), is linked to a Storefront account, or has ledger entries; otherwise calls `storage.deleteCustomer`. Ledger entries imply a sale, so the sales check already blocks them; the explicit ledger check documents the intent and survives future changes. `b2bClient` edits through `PATCH /api/customers/:id` (`routes.ts:622-644`) stay allowed: B2B status is evaluated at award time, and points already awarded stay (BR-11).

### `server/supabaseAuth.ts` changes (9.1.1, 9.1.4)
- Extend the `Request.user` type (lines 32-36) with `emailConfirmed?: boolean`.
- `isAuthenticated` sets `emailConfirmed: Boolean(data.user.email_confirmed_at)` in both branches (valid token, line 378; refreshed session, line 362). Unverified: the `email_confirmed_at` field name on the installed `@supabase/supabase-js` `User` type; verify in the package types before coding.
- New exported `optionalAuth: RequestHandler`. It reads the same cookies, validates the token the same way, may refresh the session, and sets `req.user` when valid. It never sends 401 and always calls `next()`. A private helper `resolveSupabaseSession(req, res)` holds the shared token logic for `optionalAuth`; `isAuthenticated` keeps its current body apart from the added `emailConfirmed`.
- **Link on every sign-in.** In `POST /api/auth/login`, after the user row is resolved (after line 142, outside the `if (!user)` block), call `customerService.linkAccountToCustomer({ userId: user.id, email: user.email, emailConfirmed: Boolean(data.user.email_confirmed_at) })`. In `GET /api/auth/callback`, do the same after line 312. The call is wrapped in try/catch: a link failure logs a message without PII and never blocks the login (the next sign-in and the first checkout retry it). **Not at signup** (line 88): the account is unconfirmed there and BR-12 says it links at the first sign-in after confirmation.

### `server/checkoutService.ts` changes (9.1.3, 9.1.4)
- `createOrder(request, session)` where `session: { userId?: string; email?: string | null; emailConfirmed?: boolean }`. `CheckoutRequest.isGuest` is no longer read for attribution (see Security Considerations).
- `findOrCreateCustomer(request, session)`:
  - No session user: return `GUEST_CUSTOMER_ID`. A guest checkout never matches a member by email (BR-6).
  - Session user, email confirmed: `customerService.ensureLinkedCustomer(...)`; use that record regardless of the email typed in the order details. If the record has no birthday and the request has one, set it (keeps the behaviour at `checkoutService.ts:586-591`).
  - Session user, email not confirmed: return `GUEST_CUSTOMER_ID` (not a member, not linked to an existing record).
  - The old email lookup and customer insert (`checkoutService.ts:576-620`) are removed.
- `completeSale` (line 357): after the `db.transaction` at lines 388-427 commits, and before the emails at line 429, add `await loyaltyService.awardPointsSafely(order.saleId)`. The early return at lines 382-385 (sale already paid) deliberately does NOT call the award: re-delivered webhooks would otherwise award sales that were paid before go-live.
- The webhook only reaches `completeSale` when Mollie reports `paid` (lines 346-348), so pending, failed, expired and cancelled payments award nothing (story 9.1.5).

### `server/salesService.ts` changes (9.1.7)
Add `import { loyaltyService } from "./loyaltyService"`. In every case the award runs after the transaction commits.
- `updateSaleStatus` (line 454): inside the transaction record `becamePaid = currentSale.salesStatus !== PAID_STATUS_ID && statusId === PAID_STATUS_ID` in an outer variable. The existing code lets a PAID sale be set to PAID again (it only rejects changing away from PAID at lines 476-478), so the award must be guarded on the transition, not on the final status. After the transaction: `if (becamePaid) await loyaltyService.awardPointsSafely(saleId)`.
- `createSale` (line 84): after the transaction, `if (request.salesStatus === PAID_STATUS_ID) await loyaltyService.awardPointsSafely(created.id)`.
- `updateSale` (line 203): paid sales already throw at lines 225-227, so the sale was not paid before. After the transaction: `if (request.salesStatus === PAID_STATUS_ID) await loyaltyService.awardPointsSafely(saleId)`.
- The other call to `updateSaleStatus(id, 3, true)` (`routes.ts:941`, INVOICED) is not affected.
- `storage.updateSaleStatus` and `storage.createSale` (`storage.ts:1119-1138`) have no callers in the files searched (only `salesService` is used by routes); they are left unchanged. Unverified: a complete caller search of the whole repo.

### `server/posService.ts` changes (9.2.2)
In `createPOSSale` (line 369) assign the transaction result, award, then return:
```ts
const result = await db.transaction(async (tx) => { /* unchanged body */ });
await loyaltyService.awardPointsSafely(result.sale.id);
return result;
```
The POS sale is created as PAID (`posService.ts:402-410,464`), has no shipping line, and stores any sale-level discount in `discountWithVat` (lines 440-451), which the basis query subtracts. A selected non-member, a B2B client and the guest customer fall out through `isQualifyingMember`; a POS sale with a recording failure still returns the created sale.

### Routes (`server/routes.ts`, thin: parse, authenticate, call service, respond)
| Method and path | Auth | Validation | Behaviour |
|---|---|---|---|
| `GET /api/settings/loyalty-params` | `isAdmin` | none | `{ earnRate: string }` from `loyaltyService.getEarnRateSetting()` |
| `POST /api/settings/loyalty-params` | `isAdmin` | `z.object({ earnRate: earnRateInputSchema })` | `{ earnRate }`; 400 `{ error, code: "INVALID_EARN_RATE" }` with the first Zod message; stored rate unchanged on failure |
| `GET /api/account/loyalty` | `isAuthenticated` | `z.object({ page: z.coerce.number().int().min(1).default(1) })` on the query | `LoyaltyHistoryResponse` |
| `POST /api/checkout/create` (changed) | `optionalAuth` | existing `checkoutRequestSchema` (`routes.ts:3148-3169`, `isGuest` stays accepted but is ignored) | passes `{ userId, email, emailConfirmed }` from `req.user` to `createOrder` |
| `POST /api/internal/loyalty/retry-awards` | `requireLoyaltyRetrySecret` | none | `{ processed, resolved, stillFailing }` |
| `GET /api/account/profile`, `GET /api/account/top-products`, `GET /api/checkout/customer-data` (changed) | unchanged | unchanged | use `customerService.getLinkedCustomer(userId)` instead of `storage.getCustomerByEmail` (`routes.ts:3111,3255,3365`) |
| `PUT /api/account/profile` (changed) | `isAuthenticated` | existing fields | passes `req.user.emailConfirmed`; maps `AccountNotLinkedError` to 409 |
| `DELETE /api/customers/:id` (changed, `routes.ts:646-671`) | `isAdmin` | `parseIdParam` | calls `customerService.deleteCustomer`; maps `CustomerDeletionBlockedError` to 400 `{ error: "Cannot delete customer", message }` (same response shape as today, `routes.ts:653-656`) |

`LoyaltyHistoryResponse`:
```ts
export interface LoyaltyHistoryEntry {
  id: number; createdAt: string; channel: "webshop" | "pos"; reference: string; points: number;
}
export interface LoyaltyHistoryResponse {
  isMember: boolean; balance: number; page: number; pageSize: 50;
  totalEntries: number; hasMore: boolean; entries: LoyaltyHistoryEntry[];
}
```

`requireLoyaltyRetrySecret` (new, exported from `server/security.ts`):
```ts
import { createHash, timingSafeEqual } from "crypto";
export const requireLoyaltyRetrySecret: RequestHandler = (req, res, next) => {
  const expected = process.env.LOYALTY_RETRY_SECRET;
  if (!expected) return res.status(503).json({ error: "Loyalty retry is not configured", code: "RETRY_NOT_CONFIGURED" });
  const header = req.get("authorization") ?? "";
  const provided = header.startsWith("Bearer ") ? header.slice(7) : "";
  const a = createHash("sha256").update(provided).digest();
  const b = createHash("sha256").update(expected).digest();
  if (!timingSafeEqual(a, b)) return res.status(401).json({ error: "Unauthorized", code: "UNAUTHORIZED" });
  next();
};
```
Hashing both sides to a fixed length lets `timingSafeEqual` run on equal-length buffers. The secret is never logged.

### Cloud Scheduler setup (retry every 5 minutes, N = 5)
The backend runs on Cloud Run (`Dockerfile`; `vercel.json:7` rewrites `/api/*` to `https://kruzeapp-76726500838.europe-west1.run.app`). Vercel only serves static assets and proxies `/api`, so the job must target the Cloud Run URL directly, not the Vercel domain. The service is publicly reachable (the Vercel rewrite and the Mollie webhook already call it), so the shared secret is the access control. Unverified: Cloud Run ingress and authentication settings, min-instances and CPU allocation; there is no deploy configuration in the repository.
1. Generate a secret (at least 32 random bytes): `openssl rand -base64 48`.
2. Store it in Secret Manager and expose it to the Cloud Run service as env var `LOYALTY_RETRY_SECRET`.
3. Create the job:
```
gcloud scheduler jobs create http loyalty-retry-awards \
  --location=europe-west1 \
  --schedule="*/5 * * * *" --time-zone="Europe/Brussels" \
  --uri="https://kruzeapp-76726500838.europe-west1.run.app/api/internal/loyalty/retry-awards" \
  --http-method=POST \
  --headers="Authorization=Bearer <secret value>" \
  --attempt-deadline=60s
```
4. Verify: trigger the job manually (`gcloud scheduler jobs run loyalty-retry-awards --location=europe-west1`) and expect HTTP 200 and `{ processed: 0, ... }`; a wrong secret returns 401.
5. Add `LOYALTY_RETRY_SECRET` to `.env.example` (unverified: whether the file exists; `.claude/rules/security.md` refers to it).
The global limiter (`server/index.ts:118-124,133`, 1000 requests per 15 minutes per IP) is far above 288 calls per day.

## Frontend Changes

Design references: `docs/design/design_guidelines.md` (read only through targeted search: POS touch targets at lines 146 and 234; the full file was not read), `.claude/rules/frontend.md` (brand colours, Inter/Playfair/JetBrains Mono, shadcn, TanStack Query, Wouter). Numbers in nl-BE (`Intl.NumberFormat("nl-BE")`), currency `€ 26,40` through the existing `formatCurrency` from `@/lib/currency` (imported in `Account.tsx:8`). Dates with `toLocaleDateString("nl-BE")`. No `any`.

### Admin: `client/src/pages/admin/Settings.tsx` (9.1.2)
- Add `TabsTrigger value="loyalty"` (`data-testid="tab-loyalty"`, label "Loyalty") after the shipping trigger (line 105) and `TabsContent value="loyalty"` with `<LoyaltyTab />` after line 154.
- `LoyaltyTab` is modelled on `ShippingTab` (line 2745): a `Card` titled "Loyalty Earn Rate", description "Points a member earns per EUR 1 spent on products (incl. VAT, excl. shipping)". One text input (`inputMode="decimal"`, `data-testid="input-earn-rate"`) shown with a comma decimal (nl-BE, `0,5`), validated with `earnRateInputSchema`, inline error messages, Save button (`data-testid="button-save-earn-rate"`).
- Query key `["/api/settings/loyalty-params"]`; mutation `apiRequest("POST", "/api/settings/loyalty-params", { earnRate })`, invalidates the key on success, toast "Earn rate saved". On a server rejection the toast shows the server message and the field keeps the stored value after refetch.

### Storefront: `client/src/pages/storefront/Account.tsx` (9.1.6)
- Add `{ key: "loyalty", label: "Mijn Punten", shortLabel: "Punten", icon: Award }` to `TABS` (line 143); import `Award` from `lucide-react` (already imported at lines 13-21).
- The tab bar uses `grid-cols-3` (line 158): change to `grid-cols-4`, and change the last-tab border rule at line 177, which is hard-coded to `tab.key !== "top-products"`, so it compares with the last element of `TABS`.
- Render `{activeTab === "loyalty" && <LoyaltyTab />}` after line 192.
- `LoyaltyTab` (same file, like `OrdersTab`): `useQuery<LoyaltyHistoryResponse>` on `["/api/account/loyalty", page]`. Shows the balance prominently (`Intl.NumberFormat("nl-BE")`, label "punten"), then a history list: date, channel ("Webshop" or "Winkel"), reference, points as `+42`. Pager "Vorige" and "Volgende" driven by `page` and `hasMore`. 50 entries per page. Empty state when `totalEntries === 0`: "Je hebt nog geen punten. Je verdient punten bij elke betaalde bestelling in de webshop, en in de winkel wanneer je ledencode wordt gescand."
- `data-testid`: `loyalty-balance`, `loyalty-entry-<id>`, `loyalty-empty`, `loyalty-next`, `loyalty-prev`.
- SPEC-002 adds the member-code section inside this same tab.

### Storefront: `client/src/pages/storefront/Checkout.tsx`
Remove `isGuest: !isAuthenticated` from the request body (line 230); the server decides attribution from the session cookie. (The server still accepts the field so older clients do not break.)

### Hooks and components
No new shared hooks. `LoyaltyTab` components stay local to their pages like the existing tab components.

## Security Considerations

- **Auth and roles.**
  - Admin only: `GET`/`POST /api/settings/loyalty-params` (`isAdmin`).
  - Authenticated customer: `GET /api/account/loyalty`; the customer id is read from `users.customer_id` on the server, never from client input, so a member sees only their own balance and history.
  - Public with optional session: `POST /api/checkout/create` (`optionalAuth`).
  - Public, verified upstream: the Mollie webhook is unchanged (`routes.ts:3195-3207`: payment id validated, status fetched from Mollie with the server-side key).
  - Secret-protected: `POST /api/internal/loyalty/retry-awards` (`LOYALTY_RETRY_SECRET`, `timingSafeEqual`, 503 when unset so it fails closed). No JWT or cookie applies to the Scheduler call.
  - `.claude/rules/security.md` says routes use `requireAuth`; the actual middleware names in the code are `isAuthenticated` and `isAdmin` (`supabaseAuth.ts:330,392`).
- **Existing attribution bypass, fixed here as a security fix.** Today `POST /api/checkout/create` has no auth (`routes.ts:3172`). `isGuest` is taken from the request body (`routes.ts:3168`), and when it is false, `findOrCreateCustomer` matches any existing customer by the typed email (`checkoutService.ts:576-594`). An anonymous caller can send a member's email with `isGuest: false` and have the sale attributed to that member's customer record. After this spec the server derives attribution only from the verified session, ignores `isGuest`, uses the linked record for a confirmed logged-in member, and the guest customer otherwise.
- **Email confirmation.** Linking requires `email_confirmed_at` from Supabase (server side) and refuses otherwise, in addition to the project setting "Confirm email" being ON (go-live checklist). Without the setting, signup would create confirmed accounts for unverified addresses and let anyone claim another person's customer record.
- **Input validation (Zod on every POST/PATCH/PUT).** `earnRateInputSchema`; the `page` query schema; the existing checkout schema. The retry endpoint takes no body.
- **Rate limiting.** The retry endpoint is covered by the global `/api` limiter (`server/index.ts:133`); no extra limiter is needed for a secret-protected, low-volume endpoint. Member-code limiters are in SPEC-002.
- **No PII in logs.** Award failures log the sale id and an error code only. The global response logger (`server/index.ts:34-47`) appends response bodies to log lines; the history endpoint returns no email, name or address (balance and entries only).
- **Secrets.** `LOYALTY_RETRY_SECRET` lives in Secret Manager and an env var; never in source.
- **Error responses.** Descriptive per `coding-standards.md`, no stack traces or database errors to clients.
- **Data integrity.** Ledger append-only enforced by trigger (BR-9); unique partial index (BR-10); `ON CONFLICT DO NOTHING`.

## Test Plan (TDD - tests are written BEFORE implementation)

Conventions (verified from `server/salesService.test.ts` and `server/parameterService.test.ts`): Vitest with `globals: true` and `happy-dom` (`vitest.config.ts`), `vi.mock('./db')`, `vi.mock('./storage')`, `vi.mock('@shared/schema', ...)`, `vi.mock('drizzle-orm', ...)`. Annotate each test with `// Story: 9.1.x | UC: UC-032` (`.claude/rules/testing.md`).

**Tooling gap.** `supertest` and `@testing-library/react` are not in `package.json` (verified by reading the dependency lists). Add as devDependencies: `supertest`, `@types/supertest`, `@testing-library/react`, `@testing-library/user-event`. Unverified, verify before install: run `npm audit` and check licences. Route tests mock `./supabaseAuth` (the module throws at import if `SUPABASE_URL` or `SUPABASE_ANON_KEY` is unset, `supabaseAuth.ts:7-13`) and `./db`. Unverified: that `registerRoutes` can be imported under those mocks (the other modules it imports, such as `./gcs` and `./resendClient`, were not read). Fallback if not: extract the loyalty route registrations into `server/loyaltyRoutes.ts` exporting `registerLoyaltyRoutes(app)` called from `routes.ts`, and test that file in isolation (Open Item OI-3).

**Integration tests** that need a real database (unique index, trigger, `FOR UPDATE` serialisation) live in `server/loyalty.integration.test.ts` wrapped in `describe.skipIf(!process.env.TEST_DATABASE_URL)`. The test database setup is unverified (no test database convention exists in the repository). Mocks cannot prove those database guarantees, so these rows are marked "integration".

Test files (exact paths):
- `server/loyaltyService.test.ts` (new)
- `server/customerService.test.ts` (new)
- `server/checkoutService.test.ts` (new)
- `server/posService.test.ts` (new)
- `server/salesService.test.ts` (existing; add `vi.mock('./loyaltyService', ...)` next to the existing mocks at lines 11-47, then new cases)
- `server/security.test.ts` (existing; add the retry-secret middleware cases)
- `server/routes.loyalty.test.ts` (new, supertest)
- `server/loyalty.integration.test.ts` (new, gated)
- `client/src/pages/admin/Settings.loyalty.test.tsx` (new)
- `client/src/pages/storefront/Account.loyalty.test.tsx` (new)

### Acceptance Criteria to Test Cases

#### Story 9.1.1: Link Storefront Account to a Customer Record (11 criteria)
| # | Acceptance Criterion | Type | Test Description | File |
|---|---|---|---|---|
| 1 | Link to existing record with same email, case and surrounding spaces ignored | service | `linkAccountToCustomer` with `" Jan@Example.COM "` and a customer stored as `jan@example.com` sets `users.customer_id` to that id, `created=false` | `server/customerService.test.ts` |
| 2 | Several matches: record with the most recent sale | service | Three candidates returned by the ordered query; asserts the query orders by max sale date desc then sale id desc and the first row is linked | `server/customerService.test.ts` |
| 3 | Several matches, none with a sale: most recently added | service | No sale dates; asserts ordering falls back to `creation_date` desc then `id` desc | `server/customerService.test.ts` |
| 4 | No match: new record added and linked | service | Query returns none; a customer is inserted with the user's name and email, `b2bClient=false`, and linked; `created=true` | `server/customerService.test.ts` |
| 5 | B2B matching record is linked | service | Matching customer has `b2bClient=true`; it is linked (no filter on B2B) | `server/customerService.test.ts` |
| 6 | Unconfirmed email: not linked, not a member; linked at first sign-in after confirmation | service | `emailConfirmed=false` returns `linked=false` with no DB writes; a second call with `true` links | `server/customerService.test.ts` |
| 6b | (same, auth layer) | route | Login handler calls the link with `emailConfirmed=Boolean(email_confirmed_at)`; signup handler never calls it | `server/routes.loyalty.test.ts` |
| 7 | Signing in again does not relink or duplicate | service | `users.customer_id` already set: returns it, no insert and no update | `server/customerService.test.ts` |
| 8 | Two simultaneous first sign-ins: one link, at most one new record | integration | Two parallel `linkAccountToCustomer` for one user against a real database: one `customer` row, one link | `server/loyalty.integration.test.ts` |
| 8b | (same, unit) | service | Asserts the users row is locked `FOR UPDATE` before the match query, and the update has `customer_id IS NULL` | `server/customerService.test.ts` |
| 9 | Account that saved its profile before go-live is linked to its existing record, no duplicate | service | Customer created earlier by `updateAccountProfile` with the same email is matched and linked, no insert | `server/customerService.test.ts` |
| 10 | Confirmed member who never opened the profile has a linked record at first logged-in checkout | service | `findOrCreateCustomer` with a session of an unlinked confirmed user calls `ensureLinkedCustomer`, which links and returns the customer id | `server/checkoutService.test.ts` |
| 11 | First profile save does not create a second record for a linked account | service | `updateAccountProfile` for a linked user updates by `customer_id`, no insert | `server/customerService.test.ts` |
| extra | Customer 673 is excluded from matching | service | A user whose email equals customer 673's email is not linked to 673 (a new record is created instead) | `server/customerService.test.ts` |

#### Story 9.1.2: Set the Loyalty Earn Rate (6 criteria)
| # | Acceptance Criterion | Type | Test Description | File |
|---|---|---|---|---|
| 1 | Settings shows the current rate; a fresh system shows 1 | component | Tab renders the fetched value `1`; service test: missing parameter returns `"1"` | `client/src/pages/admin/Settings.loyalty.test.tsx`, `server/loyaltyService.test.ts` |
| 2 | Save a new rate and see it after reload | service + component | `setEarnRate("2")` stores `"2"`; `getEarnRateSetting` returns it; component refetches after save | `server/loyaltyService.test.ts`, `client/src/pages/admin/Settings.loyalty.test.tsx` |
| 3 | A rate of 0.5 is accepted | service | `earnRateInputSchema.parse("0.5")` and `parse("0,5")` and `parse(0.5)` give `"0.5"`; stored as `0.5` | `server/loyaltyService.test.ts` |
| 4 | More than 2 decimals rejected with a clear message | service | `"0.555"` fails with "at most 2 decimals" | `server/loyaltyService.test.ts` |
| 5 | Zero, negative, non-number or above 100 rejected; stored rate unchanged | service + route | `"0"`, `"-1"`, `"abc"`, `"100.01"`, `""` rejected with their messages; `setParameter` never called; route returns 400 `INVALID_EARN_RATE` | `server/loyaltyService.test.ts`, `server/routes.loyalty.test.ts` |
| 5b | Boundary 100 accepted | service | `"100"` accepted | `server/loyaltyService.test.ts` |
| 6 | Only an admin can read or change the rate; the server validates | route | GET and POST return 401 without a session and 403 for a non-admin; POST with an invalid body returns 400 | `server/routes.loyalty.test.ts` |

#### Story 9.1.3: Earn Points on a Paid Webshop Order (9 criteria)
| # | Acceptance Criterion | Type | Test Description | File |
|---|---|---|---|---|
| 1 | Mollie webhook sets a member's sale PAID: one entry linked to the sale | service | `completeSale` for a member order calls `awardPointsSafely(saleId)` after the transaction; `awardPointsForSale` inserts one entry with that `saleId` | `server/checkoutService.test.ts`, `server/loyaltyService.test.ts` |
| 2 | Basis: products incl. VAT, after line and sale discounts, excl. shipping line | service + integration | `getEarnBasis` query excludes products 145 and 146 and subtracts `discountWithVat`, clamped at 0 (assert generated SQL parameters); integration seeds lines plus a shipping line and a discount | `server/loyaltyService.test.ts`, `server/loyalty.integration.test.ts` |
| 3 | Points = basis x rate, rounded down (EUR 42,50 + EUR 5,00 shipping at 1 = 42) | service | `computeEarnPoints(4250, 100) === 42`; end to end with a mocked basis of 4250 cents gives an entry of 42 | `server/loyaltyService.test.ts` |
| 4 | Rate 0.5: EUR 42,50 earns 21 | service | `computeEarnPoints(4250, 50) === 21` | `server/loyaltyService.test.ts` |
| 5 | Earn rate in force stored on the entry | service | Inserted entry has `earnRate "1.00"` and `basisAmount "42.50"` | `server/loyaltyService.test.ts` |
| 6 | After a rate change, earlier entries keep their rate; later sales use the new rate | service | Award at rate 1, change the setting to 2, award a second sale: second entry stores `2.00`; no update call on the first entry | `server/loyaltyService.test.ts` |
| 7 | Basket under 1 point records no entry | service | Basis 90 cents at rate 1 returns `BELOW_ONE_POINT`; `insertEarnEntry` not called | `server/loyaltyService.test.ts` |
| 8 | Entry never changed or deleted; every entry references a sale | service + integration | Storage exposes no update or delete method; integration: `UPDATE` and `DELETE` and `TRUNCATE` raise the trigger error; `sale_id` is `NOT NULL` | `server/loyaltyService.test.ts`, `server/loyalty.integration.test.ts` |
| 9 | A sale paid before go-live has no entry | service | `completeSale` on the already-paid branch (`checkoutService.ts:382-385`) does not call the award; `updateSaleStatus(PAID)` on a PAID sale does not call it; the retry only reads failure markers | `server/checkoutService.test.ts`, `server/salesService.test.ts`, `server/loyaltyService.test.ts` |
| extra | Rounding half up at cents | service | Basis query rounds to 2 decimals before multiplying (asserted on the query, integration with 41,995 = 42,00) | `server/loyalty.integration.test.ts` |

#### Story 9.1.4: Only the Logged-In Member Earns at Checkout (7 criteria)
| # | Acceptance Criterion | Type | Test Description | File |
|---|---|---|---|---|
| 1 | B2B client earns nothing, even with an account | service | `isQualifyingMember` false when `b2bClient=true` and `hasAccount=true`; `awardPointsForSale` returns `NOT_QUALIFIED` | `server/loyaltyService.test.ts` |
| 2 | Customer with no linked account earns nothing | service | `hasAccount=false` returns `NOT_QUALIFIED` | `server/loyaltyService.test.ts` |
| 3 | Guest customer 673 never earns | service | `customerId=673` returns false even when `hasAccount=true` and `b2bClient=false` | `server/loyaltyService.test.ts` |
| 4 | Logged-in member's sale linked to own record whatever email is typed | service | Session user linked to customer 55, request email `other@x.be`: `createOrder` inserts the sale with `customerId 55`; no email lookup | `server/checkoutService.test.ts` |
| 5 | Guest checkout earns nothing, even when email matches a member | service | No session and request `{ isGuest: false, email: <member email> }`: sale `customerId` is 673 | `server/checkoutService.test.ts` |
| 6 | Guest sale not linked to a member's record | service | The sale created by a guest checkout has `customerId 673` and never a member id | `server/checkoutService.test.ts` |
| 7 | Logged-in unconfirmed account is not a member, not linked | service | Session with `emailConfirmed=false`: customer id is 673; `linkAccountToCustomer` returns `linked=false` | `server/checkoutService.test.ts`, `server/customerService.test.ts` |
| 7b | `optionalAuth` never 401s | route | No cookie: the handler runs with `req.user` undefined; valid cookie: `req.user.emailConfirmed` is set | `server/routes.loyalty.test.ts` |
| 7c | Bypass regression | route | `POST /api/checkout/create` with a member's email and `isGuest:false` and no cookie creates a sale for 673 | `server/routes.loyalty.test.ts` |

#### Story 9.1.5: Points Survive Payment Delays and Failures (7 criteria)
| # | Acceptance Criterion | Type | Test Description | File |
|---|---|---|---|---|
| 1 | Pending or failed payment awards nothing | service | `handleWebhook` for Mollie statuses `open`, `pending`, `failed`, `expired`, `canceled`: `awardPointsSafely` not called | `server/checkoutService.test.ts` |
| 2 | Pending payment later confirmed awards then | service | Webhook with `open`, then with `paid`: the award is called exactly once after the second | `server/checkoutService.test.ts` |
| 3 | Repeated Mollie webhook awards once | service | Second `paid` webhook returns at the already-paid branch; award calls = 1; storage insert returns `undefined` on conflict gives `ALREADY_AWARDED` | `server/checkoutService.test.ts`, `server/loyaltyService.test.ts` |
| 4 | Two concurrent webhooks award once | integration | Two parallel `awardPointsForSale` for one sale: one ledger row | `server/loyalty.integration.test.ts` |
| 4b | (same, unit) | service | Concurrent calls where the second insert returns `undefined`: results `{awarded:true}` then `ALREADY_AWARDED` | `server/loyaltyService.test.ts` |
| 5 | Recording fails: sale stays PAID, customer sees no error | service + route | Insert throws: `awardPointsSafely` resolves without throwing and writes a marker; the webhook route still returns 200 `OK` | `server/loyaltyService.test.ts`, `server/routes.loyalty.test.ts` |
| 6 | Failed award retried at least every N minutes until success, recorded once | service + route | `retryFailedAwards` picks pending markers, awards, resolves them; a second run processes nothing new; endpoint returns counts; marker stores the rate snapshot and retry uses it | `server/loyaltyService.test.ts`, `server/routes.loyalty.test.ts` |
| 6b | Retry endpoint secret | route | 503 when `LOYALTY_RETRY_SECRET` unset, 401 for a wrong or missing bearer, 200 for the right one | `server/routes.loyalty.test.ts`, `server/security.test.ts` |
| 7 | Retry covers only attempted and failed awards | service | `retryFailedAwards` calls only `listPendingAwardFailures`; no storage method scans sales; a PAID sale without a marker is never awarded | `server/loyaltyService.test.ts` |

#### Story 9.1.6: See My Points Balance and History (8 criteria)
| # | Acceptance Criterion | Type | Test Description | File |
|---|---|---|---|---|
| 1 | Balance equals the sum of all entries | service + component | `getBalanceAndHistory` returns `getLedgerBalance`; component shows it with nl-BE formatting | `server/loyaltyService.test.ts`, `client/src/pages/storefront/Account.loyalty.test.tsx` |
| 2 | At most 50 per page, newest first, older reachable | service + component | Page size 50 and `ORDER BY created_at DESC, id DESC` asserted; `hasMore` true when more exist; clicking "Volgende" requests page 2 | `server/loyaltyService.test.ts`, `client/src/pages/storefront/Account.loyalty.test.tsx` |
| 3 | Each entry shows date, channel, reference, points | component | Row renders the four fields | `client/src/pages/storefront/Account.loyalty.test.tsx` |
| 4 | Reference is the webshop order number, or the sale number for POS | service | Webshop row uses `webshop_order.order_number`; POS row uses `#<sale id>` (not the invoice number) | `server/loyaltyService.test.ts`, `server/loyalty.integration.test.ts` |
| 5 | Points shown as positive numbers | component | `+42` rendered for 42 | `client/src/pages/storefront/Account.loyalty.test.tsx` |
| 6 | 0 points: empty state explaining how points are earned | component | `totalEntries=0` renders `loyalty-empty` with the explanation | `client/src/pages/storefront/Account.loyalty.test.tsx` |
| 7 | Balance counts all entries, not only the visible page | service | With 120 entries the balance comes from the SUM query, independent of the page rows | `server/loyaltyService.test.ts` |
| 8 | A member sees only their own balance and history | route + service | 401 without a session; the customer id comes from `users.customer_id`; a `customerId` query parameter is ignored; unlinked user gets `isMember:false` | `server/routes.loyalty.test.ts`, `server/loyaltyService.test.ts` |
| extra | Account tab bar | component | Four tabs render; the last tab has no right border; clicking "Punten" shows the tab | `client/src/pages/storefront/Account.loyalty.test.tsx` |

#### Story 9.1.7: Earn Points When an Admin Makes a Member's Sale Paid (6 criteria)
| # | Acceptance Criterion | Type | Test Description | File |
|---|---|---|---|---|
| 1 | Admin changes a member's sale to PAID: one entry, same basis, rate, rounding, stored rate | service | `updateSaleStatus(id, 4)` from status NEW or INVOICED calls `awardPointsSafely(id)` once, after the transaction | `server/salesService.test.ts` |
| 2 | Sale an admin adds as paid earns once | service | `createSale` with `salesStatus 4` calls the award once; with status 1 it does not | `server/salesService.test.ts` |
| 3 | Not-yet-paid sale edited to paid earns once | service | `updateSale` with `salesStatus 4` calls the award once; with another status it does not | `server/salesService.test.ts` |
| 4 | B2B client's sale earns nothing | service | Award path returns `NOT_QUALIFIED` for a B2B customer reached through the admin hook | `server/loyaltyService.test.ts`, `server/salesService.test.ts` |
| 5 | Admin-paid webshop sale confirmed later by the provider (or reverse) earns once | service | Admin sets PAID, then the webhook hits the already-paid branch: one award call in total; reverse order: `updateSaleStatus(PAID)` on an already PAID sale does not call the award (transition guard) and, even if called, the unique index returns `ALREADY_AWARDED` | `server/salesService.test.ts`, `server/checkoutService.test.ts` |
| 6 | Failed award retried automatically | service | Admin path with a throwing insert leaves a marker; `retryFailedAwards` then awards | `server/loyaltyService.test.ts` |
| extra | Award runs after commit and never fails the admin call | service | The award call happens after the mocked `transaction` resolves; an award failure does not change the returned sale | `server/salesService.test.ts` |

#### Story 9.2.2: A POS Sale for a Selected Member Earns Points (8 criteria)
| # | Acceptance Criterion | Type | Test Description | File |
|---|---|---|---|---|
| 1 | Selected member through existing customer search: one entry on the PAID POS sale | service | `createPOSSale` for a member calls `awardPointsSafely(sale.id)` after the transaction; entry `channel` is `pos` | `server/posService.test.ts`, `server/loyaltyService.test.ts` |
| 2 | Basis, rate, rounding and stored rate as 9.1.3; sale discount reduces basis; no shipping | service + integration | POS sale of EUR 50,00 with a EUR 10,00 fixed discount: basis 4000 cents; percentage discount likewise | `server/loyaltyService.test.ts`, `server/loyalty.integration.test.ts` |
| 3 | Selected customer without a Storefront account earns nothing | service | `hasAccount=false` returns `NOT_QUALIFIED` | `server/loyaltyService.test.ts` |
| 4 | Default Kruze guest customer (673) earns nothing | service | Sale for 673 returns `NOT_QUALIFIED`; no entry | `server/posService.test.ts`, `server/loyaltyService.test.ts` |
| 5 | Selected B2B client earns nothing, sale proceeds | service | `createPOSSale` still returns the sale with an invoice number; the award returns `NOT_QUALIFIED` | `server/posService.test.ts` |
| 6 | Sale without a customer earns nothing | route | `POST /api/pos/sales` without `customerId` returns 400 (existing validation, `routes.ts:2514`); no award call | `server/routes.loyalty.test.ts` |
| 7 | Recording failure does not fail the sale | service | Award insert throws: `createPOSSale` still resolves with the created sale; marker written | `server/posService.test.ts` |
| 8 | Failed award retried automatically | service | Marker from a POS sale is processed by `retryFailedAwards` | `server/loyaltyService.test.ts` |

#### Cross-cutting tests (not tied to a single criterion)
| Area | Type | Test Description | File |
|---|---|---|---|
| Customer deletion guard | service + route | Linked customer, customer with ledger entries and customer with sales are refused with a clear message; unlinked, unused customer is deleted; response shape `{ error: "Cannot delete customer", message }` | `server/customerService.test.ts`, `server/routes.loyalty.test.ts` |
| Account endpoints use the linked customer | route | `GET /api/account/profile`, `top-products`, `checkout/customer-data` return the linked record even when another customer shares the email | `server/routes.loyalty.test.ts` |
| Profile save for unconfirmed account | route | `PUT /api/account/profile` returns 409 with the Dutch message | `server/routes.loyalty.test.ts` |
| Guest constant | service | `GUEST_CUSTOMER_ID === 673` and POS/checkout use the constant | `server/posService.test.ts` |

Totals mapped for this spec: 9.1.1 = 11, 9.1.2 = 6, 9.1.3 = 9, 9.1.4 = 7, 9.1.5 = 7, 9.1.6 = 8, 9.1.7 = 6, 9.2.2 = 8, total 62 acceptance criteria, each with at least one test.

### Service Tests (Vitest, mock storage)
- `LoyaltyService`: happy paths, `NOT_PAID`, `NOT_QUALIFIED`, `BELOW_ONE_POINT`, `ALREADY_AWARDED`, `SALE_NOT_FOUND`; `computeEarnPoints` table (4250/100 = 42, 4250/50 = 21, 99/100 = 0, 100/100 = 1, 0 = 0, large basis); `awardPointsSafely` never throws even when the marker write throws; `retryFailedAwards` success, repeated failure (attempts + 1), resolution on `NOT_QUALIFIED`.
- `CustomerService`: link rules above, `ensureLinkedCustomer`, `deleteCustomer`, `updateAccountProfile` linked and unconfirmed.
- `CheckoutService`: attribution matrix (no session, confirmed session, unconfirmed session) and webhook branches.
- `SalesService` and `posService`: award hooks and guards above.

### Route Tests (supertest)
- Retry endpoint: 503, 401, 200.
- `GET`/`POST /api/settings/loyalty-params`: 401, 403, 400, 200.
- `GET /api/account/loyalty`: 401, 400 on `page=0`, 200 shape.
- `POST /api/checkout/create`: attribution cases; `PUT /api/account/profile`: 409.
- `DELETE /api/customers/:id`: 400 blocked, 200 deleted.

### Component Tests (React Testing Library)
- `Settings.loyalty.test.tsx`: renders the rate; comma input accepted; invalid inputs show messages and do not call the API; save invalidates the query.
- `Account.loyalty.test.tsx`: four tabs; balance, rows, empty state, paging.

### Implementation Order
1. Add devDependencies (`supertest`, `@types/supertest`, `@testing-library/react`, `@testing-library/user-event`) after `npm audit`.
2. Write all failing tests above (service tests first, then routes, then components; gated integration tests last).
3. Schema: edit `shared/schema.ts`, add `scripts/loyalty-setup.sql` and `scripts/loyalty-setup.ts`, add the npm script; run the Migration Procedure on a development database.
4. Storage methods; `shippingService` export; `customerConstants.ts`.
5. `loyaltyService.ts`; make its tests pass.
6. `customerService.ts` link, `ensureLinkedCustomer`, profile and deletion changes; `supabaseAuth.ts` (`emailConfirmed`, `optionalAuth`, login and callback hooks).
7. Award hooks: `checkoutService.ts`, `salesService.ts`, `posService.ts`.
8. Routes in `server/routes.ts` and `requireLoyaltyRetrySecret` in `server/security.ts`.
9. Frontend: Settings `LoyaltyTab`, Account `LoyaltyTab` and tab bar, `Checkout.tsx`.
10. Refactor with all tests green. Run `npm run check` (`tsc`) and `npm test`.
11. Release stories 9.1.3, 9.1.4 and 9.1.6 together (epic dependency order), preceded by the Go-Live Checklist.

## Go-Live Checklist
- [ ] Supabase dashboard: Authentication, Providers, Email: "Confirm email" is ON (PO decision 1; the setting is not visible in the repository, so verify it by hand).
- [ ] `SELECT count(*) FROM loyalty;` returns 0 on production before `db:push`.
- [ ] `npm run db:push` reviewed; answer "create table" (not "rename") to any rename prompt; accept the `loyalty` drop.
- [ ] `npm run db:loyalty-setup` run; verify `SELECT parameter_value FROM parameter_table WHERE parameter_name = 'loyalty_earn_rate'` returns `1`, and that the three triggers exist.
- [ ] `LOYALTY_RETRY_SECRET` set on Cloud Run; Cloud Scheduler job created against the Cloud Run URL; manual run returns 200.
- [ ] Stories 9.1.3, 9.1.4 and 9.1.6 deployed in the same release.
- [ ] Story files updated: 9.1.3 tech pointer mentions the `loyalty` table is dropped through the setup script; SPEC-002 updates the 9.2.3 endpoint pointer.

## Risks and Residual Issues
1. **Crash window.** The award runs after the sale transaction commits. If the process dies between commit and award, no points and no marker exist (a transactional outbox would close this; the PO chose the simpler design). The sale itself is never affected.
2. **Marker write can also fail** (for example the database is down). The failure is logged by sale id only and the award is lost; there is deliberately no sweep (story 9.1.5 AC 7).
3. **Rate snapshot.** The retry uses the snapshotted rate; if the rate read itself failed, the retry uses the current rate.
4. **Webhook idempotency is check-then-act** (`checkoutService.ts:370-385`); two simultaneous webhooks can both pass it (existing behaviour, also affects stock movements). The ledger's unique index keeps points correct; the stock-movement duplication is out of scope.
5. **Channel for admin-entered non-webshop sales** is `pos` (PO decision), because the schema has no other source column.
6. **Account profile or orders for pre-go-live sessions.** A session from before go-live has `customer_id = NULL` until the next sign-in or the first checkout or profile save; read endpoints return `customer: null` until then.
7. **`users.email` staleness.** Email changes in Supabase are not synced; link matching uses the stored email at link time.
8. **Unverified assumptions:** Drizzle partial-index `.where()` support; `email_confirmed_at` on the installed Supabase `User` type; `pool` multi-statement queries; Cloud Run min-instances and ingress settings; `registerRoutes` testability under mocks; existence of `.env.example`.

## Open Items
- OI-1: Confirm the Supabase "Confirm email" setting is ON in the production project (PO decision, needs a manual check).
- OI-2: Confirm the production `loyalty` table is empty (PO stated empty; the checklist verifies it).
- OI-3: Route test harness: if `registerRoutes` cannot be imported under mocks, extract `server/loyaltyRoutes.ts` (changes the confirmed file plan; needs PO approval).
- OI-4: Test database for the gated integration tests (no convention exists).
- OI-5: Verify `supertest` and `@testing-library/*` with `npm audit` and licences before install.

## Quality Checklist (run before saving)
- [x] References link the stories (9.1.1-9.1.7, 9.2.2) and UC-032 with the related UCs.
- [x] Scope states what is and is not included.
- [x] 3-layer architecture respected: routes only parse, authenticate and respond; `db.transaction` only in services; storage has no transactions or business logic. Exception noted: `customerService` runs queries on `tx` inside its own transaction, as it already does at `customerService.ts:163-218`.
- [x] Every file to create or modify has an exact path (see the file lists in Schema, Backend and Frontend; consolidated below).
- [x] Schema changes include Drizzle definitions, Zod insert schemas and the migration step (`db:push` plus `db:loyalty-setup`, as decided by the PO instead of `drizzle-kit generate`).
- [x] Security section covers roles, validation, rate limiting, logging and secrets.
- [x] Test Plan maps all 62 acceptance criteria to tests with type, description and file; implementation order is tests first.
- [x] No `any` in the specified types.
- [x] Design guidelines referenced (partial read, noted).
- [x] `roastService.ts` pattern referenced; existing patterns followed.

## Consolidated File List
Create: `server/loyaltyService.ts`, `server/customerConstants.ts`, `scripts/loyalty-setup.sql`, `scripts/loyalty-setup.ts`, `server/loyaltyService.test.ts`, `server/customerService.test.ts`, `server/checkoutService.test.ts`, `server/posService.test.ts`, `server/routes.loyalty.test.ts`, `server/loyalty.integration.test.ts`, `client/src/pages/admin/Settings.loyalty.test.tsx`, `client/src/pages/storefront/Account.loyalty.test.tsx`.

Modify: `shared/schema.ts`, `server/storage.ts`, `server/shippingService.ts`, `server/customerService.ts`, `server/supabaseAuth.ts`, `server/checkoutService.ts`, `server/salesService.ts`, `server/posService.ts`, `server/routes.ts`, `server/security.ts`, `server/salesService.test.ts`, `server/security.test.ts`, `client/src/pages/admin/Settings.tsx`, `client/src/pages/storefront/Account.tsx`, `client/src/pages/storefront/Checkout.tsx`, `package.json`, `.env.example` (if present).
