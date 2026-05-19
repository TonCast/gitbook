# React SDK — `@toncast/sdk-react`

React hooks for `@toncast/sdk`, built on top of [TanStack Query](https://tanstack.com/query/latest). REST endpoints become `useQuery` hooks; WebSocket streams use `useSyncExternalStore` so live updates never miss an intermediate value.

## Install

```bash
npm install @toncast/sdk @toncast/sdk-react @tanstack/react-query
# If you use TonConnect for wallet auth:
npm install @tonconnect/ui-react
```

Peer dependencies: React `^18 || ^19`, `@tanstack/react-query ^5`.

---

## Quick start

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
      <ParisFeed />
    </ToncastProvider>
  );
}

function ParisFeed() {
  const { data, isLoading } = useStreamList({ feed: "active" });

  if (isLoading) return <p>Loading…</p>;
  return data?.map((p) => <div key={p.id}>{p.name}</div>);
}
```

`<ToncastProvider>` creates an internal `QueryClient` automatically. To share an existing TanStack Query instance, pass `queryClient={appQueryClient}`.

---

## Provider

| Export | Description |
|---|---|
| `<ToncastProvider client queryClient?>` | Wires both clients into context. |
| `useToncastClient()` | Read the SDK client. Throws outside a provider. |

---

## Hooks reference

### High-level

| Hook | Description |
|---|---|
| `useBet(params)` | All-in-one hook for the full bet flow — summary, coin picker, quote, confirm. Recommended for most UIs. |

### Read (REST → `useQuery`)

| Hook | Wraps | Notes |
|---|---|---|
| `useParis(params)` | `paris.list` | Single page, cursor-paginated. |
| `usePari(id)` | `paris.get` | Disabled when `id` is falsy. |
| `useBets(params)` | `bets.listForUser` / `listForPariByUser` | `pariId` optional → cross-pari history. |
| `useCategories()` | `categories.list` | Raw `{ id, title }`, `staleTime: Infinity`. |
| `useCategoryFilters()` | `categories.listFilters` | UI-ready chips for a category picker. |
| `useCoins(opts)` | `coins.list` | TON + jettons. Requires `tonClient`. |
| `useBetQuote(params \| null)` | `betting.quote*Bet` | Auto re-quotes when params change. |

Pass any TanStack `UseQueryOptions` (`enabled`, `staleTime`, `select`, `refetchInterval`, …) as a second argument.

### Live (`useSyncExternalStore`)

| Hook | Description |
|---|---|
| `useStreamList(params)` | Wraps `paris.streamList`. `data` is the latest `Pari[]` snapshot; re-emits on every WS broadcast. |
| `useSubscribe(pariId)` | Wraps `paris.subscribe`. `data` is `{ pari, oddsState, coefficientHistory }`. |
| `useBetSummary(pariId)` | Wraps `betting.subscribeSummary`. Two phases: TON-only (~200 ms), then full jetton pricing (3–8 s on cold start). |

All live hooks return `{ data, status, error, isLoading, isError, isSuccess, refetch }`.

### Mutations

| Hook | Description |
|---|---|
| `useConfirmBet()` | TanStack `useMutation` around `betting.confirmQuote`. Pass `financialRiskAcknowledged: true`. |

### TonConnect bridge

```tsx
import { useTonAddress } from "@tonconnect/ui-react";
import { useTonConnectClient } from "@toncast/sdk-react";

function WalletSync() {
  // Mirrors the connected wallet address into client.userAddress.
  // Clears it when the wallet disconnects (empty string).
  useTonConnectClient(useTonAddress());
  return null;
}
```

---

## End-to-end bet flow

### Simple — `useBet`

```tsx
import { useBet } from "@toncast/sdk-react";
import { useTonAddress, useTonConnectUI } from "@tonconnect/ui-react";

function BetCard({ pariId }: { pariId: string }) {
  const userAddress = useTonAddress();
  const [tc] = useTonConnectUI();
  const bet = useBet({ pariId: userAddress ? pariId : null, defaultSide: "yes" });

  return (
    <button
      disabled={!bet.quote.isFeasible || bet.confirm.isPending}
      onClick={async () => {
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

### Advanced — composing lower-level hooks

For fine-grained control, compose `useBetSummary` → `useBetQuote` → `useConfirmBet` directly. See `examples/react-app/src/BetCard.tsx` for a full reference implementation.

---

## Prefetching with `toncastQueryKeys`

Use `toncastQueryKeys` to ensure prefetch keys match the built-in hooks exactly (including `bigint` serialization):

```tsx
import { toncastQueryKeys } from "@toncast/sdk-react";

void queryClient.prefetchQuery({
  queryKey: toncastQueryKeys.paris.detail(pariId),
  queryFn: ({ signal }) => client.paris.get(pariId, signal),
});
```
