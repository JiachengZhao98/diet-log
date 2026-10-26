# Diet Log · 饮食日志

A bilingual (English / 中文) tap-to-log food tracker. Tap a food you eat often, or a saved meal, and the page totals calories, protein, carbs and fat against a training-day or rest-day target.

The live version is a private claude.ai Artifact; its data lives in the Artifact's built-in database.

## Features

- **Tap to log**: pick Breakfast, Lunch, Dinner or Snack, then tap foods. Each tap adds one serving, and each tile has − / + buttons with the current count. A saved meal logs every item at once into its own slot.
- **Ranked by habit**: food tiles are ordered by how many times each was logged in the last 7 days; ties are alphabetical (pinyin order in Chinese). The order refreshes on load and when you switch meal or category, not while you tap.
- **Undo and edit**: every action can be undone. In the log you can change servings (decimals work, e.g. 0.9 of a pasta serving), delete an item, or clear a whole meal.
- **Nutrition Facts panel**: calories (with a kJ conversion) plus protein, carbs and fat as eaten, target and percent, and protein split by meal.
- **Training vs. rest days**: chosen by weekday by default; any single day can be switched at the top.
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
- Opened outside claude.ai, `window.claude` does not exist and the page shows a notice that it needs claude.ai.

### Data model

| Path | Contents |
| --- | --- |
| `foods/{id}` | A food: `name` / `zh` (English / Chinese name), `unit` / `unit_zh` (serving size), `kcal`, `p`, `c`, `f`, `cat`, `sort` |
| `presets/{id}` | A saved meal: `name`, `slot` (`b`/`l`/`d`/`s`), `items: [{food, qty}]`, `sort` |
| `settings/targets` | `train` and `rest` targets (`kcal`, `p`, `c`, `f`) and `trainDays` (0 = Sunday) |
| `days/{YYYY-MM-DD}` | One document per day: `dayType` (`train`, `rest`, or `null` for the weekday default) and an `entries` array |

Each entry stores the per-serving `kcal`, `p`, `c`, `f`, its names at the time it was logged, and `qty`, so editing a food later never rewrites past days.

Categories are stored as English keys (`Eggs & Dairy`, `Meat & Seafood`, `Grains`, `Fruit & Veg`, `Snacks & Protein`, `Sauces & Oils`, `Eating Out`, `Other`) and translated in the UI.

### Writes

The page updates locally first, then about 400 ms later writes the whole day document. Writes to a document are queued and sent one at a time, and incoming snapshots are ignored while a write is pending so they cannot overwrite newer local changes.

## Seed data

`seed/` holds JSON exports of the food list, saved meals and targets, keyed by document id. Nutrition values come from package labels where available and typical values otherwise.

- `seed/foods.json`: 34 foods, each with English and Chinese names
- `seed/presets.json`: saved meals (currently none)
- `seed/settings.json`: 2,200 kcal on training days, 1,780 kcal on rest days, 145 g protein every day

Daily logs (`days/`) are not included in this repository.
