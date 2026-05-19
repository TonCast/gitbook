# SDK Overview

The Toncast SDK is a TypeScript monorepo that gives you everything needed to integrate Toncast prediction markets into your application — from raw REST/WebSocket data fetching to a ready-to-embed betting UI.

## Packages

| Package | Install | Description |
|---|---|---|
| **`@toncast/sdk`** | `npm install @toncast/sdk` | Framework-agnostic core: REST, WebSocket streams, and bet-transaction building. Works in both browser and Node 20+. |
| **`@toncast/sdk-react`** | `npm install @toncast/sdk @toncast/sdk-react @tanstack/react-query` | Thin React 18/19 wrapper on TanStack Query. Provider + hooks for every SDK endpoint. |
| **`@toncast/widget`** | `npm install @toncast/widget` | Embeddable betting UI: market list, pari detail, bet placement with TonConnect. CSS-variable theming. |
| **`@toncast/widget-loader`** | `npm install @toncast/widget-loader` | Lightweight CDN loader — injects the hosted widget bundle at runtime. Useful for React apps with dynamic import. |

{% hint style="warning" %}
**Pre-release (0.0.1).** Pin exact npm versions until `1.0.0` — minor bumps may include breaking changes.
{% endhint %}

## How they fit together

```
Your app
  ├── @toncast/sdk          ← data + betting logic (always needed)
  │     └── uses @toncast/tx-sdk internally (tx building, STON.fi)
  ├── @toncast/sdk-react    ← React hooks (optional, wraps sdk)
  └── @toncast/widget       ← drop-in UI (optional, uses sdk internally)
        or @toncast/widget-loader  ← CDN variant of the widget
```

## Source code & examples

* GitHub: [https://github.com/TonCast](https://github.com/TonCast)
* Demo app (Vite + React 19 + TailwindCSS): `examples/react-app/`
* Widget configurator: `examples/widget-constructor/`
