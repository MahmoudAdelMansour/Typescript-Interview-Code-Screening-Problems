# Digital Zone  TypeScript Coding Screen Prep

Problems built from the actual business of **Digital Zone** (dz.iq  Iraq): mobile top-ups,
digital vouchers, event tickets (Ticket Zone), travel (Rahal), eSIM, and payments via
QI Card / SuperQI.

Their surface: **instant delivery + real money + millions of users.** Every problem below
sits on that surface.

> Each problem includes **the trap** that separates senior from mid, **the follow-up
> questions** you should expect, and a solution folded inside `<details>`. Try it first.

## How to use this

| Level | Focus | Target time |
|---|---|---|
| L1 | Types, pure functions, normalization | 10–15 min |
| L2 | Data shaping, money, time | 15–20 min |
| L3 | Idempotency and concurrency | 25–30 min |
| L4 | State machines and reconciliation | 25–35 min |
| L5 | Abuse limits and retry semantics | 30–40 min |

**Practice rules:**
1. Write the **types** before the logic. In a TypeScript screen, type design is half the score.
2. Say the **edge cases out loud** before you write the solution. That is the senior signal.
3. No `any`. No `as` without a comment justifying it.
4. Run `tsc --strict`. Without strict mode, TypeScript is doing nothing for you.

```bash
npm init -y && npm i -D typescript tsx @types/node
npx tsc --init --strict --target es2022 --module nodenext --moduleResolution nodenext
```

> Every solution in this file compiles under `tsc --strict` and was executed  the
> expected outputs shown are real program output, not estimates.

---

# L1  Fundamentals

## L1.1  Normalize an Iraqi MSISDN, then resolve the operator

A top-up has to know the carrier before it executes anything. Customers type the number
in whatever shape they like.

```ts
type Operator = "asiacell" | "zain" | "korek";

// The prefix table arrives as config  do not hardcode it in the function
const PREFIXES: ReadonlyArray<readonly [prefix: string, op: Operator]> = [
  ["077", "asiacell"], ["078", "asiacell"],
  ["079", "zain"],     ["078", "zain"],      // note: the overlap is deliberate
  ["075", "korek"],
];

type ParsedMsisdn =
  | { ok: true; national: string; operator: Operator }
  | { ok: false; reason: "invalid_format" | "unknown_operator" | "ambiguous_prefix" };

declare function parseMsisdn(input: string, table = PREFIXES): ParsedMsisdn;
```

**Inputs that must all work:**
```
"+964 770 123 4567"   "00964 770 123 4567"   "07701234567"
"0770-123-4567"       "٠٧٨٠١٢٣٤٥٦٧"  ← Arabic-Indic digits
"7701234567"          ""        "0770123"   "0761234567"
```

**The trap:**
- **Normalize before you match.** Matching the raw input fails on every variant above.
- **Longest-prefix match.** With both `07` and `077` in the table, the longer one wins.
- **`078` maps to two operators in that table.** This happens in reality after number
  portability. The correct behavior is to return **`ambiguous_prefix`**, not to take the
  first match. Silently picking one means topping up the wrong carrier, which is a direct
  financial loss. Missing this is the single worst signal in the problem.
- **Do not throw.** Return a Result type. The MSISDN is user input, and an exception is the
  wrong tool for an expected condition.
- Arabic-Indic digits (`٠-٩`) are a real input in the Iraqi market.

**Follow-ups:**
- How do you make the prefix table updatable without a deploy?
- If portability makes prefixes unreliable, what replaces them?
  (A lookup against the operator's API, with a cache and a TTL.)

<details>
<summary>Solution</summary>

```ts
const ARABIC_INDIC = /[٠-٩]/g;

function toLatinDigits(s: string): string {
  return s.replace(ARABIC_INDIC, (d) => String(d.charCodeAt(0) - 0x0660));
}

function normalize(input: string): string | null {
  const digits = toLatinDigits(input).replace(/[\s\-()]/g, "");
  if (!/^\+?\d+$/.test(digits)) return null;

  let d = digits.replace(/^\+/, "");
  if (d.startsWith("00964")) d = d.slice(5);
  else if (d.startsWith("964")) d = d.slice(3);
  else if (d.startsWith("0")) d = d.slice(1);

  // National form after dropping the trunk zero: 10 digits starting with 7
  if (!/^7\d{9}$/.test(d)) return null;
  return "0" + d; // re-add the zero so it lines up with the prefix table
}

function parseMsisdn(input: string, table = PREFIXES): ParsedMsisdn {
  const national = normalize(input);
  if (!national) return { ok: false, reason: "invalid_format" };

  const sorted = [...table].sort((a, b) => b[0].length - a[0].length);
  const matches = sorted.filter(([p]) => national.startsWith(p));
  if (matches.length === 0) return { ok: false, reason: "unknown_operator" };

  const bestLen = matches[0][0].length;
  const best = matches.filter(([p]) => p.length === bestLen);
  const operators = new Set(best.map(([, op]) => op));

  if (operators.size > 1) return { ok: false, reason: "ambiguous_prefix" };
  return { ok: true, national, operator: best[0][1] };
}
```

Verified output:
```
parseMsisdn("+964 770 123 4567") → { ok: true, national: '07701234567', operator: 'asiacell' }
parseMsisdn("٠٧٨٠١٢٣٤٥٦٧")       → { ok: false, reason: 'ambiguous_prefix' }
parseMsisdn("0770123")           → { ok: false, reason: 'invalid_format' }
```

`ambiguous_prefix` is the whole point: the system would rather refuse than move credit
to the wrong carrier.
</details>

---

## L1.2  Money: IQD alongside USD

Digital Zone sells in Iraqi dinars and in dollars. **The dinar has no practical subunit**
(the fils is obsolete), and the dollar has two decimal places. Any code that assumes `100`
minor units for every currency is broken.

```ts
type Currency = "IQD" | "USD";

// exponent = number of decimal places
const CURRENCY: Record<Currency, { exponent: number; symbol: string }> = {
  IQD: { exponent: 0, symbol: "د.ع" },
  USD: { exponent: 2, symbol: "$" },
};

// All amounts in minor units, always an integer
type Money = { readonly minor: number; readonly currency: Currency };

declare function money(major: number, currency: Currency): Money;
declare function addMoney(a: Money, b: Money): Money;           // same currency only
declare function formatMoney(m: Money, locale?: string): string;
declare function splitMoney(m: Money, ways: number): Money[];   // loses and gains nothing
```

**Inputs:**
```
money(1500, "IQD")  → { minor: 1500, currency: "IQD" }
money(12.5,  "USD") → { minor: 1250, currency: "USD" }
splitMoney(money(1000, "IQD"), 3) → [334, 333, 333]   sums to exactly 1000
splitMoney(money(10, "USD"), 3)   → [334, 333, 333] minor = 3.34 + 3.33 + 3.33
addMoney(money(1,"IQD"), money(1,"USD")) → must fail
```

**The trap:**
- **No floats.** `0.1 + 0.2 !== 0.3`. Everything is an integer in minor units.
- **The multiplication itself is lossy.** `19.99 * 100 = 1998.9999999999998` and
  `0.29 * 100 = 28.999999999999996`. `Math.trunc` gives you `1998`  a lost cent.
  **`Math.round` is mandatory**, not an optimization.
- **`splitMoney` is the real problem.** Division leaves a remainder, and the remainder
  **must be distributed**, not discarded. `1000 / 3` is `333.33`; returning
  `[333, 333, 333]` loses a dinar. This is the **penny allocation problem**.
- **Adding two currencies must be impossible**, by type if you can manage it, by throw
  if you cannot. Prefer the type.
- `Intl.NumberFormat` must take the exponent from config instead of assuming 2.

**Follow-ups:**
- How would you make cross-currency addition fail at **compile time** rather than runtime?
- If a fractional IQD appears out of an FX conversion, where and when do you round?

<details>
<summary>Solution</summary>

```ts
function money(major: number, currency: Currency): Money {
  const { exponent } = CURRENCY[currency];
  const minor = Math.round(major * 10 ** exponent);
  if (!Number.isSafeInteger(minor)) throw new RangeError("amount out of range");
  return { minor, currency };
}

function addMoney(a: Money, b: Money): Money {
  if (a.currency !== b.currency) {
    throw new TypeError(`currency mismatch: ${a.currency} vs ${b.currency}`);
  }
  return { minor: a.minor + b.minor, currency: a.currency };
}

function formatMoney(m: Money, locale = "ar-IQ"): string {
  const { exponent } = CURRENCY[m.currency];
  return new Intl.NumberFormat(locale, {
    style: "currency",
    currency: m.currency,
    minimumFractionDigits: exponent,
    maximumFractionDigits: exponent,
  }).format(m.minor / 10 ** exponent);
}

// largest-remainder: hand out the leftover one unit at a time, from the front
function splitMoney(m: Money, ways: number): Money[] {
  if (!Number.isInteger(ways) || ways <= 0) throw new RangeError("ways must be >= 1");
  const base = Math.trunc(m.minor / ways);
  const remainder = m.minor - base * ways;
  return Array.from({ length: ways }, (_, i) => ({
    minor: base + (i < remainder ? 1 : 0),
    currency: m.currency,
  }));
}
```

**Blocking it at compile time**  a phantom type makes the mistake unrepresentable:

```ts
type TypedMoney<C extends Currency> = { readonly minor: number; readonly currency: C };

function addTyped<C extends Currency>(a: TypedMoney<C>, b: TypedMoney<C>): TypedMoney<C> {
  return { minor: a.minor + b.minor, currency: a.currency };
}
// addTyped(iqd, usd)  ← compile error
```

Offering that in the interview is the strongest single move available in this problem.
</details>

---

# L2  Shaping data, money and time

## L2.1  Group orders by Baghdad local day

A daily merchant report. `createdAt` is stored in UTC; the merchant thinks in Baghdad time.

```ts
type Order = {
  id: string;
  createdAt: string;        // ISO UTC, e.g. "2026-10-21T21:40:00Z"
  total: Money;
  status: "fulfilled" | "refunded" | "failed";
};

type DailyBucket = { day: string; count: number; gross: Money };

declare function groupByLocalDay(
  orders: Order[],
  timeZone: string,        // "Asia/Baghdad"
  currency: Currency,
): DailyBucket[];
```

**The trap:**
- `createdAt.slice(0, 10)` **is wrong.** An order at `21:40Z` is `00:40` the *next* day in
  Baghdad (UTC+3). Orders land in the wrong bucket, and it surfaces as a merchant filing
  "your report doesn't match my sales."
- **Do not hand-roll a fixed offset.** Use `Intl.DateTimeFormat` with `timeZone`. Iraq
  currently observes no DST, but code that hardcodes `+03:00` becomes a bug the moment that
  changes by decree.
- **Empty days.** If a day has no orders, does the report need a zero row or does it skip?
  **Ask.**
- **Mixed currencies.** If the array holds both IQD and USD, summing is meaningless.
  Filter or partition.
- Do `refunded` and `failed` count toward gross? **Ask before assuming.**

<details>
<summary>Solution</summary>

```ts
function localDayKey(iso: string, timeZone: string): string {
  const d = new Date(iso);
  if (Number.isNaN(d.getTime())) throw new RangeError(`bad date: ${iso}`);
  // en-CA yields YYYY-MM-DD directly
  return new Intl.DateTimeFormat("en-CA", {
    timeZone, year: "numeric", month: "2-digit", day: "2-digit",
  }).format(d);
}

function groupByLocalDay(
  orders: Order[], timeZone: string, currency: Currency,
): DailyBucket[] {
  const acc = new Map<string, { count: number; minor: number }>();

  for (const o of orders) {
    if (o.total.currency !== currency) continue;   // never sum across currencies
    if (o.status !== "fulfilled") continue;        // explicit, documented decision
    const day = localDayKey(o.createdAt, timeZone);
    const cur = acc.get(day) ?? { count: 0, minor: 0 };
    cur.count += 1;
    cur.minor += o.total.minor;
    acc.set(day, cur);
  }

  return [...acc.entries()]
    .map(([day, v]) => ({ day, count: v.count, gross: { minor: v.minor, currency } }))
    .sort((a, b) => a.day.localeCompare(b.day));
}
```

Using `en-CA` as the locale is the well-known trick for getting `YYYY-MM-DD` without
assembling parts by hand.
</details>

---

## L2.2  Pick the best promo for a basket

Digital Zone runs campaigns: percentage off, flat amount off, and first-purchase offers.
The customer uses **one**, and you choose the one that's best for them.

```ts
type Promo =
  | { kind: "percent"; code: string; bps: number; maxDiscount?: Money }  // bps = 1/100 of a %
  | { kind: "fixed";   code: string; amount: Money }
  | { kind: "first_purchase"; code: string; bps: number };

type PromoResult = { code: string | null; discount: Money; total: Money };

declare function applyBestPromo(
  subtotal: Money, promos: Promo[], isFirstPurchase: boolean,
): PromoResult;
```

**Inputs:**
```
subtotal = 100_000 IQD
promos = [
  { kind:"percent", code:"SAVE10", bps: 1000, maxDiscount: money(5000,"IQD") },
  { kind:"fixed",   code:"FLAT7K", amount: money(7000,"IQD") },
  { kind:"first_purchase", code:"WELCOME20", bps: 2000 },
]
isFirstPurchase = false  →  FLAT7K (7,000), not SAVE10 (10% = 10k but capped at 5k)
isFirstPurchase = true   →  WELCOME20 (20,000)
```

**The trap:**
- **The `maxDiscount` cap** is exactly why the percentage promo isn't always best. Picking
  by percentage without applying the cap picks wrong.
- **`bps`, not percent.** Basis points keep decimals out of config.
  `bps / 10_000 * amount`, and the result must round to an integer.
- **A `fixed` promo larger than the subtotal** must not produce a negative total. Clamp at zero.
- **An ineligible promo** (`first_purchase` when it isn't a first purchase) is skipped entirely.
- **Ties.** If two promos give the same discount, which wins? Any **stable** rule is fine,
  but it must be **deterministic**  not `Math.random`, not `Object.keys` ordering.
- A currency mismatch between a promo and the subtotal is an error, not something to skip.

<details>
<summary>Solution</summary>

```ts
function discountOf(p: Promo, subtotal: Money, isFirst: boolean): Money | null {
  const pct = (bps: number) => ({
    minor: Math.round((subtotal.minor * bps) / 10_000),
    currency: subtotal.currency,
  });

  switch (p.kind) {
    case "percent": {
      let d = pct(p.bps);
      if (p.maxDiscount) {
        if (p.maxDiscount.currency !== subtotal.currency) throw new TypeError("currency mismatch");
        if (d.minor > p.maxDiscount.minor) d = p.maxDiscount;
      }
      return d;
    }
    case "fixed": {
      if (p.amount.currency !== subtotal.currency) throw new TypeError("currency mismatch");
      return { minor: Math.min(p.amount.minor, subtotal.minor), currency: subtotal.currency };
    }
    case "first_purchase":
      return isFirst ? pct(p.bps) : null;
    default: {
      const _exhaustive: never = p;   // any new kind breaks the build
      return _exhaustive;
    }
  }
}

function applyBestPromo(subtotal: Money, promos: Promo[], isFirst: boolean): PromoResult {
  let best: { code: string; minor: number } | null = null;

  for (const p of promos) {
    const d = discountOf(p, subtotal, isFirst);
    if (!d || d.minor <= 0) continue;
    // tie → lowest code alphabetically, so the result is deterministic
    if (!best || d.minor > best.minor || (d.minor === best.minor && p.code < best.code)) {
      best = { code: p.code, minor: d.minor };
    }
  }

  const discount = { minor: best?.minor ?? 0, currency: subtotal.currency };
  return {
    code: best?.code ?? null,
    discount,
    total: { minor: Math.max(0, subtotal.minor - discount.minor), currency: subtotal.currency },
  };
}
```

The `never` in the `default` branch is what makes TypeScript protect you: any new `kind`
added to the union stops the build until it's handled.
</details>

---

# L3  Idempotency and concurrency

> **These two are the most important problems in the file.** Their business is instant
> delivery against real money, so these are what actually determine whether you fit the
> domain.

## L3.1  Idempotent top-up

A customer tops up 10,000 dinars. The network drops and they tap again. They **must not**
be charged twice.

```ts
type TopUpRequest = {
  idempotencyKey: string;
  msisdn: string;
  amount: Money;
};

type TopUpResult =
  | { status: "fulfilled"; orderId: string; operatorRef: string }
  | { status: "failed"; reason: string };

type StoredEntry =
  | { state: "in_progress"; requestHash: string; startedAt: number }
  | { state: "completed";   requestHash: string; result: TopUpResult };

interface IdempotencyStore {
  /** Claims the key. Returns null if the claim succeeded, or the existing entry if taken. */
  claim(key: string, requestHash: string): Promise<StoredEntry | null>;
  complete(key: string, result: TopUpResult): Promise<void>;
  release(key: string): Promise<void>;
}

declare function idempotentTopUp(
  req: TopUpRequest,
  store: IdempotencyStore,
  execute: (req: TopUpRequest) => Promise<TopUpResult>,
): Promise<TopUpResult>;
```

**Required behavior:**

| Case | Result |
|---|---|
| New key | Execute, store the result, return it |
| Same key + same payload + `completed` | Return the stored result, **no execution** |
| Same key + same payload + `in_progress` | Throw `ConflictError` (409) |
| Same key + **different payload** | Throw `IdempotencyMismatchError` (422) |
| Execution threw | Release the key so a retry can work |

**The trap:**
- **`claim` must be atomic.** A `get` followed by a `set` lets two concurrent requests both
  pass the check. In Postgres: `INSERT ... ON CONFLICT (key) DO NOTHING RETURNING`.
  In Redis: `SET key val NX`. **The unique constraint is the defense, not the `if`.**
- **Write the key before executing, not after.** Written afterward, the second request finds
  nothing and executes too  reintroducing exactly the bug you were preventing.
- **The `requestHash`.** Same key with a different payload is a client bug.
  **Reject with 422; do not execute.** Most candidates forget this branch entirely.
- **`in_progress` is not a terminal error.** A `409` means "ask again shortly," not "it failed."
- **Store the whole `result`** so a retry returns the same status code and the same body.
- **TTL.** Keys cannot live forever. 24 hours is a reasonable default.

**Follow-ups:**
- If the process dies while `in_progress`, the key is stuck forever. What's the fix?
  (A TTL on `in_progress` plus a sweeper that releases after longer than max execution time.)
- Why isn't the `requestHash` computed over the whole payload? (Fields that legitimately
  change between retries, like a client timestamp, have to be excluded.)
- `execute` times out  do you `release`? **(No. See L5.2.)**

<details>
<summary>Solution</summary>

```ts
class ConflictError extends Error {
  readonly httpStatus = 409;
  constructor(msg = "request already in progress") { super(msg); this.name = "ConflictError"; }
}
class IdempotencyMismatchError extends Error {
  readonly httpStatus = 422;
  constructor(msg = "idempotency key reused with a different payload") {
    super(msg); this.name = "IdempotencyMismatchError";
  }
}

function hashRequest(req: TopUpRequest): string {
  // stable fields only, in a fixed order
  return JSON.stringify([req.msisdn, req.amount.minor, req.amount.currency]);
}

async function idempotentTopUp(
  req: TopUpRequest,
  store: IdempotencyStore,
  execute: (req: TopUpRequest) => Promise<TopUpResult>,
): Promise<TopUpResult> {
  const hash = hashRequest(req);
  const existing = await store.claim(req.idempotencyKey, hash);

  if (existing) {
    if (existing.requestHash !== hash) throw new IdempotencyMismatchError();
    if (existing.state === "completed") return existing.result;
    throw new ConflictError();
  }

  try {
    const result = await execute(req);
    await store.complete(req.idempotencyKey, result);
    return result;
  } catch (err) {
    // Only release when we are certain no side effect occurred
    if (isDefinitelyNotExecuted(err)) {
      await store.release(req.idempotencyKey);
    } else {
      // Outcome unknown  keep the key claimed and let reconciliation settle it
      await store.complete(req.idempotencyKey, {
        status: "failed", reason: "unknown_outcome_pending_reconciliation",
      });
    }
    throw err;
  }
}

function isDefinitelyNotExecuted(err: unknown): boolean {
  // e.g. local validation errors, or a 4xx from the operator: we know it didn't happen
  return err instanceof TypeError || err instanceof RangeError;
}
```

**The point worth making out loud:** the `catch` distinguishes a *confirmed failure* from an
*unknown outcome*. Blindly releasing on any exception is a **bug**  a timeout lands in that
same `catch` while the operation may well have succeeded upstream. Naming that difference
wins the problem.
</details>

---

## L3.2  Allocating voucher codes under concurrency

You hold 1,000 PUBG UC codes. 5,000 requests arrive in the same second. One code **must
never** go to two customers.

```ts
type VoucherCode = { id: string; code: string };

type Reservation = { reservationId: string; codeId: string; expiresAt: number };

type ReserveResult =
  | { ok: true; reservation: Reservation }
  | { ok: false; reason: "out_of_stock" };

interface VoucherPool {
  /** Holds one code for ttlMs. Idempotent on orderId. */
  reserve(productId: string, orderId: string, ttlMs: number): Promise<ReserveResult>;
  /** Finalizes the hold and returns the code. Idempotent. */
  commit(reservationId: string): Promise<VoucherCode>;
  /** Returns the code to stock. Idempotent. */
  release(reservationId: string): Promise<void>;
  /** Returns every expired hold to stock. Runs on a schedule. */
  sweepExpired(now: number): Promise<number>;
}
```

Implement it over an in-memory `Map`, correct under concurrent calls (`Promise.all`).

**The trap:**
- **`reserve` must be idempotent on `orderId`.** If the worker retries, it must not take a
  second code  that second code is simply lost from inventory.
- **Order of operations:** `reserve` → `capture payment` → `commit`. Capturing first and
  then discovering empty stock forces a refund, which costs money and carries chargeback risk.
- **Expiry is not optional.** If the customer closes the tab after reserving, the code has to
  come back. Without `sweepExpired`, inventory leaks until it's exhausted on paper only.
- **`commit` after expiry** needs an explicit decision. Failing is the right default, since
  the code may already belong to someone else.
- **`release` must be idempotent**  a code must never return to stock twice and clone itself.
- The SQL equivalent is **`SELECT … FOR UPDATE SKIP LOCKED`**. Name it. Without `SKIP LOCKED`,
  5,000 requests queue on the same row.

**Follow-ups:**
- How would you write this in Postgres? (The answer above  and this is a near-certain follow-up.)
- How do you alert before stock runs out? (A ratio plus a burn rate, not an absolute threshold.)
- If `commit` succeeds but the email fails, did the customer receive it?
  (Yes. The source of truth is their account in the app; email is a notification.)

<details>
<summary>Solution</summary>

```ts
type Slot =
  | { code: VoucherCode; state: "available" }
  | { code: VoucherCode; state: "reserved"; reservationId: string; orderId: string; expiresAt: number }
  | { code: VoucherCode; state: "committed"; reservationId: string };

class InMemoryVoucherPool implements VoucherPool {
  private slots = new Map<string, Slot[]>();                    // productId → slots
  private byReservation = new Map<string, { productId: string; codeId: string }>();
  private byOrder = new Map<string, Reservation>();             // the idempotency index
  private seq = 0;

  constructor(seed: Record<string, VoucherCode[]>) {
    for (const [pid, codes] of Object.entries(seed)) {
      this.slots.set(pid, codes.map((code) => ({ code, state: "available" as const })));
    }
  }

  async reserve(productId: string, orderId: string, ttlMs: number): Promise<ReserveResult> {
    const prior = this.byOrder.get(orderId);
    if (prior) return { ok: true, reservation: prior };          // idempotent

    const slots = this.slots.get(productId) ?? [];
    const idx = slots.findIndex((s) => s.state === "available");
    if (idx === -1) return { ok: false, reason: "out_of_stock" };

    const slot = slots[idx];
    const reservationId = `rsv_${++this.seq}`;
    const expiresAt = Date.now() + ttlMs;

    slots[idx] = { code: slot.code, state: "reserved", reservationId, orderId, expiresAt };
    this.byReservation.set(reservationId, { productId, codeId: slot.code.id });
    const reservation = { reservationId, codeId: slot.code.id, expiresAt };
    this.byOrder.set(orderId, reservation);
    return { ok: true, reservation };
  }

  async commit(reservationId: string): Promise<VoucherCode> {
    const slot = this.findSlot(reservationId);
    if (!slot) throw new Error(`unknown reservation ${reservationId}`);
    if (slot.state === "committed") return slot.code;            // idempotent
    if (slot.state !== "reserved") throw new Error("reservation no longer held");
    if (slot.expiresAt <= Date.now()) throw new Error("reservation expired");

    this.replace(reservationId, { code: slot.code, state: "committed", reservationId });
    return slot.code;
  }

  async release(reservationId: string): Promise<void> {
    const slot = this.findSlot(reservationId);
    if (!slot || slot.state !== "reserved") return;              // idempotent
    this.byOrder.delete(slot.orderId);
    this.replace(reservationId, { code: slot.code, state: "available" });
    this.byReservation.delete(reservationId);
  }

  async sweepExpired(now: number): Promise<number> {
    let freed = 0;
    for (const slots of this.slots.values()) {
      for (let i = 0; i < slots.length; i++) {
        const s = slots[i];
        if (s.state === "reserved" && s.expiresAt <= now) {
          this.byOrder.delete(s.orderId);
          this.byReservation.delete(s.reservationId);
          slots[i] = { code: s.code, state: "available" };
          freed++;
        }
      }
    }
    return freed;
  }

  private findSlot(reservationId: string): Slot | undefined {
    const ref = this.byReservation.get(reservationId);
    if (!ref) return undefined;
    return this.slots.get(ref.productId)?.find((s) => s.code.id === ref.codeId);
  }

  private replace(reservationId: string, next: Slot): void {
    const ref = this.byReservation.get(reservationId)!;
    const slots = this.slots.get(ref.productId)!;
    slots[slots.findIndex((s) => s.code.id === ref.codeId)] = next;
  }
}
```

**Why this is safe in JavaScript:** the event loop lets `reserve` run to completion without
interruption, because there is no `await` between the read and the write. **Say that sentence
in the interview**, then immediately add that it breaks the moment you introduce an `await`
in the middle or run more than one process  at which point atomicity has to move down into
the database.

**The Postgres equivalent:**
```sql
BEGIN;
SELECT id, code FROM voucher_codes
 WHERE product_id = $1 AND status = 'available'
 ORDER BY id LIMIT 1
   FOR UPDATE SKIP LOCKED;
UPDATE voucher_codes
   SET status='reserved', order_id=$2, expires_at = now() + interval '10 minutes'
 WHERE id = $3;
COMMIT;
```
</details>

---

# L4  State machines and reconciliation

## L4.1  Order lifecycle with out-of-order events

PSP webhooks arrive in **no guaranteed order**. You can receive `refunded` before `captured`.

```ts
type OrderState =
  | "pending" | "authorized" | "captured"
  | "fulfilling" | "fulfilled" | "refunded" | "failed";

type OrderEvent =
  | { type: "authorize"; at: number }
  | { type: "capture";   at: number }
  | { type: "start_fulfilment"; at: number }
  | { type: "deliver";   at: number; codeId: string }
  | { type: "refund";    at: number; reason: string }
  | { type: "fail";      at: number; reason: string };

type Order = {
  id: string;
  state: OrderState;
  lastEventAt: number;      // for out-of-order handling
  codeId?: string;
};

type Applied =
  | { ok: true;  order: Order }
  | { ok: false; kind: "illegal_transition" | "stale_event" | "terminal"; order: Order };

declare function applyEvent(order: Order, ev: OrderEvent): Applied;
```

**Legal transitions, and only these:**
```
pending     → authorized | failed
authorized  → captured   | failed
captured    → fulfilling | refunded
fulfilling  → fulfilled  | failed | refunded
fulfilled   → refunded
refunded    → (terminal)
failed      → (terminal)
```

**The trap:**
- **Reject backward transitions; do not apply them.** A `refund` arriving while the order is
  `pending` does not mean refund  it means an event is missing. **Log it and drop the effect.**
- **`stale_event`.** If `ev.at < order.lastEventAt`, the event is old. Don't apply it. This is
  more precise than "illegal transition"  distinguish the two in the return value.
- **Terminal states.** `refunded` and `failed` have no exit. Any later event is `terminal`.
- **Make illegal states unrepresentable** where you can: `fulfilled` **must** carry a `codeId`.
  That's a discriminated union on the state, not an optional field.
- **Don't throw**  the webhook handler has to return `200` or the PSP retries forever.
  Return a result and log.

**Follow-ups:**
- How do you prevent an `Order` in state `fulfilled` with an empty `codeId` at **compile time**?
- `capture` arrives twice  what distinguishes `illegal_transition` from an idempotent no-op?
  (Same event id → no-op. A new event of the same type → suspicious, alert on it.)

<details>
<summary>Solution</summary>

```ts
const TRANSITIONS: Record<OrderState, ReadonlySet<OrderState>> = {
  pending:    new Set<OrderState>(["authorized", "failed"]),
  authorized: new Set<OrderState>(["captured", "failed"]),
  captured:   new Set<OrderState>(["fulfilling", "refunded"]),
  fulfilling: new Set<OrderState>(["fulfilled", "failed", "refunded"]),
  fulfilled:  new Set<OrderState>(["refunded"]),
  refunded:   new Set<OrderState>(),
  failed:     new Set<OrderState>(),
};

const TARGET: Record<OrderEvent["type"], OrderState> = {
  authorize: "authorized",
  capture: "captured",
  start_fulfilment: "fulfilling",
  deliver: "fulfilled",
  refund: "refunded",
  fail: "failed",
};

function applyEvent(order: Order, ev: OrderEvent): Applied {
  if (TRANSITIONS[order.state].size === 0) {
    return { ok: false, kind: "terminal", order };
  }
  if (ev.at < order.lastEventAt) {
    return { ok: false, kind: "stale_event", order };
  }
  const next = TARGET[ev.type];
  if (!TRANSITIONS[order.state].has(next)) {
    return { ok: false, kind: "illegal_transition", order };
  }
  return {
    ok: true,
    order: {
      ...order,
      state: next,
      lastEventAt: ev.at,
      codeId: ev.type === "deliver" ? ev.codeId : order.codeId,
    },
  };
}
```

**The version that makes impossible states impossible**  this is the differentiator:

```ts
type OrderShape =
  | { state: "pending" | "authorized" | "captured" | "fulfilling"; id: string; lastEventAt: number }
  | { state: "fulfilled"; id: string; lastEventAt: number; codeId: string }   // codeId required
  | { state: "refunded";  id: string; lastEventAt: number; refundReason: string }
  | { state: "failed";    id: string; lastEventAt: number; failReason: string };
```

Now `{ state: "fulfilled" }` without a `codeId` **does not compile**. That is what the job
description means by "self-documenting code."
</details>

---

## L4.2  Reconciling your orders against the PSP

The report that catches "the customer paid and got nothing."
**This is the problem that proves you understand the domain.**

```ts
type OrderRow = { orderId: string; state: OrderState; amount: Money; capturedAt?: number };
type PspRow   = { orderId: string; pspRef: string; amount: Money; status: "captured" | "refunded" };

type Exception =
  | { kind: "captured_not_fulfilled"; orderId: string; amount: Money; ageMs: number }
  | { kind: "fulfilled_not_captured"; orderId: string; amount: Money }
  | { kind: "amount_mismatch"; orderId: string; ours: Money; theirs: Money }
  | { kind: "unknown_to_us"; pspRef: string; orderId: string; amount: Money }
  | { kind: "refunded_but_delivered"; orderId: string };

declare function reconcile(
  orders: OrderRow[], psp: PspRow[], now: number, slaMs: number,
): Exception[];
```

**The trap:**
- **Both directions.** Most candidates only loop over orders. You also have to walk the PSP
  side and find what they have and you don't (`unknown_to_us`)  the most dangerous class:
  **money taken with no order behind it.**
- **The SLA.** An order sitting in `captured` for 10 seconds is **not** an exception. Ten
  minutes is. The `ageMs > slaMs` check is the difference between a useful report and one
  that screams at everything.
- **Comparing amounts** must include currency. `1000 IQD ≠ 1000 USD`.
- **Order by severity.** A human reads these at 3am. `captured_not_fulfilled` outranks
  `amount_mismatch`. **Sort them.**
- **Time complexity.** `orders.find(...)` inside a loop over the PSP rows is O(n·m).
  Build a `Map`  they serve millions of users.

**Follow-ups:**
- Once you detect `captured_not_fulfilled`, what happens automatically? (Replay the
  fulfilment; after N failures, auto-refund. **The customer doesn't wait for the RCA.**)
- How do you run this over millions of rows without loading them all? (Stream with a cursor
  over a bounded time window.)

<details>
<summary>Solution</summary>

```ts
const SEVERITY: Record<Exception["kind"], number> = {
  captured_not_fulfilled: 0,   // paid and received nothing  the worst
  unknown_to_us: 1,            // money at the PSP with no order
  refunded_but_delivered: 2,   // direct loss
  amount_mismatch: 3,
  fulfilled_not_captured: 4,   // delivered for free
};

function sameMoney(a: Money, b: Money): boolean {
  return a.minor === b.minor && a.currency === b.currency;
}

function reconcile(
  orders: OrderRow[], psp: PspRow[], now: number, slaMs: number,
): Exception[] {
  const pspByOrder = new Map(psp.map((p) => [p.orderId, p]));
  const ourIds = new Set(orders.map((o) => o.orderId));
  const out: Exception[] = [];

  for (const o of orders) {
    const p = pspByOrder.get(o.orderId);

    if (o.state === "captured" && o.capturedAt !== undefined) {
      const ageMs = now - o.capturedAt;
      if (ageMs > slaMs) {
        out.push({ kind: "captured_not_fulfilled", orderId: o.orderId, amount: o.amount, ageMs });
      }
    }

    if ((o.state === "fulfilled" || o.state === "fulfilling") && !p) {
      out.push({ kind: "fulfilled_not_captured", orderId: o.orderId, amount: o.amount });
    }

    if (p && !sameMoney(o.amount, p.amount)) {
      out.push({ kind: "amount_mismatch", orderId: o.orderId, ours: o.amount, theirs: p.amount });
    }

    if (p?.status === "refunded" && o.state === "fulfilled") {
      out.push({ kind: "refunded_but_delivered", orderId: o.orderId });
    }
  }

  for (const p of psp) {
    if (!ourIds.has(p.orderId)) {
      out.push({ kind: "unknown_to_us", pspRef: p.pspRef, orderId: p.orderId, amount: p.amount });
    }
  }

  return out.sort((a, b) => SEVERITY[a.kind] - SEVERITY[b.kind]);
}
```
</details>

---

# L5  Abuse and retries

## L5.1  A multi-dimensional velocity limiter

An attacker holds stolen cards and tests them on small top-ups. IPs rotate cheaply, so
limiting on IP alone **fails**.

```ts
type Dimension = "user" | "card_fingerprint" | "ip" | "bin" | "device";

type Rule = { dimension: Dimension; windowMs: number; max: number };

type Attempt = {
  at: number;
  keys: Partial<Record<Dimension, string>>;
  outcome: "approved" | "declined";
};

type Decision =
  | { allow: true }
  | { allow: false; brokenRule: Rule; dimension: Dimension; key: string; retryAfterMs: number };

interface VelocityLimiter {
  check(attempt: Omit<Attempt, "outcome">): Decision;
  record(attempt: Attempt): void;
}

declare function createLimiter(rules: Rule[]): VelocityLimiter;
```

**Example rule set:**
```ts
[
  { dimension: "card_fingerprint", windowMs: 600_000, max: 3 },   // 3 attempts / 10 min
  { dimension: "ip",     windowMs: 60_000,  max: 20 },
  { dimension: "user",   windowMs: 3_600_000, max: 50 },
  { dimension: "bin",    windowMs: 300_000, max: 100 },           // BIN attack
]
```

**The trap:**
- **Sliding window, not fixed.** A fixed window permits double the limit across a boundary:
  20 attempts at 10:00:59 and 20 more at 10:01:01.
- **Declines are what you count, not all attempts.** The card-testing rule is built on
  *repeated failure*. Counting successes too punishes legitimate customers.
  (Let the rule decide  but **raise the question in the interview**.)
- **Missing keys.** If an attempt carries no `device`, device rules must be **skipped**, not
  treated as a shared `undefined` key  otherwise every device-less customer is throttled together.
- **Reclaim memory.** A `Map<string, number[]>` grows forever. Trim what left the window.
- **`retryAfterMs`** is computed from the **oldest attempt in the window**, not a constant.
- **Stop at the first broken rule.** Don't evaluate the rest.

**Follow-ups:**
- Your limiter's Redis is down  do you allow everyone or block everyone?
  (On a payment path: **fail closed**, or fall back to a conservative local limit. It's a
  business decision; document it in an ADR.)
- How does this work across instances? (Redis `INCR` + `EXPIRE` in a Lua script for
  atomicity, or a sorted set for a true sliding window.)
- Which metric alerts you to the attack? (**`auth_success_rate`**  a sudden drop is the
  shape of the attack. Not CPU.)

<details>
<summary>Solution</summary>

```ts
function createLimiter(rules: Rule[]): VelocityLimiter {
  // dimension:key → timestamps of declined attempts
  const hits = new Map<string, number[]>();
  const slot = (d: Dimension, k: string) => `${d}:${k}`;

  function prune(key: string, now: number, windowMs: number): number[] {
    const arr = hits.get(key);
    if (!arr) return [];
    const cutoff = now - windowMs;
    let i = 0;
    while (i < arr.length && arr[i] <= cutoff) i++;
    const kept = i === 0 ? arr : arr.slice(i);
    if (kept.length === 0) hits.delete(key); else hits.set(key, kept);
    return kept;
  }

  return {
    check(attempt) {
      const now = attempt.at;
      for (const rule of rules) {
        const key = attempt.keys[rule.dimension];
        if (key === undefined) continue;            // missing key → rule doesn't apply
        const recent = prune(slot(rule.dimension, key), now, rule.windowMs);
        if (recent.length >= rule.max) {
          const oldest = recent[0];
          return {
            allow: false,
            brokenRule: rule,
            dimension: rule.dimension,
            key,
            retryAfterMs: Math.max(0, oldest + rule.windowMs - now),
          };
        }
      }
      return { allow: true };
    },

    record(attempt) {
      if (attempt.outcome !== "declined") return;   // only failures count
      for (const rule of rules) {
        const key = attempt.keys[rule.dimension];
        if (key === undefined) continue;
        const k = slot(rule.dimension, key);
        const arr = hits.get(k) ?? [];
        arr.push(attempt.at);
        hits.set(k, arr);
      }
    },
  };
}
```

`prune` advances a pointer instead of calling `filter`, because the array is already
sorted  O(removed) rather than O(n).
</details>

---

## L5.2  Retries, and "a timeout is not a failure"

**This is the most important problem in the file.** Getting it wrong is how you double-charge
customers in real production.

```ts
type Outcome<T> =
  | { kind: "success"; value: T }
  | { kind: "failed"; error: Error }           // confirmed failure  it did not happen
  | { kind: "unknown"; error: Error };         // outcome unknown  it may have happened

type RetryOptions = {
  maxAttempts: number;
  baseDelayMs: number;
  maxDelayMs: number;
  jitter: (n: number) => number;               // injected, so tests are deterministic
  isIdempotent: boolean;
  /** Asks the provider what actually happened. Required when isIdempotent is false. */
  checkStatus?: () => Promise<Outcome<unknown>>;
};

declare function callWithRetry<T>(
  op: () => Promise<T>,
  classify: (err: unknown) => "retryable" | "fatal" | "unknown",
  opts: RetryOptions,
): Promise<Outcome<T>>;
```

**The rules:**

| Situation | Behavior |
|---|---|
| `retryable` (503, connection refused) | Retry with backoff + jitter |
| `fatal` (400, 422, validation) | Stop immediately, `failed` |
| `unknown` (timeout, 504, socket hangup) + **idempotent** | Safe to retry |
| `unknown` + **not idempotent** | **Never retry.** Call `checkStatus()`. None? → `unknown` |

**The trap  and this is the heart of it:**
- **A timeout is not a failure.** The request reached the PSP, the operation may have
  succeeded, and the response was lost. Retrying a non-idempotent `capture` **charges the
  customer twice.**
- Most retry wrappers treat any exception as "it failed, go again." **That is the bug.**
- **`jitter` is mandatory.** Without it every client retries in the same instant
  (`thundering herd`) and kills the service as it recovers.
- **Cap the delay.** `2 ** 10 * 1000` is 17 minutes. Set `maxDelayMs`.
- **Inject `jitter`** rather than calling `Math.random()` inside, so the test is deterministic.

**Follow-ups:**
- `checkStatus` itself times out  then what?
  (Return `unknown` and let **reconciliation** settle it. Not everything is resolvable
  inside the request.)
- How would you make `capture` idempotent in the first place? (An idempotency key sent to
  the PSP  every serious provider supports one.)

<details>
<summary>Solution</summary>

```ts
const sleep = (ms: number) => new Promise<void>((r) => setTimeout(r, ms));

async function callWithRetry<T>(
  op: () => Promise<T>,
  classify: (err: unknown) => "retryable" | "fatal" | "unknown",
  opts: RetryOptions,
): Promise<Outcome<T>> {
  let lastErr: Error = new Error("no attempt made");

  for (let attempt = 1; attempt <= opts.maxAttempts; attempt++) {
    try {
      return { kind: "success", value: await op() };
    } catch (err) {
      lastErr = err instanceof Error ? err : new Error(String(err));
      const cls = classify(err);

      if (cls === "fatal") return { kind: "failed", error: lastErr };

      if (cls === "unknown" && !opts.isIdempotent) {
        // Never retry. Ask what actually happened.
        if (!opts.checkStatus) return { kind: "unknown", error: lastErr };
        try {
          const actual = await opts.checkStatus();
          if (actual.kind === "success") return { kind: "success", value: actual.value as T };
          if (actual.kind === "failed")  return { kind: "failed", error: lastErr };
          return { kind: "unknown", error: lastErr };   // the status call is unknown too
        } catch {
          return { kind: "unknown", error: lastErr };
        }
      }

      if (attempt === opts.maxAttempts) break;

      const exp = Math.min(opts.maxDelayMs, opts.baseDelayMs * 2 ** (attempt - 1));
      await sleep(Math.floor(exp * opts.jitter(attempt)));   // full jitter
    }
  }

  // Attempts exhausted: for an idempotent op this is still unknown,
  // because the last attempt may have landed
  return opts.isIdempotent
    ? { kind: "unknown", error: lastErr }
    : { kind: "failed", error: lastErr };
}

// test jitter:       () => 1        → deterministic
// production jitter: () => Math.random()
```

**Say this verbatim:**
> "A timeout is an unknown outcome, not a failure. If the operation isn't idempotent,
> retrying it is how you double-charge a customer. The only safe move is to ask the provider
> what actually happened  and if that's also unavailable, leave it unknown and let
> reconciliation close it."
</details>

---

# What the interviewer is actually scoring

Not the solution. Most candidates reach a solution. What gets scored:

| Signal | What it looks like |
|---|---|
| **Clarifying before coding** | "Do refunds count toward gross?"  ask, don't assume |
| **Types first** | A discriminated union before any logic |
| **Naming the unknown state** | `unknown` rather than `failed`  the strongest senior signal here |
| **Determinism** | No `Math.random` or `Date.now()` buried in logic  inject them |
| **Exhaustiveness** | `const _: never = x` in the default branch |
| **Time complexity** | A `Map`, not a `find` inside a loop |
| **Testing the failure cases** | Concurrency, duplicates, boundaries  not the happy path |

## The 60-second checklist before you say "done"

1. Did you run `tsc --strict`? Any `any`, or an `as` without justification?
2. Does every union have a `never` check?
3. Is there a float anywhere in a money calculation?
4. Is this operation idempotent? What happens if it runs twice?
5. Is a timeout being treated as a failure anywhere? **Fix it.**
6. Is `Date.now()` or `Math.random()` inside the logic? **Lift it out.**
7. Did you say out loud what isn't covered and what you'd do with more time?

> That last point earns marks even when the code is incomplete. A candidate who says
> "this doesn't handle concurrency across instances, and the fix is to push atomicity into
> the database with `SELECT FOR UPDATE SKIP LOCKED`" scores higher than one who shipped
> working code and said nothing about its limits.
