# How does anchoring on a target's 52-week high affect acquirer returns in M&A?

Code and results for a behavioral finance paper on **anchoring in mergers and acquisitions**.

**Hypothesis (H1).** The target's 52-week high anchors the offer price: acquirers paying far above it earn lower announcement returns.

**Measure.** `AnchorDistance = OfferPrice / Target52WkHigh - 1`. Deals are sorted into deciles of AnchorDistance (decile 1 = lowest, paid most below the high; decile 10 = highest, paid most above it) and compared on:

- CAR(-5,+5), a market-model cumulative abnormal return around the announcement;
- buy-and-hold abnormal returns (BHAR) for four consecutive quarters and for twelve months after the announcement.

**Sample.** US completed deals announced 2005 to 2025, target and acquirer both public, third-party (target and acquirer are different companies), acquirer ends with 100% of the target. 1,461 deals enter the deciles.

> **Paper:** [`paper/paper.md`](paper/paper.md)  
> **Author:** [name] · **Course / institution:** [add] · **Contact:** [add]

## Repository layout

```
.
├── notebooks/
│   ├── 01_wrds_data_build.ipynb   # builds the data from WRDS (needs a WRDS account)
│   └── 02_analysis.ipynb          # deciles, CAR, BHAR, tests, report tables (with outputs)
├── data/                          # NOT tracked: licensed data goes here (see Data sources)
├── results/                       # aggregate output tables (CSV)
├── docs/METHODS.md                # definitions, sample rules, decision log, limitations
├── paper/                         # the final paper (paper.md) and its figures
└── requirements.txt
```

## Quick start

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
jupyter lab
```

Put the data files listed below in `data/`, then run `notebooks/02_analysis.ipynb` from top to bottom. It writes the tables in `results/`. Notebook 01 is a record of how the WRDS data was built; you only need it to rebuild the data from scratch (`pip install wrds`).

Tested with Python 3.14, pandas 3.1, numpy 2.5, scipy 1.18.

## Data sources

SDC and CRSP are licensed through [WRDS](https://wrds-www.wharton.upenn.edu/), so **no data is included** in this repository, and neither are deal-level files derived from it. Everything under `data/raw/`, `interim/`, `processed/` and `external/` is ignored by git. Only aggregate tables are published, in `results/`.

| File | Folder | Source | How it is produced |
|---|---|---|---|
| `sdc_us_public_2005_2025_pct.csv` | `raw/` | WRDS, SDC Mergers & Acquisitions (`tr_sdc_ma.wrds_ma_details`) | The `SDC_QUERY` in notebook 01: US target and acquirer, both public, completed, announced 2005-01-01 to 2025-12-31. Carries `pctacq` and `pctown`. |
| `sdc_us_public_2005_2025.csv` | `raw/` | Same query without `pctacq` / `pctown` | Earlier pull; the enriched file below was built from it. |
| `sdc_pr_52w_high_decile.csv` | `interim/` | SDC + CRSP | Enriched deal file built by notebook 01: CRSP PERMNO mapping, target 52-week high, offer price, AnchorDistance. Notebook 02 adds `pctown`, `include_decile` and `anchor_decile`. |
| `acquirer_decile_1.csv` ... `acquirer_decile_10.csv` | `interim/` | CRSP daily stock file (`crsp.dsf_v2`) | Acquirer daily returns over [announcement - 400, announcement + 400] calendar days, built by notebook 01. The decile stored inside these files is ignored; deciles always come from the SDC file. |
| `F-F_Research_Data_5_Factors_2x3_daily.csv` | `external/` | [Kenneth French Data Library](https://mba.tuck.dartmouth.edu/pages/faculty/ken.french/data_library.html) | Download "Fama/French 5 Factors (2x3) [Daily]" (CSV) and unzip. Used for the market return and the trading calendar. |

The monthly 5-factor file and the daily and monthly momentum files are kept in `external/` for the planned calendar-time alpha but are not used yet.

**Columns of the enriched SDC file.** Original SDC columns (never modified): `master_cusip` (target CUSIP), `acusip` (acquirer CUSIP), `dateann`, `dateannorig`, `amv`, `status`, `aup`, `tup`, `anation`, `tnation`, `tpublic`, `apublic`, `tticker`, `deal_value` ($ millions), `pr` (offer price per share), `consid`, `considoff`. Added by notebook 01: 6-character CUSIP keys, `is_dup`, CRSP PERMNOs (`tpermno*`, `apermno*`), `is_matched`, `target_52wk_high`, `pr_filled`, `offer_price`, `anchor_distance`. Added by notebook 02: `pctown`, `include_decile`, `anchor_decile`. A deal is one (`master_cusip`, `acusip`, `dateann`) key; 34 keys appear twice and the first copy is used.

## What the analysis does

1. **Build the data** (notebook 01, needs WRDS): map deals to CRSP PERMNOs, compute the target's 52-week high, fill missing offer prices, compute AnchorDistance, pull acquirer daily returns.
2. **Form the sample and deciles** (notebook 02): keep third-party deals with `pctown == 100` and form AnchorDistance deciles. The original SDC columns are never modified; the notebook asserts this.
3. **Event-study returns**: CAR(-5,+5) from a market model fitted on t-252 to t-31 trading days; BHAR over four 63-day quarters and 252 days, benchmarked to the market.
4. **Tests and tables**: per-decile t-tests, D1 - D10 spread tests (Welch, permutation, bootstrap, Holm across the four quarters), medians and winsorized means.

Full definitions, every sample rule, and the dated decision log are in [`docs/METHODS.md`](docs/METHODS.md).

## Outputs

| File | Contents |
|---|---|
| `results/report_table_sample.csv` | Deals per decile, with a CAR, with a full 12-month window |
| `results/report_table_by_decile.csv` | Per decile and period: n, mean, median, sd, winsorized mean, t, p |
| `results/report_table_by_decile_wide.csv` | The same means as `mean% (t)` with significance stars |
| `results/report_table_spreads.csv` | D1 - D10 and D1-2 minus D9-10 spread tests |

The analysis is deterministic: permutation and bootstrap draws use a fixed seed (`SEED` in the notebook's settings cell).

## Limitations

- The market return is the benchmark, not a size / book-to-market matched portfolio.
- Deals are treated as independent in the t-tests, and the per-decile p-values are not adjusted for the number of tests.
- Where SDC reports no per-share price, the offer price is filled as `deal_value x 1000 / shrout`, which assumes `deal_value` prices 100% of the equity. 40 of the 1,461 decile deals use this fill.
- No deal-size, overlapping-deal, industry (financials / utilities) or exchange filter is applied.
- Notebook 01 omits the CRSP `stocknames` load and the PERMNO tie-breaker; the enriched data file already contains their output.
- A calendar-time Fama-French five-factor plus momentum alpha is planned and not implemented.

## Citation and license

[Add citation and license. No license file is included yet, so by default all rights are reserved.]
