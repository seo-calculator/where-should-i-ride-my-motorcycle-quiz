# Riders Share — Motorcycle Trip Quiz

> **Find Your Perfect Motorcycle Road Trip**
> An interactive destination-matching quiz built for [Riders Share](https://www.riders-share.com) by [RiZen Metrics](https://rizenmetrics.com).

---

## Overview

A single-file interactive quiz that matches riders to a US destination, road, and rental bike based on their riding style, experience, and trip preferences. Results include a personalized touring profile, rental CTA (when applicable), and alternate destination suggestions.

**Live on GitHub Pages:** `https://seo-calculator.github.io/rs-trip-quiz/`

---

## What It Does

- 9 questions with conditional branching (10 visible if rider is not open to travel)
- Scores 132 US cities using a 30-tag taxonomy across terrain, ride style, vibe, season, duration, and bike fit
- Applies popularity dampening to prevent major hubs from dominating results
- Recommends 1 of 8 motorcycle brands via priority-rule logic
- Surfaces 3 alternate destinations from the same region pool
- Conditionally shows rental CTA and bike recommendation based on whether rider needs a bike
- Fires a tracking pixel to Google Sheets on result load
- Fully self-contained — no dependencies, no build step, no external JS

---

## Files

```
rs-trip-quiz/
├── index.html                          # Complete quiz (HTML + CSS + JS, single file)
├── riders-share-trip-quiz-tracker.gs   # Google Apps Script for Sheets tracking
├── README.md                           # This file
├── TRACKING.md                         # Tracking setup and column reference
└── CONTRIBUTING.md                     # Data update guide for cities, brands, roads
```

---

## Quiz Questions

| # | Key | Question | Notes |
|---|-----|----------|-------|
| 1 | `motivation` | What kind of riding are you planning? | 5 options |
| 2 | `duration` | How long is your trip? | 4 options |
| 3 | `season` | When are you riding? | 4 options |
| 4 | `experience` | What is your experience level? | 3 options |
| 5 | `group` | Who are you riding with? | 3 options |
| 6 | `travel_open` | Are you open to traveling to your destination? | 2 options |
| 6b | `home_region` | Where in the USA are you located? | Conditional — only shown if Q6 = No |
| 7 | `vibe` | What matters most at your stops? | 5 options |
| 8 | `needs_bike` | Will you need to rent a motorcycle? | 3 options |
| 9 | `planning_horizon` | How far in advance are you planning? | 4 options |

---

## Data Architecture

### Cities (132)

Each city entry in `CITIES` contains:

```js
'slug': {
  name: 'City, ST',
  region: 'west | mountain | south | northeast | midwest',
  roads: ['Road Name 1', 'Road Name 2', 'Road Name 3'],
  defaultBrands: ['BRAND'],
  types: ['Cruiser', 'Touring']
}
```

Each city also has a corresponding entry in `CITY_TAGS` with 10-15 tags drawn from the tag taxonomy.

### Tag Taxonomy

| Category | Tags |
|----------|------|
| Terrain | `coastal` `mountain` `canyon` `desert` `forest` `plains` `lake` `island` `river` `scenic_byway` |
| Ride Style | `twisty` `technical` `cruising` `adventure_gravel` `touring` `beginner_friendly` `sport` `intermediate` `advanced` |
| Vibe | `nature` `food_culture` `nightlife` `history_culture` `roadside_americana` `beach` `urban_gateway` |
| Season | `summer_peak` `winter_friendly` `spring_fall_best` `year_round` |
| Duration | `day_trip` `weekend` `week_plus` |
| Bike Fit | `cruiser` `touring` `adventure` `sport` `standard` |

### Brands (8)

`HARLEY-DAVIDSON` `HONDA` `BMW` `KAWASAKI` `INDIAN` `TRIUMPH` `DUCATI` `KTM`

Brand selection uses priority rules, not tag scoring, to ensure all 8 brands are reachable and distribution is realistic.

### Fly-To Markets

Cities where delivery is available (appends `/del250` to rental URL):

`las-vegas-nv` `honolulu-hi` `billings-mt` `missoula-mt` `rapid-city-sd` `anchorage-ak` `durango-co` `santa-fe-nm` `flagstaff-az` `key-west-fl` `bozeman-mt` `jackson-wy` `moab-ut`

---

## Scoring Algorithm

1. **Answer tags** — Each quiz answer maps to weighted tags via `ANSWER_TAGS`
2. **City scoring** — Each candidate city scores points for tag overlap with the answer set
3. **Popularity dampening** — `_POPULARITY` pre-pass computes each city's win rate; cities exceeding 20% of regional paths receive a scaled penalty
4. **Home region boost** — When rider is not open to travel, cities in their home region receive a +1.5 bonus
5. **Tiebreaker** — Random 0–0.8 added to prevent identical scores locking to the same city

Region pool is determined by `travel_open`:
- `yes` — all 5 regions scored
- `no` — home region plus geographic neighbors only

---

## Tracking

Results fire a pixel to a Google Apps Script webhook. See [`TRACKING.md`](./TRACKING.md) for full setup instructions and column reference.

To enable tracking, replace the placeholder in `index.html`:

```js
const WEBHOOK_URL = 'PASTE_YOUR_APPS_SCRIPT_URL_HERE';
```

---

## Deployment

No build step. Drop `index.html` into any static host.

**GitHub Pages:**
1. Push to the `main` branch of this repo
2. Go to Settings > Pages > Source: `main` / `/ (root)`
3. Live at `https://seo-calculator.github.io/rs-trip-quiz/`

**Embed on Riders Share (iframe):**
```html
<iframe
  src="https://seo-calculator.github.io/rs-trip-quiz/"
  width="100%"
  height="900"
  frameborder="0"
  scrolling="auto"
  title="Motorcycle Trip Quiz">
</iframe>
```

---

## Credits

Quiz data, scoring algorithm, tag taxonomy, and content strategy by [RiZen Metrics](https://rizenmetrics.com).
Built for [Riders Share](https://www.riders-share.com) — the USA's largest peer-to-peer motorcycle rental marketplace.
