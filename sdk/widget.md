# Embeddable Widget — `@toncast/widget`

The widget is a self-contained betting UI you can drop into any web page or React app. It includes a live market list, individual market pages, bet placement with TonConnect, and support for custom branding through CSS variables — no extra development required.

{% hint style="info" %}
**Start here:** [widget.toncast.me](https://widget.toncast.me/) — a visual configurator where you can preview the widget live, tweak colors, theme, and layout, then export in one click:

- **Download ZIP** — a ready-to-host package: `index.html` (standalone page), `index.iife.css` (widget styles), and `tonconnect-manifest.json` with your domain and app name pre-filled.
- **Copy JS snippet** — a CDN `<script>` embed for plain HTML pages.
- **Copy React snippet** — a ready-to-paste `ToncastBettingWidget` component using `@toncast/widget-loader`.
{% endhint %}

---

## Three ways to add the widget

### Option A — One script tag (CDN, no build tool needed)

The simplest option. Add a `<script>` tag and a container `<div>`, and the widget is running. No npm, no bundler.

Before you start: host a [TonConnect manifest](https://docs.ton.org/v3/guidelines/ton-connect/creating-manifest) at `https://your-app.com/tonconnect-manifest.json`. The widget needs it for wallet connections.

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

The CDN URL is versioned (`/v0/`, `/v1/`, …). You'll get bug fixes automatically within the same major version. When we release a breaking change, we bump the major — so you opt in on your own schedule.

---

### Option B — React app with your own TonConnect

If you already have TonConnect set up in a React app, use `@toncast/widget-loader`. It loads the widget from the CDN at runtime and wires it to your existing wallet connection.

```bash
npm install @toncast/widget-loader
```

```tsx
import { useEffect, useRef } from "react";
import { useTonConnectUI } from "@tonconnect/ui-react";
import ToncastWidgetLoader, { type ToncastWidgetInstance } from "@toncast/widget-loader";

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
      widgetRef.current?.dispose(); // dispose() unmounts + clears all listeners
      widgetRef.current = null;
    };
  }, [tonconnect]);

  return <div ref={containerRef} style={{ width: "100%" }} />;
}
```

Make sure your app tree is wrapped in `<TonConnectUIProvider manifestUrl="https://your-app.com/tonconnect-manifest.json">`.

---

### Option C — React component (npm package)

Install the widget as an npm package and render it as a React component. Ideal if you want to keep things in your bundle and use the `onBet` callback directly as a prop.

```bash
npm install @toncast/widget
```

```tsx
import { Widget } from "@toncast/widget/react";

const config = {
  tonconnect: {
    type: "standalone",
    options: { domain: "https://your-app.com" },
  },
};

function App() {
  return (
    <Widget
      config={config}
      className="my-widget"          // optional: extra CSS class on the root element
      style={{ borderRadius: "12px" }} // optional: inline styles on the root element
      onBet={({ pariId, amount, side }) => {
        console.log("Bet placed:", side, amount.toString(), "on", pariId);
      }}
      onRenderError={(error, info) => {
        // optional: called from the ErrorBoundary for analytics/logging
        console.error("Widget render error", error, info);
      }}
    />
  );
}
```

---

## Listening to bet events

Whenever a user successfully places a bet, both the CDN and React versions fire the same event with the same payload: `{ pariId, amount, side }`.

**Imperative API (CDN / npm class):**

```ts
const widget = new ToncastWidget(config);

widget.on("bet", ({ pariId, amount, side }) => {
  analytics.track("bet_placed", { pariId, side, amount: amount.toString() });
});

widget.mount(document.getElementById("toncast-widget"));
```

**React component:**

```tsx
<Widget
  config={config}
  onBet={({ pariId, amount, side }) => {
    analytics.track("bet_placed", { pariId, side, amount: amount.toString() });
  }}
/>
```

### Widget lifecycle methods

The imperative class exposes a small set of methods for managing the widget's lifecycle:

| Method | What it does |
|---|---|
| `widget.mount(element)` | Renders the widget inside the given element. |
| `widget.unmount()` | Removes the widget from the DOM but keeps event listeners, so you can remount later. |
| `widget.dispose()` | Removes the widget and clears all event listeners. Call this when you're fully done with the instance. |
| `widget.update(config)` | Re-renders with a new config **without unmounting**. Visual changes (theme, colors) apply instantly. Changes to API URLs, language, or referral recreate the SDK client internally. |
| `widget.on(event, fn)` | Listen to an event. Available events: `"bet"`, `"mount"`, `"unmount"`, `"error"`. |
| `widget.off(event, fn)` | Remove a specific listener. |

---

## Custom branding

The widget reads CSS variables that you control. Pass them in `widget.cssVars` to match your brand colors, and use `widget.layout.grid` to control how many market cards appear per row.

```ts
const widget = new ToncastWidget({
  tonconnect: {
    type: "standalone",
    options: { domain: "https://your-app.com" },
  },
  widget: {
    theme: "system", // "light" | "dark" | "system" — follows the user's OS preference
    cssVars: {
      accent:  "#7c3aed", // buttons, highlights
      success: "#10b981", // YES side color
      danger:  "#ef4444", // NO side color
      warn:    "#f59e0b", // warning states
      density: "compact", // "compact" | "default" | "comfortable" — controls spacing
      light: { bg: "#ffffff" },
      dark:  { bg: "#0b1020" },
    },
    layout: {
      grid: {
        mobile:  1, // cards per row on mobile
        tablet:  2,
        desktop: 3,
      },
    },
  },
});
```

The widget automatically derives hover states, borders, shadows, and spacing from your base tokens. If you want to override a specific derived value (like `successBg`), just pass it explicitly and the widget will use that exact value instead. To disable all automatic derivation, set `deriveCssVars: false`.

> **Tip:** Use the [visual configurator](https://widget.toncast.me/) to preview color changes live and export a ZIP with the final config baked in.

---

## Language

The widget has a built-in language picker. Two things control what happens with it:

| Setting | Behavior |
|---|---|
| `config.widget.language` (you set this) | Always applied on every render — overrides whatever the user picked. |
| In-widget language picker | Works freely unless `config.widget.language` is set. |

To lock the language, always pass `config.widget.language`. To let users choose freely, leave it out.

You can also control **which languages appear** in the picker via `config.widget.languages`:

```ts
// Show only English and Russian in the picker
widget: { languages: ["en", "ru"] }

// Hide the picker entirely
widget: { languages: [] }

// Show all supported languages (default — omit the option)
```

---

## Referral attribution

If you want bets placed through your integration to earn you a referral share, pass your wallet address and percentage:

```ts
const widget = new ToncastWidget({
  tonconnect: { … },
  widget: {
    referral: { address: "UQYourWallet…", pct: 5 }, // 1–7%
  },
});
```

In standalone mode, removing `referral` from the config clears it entirely. In integrated mode (where you pass your own `ToncastClient`), leaving it out means the widget won't touch whatever referral you've set on the client directly.

---

## A note on the container ID

`widget.mount(element)` accepts any DOM element — the `id="toncast-widget"` you see in examples is just a convention. The [widget configurator](https://widget.toncast.me/) scopes its exported `style.css` overrides to `#toncast-widget { … }`, so if you use a different ID, make sure to update the exported CSS accordingly.
