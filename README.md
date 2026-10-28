# Diet Log · 饮食日志

A bilingual (English / 中文) tap-to-log food tracker. Tap a food you eat often, or a saved meal, and the page totals calories, protein, carbs and fat against a training-day or rest-day target.

The live version is a private claude.ai Artifact; its data lives in the Artifact's built-in database.

## Features

- **Tap to log**: pick Breakfast, Lunch, Dinner or Snack, then tap foods. Each tap adds one serving, and each tile has − / + buttons with the current count. A saved meal logs every item at once into its own slot.
- **Meal picked by the clock**: on open, the meal is chosen from the device's local time (the user's time zone): 05:00–11:30 Breakfast, 11:30–16:30 Lunch, 16:30–21:00 Dinner, 21:00–05:00 Snack. Coming back to the page later re-picks by time, unless a meal was chosen by hand.
- **Ranked by habit**: food tiles are ordered by how many times each was logged in the last 7 days; ties are alphabetical (pinyin order in Chinese). The order refreshes on load and when you switch meal or category, not while you tap.
- **Undo and edit**: every action can be undone. In the log you can change servings (decimals work, e.g. 0.9 of a pasta serving), delete an item, or clear a whole meal.
- **Log lines show totals**: each entry's line scales with its quantity, e.g. 2 eggs show "2 eggs · 140 kcal · 12g protein". A leading amount in the serving is multiplied ("200 ml" → "400 ml", "¼ lb" → "0.5 lb"); servings with no leading amount, or described by their energy, show as "2 × 40 g dry".
- **Nutrition Facts panel**: calories plus protein, carbs and fat as eaten, target and percent, and protein split by meal.
- **kcal or kJ**: a switch on the Nutrition Facts card shows every energy figure in kcal or kJ (1 kcal = 4.184 kJ), remembered per device. Forms take input in the chosen unit; values are always stored in kcal, and fields left untouched keep their exact stored value.
- **Training vs. rest days**: chosen by weekday by default; any single day can be switched at the top.
- **Edit targets**: the "✎ Edit target" button on the Nutrition Facts card opens a sheet with calories, protein, carbs and fat for training and rest days (today's type first), a live check of what the macros add up to, and the default training weekdays.
- **Done button for the keyboard**: the iPhone number pad has no return key and the Claude app hides Safari's keyboard bar, so while a field in a sheet is being edited a Done button takes the place of × in the sheet's header, which stays pinned as the sheet scrolls. Tapping any blank area also puts the keyboard away.
- **Bilingual UI**: a button in the top bar switches between English and Chinese; the choice is remembered per device. Every food carries an English and a Chinese name and serving size.
- **Auto-translated names**: when adding or editing a food, the name in the current UI language is required and the other is optional. If it is left blank, the page asks Claude to translate it (via the Artifact `sample` capability) before saving.
- **One-off entries**: for eating out or anything not in the list, enter the numbers by hand and optionally save the food for next time. Zero-calorie items are allowed.
- **Manage**: add, edit and delete foods and meals on the page, or save any logged meal as a reusable meal.
- **Last 14 days**: each day's calories against its target, plus 7-day averages for calories and protein.

## How it works

- `index.html` is a single file: plain HTML, CSS and JavaScript with no framework. The Archivo font loads from Google Fonts.
- All UI text lives in one `I18N` dictionary with `en` and `zh` entries; static elements use `data-t`, `data-t-aria` and `data-t-ph` attributes.
- Data goes through the Artifact runtime: `await window.claude.use("db")`. It is a document store with a Firestore-like API, including live `onSnapshot` subscriptions. Translation uses `await window.claude.use("sample")`.
- The artifact declares the `db` and `sample` capabilities.
- Opened outside claude.ai (for example straight from disk in Chrome), the UI renders but `window.claude` does not exist, so there is no data and the page shows a notice that it needs claude.ai.
- An iOS version is planned; see [`docs/IOS_PLAN.md`](docs/IOS_PLAN.md).

### Data model

| Path | Contents |
| --- | --- |
| `foods/{id}` | A food: `name` / `zh` (English / Chinese name), `unit` / `unit_zh` (serving size), `kcal`, `p`, `c`, `f`, `cat`, `sort` |
| `presets/{id}` | A saved meal: `name`, `slot` (`b`/`l`/`d`/`s`), `items: [{food, qty}]`, `sort` |
| `settings/targets` | `train` and `rest` targets (`kcal`, `p`, `c`, `f`) and `trainDays` (0 = Sunday) |
| `days/{YYYY-MM-DD}` | One document per day: `dayType` (`train`, `rest`, or `null` for the weekday default) and an `entries` array |

Each entry stores the per-serving `kcal`, `p`, `c`, `f`, its names at the time it was logged, and `qty`, so editing a food later never rewrites past days. Energy is always stored in kcal; kJ is a display and input conversion only.

Categories are stored as English keys (`Eggs & Dairy`, `Meat & Seafood`, `Grains`, `Fruit & Veg`, `Snacks & Protein`, `Sauces & Oils`, `Eating Out`, `Other`) and translated in the UI.

### Writes

The page updates locally first, then about 400 ms later writes the whole day document. Writes to a document are queued and sent one at a time, and incoming snapshots are ignored while a write is pending so they cannot overwrite newer local changes.

## Seed data

`seed/` holds JSON exports of the food list, saved meals and targets, keyed by document id. Nutrition values come from package labels where available and typical values otherwise.

- `seed/foods.json`: 35 foods, each with English and Chinese names
- `seed/presets.json`: 2 saved meals, 海鲜拉面 and 咖喱海鲜饭, both for dinner
- `seed/settings.json`: training days 2,200 kcal / 145 g protein, rest days 1,900 kcal / 135 g protein; Thursday and Sunday are training days

Daily logs (`days/`) are not included in this repository.
