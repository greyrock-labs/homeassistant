# HVAC Architecture

This document captures the design of the home HVAC system in Home Assistant.
It exists so that future changes to the HVAC automations have the context they
need to make safe edits. The automations themselves encode *what* the system
does; this file encodes *why*.

## Components

### The season signal: `input_select.hvac_season`

The single source of truth for "what season are we in." Values:
**Heat / Cool / Off** (default: Off).

This was migrated from the dining room thermostat's HVAC mode on 2026-06-30.
The old thermostat could be put into `cool` mode even though it didn't drive
any actual cooling hardware; the new one cannot. Season needed to become a
manual selection rather than a thermostat mode.

**The dining room thermostat (`climate.dining_room_dining_room_thermostat`)
is now heat-only for HVAC purposes.** It is still referenced as the target of
`set_temperature` writes in `hvac_apply_mode` (5 places) — these set the
physical heat setpoint. It is no longer the season signal. Do not "clean up"
those setpoint writes thinking they're stale.

### The engine: `automation.hvac_apply_mode`

The core logic. Sets mini-split modes and temperatures based on:

- `input_select.hvac_season` — Heat / Cool / Off
- `input_boolean.any_home` — anyone home (derived from person entities)
- `input_boolean.use_oil_heat` — oil furnace fallback (auto-set on grid loss)
- `input_boolean.office_ac` — manual office AC override
- `input_boolean.master_bedroom_ac` — manual master bedroom AC override
- `input_number.setpoint_cool_office_override` — setpoint used while
  `office_ac` is on (default 72 °F). Editable from the dashboard.
- `input_number.setpoint_cool_mbr_override` — setpoint used while
  `master_bedroom_ac` is on (default 68 °F). Editable from the dashboard.
- `input_number.setpoint_cool_upstairs_default` — setpoint for MBR and
  office during normal Cool-season awake/away operation when no override
  is active and (for office) no work window is in effect (default 80 °F).
- `input_number.setpoint_cool_office_work` — setpoint for office during
  declared work windows (default 72 °F).
- `schedule.office_work` — weekly schedule of office "work mode" windows.
  Default Mon–Fri 08:30–17:00, weekends empty. Editable from the dashboard.
- Time of day (awake vs asleep windows)

Triggers on changes to season, oil heat, or presence. Also called directly by
the schedule automations (`automation.trigger` with `skip_condition: true`).

### Schedule triggers

- `hvac_schedule_awake` — 07:00 weekday / 07:30 weekend → apply awake setpoints
- `hvac_schedule_asleep_heat` — 23:00, only if season=Heat → apply heat sleep setpoints
- `hvac_schedule_asleep_cool` — 23:30, only if season=Cool → apply cool sleep setpoints
- `hvac_mbr_pre_cool` — 22:00, only if season=Cool and anyone home → drop master bedroom to its sleep setpoint 90 minutes early

### Office AC override

Hard override. When `office_ac` is on, the office mini-split is locked to
cool at `setpoint_cool_office_override` (default 72 °F). When it turns off,
the room returns to following Apply Mode's schedule.

- `hvac_office_ac_on` — when `office_ac` turns on, **immediately** set
  office HVAC to `cool` mode and write `setpoint_cool_office_override`.
  Works in both Cool and Off season — the override takes effect on the
  toggle, not on the next Apply Mode run. If season is Heat, roll the
  boolean back off (AC and heat cannot run together — see "Heat/cool
  mutex" below).
- `hvac_office_ac_off` — when `office_ac` turns off, turn off the office
  HVAC if season is Off, or restore the appropriate Cool-season setpoint
  (away/awake/sleep) if season is Cool. Heat season is a no-op (the
  override was never active).
- `hvac_office_ac_auto_off_at_7pm` — daily 19:00 reset of `office_ac`
  (skips if already off). Cascades into `hvac_office_ac_off` for the
  season-aware setpoint restore.
- `hvac_office_work_schedule_re_apply_mode` — when `schedule.office_work`
  changes state (start or end of a work window), re-runs `hvac_apply_mode`
  with `skip_condition: true`. Without this, the office setpoint would
  not update at 08:30 / 17:00 until something else triggered Apply Mode.
  Mirrors the existing awake/asleep time-trigger pattern.

### Master Bedroom AC override

Hard override, mirroring the office. When `master_bedroom_ac` is on, MBR
is locked to cool at `setpoint_cool_mbr_override` (default 68 °F).

- `hvac_master_bedroom_ac_on` — same shape as the office on automation,
  but targets `climate.hvac_master_bedroom_hvac_master_bedroom` and
  `setpoint_cool_mbr_override`. Writes the override setpoint immediately
  in both Cool and Off season so a manual toggle is never deferred to a
  later Apply Mode run.
- `hvac_master_bedroom_ac_off` — when `master_bedroom_ac` turns off,
  season-aware setpoint restore (MBR's sleep setpoint is
  `setpoint_cool_sleep_mbr`, distinct from the rest of the house).
- `hvac_master_bedroom_ac_auto_off_at_10am` — daily 10:00 reset
  (skips if already off).

### How Apply Mode handles both overrides (Off season)

In `hvac_season == Off`, Apply Mode's override logic is a 4-way choose
that considers both booleans independently. The engine turns OFF every
room that is *not* explicitly overridden:

| `office_ac` | `master_bedroom_ac` | Rooms turned off                       |
|-------------|---------------------|----------------------------------------|
| off         | off                 | dining, game, kitchen, MBR, office     |
| on          | off                 | dining, game, kitchen, MBR (office stays on) |
| off         | on                  | dining, game, kitchen, office (MBR stays on) |
| on          | on                  | dining, game, kitchen (both stay on)   |

This 4-way table exists in **both** the any_home-off and home branches of
the choose tree; if you change one, change the other.

### How Apply Mode handles both overrides (Cool season)

In `hvac_season == Cool`, Apply Mode writes setpoints to the mini-splits
on every schedule tick (away / awake / asleep). Each of those three writes
is split into three actions so the override setpoints stick:

1. Write the schedule setpoint (e.g. `setpoint_cool_awake`) to the 3
   main-floor rooms (dining, game, kitchen).
2. Choose for MBR: if `master_bedroom_ac` is on, write
   `setpoint_cool_mbr_override`; otherwise write the schedule setpoint
   (or `setpoint_cool_sleep_mbr` in the asleep branch, or
   `setpoint_cool_upstairs_default` in the awake/away branches).
3. Choose for office: if `office_ac` is on, write
   `setpoint_cool_office_override`; else if `schedule.office_work` is on
   (awake branch only), write `setpoint_cool_office_work`; otherwise
   write the schedule setpoint (or `setpoint_cool_upstairs_default` in
   the awake/away branches).

This applies in **all three** Cool-season setpoint writes: the any_home-off
branch (away), the home awake branch, and the home asleep default branch.
If you add another override setpoint, mirror the pattern in all three.

The MBR asleep setpoint (`setpoint_cool_sleep_mbr`) and the office asleep
setpoint (`setpoint_cool_sleep`) are **not affected** by either the
override or the work schedule — overnight stays as configured regardless.

### Setpoint reference (Cool season, current values)

The values below are the as-of 2026-07-14 defaults. Editable from
the dashboard; this table is a snapshot for traceability.

| Helper                          | Used when                                         | Value |
|---------------------------------|---------------------------------------------------|------:|
| `setpoint_cool_awake`           | Main-floor 3 rooms during awake window             | 74 °F |
| `setpoint_cool_sleep`           | Dining, Game, Kitchen during asleep window        | 80 °F |
| `setpoint_cool_away`            | All 5 mini-splits when no one is home             | 80 °F |
| `setpoint_cool_sleep_mbr`       | Master Bedroom during asleep window               | 68 °F |
| `setpoint_cool_upstairs_default`| MBR and office during awake/away (no override)    | 80 °F |
| `setpoint_cool_office_work`     | Office during awake + `schedule.office_work` on   | 72 °F |
| `setpoint_cool_mbr_override`    | MBR while `master_bedroom_ac` is on                | 68 °F |
| `setpoint_cool_office_override` | Office while `office_ac` is on                     | 72 °F |

The two `_override` setpoints always win when the corresponding boolean
is on, regardless of season (Cool or Off) or time of day (awake or asleep).
In Heat season the override booleans are auto-cleared by the heat/cool
mutex, so these setpoints are not active.

### Mini-splits controlled

Dining Room, Game Room, Kitchen, Master Bedroom, Office. The Tesla's climate
is independent and not part of this stack.

## Design decisions worth preserving

These are non-obvious. Changing them without understanding why will likely
regress the system.

### Why the office and MBR are turned off in heat mode

In heating season with `use_oil_heat=off`, Apply Mode turns the office and
master bedroom mini-splits **off** but leaves Dining Room, Game Room, and
Kitchen on heat. The three common-area splits carry the heating load; the
office and MBR are not brought on.

The `office_ac` boolean is **not** involved in this decision — it only
controls the office when it is turned on explicitly.

When `use_oil_heat=on`, **all five** are turned off (oil furnace handles it).

> **Rationale not captured:** the actual reasoning behind which zones run
> in heat mode (e.g., upstairs runs warm enough from the floors below;
> office/MBR aren't needed during the day) is not encoded anywhere in the
> config. If you change this behavior, ask the homeowner before assuming
> the split was an oversight.

### Why MBR pre-cools 90 minutes early

The master bedroom gets its cool-sleep setpoint at 22:00, 90 minutes before
the rest of the house transitions at 23:30. The intent is to have the room
noticeably cooler than it was during the day by the time you go to bed.

> **Not thermal mass.** The reasoning is comfort preference, not equipment
> response time. The mini-split can cool the MBR quickly; the early start
> is so the room feels cool when you walk in, not because it needs 90
> minutes to reach temperature.

### Why oil heat turns everything off

Mini-splits draw significant power. During a grid outage, the house runs on
limited backup; the oil furnace handles the heating load instead. So when
`use_oil_heat` flips on (via `automation.grid_power_lost`), Apply Mode
turns all mini-splits off. When grid returns, `use_oil_heat` flips off
after a 5-minute confirmation delay.

### Why the office AC override is two automations instead of one

The override needs to:
1. Force the office to cool when the user asks, regardless of season —
   but skip if Apply Mode is already cooling it (avoids redundant calls).
2. Release the override when the user turns it off — but only if season
   is also Off. If season is Heat or Cool, Apply Mode keeps the office
   in the right state, and the override shouldn't fight it.

A single automation can't express both directions of this asymmetry cleanly.
The master bedroom override follows the same shape for the same reason.

### Why each override also has an auto-off automation

The two daily auto-off automations (`hvac_*_ac_auto_off_at_7pm` for office,
`hvac_*_ac_auto_off_at_10am` for MBR) exist as a safety net so a forgotten
override doesn't keep the AC running indefinitely. They cascade into the
existing off automations (which restore the schedule setpoint or turn the
room off) so there is exactly one place that knows how each override
exits.

## Heat/cool mutex

The five mini-splits are single-inverter units — they cannot cool and heat
at the same time. To prevent a `Cool`-mode override from running while
`hvac_season=Heat` (or vice versa), two safeguards enforce the mutex:

1. **Rollback in `*_ac_on`:** if season is Heat when the user toggles
   `office_ac` or `master_bedroom_ac` on, the automation immediately
   calls `turn_off` on the same boolean. The user's toggle visually
   bounces back. In Cool and Off seasons, the same automation writes the
   override setpoint immediately so the toggle takes effect without
   waiting for the next Apply Mode run.
2. **Auto-shutoff on Heat flip:** the
   `hvac_ac_overrides_auto_off_on_heat_season` automation triggers on
   `input_select.hvac_season` going to `Heat` and turns off both
   overrides. This catches the case where an override was on while
   season was Cool/Off and the user then switched to Heat — the
   in-`*_ac_on` rollback wouldn't fire because the toggle already happened
   earlier.

The cascade into `hvac_*_ac_off` is a no-op in Heat season (the existing
off automations only act in Off or Cool), so the only behavior change
on Heat transition is: both override booleans go `off`, the room follows
the Heat-season Apply Mode logic, and any locked override setpoints are
released.

The Heat-rollback does **not** apply in reverse — there's no analogous
guard for "override is on, season flips Cool→Off." That transition is
handled by the existing `*_ac_on` automations (which already handle Off
season as "set the override setpoint"). If you also want to refuse
override-on when Heat is already set, that guard already exists
(`condition: state` with `state: Heat` in `*_ac_on`).

## Gotchas

### `input_select` state values are case-exact

`input_select.hvac_season` returns values matching its option labels exactly:
**`"Heat"`**, **`"Cool"`**, **`"Off"`** — capitalized. Not `heat`, `cool`,
`off`. Conditions must match this case.

This bit us on 2026-06-30: the first migration used lowercase values
(matching the old climate entity's state format), which silently failed to
match and left all mini-splits off when season was switched to Cool.

**Rule:** when writing a condition against `input_select.hvac_season`, always
use the capitalized form. When adding a new option to the helper, remember
to update all condition sites that branch on it.

### The dining room thermostat's supported modes

As of 2026-06-30, the thermostat's `hvac_modes` attribute is
`["off", "heat", "fan_only"]`. There is no `cool` option. This is the
whole reason the season signal was migrated off the thermostat onto
`input_select.hvac_season` — the new hardware cannot represent cool mode
at all.

## Safe-change checklist

When editing any of these automations, ask:

- [ ] Did I update every condition that branches on `input_select.hvac_season`?
      Check `hvac_apply_mode` (6 branches), `hvac_schedule_asleep_heat`,
      `hvac_schedule_asleep_cool`, `hvac_mbr_pre_cool`, `hvac_office_ac_on`,
      `hvac_office_ac_off`, `hvac_master_bedroom_ac_on`,
      `hvac_master_bedroom_ac_off`.
- [ ] Did I update the Off-season override logic in `hvac_apply_mode` for
      any new AC override helper? The 4-way choose table (both / office
      only / MBR only / neither) lives in **both** the any_home-off and
      home branches.
- [ ] Did I update the Cool-season setpoint writes in `hvac_apply_mode` for
      any new override helper? The three setpoint writes (away / awake /
      asleep) each need the 3-action pattern (main-floor + MBR-choose +
      office-choose). Don't replace all 5 rooms in one call.
- [ ] Are `setpoint_cool_mbr_override` and `setpoint_cool_office_override`
      still the only places the locked setpoints are stored? Anything else
      that writes the MBR or office setpoint during Cool season will clobber
      the override.
- [ ] Does any new schedule-aware setpoint write also check the override?
      The office schedule layer sits **inside** the office-choose block,
      after the override branch, so a `schedule.office_work` window does
      not override a manual `office_ac` toggle.
- [ ] If you add another season-aware helper, did the heat/cool mutex
      still hold? `*_ac_on` rollbacks must remain for any season that
      cannot coexist with AC; the `hvac_ac_overrides_auto_off_on_heat_season`
      automation must still fire on any season transition that conflicts
      with active overrides.
- [ ] Are my state values capitalized? (`Heat`, `Cool`, `Off`)
- [ ] Did I leave the 5 `set_temperature` writes to
      `climate.dining_room_dining_room_thermostat` alone? Those still belong
      to the physical thermostat and are not stale references.
- [ ] If I added a new condition, did I add it to *every* branch of the
      choose tree, or only the one that matters? The choose tree has an
      away+oil branch, an away default branch, a home+oil branch, and a
      home default branch. Each is a separate copy of the structure.

## Verifying changes

After any edit to HVAC automations:

1. `ha_get_automation_traces` on `automation.hvac_apply_mode` after
   triggering — confirm the expected `choose` branch matched (the trace
   shows `wanted_state` vs actual `state`).
2. Watch the mini-splits respond within a few seconds of triggering.
3. Verify with `ha_get_state` on the relevant `climate.hvac_*` entities.

## A note on this document

Where the rationale for a design decision isn't knowable from the config
itself, this document either says so explicitly or, if the homeowner has
provided the reasoning, records it. It does not invent plausible-sounding
explanations to fill gaps. If you (a future reader or assistant) are about
to add a "Reason:" line and don't actually know the answer, leave it out
and flag the gap instead.

## History

- **2026-07-15** — Added `hvac_office_work_schedule_re_apply_mode`: when
  `schedule.office_work` flips on/off, re-run `hvac_apply_mode` with
  `skip_condition: true`. Closes the gap where the office setpoint
  wouldn't update at the 08:30 / 17:00 schedule boundary until something
  else triggered Apply Mode. Mirrors the awake/asleep time-trigger
  pattern. Dashboard tile added to the Office AC section of the
  Automations tab.
- **2026-07-14 (part 6)** — Aligned `setpoint_cool_sleep` from 78 to 80 °F
  to match the rest of the upstairs energy-saving setpoints. Added a
  setpoint reference table to this doc.
- **2026-07-14 (part 5)** — Made the AC overrides truly instant in Cool
  season. Previously `*_ac_on` stopped in Cool and waited for the next
  Apply Mode run to apply the override setpoint, which meant a manual
  toggle could take hours to take effect (e.g., 10:30 AM toggle wouldn't
  apply until the 22:00 pre-cool). Both `*_ac_on` automations now write
  the override setpoint immediately in Cool and Off seasons. Apply Mode's
  Cool-season guard still writes the override setpoint on every tick, so
  the override remains sticky.
- **2026-07-14 (part 4)** — Added heat/cool mutex for AC overrides.
  `*_ac_on` automations now actively roll the boolean back off when
  season is Heat (instead of silently no-op'ing). New automation
  `hvac_ac_overrides_auto_off_on_heat_season` turns off both overrides
  the moment season flips to Heat, catching the case where an override
  was active before the transition.
- **2026-07-14 (part 3)** — Added `setpoint_cool_upstairs_default` (80),
  `setpoint_cool_office_work` (72), and `schedule.office_work` (Mon–Fri
  08:30–17:00 default). `hvac_apply_mode` Cool-season setpoint writes
  changed: MBR default in awake/away branches now uses
  `setpoint_cool_upstairs_default` instead of `cool_awake`/`cool_away`;
  office default in awake branch now picks
  `setpoint_cool_office_work` (when `schedule.office_work` is on) or
  `setpoint_cool_upstairs_default` (when off). Asleep branch and the
  existing override guards untouched. Dashboard gains an Office Work
  Schedule tile.
- **2026-07-14 (part 2)** — Promoted both overrides to hard locks.
  Added `setpoint_cool_mbr_override` (default 68) and
  `setpoint_cool_office_override` (default 72) helpers. `*_ac_on`
  automations now read from the new helpers and skip when season is Cool
  or Heat. `*_ac_off` automations gained a Cool-season setpoint-restore
  branch (away / awake / sleep aware). `hvac_apply_mode` Cool-season
  setpoint writes split from "all 5 rooms" into the 3-action pattern
  (main-floor + MBR-choose + office-choose) so the override setpoints
  stick on every schedule tick. Added daily auto-off automations
  `hvac_*_ac_auto_off_at_10am` (MBR) and `hvac_*_ac_auto_off_at_7pm`
  (office).
- **2026-07-14** — Added `input_boolean.master_bedroom_ac` as a symmetric
  companion to `input_boolean.office_ac`. New automations
  `hvac_master_bedroom_ac_on` / `hvac_master_bedroom_ac_off`. `hvac_apply_mode`
  Off-season override expanded from a 1-branch choose to a 4-way choose
  covering both overrides in both any_home-off and home branches. Heat and
  Cool branches unchanged.
- **2026-06-30** — Season signal migrated from
  `climate.dining_room_dining_room_thermostat` to `input_select.hvac_season`.
  Dashboard tile, Apply Mode, schedule automations, and office AC automations
  all updated. Case-sensitivity bug discovered and fixed on first live test.