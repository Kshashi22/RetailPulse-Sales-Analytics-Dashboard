# Olist Brazilian E-Commerce — Retail Sales Analysis

**An end-to-end analytics project: EDA → KPIs → RFM segmentation → BI dashboard.**
Built on the real [Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) — 99,441 orders, Sept 2016 → Aug 2018, across 9 relational tables (~120 MB).

Everything in this repo runs from the actual data. Every number below is reproducible from `analysis.py` / `sql/olist_kpis.sql`.

---

## 1. The Business Question

Olist is a Brazilian department-store marketplace connecting small sellers to the big platforms. The dataset covers ~100k orders from 2016 to 2018.

The question a Head of Growth would actually ask:

> *Revenue grew 8× in 20 months and then stopped. Why — and where is the next unit of growth?*

This project answers that in four passes: **what happened** (revenue trends, KPIs), **who** (customer segments via RFM), **what** (product/category concentration), and **why it's stalling** (delivery, retention, geography).

---

## 2. Repo Structure

```
olist_project/
├── README.md                          ← this file
├── build_db.py                        ← loads 9 CSVs → SQLite (olist.db)
├── analysis.py                        ← full pipeline: KPIs, RFM, 12 charts, CSV exports
├── sql/olist_kpis.sql                 ← 15+ documented KPI queries (SQLite/PG/BQ compatible)
├── notebook/olist_analysis.ipynb      ← narrated walkthrough, cell-by-cell
├── docs/POWER_BI_GUIDE.md             ← data model, DAX measures, page-by-page layout
├── docs/TABLEAU_GUIDE.md              ← equivalent Tableau build
├── charts/                            ← 12 publication-quality PNGs
├── exports/                           ← dashboard-ready CSVs (star schema)
└── kpis.json                          ← every headline KPI, machine-readable
```

---

## 3. Data Model & The Trap

Nine tables, three grains:

| Grain | Table | Rows | Key |
|---|---|---|---|
| **Order** | `olist_orders` | 99,441 | `order_id` |
| **Order line** | `olist_order_items` | 112,650 | `order_id` + `order_item_id` |
| **Payment record** | `olist_order_payments` | 103,886 | `order_id` + `payment_sequential` |
| Customer | `olist_customers` | 99,441 | `customer_id` → `customer_unique_id` |
| Product | `olist_products` | 32,951 | `product_id` |
| Seller | `olist_sellers` | 3,095 | `seller_id` |
| Review | `olist_order_reviews` | 99,224 | `review_id` |
| Geolocation | `olist_geolocation` | 1,000,163 | zip prefix |

> ⚠️ **The trap that breaks most Olist projects**
> `order_items` (112,650) and `order_payments` (103,886) are both **multi-row per order**. Joining them directly creates a cartesian fan-out — an order with 3 items and 2 payment records produces 6 rows, so revenue is counted 6× instead of once.
> **Every query here pre-aggregates both tables to `order_id` grain before joining.** This is why the SQL total (R$15,422,461.77) exactly matches the pandas total — the cross-check is in the notebook and in `Q2` of the SQL file.

**Revenue definition (stated once, used everywhere):**
`SUM(payment_value)` over `order_status = 'delivered'`. This is what the customer actually paid, freight included, counted once per order. Item-level **GMV** (`price + freight_value`) is used only for category/product/seller breakdowns, because payments cannot be allocated to a line item.

---

## 4. Cleaning Decisions (and why)

| Check | Result | Action |
|---|---|---|
| Duplicate `order_id` | 0 | — |
| Orders with no line items | 775 | Excluded from item-level analysis; kept for revenue |
| Orphaned item rows | 775 | Drop from joins |
| Delivered but no delivery date | 8 | Excluded from delivery stats only |
| Order status | 97.02% delivered | Filter `delivered` = the revenue-recognising population, not attrition |
| Product category in Portuguese | 74 categories | Joined to `product_category_name_translation` for English labels |
| Partial months | Sep–Dec 2016 (soft launch, <300 orders), Aug 2018 (stops on the 29th) | Trend charted on the clean window **Jan 2017 – Aug 2018** |

Also handled: `customer_id` is per-order, `customer_unique_id` is the real person — **all customer-level analysis uses `customer_unique_id`**, otherwise repeat buyers are counted as new customers. This is the second-most-common Olist mistake.

---

## 5. Headline KPIs

| KPI | Value |
|---|---|
| **Total revenue (delivered)** | **R$ 15,422,461.77** |
| Item GMV (price + freight) | R$ 15,424,463.65 |
| Freight revenue | R$ 2,202,965.54 (14.3% of GMV) |
| Delivered orders | 96,478 |
| Unique customers | 93,358 |
| **Average order value** | **R$ 159.86** |
| Median order value | R$ 105.28 |
| Revenue per customer | R$ 165.20 |
| Items per order | 1.14 |
| Average review score | 4.16 ★ |
| On-time delivery rate | 93.2% |
| Average delivery time | 12.1 days |
| Cancellation rate | 0.63% |
| **Repeat purchase rate** | **3.00%** (2,801 of 93,358) |
| Products / Sellers / Categories | 32,951 / 3,095 / 74 |
| Categories needed for 80% of GMV | **18 of 74 (24%)** |
| Top-5 state concentration | 73.2% of revenue |

---

## 6. What the Analysis Found

### 6.1 Revenue: 8× growth, then a plateau

| Month | Revenue | Orders | AOV |
|---|---|---|---|
| Jan 2017 | R$ 127,546 | 750 | R$ 170.06 |
| Nov 2017 (**Black Friday peak**) | **R$ 1,153,528** | 7,289 | R$ 158.26 |
| Dec 2017 (post-promo dip) | R$ 843,199 (−26.9% MoM) | 5,513 | R$ 152.95 |
| Mar 2018 | R$ 1,120,678 | 7,003 | R$ 160.03 |
| Apr 2018 | R$ 1,132,934 | 6,798 | R$ 166.66 |
| Aug 2018 | R$ 985,414 | 6,351 | R$ 155.16 |

Two things stand out. Nov 2017 spikes **+53.6% MoM** on Black Friday, then Dec gives back **−26.9%** — classic pull-forward, not a demand problem.

More important: **Mar–Aug 2018 is flat**, oscillating R$0.99M–1.13M with no trend, while AOV drifts *down* (R$170 → R$155). Growth came from order volume (750 → ~6,400/month), not basket size. **The acquisition engine worked and then saturated — and with a 3% repeat rate, nothing was retained to compound on.**

### 6.2 Geography: concentrated, and the periphery is served badly

| State | Revenue | Share | AOV | Late % | Review |
|---|---|---|---|---|---|
| **SP** São Paulo | R$ 5,770,266 | **37.4%** | R$ 142.48 | 4.5% | 4.25 |
| RJ Rio de Janeiro | R$ 2,055,690 | 13.3% | R$ 166.45 | **12.1%** | 3.97 |
| MG Minas Gerais | R$ 1,819,278 | 11.8% | R$ 160.23 | 4.6% | 4.19 |
| RS | R$ 861,802 | 5.6% | R$ 161.24 | 6.1% | 4.19 |
| PR | R$ 781,920 | 5.1% | R$ 158.83 | 4.0% | 4.24 |
| SC | R$ 595,208 | 3.9% | R$ 167.85 | 8.2% | 4.13 |
| **BA** Bahia | R$ 591,271 | 3.8% | **R$ 181.59** | **12.2%** | **3.93** |
| DF | R$ 346,146 | 2.2% | R$ 166.42 | 5.7% | 4.14 |
| GO | R$ 334,294 | 2.2% | R$ 170.82 | 6.5% | 4.11 |
| ES | R$ 317,683 | 2.1% | R$ 159.24 | 10.7% | 4.08 |

**SP alone is 37.4%; the top five are 73.2%.** But the interesting cell is the contradiction: **Bahia has the highest AOV of the top 10 (R$181.59) and the worst service (12.2% late, 3.93★).** RJ is the same story — 13.3% of revenue at 12.1% lateness. The Northeast wants to buy; logistics is the binding constraint, not demand.

### 6.3 Product: a long tail with a narrow head

| Category | GMV | Units | Avg price | Share |
|---|---|---|---|---|
| health_beauty | R$ 1,412,271 | 9,465 | R$ 130.28 | 9.2% |
| watches_gifts | R$ 1,264,333 | 5,859 | R$ 199.04 | 8.2% |
| bed_bath_table | R$ 1,225,209 | 10,953 | R$ 93.44 | 7.9% |
| sports_leisure | R$ 1,119,924 | 8,431 | R$ 113.25 | 7.3% |
| computers_accessories | R$ 1,032,724 | 7,644 | R$ 116.26 | 6.7% |
| furniture_decor | R$ 881,064 | 8,160 | R$ 87.25 | 5.7% |
| housewares | R$ 759,313 | 6,795 | R$ 90.60 | 4.9% |
| cool_stuff | R$ 691,681 | 3,718 | R$ 164.12 | 4.5% |
| auto | R$ 669,560 | 4,140 | R$ 139.85 | 4.3% |
| garden_tools | R$ 567,354 | 4,268 | R$ 110.24 | 3.7% |

**Pareto is sharp: 18 of 74 categories (24%) = 80% of GMV.** The top three are three different business models — bed_bath_table wins on volume at the lowest price, watches_gifts on price at half the volume.

No individual SKU matters: the #1 product is R$67,258, **0.44% of GMV**. Merchandising decisions belong at category level.

**Momentum (last 6 months vs prior 6):** housewares **+90.0%**, health_beauty +56.5%, baby +44.5%, auto +44.4%, watches_gifts +36.5%. Declining: computers **−54.9%**, toys −37.5%, cool_stuff −34.1%, stationery −18.7%. The growing categories are cheap, repeatable consumables; the declining ones are high-ticket discretionary — consistent with a plateauing, price-sensitive customer base.

### 6.4 RFM: the honest version

Olist is transactional, not subscription: **97.0% of customers buy exactly once.**

That has two consequences almost every portfolio project misses:

1. **`qcut(5)` on raw frequency produces duplicate bin edges and a fake F dimension.** Frequency must be bucketed by business logic (1 order / 2 / 3–4 / 5+), or rank-transformed before quintiling — this notebook does the former and documents why.
2. **The segmentation that matters is Recency × Monetary.** With F ≈ constant, RFM collapses into an R×M matrix: *how recently did they buy* × *how much are they worth*.

| Segment | Customers | Cust % | Revenue | Rev % | Avg recency | Avg spend | Avg review |
|---|---|---|---|---|---|---|---|
| Potential Loyalist | 21,888 | 23.4% | R$ 5,169,845 | **33.5%** | 91 days | R$ 236.20 | 4.17 |
| Cannot Lose Them | 13,739 | 14.7% | R$ 4,229,927 | **27.4%** | 394 days | R$ 307.88 | 4.12 |
| Promising | 18,105 | 19.4% | R$ 2,739,995 | 17.8% | 220 days | R$ 151.34 | 4.01 |
| Lost | 15,297 | 16.4% | R$ 852,563 | 5.5% | 396 days | R$ 55.73 | 4.23 |
| New Customers | 14,336 | 15.4% | R$ 783,163 | 5.1% | 90 days | R$ 54.63 | 4.25 |
| Hibernating | 7,192 | 7.7% | R$ 782,613 | 5.1% | 395 days | R$ 108.82 | 4.15 |
| Loyal Customers | 1,691 | 1.8% | R$ 507,958 | 3.3% | 135 days | R$ 300.39 | 4.21 |
| At Risk | 988 | 1.1% | R$ 288,814 | 1.9% | 382 days | R$ 292.32 | 4.25 |
| Champions | 122 | 0.1% | R$ 67,585 | 0.4% | 90 days | R$ 553.97 | 4.46 |

**Two segments carry 60.9% of revenue.** "Champions" is structurally tiny (122 people) — a real finding, not a bug, and it's the direct consequence of a 3% repeat rate. The actionable insight sits in **Cannot Lose Them: 13,739 customers worth R$308 each who have been silent for ~13 months.** That is the single highest-leverage cohort in the dataset, and it's invisible if you only look at revenue by month.

### 6.5 Operations: the strongest causal signal in the data

| Delivery vs estimate | Orders | % of orders | Avg review |
|---|---|---|---|
| Early | 88,644 | 91.9% | **4.30 ★** |
| On time | 1,292 | 1.3% | 4.04 ★ |
| 1–3 days late | 1,870 | 1.9% | 3.29 ★ |
| 4–7 days late | 1,802 | 1.9% | **2.11 ★** |
| 8–15 days late | 1,637 | 1.7% | **1.68 ★** |
| 15+ days late | 1,225 | 1.3% | 1.73 ★ |

Review score collapses monotonically with lateness — **4.30★ → 2.11★ at 4–7 days late.** A 2.2-point drop from a purely operational variable.

But note the volume: **91.9% of orders arrive early**, because Olist pads its delivery estimates. That padding is why CSAT holds at 4.16★ overall — and it's flattering the metric. Only 8,444 orders (8.8%) are actually late, and they own the entire 1★ tail.

### 6.6 Payments: installment financing is a real lever

Credit card 76.6% · boleto 19.9% · voucher 1.9% · debit card 1.5%.

| Installments | Orders | Avg order value |
|---|---|---|
| 1 (pay in full) | 46,811 | R$ 120.21 |
| 4 | 6,864 | R$ 164.72 |
| 6 | 3,792 | R$ 210.31 |
| 8 | 4,120 | R$ 308.28 |
| **10** | 5,140 | **R$ 414.00** |

AOV **triples** from 1 to 10 installments. Customers use *parcelamento* to afford bigger baskets — promoting installments on high-ticket categories lifts basket size without discounting. Boleto (19.9%) matters for the unbanked but clears in 1–3 days, which is a conversion cost worth quantifying.

---

## 7. Recommendations

| # | Finding | Recommendation | Est. impact |
|---|---|---|---|
| 1 | Revenue flat since Mar 2018 (~R$1.1M/mo) | Growth was acquisition-led and saturated. Reallocate to retention. | — |
| 2 | **97% buy once**, repeat rate 3.0% | Post-purchase lifecycle: day 7 review nudge, day 21 cross-sell in the same category | +1pp repeat ≈ **R$154k**/yr at current AOV |
| 3 | **Cannot Lose Them**: 13,739 customers, R$4.23M, silent ~13 months | Time-boxed win-back with a free-freight incentive | Recovering 5% ≈ **R$211k** |
| 4 | 18 of 74 categories = 80% GMV | Focus inventory, seller support and merchandising on the top 18; de-prioritise the tail | — |
| 5 | Late delivery: 4.30★ → 2.11★ | Fix RJ/BA carrier SLAs first — highest lateness, lowest reviews | Direct CSAT |
| 6 | SP 37.4%, top-5 73.2%; BA has highest AOV but 12.2% late | Logistics investment in the Northeast unlocks untapped high-value demand | — |
| 7 | AOV R$120 → R$414 across installment plans | Surface *parcelamento* on watches_gifts, computers, furniture | Basket uplift |
| 8 | Freight = 14.3% of GMV | Free-shipping thresholds are economically material — model them before scaling | Margin |

---

## 8. Limitations (say these in an interview — it reads as maturity)

- **Data ends Aug 2018.** Historical snapshot; Aug 2018 is partial (through the 29th) and 2016 is a soft launch.
- **No cost, margin or profit fields.** Everything is revenue/GMV. "Top category" ≠ "most profitable category."
- **No marketing spend or traffic data**, so CAC, ROAS and LTV:CAC are not computable. Retention is a census measure, not cohort-controlled.
- **Reviews are order-level, not product-level** — category satisfaction is a proxy.
- 775 orders lack line items and 8 delivered orders lack a delivery date. Excluded from the specific stats they would distort (each is counted in the cleaning section).
- RFM **frequency is degenerate by design** of the business model. The R×M view is reported alongside it rather than hiding the issue.

---

## 9. How to Run

```bash
pip install pandas numpy matplotlib seaborn jupyter
python build_db.py       # 9 CSVs → olist.db (needs the Kaggle files in data/)
python analysis.py       # KPIs, RFM, 12 charts → charts/, 8 CSVs → exports/
jupyter notebook notebook/olist_analysis.ipynb
```

**BI:** open `docs/POWER_BI_GUIDE.md` (model + 14 DAX measures + 4-page layout) or `docs/TABLEAU_GUIDE.md`. Import from `exports/`.

> **Note:** a `.pbix` file cannot be generated programmatically — it's a proprietary binary that only Power BI Desktop can write. This repo ships the **complete build guide, the star-schema CSVs, and all DAX measures** so the dashboard can be assembled in ~30 minutes. Same for `.twbx` (Tableau).

---

## 10. Interview Q&A — the questions this project invites

**"Why filter to delivered orders?"**
Delivered is where revenue is actually recognised; canceled/unavailable orders would inflate GMV with money never collected. 97% of orders are delivered, so it isn't attritional.

**"Why sum `payment_value` instead of `price + freight`?"**
They measure different things. `payment_value` is cash collected and reconciles to the P&L, but it can't be split across line items. `price + freight_value` attributes to a SKU, which is what category analysis needs. The two totals differ by R$2,002 (0.01%) — the gap is vouchers and rounding — which is itself a useful sanity check.

**"How did you avoid double-counting revenue?"**
Both `order_items` and `order_payments` are multi-row per order. Aggregating each to `order_id` before joining is mandatory; skipping it fans out the join and multiplies revenue. Verified by SQL↔pandas parity: R$15,422,461.77 from both paths.

**"customer_id or customer_unique_id?"**
`customer_id` is per-order. `customer_unique_id` is the person. Using the former makes every repeat buyer look like a new customer and would report a 0% repeat rate.

**"Your RFM frequency is all 1s — isn't the model broken?"**
The model is fine; the business is transactional. 97% order once, so quintiling raw frequency creates duplicate bin edges. I bucket by business logic and lead with the R×M matrix, which is where the decision value actually is.

**"What's the most surprising finding?"**
That 91.9% of orders arrive *early*, which is why CSAT looks healthy at 4.16★ — the padding hides an operational problem. The 8.8% that are late carry essentially all the 1★ reviews.

**"What would you do with 6 more months of data?"**
Cohort-controlled retention with marketing spend attached, so LTV can be computed properly rather than as a census average. That's the missing piece for any CAC/LTV decision.
