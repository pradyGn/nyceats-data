# nyceats-data

Public-data repository for the NYCEats iOS app. Serves static JSON over
GitHub Pages so the app can refresh event listings without an App Store
release.

## What's published

| Path | URL | Schema |
| --- | --- | --- |
| `docs/v1/events.json` | https://pradygn.github.io/nyceats-data/v1/events.json | v1 events envelope |

The `docs/` folder is published by GitHub Pages. The `/v1/` path segment is
the wire-format major version — a future breaking schema change would publish
to `docs/v2/` while keeping `/v1/` available for already-installed app builds.

## Schema (v1)

```jsonc
{
  "schema_version": 1,
  "generated_at": "2026-05-24T22:30:00Z",  // ISO 8601 UTC; powers "Updated Nm ago" in the app
  "ttl_seconds": 600,                       // soft expiry (informational; ETag is authoritative)
  "events": [
    {
      "event_id": 1,                        // integer; stable across publishes
      "event_date": "2026-01-20",           // free-form per AppEvent.parseSortDate
      "event_name": "NYC Restaurant Week",
      "event_desc": "Citywide prix-fixe …",
      "event_location": "Various …",
      "more_info": "official page: https://www.nyctourism.com/restaurant-week/"
    }
  ]
}
```

Field names match the `AppEvent` Swift struct 1:1, so the JSON path reuses
`AppEvent.parseSortDate` and `AppEvent.extractURL` unchanged.

## How updates flow

The source-of-truth CSV lives in the main NYCEats repo at
`NYCEats/NYCEats/Resources/2026_food_events.csv`. The publisher script
`tools/publish_events.py` (also in that repo) reads the CSV, validates each row,
and writes JSON to:

1. `NYCEats/NYCEats/Resources/events_fallback.json` — the bundled offline copy
   shipped with the next iOS build.
2. `../nyceats-data/docs/v1/events.json` — this file. After running the
   publisher, commit + push this repo and GitHub Pages serves the update
   within ~60 seconds.

Typical workflow:

```bash
# In the main NYCEats repo:
uv run tools/publish_events.py

# Then in this repo:
git -C ../nyceats-data add docs/v1/events.json
git -C ../nyceats-data commit -m "events: <one-line change>"
git -C ../nyceats-data push origin main
```

## Validation

Every push to `main` runs the workflow at `.github/workflows/validate.yml`,
which loads `docs/v1/events.json` with `python -m json.tool`. A typo'd direct
web-UI edit therefore fails CI rather than serving a broken payload to clients.
