# Iframe Component Embedding Protocol

A standard for embedding self-contained HTML components into a host page via iframes, with bi-directional communication via `postMessage`.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Message Protocol](#message-protocol)
  - [Envelope Format](#envelope-format)
  - [Messages: Component → Host](#messages-component--host)
  - [Messages: Host → Component](#messages-host--component)
- [Lifecycle](#lifecycle)
- [Building a New Component](#building-a-new-component)
  - [Required CSS](#required-css)
  - [Required JavaScript](#required-javascript)
  - [Minimal Boilerplate](#minimal-boilerplate)
- [Building the Host Page](#building-the-host-page)
  - [Embedding the Iframe](#embedding-the-iframe)
  - [Handling Messages](#handling-messages)
  - [Minimal Host Boilerplate](#minimal-host-boilerplate)
- [URL Parameter Forwarding](#url-parameter-forwarding)
- [Height Management](#height-management)
- [Security Considerations](#security-considerations)
- [Reference: Content Grid Component](#reference-content-grid-component)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│  HOST PAGE (e.g. www.point.me)                      │
│                                                     │
│  ┌───────────────────────────────────────────────┐  │
│  │  <iframe src="component.html?utm_source=..."> │  │
│  │                                               │  │
│  │  ┌───────────────────────────────────────┐    │  │
│  │  │  COMPONENT PAGE                       │    │  │
│  │  │  - Self-contained HTML/CSS/JS         │    │  │
│  │  │  - Transparent background             │    │  │
│  │  │  - Reports height via postMessage     │    │  │
│  │  │  - Reads URL params for tracking      │    │  │
│  │  │  - Reports user interactions          │    │  │
│  │  └───────────────────────────────────────┘    │  │
│  │                                               │  │
│  └───────────────────────────────────────────────┘  │
│                                                     │
│  window.addEventListener('message', handler)        │
│  - Resizes iframe based on height messages          │
│  - Logs/tracks events from component                │
└─────────────────────────────────────────────────────┘
```

**Key principles:**

1. Each component is a **single, self-contained HTML file** — no build tools, no external JS dependencies.
2. The component communicates with the host page **exclusively via `window.postMessage`**.
3. The host page identifies messages by a **`source` field** unique to each component.
4. Tracking parameters are passed to the component via **URL query string** on the iframe `src`.
5. The component has a **transparent background** so it visually integrates with the host.
6. Height is managed by the component **measuring its own content** and reporting it to the host.

---

## Message Protocol

### Envelope Format

Every message sent between component and host is a plain JavaScript object passed through `window.postMessage`. All messages follow this envelope structure:

#### Component → Host

```javascript
{
  source: "component-id",   // Unique string identifying this component
  type:   "message-type",   // The event type (see table below)
  // ...additional fields specific to the message type
}
```

#### Host → Component

```javascript
{
  target: "component-id",   // Must match the component's source ID
  type:   "message-type",   // The command type (see table below)
  // ...additional fields specific to the message type
}
```

The `source` / `target` field is how both sides filter messages. This allows multiple iframes on the same page without collision.

---

### Messages: Component → Host

These are sent by the component inside the iframe to the host page via `window.parent.postMessage(...)`.

#### `ready`

Sent once when the component has fully initialized.

```javascript
{
  source: "component-id",
  type: "ready",
  height: 528,                          // Content height in pixels
  tracking: {                           // All URL params received
    utm_source: "google",
    gclid: "abc123"
  }
}
```

**When:** Fires once, at the end of the component's initialization script.
**Host should:** Set the initial iframe height. Confirm the component loaded successfully.

---

#### `resize`

Sent whenever the component's content height changes.

```javascript
{
  source: "component-id",
  type: "resize",
  height: 612                           // New content height in pixels
}
```

**When:** Fires on window resize, responsive breakpoint changes, content reflow — any time the measured height differs from the last reported value.
**Host should:** Update `iframe.style.height` to the reported value.

---

#### `cta-click`

Sent when the user clicks the primary call-to-action.

```javascript
{
  source: "component-id",
  type: "cta-click",
  href: "https://www.point.me/our-services/?utm_source=google&gclid=abc123",
  tracking: {
    utm_source: "google",
    gclid: "abc123"
  }
}
```

**When:** Fires on the CTA button/link click event.
**Host should:** Log analytics, fire conversion pixels, or optionally intercept navigation.

---

#### `card-visibility`

Sent when an individual card enters or leaves the viewport, or crosses a visibility threshold.

```javascript
{
  source: "component-id",
  type: "card-visibility",
  card: "search",                       // Value of data-card attribute
  visible: true,                        // Whether the card is in viewport
  ratio: 0.75                           // Intersection ratio (0 to 1)
}
```

**When:** Fires at threshold crossings: 0%, 25%, 50%, 75%, 100% visibility.
**Host should:** Track impressions, trigger scroll-based analytics.

---

#### `section-visibility`

Sent when the entire component section enters or leaves the viewport.

```javascript
{
  source: "component-id",
  type: "section-visibility",
  visible: true,
  ratio: 0.5
}
```

**When:** Same thresholds as `card-visibility`, but for the whole component.
**Host should:** Track component impressions, lazy-load adjacent content.

---

#### `visibility-change`

Sent when the browser tab is hidden or shown (user switches tabs).

```javascript
{
  source: "component-id",
  type: "visibility-change",
  hidden: true                          // true = tab hidden, false = tab visible
}
```

**When:** Fires on `document.visibilitychange` events inside the iframe.
**Host should:** Pause/resume analytics timers, pause expensive animations.

---

#### `info`

Sent in response to a `get-info` request from the host.

```javascript
{
  source: "component-id",
  type: "info",
  height: 528,
  tracking: { utm_source: "google" },
  url: "https://cdn.example.com/components/content-grid.html?utm_source=google"
}
```

**When:** Only in response to a `get-info` message from the host.
**Host should:** Use for debugging, diagnostics, or state inspection.

---

### Messages: Host → Component

These are sent by the host page to the component via `iframe.contentWindow.postMessage(...)`.

#### `get-height`

Requests the component to re-measure and report its height.

```javascript
{
  target: "component-id",
  type: "get-height"
}
```

**Component responds with:** A `resize` message.

---

#### `get-info`

Requests diagnostic information from the component.

```javascript
{
  target: "component-id",
  type: "get-info"
}
```

**Component responds with:** An `info` message.

---

## Lifecycle

```
Host loads page
  │
  ├─ Host renders <iframe src="component.html?utm_source=...">
  │
  ├─ Component HTML loads inside iframe
  │    │
  │    ├─ CSS parsed, content rendered
  │    │
  │    ├─ JS executes:
  │    │    ├─ Reads URL params from iframe src
  │    │    ├─ Appends tracking params to CTA href
  │    │    ├─ Measures content height (section.offsetHeight)
  │    │    ├─ Sets up ResizeObserver on document
  │    │    ├─ Sets up IntersectionObservers on cards + section
  │    │    ├─ Sets up visibilitychange listener
  │    │    ├─ Sets up message listener for host commands
  │    │    │
  │    │    └─ Sends "ready" message to host
  │    │         │
  │    │         └─── Host receives "ready"
  │    │              ├─ Sets iframe height
  │    │              └─ Logs event
  │    │
  │    ├─ User scrolls → card-visibility / section-visibility messages
  │    │
  │    ├─ Window resizes → "resize" message with new height
  │    │    └─── Host updates iframe height
  │    │
  │    ├─ User clicks CTA → "cta-click" message
  │    │    └─── Host logs analytics
  │    │
  │    └─ User switches tab → "visibility-change" message
  │
  └─ Host can send "get-height" or "get-info" at any time
```

---

## Building a New Component

### Required CSS

Every component **must** include these base styles to work correctly inside an iframe:

```css
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

html, body {
    height: auto;
    overflow: hidden;
}

body {
    background: transparent;
}
```

**Why each rule matters:**

| Rule | Reason |
|------|--------|
| `box-sizing: border-box` | Prevents padding from expanding elements beyond their set width |
| `margin: 0; padding: 0` | Eliminates default browser margins that would add to reported height |
| `html, body { height: auto }` | Prevents the document from stretching to fill the iframe viewport — critical for accurate height reporting |
| `overflow: hidden` | Prevents the iframe from developing scrollbars |
| `background: transparent` | Lets the host page's background show through |

### Required JavaScript

Every component needs this communication layer. The pattern below is copy-paste ready — just change `componentId` and add your own event bindings.

```javascript
(function () {
    'use strict';

    // ─── CONFIGURE THESE ───
    var PARENT_ORIGIN = '*';                    // Restrict in production
    var componentId   = 'my-component-name';    // Unique per component
    var rootElementId = 'my-root';              // ID of your outermost content element

    // ─── CORE: postMessage helper ───
    function postToParent(type, data) {
        if (window.parent === window) return;   // Not in an iframe
        window.parent.postMessage(
            Object.assign({ source: componentId, type: type }, data || {}),
            PARENT_ORIGIN
        );
    }

    // ─── CORE: Height reporting ───
    var lastHeight = 0;
    function reportHeight() {
        var el = document.getElementById(rootElementId);
        var h = el ? el.offsetHeight : document.documentElement.scrollHeight;
        if (h !== lastHeight) {
            lastHeight = h;
            postToParent('resize', { height: h });
        }
    }

    if (typeof ResizeObserver !== 'undefined') {
        new ResizeObserver(reportHeight).observe(document.documentElement);
    }
    window.addEventListener('resize', reportHeight);

    // ─── CORE: URL param forwarding ───
    function getTrackingParams() {
        var params = new URLSearchParams(window.location.search);
        var tracking = {};
        // Well-known tracking params get priority
        ['utm_source','utm_medium','utm_campaign','utm_term','utm_content',
         'gclid','fbclid','msclkid','dclid','twclid',
         'ref','source','campaign_id','ad_id','adset_id'
        ].forEach(function (k) {
            if (params.has(k)) tracking[k] = params.get(k);
        });
        // Pass through any other params too
        params.forEach(function (v, k) {
            if (!(k in tracking)) tracking[k] = v;
        });
        return tracking;
    }

    // ─── CORE: Apply tracking to any links ───
    function applyTrackingToLinks(selector) {
        var tracking = getTrackingParams();
        document.querySelectorAll(selector).forEach(function (link) {
            var url = new URL(link.href);
            Object.keys(tracking).forEach(function (k) {
                url.searchParams.set(k, tracking[k]);
            });
            link.href = url.toString();
        });
    }

    // ─── CORE: Visibility change ───
    document.addEventListener('visibilitychange', function () {
        postToParent('visibility-change', { hidden: document.hidden });
    });

    // ─── CORE: Listen for host commands ───
    window.addEventListener('message', function (e) {
        if (!e.data || e.data.target !== componentId) return;
        if (e.data.type === 'get-height') reportHeight();
        if (e.data.type === 'get-info') {
            var el = document.getElementById(rootElementId);
            postToParent('info', {
                height: el ? el.offsetHeight : 0,
                tracking: getTrackingParams(),
                url: window.location.href
            });
        }
    });

    // ═══ YOUR COMPONENT-SPECIFIC CODE BELOW ═══

    // Example: CTA click tracking
    var cta = document.getElementById('cta-button');
    if (cta) {
        cta.addEventListener('click', function (e) {
            postToParent('cta-click', {
                href: e.currentTarget.href,
                tracking: getTrackingParams()
            });
        });
    }

    // Example: Card-level intersection tracking
    if (typeof IntersectionObserver !== 'undefined') {
        var observer = new IntersectionObserver(function (entries) {
            entries.forEach(function (entry) {
                postToParent('card-visibility', {
                    card: entry.target.dataset.card,
                    visible: entry.isIntersecting,
                    ratio: Math.round(entry.intersectionRatio * 100) / 100
                });
            });
        }, { threshold: [0, 0.25, 0.5, 0.75, 1] });

        document.querySelectorAll('[data-card]').forEach(function (card) {
            observer.observe(card);
        });
    }

    // ─── CORE: Fire ready event (must be last) ───
    applyTrackingToLinks('.cta');
    reportHeight();
    var root = document.getElementById(rootElementId);
    postToParent('ready', {
        tracking: getTrackingParams(),
        height: root ? root.offsetHeight : 0
    });
})();
```

### Minimal Boilerplate

Here's a complete empty component file to start from:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Component Name</title>
    <link href="https://fonts.googleapis.com/css2?family=Figtree:wght@400;600;700&display=swap" rel="stylesheet">
    <style>
        *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
        html, body { height: auto; overflow: hidden; }
        body {
            font-family: 'Figtree', sans-serif;
            background: transparent;
            color: #1a1a2e;
            -webkit-font-smoothing: antialiased;
        }

        /* Your component styles here */
    </style>
</head>
<body>

<section id="component-root">
    <!-- Your component HTML here -->
</section>

<script>
(function () {
    'use strict';
    var PARENT_ORIGIN = '*';
    var componentId = 'your-component-id';      // Change this
    var rootElementId = 'component-root';        // Match your root element

    function postToParent(type, data) {
        if (window.parent === window) return;
        window.parent.postMessage(
            Object.assign({ source: componentId, type: type }, data || {}),
            PARENT_ORIGIN
        );
    }

    var lastHeight = 0;
    function reportHeight() {
        var el = document.getElementById(rootElementId);
        var h = el ? el.offsetHeight : document.documentElement.scrollHeight;
        if (h !== lastHeight) {
            lastHeight = h;
            postToParent('resize', { height: h });
        }
    }

    if (typeof ResizeObserver !== 'undefined') {
        new ResizeObserver(reportHeight).observe(document.documentElement);
    }
    window.addEventListener('resize', reportHeight);

    function getTrackingParams() {
        var params = new URLSearchParams(window.location.search);
        var tracking = {};
        ['utm_source','utm_medium','utm_campaign','utm_term','utm_content',
         'gclid','fbclid','msclkid','dclid','twclid',
         'ref','source','campaign_id','ad_id','adset_id'
        ].forEach(function (k) { if (params.has(k)) tracking[k] = params.get(k); });
        params.forEach(function (v, k) { if (!(k in tracking)) tracking[k] = v; });
        return tracking;
    }

    document.addEventListener('visibilitychange', function () {
        postToParent('visibility-change', { hidden: document.hidden });
    });

    window.addEventListener('message', function (e) {
        if (!e.data || e.data.target !== componentId) return;
        if (e.data.type === 'get-height') reportHeight();
        if (e.data.type === 'get-info') {
            var el = document.getElementById(rootElementId);
            postToParent('info', {
                height: el ? el.offsetHeight : 0,
                tracking: getTrackingParams(),
                url: window.location.href
            });
        }
    });

    // ── Your component-specific event bindings here ──

    // ── Ready (keep last) ──
    reportHeight();
    var root = document.getElementById(rootElementId);
    postToParent('ready', {
        tracking: getTrackingParams(),
        height: root ? root.offsetHeight : 0
    });
})();
</script>

</body>
</html>
```

---

## Building the Host Page

### Embedding the Iframe

```html
<iframe
    id="component-iframe"
    src="https://cdn.example.com/components/content-grid.html?utm_source=google&gclid=abc123"
    title="Descriptive title for accessibility"
    style="width: 100%; border: none; display: block;"
></iframe>
```

**Key attributes:**

| Attribute | Value | Why |
|-----------|-------|-----|
| `src` | Component URL + query params | Tracking params are passed here |
| `style="width: 100%"` | Full container width | Component is responsive internally |
| `style="border: none"` | No iframe border | Seamless visual integration |
| `style="display: block"` | Block layout | Removes inline whitespace gap below iframe |
| `title` | Descriptive text | Accessibility (screen readers) |

Do **not** set a fixed `height` — the component will tell you what it needs.

### Handling Messages

The host needs a single message listener that filters by `source`:

```javascript
window.addEventListener('message', function (e) {
    // Filter: only handle messages from our component
    if (!e.data || e.data.source !== 'your-component-id') return;

    switch (e.data.type) {
        case 'ready':
        case 'resize':
            // Set iframe height
            document.getElementById('component-iframe').style.height = e.data.height + 'px';
            break;

        case 'cta-click':
            // Track CTA clicks
            console.log('CTA clicked:', e.data.href);
            // analytics.track('cta_click', e.data.tracking);
            break;

        case 'card-visibility':
            // Track which cards were seen
            if (e.data.visible && e.data.ratio >= 0.5) {
                // analytics.track('card_impression', { card: e.data.card });
            }
            break;

        case 'section-visibility':
            // Track component impression
            if (e.data.visible && e.data.ratio >= 0.5) {
                // analytics.track('component_impression');
            }
            break;

        case 'visibility-change':
            // Tab hidden/shown
            break;

        case 'info':
            // Response to get-info request
            console.log('Component info:', e.data);
            break;
    }
});
```

### Sending Commands to the Component

```javascript
var iframe = document.getElementById('component-iframe');

// Request height re-measurement
iframe.contentWindow.postMessage(
    { target: 'your-component-id', type: 'get-height' },
    '*'
);

// Request diagnostic info
iframe.contentWindow.postMessage(
    { target: 'your-component-id', type: 'get-info' },
    '*'
);
```

### Minimal Host Boilerplate

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Host Page</title>
</head>
<body>

<!-- Your page content above -->

<iframe
    id="component-iframe"
    src="component.html?utm_source=website&utm_campaign=test"
    title="Embedded Component"
    style="width: 100%; border: none; display: block;"
></iframe>

<!-- Your page content below -->

<script>
(function () {
    var iframe = document.getElementById('component-iframe');
    var COMPONENT_ID = 'your-component-id';

    window.addEventListener('message', function (e) {
        if (!e.data || e.data.source !== COMPONENT_ID) return;

        // Auto-resize iframe
        if ((e.data.type === 'resize' || e.data.type === 'ready') && e.data.height > 0) {
            iframe.style.height = e.data.height + 'px';
        }

        // Handle other events as needed
        // if (e.data.type === 'cta-click') { ... }
    });
})();
</script>

</body>
</html>
```

---

## URL Parameter Forwarding

Tracking parameters flow through the system like this:

```
User visits host page:
  https://www.point.me/?utm_source=google&gclid=abc123
       │
       │  Host page reads its own URL params and passes them
       │  to the iframe src attribute:
       ▼
  <iframe src="component.html?utm_source=google&gclid=abc123">
       │
       │  Component JS reads params from its own URL:
       │  new URLSearchParams(window.location.search)
       ▼
  CTA button href gets params appended:
  https://www.point.me/our-services/?utm_source=google&gclid=abc123
       │
       │  On click, component sends cta-click message
       │  with full tracking payload back to host
       ▼
  Host can fire analytics with the tracking data
```

**Supported parameters (with priority):**

| Parameter | Source |
|-----------|--------|
| `utm_source`, `utm_medium`, `utm_campaign`, `utm_term`, `utm_content` | Standard UTM |
| `gclid` | Google Ads |
| `fbclid` | Facebook/Meta Ads |
| `msclkid` | Microsoft Ads |
| `dclid` | DoubleClick |
| `twclid` | Twitter/X Ads |
| `ref`, `source`, `campaign_id`, `ad_id`, `adset_id` | Generic |

Any additional query parameters not in this list are also forwarded — the list just ensures these well-known ones take precedence if duplicated.

---

## Height Management

This is the trickiest part of iframe embedding. Here's exactly how it works and why.

### The Problem

An iframe defaults to 150px tall. The content inside may be 500px, 800px, or vary with viewport width. The host page has no way to measure the iframe's content directly (cross-origin security). So the component must self-report.

### The Solution

```
Component                              Host
   │                                     │
   ├─ Measures: section.offsetHeight     │
   │  (NOT document.scrollHeight)        │
   │                                     │
   ├─ postMessage({ type: 'resize',  ──► │
   │    height: 528 })                   ├─ iframe.style.height = '528px'
   │                                     │
   │  [user resizes window]              │
   │                                     │
   ├─ ResizeObserver fires               │
   ├─ Re-measures: section.offsetHeight  │
   ├─ Height changed? Send resize    ──► │
   │                                     ├─ iframe.style.height = '612px'
```

### Critical: Measure the Content Element, Not the Document

```javascript
// WRONG — creates feedback loop, never shrinks
var h = document.documentElement.scrollHeight;

// CORRECT — measures actual content, shrinks when content shrinks
var section = document.getElementById('component-root');
var h = section.offsetHeight;
```

**Why:** Once the host sets `iframe.style.height = '600px'`, the iframe's viewport becomes 600px. `document.documentElement.scrollHeight` will then report *at least* 600px (because the document expands to fill the viewport), meaning the height can never decrease. Measuring a specific content element avoids this.

### Required CSS for Accurate Measurement

```css
html, body {
    height: auto;      /* Don't stretch to fill iframe viewport */
    overflow: hidden;  /* No scrollbars inside the iframe */
}
```

Without `height: auto`, the `<html>` element stretches to match the iframe height, and the content element's `offsetHeight` may be influenced by its parent's height.

---

## Security Considerations

### Origin Restriction

The boilerplate uses `PARENT_ORIGIN = '*'` for development. In production, restrict this:

```javascript
// Component: only send to known host
var PARENT_ORIGIN = 'https://www.point.me';

// Host: validate message origin
window.addEventListener('message', function (e) {
    if (e.origin !== 'https://cdn.point.me') return;
    // ...handle message
});
```

### Input Validation

Always validate message payloads before acting on them:

```javascript
// Host: validate height is a positive number
if (e.data.type === 'resize') {
    var h = parseInt(e.data.height, 10);
    if (h > 0 && h < 10000) {  // Sanity check
        iframe.style.height = h + 'px';
    }
}
```

### Content Security Policy

If the host has a CSP, add the component's origin to `frame-src`:

```
Content-Security-Policy: frame-src https://cdn.point.me;
```

---

## Reference: Content Grid Component

The first component built using this protocol.

| Field | Value |
|-------|-------|
| **Component ID** | `content-grid-headline` |
| **File** | `content-grid.html` |
| **Root element** | `<section id="content-grid">` |
| **CTA link** | `https://www.point.me/our-services/` |
| **Breakpoints** | 768px (2-col), 1024px (3-col + centered 4th), 1280px (4-col) |
| **Custom events** | `cta-click`, `card-visibility` (search, save, live, confidence), `section-visibility` |

### Card identifiers

Each card has a `data-card` attribute used in `card-visibility` messages:

| `data-card` | Card title |
|-------------|-----------|
| `search` | Search 100+ airlines in one click |
| `save` | Save 50%-90% of your points |
| `live` | See live availability, every time |
| `confidence` | Confidence built in |
