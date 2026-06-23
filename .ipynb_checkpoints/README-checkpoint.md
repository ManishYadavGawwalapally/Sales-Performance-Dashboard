# Sales Performance Dashboard — KiranaPlus (Indian FMCG)

**Replace the VP's Monday PowerPoint with one screen.**

A single Tableau dashboard that answers every question the VP of Sales asks in the weekly review — by region, by product, by month — built on a 100,000-row FMCG sales dataset and designed to load in under three seconds on a standard laptop.

🔗 **Live dashboard:** https://public.tableau.com/app/profile/manish.yadav.gawwalapally/viz/INDIANSALESFMCGDASHBOARD/Dashboard1
📄 **Source workbook:** [`FMCG_SALES_DASHBOARD.twbx`](./FMCG_SALES_DASHBOARD.twbx)
📓 **Data prep notebook:** [`bigmart_practice.ipynb`](./bigmart_practice.ipynb)

---

## The problem

Every Monday morning, the VP of Sales opens four separate Excel files, copies numbers into a PowerPoint, and spends two hours assembling a picture that should take ten minutes to read. This has been the weekly routine for three years.

The goal of this project is to end that ritual: one dashboard that holds the entire review on a single screen, refreshes itself, and surfaces not just the headline totals but the signals hiding underneath them.

---

## What the dashboard reveals that the raw data hid

At the aggregate level the business looks calm. Revenue is split almost evenly across the ten states (roughly ₹25 Cr each over the period), the six product categories each hold about 16.7% of sales, and demand barely shifts across the days of the week. A VP scanning those totals would reasonably conclude that nothing needs attention — which is exactly the trap a four-file Excel scan sets. The dashboard's value is in what those averages conceal. Returns are not spread evenly: they cluster in high-value durables (Electric Appliances ≈ 11%, Furniture ≈ 9%) against barely 2% for Fast Food, and the most recent month shows the return rate jumping roughly 32% month-over-month even as revenue holds flat. Most importantly, three sub-categories — **Beds, Detergents, and Tables** — have now declined for three consecutive months, a slow bleed that is invisible in the flat category totals and would never surface in a manual scan. Those are precisely the signals the Monday review exists to catch, now surfaced on one screen instead of in two hours of copy-paste.

---

## Questions answered

| # | Business question | How it's built |
|---|---|---|
| 1 | Revenue this month vs last month, with green/red indicator | LOD month flags + MoM % and a direction field |
| 2 | Revenue by Indian state | Filled map (state geographic role) |
| 3 | Top 10 SKUs by revenue, filterable by region and category | Bar chart + Top N filter, region/category as context filters |
| 4 | Target vs actual by region | Bullet graph (actual bar + target reference line) |
| 5 | Monthly revenue trend (seasonality) | Continuous-month line over the full history |
| 6 | Revenue split by category | Treemap |
| 7 | Products in 3 consecutive months of decline | Nested table calc, flagged in a dedicated view |
| 8 | KPI scorecard: Revenue, Units, AOV, Return Rate (all MoM) | Four BAN cards with MoM deltas |
| 9 | Day-of-week sales pattern | Bar chart by weekday |
| 10 | All slicers connected | Global filters applied across every worksheet |

---

## Tech stack

- **Python** (pandas, numpy) — data cleaning and feature engineering
- **Jupyter Notebook** — EDA (missing-value analysis, duplicate checks, Sweetviz profiling)
- **Tableau** — data modelling, calculated fields, dashboard, and publishing
- **Tableau Public** — hosting the live, shareable link

---

## Repository contents

```
├── FMCG_SALES_DASHBOARD.twbx     # Packaged Tableau workbook (dashboard + data)
├── bigmart_practice.ipynb        # Data preparation notebook
├── fact_sales.csv                # Cleaned transaction-level fact table
├── targets.csv                   # Generated region × month revenue targets
└── README.md
```

---

## Data pipeline

The raw file is a synthetic 100,000-row FMCG sales export. Preparation (in the notebook) covers:

**Cleaning and standardisation.** Column names normalised to lowercase snake_case; descriptive renames applied (`Customer Name → customer_first_name`, `Last Name → customer_last_name`, `Sub-Category → product_sub_category`). The unreliable `year` and `sales_date` columns were dropped — they were randomised and inconsistent with the actual order dates — leaving `order_date` as the single source of truth for all time-based analysis.

**Region mapping.** The raw `region` field was geographically scrambled (every state mapped to all four regions), so it was rebuilt from `state` using a clean lookup: North (Delhi, Punjab, Uttar Pradesh), South (Karnataka, Tamil Nadu), East (West Bengal), West (Gujarat, Rajasthan, Maharashtra, Madhya Pradesh).

**Return flag.** The raw data has no return or order-status column, so `is_returned` is modelled synthetically — category-weighted (durables return more than perishables) and seeded for reproducibility, landing at an overall rate of about 5.7%.

**Targets table.** `targets.csv` is generated at region × month grain, derived from actual revenue scaled by a random factor, to give the Target-vs-Actual view a realistic comparison baseline.

---

## Data model

The dashboard uses a **relationship** (not a join) between the fact table and the targets table, which keeps each at its own grain and avoids the fan-out that would inflate target totals. The relationship matches on two clauses:

- `region = region`
- `DATE(DATETRUNC('month', [order_date])) = DATE([month_start])`

The `DATE()` wrappers align the data types on both sides so the month keys match cleanly.

---

## Selected calculated fields

```
Average Order Value   SUM([sales]) / COUNTD([order_id])

Return Rate           SUM([is_returned]) / COUNTD([order_id])

This Month Flag       DATETRUNC('month',[order_date]) = { MAX(DATETRUNC('month',[order_date])) }

Revenue MoM %         ([Revenue This Month] - [Revenue Last Month]) / [Revenue Last Month]

Performance vs target IF SUM([sales]) >= SUM([target_revenue]) THEN 'Beat' ELSE 'Miss' END

Monthly Decline       IF SUM([sales]) < LOOKUP(SUM([sales]), -1) THEN 1 ELSE 0 END

Decline Flag          IF WINDOW_SUM([Monthly Decline], -2, 0) = 3
                      THEN '⚠ Declining 3 months' ELSE 'OK' END
```

The KPI cards use the `This Month` / `Last Month` flag pattern to compute month-over-month deltas. Note that the Return Rate direction is deliberately inverted — a rising return rate is flagged red, not green. The Q7 decline view stacks two table calculations (`Monthly Decline` feeding `Decline Flag`), both computing along the month dimension, then filters to the latest month with `LAST() = 0` to show only sub-categories currently in a three-month slide.

---

## Notes and assumptions

- The dataset is **synthetic**. Across most dimensions its distribution is close to uniform, which is why several breakdowns (state revenue, category split, day-of-week) appear near-flat — that flatness is a property of the data, not a charting error, and the genuine signals live in returns and the decline detection.
- `is_returned` and `targets.csv` are **modelled fields**, not source data, as described above.
- The source has no city-level column, so geographic analysis is at **state** grain; time analysis is **monthly / daily** rather than weekly.
- Performance target on a standard laptop is met using a Tableau **extract** over the 100k-row dataset.

---

## How to reproduce

1. Run `bigmart_practice.ipynb` to regenerate `fact_sales.csv` and `targets.csv` from the raw export.
2. Open `FMCG_SALES_DASHBOARD.twbx` in Tableau (the packaged workbook already embeds the data).
3. To republish: **Server → Tableau Public → Save to Tableau Public As…**, then update the live link above.

---

