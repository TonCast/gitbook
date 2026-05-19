# React SDK — `@toncast/sdk-react`

If you're building a React app, this package is your best starting point. It wraps the core SDK in React-friendly hooks powered by [TanStack Query](https://tanstack.com/query/latest) — REST calls become `useQuery` hooks, and live WebSocket streams update your components automatically without any extra wiring.

## Installation

```bash
npm install @toncast/sdk @toncast/sdk-react @tanstack/react-query

# If you're using TonConnect for wallet connections:
npm install @tonconnect/ui-react
```

Requires React 18 or 19 and `@tanstack/react-query` v5.

---

## Getting started

Wrap your app in `<ToncastProvider>` and you're ready to use any hook:

```tsx
import { TonClient, ToncastClient } from "@toncast/sdk";
import { ToncastProvider, useStreamList } from "@toncast/sdk-react";

const tonClient = new TonClient({
  endpoint: "https://toncenter.com/api/v2/jsonRPC",
  apiKey: import.meta.env.VITE_TONCENTER_API_KEY,
});

const client = new ToncastClient({ tonClient });

function App() {
  return (
    <ToncastProvider client={client}>
      <MarketList />
    </ToncastProvider>
  );
}

function MarketList() {
  const { data, isLoading } = useStreamList({ feed: "active" });

  if (isLoading) return <p>Loading markets…</p>;
  return data?.map((p) => <div key={p.id}>{p.name}</div>);
}
```

`<ToncastProvider>` creates its own internal `QueryClient`. If your app already uses TanStack Query, pass yours in: `<ToncastProvider client={client} queryClient={appQueryClient}>`.

---

## Provider

| Export | What it does |
|---|---|
| `<ToncastProvider client queryClient?>` | Sets up the SDK and TanStack Query contexts. |
| `useToncastClient()` | Access the SDK client from any component inside the provider. |

---

## Hooks

### All-in-one bet hook

| Hook | What it does |
|---|---|
| `useBet(params)` | The easiest way to build a betting UI. Manages the full flow — fetching market data, pricing coins, building the quote, and confirming. See the example below. |

### Reading data

These hooks fetch data from the REST API via TanStack Query. They cache results and re-fetch automatically.

| Hook | What it fetches |
|---|---|
| `useParis(params)` | A single page of markets (cursor-paginated). |
| `usePari(id)` | One market by ID. Skips the request if `id` is falsy. |
| `useBets(params)` | A user's bet history. Leave `pariId` empty for history across all markets. |
| `useInfiniteBets(params)` | Same as `useBets` but with infinite scroll support. |
| `useCategories()` | Raw category list `[{ id, title }]`. Cached indefinitely. |
| `useCategoryFilters()` | Category filter chips ready to plug into `useStreamList`. |
| `useCoins(opts)` | The user's TON and jetton balances. Needs `tonClient` on the client. |
| `useBetQuote(params \| null)` | A bet quote — re-fetches automatically whenever params change. Params require a `mode` field: `{ mode: "market", ... }`, `{ mode: "fixed", ... }`, or `{ mode: "limit", ... }`. |
| `useMarketCapacity(source, isYes, opts?)` | How many matched tickets are available on the given side, broken down per odds level. Useful for building a ticket-count slider. |

All of these accept standard TanStack Query options (`enabled`, `staleTime`, `select`, `refetchInterval`, …) as a second argument.

### Live data

These hooks connect to live WebSocket streams and update your component on every change.

| Hook | What you get |
|---|---|
| `useStreamList(params)` | Live list of markets. `data` is the latest snapshot, updated on every WebSocket broadcast. |
| `useSubscribe(pariId, params?)` | Live view of one market. `data` is `{ pari, oddsState, coefficientHistory }`. Disabled when `pariId` is falsy. |
| `useBetSummary(pariId)` | Streaming bet summary with two phases: TON prices arrive in ~200 ms, jetton prices follow in 3–8 s. |

All live hooks return: `{ data, status, error, streamStatus, isLoading, isError, isSuccess, refetch }`. The `streamStatus` field mirrors the underlying stream's connection state (`"loading" | "live" | "polling" | "stopped"`).

### Actions

| Hook | What it does |
|---|---|
| `useConfirmBet()` | A TanStack mutation that wraps `betting.confirmQuote`. Call `mutateAsync` right before the user signs. |

### Language

| Hook | What it does |
|---|---|
| `useToncastLanguage()` | Returns `{ lang, setLang }`. Use this to build a language picker or keep your app's locale in sync with the SDK. Invalidates all cached queries automatically when the language changes. |

### TonConnect wallet sync

```tsx
import { useTonAddress } from "@tonconnect/ui-react";
import { useTonConnectClient } from "@toncast/sdk-react";

function WalletSync() {
  // Keeps client.userAddress in sync with the connected wallet.
  // Automatically clears it when the user disconnects.
  useTonConnectClient(useTonAddress());
  return null;
}
```

---

## Placing a bet with `useBet`

`useBet` is the recommended way to build a betting card. It composes all the lower-level hooks and manages state for you:

```tsx
import { useBet } from "@toncast/sdk-react";
import { useTonAddress, useTonConnectUI } from "@tonconnect/ui-react";

function BetCard({ pariId }: { pariId: string }) {
  const userAddress = useTonAddress();
  const [tc] = useTonConnectUI();

  // Pass null when the wallet isn't connected yet — the hook safely does nothing
  const bet = useBet({ pariId: userAddress ? pariId : null, defaultSide: "yes" });

  return (
    <button
      disabled={!bet.quote.isFeasible || bet.confirm.isPending}
      onClick={async () => {
        // confirmCurrent re-checks prices and builds the transaction
        const confirmed = await bet.confirmCurrent({ financialRiskAcknowledged: true });
        if (!confirmed) return;

        await tc.sendTransaction({
          messages: confirmed.messages,
          validUntil: Math.floor(Date.now() / 1000) + 5 * 60,
        });
      }}
    >
      Bet {bet.side.toUpperCase()}
    </button>
  );
}
```

Need more control? You can compose the lower-level hooks yourself: `useBetSummary` → `useBetQuote` → `useConfirmBet`. Check `examples/react-app/src/BetCard.tsx` for a complete example.

---

## Prefetching

Use `toncastQueryKeys` when prefetching to make sure keys match what the hooks use internally (including `bigint` serialization):

```tsx
import { toncastQueryKeys } from "@toncast/sdk-react";

// In a loader, route guard, or server component:
void queryClient.prefetchQuery({
  queryKey: toncastQueryKeys.paris.detail(pariId),
  queryFn: ({ signal }) => client.paris.get(pariId, signal),
});
```
