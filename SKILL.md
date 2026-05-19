---
name: bka-dashboard
description: |
  Expert context for maintaining and editing the BKA (Bookkeeping Advisor) Capacity Dashboard at Housecall Pro. Use this skill whenever Steve asks about the BKA dashboard, capacity model, adding or removing BKAs, changing thresholds, updating the chart, deploying changes to GitHub, or anything related to the BKA team's workload tracking system. Trigger on: "BKA dashboard", "capacity dashboard", "BKA capacity", "add a BKA", "update the dashboard", "deploy the dashboard", "change the thresholds", "BKA model", or any mention of the BKA team's hours/load tracking.
---

# BKA Capacity Dashboard — Skill

This skill gives you instant expert knowledge of Housecall Pro's BKA Capacity Dashboard so you can make edits quickly and confidently without re-explaining the whole setup each time.

---

## System Architecture

```
Google Sheets (MBO data source)
        │
        ▼
Apps Script (DashboardAggregation.gs)
  • aggregateCapacityData() — runs every Monday 7am CT
  • doGet() — serves JSON via Web App URL
        │
        ▼
GitHub Pages (public HTML dashboard)
  github.com/stevehyre/bka-capacity-dashboard
  https://stevehyre.github.io/bka-capacity-dashboard
```

The dashboard is a **single self-contained HTML file** (`index.html`) on GitHub Pages. It fetches live data from the Apps Script Web App each time it loads. No login required, shareable with anyone.

---

## Key URLs & IDs

| Resource | Value |
|---|---|
| **Live Dashboard URL** | `https://stevehyre.github.io/bka-capacity-dashboard` |
| **GitHub Repo** | `github.com/stevehyre/bka-capacity-dashboard` |
| **Apps Script Web App URL** | `https://script.google.com/a/macros/housecallpro.com/s/AKfycbwG7AFxmYZY_oSIWR2rk27894JyV-hitWXV3MDBC8ntAiVpSW83rTV-dCNBAZ3lNaNKCA/exec` |
| **MBO Spreadsheet ID** | `1fPxMHoHhzWUfx8bQdkWLQxLddosAgmdWiu3fi8yrcRY` |
| **Data Source Spreadsheet ID** | `18yUjT6BjAAU5HTBoBxChbMfM5skNCS08QBwdfZWJ5VM` |
| **Apps Script Project ID** | `15p63Bx-rx67fCCOM3K3PemQPg6q1gerZYtH1j0Z9fQFsMQmIXxIN36Il` |
| **Dashboard_Data Tab** | Sheet named `Dashboard_Data`, cell `A1` contains the full JSON |

---

## BKA Roster & Capacity Bands

There are two BKA types with different capacity bands:

### Paired BAs (work in pairs, ~125h/month budget each)
| Status | Hours Threshold |
|---|---|
| Healthy | ≤ 258h |
| Stretched | 259–323h |
| Critical | ≥ 371h |

**Current Paired BAs** (6 total, but check live data for current roster):
- Abbey Swenson
- Ana Perez
- Liz Morales
- Erika Melchor
- Nancy
- (check Dashboard_Data for current names)

### Solo BKA (Mauricio — handles his own clients)
| Status | Hours Threshold |
|---|---|
| Healthy | ≤ 132h |
| Stretched | 133–165h |
| Critical | ≥ 190h |

---

## Time Model (Default Parameters)

These are the default minutes per activity type used to convert raw MBO activity counts into estimated hours:

| Activity | Default Minutes |
|---|---|
| Call | 10 min |
| Zoom / Demo | 45 min |
| Non-bulk email | 5 min |
| Bulk email | 1 min |
| Bulk % of emails | 30% |
| Text message | 2 min |
| Chat message | 20 min |

The Time Model tab in the dashboard has sliders to adjust these in real time. The sliders update all cards and charts instantly.

---

## Dashboard Structure (3 Tabs)

1. **Org Load** — Card per BKA showing: estimated hours, status pill (Healthy/Stretched/Critical), capacity bar, and mini stats (calls, emails, texts)
2. **Time Model** — Sliders to adjust per-activity minutes; recalculates everything live
3. **Monthly Trend** — Bar chart showing 6-month history of estimated hours per BKA

---

## How to Make Common Edits

### Adding a new BKA

In `index.html`, find the `BKAS` array near the top of the `<script>` section:

```javascript
const BKAS = [
  { name: 'Abbey Swenson', type: 'paired', ... },
  { name: 'Mauricio', type: 'solo', ... },
  // Add new entry here
  { name: 'New Name', type: 'paired', history: [0,0,0,0,0,0] },
];
```

Then also add the BKA to the Apps Script aggregation function in the `BKA_NAMES` array.

### Removing a BKA

Remove their entry from `BKAS` array in `index.html` AND from `BKA_NAMES` in Apps Script.

### Changing capacity thresholds

Find the `getStatus()` function in `index.html`:

```javascript
function getStatus(hours, type) {
  if (type === 'solo') {
    if (hours <= 132) return 'healthy';
    if (hours <= 165) return 'stretched';
    return 'critical';
  }
  // Paired BAs
  if (hours <= 258) return 'healthy';
  if (hours <= 323) return 'stretched';
  return 'critical';
}
```

### Changing time model defaults

Find the `DEFAULTS` object near the top of the script:

```javascript
const DEFAULTS = {
  callMin: 10,
  zoomMin: 45,
  emailMin: 5,
  bulkEmailMin: 1,
  bulkPct: 30,
  textMin: 2,
  chatMin: 20,
};
```

### Adding a new chart or metric

Use Chart.js v4 (already loaded via CDN). Add a new panel div in the tab bar, a new `<canvas>` element, and initialize a `new Chart(ctx, { type: 'bar', ... })` in the script.

---

## Deployment Workflow

Every dashboard change requires updating `index.html` on GitHub:

### Via GitHub MCP (preferred — can do this directly in Cowork)
```
Use mcp__github__create_or_update_file to push the updated index.html
  repo: "stevehyre/bka-capacity-dashboard"
  path: "index.html"
  branch: "main"
  message: "Update dashboard: [describe change]"
```

### Via GitHub Web UI (manual fallback)
1. Go to `github.com/stevehyre/bka-capacity-dashboard`
2. Click `index.html` → pencil icon (Edit)
3. Make changes → Commit directly to main
4. Wait ~60 seconds for GitHub Pages to rebuild

GitHub Pages auto-deploys on every push to `main`. No build step needed.

---

## Apps Script (DashboardAggregation.gs)

The Apps Script runs in the context of Steve's Google account on the MBO spreadsheet. Key functions:

- **`aggregateCapacityData()`** — Reads MBO activity data, computes totals per BKA, writes JSON to `Dashboard_Data!A1`. Runs automatically every Monday at 7am CT via a time-based trigger.
- **`doGet()`** — Serves the JSON from `Dashboard_Data!A1` via HTTP GET. This is what the dashboard fetches.
- **`setWeeklyTrigger()`** — One-time setup function to create the Monday 7am trigger.

To edit the Apps Script: Go to `script.google.com`, open the project associated with the MBO spreadsheet (ID: `1fPxMHoHhzWUfx8bQdkWLQxLddosAgmdWiu3fi8yrcRY`).

The Web App deployment settings:
- Execute as: **Me** (Steve's account)
- Who has access: **Anyone** (needed for the public dashboard to fetch data)

After any Apps Script changes, redeploy the Web App: **Deploy → Manage deployments → Edit → New version → Deploy**. The URL stays the same.

---

## Data Flow: What the JSON Looks Like

The Apps Script writes JSON to `Dashboard_Data!A1` in this shape:

```json
{
  "lastUpdated": "2026-05-19T07:00:00",
  "bkas": [
    {
      "name": "Abbey Swenson",
      "type": "paired",
      "last30d": { "calls": 45, "emails": 120, "texts": 30, "chats": 8, "zooms": 3 },
      "history": [210, 230, 198, 245, 260, 218]
    }
  ]
}
```

The `history` array is 6 months of estimated hours (oldest → newest). The dashboard uses the last element as the current month.

---

## Troubleshooting

**Dashboard shows static/fallback data** → The Web App fetch failed. Check:
1. Web App URL is correct in `index.html` (search for `WEBAPP_URL`)
2. Web App is deployed as "Anyone" access
3. Apps Script `doGet()` function exists and is working (test by visiting the URL directly in a browser)

**Apps Script trigger not running** → Go to Apps Script → Triggers (clock icon) → Verify the Monday 7am trigger exists. If not, run `setWeeklyTrigger()` manually.

**GitHub Pages not updating** → Check `github.com/stevehyre/bka-capacity-dashboard/settings/pages` — it should say "Your site is live at...". Changes take ~60 seconds to propagate.

**CORS error in browser console** → This is expected if you open `index.html` directly from your filesystem. Always view via the GitHub Pages URL.

---

## Important Notes

- **No GCP project needed** — The Apps Script Web App approach bypasses Google Cloud entirely. HCP's IT policy (which required an org parent for GCP projects) is not a factor.
- **The dashboard URL is permanent** — `https://stevehyre.github.io/bka-capacity-dashboard` — bookmark it and share it freely.
- **Data updates automatically every Monday** — No manual steps needed for the weekly refresh.
- **The GitHub repo is public** — This is required for GitHub Pages free tier. The data itself is non-sensitive (just hours estimates), so this is fine.
