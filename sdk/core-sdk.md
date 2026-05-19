# Core SDK — `@toncast/sdk`

## Install

```bash
npm install @toncast/sdk
```

`@toncast/tx-sdk`, `@ton/ton`, `@ston-fi/api`, and `@ston-fi/sdk` are pulled in automatically as hard dependencies — the full betting flow requires all of them.

---

## Quick start

```ts
import { TonClient, ToncastClient, TON_ADDRESS } from "@toncast/sdk";

// RPC client for on-chain reads (jetton balances, STON.fi routing)
const tonClient = new TonClient({
  endpoint: "https://toncenter.com/api/v2/jsonRPC",
});

// --- Phase 1: public reads, no wallet needed ---
const client = new ToncastClient({ tonClient });

const page = await client.paris.list({ limit: 20 });
const pari = page.items[0];

// --- Phase 2: wallet is connected ---
client.setUserAddress(userAddress); // from TonConnect or server config

const summary = await client.betting.summary(pari.id);
// summary.capacities → all user coins annotated with viability + min/maxBetTon

const picked = summary.capacities.find(
  (c) => c.source.address === TON_ADDRESS && c.feasible
);
if (!picked) throw new Error("no viable funding source");

const quoteParams = {
  pariId: pari.id,
  isYes: true,
  maxBudgetTon: 5_000_000_000n, // 5 TON in nano
  source: picked.source.address,
  pricedCoins: summary.pricedCoins,
  oddsState: summary.oddsState,
  financialRiskAcknowledged: true,
};

const quote = await client.betting.quoteMarketBet(quoteParams);

// Required before every sign — re-simulates the STON.fi route
const confirmed = await client.betting.confirmQuote(quote, quoteParams);

// Hand off to your wallet bridge (SDK never signs or sends)
await tonConnectUI.sendTransaction({
  messages: confirmed.messages,
  validUntil: Math.floor(Date.now() / 1000) + 5 * 60,
});
```

---

## Reading data

```ts
// Categories
const categories = await client.categories.list();       // [{ id, title }]
const filters = await client.categories.listFilters();   // UI-ready chips

// Paris — three feeds
const active   = await client.paris.list({ feed: "active", limit: 20 });
const finished = await client.paris.list({ feed: "finished" });
const pending  = await client.paris.list({ feed: "pending" });
const results  = await client.paris.list({ search: "ETH price" });

// Single pari
const pari    = await client.paris.get(pariId);
const odds    = await client.paris.getOddsState(pariId);
const history = await client.paris.getCoefficientHistory(pariId, { timeframe: "ALL" });
const winners = await client.paris.getWinners(pariId);  // empty until resolved

// User bets
const userBets = await client.bets.listForUser({ pageSize: 20 });
for await (const b of client.bets.iterateForUser()) console.log(b);
const onPari   = await client.bets.listForPariByUser({ pariId, pageSize: 15 });

// Wallet balances
const coins = await client.coins.list(); // TON + all jettons the wallet holds
```

### Pagination

```ts
let page = await client.paris.list({ feed: "finished", limit: 20 });
while (page.hasMore && page.nextCursor) {
  page = await client.paris.list({ feed: "finished", cursor: page.nextCursor });
}

// Or use the async iterator — handles cursors for you
for await (const pari of client.paris.iterate({ feed: "finished" })) { /* … */ }
```

---

## Live data

### `paris.streamList` — live market list

Self-managing list that stays in sync via WebSocket broadcast (with automatic polling fallback, reconnect, and gap recovery).

```ts
const stream = client.paris.streamList({ feed: "active", pageSize: 20 });

// Fires immediately with current state, then on every change
const unsub = stream.onSnapshot((paris) => setUiState(paris));

stream.onStatus((s) => console.log(s)); // "loading" | "live" | "polling" | "stopped"

await stream.loadMore(); // pagination
stream.hasMore;
stream.snapshot();       // synchronous read without a listener

stream.dispose();        // close WS + stop polling
```

### `paris.subscribe(pariId)` — single pari view

Initial parallel fetch of pari + odds + coefficient history, then per-pari WebSocket for incremental updates.

```ts
const stream = client.paris.subscribe(pariId);

stream.onPari((pari) => setPari(pari));
stream.onOddsState((odds) => setOdds(odds));
stream.onCoefficientHistory((points) => setHistory(points));
stream.onBetEvent((event) => console.log("new bets", event.newBets));
stream.onStatus((s) => console.log(s));

stream.snapshot(); // { pari, oddsState, coefficientHistory }
stream.dispose();
```

---

## Betting

The SDK builds a ready-to-sign transaction. **You** sign and send it. The SDK never holds private keys.

### Key concepts

| Term | Meaning |
|---|---|
| **Pari** | A single prediction market (e.g. "Will ETH be above $2,500?"). |
| **`yesOdds`** | Integer 2–98 (even). Implied YES probability in percent. |
| **`isYes`** | Which side you're betting: `true` = YES, `false` = NO. |
| **Tickets** | Units purchased. Each YES ticket costs `yesOdds × 0.001 TON`; each NO ticket costs `(100 − yesOdds) × 0.001 TON`. |
| **TON amounts** | Always bigints in **nanoTON** (`1 TON = 1_000_000_000n`). |

### Three bet modes

#### Market (most common)

Spends greedily on the best counter-side liquidity up to your budget.

```ts
const quote = await client.betting.quoteMarketBet({
  pariId,
  isYes: true,
  maxBudgetTon: 5_000_000_000n,
  source: TON_ADDRESS,
  pricedCoins: summary.pricedCoins,
  oddsState: summary.oddsState,
  financialRiskAcknowledged: true,
});
const confirmed = await client.betting.confirmQuote(quote, quoteParams);
```

#### Limit

Matches available liquidity up to `worstYesOdds`, parks the remainder as a new limit order.

```ts
const quote = await client.betting.quoteLimitBet({
  pariId, isYes: true,
  worstYesOdds: 56,
  ticketsCount: 300,
  source: TON_ADDRESS,
  financialRiskAcknowledged: true,
});
```

#### Fixed

One specific odds level, one ticket count.

```ts
const quote = await client.betting.quoteFixedBet({
  pariId, isYes: true,
  yesOdds: 56,
  ticketsCount: 10,
  source: TON_ADDRESS,
  financialRiskAcknowledged: true,
});
```

### TON vs Jetton

| Path | How it works |
|---|---|
| **TON (direct)** | Synchronous, CPU-only for the quote. No STON.fi swap. `confirmQuote` is still required before signing. |
| **Jetton** | Async — STON.fi swap simulation. `confirmQuote` re-simulates right before signing to catch slippage drift. **Never skip `confirmQuote` for jettons.** |

To bet with a jetton, pass the jetton master address as `source`. Use `summary.capacities` to filter coins by `feasible: true` first.

```ts
const usdt = summary.capacities.find((c) => c.source.symbol === "USDT" && c.feasible);
if (!usdt) throw new Error("no viable USDT balance");

const quote = await client.betting.quoteMarketBet({
  ...quoteParams,
  source: usdt.source.address,
  maxBudgetTon: 5_000_000_000n,
});
const confirmed = await client.betting.confirmQuote(quote, quoteParams);
// confirmed.messages → TonConnect
// confirmed.txs      → raw TxParams[]
```

### Three roles per bet

| Role | Field | Default |
|---|---|---|
| **Signer** (funds & signs) | `senderAddress` | `client.userAddress` |
| **Beneficiary** (receives payout) | `beneficiary` | `senderAddress` |
| **Referral** (earns a share) | `referral` + `referralPct` | `client.referral` option |

Pass any of them explicitly in `quoteParams` to override the defaults.

---

## Configuration

```ts
const client = new ToncastClient({
  baseUrl: "https://toncast.me/api",
  wsUrl: "wss://toncast.me",
  language: "en",           // en | ru | hi | es | zh | fr | de | pt | fa | ar
  userAddress,
  tonClient,
  referral: { address: "UQMyWallet…", pct: 5 }, // 0..7
  requestTimeoutMs: 15_000,
  maxAttempts: 3,
  retryDelayMs: 1000,
  prefetch: { categories: true, coins: false, swapMarkets: false },
  logger: console,
  onBackgroundError(error, task) {
    console.warn("Toncast background task failed", task, error);
  },
});
```

All fields are optional. `new ToncastClient()` works for public read-only methods. Personal methods (`coins`, user bets, betting) require `userAddress` and `tonClient`.

### User address

```ts
// Three levels — last one wins:
new ToncastClient({ userAddress });      // constructor
client.setUserAddress(addr);            // runtime swap
client.clearUserAddress();
client.bets.listForUser({ userAddress: other }); // per-call override
```

### Language

```ts
const client = new ToncastClient({ language: "ru-RU" }); // → "ru"
client.setLanguage("zh-Hans-CN");                         // → "zh"
```

---

## Error handling

All SDK errors extend `ToncastError`:

| Class | When |
|---|---|
| `ToncastApiError` | REST non-2xx response. Has `status`, `endpoint`, optional `requestId`. |
| `ToncastRateLimitError` | HTTP 429. Has `retryAfterMs`. |
| `ToncastWsError` | WebSocket transport or protocol failure. |
| `ToncastValidationError` | Backend response failed the SDK's Zod contract. |

```ts
import { ToncastError, ToncastRateLimitError } from "@toncast/sdk";

try {
  await client.paris.get(pariId);
} catch (err) {
  if (err instanceof ToncastRateLimitError) {
    showRetryAfter(err.retryAfterMs);
  } else if (err instanceof ToncastError) {
    showSdkError(err.message);
  } else {
    throw err;
  }
}
```

For bet errors specifically, use `classifyBetFlowError(err)` to bucket failures into `toncast | wallet_user_rejected | wallet_failed | network | unknown`.

---

## Cleanup

Always call `dispose()` when a component unmounts, a route changes, or the server shuts down:

```ts
stream.dispose();         // one stream
client.paris.dispose();   // all pari/list streams
client.dispose();         // all SDK-owned resources
```

---

## Production checklist

* Pin exact package versions until `1.0.0`.
* Never hardcode `userAddress`, `senderAddress`, or `beneficiary` — always read from the connected wallet.
* Always run `confirmQuote` immediately before the user signs — never reuse an old confirmed result.
* For jetton bets, never skip `confirmQuote` (slippage drift).
* Smoke-test on mainnet with minimal amounts before going live.
* Surface SDK errors to the UI — do not turn them into silent empty states.
