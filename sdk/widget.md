# Embeddable Widget — `@toncast/widget`

A fully-featured betting UI you can embed in any web app: market list, pari detail view, bet placement with TonConnect, and white-label theming via CSS variables.

{% hint style="info" %}
**Two integration paths:** load the hosted CDN bundle (no bundler required) or install the npm package and use the React component or imperative API.
{% endhint %}

---

## Option A — CDN (plain HTML, no bundler)

Host a [TonConnect manifest](https://docs.ton.org/v3/guidelines/ton-connect/creating-manifest) at `{your-domain}/tonconnect-manifest.json`. The widget's standalone mode derives the manifest URL from `tonconnect.options.domain`.

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>My app</title>
  </head>
  <body>
    <div id="toncast-widget"></div>

    <script src="https://widget.toncast.me/v0/index.iife.js"></script>
    <script>
      const Widget = window.ToncastWidget.ToncastWidget;

      const widget = new Widget({
        tonconnect: {
          type: "standalone",
          options: { domain: "https://your-app.com" },
        },
      });

      widget.mount(document.getElementById("toncast-widget"));
    </script>
  </body>
</html>
```

CDN URLs are **major-versioned** (`/v0/`, `/v1/`, …): you get non-breaking patch updates within a major; change the path for breaking releases.

---

## Option B — npm loader (React + existing TonConnect)

Install the loader:

```bash
npm install @toncast/widget-loader @tonconnect/ui-react
```

```tsx
import { useEffect, useRef } from "react";
import { useTonConnectUI } from "@tonconnect/ui-react";
import ToncastWidgetLoader, {
  type ToncastWidgetInstance,
} from "@toncast/widget-loader";

function ToncastBettingWidget() {
  const [tonconnect] = useTonConnectUI();
  const containerRef = useRef<HTMLDivElement>(null);
  const widgetRef = useRef<ToncastWidgetInstance | null>(null);

  useEffect(() => {
    let active = true;
    ToncastWidgetLoader.load()
      .then((Widget) => {
        if (!active || !containerRef.current) return;
        widgetRef.current = new Widget({
          tonconnect: { type: "integrated", instance: tonconnect },
        });
        widgetRef.current.mount(containerRef.current);
      })
      .catch((err) => console.error("[ToncastWidget] load failed:", err));

    return () => {
      active = false;
      widgetRef.current?.unmount();
      widgetRef.current = null;
    };
  }, [tonconnect]);

  return <div ref={containerRef} style={{ width: "100%" }} />;
}
```

Wrap your app tree in `<TonConnectUIProvider manifestUrl="https://your-app.com/tonconnect-manifest.json">`.

---

## Option C — React component (`@toncast/widget/react`)

```tsx
import { Widget } from "@toncast/widget/react";

const config = {
  tonconnect: {
    type: "standalone",
    options: { domain: "https://your-app.com" },
  },
};

function App() {
  return <Widget config={config} onBet={({ pariId, amount, side }) => {
    analytics.track("bet_sent", { pariId, amount: amount.toString(), side });
  }} />;
}
```

---

## Subscribing to bet events

Both the imperative and React APIs emit the same `{ pariId, amount, side }` payload after a bet transaction is sent.

**Imperative (CDN or npm):**

```ts
const widget = new ToncastWidget(config);

widget.on("bet", ({ pariId, amount, side }) => {
  console.log("bet placed", pariId, side, amount.toString());
});

widget.mount(document.getElementById("toncast-widget"));
```

**React component:**

```tsx
<Widget config={config} onBet={({ pariId, amount, side }) => {
  analytics.track("bet_sent", { pariId, amount: amount.toString(), side });
}} />
```

### Lifecycle events

The imperative instance also exposes `mount`, `unmount`, and `error` events.

| Method | Description |
|---|---|
| `widget.mount(el)` | Mount the widget into an element. |
| `widget.unmount()` | Unmount, but keep event listeners (allows remounting). |
| `widget.dispose()` | Unmount + clear all listeners. Call when discarding the instance. |
| `widget.on(event, fn)` | Add a listener. Events: `"bet"`, `"mount"`, `"unmount"`, `"error"`. |
| `widget.off(event, fn)` | Remove a specific listener. |

---

## White-label theming

Pass `widget.cssVars` to customize colors and `widget.layout.grid` for responsive columns.

```ts
const widget = new ToncastWidget({
  tonconnect: {
    type: "standalone",
    options: { domain: "https://your-app.com" },
  },
  widget: {
    theme: "system", // "light" | "dark" | "system"
    cssVars: {
      accent:  "#7c3aed",
      success: "#10b981",
      danger:  "#ef4444",
      warn:    "#f59e0b",
      density: "compact", // "compact" | "normal" | "spacious"
      light: { bg: "#ffffff" },
      dark:  { bg: "#0b1020" },
    },
    layout: {
      grid: {
        mobile:  1,
        tablet:  2,
        desktop: 3,
      },
    },
  },
});
```

Source tokens (`accent`, `bg`, `success`, `danger`, `warn`, `density`) are automatically resolved into hover states, borders, shadows, and spacing. Explicit low-level values (e.g. `successBg`) always override derivation. Set `deriveCssVars: false` to disable all derivation.

---

## Language

The widget ships a built-in language picker. Two layers control the language:

| Source | Wins on conflict |
|---|---|
| `config.widget.language` (host-set) | Always — reapplied on every re-render. |
| In-widget picker (user-selected) | Until the host changes `config.widget.language`. |

To lock the language, always pass `config.widget.language`. To let the user choose freely, omit it.

---

## Container ID convention

`mount(container)` accepts any `Element`. The `#toncast-widget` id used in the snippets above is a convention: the widget-constructor tool scopes its exported `style.css` overrides under `#toncast-widget { … }`. Keep that id (or update the CSS scope) when reusing the exported stylesheet.

---

## Referral attribution

Pass `widget.referral` in the config to attribute bets to your wallet:

```ts
const widget = new ToncastWidget({
  tonconnect: { … },
  widget: {
    referral: { address: "UQMyWallet…", pct: 5 }, // 0..7
  },
});
```

In **standalone** mode, removing `widget.referral` from config clears it on the SDK client. In **integrated** mode (you pass your own `ToncastClient`), an absent `widget.referral` is treated as "host manages this directly" and is never cleared by the widget.
