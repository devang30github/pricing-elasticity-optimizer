# Price Elasticity & Discount Optimizer

How much does demand respond when a retailer changes its prices, and what does that mean for revenue and margin? This project estimates price elasticity from two years of retail transactions, validates the estimate with independent methods, builds a demand forecasting model, and wraps everything in a Power BI report with a what-if price simulator.

**Stack:** Python (pandas, scikit-learn, XGBoost, statsmodels, pyfixest), Power BI (DAX, what-if parameters), Git.

<!-- Add screenshots: docs/overview.png, docs/simulator.png, docs/price_response_binned.png -->
![Price simulator](docs/price-simulator.png)

## Headline findings

- **Price elasticity is about -1.7** (95% CI -1.73 to -1.65): a 1% price cut raises units by roughly 1.7%. Three methods agree: fixed-effects regression (-1.73), Poisson regression (-1.92) and Double ML (-1.69).
- **Categories differ.** Stationery, paper/party and toys are the most price-sensitive (-1.9 to -2.2); jewellery and kitchen the least (-1.3 to -1.5).
- **Discounts raise volume and revenue but lower gross margin.** In the simulator, a 10% price cut lifts units about 20% and revenue about 7.6% but cuts gross margin about 18.5%. For a cut to be profit-neutral, elasticity must reach about 1 / margin rate. At a 31.5% baseline margin that is about -3.2, well beyond the estimates here (assumes unit cost = 60% of median price).
- **The ML forecast beats a trailing-average baseline by about 15%** at both product-week and category-week level, but the same ML models gave unstable price responses (-1.6 to -7.1) and were not used for pricing.
- **Price response is roughly linear for cuts down to about -20% and flattens beyond about -25%**, so the simulator is limited to +/-20%.

## Data and cleaning

Source: [Online Retail II](https://archive.ics.uci.edu/dataset/502/online+retail+ii) (UCI), a UK online gift retailer, about 1.07M invoice lines from Dec 2009 to Dec 2011.

| Step | Effect |
|---|---|
| Exact duplicates removed (overlap between the two sheets) | 34,335 rows |
| Cancellations, non-product codes (postage, fees, test codes), non-positive quantity or price removed | 1,003,214 rows kept (94.0%) |
| Orders later cancelled in full removed by capping quantity at the 99.9th percentile (500 units) | 948 rows |
| Line-level price errors (more than 5x or less than 0.2x the product's median price) | 3,045 rows |
| Weekly panel keeps product-weeks with one clear price (max/min price at most 1.10) | 58.1% of product-weeks |
| Products need 20+ weeks of sales, 3+ price levels and at least 3% price variation | **1,700 products, 66,694 product-weeks** |

Categories are derived from product descriptions with keyword rules (about 30% of products fall in "Other").

## Method

1. **Product-week panel.** Price is the weekly *median* line price rather than revenue / units, because wholesale and retail buyers pay different unit prices and bulk orders would otherwise create false elasticity.
2. **Elasticity.** `log(units)` on `log(price)` with product and week fixed effects, clustered by product. A naive regression without controls gives -0.66; adding the controls moves it to -1.73.
3. **Robustness.** Poisson fixed effects (-1.92); excluding Q4 (-1.77); controlling for order count (-1.35, treated as a lower bound because order count is itself an outcome of price).
4. **Double ML.** Gradient boosting predicts units and relative price (price vs. the product's own prior 8-week average) from history, seasonality and category momentum, using time-blocked cross-fitting. The residual-on-residual slope is the elasticity (-1.69). Controls explain only 0.6% of relative price, so this is close to a within-product regression with flexible controls.
5. **Functional form.** Binned residuals show a roughly linear response from -20% to +20%, flattening for deep cuts. Separate slopes for cuts (-1.85, SE 0.09) and hikes (-1.62, SE 0.12) within +/-20% are not statistically different.
6. **Demand forecasting.** Three expanding-window folds; XGBoost with a Tweedie objective and monotone constraints on price features.

## Results

### Elasticity by category (Double ML vs. fixed effects)

| Category | DML | 95% CI | Fixed effects |
|---|---|---|---|
| Stationery | -2.17 | -2.41 to -1.93 | -1.99 |
| Paper & Party | -1.97 | -2.15 to -1.78 | -1.95 |
| Toys & Kids | -1.94 | -2.28 to -1.59 | -2.23 |
| Christmas & Seasonal | -1.84 | -2.07 to -1.61 | -1.86 |
| Garden & Outdoor | -1.79 | -1.94 to -1.64 | -1.71 |
| Lighting & Candles | -1.72 | -1.82 to -1.62 | -1.80 |
| Other | -1.64 | -1.71 to -1.57 | -1.67 |
| Home Decor | -1.62 | -1.72 to -1.52 | -1.66 |
| Bags & Storage | -1.62 | -1.73 to -1.50 | -1.73 |
| Kitchen & Dining | -1.55 | -1.65 to -1.44 | -1.62 |
| Jewellery & Accessories | -1.31 | -1.55 to -1.07 | -1.35 |

Intervals overlap among the middle categories, so only the groupings (most vs. least sensitive) are claimed, not an exact ranking.

### Demand forecasting (average of 3 time-based folds)

| Model | Product-week WAPE | Category-week WAPE | Bias |
|---|---|---|---|
| Trailing 4-week mean (baseline) | 0.882 | 0.272 | -1.5% |
| XGBoost, log target, bias-corrected | 0.766 | 0.257 | +1.7% |
| **XGBoost, Tweedie, pricing + order-size features** | **0.750** | **0.229** | +2.3% |

A first version looked better (product-week WAPE 0.689) only because a log-trained model predicts medians: it under-forecast totals by 34%. Scoring models on equal terms (unbiased, multiple grains) removed that advantage.

### Why ML models were not used for price response

Price shifts of +/-5-10% applied to the ML models implied elasticities from -1.6 to -7.1, differing for cuts and hikes. Tree models absorbed promotion effects into price. The simulator therefore uses the regression-based elasticities.

## Power BI report

- **Overview:** revenue, units, average price and margin KPIs; monthly trend; revenue by category; top products.
- **Price simulator:** sliders for price change (+/-20%) and the cost ratio, a Low / Base / High sensitivity choice (ends of the DML confidence interval), a response curve across the whole range, and an impact-by-category view.

The simulator applies `units x (1 + price change) ^ elasticity` to every product-week in the sample. It is a what-if on historical data, not a forecast.

| Scenario (Base, 60% cost ratio) | Units | Revenue | Gross margin |
|---|---|---|---|
| Price -10% | +20.0% | +7.6% | -18.5% |
| Price +10% | -15.2% | -6.4% | +12.2% |

Baseline for these readings: £1.61M revenue and £507K gross margin in the analysis sample. The pattern holds in every category: a 10% discount lifts volume but loses gross margin, because even the most price-sensitive category (stationery, -2.2) is far from the break-even elasticity of about -3.2.

## Limitations

- **Observational data.** Unobserved promotions, emails and displays can make price cuts look more (or less) effective than they are.
- **Short-run response.** Elasticity is the reaction to temporary deviations from a product's usual price, not to a permanent list-price change.
- **Price jitter.** Weekly median prices change often; some of that is noise, which biases estimates toward zero.
- **Wholesale-heavy customers** buy in bulk and are likely less price-sensitive than retail shoppers.
- **Weeks with no sales are not in the panel**, so demand at high prices is somewhat overstated.
- **No cost data.** Margin results rest on an assumed unit cost of 60% of median price and swing strongly with that assumption.
- **Selected sample.** Totals come from 1,700 products with usable price variation, not the whole business.
- **One retailer, one market.** Results do not generalize beyond this business.

## Reproduce

1. Download Online Retail II and place the Excel file in `data/raw/`, then convert it to Parquet (see notebook 01).
2. Run the notebooks in `notebooks/` in order: `01_cleaning_eda`, `02_features`, `03_elasticity`, `04_demand_model`, `05_model_v2`, `06_dml_elasticity_final`.
3. Export `data/powerbi_panel.csv` and open `powerbi/pricing-elasticity.pbix`, pointing it at `data/powerbi_panel.csv` and `data/elasticity_final.csv`.

```bash
pip install -r requirements.txt
```

Raw data and large Parquet files are excluded from Git.
