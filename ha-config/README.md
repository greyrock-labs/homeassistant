# HA Config Snapshots

This directory contains point-in-time snapshots of Home Assistant (HA) config
for the HVAC project. The live source of truth is HA itself; this is the
restore path if HA gets blown away.

## What's in here

- `hvac-automations/` — 20 JSON files, one per HVAC automation, named by alias
  slug (e.g. `hvac-apply-mode.json`, `hvac-office-ac-on.json`).
- `hvac-dashboard/hvac-control.json` — the `hvac-control` Lovelace dashboard
  (Overview / Setpoints / Automations tabs).

## When to refresh

After any change to the HVAC automations or the HVAC dashboard, export a
fresh snapshot. Self-imposed task: at the end of any session that touches
HVAC, run the export and commit.

## How to export

Use the ha-mcp tools to pull live state and write to the corresponding
JSON file:

```
ha_config_get_automation(identifier="automation.<id>")  # returns config dict
ha_config_get_dashboard(url_path="hvac-control")          # returns config dict
```

The config dict is what you write to disk (no `config_hash` field — that's
HA's internal optimistic-locking token, not part of the automation itself).

## How to restore

HA's automation editor doesn't have a JSON import. To restore from these
snapshots, use `ha_config_set_automation` with the `config` field set to
the snapshot's contents. For the dashboard, use `ha_config_set_dashboard`
with `config` and `config_hash` (use the current live hash, not the one
in the snapshot — that field is HA-internal).

## File format

JSON, 1:1 with HA's internal storage. The `description` field in each
automation carries human-readable intent. The `hvac-design.md` doc at
the repo root is the prose source of truth for design rationale —
the JSON here is the literal current state.
