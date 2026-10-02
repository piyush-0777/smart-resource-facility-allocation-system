# Design — WorkFlow

## 1. Design Philosophy

**"Liquid glass on a canvas."** The interface should feel like frosted Apple panels floating
over a soft, slowly-shifting colour mesh — depth through translucency and blur, never
through heavy borders or drop shadows. Every surface is a *layer*: content sits on glass,
glass sits on a gradient field.

Three rules:

1. **Transparency carries hierarchy.** The topmost surface is the most opaque; nested
   surfaces get lighter, not darker.
2. **Blur everywhere, borders nowhere.** Separation comes from a 1 px inner highlight, not
   a solid outline.
3. **Motion is physical.** Springs, 150–400 ms, `cubic-bezier(0.32, 0.72, 0, 1)` — the
   Apple "sheet" easing. Nothing pops instantly.

Tone: calm, precise, generous whitespace. This is an operations tool used by facilities
staff under time pressure — legibility outranks decoration.

## 2. Colour System

Semantic tokens only. Components never reference raw hex.

| Token | Light | Dark | Use |
| --- | --- | --- | --- |
| `--bg-base` | `#f2f2f7` | `#08080c` | Page background |
| `--bg-mesh-a` | `#a8c0ff` | `#2b3a8f` | Gradient blob 1 |
| `--bg-mesh-b` | `#ffc2d6` | `#7a2f6b` | Gradient blob 2 |
| `--bg-mesh-c` | `#a8f0d8` | `#1f6f6a` | Gradient blob 3 |
| `--glass-bg` | `rgb(255 255 255 / 0.55)` | `rgb(255 255 255 / 0.07)` | Panel fill |
| `--glass-bg-strong` | `rgb(255 255 255 / 0.72)` | `rgb(255 255 255 / 0.12)` | Modal/sidebar |
| `--glass-blur` | `28px` | `36px` | Backdrop blur radius |
| `--glass-border` | `rgb(255 255 255 / 0.7)` | `rgb(255 255 255 / 0.14)` | 1 px inner highlight |
| `--text-primary` | `#0b0b12` | `#f5f5f7` | Headings, body |
| `--text-secondary` | `#5b5b6b` | `#a0a0b0` | Meta, captions |
| `--text-tertiary` | `#8e8e9a` | `#6e6e80` | Disabled, hints |
| `--accent` | `#0a84ff` | `#0a84ff` | Primary actions, links |
| `--accent-hover` | `#0060df` | `#3d9bff` | Hover state |
| `--success` | `#1f9d55` | `#30d17a` | Approved / available |
| `--warning` | `#c47a08` | `#ffb020` | Pending / due soon |
| `--danger` | `#d92d3f` | `#ff5a63` | Rejected / overdue / destructive |
| `--info` | `#0a84ff` | `#64a8ff` | Informational badges |

Backdrop mesh: three radial gradients (blob colours above) positioned at 20 % / 80 % / 50 %
with `filter: blur(120px)` and a slow 30 s drift animation. Disabled under
`prefers-reduced-motion`.

Dark mode is not an inversion — surfaces go *darker and more transparent*, blur increases,
text stays at 95 % white, and saturation of the mesh drops so content remains the focus.

## 3. Glass Primitives (Tailwind)

```css
/* tailwind.css */
@layer components {
  .glass {
    background: var(--glass-bg);
    -webkit-backdrop-filter: blur(var(--glass-blur)) saturate(180%);
            backdrop-filter: blur(var(--glass-blur)) saturate(180%);
    border: 1px solid var(--glass-border);
    box-shadow:
      inset 0 1px 0 0 rgb(255 255 255 / 0.35),   /* top highlight  */
      0 8px 32px rgb(0 0 0 / 0.08);             /* ambient lift   */
  }
  .glass-strong { background: var(--glass-bg-strong); }
  .glass-interactive {
    @apply glass transition-transform duration-200 ease-[cubic-bezier(.32,.72,0,1)];
    will-change: transform;
  }
  .glass-interactive:hover  { transform: translateY(-2px) scale(1.005); }
  .glass-interactive:active { transform: scale(0.985); }
  .glass-sheen { position: relative; overflow: hidden; }
  .glass-sheen::after {           /* diagonal specular sweep on hover */
    content: ""; position: absolute; inset: 0;
    background: linear-gradient(115deg, transparent 40%,
                rgb(255 255 255 / 0.22) 50%, transparent 60%);
    transform: translateX(-100%); transition: transform .6s ease;
  }
  .glass-sheen:hover::after { transform: translateX(100%); }
}
```

Dark mode adds a subtler top highlight (`rgb(255 255 255 / 0.12)`) — the same CSS, one
`.dark` class on `<html>` swapping custom properties. Tailwind `darkMode: 'class'`.

## 4. Component Inventory

| Component | Notes |
| --- | --- |
| `GlassPanel` | Base surface. Props: `blur`, `padding`, `sheen`, `interactive`, `as` |
| `GlassButton` | Variants `primary` (accent fill, 92 % opacity), `glass`, `ghost`, `danger`; sizes `sm/md/lg`; loading spinner state |
| `GlassInput` / `GlassSelect` / `GlassDatePicker` | Translucent fill, focus ring = 2 px accent at 40 % + 12 px accent-tinted glow |
| `GlassModal` | Sheet slides up from bottom on mobile, scales in from centre on desktop; scrim = `rgb(0 0 0 / 0.35)` + blur 8 px |
| `GlassSidebar` | Fixed 260 px; collapsed rail at `< lg` with lucide icon-only tooltips |
| `GlassCard` (asset) | 4:3 image, glass body, status pill, hover lift 2 px |
| `StatusBadge` | `available`→success, `booked`→info, `maintenance`→warning, `approved`→success, `pending`→warning, `rejected`/`cancelled`→neutral, `active`→accent, `completed`→neutral, `overdue`→danger. Pill = glass with 1 dot indicator |
| `MetricCard` | Glass, big tabular-nums value, lucide icon in a 36 px glass tile, delta chip |
| `Timeline` | Vertical glass rail of bookings with 8 px status dots |
| `Skeleton` | Shimmering glass blocks (no spinner spinners) |
| `EmptyState` | Lucide icon in glass circle + one-line hint + primary action |
| `Toast` | Top-right stack, glass-strong, auto-dismiss 4 s, pause on hover |
| `ConfirmDialog` | For destructive/irreversible actions (reject, cancel, maintenance) |

Icon set is exclusively **lucide-react**, 20 px default / 1.75 stroke, `aria-hidden` on
decorative icons.

## 5. Layout

**App shell**
```
┌──────────┬────────────────────────────────────────┐
│ Sidebar  │ Topbar  (page title · search · 🔔 · ☾/☀ · avatar)
│ (glass)  ├────────────────────────────────────────┤
│          │  <Outlet/>  — content on translucent cards
│          │  floating "+ Book" FAB (bottom-right, mobile)
└──────────┴────────────────────────────────────────┘
```

- Max content width 1200 px, 24 px gutters (16 px on mobile).
- 8 px spacing scale; 12/16/24/32 section rhythm.
- Radius: panels 24 px, buttons 14 px, inputs 12 px, pills 999 px.
- Grid: asset catalog 1 / 2 / 3 columns at `sm / md / lg`.

## 6. Theming (Light / Dark)

- `ThemeProvider` at app root; initial value = `localStorage.theme` ?? `prefers-color-scheme`.
- Toggle is a 3-state switch (`light → dark → system`), persisted, and applied as
  `.dark` on `<html>` before first paint via an inline script in `index.html` (no FOUC).
- Contrast targets: body text ≥ 4.5:1 in both themes; status pills pair colour with a
  **shape/dot**, never colour alone.
- `prefers-reduced-motion: reduce` disables mesh drift, sheen sweep, and slide-ins.

## 7. UX Flows

### 7.1 First run
Login → role-based redirect (`user → /catalog`, `admin → /admin`). Admin sees a one-time
3-step tour of the dashboard; users see a dismissible hint pointing at the catalog.

### 7.2 Booking wizard (user)
Step 1 pick asset → step 2 pick window (calendar with busy blocks rendered as hatched
glass blocks) → step 3 purpose + review → submit.

- Live overlap feedback: while dragging/selecting times, the client hits
  `GET /assets/:id/availability` and paints conflicting blocks in `--danger` at 30 %.
  *The client hint is cosmetic only — the server still rejects authoritatively.*
- Success: glass toast + optimistic Redux insert into `bookings.mine`.
- Conflict (`409`): modal explaining the blocking window with a "pick another time" action.

### 7.3 Approval (admin)
Pending queue as glass rows. Row expands (height animation) to show requester, asset,
window, purpose, and any conflicts. Approve / Reject inline; Reject requires a reason in a
glass confirm dialog. On approve, the row springs out and the Pending metric animates down.

### 7.4 Real-time notification
Toast slides in from the top-right (spring), lucide `BellRing` icon, accent glass tint,
4 s auto-dismiss, click → navigate to the booking. Sidebar bell shows an unread dot.

### 7.5 Loading & empty states
Skeletons matching final layout (no layout shift). Every empty collection gets an
`EmptyState` with one clear next action.

### 7.6 Error handling
- Field errors: red hairline + message under the input.
- Form-level: glass banner at the top of the card.
- 401 → redirect to login preserving the intended route.
- 403 → glass "restricted" modal, then redirect to `/catalog`.
- Offline/timeout: non-blocking retry toast.

## 8. Accessibility

- All interactive glass elements meet 3:1 against their backdrop; focus is never
  conveyed by blur alone — a 2 px accent ring + offset is always present.
- Full keyboard traversal; modals trap focus and restore it on close; `Esc` closes.
- Semantic landmarks (`nav`, `main`, `aside`), `aria-live="polite"` for toasts.
- Colour-blind safe status encoding (dot + label).
- Minimum body text 14 px (16 px for forms), `line-height: 1.55`.

## 9. Performance Budgets

| Metric | Target |
| --- | --- |
| Backdrop-filter cost | Blur ≤ 36 px, capped at 3 simultaneous large glass layers (avoid stacking blur-on-blur) |
| Initial JS (gzip) | < 220 KB (React + Redux + Router + Lucide tree-shaken) |
| LCP | < 2.0 s on throttled 4G |
| Motion | All transitions ≤ 400 ms, GPU-composited (`transform`/`opacity` only) |
| Mesh animation | `transform` only, `will-change: transform`, paused when tab hidden |

## 10. Do / Don't

**Do:** layer glass (panel on panel), keep blur high and borders invisible, animate with
springs, use generous whitespace, keep the mesh subtle.

**Don't:** place high-contrast text directly on the mesh (no glass panel behind it),
stack 4+ blurred layers, use heavy drop shadows, put body copy inside a pill, or use blur
on scrolling lists larger than a viewport.