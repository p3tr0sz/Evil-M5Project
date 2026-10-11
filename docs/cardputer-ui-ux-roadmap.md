# Cardputer UI/UX Roadmap

Working notes for the Evil-Cardputer firmware menu UX, tracked on branch
`p3tr0sz/ux-experience`. All changes live in `Evil-Cardputer-v1-5-6.ino`
unless noted.

- **Device:** M5Cardputer, ESP32-S3, 240×135 IPS, M5GFX / M5Unified.
- **Menu font:** built-in font 1 (GLCD 6×8), scaled by a float text size.
- **Design system reference (live mockups + token tables):**
  https://claude.ai/artifact/F2Q4sRjQV4F6Eu1fqt85mC

The design-system page is the visual source of truth for colors, the type
scale, the layout grid, and the three overflow behaviours described below.

---

## Done

| Change | Where | Commit |
|---|---|---|
| Clip long labels instead of wrapping (fixes row overlap) | `drawMenu()` — `setTextWrap(false)` | `bc90b1a` |
| Focused-label marquee + ".." ellipsis on idle rows | `drawMenuLabel()`, `loop()` tick | `24c9baf` |
| Scroll-position rail on the right edge | `drawMenu()` | `dd6cf96` |
| Persistent zoom level (`menuZoomLevel`) across reboots | `cycleMenuZoom()`, `restoreConfigParameter()`, `setup()`, `config.txt` | `dd6cf96` |
| Focus-edge accent (2px stripe on selected row) | `drawMenu()` | `dd6cf96` |

**Marquee details:** `drawMenuLabel()` scrolls only the focused row while it
overflows; `loop()` repaints the menu on a ~40 ms tick gated by
`menuFocusMarquee`, so idle menus stay at 0% redraw.

---

## Next steps

Ordered by value-to-effort. None of these are blocked on each other.

### C — Category separators (highest UX value, medium risk)

Split the flat `menuItems[]` into labelled sections (e.g. WiFi / Portal /
Probe & Karma / Sniff / BLE / RF / Tools / System) so the 50+ item list is
scannable.

**Why it's not a one-liner:** `menuItems[]` is a flat array and
`executeMenuItem(int index)` is a position-keyed `switch`. Inserting
selectable rows would shift every index and break the switch. The view layer
(`viewCount()`, `mapViewToRealIndex()`, `menuStartIndex`, `currentIndex`,
`clampMenuSelection()`) also assumes **1 visible row = 1 selectable item**.

**Safe approach — keep `executeMenuItem()` indices untouched.** Add a separate
header table, e.g.:

```cpp
struct MenuSection { int beforeIndex; const char* title; };
static const MenuSection menuSections[] = {
  {0,  "WIFI"}, {6, "PORTAL"}, {11, "MONITOR"}, {12, "PROBE & KARMA"},
  {20, "WARDRIVE"}, {22, "ATTACK"}, {26, "SNIFF"}, /* ... */
};
```

Two implementation options:

- **Option 1 — render-only headers (low risk, do this first).** Draw a header
  line in `drawMenu()`'s loop whenever the next real item index matches a
  section start. Selection and scroll math stay in item-space; headers are
  purely visual. Downside: a header consumes a row's height, so fewer items
  show, and a header can scroll partly off the top. Good enough and cheap.
- **Option 2 — headers as skipped view rows (cleaner, more work).** Extend the
  view model so a view position is either a header or an item; navigation
  (up/down, `clampMenuSelection`, `menuStartIndex`) skips headers;
  `maxMenuDisplay` accounts for header rows. Nicer scrolling, but touches all
  the view math — test wrap-around and search mode carefully.

**Decision needed:** confirm the section grouping (titles + which menu index
each starts at). Draft above is a guess from the current order.

### Perf — off-screen sprite for the menu (removes marquee flicker)

The menu draws directly to the panel: `drawMenu()` clears the list region then
repaints. At the marquee's ~25 fps the selected bar can shimmer on some panels.

**Fix:** render the list into an `LGFX_Sprite` (≈ 240×122×2 ≈ 58 KB, fits in
the Cardputer's PSRAM) and `pushSprite()` once per frame. Eliminates the
clear-then-redraw flash and cuts SPI traffic.
**Risk:** sprite lifecycle / memory; allocate once (like `taskBarCanvas`),
never per-frame. Do this only if flicker is actually visible on hardware.

### E — Semantic color contract (low risk, incremental)

Replace scattered inline `TFT_GREEN` / `TFT_RED` / `TFT_YELLOW` status literals
with three named, theme-overridable globals:

```cpp
int statusOkColor   = TFT_GREEN;   // On / valid / link up / battery >= 70%
int statusBadColor  = TFT_RED;     // Off / fail / no IP / battery < 40%
int statusWarnColor = TFT_YELLOW;  // mid battery, search prompt
```

Default them next to the existing theme block (`// THEME START`), optionally
load from `theme.ini`, then migrate call sites incrementally — start with
`drawTaskBar()`. Prevents a theme from accidentally making "Off" unreadable.

### B — Taskbar position counter (optional, low priority)

Show `currentIndex+1 / viewCount()` (e.g. `07/58`) in the taskbar's free gap
between `C:` (x≈140) and the battery readout, in `drawTaskBar()`. Partly
redundant with the scroll rail, and the bar is already crowded — only worth it
if the rail proves too coarse in practice.

---

## Notes

- **Not built in this environment.** Changes are reviewed by hand; the ESP32
  toolchain + libraries aren't available here. Flash to a Cardputer to verify.
- Test matrix for any menu change: all three zoom levels (`;`), search mode,
  the `laika` grey theme (`theme_laika.ini`), and a menu long enough to scroll.
