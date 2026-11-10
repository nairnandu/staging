# Modern Web Guidance — External Metrics Snapshot

This branch hosts an automated twice-daily snapshot ([`mwg_external_metrics.json`](./mwg_external_metrics.json)) of public adoption metrics for [`modern-web-guidance`](https://www.npmjs.com/package/modern-web-guidance).

## How It Works

[`.github/workflows/sync-mwg-metrics.yml`](./.github/workflows/sync-mwg-metrics.yml) runs twice daily at **`05:30` and `17:30 UTC`** (and supports manual `workflow_dispatch` runs) to fetch:

1. **npm Registry Downloads**: Queries the official npm API across the full lifetime of the package (`https://api.npmjs.org/downloads/range/2026-04-29:<today>/modern-web-guidance`).
2. **skills.sh Installations**: Parses the `schema.org/InstallAction` `userInteractionCount` metadata from [`skills.sh`](https://www.skills.sh/googlechrome/modern-web-guidance/modern-web-guidance).

### Snapshot Schema (`mwg_external_metrics.json`)

| Field | Type | Description |
| :--- | :--- | :--- |
| `updated_at` | `string` | ISO-8601 UTC timestamp when the snapshot was generated. |
| `npm_total` | `number` | Sum of all daily `downloads` entries returned by `api.npmjs.org`. |
| `skills_total` | `number` | Cumulative install count reported by `skills.sh`. |
| `npm_downloads` | `array` | Daily `{ "day": "YYYY-MM-DD", "downloads": <int> }` series from `2026-04-29` to `updated_at`. |

---

## Why Cumulative npm Totals May Differ Across Tools

If you compare `mwg_external_metrics.json` against third-party wrappers such as [`npm-stat.com`](https://npm-stat.com/charts.html?package=modern-web-guidance) or downstream dashboards, **most weekly numbers will align closely, while cumulative all-time totals can differ**.

Both pull from `api.npmjs.org`, so normal completed weeks match. Divergences in the overall total come from how each tool handles caching, upstream npm pipeline outages, and daily cutoffs:

### 1. Full-Range Re-Fetch vs. Write-Once Caching (Primary Driver)
* **This repository (`mwg_external_metrics.json`)** re-queries the entire date range from `api.npmjs.org` on every run. Whenever npm retroactively backfills or re-indexes historical dates, those updates are automatically reflected on the next sync.
* **`npm-stat.com`** uses a write-once database cache: it fetches only uncached dates from `api.npmjs.org` and permanently stores non-zero counts once `date <= today - 1`, and **`0` counts once `date <= today - 7`**. Once a date is stored in its database, it is never re-queried.
* **Effect on variance weeks:** `api.npmjs.org` occasionally suffers daily stats pipeline outages where valid traffic days initially return `0` and are backfilled weeks later. If `npm-stat.com` is queried while an outage day is more than 7 days old, that `0` is locked in permanently even after `api.npmjs.org` backfills it (or conversely, `npm-stat.com` may retain an earlier non-zero value if an upstream re-index later zeroes a date out).
  * *Examples:* Weeks containing late-backfilled outage days—such as **Sep 1–7, 2026** (`2026-09-03`), **Oct 13–19, 2026** (`2026-10-13`), and **Oct 20–26, 2026** (`2026-10-21`), each of which initially reported `0` on `api.npmjs.org` before being backfilled 3–9 weeks later—will show localized weekly differences that accumulate into the all-time total gap.

### 2. Daily UTC Finalization Timing
* `api.npmjs.org` typically finalizes the previous UTC day's download count around **`02:00 UTC`**.
* Between `02:00 UTC` and the next scheduled sync (`05:30 UTC`, or following a manual sync triggered before `02:00 UTC`), live queries will include yesterday's completed count one cycle ahead of the committed JSON file.

### 3. Query Start Date (`2026-01-01` vs. `2026-04-29`) — No Impact
* `modern-web-guidance` recorded its first npm downloads on **`2026-04-30`** (ahead of its May 19, 2026 launch). Querying from `2026-01-01` vs. `2026-04-29` produces zero difference, as all dates prior to April 30 have `0` downloads.

### 4. Raw API Counts vs. Gap-Interpolated Reports
* Both `mwg_external_metrics.json` and `npm-stat.com` report **raw, unadjusted** figures (preserving literal `0`s on any `api.npmjs.org` outage days that npm has not yet backfilled). Analytics reports that apply day-of-week interpolation across unresolved `0`-download outage days will naturally show a higher adjusted cumulative total.
