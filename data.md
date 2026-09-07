# Data

The original dataset used in this project is not included in this repository. It was provided for educational and research purposes and may be subject to licensing or redistribution restrictions.

## Required Data

To reproduce the analysis, users should provide daily OHLCV data for the ETF universe, together with daily VIX and MOVE index data.

The main asset dataset should contain at least the following fields:

| Column | Description |
|---|---|
| `ts_event` | Trading date or timestamp |
| `ticker` | Asset ticker symbol |
| `open` | Daily opening price |
| `high` | Daily highest price |
| `low` | Daily lowest price |
| `close` | Daily closing price |
| `volume` | Daily trading volume |

The VIX and MOVE datasets should contain a trading date and the corresponding daily index value.

## Expected Directory Structure

Place locally obtained data files in this directory using a structure similar to:

```text
data/
├── README.md
├── etf_ohlcv_data.csv
├── VIX_index_1d_2020_now.csv
└── MOVE_index_1d_2020_now.csv
```

Update the file paths in `LS_MA_Quant_Project.ipynb` to match the names and locations of your local files.

## Data Preparation

Before running the research pipeline, the data should be:

- sorted by `ticker` and `ts_event`;
- deduplicated at the ticker-date level;
- aligned to a consistent trading calendar;
- checked for missing or invalid prices;
- adjusted for stock splits and other relevant corporate actions;
- organized so that each row represents one asset on one trading date.

The project identified a major data-quality issue involving 2-for-1 share splits in several sector ETFs. Unadjusted price changes can create artificial returns and distort signals, prediction errors, covariance estimates, and backtest performance. Reproduction data should therefore use verified split-adjusted prices.

## Data Licensing

Users are responsible for obtaining data from an appropriately licensed public or commercial provider and for complying with that provider's terms of use.

Do not commit confidential, proprietary, credential-protected, or redistribution-restricted data to a public repository.

## Reproducibility Note

Because the original dataset is not distributed, reproduced numerical results may differ due to differences in:

- data vendors;
- adjusted-price methodology;
- corporate-action treatment;
- trading calendars;
- missing-value handling;
- observation timestamps.

The repository is intended to demonstrate the research methodology and implementation framework rather than distribute the underlying market data.
