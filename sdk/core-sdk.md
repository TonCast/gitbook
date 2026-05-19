# Core SDK — `@toncast/sdk`

The core SDK gives you full access to the Toncast API — market data, live streams, wallet balances, and bet transactions — without any framework requirements. It runs in the browser and in Node 20+.

## Installation

```bash
npm install @toncast/sdk
```

A few extra packages (`@toncast/tx-sdk`, `@ton/ton`, `@ston-fi/api`, `@ston-fi/sdk`) are installed automatically — they handle transaction building and STON.fi routing under the hood.

---

## Your first integration

Here's the full flow from "no wallet connected" to "bet ready to sign":

```ts
import { TonClient, ToncastClient, TON_ADDRESS } from "@toncast/sdk";

// Connect to the TON network for reading balances and routing swaps
const tonClient = new TonClient({
  endpoint: "https://toncenter.com/api/v2/jsonRPC",
});

// Create the Toncast client — no wallet needed yet for public data
const client = new ToncastClient({ tonClient });

// Browse markets
const page = await client.paris.list({ limit: 20 });
const pari = page.items[0];

// Once the user connects their wallet, tell the SDK about it
client.setUserAddress(userAddress); // from TonConnect, server config, etc.

// Fetch the market + the user's wallet options in one call
const summary = await client.betting.summary(pari.id);

// Pick a funding source — here we use TON, filtered to only viable options
const picked = summary.capacities.find(
  (c) => c.source.address === TON_ADDRESS && c.feasible
);
if (!picked) throw new Error("no viable funding source");

// Build a quote for a market bet on YES
const quoteParams = {
  pariId: pari.id,
  isYes: true,
  maxBudgetTon: 5_000_000_000n, // 5 TON, in nanoTON
  source: picked.source.address,
  pricedCoins: summary.pricedCoins,
  oddsState: summary.oddsState,
  financialRiskAcknowledged: true,
};

const quote = await client.betting.quoteMarketBet(quoteParams);

// Always run confirmQuote right before the user signs — it re-checks current prices
const confirmed = await client.betting.confirmQuote(quote, quoteParams);

// Hand the messages to your wallet — the SDK never signs anything itself
await tonConnectUI.sendTransaction({
  messages: confirmed.messages,
  validUntil: Math.floor(Date.now() / 1000) + 5 * 60,
});
```

---

## Reading market data

```ts
// Categories — useful for building filter chips in your UI
const categories = await client.categories.list();       // [{ id, title }]
const filters = await client.categories.listFilters();   // ready-to-use filter params

// Markets — three feeds to choose from
const active   = await client.paris.list({ feed: "active", limit: 20 });
const finished = await client.paris.list({ feed: "finished" });  // resolved markets
const pending  = await client.paris.list({ feed: "pending" });   // ended, awaiting oracle
const searched = await client.paris.list({ search: "ETH price" });

// A single market and its data
const pari    = await client.paris.get(pariId);
const odds    = await client.paris.getOddsState(pariId);
const history = await client.paris.getCoefficientHistory(pariId, { timeframe: "ALL" });
const winners = await client.paris.getWinners(pariId);   // empty until the market resolves

// A user's bet history
const userBets = await client.bets.listForUser({ pageSize: 20 });
for await (const b of client.bets.iterateForUser()) console.log(b);
const onPari   = await client.bets.listForPariByUser({ pariId, pageSize: 15 });

// Wallet balances — TON and every jetton the user holds
const coins = await client.coins.list();
```

### Pagination

Markets use cursor-based pagination. Pass the cursor back to get the next page:

```ts
let page = await client.paris.list({ feed: "finished", limit: 20 });
while (page.hasMore && page.nextCursor) {
  page = await client.paris.list({ feed: "finished", cursor: page.nextCursor });
}

// Or use the iterator — it handles cursors for you automatically
for await (const pari of client.paris.iterate({ feed: "finished" })) { /* … */ }
```

---

## Live market data

### Live market list — `paris.streamList`

This keeps a list of markets in sync via WebSocket. You just listen to snapshots — the SDK handles reconnects, missed messages, and falls back to polling if the socket drops.

```ts
const stream = client.paris.streamList({ feed: "active", pageSize: 20 });

// Your callback fires immediately with the current list, then again on every update
const unsub = stream.onSnapshot((paris) => setUiState(paris));

// See what's happening with the connection
stream.onStatus((s) => console.log(s)); // "loading" | "live" | "polling" | "stopped"

// Load more markets (pagination still works in live mode)
await stream.loadMore();
stream.hasMore;

// Read the current list without subscribing
stream.snapshot();

// Clean up when you're done
stream.dispose();
```

### Single market stream — `paris.subscribe(pariId)`

Opens a dedicated WebSocket for one market. You get the current state immediately (from a parallel fetch), then live updates as bets come in and odds change.

```ts
const stream = client.paris.subscribe(pariId);

stream.onPari((pari) => setPari(pari));                               // status, volumes
stream.onOddsState((odds) => setOdds(odds));                          // current order book
stream.onCoefficientHistory((points) => setHistory(points));          // odds history
stream.onBetEvent((event) => console.log("new bets", event.newBets)); // individual bets
stream.onStatus((s) => console.log(s));

// Read the full current state without subscribing
stream.snapshot(); // → { pari, oddsState, coefficientHistory }

stream.dispose();
```

---

## Placing a bet

The SDK builds a transaction ready for signing. **You send it to the user's wallet — the SDK never touches private keys.**

### Quick glossary

| Term | What it means |
|---|---|
| **Pari** | A single prediction market, e.g. "Will ETH be above $2,500 on April 25?" |
| **`yesOdds`** | A number from 2 to 98 (always even). Roughly the implied YES probability in percent. |
| **`isYes`** | Which side you're betting: `true` = YES, `false` = NO. |
| **Tickets** | The units you're buying. A YES ticket at `yesOdds = 60` costs `0.06 TON`; a NO ticket costs `0.04 TON`. Each ticket pays out `0.1 TON` if it wins. |
| **TON amounts** | Always in nanoTON as a bigint. `1 TON = 1_000_000_000n`. |

### Three ways to bet

#### Market bet — the most common

The SDK spends your budget on the best available counter-side liquidity. Great for simple "I want to bet X TON on YES" UIs.

```ts
const quote = await client.betting.quoteMarketBet({
  pariId,
  isYes: true,
  maxBudgetTon: 5_000_000_000n, // spend up to 5 TON
  source: TON_ADDRESS,
  pricedCoins: summary.pricedCoins,
  oddsState: summary.oddsState,
  financialRiskAcknowledged: true,
});
const confirmed = await client.betting.confirmQuote(quote, quoteParams);
```

#### Limit bet

Set a worst-acceptable odds, and the SDK matches what it can at that level or better. Any remainder goes in as a limit order on the order book.

```ts
const quote = await client.betting.quoteLimitBet({
  pariId, isYes: true,
  worstYesOdds: 56,   // "I'll only bet if I get at least 56% implied probability for YES"
  ticketsCount: 300,
  source: TON_ADDRESS,
  financialRiskAcknowledged: true,
});
```

#### Fixed bet

Exactly the odds you want, exactly the number of tickets you want. Current liquidity is ignored.

```ts
const quote = await client.betting.quoteFixedBet({
  pariId, isYes: true,
  yesOdds: 56,
  ticketsCount: 10,
  source: TON_ADDRESS,
  financialRiskAcknowledged: true,
});
```

### Betting with TON vs a jetton (USDT, etc.)

You can fund a bet with TON or with any jetton in the user's wallet. The difference matters:

| | TON | Jetton (e.g. USDT) |
|---|---|---|
| How it works | Sent directly to the market contract | Swapped through STON.fi first, then forwarded |
| Quote speed | Instant, pure CPU | Requires a STON.fi simulation (~async) |
| `confirmQuote` | Still required before signing | **Always required** — re-simulates the swap to catch price drift |

To use a jetton, find one from `summary.capacities` marked `feasible: true` and pass its address as `source`:

```ts
const usdt = summary.capacities.find((c) => c.source.symbol === "USDT" && c.feasible);
if (!usdt) throw new Error("user doesn't have enough USDT to bet");

const quote = await client.betting.quoteMarketBet({
  ...quoteParams,
  source: usdt.source.address,
});
const confirmed = await client.betting.confirmQuote(quote, quoteParams);
// confirmed.messages → pass to TonConnect
// confirmed.txs      → use directly with a raw signer
```

### Roles in a bet

Every bet has three roles. By default they all resolve to the connected wallet — but you can override any of them:

| Role | Field | Default |
|---|---|---|
| Who pays & signs | `senderAddress` | `client.userAddress` |
| Who receives the payout | `beneficiary` | same as `senderAddress` |
| Who earns a referral cut | `referral` + `referralPct` (0–7) | `client.referral` option |

---

## Configuration

All options are optional. A bare `new ToncastClient()` works for public read methods.

```ts
const client = new ToncastClient({
  language: "en",        // en | ru | hi | es | zh | fr | de | pt | fa | ar
  userAddress,           // set once the wallet connects
  tonClient,             // required for balance reads and jetton betting
  referral: { address: "UQMyWallet…", pct: 5 }, // your referral wallet, 0–7%
  requestTimeoutMs: 15_000,
  maxAttempts: 3,
  retryDelayMs: 1000,
  prefetch: { categories: true, coins: false, swapMarkets: false },
  logger: console,
  onBackgroundError(error, task) {
    console.warn("background task failed", task, error);
  },
});
```

### Changing the user address at runtime

```ts
// Three levels — the most specific one wins:
new ToncastClient({ userAddress });             // set at startup
client.setUserAddress(addr);                   // update when wallet connects
client.clearUserAddress();                     // clear when wallet disconnects
client.bets.listForUser({ userAddress: other }); // override for one specific call
```

### Language

The SDK sends the language as `Accept-Language` on every request and uses it to localize market names in live events.

```ts
new ToncastClient({ language: "ru-RU" }); // → "ru"
client.setLanguage("zh-Hans-CN");         // → "zh"
```

---

## Handling errors

All SDK errors extend `ToncastError`, so you can catch them broadly or by specific type:

| Error class | When it happens |
|---|---|
| `ToncastApiError` | The API returned a non-2xx response. Has `status`, `endpoint`, and optionally `requestId`. |
| `ToncastRateLimitError` | HTTP 429. Has `retryAfterMs` so you can show a countdown. |
| `ToncastWsError` | Something went wrong with the WebSocket connection. |
| `ToncastValidationError` | The API returned data that doesn't match the expected shape — likely a backend change. |

```ts
import { ToncastError, ToncastRateLimitError } from "@toncast/sdk";

try {
  await client.paris.get(pariId);
} catch (err) {
  if (err instanceof ToncastRateLimitError) {
    showRetryCountdown(err.retryAfterMs);
  } else if (err instanceof ToncastError) {
    showErrorMessage(err.message);
  } else {
    throw err; // re-throw unexpected errors
  }
}
```

For bet-specific errors, `classifyBetFlowError(err)` helps you tell apart user cancellations (`wallet_user_rejected`), wallet failures (`wallet_failed`), network issues (`network`), and SDK-level errors (`toncast`).

---

## Cleaning up

Always clean up streams when you're done with them — otherwise WebSockets stay open and polling keeps running.

```ts
stream.dispose();         // stop one stream
client.paris.dispose();   // stop all market streams at once
client.dispose();         // stop everything the client owns
```

Call `client.dispose()` on route changes, component unmounts, or server shutdown.

---

## Before you go live

* Lock to exact package versions until `1.0.0`.
* Never hardcode wallet addresses — always read them from the connected wallet.
* Always call `confirmQuote` immediately before the user signs, every time. Don't cache or reuse the result.
* For jetton bets, `confirmQuote` is especially critical — swap rates change by the second.
* Test on mainnet with tiny amounts before turning on real traffic.
* Always surface SDK errors to the user. Silent empty states are confusing and hide real problems.
