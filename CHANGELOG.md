# Maintenance Tracker — Changelog

## v0.19.1 — 2026-08-28 — dark buttons + a real sun

Owner follow-ups on v0.19.0: "Buttons haven't changed, would be nicer
if they're dark" and "is there a better sun icon? this one looks like
a shuriken."

- **Sun face**: the dark-mode toggle face is now `sun-bright` (circle
  with distinct rays) — the plain FA `sun` really is an eight-spiked
  star. Owner picked it from a rendered sample sheet showing eight
  candidates as the actual white-on-indigo button.

- `btn_fill`/`btn_disabled_fill` moved into the palette (dark: muted
  indigo #4a5473 / #3a3e4a; light unchanged). At startup the aspect
  styles pick the persisted theme's colors; at toggle, `_retheme_all`
  restyles every normal-style button live through its `${tag}-btn`
  shape tag — a round dbutton's face is several ovals/rects that ALL
  carry that tag with `-fill` and `-outline`
  (de1app-core/dui.tcl:9896, 8476-8478), and `dui item config`
  itemconfigures every match (dui.tcl:7440-7452), so one bare-tag call
  recolors the whole face, disabled colors included.
- Unchanged on purpose: danger red, white labels, and the invisible
  tap zones (picker cells, card text areas) — the restyle walk lists
  visible normal buttons explicitly and can never touch a tap zone.

**Safety status**: pure UI patch — write behavior identical to
v0.19.0.

Offline verification: 246 checks — the v0.19.0 suite extended with:
the two button colors join the exactly-once literal sweep, the toggle
flips `btn_fill` both ways, the retheme walk restyles a real button
shape tag, it provably never touches an invisible tap zone, and the
face proc uses sun-bright with the plain sun name fully gone.

## v0.19.0 — 2026-08-28 — Pass 20: dark mode

Owner request: sun/moon toggle on the main page, switching instantly.

- **Palette**: every non-state color the plugin paints moved from
  scattered literals into `_apply_palette` (light and dark values,
  chosen per `settings(theme)`). The state tints recompute from the
  card color automatically (stronger blend on dark so the plates stay
  visible). The offline harness now enforces that each themed color
  literal appears exactly once in the source — a stray hardcoded color
  can never ship again.
- **Instant toggle**: the sun/moon button (top-right header corner,
  moon in light mode / sun in dark, FA glyphs with a text fallback)
  calls `toggle_theme`: palette swap → `_retheme_all` repaints every
  page's static items by bare tag (backgrounds, all text roles,
  card/picker backdrops, amber ticks, the two name entries — Tk
  widgets reconfigure through the same `dui item config` path) → the
  main page's refresh repaints its dynamic parts under your finger.
  Other pages' dynamics repaint in their own show-refresh, which runs
  on every entry.
- Deliberately identical in both themes: ok/amber/red state colors,
  and all button faces (dbutton compounds are the one thing not
  restyled at runtime — the periwinkle/red with white labels reads on
  both palettes).
- The choice persists in `settings(theme)` (default light; unknown
  values heal to light).

**Safety status**: the ONE new write is `settings(theme)` into the
plugin's own settings.tdb on each explicit toggle tap. SDB stays
read-only; history files untouched; other automatic writes exactly as
in v0.13.0.

Offline verification: 238 checks — the full v0.18.0 suite (which
exercises both palettes' shared code paths) plus: byte-compile of the
four new procs, the exactly-once literal sweep over 12 themed colors,
theme default + healing, glyph/text button faces in both modes, a
full toggle round-trip (dark palette applied, tints recomputed,
settings saved, retheme touched diagnostics rows / card backdrops /
the edit entry, light palette restored exactly), and the button's
placement above the toolbar in the top-right corner.

## v0.18.0 — 2026-08-28 — Pass 19: icon names + Detail header icon

Owner request: name the selected icon next to "Icon:", and show a
tracker's icon when opening it.

- **Icon names**: a 24-entry `icon_labels` table gives every picker
  icon a human-readable meaning (Steam wand, Drain pipe, Ball joint,
  Flat gasket, Descale, …). `_refresh_picker` rewrites the "Icon:"
  label as "Icon:  <name>" on every selection change — one existing
  item, so no new layout zone and nothing to collide with the Add
  page's validation message.
- **Detail header icon**: the Detail page shows the tracker's icon
  top-left (called: corner, not inline — the centered title would need
  text measuring) as a card-style plate: rounded backdrop tinted by
  state, glyph or vector drawing colored by state, aligned with the
  Edit button's header slot. Hidden on the nothing-selected branch.
- The card refresh's glyph-vs-vector swap was factored into a shared
  `_apply_item_icon` renderer used by both the cards and the Detail
  header (identical behavior, verified by the existing card checks).

**Safety status**: pure UI pass — no write-behavior changes; SDB
read-only, history files untouched, automatic writes exactly as in
v0.13.0.

Offline verification: 205 checks — the full v0.17.0 suite plus: every
picker icon has a display name and no stale keys linger, unknown
names fall back to the raw name, the picker refresh writes the
selected icon's name onto the label, the Detail page builds
plate/glyph/vector items (vectors born hidden, plate in the top-left
corner), and Detail refresh runs clean and colors the right icon kind
for both a vector-icon and a glyph-icon tracker.

## v0.17.0 — 2026-08-28 — Pass 18: plugin-drawn vector icons

Owner rejected the FA "wand" glyph as a steam wand ("what is this
Harry Potter wooden stick?") and wanted the gasket "similar to ring
but flatter". No honest FA glyph exists for either, so both mockup
sheets (FA alternatives + generated stroke designs, rendered as PNG
with the tablet's own font) went to the owner, who picked two
generated designs: **steam-wand B** (spout close-up blasting steam)
and **gasket B** (the ring squashed flat).

- **Vector icon mechanism**: the two icons are stroke drawings the
  plugin renders itself — round-capped canvas lines and hollow ovals
  defined in a 0..100 design box (`vector_defs`), scaled into new
  `L(vec_box_plate)`/`L(vec_box_pick)` tokens sized via a
  physical→virtual factor so they visually match the FA glyphs (which
  are sized in physical px). `dui add canvas_item` rescales
  coordinates AND `-width` (dui.tcl:9551), and honors
  `-initial_state` (it routes through process_tags_and_var,
  dui.tcl:9542/5364).
- Segments carry unique bare tags (`<base>_s<i>`, the v0.6.2 rule);
  helpers `_add_vector_icon` / `_config_vector_icon` /
  `_show_vector_icon` create, recolor (lines `-fill`, ovals
  `-outline`) and swap them. Card plates carry both vector icons born
  hidden (v0.10.1 no-flash rule); refresh shows at most one and hides
  the glyph dtext — the stacked-but-exclusive mechanism the bottom
  bar already used. Picker cells for vector slots draw segments
  instead of a glyph dtext; `_refresh_picker` recolors either kind.
- `_item_icon_name` split out of `_item_glyph` (which now returns ""
  for vector icons instead of falling back to the wrench glyph).
- Picker: steam-wand replaced wand (slot 8), gasket-flat replaced
  record-vinyl (slot 20). Retired names keep rendering via the
  v0.16.0 legacy-icon mechanism.

**Safety status**: pure UI pass — no write-behavior changes; SDB
read-only, history files untouched, automatic writes exactly as in
v0.13.0.

Offline verification: 193 checks — the full v0.16.0 suite plus: the
FA-name sweep now exempts vector names, slots 8/20 hold the vector
icons, byte-compile of the five new/changed procs, vector name/glyph
resolution ("" glyph, name intact), picker cells build 5 stroke
segments (slot 8) / 1 oval (slot 20) with no stray dtext and all
segment points inside their cell, card plates carry both vector icons
born hidden, and a settings refresh with a vector-icon tracker runs
clean and recolors the shown segments.

## v0.16.0 — 2026-08-28 — Pass 17: required-meaning icons

Owner request: meaningful icons for steam wand, drain pipe, ball joint
and gasket, replacing duplicated picker slots 8/14/15/20.

- Picker slots replaced (names verified in the app's FA6 Pro symbol
  table): pump-soap → **wand** (steam wand), coffee-pot →
  **pipe-section** (drain pipe), mug-saucer → **circle-dot** (ball
  joint), brush → **record-vinyl** (flat gasket, distinct from the
  O-ring "ring" in row 1).
- Legacy-icon safety: `open_edit` now seeds the Edit page with the
  tracker's STORED icon even when that name is no longer a picker
  candidate. Retired names keep rendering (glyphs resolve from the FA
  table by name, not by picker membership), and since `edit_save` only
  writes picker-member icons, an unrelated rename/threshold Save keeps
  a legacy icon instead of silently swapping it for the wrench
  fallback. The picker simply shows no selection until a new icon is
  tapped.

**Safety status**: pure UI pass — no write-behavior changes of any
kind; SDB read-only, history files untouched, automatic writes exactly
as in v0.13.0.

Offline verification: 166 checks — the full v0.15.0 suite (the
core-table name sweep re-validated the four new names live) plus: new
names present / retired names absent at the owner's exact slots, a
legacy-icon tracker seeds Edit correctly, an unrelated Save keeps the
legacy icon while applying the rename, and picking a new icon still
saves normally.

## v0.15.0 — 2026-08-28 — Pass 16: consolidated trackers + button standard

Owner request: "All trackers can be deleteable and editable, even the
default ones (consolidate)… Kindly use a standard for these buttons."
Owner decisions (asked before building): built-ins stay undeletable
(Hide is their reversible retirement — deleting would permanently lose
the auto-record/burr-offset/water-meter wiring), Delete moves into the
Edit page, and hiding is capped at 6 trackers at once.

- **Every tracker is editable**: built-in dicts gained `label` (empty =
  fixed default name) and `icon` fields; `_item_label` / `_item_glyph`
  now prefer the dict for all items and fall back to the fixed tables.
  Edit seeds and saves name/threshold/icon for built-ins exactly as for
  customs; the counting unit stays locked for everyone. Two picker
  slots swapped (screwdriver-wrench→gears, soap→droplet-slash — both
  near-duplicates) so every built-in default glyph is pickable.
- **Every tracker is hideable**, at most 6 at once (`hide_max`, the
  restore-chip count). At the cap the Detail page's Hide button
  disables with a grey explanation. `apply_defaults` validates
  hidden_ids against built-ins AND customs and truncates overflow;
  deleting a hidden custom frees its slot.
- **Delete relocated to the Edit page** (customs only, bottom-center,
  danger style): first tap arms ("Yes, Delete Tracker" + message, Save
  hides), second tap deletes and returns to the card list; Cancel
  disarms first, any page show disarms (stuck-flag rule). The Detail
  page's delete button and `confirm_delete` mode are gone — Detail's
  bar is now Back | Hide | Undo for every tracker.
- **Wording**: "Add Custom Tracker" page → "New Tracker", main-page
  button → "New Tracker", its action button → "Save".
- **Button standard documented** in the implementation header:
  bottom-left safe navigation (disarms armed confirms first),
  bottom-right single positive action, red two-step isolated
  destructive actions, Edit in the header slot, Prev/Next paired and
  disabled at ends, text labels not icons.

**Safety status**: no new destructive capability — deleting remains
possible only for custom trackers (two-tap, escrowed in
`last_deleted_custom`), now reached via Edit. Built-ins, SDB and
history files can never be deleted or altered. SDB stays read-only;
the automatic writes remain exactly v0.13.0's (own settings.tdb).

Offline verification: 149 checks — the full v0.14.0 suite plus:
built-in label/icon migration, dict-first label/glyph resolution, all
built-in glyphs pickable, open_edit/edit_save on a built-in
(unit/events untouched), the 6-hide cap (7th refused, validation
truncates, custom ids valid, deleted custom frees its slot), the full
two-step Edit-page delete flow (arm, cancel-disarms-without-leaving,
save-ignored-while-armed, show-disarms, two taps delete and navigate,
built-in no-op), Detail has no delete button, Edit's delete clears
Cancel/Save by >100px, and the new labels.

## v0.14.0 — 2026-08-28 — Pass 15: 24-icon picker

Owner request after v0.13.0 was confirmed working on the tablet:
"Add 12 more icons, another row of icons, variety."

- `picker_icons` doubles to **24 glyphs in two rows of 12**. The new
  variety row: bottle-water, coffee-pot, mug-saucer, gauge-high,
  scale-balanced, temperature-half, screwdriver-wrench, brush, soap,
  calendar-check, bell, star. Every name was verified against the
  app's own FA6 Pro symbol table (de1app-core/dui.tcl:1346+) — an
  unknown name would render an empty cell (render-time wrench fallback
  still guards stored bad data).
- The shared `_build_picker_row` now wraps at `picker_cols` (12): same
  cell size, gaps and tap mechanism, tags `pick0`–`pick23`, second row
  one `xs` token below the first. `_refresh_picker`, `select_icon`
  bounds guards and icon validation in Add/Edit save paths needed no
  changes (they all iterate/validate against `picker_icons`).
- Layout: the Add page's validation message moved up beside the
  "Icon:" label (width trimmed so they can never touch) and the
  hidden-tracker chips moved up 17 px — the second picker row takes
  the freed band, and every zone keeps its clearances (chips still end
  38 px above the bottom bar). The Edit page's note and error moved
  down below the second row.

**Safety status**: pure UI pass — no write behavior of any kind was
added or changed; SDB stays read-only, history files untouched. The
automatic writes remain exactly v0.13.0's (own settings.tdb: user
taps, auto-record events, water-meter updates).

Offline verification: 94 checks — the full v0.13.0 suite plus: all 24
names present in the real core symbol table, no duplicates, both pages
build 24 non-overlapping cells in two distinct rows, no other page
item overlaps a picker cell, nothing overruns the bottom bar, refresh
walks 24 cells, and the tap guard accepts index 23 / rejects 24.

## v0.13.0 — 2026-08-28 — Pass 14: water-bottle tracking in ml

Owner request: a fresh-water supply feeds the machine from a 5-gallon
bottle — track how much is left from what the machine has consumed.

- **Lifetime water meter** (`settings(water_total_ml)`): the machine
  reports its dispensed volume for every operation in `::de1(volume)`
  (integrated from the same flow field the shot chart uses; steam
  included), and the core resets it only at the *start* of the next
  operation — so the existing `on_major_state_change` listener now
  harvests it whenever the machine **leaves** a water-drawing state
  (Espresso, Steam, HotWater, HotWaterRinse, SteamRinse, Clean,
  Descale). An armed/cleared flag means a duplicate event or an app
  restart mid-flow can only ever *miss* water, never count it twice
  (the auto-record philosophy); per-flow reads are sanity-clamped to
  0–5000 ml.
- **New built-in "Water bottle" tracker** (icon: water tank), new unit
  **ml**, default threshold 18900 ml (5 US gal). Record = "fresh bottle
  attached": the record event stores the meter reading as the new
  baseline, and the tracker's value is meter − newest baseline — so
  **Undo restores the previous bottle's baseline** through the existing
  events-are-truth design, with zero new bookkeeping.
- **Bottle confirm page** (`MaintenanceTracker_confirm_bottle`): the
  burr-offset stepper mechanism verbatim, carrying the bottle size
  (±100/±1000 ml, clamped 500–99999); Confirm writes it into the
  item's threshold. Built-ins have no Edit page — the swap confirmation
  is where the size is adjusted.
- **Display**: ml counters render in litres from 1 L up ("0.8 / 18.9 L");
  the card caption leads with **"About X L left"**; the Detail line and
  the never-recorded wording are bottle-aware.
- **ml as a third custom-tracker unit**: the Add page's unit toggle now
  cycles days → shots → ml (steppers ±100/±1000, default 10000 ml), for
  e.g. a water filter tracked by real throughput. The Edit page keeps
  units non-editable, ml included.
- **Diagnostics**: new "Water dispensed (lifetime)" row; the two
  row-count rows merged into "Rows raw / counted / excluded" (the page
  holds exactly 16 rows above the bottom bar).
- Honest limits, stated on the Add page hint too: the meter only runs
  while the app runs, and it is the machine's flow *estimate* — a few
  percent drift is expected, so the amber 80% mark is the practical
  "get a bottle ready" signal.

**Safety status**: SDB stays strictly read-only (no new queries);
history files remain untouched; no popups. The ONE new automatic write
is the plugin's **own settings.tdb** being saved when a flow/cycle
completes (the same `save_settings` path auto-record already used).
The one destructive capability remains deleting a custom tracker
(two-tap, escrowed), unchanged from v0.7.0.

Offline verification: 80 checks — byte-compile of all 24 touched procs,
defaults/migration incl. corrupt-meter self-heal and custom-ml
validation, the full harvest state machine (arm/harvest/disarm,
duplicate exit, implausible/negative/unarmed reads, Refill never arms),
record/baseline/undo round-trips, status math without any database,
litres formatting, unit cycling and steppers, and stub-dui geometry of
the new confirm page (bounds, bottom-bar clearance, non-empty labels,
page registration).

## v0.12.0 — 2026-08-27 — Pass 13: edit threshold and icon too

Owner request, extending v0.11.0's rename.

- The Detail page's **Rename** button became **Edit**, opening a full
  Edit page for custom trackers: the name entry (prefilled, top half of
  the screen), **threshold steppers** scaled to the tracker's own unit
  (±1/±10 for days, ±10/±100 for shots, clamped 1..99999), and the
  **icon picker** with the tracker's current glyph pre-selected.
- Save rewrites only `label`, `threshold` and `icon` — the event
  history, counting unit and id stay untouched (verified field-by-field
  in the harness). A threshold change invalidates the status cache so
  the card's state and wear bar recompute on return. Name validation
  is unchanged from Add/Rename.
- The counting **unit is deliberately not editable**: flipping
  days↔shots would silently change what the whole history means;
  delete-and-recreate is the honest path for that.
- The icon-picker row and its selection refresh were factored into
  shared helpers (`_build_picker_row` / `_refresh_picker`) used by both
  the Add and Edit pages, so the two can never drift apart.
- Offline verification: 68 checks — seeding of all four fields,
  per-unit stepper relabels, clamps, icon tap + outline, empty/overlong
  rejection with dict-unchanged assertion, three-field save,
  history/unit untouched, glyph and label lookup follow-through, cache
  invalidation, per-mode button visibility, click guard, entry
  placement, and the full Pass 12 suite (the Add page's picker
  re-verified on the shared helpers).

**Safety status: unchanged — three fields of one dict in the plugin's
own settings.tdb rewritten per explicit Save tap. SDB read-only; no
history-file access; no popups.**

## v0.11.0 — 2026-08-27 — Pass 12: rename custom trackers

Owner request: rename a custom tracker without losing anything.

- A **Rename** button sits top-right on a custom tracker's Detail page
  (the design system's mode-button slot; the page title's wrap width was
  narrowed so a long name can never run under it). It opens a small
  Rename page: the current name prefilled in a text entry (top half of
  the screen for the Android keyboard), a note stating that only the
  name changes, Cancel / Save.
- Save applies the Add page's exact validation (whitespace collapsed
  and trimmed, non-empty, max 40 characters; rejections show on-page
  and change nothing) and rewrites **only the `label` field** — the
  event history, icon, unit, threshold and id are untouched, verified
  field-by-field in the harness. Built-ins keep their fixed names; the
  button is hidden for them and while any confirm is armed.
- All the recent no-flash and label rules are honored: the button is
  created `-initial_state hidden` with `-initial 1` show/hides, and
  every label present at creation.
- Offline verification: 61 checks — builtin refusal, seeding, empty /
  overlong rejection, whitespace collapse, untouched-fields audit,
  label lookup follow-through, per-mode button visibility, click guard,
  entry placement, plus the full Pass 11 suite.

**Safety status: unchanged — one dict field in the plugin's own
settings.tdb rewritten per explicit Save tap. SDB read-only; no
history-file access; no popups.**

## v0.10.1 — 2026-08-26 — bugfix: one-frame flash of hidden items on page entry

Owner report: entering/leaving the Add Tracker page flashed the hidden
tracker's card (settings page) and the blank unused restore chips (Add
page) for a split second before they disappeared.

Root cause in the core's page machinery: `dui page load` re-shows every
item of the incoming page that does not carry an `st:hidden` tag
(de1app-core/dui.tcl:6615-6644) BEFORE the page's `show{}` callback
runs — and plain `dui item show/hide` only changes the *current* view,
not that tag. So every page entry restored creation-state visibility
for one paint until our refresh re-hid the unused items.

- Every dynamic show/hide (settings card rows + segments, Detail bar
  buttons, Add-page chips and title) now passes **`-initial 1`**, the
  framework's own option for persisting visibility across page shows
  (dui.tcl:7987-8010) — the hidden state survives page transitions, so
  there is nothing to flash.
- Contextual items that start hidden (restore chips, hidden-trackers
  title, Detail's Undo/Delete/Hide buttons) are additionally created
  with **`-initial_state hidden`**, killing the first-ever-open flash
  too. This is the core's own pattern for composite widgets
  (dui.tcl:12422); the label sub-items inherit the `st:hidden` tag via
  the processed tag list (dui.tcl:5387, 5573).
- Offline harness upgraded to track both: it records `-initial 1` on
  show/hide and `-initial_state hidden` at creation, with new checks
  that the chips, bars and card rows all use them.

**Safety status: unchanged — visibility-mechanics only.**

## v0.10.0 — 2026-08-26 — Pass 11: per-item restore for hidden trackers

- The Add page's all-or-nothing **Restore All** button is replaced by
  one **chip button per hidden tracker**, wearing the tracker's name;
  tapping a chip restores exactly that tracker and the row re-renders.
  Six fixed chip slots (there are only six built-ins) are relabeled and
  shown/hidden per refresh — created with non-empty placeholder labels
  (the v0.7.1 empty-label lesson) and relabeled via bare tags (the
  v0.6.2 lesson). The tapped id maps through the rendered list cached
  at refresh time, never through live settings, so a chip always
  restores the tracker whose name it shows.
- `unhide_all` is replaced by `unhide_item`; hiding (Detail page) is
  unchanged.
- Offline verification: chip labels/order, slot visibility, tap →
  restore → chip slide-down, out-of-range tap ignored, not-hidden id
  refused, empty-state hides the whole row; full Pass 10 suite
  re-passed (geometry audit includes the six chips).

**Safety status: unchanged — restoring only removes an id from
`hidden_ids` in the plugin's own settings.tdb, per explicit tap. SDB
read-only; no history-file access; no popups.**

## v0.9.0 — 2026-08-26 — Pass 10: "Service Bay" card redesign + icon picker

Owner picked Concept 01 of the three delivered mockups.

- **Cards redesigned**: each card now shows a state-tinted **icon
  plate** (solid tint — canvas has no alpha — computed by blending the
  state color onto the card white), the tracker name, an uppercase
  state word in the state color, a right-aligned counter ("380 / 365
  days"), a **segmented wear bar** (20 segments whose fills reconfigure
  — the proven dot mechanism, no canvas-coords manipulation) with a
  static tick at the amber threshold, and the last-done caption.
  Never-recorded shows an empty bar with the "Tap Record…" hint;
  database-unavailable shows "?" and the Diagnostics hint.
- **Worst-first sorting**: the card list orders red → amber → ok →
  no-data → never-recorded (stable within each group), so trouble is
  always on page one. The rendered order is cached (`displayed_ids`)
  and taps map against exactly what is on screen. Toolbar now says
  "worst first".
- **Icons from the app's own Font Awesome 6 Pro symbol table**
  (`dui symbol exists/get`, font loaded via the core's
  `dui::font::add_or_get_familyname`, physical-pixel sized like every
  other font). Built-ins: grate-droplet (backflush), droplet-slash
  (descale), ring (gasket), coffee-beans (burr clean), gears (burr
  install), filter (water filter). If the font or a symbol is missing,
  plates render glyph-less and everything else still works.
- **Icon picker on the Add page**: a row of 12 glyph cells (wrench,
  mug-hot, coffee-beans, filter, droplet, grate-droplet, ring,
  pump-soap, faucet-drip, tank-water, stopwatch, spray-can); the
  selection is outlined, stored in the custom tracker's new `icon`
  field (default wrench, migration fills existing customs), and an
  unknown stored name falls back to wrench at render time. The Add
  page's column moved up to make room; the name entry stays in the top
  half of the screen.
- Offline verification: 97 procs byte-compiled; 36 checks — glyph
  resolution incl. fallbacks, tint computation, worst-first ordering,
  per-row render (texts, colors, segment fills, plate tint+glyph),
  picker selection flow and icon persistence, geometry audit with the
  picker tap zones, and hide/delete/detail regressions.

**Safety status: presentation-only pass — zero new write behavior. The
only settings change is the custom-tracker `icon` field written by the
same explicit Add Tracker tap as before. SDB read-only; no history-file
access; no popups.**

## v0.8.0 — 2026-08-26 — Pass 9: hide/restore built-in trackers

Owner requests: remove burr install (burrs last ~30k shots — not routine
maintenance) and make the official trackers hideable in general.

- **Hide Tracker** on every built-in item's Detail page (center slot,
  normal style — it is non-destructive): one tap hides the tracker from
  the card list AND from the status rollup, so a hidden item can never
  light the Lumen dot. The item's dict, event history and burr offset
  stay untouched in settings.
- **Restore All** on the Add page: a "Hidden built-in trackers: …" line
  (with names) plus a button appear whenever something is hidden;
  restoring brings every hidden tracker back. Per-item restore can come
  with the upcoming card redesign.
- **Burr install starts hidden** via a one-shot migration
  (`hide_burr_done` flag — a deliberate later restore sticks across
  restarts). Its fresh-install threshold default is now 30000 shots
  (was 1500); existing settings keep their stored value.
- Custom trackers cannot be hidden (they are deleted instead);
  `hidden_ids` is validated to built-in ids on every load.
- Detail page center slot now holds two stacked, mutually-exclusive
  buttons (Delete for customs / Hide for built-ins — at most one is
  ever shown; hidden canvas items receive no events, the same mechanism
  dui uses for page switching). Diagnostics gained a "Hidden trackers"
  row (rows 15 → 16).
- Offline verification: 92 procs byte-compiled; 38 checks including
  migration one-shot-ness, hidden-list validation, per-mode button
  visibility on the Detail page, rollup exclusion (a hidden overdue item
  cannot turn the dot red), restore flow, and v0.7.0 regressions.

**Safety status: unchanged write surface — only the plugin's own
settings.tdb via `plugins save_settings`. Hiding is non-destructive
(list membership only, all data kept) and reversible via Restore All.
The only destructive capability remains deleting a CUSTOM tracker
(two-tap confirm, escrowed). SDB read-only; no history-file access; no
popups.**

## v0.7.1 — 2026-08-26 — bugfix: Add-page stepper buttons rendered blank

Owner report: the threshold stepper buttons on the Add Custom Tracker
page had empty faces. Root cause in the core: a dbutton created with
`-label ""` never gets a label sub-item at all
(de1app-core/dui.tcl:10227-10235 only adds the label dtext when the
creation label is non-empty), so the per-unit relabel in `refresh` had
nothing to configure — and failed invisibly inside its `catch`. The
four steppers are now created with their day-unit labels (−10 −1 +1
+10) and the relabel works exactly as on-device-proven bare-tag
relabels always have. The offline harness stub now models this: it
rejects any `-label` config on a button whose creation label was empty,
so this class of failure can never pass verification again.

**Safety status: unchanged — creation-label fix only.**

## v0.7.0 — 2026-08-26 — Pass 8: custom trackers

Owner request: users with more than one grinder (or any gear beyond the
built-in six items) want their own named trackers with their own dates.

- **Add Tracker** button (settings page bottom bar, center) opens a new
  page: tracker name (free-text entry, ShotHistoryEditor's tablet-proven
  entry pattern, kept in the top half of the screen for the Android
  keyboard), a days/shots unit toggle, and a threshold set with stepper
  buttons whose step sizes follow the unit (±1/±10 for days, ±10/±100
  for shots — no keyboard needed). Name is required, trimmed, whitespace-
  collapsed, and capped at 40 characters; rejections show an on-page
  message and change nothing.
- Custom trackers are full citizens: same card list (after the
  built-ins, paging adapts automatically), same Record/Confirm flow,
  same event log, Detail page and Undo, same `status_summary` rollup
  (an overdue custom tracker lights the Lumen dot). They are
  **manual-record only** — auto-recording stays wired to the machine's
  own clean/descale cycles.
- **Shot-unit caveat, stated in the UI:** shot-based custom counters
  count every shot on the machine; the shot database cannot tell which
  grinder pulled which shot. The Add page recommends days for gear that
  does not see every shot.
- **Delete Tracker** (custom items only — built-ins can never be
  deleted): a danger button on the custom tracker's Detail page, two-tap
  confirm in the proven single-page mode-swap pattern (armed delete and
  armed undo are mutually exclusive; Cancel or any page switch disarms).
  The removed dict is kept in `settings(last_deleted_custom)` (newest
  only) so a mistaken delete is recoverable by hand from settings.tdb.
- Storage: `custom_ids` ordered list + never-reused `custom_next`
  counter; ids are `custom_<n>`. Idempotent migration in
  `apply_defaults` validates the list, fills missing fields, seeds event
  logs, and heals blank labels/bogus units.
- Diagnostics gained a "Custom trackers" count row (rows 14 → 15). The
  card list clamps its page index after a delete shrinks the list.
- Fixed while building: stepper relabels avoid `expr` for the "+10"
  label (a braced expr canonicalizes it back to 10) and use bare tags
  only (v0.6.2 rule).
- Offline verification: 87 procs byte-compiled; migration (dirty lists,
  duplicate/bogus ids, leading-zero ids, missing fields), add
  validation/rejection/whitespace collapse, unit toggle + clamps,
  record/undo on customs, status shape + custom rollup, delete
  arm/cancel/confirm/escrow/double-delete/builtin-refusal, page-index
  clamp, geometry audit (no dbutton overlaps on any page, name entry in
  the top screen half), and a wildcard-`-label`-rejecting dui stub.

**Safety status: writes remain confined to the plugin's own settings.tdb
via `plugins save_settings`. The ONE destructive capability added this
pass is deleting a CUSTOM tracker — user-created data inside the
plugin's own settings only, behind a two-tap confirm, with the removed
dict escrowed in `last_deleted_custom`. Built-in items cannot be
deleted. SDB access unchanged: read-only handle, SELECT COUNT only. No
history-file access. No popups.**

## v0.6.2 — 2026-08-26 — bugfix: Undo relabel actually happens on-device

v0.6.1's label swap never showed on the tablet (owner report "did not
work"): the five bottom-bar relabels used the wildcard tag form
(`dui item config $page bar_undo* -label ...`), which is only proven for
`show`/`hide`/`-state`. For `-label` the config must target the BARE
main tag (ShotHistoryEditor's tablet-proven caption swaps,
SHE:2640-2642); the wildcard form errors against the button's shape
sub-items — invisibly, inside the surrounding `catch`. All five configs
now use bare tags, and the offline harness stub was hardened to REJECT
wildcard `-label` configs so this class of failure can never pass
verification again (the v0.6.1 stub accepted anything, which is exactly
why it passed).

**Safety status: unchanged — tag-form fix only.**

## v0.6.1 — 2026-08-26 — Undo confirm label spells out the deletion

Owner request: after the first tap of Undo Last Record, the eye is
locked on the button — so the button itself must carry the warning, not
just the message above it. In confirm mode it now reads **"Yes, Delete
Last Record"** (was "Confirm Undo"); Cancel still restores the normal
view. The danger button widened to 300px (`btn_w_xwide`) so the longer
label fits at the shared 20px button face in both states.

**Safety status: unchanged from v0.6.0 — label and width only.**

## v0.6.0 — 2026-08-26 — Pass 7: auto-recording

Built from the approved auto-detect discovery report (core citations in
PROJECT_STATE).

- New silent `on_major_state_change` listener: the DE1's Clean (18) and
  Descale (10) machine states are NOT flow states, so the existing
  after-flow listener never sees them. Enter/exit stamps from the event
  dict's `event_time`; a completed cycle appends one `source auto` event —
  Clean ≥ 90 s → backflush (the firmware's CleanSoak alone is 60 s),
  Descale ≥ 300 s → descale. Shorter (aborted) cycles never count. The
  enter stamp is memory-only: a restart mid-cycle misses that cycle,
  never double-counts.
- The after-flow callback (`_on_flow_complete`) still invalidates the
  cache and now also detects blind-basket backflushes run as cleaning
  profiles: previous state Espresso + `beverage_type` "cleaning"
  (case-insensitive) + ≥ 15 s → backflush `source auto`.
- Auto events ride the exact Pass 6 append path (cap 20,
  `_sync_last_done`, save, cache invalidation), show on the Detail page
  as "recorded automatically", and Undo works on them unchanged. A
  duplicate within 30 s of the item's newest event is swallowed.
- Settings: `auto_record` (1) master toggle plus tunable thresholds
  `auto_clean_min_s` (90) / `auto_descale_min_s` (300) /
  `auto_bf_shot_min_s` (15); Diagnostics gained an "Auto-record cycles"
  row (rows 13 → 14).
- Handlers never throw into the core's event dispatch (whole-body catch,
  no bare `return` inside catch — TCL_RETURN would read as an error).
- Offline verification: 69 procs compiled; state machine driven through
  real/aborted clean and descale, exit-without-enter, re-enter, back-to-
  back cycles, the 30 s dup guard, cleaning-profile vs normal espresso,
  toggle-off inertness, undo-on-auto, malformed event dicts, and the
  unchanged status_summary shape.

**Safety status: writes remain confined to the plugin's own settings.tdb
via `plugins save_settings` — user taps as before, plus one `source
auto` event per completed real maintenance cycle (undoable, and off with
`auto_record 0`). No popups; both listeners are silent. SDB read-only
(SELECT COUNT only); no history-file access.**

## v0.5.0 — 2026-08-25 — Pass 6: event log + revert

- Storage converted to an append-only per-item event log: each event is
  `{ts <unixtime> source <manual|auto> note <string>}`, newest last,
  capped at the newest 20. `source` is always `manual` in this version
  (`auto` reserved for a future auto-detection pass; the UI already
  displays it if present).
- `last_done` stays in the schema and in `status_summary` but is now
  derived-and-stored via `_sync_last_done` on every mutation (record
  append, undo pop, and self-healing in `apply_defaults`). All counter,
  state, and burr-offset logic reads it unchanged.
- Idempotent migration in `apply_defaults`: a v0.4.x item dict gains an
  `events` log seeded from its `last_done` (one manual event, or empty
  if never recorded); dicts that already have a log are only re-synced.
  `events` is deliberately kept OUT of the defaults list so the
  field-fill step cannot plant an empty log over a real `last_done`.
- Recording (Confirm on the existing confirmation pages) now appends an
  event instead of overwriting; Cancel unchanged; burr `pre_sdb_offset`
  untouched by events.
- New per-item Detail page, opened by tapping a card's text area (a new
  invisible tap zone — the core's own clickable-rect mechanism,
  de1app-core/dui.tcl:10247-10253 — ending an xl-gap short of the Record
  button, so no interactive rectangles overlap). Shows state dot +
  counter (reusing the card's formatting procs), the 5 newest events,
  and Undo Last Record (danger-red style, hidden when the log is empty).
  Undo confirms in-page by relabeling the same two dispatcher buttons
  (burr-stepper single-page pattern, no stacked widgets); Confirm Undo
  pops exactly the newest event and returns to the card list. Every page
  show resets the confirm mode (stuck-flag rule).
- `status_summary`'s return shape is UNCHANGED — verified key-for-key
  against the v0.4.0 shape in offline tests (top-level ok/state/sdb_ok/
  generated/items; item entries carry the same keys and types, no
  additions). Lumen's consumption is unaffected.
- Offline verification: 66 procs byte-compiled; migration idempotence
  and self-healing proven; append/cap-at-20/undo-to-empty semantics
  asserted; full detail-page dispatcher flow simulated; geometry audit
  (all pages inside the virtual canvas, card text zone disjoint from the
  Record button by 64 px).

**Safety status: the ONLY writes in this version are to the plugin's own
`settings.tdb` via `plugins save_settings`, each behind an explicit user
tap (Confirm to record, Confirm Undo to remove the single newest
record). No bulk delete or reset exists. SDB access unchanged: read-only
handle, SELECT COUNT / sqlite_master / PRAGMA only. No history-file
access. No popups; the one event listener still only invalidates the
counter cache.**

## v0.4.0 — 2026-08-25 — Pass 5: UI polish + count-query fix

Root cause found by remote screen verification (adb screenshots of the
live tablet) after v0.3.0 testing: Diagnostics showed
`SDB read failed: ICU error: u_strToLower(): link error` — the tablet's
AndroWish SQLite build cannot execute `lower()`, so every
exclusion-filtered count failed (plain COUNT worked). The card UI itself
rendered exactly to spec.

- Exclusion SQL rewritten without `lower()`: bare `LIKE` for beverage
  types (SQLite LIKE is ASCII case-insensitive by default) and
  `NOT LIKE '%kw%'` for title keywords. Verified against a copy of the
  real shots.db: identical results to the old SQL (1064 counted / 7
  excluded of 1071 raw).
- Filter failure no longer takes counting down: on any exclusion-SQL
  error the plugin logs it, counts unfiltered, and flags the condition
  on the new Diagnostics "Cleaning filter" row (active / unavailable
  with the error). Per-item count failures are now caught individually.
- Diagnostics renders `n/a` instead of blank cells (rows 12 → 13).
- Also verified live on-device via adb: stale-confirm-page resurfacing
  after the skin's own home navigation (core fpdialog/page_stack
  behavior) recovers cleanly through Cancel's `_return_to_page`
  mechanism; no plugin change needed.

**Safety status: unchanged from v0.3.0 — SDB read-only (SELECT COUNT
only), no history-file access, no popups, the only write is the plugin's
own `settings.tdb` after explicit Confirm.**

## v0.3.0 — 2026-08-25 — Pass 3: recording + status cards

- Main page redesigned as the standard card list (ShotHistoryEditor card
  system): 5 cards per page with Prev/Next paging in the toolbar (6 items
  → 2 pages), each card with a rounded state dot (green ok / amber due
  soon / red overdue / grey never recorded or no data), three text lines
  (name+state, counter detail, last-done date), and a Record button.
- Recording flow: Record → dedicated confirmation page showing the
  current counter → Confirm writes `last_done = now` into the item's dict
  in the plugin's own settings and saves via `plugins save_settings`;
  Cancel changes nothing. The pending-item flag is cleared on every
  settings-page show (stuck-flag rule).
- Burr install confirmation page adds the pre-history shot offset
  (−100/−10/+10/+100 steppers, clamped 0..99999, no Android keyboard);
  the burr counter is offset + shots since install.
- New tokens: toolbar zone, card set, state colors, solo header;
  `rounded_rect` helper copied verbatim from ShotHistoryEditor.
- Prev/Next disable at the ends (GFC `-state` pattern); unused card rows
  hidden with the SHE show/hide wildcard pattern (no stacked widgets).
- Offline verification: 56 procs byte-compiled; all four pages' geometry
  checked inside the virtual canvas with no zone overlaps; record /
  cancel / paging / offset-clamp flows exercised end-to-end against
  stubbed dui + fake DB.

**Safety status: the ONLY write in this version — including the new
Record feature — is to the plugin's own `settings.tdb` via
`plugins save_settings`, after an explicit user Confirm. SDB access is
unchanged from v0.2.0: read-only handle, SELECT COUNT only. No
history-file access. No popups; the one event listener only invalidates
the counter cache.**

## v0.2.0 — 2026-08-25 — Pass 2: safe data reading

- Own read-only SDB connection (`sqlite3 ... -readonly true`, ShotHistoryEditor
  `_open_ro_db` pattern with GrindAdvisor's no-`-readonly` fallback), opened
  per refresh and closed immediately after. The only SQL shapes issued are
  `SELECT COUNT(...)`, `sqlite_master` table listing, and `PRAGMA table_info`.
- Dynamic schema detection: shot table and clock/beverage_type/profile_title
  columns resolved by ordered regex patterns; nothing assumed. Missing
  optional columns simply disable the parts that need them.
- Defensive cleaning-row exclusion filter (beverage types cleaning/
  calibration/test/testing; title keywords rinse/flush/backflush/clean/
  descale/calibrat), built only from columns that exist; keyword lists
  surfaced verbatim on Diagnostics.
- Real `status_summary`: per-item states ok/amber/red/unset/unknown, worst
  state rollup, counts as raw rows per the Pass 0 design rule (soft-deleted
  shots still count; ShotHistoryEditor's trash manifest is never read).
  Burr-install total = pre_sdb_offset + count since install. Cached with a
  10-minute TTL; invalidated by a silent `after_flow_complete` listener
  (with a 60 s re-invalidation to catch SDB's own sync lag) and forced
  fresh whenever the plugin's own pages open.
- Settings page now shows the database headline + six per-item status
  lines; new Diagnostics page (12 label/value rows) with Back via the
  one-close_dialog-per-level `_return_to_page` mechanism.
- Offline verification: 40 procs byte-compiled; full status logic exercised
  against a fake recording DB handle; all generated SQL replayed via
  sqlite3.exe against a read-only copy of the real shots.db (1071 raw rows,
  7 excluded by the filter).

**Safety status: SDB access is read-only (own `-readonly true` handle,
SELECT COUNT only — verified by replaying every generated statement). No
history-file access. No popups; the one event listener only invalidates
the counter cache. The only file written remains the plugin's own
`settings.tdb` via `plugins save_settings`.**

## v0.1.0 — 2026-08-25 — Pass 1: minimal loadable plugin

First version. Goal: appear in Extensions, load without errors, navigate
cleanly, persist settings. No features yet.

- Plugin manifest with author/version/description and settings defaults
  (six maintenance items as dicts: `last_done`, `note`, `threshold`,
  `unit`, plus `pre_sdb_offset` on `item_burr_install`; `amber_fraction`).
- `apply_defaults` fills missing keys/fields after `plugins load_settings`,
  so future schema additions stay backward compatible.
- One fpdialog settings page (`MaintenanceTracker_settings`,
  `-namespace true -theme default`) with a static "loaded successfully"
  confirmation, self-painted background, and a Done button.
- Layout/font system copied from ShotHistoryEditor v0.7.1: virtual
  2560x1600 coordinates, physical-pixel fonts through shared `MT_*` font
  objects including `font_button` (`-label_font` per instance), button
  aspect style registered with `-theme default`, explicit fills.
- Navigation copied verbatim from ShotHistoryEditor v0.5.3:
  `open_page` cascade (open_dialog → load → show), `_capture_return_page`
  (skips transient flow pages and own pages), `_navigate_done` (validated
  `dui page load`, `close_dialog` fallback), failures logged via `msg`.
- Public API stubs for the Lumen skin: `status_summary` (returns `ok 0`
  "not implemented" — guarded callers show nothing) and `open_page`.

**Safety status: no write behavior exists in this version except the
plugin's own `settings.tdb`, written by the app's standard
`plugins save_settings` mechanism. No SDB access of any kind (not even
read-only). No history-file access. No hooks, popups, or automatic
triggers.**
