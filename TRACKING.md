# Tracking Setup — Riders Share Trip Quiz

Google Sheets tracking via Apps Script webhook. Uses a `new Image().src` pixel fire to avoid cross-domain CORS issues on embedded iframes.

---

## Setup Steps

### 1. Create the Spreadsheet

Create a new Google Sheet. In Row 1, add these column headers **in exact order**:

| Col | Header |
|-----|--------|
| A | Timestamp |
| B | Motivation |
| C | Duration |
| D | Season |
| E | Experience |
| F | Group |
| G | Travel Open |
| H | Home Region |
| I | Vibe |
| J | Needs Bike |
| K | Planning Horizon |
| L | Destination City |
| M | Destination Slug |
| N | Destination Region |
| O | Brand |
| P | Bike Type |
| Q | Fly-To Market |
| R | Needs Rental |
| S | Alt 1 |
| T | Alt 2 |
| U | Alt 3 |

### 2. Open Apps Script

From the spreadsheet: **Extensions > Apps Script**

Paste the full contents of `riders-share-trip-quiz-tracker.gs` into the editor, replacing any existing code. Save.

### 3. Deploy as Web App

1. Click **Deploy > New deployment**
2. Click the gear icon next to **Type** and select **Web app**
3. Set **Execute as: Me**
4. Set **Who has access: Anyone**
5. Click **Deploy**
6. Authorize the permissions when prompted
7. Copy the `/exec` URL — it will look like:
   `https://script.google.com/macros/s/XXXXXXXXXX/exec`

### 4. Paste URL into index.html

Find this line near the top of the `<script>` block:

```js
const WEBHOOK_URL = 'PASTE_YOUR_APPS_SCRIPT_URL_HERE';
```

Replace with your deployed URL:

```js
const WEBHOOK_URL = 'https://script.google.com/macros/s/XXXXXXXXXX/exec';
```

Redeploy to GitHub Pages.

---

## What Gets Tracked

Every field is written as a new row when a user reaches their result screen.

### Quiz Answers

| Parameter | Values |
|-----------|--------|
| `motivation` | `scenic_cruise` `mountain_twisty` `adventure_gravel` `cross_country` `weekend_escape` |
| `duration` | `day` `weekend` `week` `epic` |
| `season` | `spring` `summer` `fall` `winter` |
| `experience` | `beginner` `intermediate` `experienced` |
| `group` | `solo` `passenger` `group` |
| `travel_open` | `yes` `no` |
| `home_region` | `west` `mountain` `south` `northeast` `midwest` (blank if `travel_open = yes`) |
| `vibe` | `nature` `food_culture` `technical` `history_culture` `nightlife` |
| `needs_bike` | `yes` `own` `unsure` |
| `planning_horizon` | `this_week` `this_month` `few_months` `just_browsing` |

### Result Data

| Parameter | Description |
|-----------|-------------|
| `destination_city` | Full city name, e.g. `Knoxville, TN` |
| `destination_slug` | URL slug, e.g. `knoxville-tn` |
| `destination_region` | `west` `mountain` `south` `northeast` `midwest` |
| `brand` | Recommended brand, e.g. `KAWASAKI` |
| `bike_type` | Recommended type, e.g. `Sport` |
| `is_fly_to` | `Yes` or `No` |
| `needs_rental` | `Yes` or `No` (based on `needs_bike` answer) |
| `alt_1` `alt_2` `alt_3` | Runner-up destination city names |

---

## Marketing Use Cases

**Which ride styles are most common?**
Filter Column B (`motivation`). If `scenic_cruise` dominates, content and ad creative should lead with cruising imagery, not adventure.

**Rental demand signal**
Filter Column R (`needs_rental`) = `Yes`. Cross-reference with Column L (`destination_city`) to see which markets have the most unmet rental demand from quiz traffic.

**Planning horizon segmentation**
Column K (`planning_horizon`). `this_week` / `this_month` = high-intent, retarget immediately. `just_browsing` = nurture sequence.

**Travel willingness by region**
Cross-tab Column G (`travel_open`) with Column H (`home_region`). Tells you whether your home-region audience is more or less willing to fly to a destination, useful for campaign geo-targeting.

**Brand demand by experience level**
Cross-tab Column E (`experience`) with Column O (`brand`). Surfaces which brands beginner vs. experienced riders are being matched to, which informs listing acquisition priorities by market.

**Top destination results**
Sort or pivot Column L (`destination_city`). Top cities = highest quiz-driven demand, prioritize those markets for SEO and listing growth.

---

## Redeployment

If you update the Apps Script code, you must create a **new deployment** (not update the existing one) for changes to take effect. The URL changes each time — update `WEBHOOK_URL` in `index.html` accordingly.

---

## Troubleshooting

**Rows not appearing in the sheet**
- Confirm the web app is deployed with "Anyone" access, not "Anyone with link"
- Open the `/exec` URL directly in a browser — you should see `ok`
- Check Apps Script **Executions** log for errors

**Data appears but columns are shifted**
- Row 1 headers must match the order above exactly
- The Apps Script writes positionally — if you added or removed a column, update `doGet()` to match

**Tracking fires on localhost but not on the live site**
- The pixel method is cross-domain safe. If it fires locally but not live, the `WEBHOOK_URL` in the deployed `index.html` is likely still set to the placeholder
