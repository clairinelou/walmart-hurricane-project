# Walmart Hurricane Helene: Demand Surge Analysis
> **DS 2002 Midterm Project** · Malia Solomon, Anika Shah, and Clairine Lou · November 8, 2026

Our notebook is an end-to-end data pipeline and analysis of Walmart sales across Florida stores during September 2024, right before Hurricane Helene's landfall. We clean five data files (six data frames in total, since one of the files splits into two), join them with weather data from a public API, and use the results to answer five business questions about hurricane-driven demand.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Repository Contents](#repository-contents)
- [Data Sources](#data-sources)
- [Getting Started](#getting-started)
- [Pipeline Summary](#pipeline-summary)
- [Business Questions and Findings](#business-questions-and-findings)
- [Reflection Questions](#reflection-questions)
- [Key Design Decisions](#key-design-decisions)
- [Known Limitations](#known-limitations)
- [Team and Contributions](#team-and-contributions)
- [License](#license)
- [Acknowledgments](#acknowledgments)

---

## Project Overview

**Background:** In 2004, Walmart's data science team discovered that Strawberry Pop-Tarts sales surged roughly sevenfold before hurricanes hit Florida. That insight, buried inside messy and fragmented point-of-sale data, became a textbook example of how data pipelines unlock business value. This project is built around that example. The assignment is based on a case study that we reviewed in class(*Walmart Case.pdf*).

**Goal:** We are Walmart data science consultants hired to make sense of five data files that don't agree with each other: sales records, store records, a product catalog, distribution center information, and an inventory database. Our goal is to clean the data so it can be trusted, add real weather from an API, answer five questions, and tell Walmart leadership what to do.

**Key dates used throughout the analysis:**

| Period | Dates |
|---|---|
| Baseline (normal demand) | Sept 1–14, 2024 |
| Pre-landfall surge window | Sept 19–25, 2024 |
| Landfall marker used in charts | Sept 26, 2024 |

**What the notebook does:**

1. Loads six raw datasets (CSV files plus a SQLite database).
2. Cleans each one and logs every decision in a decision log and markdown explanations.
3. Consolidates 45 messy SKUs into 20 real products.
4. Pulls daily weather data from the Open-Meteo API.
5. Answers the five analysis questions with graphs and visualizations, and explains each decision for Walmart leadership.
6. Answers five written reflection questions about data quality, ETL trade-offs, pipeline trust, business vs. data, and our teamwork.

---

## Repository Contents

| File | Description |
|---|---|
| `DS_2002_Project_Working_Doc.ipynb` | Our main notebook with the full pipeline and analysis |
| `README.md` | This file |
| `inventory_and_sales.db` | SQLite database, downloaded automatically on the first run |

> **Note on weather caching:** if the Open-Meteo API fails, the notebook loads a saved copy of the weather for every store (`weather_by_store_2024-09-01_2024-09-30.csv`) from the `data` folder of our GitHub repository (see [Weather integration](#4-weather-integration)). Nothing needs to be saved locally for the notebook to run.

---

## Data Sources

All of the raw data is loaded directly from our project's GitHub repository:

`https://raw.githubusercontent.com/clairinelou/walmart-hurricane-project/main/data`

| Dataset | Source file | Raw shape | Description |
|---|---|---|---|
| `transactions` | `walmart_transactions.csv` | 25,562 × 8 | Individual sales transactions |
| `distributions` | `distribution_centers.csv` | 5 × 8 | Distribution center capacity and costs |
| `locations` | `store_locations.csv` | 14 × 8 | Store addresses and coordinates |
| `product` | `product_catalog.csv` | 45 × 7 | Product catalog (SKU variants included) |
| `daily_sales` | `inventory_and_sales.db` → `daily_sales_summary` | 3,600 × 5 | Daily sales by store and category |
| `inventory` | `inventory_and_sales.db` → `inventory_levels` | 4,320 × 6 | Daily inventory by store and SKU |

**External data:**

- **Weather:** [Open-Meteo Historical Weather API](https://open-meteo.com/) (daily precipitation sum, max 10 m wind speed, and mean sea-level pressure; no API key required). The notebook uses the archive endpoint `https://archive-api.open-meteo.com/v1/archive`.
- **Hurricane path:** NHC Forecast Advisory 9 for Helene (issued 1500 UTC Sept 25, 2024, about a day and a half before landfall), plus the actual landfall point (30.0°N, 83.7°W, near the mouth of the Aucilla River, about 10 miles WSW of Perry, FL).

---

## Getting Started

### Requirements

- Python 3.9+
- Internet connection (data and weather are fetched at runtime)

**Python packages:**

```
pandas
numpy
matplotlib
requests
```

`sqlite3`, `pathlib`, `re`, and `time` are part of the standard library.

### Running in Google Colab

1. Upload `DS_2002_Project_Working_Doc.ipynb` to Colab (or open it from Google Drive).
2. Choose **Runtime → Run all**.


> **Run order matters.** Later cells reuse objects created earlier (for example `sku_map`, `transactions_clean`, `locations_clean`, and the Q1 date variables). Run the notebook from top to bottom.

---

## Pipeline Summary

### 1. Load

Reads all of the CSVs from GitHub, downloads the SQLite database, and queries both tables. The notebook prints the shape of every raw dataset as a first check.

### 2. Clean

Each dataset has its own cleaning functions, and every decision is noted in a decision log. We first profile each raw table (unique values, missing counts, category spellings, ID formats) so we only clean what actually needs cleaning.

| Dataset | Main issues handled |
|---|---|
| **Product catalog** | 45 SKU variants for 20 products; 27 spellings of 10 categories; missing `unit_cost` (flagged, not filled); unreliable `supplier` (e.g., Hasbro listed as supplying beer), `subcategory` (cut off, e.g., "First", "Peanut"), and `package_size` (contradicts product names) columns dropped |
| **Transactions** | 30 exact duplicates removed; inconsistent store IDs (`fl-412`, `FL_105`); mixed timestamp formats (original timestamps kept for comparison); prices stored as text (`$`); 130 negative quantities set to missing; 884 missing timestamps kept and flagged |
| **Distribution centers** | `surge_capacity` stored as text (Miami listed as `~750`), converted to numeric with an approximate flag; Birmingham's missing shipment cost left as missing; no duplicate rows found |
| **Store locations** | Mixed ID formats and state labels (`FL`, `Florida`, `Fla.`); ZIP codes stored as text; duplicate Orlando and Kissimmee stores removed (14 → 12 stores); Tallahassee's missing coordinates flagged |
| **Inventory** | 626 missing `units_on_order` values kept as missing; 355 rows with 4 unmapped SKUs kept and flagged (see below); all store IDs matched the locations table, with no negative or fractional quantities, missing dates, or repeated date/store/SKU combinations |
| **Daily sales** | Mixed date formats (`2024-09-12` vs `20240912`, 218 rows reformatted); category labels standardized to match the product catalog; 76 fractional unit totals and 34 negative revenues flagged (`total_units_fractional`, `total_revenue_negative`), not altered |

**Raw profile findings (before cleaning):**

- **Product catalog:** 45 rows, 45 unique SKUs, no duplicate rows; 13 missing `subcategory` and 5 missing `unit_cost` values.
- **Transactions:** 25,562 rows with 886 missing timestamps (884 after the 30 duplicates were removed), 60 spellings of store IDs, 63 product names, 37 category labels, and quantities ranging from −5 to 6.
- **Store locations:** 14 rows covering only 12 unique store names, 3 spellings of the state, 7 region labels (before standardizing), and 1 store with missing coordinates.
- **Inventory:** 4,320 rows covering 12 stores, 30 dates (Sept 1–30), and 49 SKUs (4 more than the 45 in the catalog); `units_on_hand` ranges from 5 to 200, `units_on_order` from 0 to 100, and `reorder_point` from 20 to 50.

**Rows affected by key cleaning decisions:**

| Dataset | Decision | Rows affected |
|---|---|---|
| Product catalog | Category labels standardized | 32 |
| Product catalog | SKU differs from its canonical SKU | 25 |
| Product catalog | Product family lists more than one pack size | 37 |
| Transactions | Text/ID cleaning (spaces, blanks, store IDs) | 5,725 |
| Transactions | SKUs replaced with canonical SKUs | 13,991 |
| Transactions | Timestamps parsed to datetime | 24,648 |
| Transactions | Unparseable timestamps | 0 |
| Transactions | Prices converted to numeric | 25,532 |
| Transactions | Category labels normalized | 19,112 |
| Transactions | Rows missing a timestamp or quantity (flagged for exclusion) | 1,011 |
| Store locations | Extra spaces stripped | 1 |
| Store locations | Store IDs reformatted | 2 |
| Store locations | State labels standardized | 4 |
| Store locations | ZIP codes converted to text | 14 |

**Shape of each dataset after cleaning:**

| Dataset | Clean shape | What changed |
|---|---|---|
| `transactions_clean` | 25,532 × 15 | 30 duplicates removed; canonical SKU, clean name and category, original timestamp, and three flag columns added |
| `distributions_clean` | 5 × 9 | `surge_capacity_approximate` flag added |
| `locations_clean` | 12 × 9 | 2 duplicate stores removed; `coordinates_missing` flag added |
| `product_clean` | 45 × 8 | 3 unreliable columns dropped; canonical SKU, clean name, clean category, and `unit_cost_missing` added |
| `inventory_clean` | 4,320 × 22 | Date, missing, negative, and fractional flags added; canonical SKU merged in; `sku_unmapped` flag added |
| `sales_clean` | 3,600 × 7 | Dates parsed; `total_units_fractional` and `total_revenue_negative` flags added |

**Cleaned distribution centers:**

| DC | City | Avg daily shipments | Surge capacity | Miles to Orlando | Cost per shipment |
|---|---|---|---|---|---|
| DC-JAX | Jacksonville, FL | 500 | 1,000 | 141 | $245.00 |
| DC-ATL | Atlanta, GA | 400 | 700 | 438 | $412.50 |
| DC-DAL | Dallas, TX | 300 | 600 | 1,108 | $687.00 |
| DC-MIA | Miami, FL | 350 | 750 (approximate) | 235 | $298.00 |
| DC-BHM | Birmingham, AL | 250 | 500 | 503 | missing |

**Consistency check across files:** after cleaning, asserts confirm that every store ID in the transactions, daily sales, and inventory tables (12 stores each) exists in the cleaned store locations table, that every transaction SKU is in the SKU map, and that the category labels match across the transactions, daily sales, and product catalog.

**Unmapped inventory SKUs:** four inventory SKUs do not appear in the product catalog: `UNKNOWN-99` (91 rows), `LEGACY-PT` (90), `OLD-BATT-50` (90), and `MISC-001` (84). They are not in the product catalog or the transactions, so there is no way to tell what product they are. We left them unmapped and flagged them with `sku_unmapped` rather than guess their category.

### 3. SKU consolidation

A shared `sku_map` maps all 45 SKUs to 20 canonical products across 10 categories. A `name_key()` function (lowercase, strip non-letters, drop plural "s", drop pack words like `pk`/`ct`, sort the words) grouped 34 of the SKUs automatically, and we mapped the other 11 by hand. The map is applied to the transactions, product catalog, and inventory. Asserts confirm that every SKU maps to exactly one product, that the map covers the whole catalog, and that each category label agrees with the mapping table.

Different pack sizes within a product family (for example Pop-Tarts 4-pack vs. 6-pack) were merged into one product, because the `package_size` column contradicts the product names and can't be trusted to split products. Board games and playing cards were kept as separate canonical products inside the Entertainment category so Q5 can test them individually.

**The 20 canonical products:**

| Canonical SKU | Product | Category | SKU variants merged |
|---|---|---|---|
| `POPTART-STRAW` | Pop-Tarts Strawberry | Breakfast | 4 |
| `WATER24PK` | Bottled Water 24-Pack | Emergency | 3 |
| `BAT-AA4` | AA Batteries 4-Pack | Emergency | 3 |
| `LED-FLASH` | LED Flashlight | Emergency | 2 |
| `CANDLE-EMRG` | Emergency Candles | Emergency | 2 |
| `TARP-HD` | Heavy Duty Tarp | Emergency | 2 |
| `BEER-24` | Light Beer 24-Pack | Alcohol | 2 |
| `MONOPOLY-01` | Monopoly Board Game | Entertainment | 3 |
| `PLAYCARD-STD` | Playing Cards | Entertainment | 2 |
| `ROKU-STD` | Roku Streaming Stick | Electronics | 2 |
| `PORTCHARGER` | Portable Phone Charger | Electronics | 2 |
| `SOUP-CAN` | Canned Soup | Grocery | 2 |
| `BREAD-WH` | White Bread | Grocery | 2 |
| `PB-16` | Peanut Butter | Grocery | 2 |
| `CHIPVARIETY` | Chips Variety Pack | Snacks | 2 |
| `FIRSTAID-KIT` | First Aid Kit | Health | 2 |
| `PAPERTOWEL6` | Paper Towels | Household | 2 |
| `TOILETPAPER12` | Toilet Paper 12-Roll | Household | 2 |
| `DIAPERS-3` | Diapers Size 3 | Baby | 2 |
| `BABYFORMULA` | Baby Formula | Baby | 2 |

After cleaning, the 45 SKUs in the product catalog fall into the 10 categories as follows: Emergency 12, Grocery 6, Entertainment 5, Breakfast 4, Electronics 4, Baby 4, Household 4, Alcohol 2, Snacks 2, and Health 2. The cleaned transactions also show 45 original SKUs collapsing into 20 canonical products, and none of the transaction SKUs are unmapped. The 11 SKUs that the name-matching function could not group on its own (for example `WTR_24PK`, `STREAM-01`, `ROKU-STD`, `CHRG-PORT`, `PORTCHARGER`, and `TP-12`) were checked and mapped by hand.

### 4. Weather integration

`daily_weather()` calls the Open-Meteo API with retry and backoff for rate limits and server errors (codes 429, 500, 502, 503, 504, plus timeouts and connection errors), waiting 2 and then 4 seconds between attempts, up to three tries.

Weather is pulled for each of the 11 stores with coordinates (Tallahassee is skipped), using the store's coordinates and local time zone (Pensacola uses Central Time; the rest are Eastern), with a half-second pause between requests. The results are stacked into `weather_by_store`. If the API still fails, the notebook loads the saved copy from our GitHub repository instead. A `source` column records which version was used (`open-meteo` or `saved copy`). Asserts confirm there are no duplicate store/date rows, every store has weather, and every store has all 30 days.

In Q3, daily sales per store and category are joined to `weather_by_store` with a `many_to_one` merge, and asserts confirm the merge did not change the row count or total units. Sales timestamps have no time zone attached, so they are assumed to be store-local (every time zone's sales run from 6 AM to 10 PM local time).

In our run, the API call succeeded (`source` = `open-meteo`) for all 11 stores with coordinates, with no missing weather values in any of the three variables. After dropping rows with a missing timestamp or quantity, 24,521 transactions remained for Q3 (2,066 in Pensacola's Central Time and 22,455 in Eastern Time). The Q3 merge produced 3,300 store/day/category rows matched to weather (11 stores × 30 days × 10 categories) and 300 unmatched rows, all from Tallahassee (FL-619), which has no coordinates and is left out of the weather analysis.

### 5. Decision log

All cleaning steps are combined in a single `full_decision_log` table that records the step number, the file, the issue, the decision made, the rows affected, and our reasoning. Each dataset's log must have the same five columns (`File`, `Issue`, `Decision`, `Rows affected`, `Reason`) or an assert stops the notebook.

The combined log has 40 steps: 6 for the product catalog, 10 for transactions, 7 for store locations, 6 for distribution centers, 5 for inventory, and 6 for daily sales.

---

## Business Questions and Findings

### Q1: Demand surge identification

Which categories surged most in the 7 days before landfall versus baseline?

Average daily units were compared for Sept 1–14 (baseline) and Sept 19–25 (surge). Days with no sales in a category were counted as zero, and the notebook checks that every date has transaction records. Daily averages are used because the two periods have different lengths.

| Category | Baseline units/day | Surge units/day | % increase |
|---|---|---|---|
| Breakfast | 73.93 | 376.86 | **409.76%** |
| Emergency | 356.57 | 1,006.43 | 182.25% |
| Electronics | 69.93 | 175.86 | 151.48% |
| Alcohol | 61.57 | 138.86 | 125.52% |
| Health | 42.36 | 95.00 | 124.28% |

Emergency supplies sold the most units, but breakfast grew the most relative to its baseline. In the daily trend chart, Emergency peaks on Sept 24 and Breakfast peaks on Sept 25, so purchasing rose before the hurricane arrived.

**Remaining categories (same comparison):**

| Category | Baseline units/day | Surge units/day | % increase |
|---|---|---|---|
| Entertainment | 102.64 | 228.57 | 122.69% |
| Household | 85.36 | 176.29 | 106.53% |
| Grocery | 172.29 | 306.57 | 77.94% |
| Baby | 85.21 | 133.71 | 56.92% |
| Snacks | 59.29 | 73.71 | 24.34% |

Rows missing a timestamp, quantity, or category are dropped before the averages are calculated. The daily trend chart plots the top five categories with the baseline (Sept 1–14) and surge (Sept 19–25) periods shaded and landfall marked with a dashed line.

**Why the baseline stops at Sept 14:** sales rose above the baseline standard deviation at every store on Sept 16 or 17. Extending the baseline to Sept 18 would have pulled early pre-storm shopping into the "normal" average, which would inflate the baseline and understate the real increase. Sept 15–18 is treated as a transition period: purchases were already different from normal, but not yet at the concentrated, statewide peak right before landfall. Leaving those days out keeps the baseline normal and the surge window a genuine peak.

### Q2: The Pop-Tarts effect

Four Pop-Tarts SKUs (`PT-12`, `SKU#459812`, `POPTART-STRAW`, `PT_STRAWBERRY`) were consolidated into one product (5,005 total units, peaking at 678 units on September 25, one day before landfall). Sales began rising around mid-September, increased the most in the days before landfall, and then declined. The fragmented view makes each SKU look small and could lead to under-ordering. The consolidated view shows the true demand curve. Consolidation does not create additional sales: the combined line equals the sum of the four separate lines each day, which the notebook verifies.

**Units by SKU variant (Sept 1–30, valid dates and quantities):**

| SKU | Total units |
|---|---|
| `PT-12` | 1,332 |
| `POPTART-STRAW` | 1,239 |
| `PT_STRAWBERRY` | 1,230 |
| `SKU#459812` | 1,204 |
| **Combined** | **5,005** |

No single SKU accounts for more than about 27% of the total, so each code alone looks like a small product. During the baseline, each SKU sold roughly 7 to 34 units per day across all stores. Sales began climbing around Sept 16 and roughly doubled again by Sept 21, when the four SKUs reported 72–100 units each. All 45 transaction SKUs are mapped, and there are no unmapped Pop-Tarts SKUs in the transactions. The chart shows the fragmented and consolidated views side by side on the same 0–700 scale, with landfall marked.

**Business decision:** keep the original SKU information for tracking individual records, but use consolidated totals for order planning and for scheduling replenishment before the pre-landfall peak.

### Q3: Weather correlation and lag

Spearman correlations compare each store's daily sales with local weather, overall and by category. Because stores differ in size, units are converted to a `sales_index` (each day's units divided by that store's own baseline average for the category, so 1 is a normal day and 3 is triple). Spearman is used because it compares rankings, so it is less thrown off by big spike days in a small 30-day sample and doesn't assume a linear relationship.

Lag analysis shifts the weather from 7 days earlier to 7 days later (within each store, so one store's weather is never matched to another's sales) and recalculates the correlation at each shift. A positive lag means sales came before the weather; a negative lag means the weather came first.

| Indicator | Strongest lag (days) | Correlation at that lag | Correlation at lag 0 |
|---|---|---|---|
| Precipitation | −4 | −0.53 | −0.31 |
| Max wind speed | +4 | 0.35 | 0.27 |
| Mean pressure | −6 | −0.51 | −0.39 |

An event-timing check also finds, for each store, the first day after the baseline when sales rose more than 2 standard deviations above normal, and the first day wind rose, pressure dropped, or rain rose by the same amount.

Tallahassee (FL-619) is excluded because it has no coordinates.

**Same-day Spearman correlations by category** (sales index vs. weather, sorted by pressure):

| Category | Precipitation | Max wind speed | Mean pressure |
|---|---|---|---|
| Emergency | −0.30 | 0.26 | −0.38 |
| Breakfast | −0.29 | 0.25 | −0.38 |
| Entertainment | −0.18 | 0.27 | −0.29 |
| Electronics | −0.21 | 0.23 | −0.27 |
| Household | −0.16 | 0.22 | −0.24 |
| Health | −0.17 | 0.23 | −0.22 |
| Grocery | −0.28 | 0.17 | −0.21 |
| Alcohol | −0.18 | 0.15 | −0.20 |
| Baby | −0.24 | 0.07 | −0.12 |
| Snacks | −0.07 | 0.04 | 0.01 |

A heatmap of this table is drawn in the notebook (red = sales rise with the weather variable, blue = sales fall).

**Full lag results** (Spearman correlation of total daily sales index with weather shifted by the lag):

| Lag (days) | Precipitation | Max wind speed | Mean pressure |
|---|---|---|---|
| −7 | −0.14 | −0.12 | −0.41 |
| −6 | −0.25 | −0.17 | −0.51 |
| −5 | −0.39 | −0.22 | −0.51 |
| −4 | −0.53 | −0.22 | −0.37 |
| −3 | −0.45 | −0.15 | −0.31 |
| −2 | −0.41 | 0.01 | −0.34 |
| −1 | −0.42 | 0.13 | −0.37 |
| 0 | −0.31 | 0.27 | −0.39 |
| +1 | −0.27 | 0.27 | −0.43 |
| +2 | −0.25 | 0.31 | −0.42 |
| +3 | −0.24 | 0.34 | −0.41 |
| +4 | −0.22 | 0.35 | −0.32 |
| +5 | −0.18 | 0.29 | −0.14 |
| +6 | −0.10 | 0.23 | −0.02 |
| +7 | −0.03 | 0.22 | −0.07 |

**Event timing by store** (first day after the baseline that each measure moved more than 2 standard deviations; NaT in the notebook means it never did):

| Store | Sales jump | Wind jump | Pressure drop | Rain jump | Days sales led wind | Days sales led pressure | Days sales led rain |
|---|---|---|---|---|---|---|---|
| FL-078 | Sept 16 | Sept 26 | Sept 26 | Sept 26 | 10 | 10 | 10 |
| FL-105 | Sept 16 | Sept 25 | Sept 26 | Sept 26 | 9 | 10 | 10 |
| FL-156 | Sept 17 | Sept 25 | Sept 26 | none | 8 | 9 | none |
| FL-239 | Sept 16 | Sept 25 | Sept 26 | none | 9 | 10 | none |
| FL-287 | Sept 17 | Sept 21 | Sept 26 | none | 4 | 9 | none |
| FL-302 | Sept 16 | Sept 25 | Sept 26 | none | 9 | 10 | none |
| FL-330 | Sept 16 | Sept 25 | Sept 20 | Sept 17 | 9 | 4 | 1 |
| FL-412 | Sept 16 | Sept 26 | Sept 26 | none | 10 | 10 | none |
| FL-445 | Sept 16 | Sept 25 | Sept 26 | Sept 26 | 9 | 10 | 10 |
| FL-501 | Sept 16 | Sept 25 | Sept 20 | Sept 18 | 9 | 4 | 2 |
| FL-710 | Sept 16 | none | Sept 26 | none | none | 10 | none |

Across stores, sales led wind by 4–10 days (median 9), pressure by 4–10 days (median 10), and rain by 1–10 days (median 10). The time-series chart of the average sales index next to wind and pressure shows sales starting to climb on about Sept 16 and peaking at roughly 3.5 times normal on Sept 24–25, while wind and pressure stayed near normal until the day before landfall.

**Interpretation:**

- **Same-day relationships are weak to moderate** (none stronger than ±0.4). Pressure had the strongest same-day correlation (−0.39), followed by precipitation (−0.31) and wind (+0.27). Sales were higher on lower-pressure and windier days.
- **Storm-prep categories responded most.** Pressure was negatively correlated with sales in nine of the ten categories. Emergency and Breakfast (both −0.38 with pressure), Entertainment (−0.29), and Electronics (−0.27) matched the categories that surged in Q1. Wind was positively correlated in every category (mostly +0.15 to +0.27). Baby responded weakly, and Snacks showed almost no relationship with any indicator (−0.07, +0.04, +0.01).
- **Precipitation was negative in every category (−0.07 to −0.30)**, likely a timing coincidence: the peak shopping week (Sept 19–24) was very dry, and the heaviest rain came with landfall, after stocking up had ended.
- **Sales came before the weather.** At every store, sales first jumped above normal on September 16 or 17, while wind and pressure mostly did not change until September 25 or 26, so sales led by 9–10 days. Even the earliest weather changes (wind at Kissimmee on the 21st, pressure at Miami and Fort Myers on the 20th) came at least 4 days after sales jumped. Rain only jumped above normal at 5 stores, so it is not a focus. Shoppers were likely reacting to forecasts and news, not to the weather they were seeing.
- **The lag correlations support this but show no single clear peak.** Wind turns positive after lag −2 and peaks at +4 days (+0.35), so high-sales days came before windy days. The negative-lag peaks for pressure (−6 days) and precipitation (−4 days) don't mean weather came first: they pair the surge days with the dry, high-pressure days before the storm, and the post-landfall sales drop with the storm's low pressure and heavy rain.
- **For Walmart:** local weather readings arrive too late to trigger restocking. Use storm forecasts instead, and stock Emergency, Breakfast, Entertainment, and Electronics at least 10 days before expected landfall. Sales should be expected to drop after landfall, so avoid over-ordering for those days.

**Caveats:** these results come from one storm over 30 days, so some correlations may reflect the timeline of the month rather than cause and effect. All stores share the same days and the same storm, so store-days are not independent observations. Most stores were far from the landfall site, so correlations may understate the response closer to landfall. The lags should be confirmed across more storms before being used for store decisions.

### Q4: Store-level geographic patterns

Each store's distance to Helene's *predicted* path was compared with surge size and timing. We use the forecast track from NHC Forecast Advisory 9 (Sept 25) to measure each store's distance from the predicted path. The forecast only gives a few points, so the notebook fills in 100 evenly spaced points between each pair and measures each store's distance (using the haversine formula) to the closest point on that line. Distance to the actual landfall point is also calculated for reference.

**Forecast track used (NHC Forecast Advisory 9):**

| Time (UTC) | Latitude (°N) | Longitude (°W) |
|---|---|---|
| Sept 25, 15:00 | 21.6 | 86.3 |
| Sept 26, 00:00 | 23.0 | 86.4 |
| Sept 26, 12:00 | 25.7 | 85.7 |
| Sept 27, 00:00 | 29.7 | 84.6 |
| Sept 27, 12:00 | 34.3 | 85.0 |

How the surge measures were defined:

- **Surge size:** percent increase in average daily units, surge period vs. baseline.
- **Surge onset:** the first day a store's sales exceeded its baseline average plus 2 standard deviations.
- **Peak day:** the day with the most units sold before landfall.
- **Outliers:** a store is flagged if its surge is more than 2 standard deviations from what its distance predicts (from a fitted line), if its surge size is more than 2 standard deviations from the average store, or if it never surged at all. The second check lets us still judge Tallahassee, which has no distance.

Results:

- Distance vs. % increase: Spearman r = **0.05** (essentially no relationship)
- Distance to actual landfall vs. % increase: Spearman r = **0.02**
- Distance vs. days of early warning: Spearman r = **−0.45**
- Outlier stores flagged: **Orlando** and **Miami**

**Store-level results** (sorted from closest to farthest from the predicted path):

| Store | City | Region | Miles to path | Miles to landfall | Baseline units/day | Surge units/day | % increase | Surge onset | Peak day | Transactions |
|---|---|---|---|---|---|---|---|---|---|---|
| FL-078 | Gainesville | North | 137 | 86 | 92.1 | 219.0 | 137.9% | Sept 16 | Sept 25 | 2,001 |
| FL-445 | St. Petersburg | West | 148 | 167 | 95.9 | 232.0 | 141.8% | Sept 16 | Sept 24 | 2,074 |
| FL-710 | Pensacola | Panhandle | 152 | 212 | 93.3 | 227.9 | 144.3% | Sept 16 | Sept 25 | 2,066 |
| FL-105 | Tampa | West | 155 | 160 | 91.8 | 231.7 | 152.5% | Sept 16 | Sept 25 | 2,043 |
| FL-412 | Jacksonville | North | 179 | 124 | 91.1 | 222.9 | 144.5% | Sept 16 | Sept 25 | 2,028 |
| FL-302 | Lakeland | Central | 184 | 172 | 98.0 | 239.4 | 144.3% | Sept 16 | Sept 25 | 2,123 |
| FL-239 | Orlando | Central | 208 | 173 | 94.4 | 209.6 | 122.1% | Sept 16 | Sept 24 | 2,010 |
| FL-287 | Kissimmee | Central | 211 | 182 | 91.0 | 220.1 | 141.9% | Sept 17 | Sept 25 | 2,053 |
| FL-501 | Fort Myers | West | 213 | 257 | 93.1 | 220.6 | 137.0% | Sept 16 | Sept 24 | 2,002 |
| FL-156 | Daytona Beach | East | 218 | 170 | 95.2 | 229.1 | 140.7% | Sept 17 | Sept 24 | 2,074 |
| FL-330 | Miami | South | 330 | 363 | 86.9 | 235.4 | 170.8% | Sept 16 | Sept 24 | 2,066 |
| FL-619 | Tallahassee | Panhandle | no coords | no coords | 86.4 | 224.1 | 159.6% | Sept 16 | Sept 24 | — |

**Outlier check** (line fitted to distance vs. % increase; residual z compares each store to the fitted line, surge-size z compares it to the average store):

| Store | City | Expected % increase | Residual z | Surge-size z | Outlier |
|---|---|---|---|---|---|
| FL-239 | Orlando | 144.97 | −2.26 | −1.87 | Yes |
| FL-105 | Tampa | 139.11 | 1.32 | 0.63 | No |
| FL-330 | Miami | 158.45 | 1.23 | 2.15 | Yes |
| FL-501 | Fort Myers | 145.52 | −0.84 | −0.64 | No |
| FL-710 | Pensacola | 138.78 | 0.54 | −0.04 | No |
| FL-156 | Daytona Beach | 146.07 | −0.54 | −0.34 | No |
| FL-445 | St. Petersburg | 138.34 | 0.35 | −0.24 | No |
| FL-287 | Kissimmee | 145.30 | −0.33 | −0.24 | No |
| FL-412 | Jacksonville | 141.76 | 0.27 | −0.02 | No |
| FL-302 | Lakeland | 142.32 | 0.20 | −0.04 | No |
| FL-078 | Gainesville | 137.12 | 0.07 | −0.57 | No |
| FL-619 | Tallahassee | n/a | n/a | 1.22 | No |

**Region summary:**

| Region | Stores | Avg miles to path | Avg % increase | Min % | Max % | Earliest onset (days before landfall) | Avg peak (days before landfall) |
|---|---|---|---|---|---|---|---|
| Panhandle | 2 | 152.0 | 151.9 | 144.3 | 159.6 | 10 | 1.5 |
| North | 2 | 158.0 | 141.2 | 137.9 | 144.5 | 10 | 1.0 |
| West | 3 | 172.0 | 143.8 | 137.0 | 152.5 | 10 | 1.7 |
| Central | 3 | 201.0 | 136.1 | 122.1 | 144.3 | 10 | 1.3 |
| East | 1 | 218.0 | 140.7 | 140.7 | 140.7 | 9 | 2.0 |
| South | 1 | 330.0 | 170.8 | 170.8 | 170.8 | 10 | 2.0 |

(The Panhandle's average distance uses only Pensacola because Tallahassee has no coordinates.) Surge multiples (surge-period units divided by baseline units) ranged from 2.22× (Orlando) to 2.71× (Miami). The notebook also draws a map of the stores, the predicted path, and the actual landfall; a bubble chart of distance vs. surge multiple (bubble size = number of transactions); a scatter of distance vs. surge start; small-multiple daily sales charts for every store; and a bar chart of each store's surge colored by region with outliers outlined. Gainesville had the fewest recorded transactions (2,001) and a surge of 2.4× its baseline.

**Interpretation:**

- Stores closer to the predicted path did **not** surge more. Surges ranged from +122% to +171%, and nine of the 12 stores fell between +137% and +153%. Gainesville, the closest store (137 miles), had one of the smaller surges (+138%), while Miami, the farthest (330 miles), had the largest (+171%). Using distance to the landfall point instead of the predicted path gave the same result (r = 0.02).
- Grouping stores by region showed the same pattern: every region's average surge fell between +136% and +152%, except South Florida (Miami only, +171%). Every region started surging 9–10 days before landfall and peaked 1–2 days before.
- Surge timing varied little across stores: the estimated start dates were only one day apart. Every store except Kissimmee and Daytona Beach began surging on September 16th (10 days before landfall); those two started on the 17th. Every store peaked on September 24th or 25th. The −0.45 correlation depends on those two later stores, which happen to be farther from the path, so it reflects a single one-day difference and isn't meaningful.
- The estimated surge start dates preceded the September 25th advisory used to measure distance to the forecast path. The per-store daily sales charts show nearly identical curves regardless of distance.
- **Orlando** had the weakest surge (+122%), below the roughly +145% predicted from its distance (residual z = −2.26, a borderline outlier just past the cutoff of 2), and the lowest surge-week volume of any store (209.6 units/day). Possible explanations include stock-outs, competition from other stores, or less concern because it is inland. **Miami** had the strongest surge (+171%) and was flagged by surge size (z = +2.15), not by its residual (z = +1.23), because as the farthest store it pulls the fitted line toward itself. Its surge may reflect its low baseline (86.9 units/day) or its coastal location. Tallahassee, excluded from the distance analysis, had the second-largest surge (+160%) and is the store closest to the actual landfall.
- **For Walmart:** large surges happened at both nearby and distant stores, so stock all stores, not only those near the predicted path.

**Caveats:** only 11 stores have distances and there is a single storm, so one store like Miami can shift the results. Sales reflect what was sold, not true demand, since stock-outs can hide it. Adding Tallahassee's coordinates is the most useful next step.

### Q5: Signal vs. artifact (board games)

The project prompt reports a 250% increase in Entertainment sales. This question tests whether that is a real demand signal or a data artifact (SKU miscoding, category lumping, or reporting lag), using SQL queries against the SQLite database alongside the cleaned transactions.

**Evidence:**

| Check | What we found |
|---|---|
| **A. Transaction-level breakdown** | In the raw receipts, average daily units rose **156.6%** for Monopoly (43.4 → 111.4/day) and **97.8%** for Playing Cards (59.2 → 117.1/day), baseline vs. surge. Entertainment overall rose **122.7%**. |
| **B. Database vs. transactions** | For Sept 19–25, the summary database reports **9,761.5** Entertainment units while the sampled transaction logs show **1,600** (780 Monopoly + 820 Playing Cards), a gap of **8,161.5** units. All **360 of 360** store-days mismatch, and every category has a gap of this kind (Breakfast's is the largest). A daily bar chart (database vs. transactions) shows the gap visually. |
| **C. Growth in each source** | Using daily averages, Entertainment grew **144.4%** in the database and **122.7%** in the receipts. Neither source reaches +250%, but both show sales more than doubling. |
| **D. Category lumping** | The database uses a single raw `Entertainment` label, but it doesn't show which products are grouped under it. |
| **E. SKU miscoding** | All valid transactions have mapped SKUs (0 unmapped rows). Four inventory SKUs (`UNKNOWN-99`, `LEGACY-PT`, `OLD-BATT-50`, `MISC-001`) have no product mapping, and their on-hand counts swing erratically (for example, `LEGACY-PT` jumps to 1,012 units on Sept 20). Their stock fell by 3,864 units during the surge, but inventory decreases are not confirmed sales, so we can't tell how much of the gap they explain. |
| **F. Reporting lag** | The database and receipt series match best with no time shift (r = 0.958), and no store-days are missing or zero, so there is no evidence of delayed or catch-up reporting. The database runs about 5.7× the receipts on average (6.1× during the surge, 5.5× outside it). |

**Supporting detail for the checks above:**

*Check A. Entertainment units by product in the receipts:*

| | Monopoly | Playing Cards |
|---|---|---|
| Baseline total units (14 days) | 608 | 829 |
| Surge total units (7 days) | 780 | 820 |
| Baseline units/day | 43.43 | 59.21 |
| Surge units/day | 111.43 | 117.14 |
| Change in units/day | +68.00 | +57.93 |
| % change | 156.58% | 97.83% |

*Check B. Entertainment units per day during the surge (database vs. receipts):*

| Date | Database units | Receipt units | Difference |
|---|---|---|---|
| Sept 19 | 783.0 | 141 | 642.0 |
| Sept 20 | 760.0 | 121 | 639.0 |
| Sept 21 | 1,370.0 | 205 | 1,165.0 |
| Sept 22 | 1,399.0 | 223 | 1,176.0 |
| Sept 23 | 1,282.5 | 236 | 1,046.5 |
| Sept 24 | 2,113.0 | 326 | 1,787.0 |
| Sept 25 | 2,054.0 | 348 | 1,706.0 |

The largest store-day gaps were Tallahassee on Sept 24 (303 database units vs. 25 receipt units), Miami on Sept 25 (267 vs. 19), and Lakeland on Sept 24 (262 vs. 30). None of the 360 store-days appears only in one source.

*Database vs. receipts by category (Sept 19–25), with growth from baseline in each source:*

| Category | Database units | Receipt units | Gap | Database growth | Receipt growth |
|---|---|---|---|---|---|
| Breakfast | 13,997.5 | 2,638 | 11,359.5 | 236.0% | 409.8% |
| Entertainment | 9,761.5 | 1,600 | 8,161.5 | 144.4% | 122.7% |
| Health | 8,770.0 | 665 | 8,105.0 | 117.3% | 124.3% |
| Alcohol | 8,928.0 | 972 | 7,956.0 | 114.9% | 125.5% |
| Electronics | 8,404.5 | 1,231 | 7,173.5 | 98.2% | 151.5% |
| Household | 8,392.5 | 1,234 | 7,158.5 | 103.7% | 106.5% |
| Baby | 6,490.5 | 936 | 5,554.5 | 53.9% | 56.9% |
| Snacks | 5,292.5 | 516 | 4,776.5 | 19.3% | 24.3% |
| Grocery | 6,645.5 | 2,146 | 4,499.5 | 62.0% | 77.9% |
| Emergency | 10,896.5 | 7,045 | 3,851.5 | 179.5% | 182.3% |

*Check D. Category lumping:* the database's `daily_sales_summary` table has 10 category labels with 360 rows each (12 stores × 30 days), and Entertainment appears under a single raw label (360 rows, 24,070 total units). No alternate spelling such as "Games & Entertainment" was found in the database, so no extra labels were merged into it during cleaning.

*Check E. Unmapped inventory SKUs:*

| SKU | Rows | Total units on hand reported (Sept) | Units depleted Sept 19–25 | Daily units on hand during the surge |
|---|---|---|---|---|
| `UNKNOWN-99` | 91 | 7,383 | 1,042 | 415, 899, 79, 57, 87, 95, 109 |
| `OLD-BATT-50` | 90 | 6,353 | 762 | 378, 58, 269, 135, 102, 86, 180 |
| `LEGACY-PT` | 90 | 6,965 | 1,355 | 252, 1,012, 58, 88, 117, 61, 31 |
| `MISC-001` | 84 | 6,244 | 705 | 251, 134, 69, 0, 148, 128, 57 |

Zero of the 24,521 valid transactions use an unmapped SKU, so the unmapped codes cannot explain the gap on the receipt side: real plus unmapped receipt units still equal 1,600, leaving the full 8,161.5-unit gap unexplained, and none of the 84 surge store-days reconciles exactly. The 3,864 units depleted equal about 47.3% of the gap, but inventory decreases are not confirmed sales, so that percentage does not show how much of the gap these SKUs explain.

*Check F. Reporting-lag tests:*

| Test | Result |
|---|---|
| Peak day | Database peaks Sept 24 (2,113 units); receipts peak Sept 25 (348 units). The database peaks one day *earlier*, not later, which argues against a simple delay. |
| Correlation at shifts of −3 to +3 days | 0.505, 0.618, 0.845, **0.958** (no shift), 0.688, 0.415, 0.275 |
| Missing or zero database store-days | 0 of 360 (12 stores × 30 days), so no gaps followed by catch-up spikes |
| Daily database-to-receipts ratio | Mean 5.65× (range 3.89× to 7.16×); 6.09× during the surge and 5.52× outside it |

Because the ratio is roughly constant across the whole month, the difference looks like a structural scaling gap between the two sources rather than a bug that appeared only during the hurricane.

**Our reading:** Entertainment sales really did rise before the storm, but the +250% figure is not supported by either source. The gap between the database and the receipts is real, but we could not establish its cause: it could be a reporting error, unmapped SKUs, or simply the fact that the transaction file is a sample.

**Recommendation:** Walmart should keep **normal stock levels** for board games and playing cards for the next storm rather than ordering against the +250% figure. Entertainment grew at a similar rate to Health and Alcohol, far less than Breakfast (+410%) and Emergency (+182%), so extra shelf space and budget should go to Emergency supplies and Breakfast items. Before increasing Entertainment orders, Walmart should check complete sales and inventory records to see whether stores ran out of board games or playing cards during the surge.

**Caveats:** the transaction file is a sample (roughly 2,000 transactions per store), so it will undercount absolute volume relative to the database. We did not establish whether stores ran out of stock, and the data can't tell us why shoppers bought more.

---

## Reflection Questions

The notebook ends with five written reflections. Short versions:

1. **Data quality impact.** The Pop-Tarts appeared under four different SKUs. Consolidating them through the shared mapping table showed 5,005 units sold and a 678-unit peak on Sept 25. Without it, looking at any single SKU would have badly underestimated demand.
2. **ETL trade-offs.** We left the 884 missing transaction timestamps as missing instead of filling them from neighboring rows. The rows are not in chronological order, so filling would have assigned wrong days and invented data we couldn't verify. The cost is that those transactions are excluded from daily totals.
3. **Pipeline trust.** We agree that the pipeline is a bigger risk than the model. The most fragile part was the high-level database summary vs. the raw receipts (9,761.5 vs. 1,600 Entertainment units). We found unmapped legacy SKUs but could not verify whether they caused the difference, and since the transaction file is a sample, its totals may differ from the database even without a reporting error. Pop-Tarts split across four SKUs was another fragile piece.
4. **Business vs. data.** We decided against increasing orders based solely on the reported +250% surge, since stocking up on the raw database figure could leave Walmart with too many games and take space from higher-priority products like Pop-Tarts and emergency supplies. Both sources still showed growth (122.7% in the sampled receipts), so with more time and complete sales records we would check whether a modest increase in board-game orders is justified, and audit the other categories to identify the unmapped SKUs and whether they affect category totals.
5. **Team collaboration.** We planned the division of work in our team charter, but ended up cleaning the data together to better understand each part. With another week, we would agree on shared column names and the SKU mapping earlier to spend less time reconciling code. The most valuable skills each member brought: Malia communicated her ideas clearly even when working individually, Anika organized meeting times and rooms that worked for everyone's schedules, and Clairine checked everyone's work to make sure it made sense from an outside perspective.

---

## Key Design Decisions

- **Raw data is never modified.** Every cleaning step works on a copy.
- **Flag, don't fill.** Missing and questionable values are flagged (`unit_cost_missing`, `quantity_invalid`, `coordinates_missing`, etc.) rather than guessed.
- **Assertions guard every merge.** Row counts and totals are checked after joins (and merges use `validate=` arguments) to catch silent duplication.
- **Averages, not totals, for period comparisons.** The baseline covers 14 days and the surge window covers 7, so daily averages keep them comparable.
- **Store-level baselines.** Stores differ in size, so Q3 and Q4 compare each store to its own normal.
- **Predicted path, not actual path.** Q4 measures distance to the forecast track because shoppers could only react to the forecast.
- **Keep original SKUs, plan with consolidated totals.** Original SKU codes stay in the data for record-level tracking, while the canonical SKU is used for demand analysis and ordering.

---

## Known Limitations

- **Missing timestamps:** 884 transactions have no timestamp and are excluded from date-based analyses.
- **Missing quantities:** 130 transactions have negative quantities that were set to missing and excluded from volume counts.
- **Tallahassee (FL-619):** No coordinates, so it is excluded from weather and distance analyses.
- **Unmapped inventory SKUs:** 355 inventory rows can't be tied to a verified product.
- **Flagged daily sales rows:** 76 fractional unit totals and 34 negative revenues were flagged but not corrected.
- **Time zones:** Sales timestamps have no time zone attached, so they're assumed to be store-local time.
- **Small sample per store:** Roughly 2,000 transactions per store, so store-level results carry uncertainty.
- **Single month of data:** September 2024 only, with no prior-year comparison.
- **Single storm:** Weather lags (Q3) and geographic patterns (Q4) come from one hurricane and should be validated on others.
- **Sales vs. demand:** Sales reflect what was sold, not true demand, since stock-outs can hide it.
- **Unexplained database gap:** The summary database reports about 5.7× the units in the sampled transaction file, and we could not establish whether the cause is a reporting error, the unmapped SKUs, or the transaction file being a sample.
- **Non-independent observations:** All stores share the same days and the same storm, so store-days are not independent observations in the weather correlations.
- **Unused tables:** The distribution center data was cleaned but not used in any of the five questions, and missing `unit_cost` values were flagged but never needed.

---

## Team and Contributions

Each team member contributed equally. We divided the analysis questions and graphs, reviewed each other's code and results, and wrote the code and explanations together.

| Name | Role / contributions |
|---|---|
| Malia Solomon | Weather API integration, part of the data cleaning, SKU consolidation, Questions 3 and 4 |
| Anika Shah | Part of the data cleaning, SKU consolidation, Question 5, reflection questions |
| Clairine Lou | Cleaned the larger datasets, organized the cleaning steps into a repeatable pipeline, SKU consolidation, Questions 1 and 2 |

---

## License

This project was created as coursework for DS 2002 and is not licensed for reuse or redistribution.

---

## Acknowledgments

- Weather data: [Open-Meteo](https://open-meteo.com/)
- Hurricane track: [National Hurricane Center](https://www.nhc.noaa.gov/)
- Course: DS 2002, [Professor Williamson / University of Virginia]
