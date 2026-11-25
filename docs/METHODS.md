# Methods and decision log

This document records what the code does and why. It reports definitions and facts only; it does not interpret results.

## 1. Research design

- **Hypothesis (H1).** The target's 52-week high anchors the offer price in M&A: acquirers paying far above the high earn lower announcement returns.
- **Sample.** US completed deals announced 2005 to 2025 in which the target and the acquirer are both publicly listed.
- **Key variable.** `AnchorDistance = OfferPrice / Target52WkHigh - 1`.
- **Deciles.** Deals are sorted into ten groups by AnchorDistance. **Decile 1 = lowest** AnchorDistance (paid most below the high, "discount buyers"). **Decile 10 = highest** (paid most above the high, "premium payers").
- **Outcomes.** CAR(-5,+5) from a market model, and buy-and-hold abnormal returns (BHAR) over four consecutive quarters and over twelve months.

## 2. Pipeline

| Stage | Where | Output |
|---|---|---|
| A. WRDS build | `notebooks/01_wrds_data_build.ipynb` (requires WRDS) | Enriched SDC file; acquirer daily returns |
| B. Sample and deciles | `notebooks/02_analysis.ipynb` | `pctown`, `include_decile`, `anchor_decile` |
| C. Event study | `notebooks/02_analysis.ipynb` | CAR(-5,+5), quarterly and 12-month BHAR per deal |
| D. Inference and tables | `notebooks/02_analysis.ipynb` | `results/report_table_*.csv` |

## 3. Sample construction

1. **SDC query** (`SDC_QUERY` in notebook 01): announced 2005-01-01 to 2025-12-31; target nation and acquirer nation = United States; target and acquirer public status = `Public`; deal status = `Completed`.
2. **Duplicates.** A deal is the key (target CUSIP, acquirer CUSIP, announcement date). Repeated keys are flagged (`is_dup`) and never deleted; the first copy is used wherever deals are counted or merged.
3. **CRSP mapping.** Target and acquirer are matched to PERMNOs on the 6-character CUSIP, within the CRSP name-validity window at the announcement date; if that fails, the ultimate parent's CUSIP is tried. Deals with several candidates are flagged and resolved by a tie-breaker. `is_matched = y` when both sides have a final PERMNO.
4. **Third-party deals only.** `include_decile = n` when `tpermno_final == apermno_final` (target and acquirer are the same company). *Reason (2026-11-24):* SDC reports many self-deals, repurchases and restructurings with a "target" equal to the acquirer; their offer price is undefined.
5. **Full ownership only.** `include_decile = y` requires `pctown == 100`. A missing `pctown` gives `n`. *Reason (2026-11-24):* the offer price per share does not describe a control purchase when the acquirer ends with a partial stake.
6. **Valid AnchorDistance.** A deal needs a non-missing `anchor_distance` to enter the deciles.

Deciles are formed with `pandas.qcut` on the deals that pass steps 3 to 6 (equal-count bins, 10 groups).

**Not applied in this repository:** deal-size filters, exclusion of overlapping acquirer deals, exclusion of financial and utility targets, and the NYSE / AMEX / Nasdaq exchange check.

## 4. Offer price and AnchorDistance

- `offer_price = pr` (SDC offer price per share) where present, else `pr_filled`.
- `pr_filled = deal_value x 1000 / shrout`, with `deal_value` in $ millions and CRSP `shrout` in thousands of shares, using the last `shrout` on or before the announcement date.
- `target_52wk_high` = maximum CRSP daily price over [announcement - 400, announcement - 21] calendar days.
- **Limitation of `pr_filled`.** It treats `deal_value` as the price of 100% of the target's equity. Where SDC has no per-share price, `deal_value` is often a capped or partial amount (for example "Up to USD 400 mil" repurchase programs). In the final decile sample, 1,421 deals use the reported `pr` and 40 use `pr_filled`.

## 5. Event-study returns

- **Calendar.** The dates of the Fama-French daily file are the trading days. **t = 0** is the first trading day on or after `dateann`. No adjustment is made for after-close announcements.
- **Market return.** `(Mkt-RF + RF) / 100`, the CRSP value-weighted market return.
- **CAR(-5,+5).** Market model `R_i = alpha + beta x R_m`, OLS on trading days t-252 to t-31 (222 days). Abnormal return = `R_i - (alpha + beta x R_m)`, summed over t-5 to t+5 (11 days). A deal needs at least 100 non-missing estimation-window returns and all 11 event-window returns; otherwise it gets a status code and no CAR.
- **BHAR.** Acquirer compounded return minus market compounded return over the same trading days. Four quarters of 63 trading days after t = 0 (t+1..t+63, t+64..t+126, t+127..t+189, t+190..t+252) and the full 252 days. Each quarter is compounded on its own, so quarters do not add up to the 12-month BHAR. A deal needs all 252 acquirer returns; deals whose acquirer stops trading within the year have no BHAR.

## 6. Inference

- **Per-decile:** one-sample t-test of mean = 0 (two-sided), with the median and a winsorized (1/99) mean alongside.
- **Spreads** (D1 - D10, and D1-2 minus D9-10): Welch t-test; permutation p-value (10,000 label shuffles); bootstrap 95% interval of the mean difference (5,000 resamples); median and winsorized differences. Seed 0.
- **Multiple testing:** Holm-adjusted p-values across the four quarterly D1 - D10 BHAR tests only. The per-decile p-values are not adjusted.
- **Independence:** deals are treated as independent. Events that share calendar time share market moves, so the t-statistics may be overstated.
- **Benchmark:** the unmatched market return, not a size / book-to-market matched portfolio.

## 7. Decision log

| Date | Decision | Reason |
|---|---|---|
| 2026-11-21 | Environment: Python 3.14 virtual environment; pandas, numpy, JupyterLab | Needed for the analysis |
| 2026-11-23 | Decile orientation fixed: decile 1 = lowest AnchorDistance | Correction recorded by the author; decile 1 is the long leg |
| 2026-11-24 | Original SDC columns are never modified; added columns are the only change; duplicates are flagged, not deleted | Open-closed data rule |
| 2026-11-24 | Exclude deals where target PERMNO equals acquirer PERMNO | Same-company deals have no meaningful offer price (found by checking tickers and PERMNOs of decile-1 deals) |
| 2026-11-24 | Keep only `pctown == 100`; missing `pctown` counts as excluded | Partial-stake purchases do not reflect a per-share control price |
| 2026-11-24 | Write `include_decile` and `anchor_decile` into the SDC file in place; keep `pctown` as an SDC column | Derived columns may be overwritten; the 17 original WRDS columns are asserted unchanged on every run |
| 2026-11-24 | t = 0 is the first trading day on or after the announcement date | Convention; announcement times are not in the data |
| 2026-11-24 | Minimum 100 estimation-window returns and all 11 event-window returns for a CAR | Guards against thin or gappy estimation windows |
| 2026-11-24 | BHAR requires all 252 post-announcement returns | Keeps the same deals in every quarter |
| 2026-11-24 | Code lives in two notebooks (no package); the acquirer return files are no longer rewritten per decile | Deciles are attached from the SDC file at run time; output verified identical to the notebook version |
| 2026-11-25 | Dropped the unit tests and command-line entry point; all code is in the two notebooks | Simpler to read and run; the analysis output was verified byte-identical before and after |

## 8. Known limitations and open items

- `notebooks/01_wrds_data_build.ipynb` does not contain the steps that load CRSP `stocknames` and run the PERMNO tie-breaker; the enriched file already holds their output.
- The sample ends up with 1,461 deals in deciles; of these, 1,441 have a CAR and 1,432 have a full-year BHAR.
- The 29 deals without a full-year BHAR have at least one missing acquirer return in the 252-day window. The cause (for example a delisting) has not been checked, and no delisting adjustment is made.
- 34 repeated deal keys exist in the SDC file; the first copy is used.
- The calendar-time portfolio alpha (Fama-French five factors plus momentum) is planned and not implemented. The factor files are stored in `data/external/` for it.
