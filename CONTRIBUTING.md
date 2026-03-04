# Contributing — Riders Share Trip Quiz

Internal reference for RiZen Metrics. Documents how to update cities, roads, brands, and scoring logic without breaking the algorithm.

---

## File Structure

Everything lives in `index.html`. The script block is organized into clearly labeled steps:

```
STEP 1: DATA OBJECTS
  ├── FLY_TO_MARKETS      (Set of city slugs with delivery available)
  ├── ANSWER_TAGS         (Answer ID -> weighted tag map)
  ├── BRAND_PROFILES      (Brand -> types, tags, emoji, name)
  ├── CITY_TAGS           (City slug -> tag array)
  └── CITIES              (City slug -> name, region, roads, defaultBrands, types)

STEP 2: QUIZ ENGINE       (conditional-aware navigation, progress dots)

STEP 3: MATCHING ALGORITHM
  ├── _POPULARITY         (pre-pass win-rate computation for dampening)
  ├── _rawScore()         (tag overlap score, no dampening)
  ├── computeResult()     (region pool, dampening, home boost, tracking fire)
  └── getBrandProfile()   (priority-rule brand selection)

STEP 4: RESULT RENDERING
  ├── buildTouringProfile()
  ├── getRouteDescription()
  ├── buildRentalURL()
  ├── renderResult()
  └── loadAltResult()
```

---

## Adding or Updating a City

### 1. Add to `CITIES`

```js
'city-name-st': {
  name: 'City Name, ST',
  region: 'west',   // west | mountain | south | northeast | midwest
  roads: ['Primary Road', 'Secondary Road', 'Tertiary Road'],
  defaultBrands: ['HARLEY-DAVIDSON', 'HONDA'],
  types: ['Cruiser', 'Touring']
},
```

Slug format: `city-name-st` (lowercase, hyphens, two-letter state code).

### 2. Add to `CITY_TAGS`

```js
'city-name-st': ['coastal', 'twisty', 'food_culture', 'year_round', 'weekend', 'cruiser', 'sport'],
```

**Tag selection guidelines:**
- Use 10-15 tags per city
- At minimum include: 2-3 terrain tags, 1-2 ride style tags, 1-2 vibe tags, 1-2 season tags, 1 duration tag, 1-2 bike fit tags
- If a city is geographically close to a dominant hub (e.g. new suburb vs. existing metro), add 1-2 differentiating tags it alone has — otherwise it will be orphaned by the scoring

### 3. If it is a fly-to market, add to `FLY_TO_MARKETS`

```js
const FLY_TO_MARKETS = new Set([
  ...
  'city-name-st',
]);
```

### 4. Run validation

After adding, open the browser console and run:

```js
Object.keys(CITY_TAGS).filter(s => !CITIES[s])  // Tags without a CITIES entry
Object.keys(CITIES).filter(s => !CITY_TAGS[s])  // CITIES without tags
```

Both should return empty arrays.

---

## Removing a City

1. Delete from `CITIES`
2. Delete from `CITY_TAGS`
3. Remove from `FLY_TO_MARKETS` if present
4. Search for the slug anywhere else in the file (road descriptions, etc.)

---

## Updating Roads for a City

Roads live in the `CITIES` object under `roads: []`. The first road in the array (`roads[0]`) is used as the headline on the result card and in alt-result cards. Order matters — put the most recognizable road first.

---

## Adding a Quiz Question

Quiz questions are defined in `QUIZ_QUESTIONS`. Each object follows this shape:

```js
{
  id: 'q10',
  text: "Question text shown to the rider",
  sub: "Subtext below the question",
  key: 'answer_key',             // stored in answers object
  conditional: (ans) => ...,     // optional — omit if always shown
  options: [
    { id: 'option_id', icon: '🏍', title: 'Option Title', desc: 'Short description' }
  ]
}
```

**If the new answer should affect city scoring**, add corresponding entries to `ANSWER_TAGS`:

```js
ANSWER_TAGS.your_new_key = {
  option_id_1: { tag1: 3, tag2: 2 },
  option_id_2: { tag3: 4 }
}
```

Then reference it in `_rawScore()`:

```js
add(ANSWER_TAGS.your_new_key[your_new_key] || {});
```

And include it in the `_POPULARITY` pre-pass loops.

**If it is informational only** (like `planning_horizon`), just add it to `QUIZ_QUESTIONS` and make sure it is pulled in `computeResult()` for the tracking payload.

---

## Updating Brand Logic

Brand selection is in `getBrandProfile()` using priority rules, not tag scoring. The order of `if / else if` conditions is intentional — higher conditions are rarer and more specific, lower conditions are broader fallbacks.

To add a new brand:
1. Add to `BRAND_PROFILES` with `types`, `tags`, `emoji`, `name`
2. Add a condition in `getBrandProfile()` that uniquely identifies that brand's rider profile
3. Add a reason string to the `reasons` object inside `getBrandProfile()`
4. Test that the existing distribution hasn't shifted too far by running the simulation script (see below)

---

## Scoring Distribution Validation

Run this in the browser console after loading the page to simulate distribution across all answer paths:

```js
// Paste this into the console with index.html open
const MOTIVATIONS = ['scenic_cruise','mountain_twisty','adventure_gravel','cross_country','weekend_escape'];
const DURATIONS   = ['day','weekend','week','epic'];
const SEASONS     = ['spring','summer','fall','winter'];
const EXPERIENCES = ['beginner','intermediate','experienced'];
const GROUPS      = ['solo','passenger','group'];
const REGIONS     = ['west','mountain','south','northeast','midwest'];
const VIBES       = ['nature','food_culture','technical','history_culture','nightlife'];

const cityWins = {}, brandWins = {};
let total = 0;

for (const m of MOTIVATIONS) for (const d of DURATIONS) for (const s of SEASONS)
for (const e of EXPERIENCES) for (const g of GROUPS) for (const r of REGIONS)
for (const v of VIBES) {
  answers = { motivation:m, duration:d, season:s, experience:e, group:g,
              travel_open:'yes', vibe:v, needs_bike:'yes', planning_horizon:'this_month' };
  // Score cities for this region
  const candidates = Object.entries(CITY_TAGS).filter(([sl]) => CITIES[sl]?.region === r);
  const best = candidates.reduce((a, [sl]) => {
    const sc = _rawScore(sl, m, d, s, e, g, v);
    return sc > a.sc ? { sl, sc } : a;
  }, { sl: null, sc: -Infinity });
  if (best.sl) cityWins[best.sl] = (cityWins[best.sl] || 0) + 1;
  brandWins[getBrandProfile(m,e,g,v).brand] = (brandWins[getBrandProfile(m,e,g,v).brand] || 0) + 1;
  total++;
}

const cities = Object.keys(cityWins).length;
const orphans = Object.keys(CITY_TAGS).filter(s => !cityWins[s]).length;
console.log(`Cities winning: ${cities}/132 | Orphaned: ${orphans}`);
console.log('Brand distribution:', Object.entries(brandWins).sort((a,b)=>b[1]-a[1])
  .map(([b,w]) => `${b}: ${(w/Object.values(brandWins).reduce((a,n)=>a+n,0)*100).toFixed(1)}%`).join(' | '));
```

**Targets to maintain:**
- Cities winning at least once: 85+/132
- No single city above 25% of its region's paths
- All 8 brands reachable
- Harley-Davidson should be the most common result (broadest rider base)

---

## Tracking

See [`TRACKING.md`](./TRACKING.md) for full setup, column reference, and marketing use cases.

The tracking payload is assembled in `computeResult()` and fired via `trackQuizResult()`. To add a new tracked field:
1. Add the parameter to the object passed to `trackQuizResult()`
2. Add a corresponding `e.parameter.your_field || ""` line to `doGet()` in the Apps Script
3. Add a new column header to the Google Sheet — **append to the end** to avoid shifting existing columns

---

## Deployment Checklist

Before pushing to GitHub Pages:

- [ ] `WEBHOOK_URL` is set to the live Apps Script `/exec` URL (not the placeholder)
- [ ] No `console.log` statements left in the code
- [ ] File opens cleanly in browser with no JS errors
- [ ] Quiz completes end-to-end and result renders correctly
- [ ] Tracking row appears in Google Sheet after completing the quiz
- [ ] `index.html` is clean UTF-8 with no BOM
