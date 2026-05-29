# Pre-Market Brief: UX/UI Design Specification

**Document type**: Design specification (Phase 4 handoff artifact)
**Status**: Proposed. Not yet scheduled for a sprint.
**Audience**: Bryce (developer) + frontend-developer agent (template implementation)
**Companion docs**: `premarket_brief_design.md`, `premarket_brief_architecture.md`, `premarket_brief_build_plan.md`
**Render targets**: (1) `premarket.home` web dashboard, (2) 07:00 ET email

---

## 0. Overview and design philosophy

The brief is a pre-flight checklist, not a trading terminal. It runs once a day, is read in roughly 30 seconds, and the most important outcome is that Allie leaves her desk knowing two things: her risk posture for the session, and which clock-times to put a timer on. Everything else is supporting evidence for those two conclusions.

The design problem is a specific one: the trader has ADHD, has about one year of futures experience, and is reading this at 07:00 before the market opens, when executive function is not at its peak and the temptation to overtrade is highest. "Highly compelling fintech" and "must not overwhelm" are not in tension if the design principle is that **clarity is the form of luxury here**. The brief earns visual authority through precision, restraint, and consistency, not density.

### Visual direction

The aesthetic is dark-mode-first, monochrome-anchored, with a single green/red/gray semantic accent layer. Reference points: the quiet authority of a Bloomberg terminal stripped of its noise, the typographic rigor of a well-designed CLI readout, the spatial calm of a medical monitoring display. None of those literally, but those are the emotional registers.

What this is not: neon trading-app urgency (high arousal = bad for ADHD at 07:00), crypto-dashboard clutter, or generic fintech gradients. The palette runs nearly black backgrounds with off-white text and extremely deliberate use of color. Color only means something, and it only means green/red/gray.

The typeface system uses two families: a geometric sans-serif for all labels, text, and UI chrome, and a monospaced font with tabular figures for all numbers. This distinction is functional. Monospace + tabular figures prevents number columns from jittering as values change, makes alignment readable at a glance, and signals to the reader that a number is a number, not prose.

Motion is nearly absent by design. The one exception is the HTMX poll-to-refresh, which updates the timestamp and price silently without repainting the whole screen. Calm over responsive; certainty over animation.

---

## 1. Design tokens

### 1.1 Color system (CSS custom properties)

All values are defined on `:root` for the web dashboard. The email variant inlines these as literal hex values because email clients do not evaluate CSS variables.

```css
:root {
  /* --- Backgrounds --- */
  --color-bg-base:        #0d0f11;   /* near-black, primary page background */
  --color-bg-surface:     #141719;   /* card / panel surface */
  --color-bg-elevated:    #1c1f22;   /* elevated card, e.g. price ladder */
  --color-bg-inset:       #0a0c0e;   /* inset well, e.g. drill-down section */
  --color-bg-border:      #272b2f;   /* subtle separator */

  /* --- Text --- */
  --color-text-primary:   #e8eaed;   /* primary readable text, ~92% white */
  --color-text-secondary: #8b9199;   /* secondary labels, metadata */
  --color-text-muted:     #545b63;   /* timestamps, footnotes */
  --color-text-inverse:   #0d0f11;   /* text on colored badge fills */

  /* --- Semantic: direction and state (FIXED, per CLAUDE.md) --- */
  --color-positive:       #22c55e;   /* green: up / positive / pass   (WCAG AA on bg-surface: 4.8:1) */
  --color-positive-dim:   #15803d;   /* muted green for fills/backgrounds */
  --color-positive-bg:    #052e16;   /* very dark green for badge fill */

  --color-negative:       #ef4444;   /* red: down / negative / fail    (WCAG AA on bg-surface: 4.6:1) */
  --color-negative-dim:   #b91c1c;   /* muted red for fills/backgrounds */
  --color-negative-bg:    #2d0a0a;   /* very dark red for badge fill */

  --color-neutral:        #71717a;   /* gray: inconclusive / insufficient */
  --color-neutral-dim:    #3f3f46;   /* muted gray for fills */
  --color-neutral-bg:     #18181b;   /* very dark gray for badge fill */

  /* --- Warning (YELLOW mode badge only) --- */
  --color-caution:        #eab308;   /* yellow: elevated risk, trade smaller */
  --color-caution-dim:    #a16207;
  --color-caution-bg:     #1c1500;

  /* --- Accent (structural, non-semantic) --- */
  --color-accent:         #38bdf8;   /* sky blue: current price marker, active UI chrome */
  --color-accent-dim:     #0369a1;
  --color-accent-bg:      #0c1a26;

  /* --- Functional --- */
  --color-focus-ring:     #38bdf8;   /* keyboard focus outline */
  --color-scrollbar:      #272b2f;
}
```

**WCAG contrast notes:**
- `--color-text-primary` (#e8eaed) on `--color-bg-surface` (#141719): ~12.8:1. Exceeds AAA.
- `--color-positive` (#22c55e) on `--color-bg-surface` (#141719): ~4.8:1. Passes AA for normal text (3:1 minimum for large/UI text, 4.5:1 for small text). For the badge fill pair (`--color-positive` on `--color-positive-bg`): ~6.2:1. Passes AA.
- `--color-negative` (#ef4444) on `--color-bg-surface` (#141719): ~4.6:1. Passes AA.
- `--color-caution` (#eab308) on `--color-bg-surface` (#141719): ~7.1:1. Passes AAA.
- `--color-accent` (#38bdf8) on `--color-bg-surface` (#141719): ~5.9:1. Passes AA.
- `--color-text-secondary` (#8b9199) on `--color-bg-base` (#0d0f11): ~4.6:1. Passes AA.

Color is never the sole encoding for state. See section 7 (accessibility) for the icon/shape layer applied to every color-coded element.

### 1.2 Typography scale

**UI font (sans-serif)**: Inter, with the fallback chain `'Inter', 'SF Pro Text', system-ui, -apple-system, sans-serif`. Load Inter from a vendored local copy (not a CDN) to keep the service fully self-hosted and to eliminate a network dependency at 07:00 when the brief must render reliably.

**Numeric font (monospace/tabular)**: JetBrains Mono, with the fallback chain `'JetBrains Mono', 'Fira Code', 'Cascadia Code', 'Menlo', monospace`. Apply `font-variant-numeric: tabular-nums; font-feature-settings: 'tnum' 1;` globally to this font. Every price, point-distance, percentage, and ATR value uses this family. Labels and prose use Inter.

```css
:root {
  /* --- Type families --- */
  --font-ui:     'Inter', 'SF Pro Text', system-ui, -apple-system, sans-serif;
  --font-mono:   'JetBrains Mono', 'Fira Code', 'Cascadia Code', 'Menlo', monospace;

  /* --- Scale (rem, base 16px) --- */
  --text-xs:     0.6875rem;  /* 11px, timestamps, footnotes */
  --text-sm:     0.75rem;    /* 12px, secondary labels, metadata */
  --text-base:   0.875rem;   /* 14px, body text, component labels */
  --text-md:     1rem;       /* 16px, headline supporting text */
  --text-lg:     1.125rem;   /* 18px, section headers */
  --text-xl:     1.375rem;   /* 22px, headline line */
  --text-2xl:    1.75rem;    /* 28px, Risk Mode badge text */
  --text-3xl:    2.25rem;    /* 36px, reserved, not used in current design */

  /* --- Weights --- */
  --weight-regular: 400;
  --weight-medium:  500;
  --weight-semibold: 600;
  --weight-bold:    700;

  /* --- Line heights --- */
  --leading-tight:  1.2;
  --leading-snug:   1.375;
  --leading-normal: 1.5;
  --leading-loose:  1.75;

  /* --- Letter spacing --- */
  --tracking-tight:  -0.01em;
  --tracking-normal:  0;
  --tracking-wide:    0.04em;   /* used for ALL CAPS labels and badge text */
  --tracking-widest:  0.08em;   /* used for section divider labels */
}
```

**Application rules:**

| Element | Font | Size | Weight | Color |
|---|---|---|---|---|
| Risk Mode badge text | Inter | `--text-2xl` | bold | `--color-text-inverse` on colored fill |
| Risk Mode badge label | Inter | `--text-sm` | medium | `--color-text-secondary` |
| Headline line text | Inter | `--text-xl` | semibold | per semantic color |
| Headline numbers | JetBrains Mono | `--text-xl` | semibold | per semantic color |
| Catalyst banner | Inter | `--text-base` | medium | `--color-text-primary` |
| Price ladder level labels | Inter | `--text-sm` | regular | `--color-text-secondary` |
| Price ladder prices | JetBrains Mono | `--text-base` | semibold | `--color-text-primary` |
| Price ladder distances | JetBrains Mono | `--text-sm` | regular | `--color-text-secondary` |
| Current price marker | JetBrains Mono | `--text-md` | bold | `--color-accent` |
| Volatility line | Inter | `--text-base` | regular | `--color-text-primary` |
| Volatility numbers | JetBrains Mono | `--text-base` | semibold | `--color-text-primary` |
| Section headers | Inter | `--text-sm` | semibold | `--color-text-muted` tracking-widest uppercase |
| Body / drill-down | Inter | `--text-base` | regular | `--color-text-secondary` |
| Timestamp / footnote | Inter | `--text-xs` | regular | `--color-text-muted` |

### 1.3 Spacing scale

```css
:root {
  --space-1:   4px;
  --space-2:   8px;
  --space-3:  12px;
  --space-4:  16px;
  --space-5:  20px;
  --space-6:  24px;
  --space-8:  32px;
  --space-10: 40px;
  --space-12: 48px;
  --space-16: 64px;
}
```

The brief uses a loose vertical rhythm: `--space-6` (24px) between the five main blocks, `--space-4` (16px) between sub-elements within a block, `--space-2` (8px) for tight pairs (label + value).

### 1.4 Border radius

```css
:root {
  --radius-sm:  4px;    /* inner elements, badge corners */
  --radius-md:  8px;    /* cards, panels */
  --radius-lg: 12px;    /* price ladder container */
  --radius-pill: 999px; /* Risk Mode badge itself */
}
```

### 1.5 Elevation and shadow

The dark-mode surface hierarchy is achieved primarily through background color steps, not drop shadows. Shadows are used sparingly for the one element that needs to float: the current-price marker on the price ladder.

```css
:root {
  --shadow-none:    none;
  --shadow-inset:   inset 0 1px 0 0 rgba(255,255,255,0.04);
  --shadow-marker:  0 0 0 2px var(--color-bg-elevated),
                    0 0 0 3px var(--color-accent);   /* price marker ring */
  --shadow-card:    0 1px 3px 0 rgba(0,0,0,0.6),
                    0 1px 2px -1px rgba(0,0,0,0.6);
}
```

### 1.6 Motion timing

Motion is used only where it reduces cognitive overhead (silent HTMX refresh, expand/collapse). It is never used decoratively.

```css
:root {
  --duration-instant:  0ms;    /* state changes that must not animate (badge color flip) */
  --duration-fast:   120ms;    /* micro-transitions: focus ring, hover tint */
  --duration-normal: 200ms;    /* expand/collapse, drill-down reveal */
  --duration-slow:   300ms;    /* HTMX content swap fade */
  --ease-standard:   cubic-bezier(0.4, 0, 0.2, 1);
  --ease-decel:      cubic-bezier(0, 0, 0.2, 1);   /* panel opening */
  --ease-accel:      cubic-bezier(0.4, 0, 1, 1);   /* panel closing */
}
```

**Deliberate non-animations:** The Risk Mode badge color must change with zero animation. An animated badge color change (green to red) reads as "in transition," which is wrong for a status signal. The current price marker on the ladder must not pulse or blink. Number updates via HTMX must not count-up or animate values; silent replacement only.

---

## 2. Layout system

### 2.1 The responsive grid

The brief is a narrow, vertical document. It does not use a multi-column grid. The layout is a centered single column, maximum width capped to preserve comfortable line lengths and avoid the brief feeling like a spreadsheet.

```css
.brief-container {
  max-width: 680px;          /* comfortable reading width for this content */
  margin: 0 auto;
  padding: var(--space-6) var(--space-4);
}

/* On desktop screens wider than 960px, add a right panel for the drill-down */
@media (min-width: 960px) {
  .brief-layout {
    display: grid;
    grid-template-columns: 680px 1fr;
    grid-template-areas:
      "brief  drill"
      "brief  drill";
    gap: var(--space-8);
    max-width: 1160px;
    margin: 0 auto;
    align-items: start;
  }
  .brief-main  { grid-area: brief; }
  .brief-drill { grid-area: drill; position: sticky; top: var(--space-6); }
}
```

On mobile (viewport below 768px), the single-column layout is the natural default. The drill-down section stacks below the main five blocks.

### 2.2 One-screen glance composition

The five main blocks fit on one screen. The target is that a user at 1280x800 desktop or an iPhone 14 Pro in portrait (390x844) can read the complete decision-making summary without scrolling. Below is the vertical priority order, with approximate height allocations:

| Block | Desktop height | Mobile height |
|---|---|---|
| Page header (title, timestamp, instrument) | ~48px | ~48px |
| Block 1: Risk Mode badge | ~72px | ~72px |
| Block 2: Headline line | ~56px | ~64px |
| Block 3: Catalyst banner | ~56-80px | ~64-96px |
| Block 4: Price ladder | ~220px | ~220px |
| Block 5: Volatility + sizing | ~64px | ~80px |
| Total (approximate) | ~516px | ~584px |

This keeps the entire glance within a 680px-tall viewport on desktop, and within the 844px iPhone 14 Pro viewport with space for browser chrome. Drill-down sits below the fold, intentionally.

### 2.3 Visual hierarchy summary

Reading priority order, encoded through size, weight, position, and color:

1. **Risk Mode badge** (largest type, colored fill, top position)
2. **Headline numbers** (largest monospace, semantic color)
3. **Catalyst times** (high-contrast, bold, action-oriented)
4. **Current price position on ladder** (accent color marker, bold)
5. **Nearest levels above/below** (bold labels)
6. **Volatility + sizing numbers**
7. **Secondary labels, metadata, distances**
8. **Timestamp, footnotes, drill-down**

---

## 3. ASCII wireframes

### 3.1 Desktop layout (1280px wide, brief column)

```
+------------------------------------------------------------------+
|  premarket.home                                  07:04 ET snapped|
+------------------------------------------------------------------+
|  NQ / MNQ  WEDNESDAY MAY 28 2026                                 |
+------------------------------------------------------------------+
|                                                                  |
|  +------------------------------------------------------------+  |
|  |  [GREEN badge]  TRADE NORMAL SIZE                          |  |
|  |  Normal day. Trade your plan.                              |  |
|  +------------------------------------------------------------+  |
|                                                                  |
|  NQ +78 pts (+0.4%)  gap up, near top of overnight range        |
|  Gap is 0.3x ATR -- modest.                                     |
|                                                                  |
|  +-[ CATALYSTS ]---------------------------------------------+  |
|  |  No major scheduled events today.                         |  |
|  |  Normal session.                                          |  |
|  +-----------------------------------------------------------+  |
|                                                                  |
|  +-[ LEVELS ]------------------------------------------------+   |
|  |                                                           |   |
|  |         ONH      20,460  +42 pts  (+1.0x ATR above)      |   |
|  |  >>>  20,418  ////// CURRENT PRICE //////                 |   |
|  |    *  PDH      20,405  -13 pts  (0.3x ATR below) *       |   |
|  |         ON VWAP  20,360  -58 pts                          |   |
|  |         PDL      20,290  -128 pts                         |   |
|  |         ONL      20,255  -163 pts                         |   |
|  |                                                           |   |
|  +-----------------------------------------------------------+   |
|                                                                  |
|  ATR 14d: 210 pts  |  Gap: 0.3x ATR  |  VIX 16.2  ~1.0% move  |
|  Normal vol. 1 MNQ full stop = approx $X. Daily stop: N stops.  |
|                                                                  |
|  [ + Show detail ]                                               |
|                                                                  |
+------------------------------------------------------------------+
```

### 3.2 Desktop two-column: brief + drill-down open (960px+)

```
+----------------------------------------------+ +-------------------+
|  NQ / MNQ  WED MAY 28                        | | OVERNIGHT DETAIL  |
+----------------------------------------------+ |                   |
|                                              | | Open:    20,340   |
|  [GREEN]  TRADE NORMAL SIZE                  | | High:    20,460   |
|                                              | | Low:     20,255   |
|  NQ +78 pts (+0.4%)  gap up                  | | Close:   20,418   |
|  Gap is 0.3x ATR -- modest.                  | | Range:   205 pts  |
|                                              | | Avg ONR: 180 pts  |
|  No major events today.                      | | Character: mild   |
|                                              | |                   |
|  +--[ LEVELS ]---------------------------+   | | FULL NUMBERS      |
|  |  ONH    20,460  +42 (1.0x above)     |   | | Prior settle:     |
|  |  >>> CURRENT 20,418 <<<              |   | |   20,340          |
|  |  * PDH  20,405  -13 (0.3x below) *  |   | | PDH:  20,490      |
|  |  ON VWAP 20,360  -58               |   | | PDL:  20,201      |
|  |  PDL    20,290  -128              |   | |                   |
|  |  ONL    20,255  -163              |   | | CATALYST DETAIL   |
|  +------------------------------------+   | | (empty today)     |
|                                              | |                   |
|  ATR 210 | Gap 0.3x | VIX 16.2 ~1.0%        | | Snapshot at:      |
|                                              | | 07:04:22 ET       |
|  [ - Hide detail ]                           | | Data: Databento   |
+----------------------------------------------+ +-------------------+
```

### 3.3 Mobile layout (390px, iPhone portrait)

```
+-------------------------------+
|  premarket.home   07:04 ET    |
+-------------------------------+
|  NQ / MNQ  Wed May 28         |
+-------------------------------+
|                               |
|  +---------------------------+|
|  |   GREEN                   ||
|  |   TRADE NORMAL SIZE       ||
|  |   Normal day.             ||
|  +---------------------------+|
|                               |
|  NQ +78 pts (+0.4%)           |
|  gap up, quiet overnight      |
|  Gap = 0.3x ATR               |
|                               |
|  +---------------------------+|
|  | No major events today.    ||
|  | Normal session.           ||
|  +---------------------------+|
|                               |
|  +--[ LEVELS ]---------------+|
|  |  ONH    20,460   +42      ||
|  |  >>>   20,418   <<<       ||
|  |  *PDH  20,405   -13 *    ||
|  |  VWAP  20,360   -58      ||
|  |  PDL   20,290  -128      ||
|  |  ONL   20,255  -163      ||
|  +---------------------------+|
|                               |
|  ATR 210 | Gap 0.3x | ~1.0%  |
|  Normal vol. Daily stop: N    |
|                               |
|  [ + Show detail ]            |
+-------------------------------+
```

### 3.4 RED day / event day variant (mobile)

```
+-------------------------------+
|  premarket.home   07:04 ET    |
+-------------------------------+
|  NQ / MNQ  Wed May 28         |
+-------------------------------+
|                               |
|  +---------------------------+|
|  |   RED                     ||
|  |   STAND ASIDE              ||
|  |   High-impact event today. ||
|  +---------------------------+|
|                               |
|  NQ +24 pts (+0.1%)            |
|  gap up, choppy overnight     |
|  Gap = 0.1x ATR (non-event)   |
|                               |
|  +--[ CATALYSTS ]------------+|
|  |  Avoid 08:28-08:35 (CPI)  ||
|  |  Avoid 10:00-10:05 (ISM)  ||
|  |                           ||
|  |  First loss = info.       ||
|  |  Daily stop: N stops.     ||
|  +---------------------------+|
|                               |
|  [price ladder -- same]       |
|                               |
|  ATR 310 | Gap 0.1x | ~2.1%  |
|  HIGH VOL. Wider stop,        |
|  fewer contracts.             |
+-------------------------------+
```

---

## 4. Component-by-component design

### 4.1 Risk Mode badge

**Purpose**: The single synthesized conclusion. Reads in under one second. Everything else is evidence.

**Anatomy:**

```
+------------------------------------------------+
|                                                |
|   [ICON]  MODE TEXT                            |
|           Supporting one-liner                 |
|                                                |
+------------------------------------------------+
```

- Outer container: full width of the brief column, `border-radius: --radius-lg`, no border, background from the mode-specific `--color-*-bg` token.
- A left border accent, 4px wide, solid, `--color-*` token (the full-brightness version). This provides a strong visual edge without filling the whole card in saturated color.
- Mode text: `--font-ui`, `--text-2xl`, `--weight-bold`, `--tracking-wide`, all caps. Color: the full-brightness semantic token (`--color-positive`, `--color-caution`, `--color-negative`).
- Supporting one-liner: `--font-ui`, `--text-base`, `--weight-regular`, `--color-text-secondary`.
- Icon to the left of mode text: a filled circle (12px diameter) in the semantic color. This provides the non-color encoding required for accessibility (see section 7).

**Three states:**

| Mode | Background | Left border | Text color | Icon | Mode text |
|---|---|---|---|---|---|
| GREEN | `--color-positive-bg` | `--color-positive` | `--color-positive` | Filled circle, positive | TRADE NORMAL SIZE |
| YELLOW | `--color-caution-bg` | `--color-caution` | `--color-caution` | Warning triangle, caution | TRADE SMALLER |
| RED | `--color-negative-bg` | `--color-negative` | `--color-negative` | Octagon/stop, negative | STAND ASIDE |

**No animation on mode change.** State is set server-side at render time; there is no client-side transition between modes.

**Supporting one-liners per mode (static, set by the brief engine):**

- GREEN: "Normal day. Trade your plan."
- YELLOW: "Elevated volatility or near catalyst. Reduce size."
- RED: "High-impact event before or during your session. Stand aside or trade tiny until it clears."

The mode badge must appear above the fold on all target devices with no scrolling required. It is the first element in DOM order.

**HTMX behavior**: the badge is inside `brief_fragment.html`, which is the target of the poll-to-refresh. The badge re-renders silently with the rest of the fragment.

**HTML structure (Jinja2 sketch):**
```html
<div class="risk-badge risk-badge--{{ brief.risk_mode | lower }}"
     aria-label="Risk mode: {{ brief.risk_mode }}">
  <span class="risk-badge__icon" aria-hidden="true"></span>
  <div class="risk-badge__content">
    <span class="risk-badge__mode">{{ brief.risk_mode_label }}</span>
    <span class="risk-badge__sub">{{ brief.risk_mode_sub }}</span>
  </div>
</div>
```

### 4.2 Headline line

**Purpose**: The primary quantitative fact about today, in one sentence, color-coded by direction, grayed out when the gap is too small to read as a signal.

**Anatomy:**
```
NQ  +78 pts  (+0.4%)   gap up,  near top of overnight range
[label] [delta][pct]   [direction qualifier]  [position qualifier]
```

- Container: no card, no border. Sits as a text block below the badge with `--space-4` gap.
- "NQ": `--font-ui`, `--text-md`, `--weight-semibold`, `--color-text-muted`. This is always gray because the instrument label is not semantic.
- Delta value and percentage: `--font-mono`, `--text-xl`, `--weight-semibold`. Color: `--color-positive` when gap up, `--color-negative` when gap down, `--color-neutral` when gap is under ~0.2 ATR.
- Direction and position qualifier: `--font-ui`, `--text-md`, `--weight-regular`, `--color-text-secondary`.
- Volatility qualifier (e.g., "Gap = 0.3x ATR"): a second line, `--font-mono`, `--text-sm`, `--weight-regular`, `--color-text-muted`.

**States:**

- Normal gap up/down: semantic color on the numbers only.
- Gray/inconclusive (gap under 0.2 ATR): all parts of the headline render in `--color-neutral` and `--color-text-muted`. No green or red. The volatility qualifier says "Gap under threshold (non-event)."
- No data / CME holiday: headline reads "No session today (CME holiday)." in `--color-text-muted`.

**Number precision:** Delta in whole points (no decimal). Percentage to one decimal place. ATR fraction to one decimal place. Never show more precision than is decision-relevant.

**Direction encoding (beyond color):**
- An up or down arrow glyph (Unicode: `▲` and `▼`, or SVG chevrons) appears before the delta. The glyph uses the same semantic color as the delta. On gray/inconclusive days the glyph is replaced with `~` to signal "approximately flat."

### 4.3 Catalyst banner

**Purpose**: Transform an abstract event list into actionable clock-time windows Allie can set a phone timer for.

**Anatomy:**

**Case: no events**
```
+--------------------------------------------------+
|  No major events today.  Normal session.         |
+--------------------------------------------------+
```
Light border, `--color-bg-surface` fill, text in `--color-text-secondary`. Calm. This is important feedback: the absence of events is a positive signal.

**Case: 1-3 events**
```
+--------------------------------------------------+
|  CATALYSTS                                       |
|                                                  |
|  [clock] Avoid  08:28 - 08:35   CPI (high)       |
|  [clock] Avoid  10:00 - 10:05   ISM (medium)     |
|                                                  |
|  Hard rule: do not hold through these prints.    |
|  Stops do not protect you through the number.   |
+--------------------------------------------------+
```

- Container: `--color-bg-surface` fill, left border 4px in `--color-negative` (because events are inherently risk-elevating). `--radius-md`. `--space-4` internal padding.
- Section label "CATALYSTS": `--font-ui`, `--text-sm`, `--weight-semibold`, `--color-text-muted`, `--tracking-widest`, uppercase.
- Each event row: icon (clock SVG, `--color-text-muted`, 14px), then "Avoid" in `--font-ui` `--text-base` `--weight-semibold` `--color-negative`, then the time window in `--font-mono` `--text-base` `--weight-semibold` `--color-text-primary`, then the event name and impact in `--font-ui` `--text-base` `--weight-regular` `--color-text-secondary`.
- High-impact events: "Avoid" text is `--color-negative`. Medium-impact: "Avoid" text is `--color-caution`.
- Hard rule line: `--font-ui`, `--text-sm`, `--weight-regular`, `--color-text-muted`. Always present when any event is shown. Static copy, never changes.
- Cap at 3 events. If more than 3 exist, append "+ N more (see detail)" linking to the drill-down.

**Time window formatting:**

The brief engine computes windows as: event time minus 2 minutes (for pre-event freeze) to event time plus 5 minutes (for post-release volatility to clear). Displayed as "HH:MM - HH:MM" in ET. Always ET, always include the "ET" suffix on the first occurrence in the section.

**FOMC variant:** If the event is FOMC (14:00) or Fed presser (14:30), the hard rule line changes to: "FOMC: entire 13:55-15:00 window is high-risk. If your session runs this late, plan your exit before 13:50."

### 4.4 Price ladder (the hero component)

**Purpose**: Show where current price sits in physical space relative to the levels that matter. Proximity encoded by position, not arithmetic.

**Design principles:**

1. The vertical axis represents price. Higher price is higher on the screen. Current price always sits at its true proportional position within the displayed range.
2. Only the levels that matter today are shown (per the cut list in design doc 3A): ONH, PDH, ON VWAP, PDL, ONL, plus conditional weekly/monthly levels only when within ~1 ATR.
3. The nearest level above and the nearest level below are bolded (heavier weight, full-brightness color).
4. Distance is shown in points AND ATR-fractions. ATR-fractions are the decision-relevant unit; points are shown alongside for calibration.

**ASCII mock (detailed):**

```
 +-------------------------------------------------+
 |  LEVELS                          NQ  20,418     |
 |                                                  |
 |    20,460  ONH       +42 pts  1.0x ATR above     |
 |     ....                                         |
 |     ....                                         |
 | >>>20,418  CURRENT PRICE <<<                     |
 |     ....                                         |
 |  *  20,405  PDH   *  -13 pts  0.3x ATR below *   |
 |                                                  |
 |    20,360  ON VWAP   -58 pts  1.4x ATR below     |
 |                                                  |
 |    20,290  PDL      -128 pts  3.0x ATR below     |
 |                                                  |
 |    20,255  ONL      -163 pts  3.9x ATR below     |
 |                                                  |
 +-------------------------------------------------+
```

**SVG/CSS implementation approach:**

The ladder is a server-rendered SVG fragment embedded inline in `brief_fragment.html`. No JavaScript is needed to draw it. The FastAPI brief engine computes all positions as percentages of the displayed range before rendering.

```
Rendered range = (ONH + 1 ATR buffer top) to (ONL - 1 ATR buffer bottom)
Position of level L = (L - range_bottom) / (range_top - range_bottom)
```

SVG structure:

```svg
<svg class="price-ladder"
     viewBox="0 0 320 220"
     width="100%" height="220"
     aria-label="Price ladder showing current price relative to key levels"
     role="img">

  <!-- Background -->
  <rect width="320" height="220" fill="var(--color-bg-elevated)" rx="8"/>

  <!-- Vertical spine: the price axis -->
  <line x1="100" y1="16" x2="100" y2="204"
        stroke="var(--color-bg-border)" stroke-width="1"/>

  <!-- Level rows: one per level, y computed from price position -->
  <!-- Each row: dashed horizontal tick, label text (left), price text (mid), distance text (right) -->

  <!-- Example: ONH level at y=28 -->
  <line x1="94" y1="28" x2="106" y2="28"
        stroke="var(--color-text-muted)" stroke-width="1"/>
  <text x="88" y="32" text-anchor="end" font-size="11"
        fill="var(--color-text-secondary)" font-family="var(--font-ui)">ONH</text>
  <text x="112" y="32" text-anchor="start" font-size="12"
        fill="var(--color-text-primary)" font-family="var(--font-mono)">20,460</text>
  <text x="240" y="32" text-anchor="start" font-size="11"
        fill="var(--color-text-muted)" font-family="var(--font-mono)">+42 / 1.0x</text>

  <!-- Nearest level below: bold weight, accent or brighter color -->
  <!-- PDH, nearest level below current price -->
  <line x1="92" y1="110" x2="108" y2="110"
        stroke="var(--color-text-primary)" stroke-width="1.5"/>
  <text x="88" y="114" text-anchor="end" font-size="11" font-weight="700"
        fill="var(--color-text-primary)" font-family="var(--font-ui)">PDH</text>
  <text x="112" y="114" text-anchor="start" font-size="12" font-weight="700"
        fill="var(--color-text-primary)" font-family="var(--font-mono)">20,405</text>
  <text x="240" y="114" text-anchor="start" font-size="11"
        fill="var(--color-text-secondary)" font-family="var(--font-mono)">-13 / 0.3x</text>

  <!-- Current price marker: horizontal bar spanning full width, accent color -->
  <!-- y position is computed: (current_price - range_bottom) / range_span * chart_height -->
  <rect x="0" y="90" width="320" height="18"
        fill="var(--color-accent-bg)" rx="2"/>
  <line x1="0" y1="99" x2="320" y2="99"
        stroke="var(--color-accent)" stroke-width="1.5" stroke-dasharray="none"/>
  <text x="160" y="95" text-anchor="middle" font-size="13" font-weight="700"
        fill="var(--color-accent)" font-family="var(--font-mono)">20,418</text>
  <text x="8" y="95" text-anchor="start" font-size="10"
        fill="var(--color-accent)" font-family="var(--font-ui)">NOW</text>

</svg>
```

**Jinja2 rendering approach:**

The brief engine passes a `levels_svg` string to the template. `levels.py` computes the SVG as a Python string (or uses a simple template fragment), with all y-coordinates pre-calculated from the price data. No client-side computation.

**Conditional levels (weekly/monthly):** If `weekly_high` is within 1 ATR of current price, it is inserted into the levels list and rendered as a row with a subtler tick (dashed, `--color-text-muted`). If not within 1 ATR, it does not appear. The brief engine handles this suppression; the template never knows about levels that were suppressed.

**Distance column format:** All distances displayed as "±NNN / N.Nx" where NNN is whole points and N.N is ATR-fraction to one decimal place. The sign (+ for above, - for below) is always shown. Positive = above current price (resistance), negative = below (support). This consistent sign convention prevents misreading.

**Mobile adaptation:** On viewports below 480px, the distance column is compressed to show ATR-fraction only (the more decision-relevant value), with full values accessible in the drill-down.

### 4.5 Volatility and sizing line

**Purpose**: Give the daily volatility context and derive a concrete position-sizing implication. One compact block.

**Anatomy:**

```
ATR 14d: 210 pts  |  Gap: 0.3x ATR  |  VIX 16.2  (~1.0% expected move)
Normal vol day. 1 MNQ full stop = $X.  Daily stop: N stops.
```

- Container: no card border. Sits below the ladder with `--space-4` gap.
- First line: three metrics separated by vertical rules (`|` character styled with `--color-bg-border`). Each metric label in `--font-ui` `--text-sm` `--weight-regular` `--color-text-muted`, each value in `--font-mono` `--text-base` `--weight-semibold` `--color-text-primary`.
- Second line: the plain-language sizing implication. `--font-ui`, `--text-base`, `--weight-regular`, `--color-text-secondary`. Dollar and contract values in `--font-mono` inline.
- When config values (per-trade risk, daily stop) are not set: the second line reads "Risk settings not configured. See setup guide." in `--color-text-muted`. Dollar and contract values are replaced with "??". The brief engine never invents these values.
- HIGH VOLATILITY day (gap over 1x ATR or VIX above a threshold): the "Normal vol day." prefix becomes "HIGH VOL DAY." in `--color-caution` `--weight-semibold`.

**On mobile:** the three metrics stack vertically (each on its own line) rather than inline with separators.

### 4.6 Drill-down section

**Purpose**: The evidence layer for traders who want to verify the brief's synthesis. Collapsed by default. One click to expand.

**Anatomy when collapsed:**
```
[ + Show detail ]
```
A text button, `--font-ui`, `--text-sm`, `--weight-medium`, `--color-text-muted`. On hover (desktop only): `--color-text-secondary`. The `+` becomes `-` when expanded.

**Anatomy when expanded (four sub-sections):**

```
OVERNIGHT CHARACTER
  Open:     20,340    Range:    205 pts
  High:     20,460    Avg ONR:  180 pts (20-day)
  Low:      20,255    vs avg:   +14%  (slightly elevated)
  Trend:    choppy (net 78 pts in 14 hours)

FULL NUMBERS
  Prior settlement:  20,340
  PDH:               20,490
  PDL:               20,201
  Weekly H/L:        -- (not within 1 ATR)
  Monthly H/L:       -- (not within 1 ATR)

CATALYST DETAIL
  No events today.

SNAPSHOT INFO
  Data as of: 07:04:22 ET  (06:34 ingest + 07:00 render)
  Source: Databento GLBX.MDP3  NQ.c.0
  ATR basis: 14-day RTH
  ONR basis: 20-day prior sessions
  [ Report a data issue ]
```

- Container: `--color-bg-inset` background, `--radius-md`, `--space-4` padding, `--shadow-inset` top edge.
- Sub-section labels: `--font-ui`, `--text-xs`, `--weight-semibold`, `--color-text-muted`, `--tracking-widest`, uppercase.
- Labels within sub-sections: `--font-ui`, `--text-sm`, `--color-text-muted`.
- Values: `--font-mono`, `--text-sm`, `--color-text-primary`.
- Suppressed levels (weekly, monthly when not within ATR) display "--" with `--color-text-muted`. They are shown here to confirm they were computed, just suppressed from the main view.

**HTMX expand/collapse implementation (Alpine.js):**

```html
<div x-data="{ open: false }">
  <button @click="open = !open"
          class="drill-toggle"
          :aria-expanded="open">
    <span x-text="open ? '- Hide detail' : '+ Show detail'"></span>
  </button>
  <div x-show="open"
       x-transition:enter="transition ease-decel duration-200"
       x-transition:enter-start="opacity-0 translate-y-1"
       x-transition:enter-end="opacity-100 translate-y-0"
       x-transition:leave="transition ease-accel duration-120"
       x-transition:leave-start="opacity-100"
       x-transition:leave-end="opacity-0"
       class="drill-panel">
    <!-- drill-down content here -->
  </div>
</div>
```

The drill-down content is server-rendered inside `brief_fragment.html` and is present in the DOM when the page loads. Alpine only toggles visibility. No additional network request is needed for the drill-down.

---

## 5. Micro-interactions (HTMX + Alpine only)

### 5.1 Poll-to-refresh during pre-market window

The brief is a daily snapshot computed at 07:00 ET. However, the 06:30 ingest may still be running when a user loads the page before 07:00. Between 06:00 and 07:30 ET, the dashboard polls for a refreshed brief fragment.

```html
<!-- In dashboard.html, the fragment container -->
<div id="brief-fragment"
     hx-get="/api/brief/fragment"
     hx-trigger="load, every 90s [window.premarketActive]"
     hx-target="#brief-fragment"
     hx-swap="outerHTML">
  <!-- Initial server-rendered content here -->
</div>

<!-- Alpine controls the poll window -->
<script>
  window.premarketActive = (() => {
    const now = new Date();
    const etHour = /* compute ET hour server-side and inject */;
    return etHour >= 6 && etHour < 8;
  })();
</script>
```

The poll interval is 90 seconds, not shorter. Faster polling would imply that the brief content changes frequently, which it does not. The brief is a snapshot; the poll exists only to catch the case where the 06:30 ingest completes after the page was already open.

**On refresh**: the entire `brief-fragment` div is replaced silently. No flash, no loading spinner, no animation. The timestamp in the fragment header updates, which is the only visible signal that a refresh occurred.

**After 07:30 ET**: the poll stops. The timestamp shows "07:00 ET snapshot" and no further updates occur. The brief does not refresh intraday; it is a morning pre-flight, not a live terminal.

### 5.2 Stale snapshot indicator

If the timestamp on the returned fragment is older than 120 minutes (e.g., the 07:00 job failed), a subtle stale indicator appears in the page header: a small orange dot next to the timestamp, with title text "Brief may be stale. Last updated: HH:MM ET." This is rendered server-side by checking `brief_snapshots.brief_date` against the current time.

```html
<span class="timestamp">
  {{ brief.rendered_at_et }}
  {% if brief.is_stale %}
    <span class="stale-dot" title="Brief may be stale. Last updated: {{ brief.rendered_at_et }}">
      &#9679;
    </span>
  {% endif %}
</span>
```

### 5.3 Loading / no-data states

When the brief fragment is loading (on initial page load, before the first HTMX response), a minimal skeleton is shown. The skeleton is pure CSS (no JavaScript animation):

```css
.skeleton-line {
  background: linear-gradient(90deg, var(--color-bg-surface) 0%,
              var(--color-bg-elevated) 50%, var(--color-bg-surface) 100%);
  background-size: 200% 100%;
  animation: skeleton-shimmer 1.5s ease-in-out infinite;
  border-radius: var(--radius-sm);
  height: 1em;
}
@media (prefers-reduced-motion: reduce) {
  .skeleton-line { animation: none; background: var(--color-bg-elevated); }
}
```

The skeleton mirrors the layout of the five blocks (badge height, headline height, catalyst banner height, ladder height, volatility line height) so the page does not reflow on content load.

**No-data / CME holiday state**: the brief engine renders a specific state. The badge reads "NO SESSION" in `--color-neutral`. The headline reads "CME holiday today. No brief." The ladder and volatility blocks are replaced with a single centered line: "No trading session." Catalyst banner reads "No session, no events."

### 5.4 What to deliberately NOT animate

- The Risk Mode badge color change. Always `--duration-instant`.
- Number values on refresh. Silent replacement.
- The current price marker on the ladder. Static SVG; no pulse, no blinking cursor.
- Page-load transitions. No fade-in of the brief container.
- The skeleton shimmer on `prefers-reduced-motion: reduce`.
- Any element that encodes urgency through motion (prohibited: pulsing red badge, flashing catalyst times, animated price ticker).

---

## 6. Email variant

### 6.1 What is kept vs simplified

The email is a read-only push artifact. It is opened in Gmail, Apple Mail, or Outlook mobile. It has no JavaScript, no CSS variables, no Alpine, no HTMX. It uses inlined CSS and a table-based layout.

| Element | Web dashboard | Email variant |
|---|---|---|
| Risk Mode badge | Styled div with CSS var colors | Table cell, inlined background-color hex |
| Headline line | Inline text block | Table cell, inlined font/color |
| Catalyst banner | Div card with border | Table cell with left border, inlined |
| Price ladder | Inline SVG | Plain text table (see below) |
| Volatility line | Div text block | Table cell |
| Drill-down expand | Alpine toggle | Not included; replaced by full text |
| Poll-to-refresh | HTMX | Not applicable |
| Dark mode | CSS media / vars | Supported via meta tag + media (limited) |

### 6.2 Price ladder in email

SVG is not safe in most email clients. The price ladder in email is rendered as a preformatted HTML table, using ASCII-style alignment. The brief engine has a separate `format_ladder_email()` function (in `brief/levels.py`) that produces this table.

```html
<table cellpadding="4" cellspacing="0"
       style="font-family: 'Courier New', monospace; font-size: 13px;
              color: #e8eaed; border-collapse: collapse; width: 100%;">
  <tr>
    <td style="color: #8b9199; width: 80px;">ONH</td>
    <td style="color: #e8eaed; width: 90px; text-align: right;">20,460</td>
    <td style="color: #8b9199;">+42 pts / 1.0x ATR above</td>
  </tr>
  <tr>
    <td colspan="3" style="padding: 6px 0;">
      <div style="background-color: #0c1a26; border-left: 3px solid #38bdf8;
                  padding: 4px 8px; color: #38bdf8; font-weight: bold;">
        NOW  20,418
      </div>
    </td>
  </tr>
  <tr>
    <td style="color: #e8eaed; font-weight: bold; width: 80px;">* PDH *</td>
    <td style="color: #e8eaed; font-weight: bold; width: 90px; text-align: right;">20,405</td>
    <td style="color: #8b9199;">-13 pts / 0.3x ATR (nearest below)</td>
  </tr>
  <!-- ... remaining levels -->
</table>
```

### 6.3 Table layout structure for email

```html
<!-- email.html Jinja2 template structure -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta name="color-scheme" content="dark light">
  <meta name="supported-color-schemes" content="dark light">
  <title>NQ Pre-Market Brief {{ brief.date_et }}</title>
  <style>
    /* Inlined by the email sender; this block is for reference only */
    @media (prefers-color-scheme: dark) {
      body { background-color: #0d0f11 !important; color: #e8eaed !important; }
      .email-surface { background-color: #141719 !important; }
    }
  </style>
</head>
<body style="margin:0; padding:0; background-color:#0d0f11; font-family:'Helvetica Neue',Arial,sans-serif;">
  <table width="100%" cellpadding="0" cellspacing="0">
    <tr>
      <td align="center" style="padding: 16px 8px;">
        <table width="600" cellpadding="0" cellspacing="0"
               style="max-width:600px; background-color:#141719; border-radius:8px;">

          <!-- Header row -->
          <tr>
            <td style="padding:16px 24px; border-bottom:1px solid #272b2f;">
              <span style="color:#8b9199; font-size:12px;">NQ / MNQ</span>
              <span style="color:#545b63; font-size:12px; float:right;">07:00 ET snapshot</span>
            </td>
          </tr>

          <!-- Risk Mode badge row -->
          <tr>
            <td style="padding:16px 24px;">
              <!-- background-color and color are set by template logic, not CSS vars -->
              <div style="background-color:{{ badge_bg_hex }};
                          border-left:4px solid {{ badge_color_hex }};
                          padding:12px 16px; border-radius:6px;">
                <div style="color:{{ badge_color_hex }}; font-size:22px;
                            font-weight:bold; letter-spacing:0.04em;">
                  {{ brief.risk_mode_label }}
                </div>
                <div style="color:#8b9199; font-size:14px; margin-top:4px;">
                  {{ brief.risk_mode_sub }}
                </div>
              </div>
            </td>
          </tr>

          <!-- Headline row -->
          <!-- Catalyst row -->
          <!-- Ladder row -->
          <!-- Volatility row -->

          <!-- Footer row -->
          <tr>
            <td style="padding:12px 24px; border-top:1px solid #272b2f;">
              <span style="color:#545b63; font-size:11px;">
                Snapshot: {{ brief.rendered_at_et }} ET &bull;
                Databento GLBX.MDP3 &bull;
                premarket.home for full detail
              </span>
            </td>
          </tr>

        </table>
      </td>
    </tr>
  </table>
</body>
</html>
```

### 6.4 CSS inlining

The `email_sender/send.py` module uses `premailer` (or equivalent) to inline all CSS into style attributes before sending. The `email.html` template is written with inline styles as the primary styling approach (not class-based), so the inlining step is a safety net rather than the primary mechanism.

### 6.5 Dark-mode email caveats

- Apple Mail on iOS/macOS: respects `@media (prefers-color-scheme: dark)` and `<meta name="color-scheme">`. The dark theme will work.
- Gmail (web + app): ignores most `@media` queries and will likely render the light fallback. The light fallback should be tolerable (near-black background degrades to white with dark text). Set explicit `background-color` on the `body` and all table cells; Gmail sometimes overrides if these are absent.
- Outlook (Windows): renders using Word's HTML engine. Background colors on `<div>` are dropped; use `bgcolor` attribute on `<td>` elements for critical fills. The badge fill will degrade gracefully as long as the text color is still readable.
- The semantic green/red/gray color system must still communicate without background fills as a fallback. This is why the mode text is all-caps and uses the bold-weight label ("STAND ASIDE" reads clearly even if the badge background is white). Non-color encoding is not optional in email.

---

## 7. Accessibility

### 7.1 Contrast targets

All text meets WCAG 2.1 AA minimum. Critical information text meets AAA.

| Element | Foreground | Background | Ratio | Target |
|---|---|---|---|---|
| Body text | #e8eaed | #141719 | 12.8:1 | AAA (7:1) |
| Secondary text | #8b9199 | #0d0f11 | 4.6:1 | AA (4.5:1) |
| Green badge text | #22c55e | #052e16 | 6.2:1 | AA |
| Red badge text | #ef4444 | #2d0a0a | 5.8:1 | AA |
| Yellow badge text | #eab308 | #1c1500 | 8.4:1 | AAA |
| Accent (current price) | #38bdf8 | #0c1a26 | 5.9:1 | AA |
| Muted text | #545b63 | #0d0f11 | 3.4:1 | AA large text only |

The muted text (`--color-text-muted`) is used exclusively for non-critical metadata (timestamps, footnotes). Never for content that affects a trading decision.

### 7.2 Non-color encoding

Color is never the sole encoding. Every semantic color state has a paired non-color signal:

| State | Color | Icon/shape | Text |
|---|---|---|---|
| GREEN / positive | Green | Filled circle | "TRADE NORMAL SIZE" |
| YELLOW / caution | Yellow | Triangle/warning | "TRADE SMALLER" |
| RED / negative | Red | Octagon/stop | "STAND ASIDE" |
| Gap up | Green | ▲ glyph | "+78 pts (+0.4%)" |
| Gap down | Red | ▼ glyph | "-42 pts (-0.2%)" |
| Inconclusive | Gray | ~ glyph | text + "(non-event)" |
| Nearest level | Bold | Asterisk (*) | "(nearest below)" |
| Stale data | Orange dot | Dot glyph | "may be stale" tooltip |

The price ladder SVG has an `aria-label` attribute describing the current price relationship to the nearest levels in plain text: `aria-label="Current price 20,418. Nearest resistance: ONH at 20,460 (+42 pts). Nearest support: PDH at 20,405 (-13 pts)."` This is computed server-side.

### 7.3 Font sizes

No functional text appears below 11px (`--text-xs`). Body text is 14px (`--text-base`). The page `<html>` element sets `font-size: 100%` to respect user browser font-size preferences. All internal sizes use `rem` units, not `px`, so a user who has set their browser default to 18px will see proportionally larger text throughout.

### 7.4 Focus states

Keyboard navigation is supported. All interactive elements have explicit focus rings:

```css
:focus-visible {
  outline: 2px solid var(--color-focus-ring);
  outline-offset: 2px;
  border-radius: var(--radius-sm);
}
```

The `drill-toggle` button and any `<a>` links in the drill-down footer have visible focus states. `:focus` (not `:focus-visible`) is used for email-rendered content where browser support is variable.

### 7.5 Motion and vestibular

All animations respect `prefers-reduced-motion`:

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

The skeleton shimmer is replaced with a static fill under reduced motion. The Alpine drill-down transition degrades to an instant toggle.

### 7.6 Semantic HTML

- The brief container is a `<main>` element with `aria-label="Pre-market trading brief"`.
- The Risk Mode badge is an `<aside role="status">` so screen readers announce it on load.
- The price ladder SVG has `role="img"` and a full `aria-label` (see 7.2).
- Section headers are `<h2>` elements (visually styled as small caps labels via CSS, not by reducing their semantic weight).
- The drill-down toggle uses `aria-expanded` (as shown in section 4.6).
- The catalyst event list is a `<ul>` with each event as a `<li>`.

---

## 8. Implementation notes: mapping to architecture and build plan

### 8.1 File mapping

From the architecture doc (`src/templates/`):

| Template file | Contents |
|---|---|
| `base.html` | HTML shell, `<head>` with CSS tokens (as a `<style>` block in the `<head>` or a vendored `premarket.css`), Inter + JetBrains Mono font loading (local files under `static/fonts/`), Alpine.js (vendored, loaded from `static/js/alpine.min.js`), HTMX (vendored, loaded from `static/js/htmx.min.js`). Dark background on `<body>`. |
| `dashboard.html` | Extends `base.html`. Page header (instrument, date, timestamp). The `<div id="brief-fragment">` with the HTMX polling attributes. The Alpine poll-control script. |
| `brief_fragment.html` | The five blocks (badge, headline, catalyst, ladder, volatility). The drill-down section. The stale indicator logic. The inline SVG for the price ladder (generated by the brief engine). This is the HTMX partial. |
| `email.html` | Standalone (no `base.html` extension). Table-based layout. Inlined CSS. No JS references. The `format_ladder_email()` text table. |

### 8.2 Static assets

```
app/
  static/
    css/
      premarket.css     -- design tokens + component styles
    js/
      htmx.min.js       -- vendored, v1.9+ recommended
      alpine.min.js     -- vendored, v3.x
    fonts/
      Inter-Regular.woff2
      Inter-Medium.woff2
      Inter-SemiBold.woff2
      Inter-Bold.woff2
      JetBrainsMono-Regular.woff2
      JetBrainsMono-SemiBold.woff2
```

Fonts are served locally (FastAPI `StaticFiles` mount at `/static`). No external font CDN calls. This is a self-hosted service on a LAN; keeping dependencies local prevents a 07:00 failure if the internet is briefly unavailable.

### 8.3 Brief engine outputs for rendering

The `brief/engine.py` `build_brief()` function returns a `BriefSnapshot` pydantic model. The fields used by the templates include:

| Field | Type | Used by |
|---|---|---|
| `risk_mode` | `Literal["GREEN","YELLOW","RED"]` | badge, email badge |
| `risk_mode_label` | `str` | badge text |
| `risk_mode_sub` | `str` | badge sub-line |
| `headline_text` | `str` | headline line |
| `headline_color` | `Literal["positive","negative","neutral"]` | headline CSS class |
| `gap_pts` | `int` | headline |
| `gap_pct` | `float` | headline |
| `gap_as_atr_frac` | `float` | headline qualifier, volatility line |
| `catalysts` | `list[CatalystEvent]` | catalyst banner |
| `levels_svg` | `str` | SVG string for web ladder |
| `levels_email_table` | `str` | HTML string for email ladder |
| `atr_14` | `float` | volatility line |
| `vix_prior_close` | `float` | volatility line |
| `expected_move_pct` | `float` | volatility line |
| `vol_regime_label` | `Literal["normal","elevated","high"]` | volatility line |
| `sizing_text` | `str` | volatility line (may be "not configured") |
| `drill_overnight` | `OvernightDetail` | drill-down |
| `drill_full_levels` | `FullLevelsDetail` | drill-down |
| `rendered_at_et` | `str` | timestamp |
| `is_stale` | `bool` | stale indicator |
| `is_holiday` | `bool` | no-session state |

### 8.4 Agent responsibilities for Phase 4

Per the build plan's Phase 4 assignment:

- **ui-designer agent** (this document): owns the design system, design tokens, component specifications, and visual logic. Outputs this specification document. Does not write code.
- **frontend-developer agent**: implements `premarket.css` from the token definitions in this document, builds `base.html`, `dashboard.html`, `brief_fragment.html`, and `email.html` from the wireframes and component specs here, and wires the HTMX/Alpine interactions described in section 5. The frontend-developer agent should treat this document as the source of truth for visual decisions and escalate any ambiguity about visual behavior back to this spec rather than making ad-hoc choices.

The SVG generation logic in `brief/levels.py` sits at the boundary: the frontend-developer agent implements the SVG template structure, and the backend-developer agent implements the data computation that feeds the SVG coordinates. The coordinate mapping formula is specified in section 4.4 of this document.

### 8.5 CSS architecture

All design tokens are declared on `:root` in `premarket.css`. Component styles use the token names, not raw hex values. This means:

- A single token change (e.g., adjusting `--color-positive` for better contrast on a new display) propagates everywhere automatically.
- The email template uses raw hex values (the token values, copied out), since CSS variables do not work in email clients. When a token value is changed, the corresponding email hex values in `email.html` must be updated manually (or by a simple find-replace). The frontend-developer agent should note which hex values in `email.html` correspond to which tokens.

One practical approach: the brief engine's `send.py` resolves token hex values from a Python `DESIGN_TOKENS` dict (which mirrors the CSS token file) and injects them into the email template at send time. This keeps a single source of truth.

```python
# In email_sender/send.py or brief/engine.py
DESIGN_TOKENS = {
    "badge_bg_green":    "#052e16",
    "badge_color_green": "#22c55e",
    "badge_bg_yellow":   "#1c1500",
    "badge_color_yellow":"#eab308",
    "badge_bg_red":      "#2d0a0a",
    "badge_color_red":   "#ef4444",
    "bg_surface":        "#141719",
    "text_primary":      "#e8eaed",
    "text_secondary":    "#8b9199",
    "text_muted":        "#545b63",
    "accent":            "#38bdf8",
    "accent_bg":         "#0c1a26",
    "bg_border":         "#272b2f",
}
```

---

## Appendix A: complete design token quick-reference

For the frontend-developer agent, a consolidated single-block reference:

```css
:root {
  /* Backgrounds */
  --color-bg-base:       #0d0f11;
  --color-bg-surface:    #141719;
  --color-bg-elevated:   #1c1f22;
  --color-bg-inset:      #0a0c0e;
  --color-bg-border:     #272b2f;
  /* Text */
  --color-text-primary:  #e8eaed;
  --color-text-secondary:#8b9199;
  --color-text-muted:    #545b63;
  --color-text-inverse:  #0d0f11;
  /* Semantic */
  --color-positive:      #22c55e;
  --color-positive-dim:  #15803d;
  --color-positive-bg:   #052e16;
  --color-negative:      #ef4444;
  --color-negative-dim:  #b91c1c;
  --color-negative-bg:   #2d0a0a;
  --color-neutral:       #71717a;
  --color-neutral-dim:   #3f3f46;
  --color-neutral-bg:    #18181b;
  --color-caution:       #eab308;
  --color-caution-dim:   #a16207;
  --color-caution-bg:    #1c1500;
  /* Accent */
  --color-accent:        #38bdf8;
  --color-accent-dim:    #0369a1;
  --color-accent-bg:     #0c1a26;
  --color-focus-ring:    #38bdf8;
  /* Typography */
  --font-ui:   'Inter', 'SF Pro Text', system-ui, -apple-system, sans-serif;
  --font-mono: 'JetBrains Mono', 'Fira Code', 'Cascadia Code', 'Menlo', monospace;
  --text-xs:   0.6875rem;
  --text-sm:   0.75rem;
  --text-base: 0.875rem;
  --text-md:   1rem;
  --text-lg:   1.125rem;
  --text-xl:   1.375rem;
  --text-2xl:  1.75rem;
  --weight-regular: 400;
  --weight-medium:  500;
  --weight-semibold:600;
  --weight-bold:    700;
  --leading-tight:  1.2;
  --leading-snug:   1.375;
  --leading-normal: 1.5;
  --tracking-wide:  0.04em;
  --tracking-widest:0.08em;
  /* Spacing */
  --space-1: 4px;  --space-2: 8px;   --space-3: 12px;
  --space-4: 16px; --space-5: 20px;  --space-6: 24px;
  --space-8: 32px; --space-10: 40px; --space-12: 48px;
  /* Radii */
  --radius-sm:  4px;
  --radius-md:  8px;
  --radius-lg: 12px;
  --radius-pill: 999px;
  /* Shadows */
  --shadow-inset:  inset 0 1px 0 0 rgba(255,255,255,0.04);
  --shadow-marker: 0 0 0 2px var(--color-bg-elevated), 0 0 0 3px var(--color-accent);
  --shadow-card:   0 1px 3px 0 rgba(0,0,0,0.6), 0 1px 2px -1px rgba(0,0,0,0.6);
  /* Motion */
  --duration-instant: 0ms;
  --duration-fast:   120ms;
  --duration-normal: 200ms;
  --duration-slow:   300ms;
  --ease-standard:   cubic-bezier(0.4, 0, 0.2, 1);
  --ease-decel:      cubic-bezier(0, 0, 0.2, 1);
  --ease-accel:      cubic-bezier(0.4, 0, 1, 1);
}
```

---

*Document authored by ui-designer agent. Implementation by frontend-developer agent. Review with Bryce before Phase 4 sprint begins.*
