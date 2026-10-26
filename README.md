# E-commerce Delivery Operations & Seller SLA Analytics

**Business question:** a marketplace promises a delivery date at checkout. This project asks:
- Which orders miss that promise?
- Where in the order-to-doorstep journey do they lose time?
- Which routes and which sellers drive the misses?
- Can the operations team see a late order coming?

**Data:** the [Olist](https://github.com/olist/work-at-olist-data) public dataset. It has 99,441 orders from a Brazilian e-commerce marketplace, spread across six tables: orders, items, products, customers, sellers and reviews. After data-quality checks, **94,819 delivered orders** from **2,924 sellers** are analysed, covering purchases from Jan 2017 to Aug 2018.

**Stack:** SQL (SQLite), Python (pandas, scikit-learn), Matplotlib

---

## Key findings

| # | Finding | Evidence |
|---|---------|----------|
| 1 | **6.9% of delivered orders (6,506) arrived after the promised date.** This happened even though promises averaged **23.7 days** against an actual **12.6 days**. | `sql/01_order_fact.sql` |
| 2 | **Lateness destroys ratings.** Late orders averaged **2.27 stars vs 4.29** (gap 2.0, bootstrapped 95% CI 1.98 to 2.06). **54%** of late orders got 1 star, against 6.6% of on-time orders (**8.2x**). | Chart 5 |
| 3 | **Late orders lose their time in carrier transit.** They took 33.9 days against 11.0. **87%** of the extra 22.9 days was carrier transit and 13% was seller dispatch. | Chart 1 |
| 4 | **Routes matter.** Orders to **Rio de Janeiro + the Northeast** are **22% of volume but 40% of late deliveries**: a 12.5% late rate, against 4.6% for São Paulo. | `sql/03_state_scorecard.sql`, Chart 3 |
| 5 | **Seller dispatch SLA.** **9.0%** of orders were handed to the carrier after the seller's dispatch deadline. They arrived late **21.2%** of the time, against **5.4%** for on-time handovers (**3.9x**). Compared with compliant orders on the same route in the same month, breaches account for about **1,300 extra late orders (≈20% of all late deliveries)**. | `sql/04_seller_scorecard.sql` |
| 6 | **The problem is concentrated.** **82 sellers (2.8%) caused half of all dispatch-SLA breaches.** They handle 31% of orders and breach on 14.8% of them, against 6.5% for all other sellers. | Chart 4, `reports/seller_watchlist.csv` |
| 7 | **Peaks and disruptions.** In the **Black Friday month (Nov 2017), orders rose 63%** over October and the **late rate tripled** (4.2% to 12.4%), while seller breaches rose from 7.9% to 12.9%. The Feb–Mar 2018 spike (14.2% and 19.0% late) was different: carrier transit time rose 26% against January, while breaches did not rise in March. | `sql/02_monthly_scorecard.sql`, Chart 2 |
| 8 | **Late orders can be seen coming.** An early-warning score at carrier handoff was trained on 2017 orders and tested on 2018 orders (AUC 0.74). It **flags 10% of orders and catches 34% of late deliveries** at 27% precision, which is **3.4x the 7.9% base rate**. A spreadsheet rule (days of slack left vs the route's P90 transit time) catches 31%. | Chart 6 |

![Where late orders lose time](reports/figures/01_where_late_orders_lose_time.png)

---

## What I would do about it: a 5-part ops program

| Initiative | What to do | Track | Expected effect |
|---|---|---|---|
| **1. Seller dispatch-SLA program** | Publish a weekly seller scorecard. Start with the 82-seller watchlist: fix pickup slots and packing capacity, then warn and penalise repeat breaches. | Breach rate (9.0% today), seller late rate | Up to ~20% fewer late deliveries if breached orders behave like compliant ones on the same route. This is an upper bound; test it with a pilot on half the watchlist first. |
| **2. Carrier and route review: RJ + Northeast** | Review carrier SLAs and carrier mix on these lanes, and add route-level buffers to the promised date. | Lane late rate (12.5% today) | These lanes carry 40% of late orders. |
| **3. Peak-season playbook** | Before peak, get capacity commitments from top sellers and carriers, and add a temporary promise buffer in peak weeks. | Peak-month late rate (12.4% last Black Friday) | Avoids a repeat of the 3x jump. |
| **4. Proactive customer alerts** | Each day, list the riskiest 10% of orders handed to carriers. Message those customers or expedite their orders before the promise is missed. | Share of late orders caught (34%), ratings of late orders | Targets a third of late deliveries with a tenth of the effort. |
| **5. Event-data quality** | Fix the logging behind **1,373 orders with impossible timestamps**, such as a carrier handoff recorded before payment approval. | Share of orders with out-of-order timestamps | Cleaner SLA measurement. |

---

## Method

### 1. SQL order fact table (`sql/`)
- **`01_order_fact.sql`** joins the six raw tables into one row per delivered order. For each order it records:
  - stage durations: payment approval, seller dispatch and carrier transit
  - the seller's dispatch deadline and whether it was breached
  - the delivery promise and whether the order was late
  - a data-quality flag
- **Data-quality filter:** from the 96,478 delivered orders, the analysis drops:
  - 1,373 orders with out-of-order timestamps
  - 23 orders with missing timestamps
  - 263 orders from the platform's sparse launch months in 2016
- **`02`, `03` and `04`** produce the monthly, state and seller scorecards (`reports/*.csv`).

### 2. Lead-time breakdown
Each stage's average is compared between late and on-time orders, to see where the extra days come from.

### 3. Routes and sellers
- **State scorecard:** compares each state's share of orders with its share of late deliveries.
- **Seller Pareto:** ranks sellers by number of breaches.
- **Breach impact estimate:** each breached order is compared with compliant orders on the same seller-state to customer-state route in the same month. If a route-month has fewer than 20 orders, the comparison falls back to customer-state x month, then to month.

### 4. Monthly scorecard
- Volume, late %, breach % and average stage times are tracked by purchase month.

![Monthly](reports/figures/02_monthly_volume_and_late_rate.png)

### 5. Early-warning score at carrier handoff
- **Inputs:** only what is known when the parcel reaches the carrier:
  - days of slack left before the promised date
  - dispatch delay and breach margin
  - route history from 2017 (median and P90 transit, late rate)
  - seller history from 2017 (late and breach rates, shrunk towards the average for small sellers)
  - origin and destination state, freight, value, item count, weight, weekday
- **Time-based validation:** trained on orders purchased in 2017 and tested on orders purchased in 2018.

| Model (test: 51,510 orders from 2018) | ROC-AUC | Top 10% precision | Late orders caught by top 10% |
|---|---|---|---|
| Logistic regression | **0.743** | 27.0% | **34.3%** |
| Gradient boosting | 0.730 | 27.0% | 34.3% |
| Rule: slack vs route P90 transit | 0.728 | 24.3% | 30.8% |

- Logistic regression is kept because it is as good as gradient boosting and easier to explain.
- The simple rule gets most of the value, which makes it a good first version for an ops team.

![Early warning](reports/figures/06_early_warning_capture.png)

---

## Assumptions and limitations
- **The data is observational.** The seller-breach impact estimate is adjusted only for route and month. Sellers who breach may differ in other ways, so treat the 20% as an upper bound to test.
- **"Late"** means delivered after the estimated date the customer saw at purchase.
- **Multi-seller orders** (1.3% of orders) are attributed to one seller.
- **The test year was harder than the training year.** 2018 had a 7.9% late rate against 5.7% in 2017, mainly because of the Feb–Mar disruption. The early-warning score may perform differently in calm months. All route and seller history features use 2017 data only.
- **This is a Brazilian marketplace.** The structure of the findings (where time is lost, how concentrated the vendor problems are) transfers better than the exact numbers.

## Run it
```bash
pip install -r requirements.txt
python src/delivery_ops_analysis.py     # ~15 seconds; writes reports/ and reports/figures/
```

## Repository layout
```
data/                      Olist tables used (orders, items, products, customers, sellers, reviews) + licence
sql/01_order_fact.sql      cleaning, joins, stage durations, SLA and lateness flags
sql/02-04_*.sql            monthly, state and seller scorecards
src/delivery_ops_analysis.py   lead-time breakdown, route/seller analysis, surges, early-warning score, charts
reports/metrics.json       every number quoted above
reports/*.csv              scorecards, seller watchlist, model coefficients
reports/figures/*.png      charts
```

Data: Olist, *work-at-olist-data* (MIT licence, see `data/LICENSE-olist-data.txt`).
