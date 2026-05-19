# Toncast SDK

Want to add prediction markets to your app? You're in the right place.

The Toncast SDK is a set of TypeScript packages that handle everything — fetching market data, streaming live odds, building bet transactions, and even rendering a full betting UI if you don't want to build one yourself.

---

## What's available

| Package | What it does |
|---|---|
| **`@toncast/sdk`** | The core. Fetch markets, stream live updates, and build ready-to-sign bet transactions. Works everywhere — browser and Node 20+. |
| **`@toncast/sdk-react`** | React hooks built on top of the core SDK. If you're building a React app, start here instead of using the core SDK directly. |
| **`@toncast/widget`** | A plug-and-play betting UI — market list, live odds, TonConnect wallet, the works. Drop it in and you're done. |
| **`@toncast/widget-loader`** | A tiny script that pulls the widget from our CDN at runtime. Handy when you want to avoid adding the widget bundle to your own build. |

{% hint style="warning" %}
**Still in early development (0.0.1).** Lock your npm versions to exact numbers until we reach `1.0.0` — we may ship breaking changes in minor releases.
{% endhint %}

---

## How to choose

**Just want the widget with no extra work?**
Use the CDN embed or `@toncast/widget` — see the [Widget page](widget.md).

**Building a React app with custom UI?**
Use `@toncast/sdk-react` — pre-built hooks for every endpoint, live data included. See the [React SDK page](react-sdk.md).

**Need full control — custom framework, server-side, or complex betting logic?**
Use `@toncast/sdk` directly. See the [Core SDK page](core-sdk.md).

---

## How the packages relate

```
Your app
  ├── @toncast/sdk            ← always the foundation
  │     └── @toncast/tx-sdk  ← handles transaction building + STON.fi routing (auto-installed)
  │
  ├── @toncast/sdk-react      ← React wrapper around the core (optional)
  │
  └── @toncast/widget         ← full betting UI, uses the core internally (optional)
        @toncast/widget-loader ← CDN version of the widget (optional)
```

---

## Resources

* **Widget constructor** — configure the widget visually and get a ready-to-paste code snippet: [widget.toncast.me](https://widget.toncast.me/)
* **GitHub** — source code and examples: [github.com/TonCast](https://github.com/TonCast)
* **Demo app** — a working Vite + React 19 app showing the full flow: `examples/react-app/`
