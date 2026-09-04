# TTC Station Reliability

Which subway stations are actually unreliable, and which just look bad because they're busy?

By raw delay count, Bloor-Yonge is worst. But it's an interchange — twice the trains, twice
the chances to log a delay. Divide by scheduled trips and it drops to 8th. So do the other
three interchanges. Nothing else moves more than two places.

The stations that stay bad are terminals: Vaughan Metropolitan Centre, Finch, Kennedy,
Kipling.

DuckDB + pandas in `explore.ipynb`, exported to CSV for Tableau.

## Data

| Source     | Coverage                                   | Use         |
|------------|--------------------------------------------|-------------|
| Delay data | 263,023 incidents, 2014-01-01 – 2026-06-30 | numerator   |
| GTFS feed  | summer 2026 service period                 | denominator |
| Ridership  | 1985–2019                                  | unused      |

Ridership would be the obvious denominator, but the series ends in 2019 — it misses COVID
and everything after.

The ten delay workbooks hold **61 sheets** — five are one sheet per month.
`pd.read_excel(path)` reads only the first one and says nothing, dropping 76,107 rows (29%),
mostly from 2017–2021. Use `sheet_name=None`.

## Method

Scheduled trips per station and line, from `stop_times` joined to `trips` on
`route_id IN (1,2,4)`. GTFS lists Bloor-Yonge as two stations ("Bloor" on Line 1, "Yonge" on
Line 2), merged here. 70 stations, 74 station-lines.

Interchanges are split by line, since the platforms are separate. The delay feed says which
line each incident was on, and `Bound` agrees with it ~99% of the time (Line 1 is
north-south, Lines 2 and 4 east-west).

`delay_rate = incidents / scheduled trips`, plus p50 and p95 delay minutes. Rows with
`Min Delay = 0` are dropped — 64% of the data, logged events with no service impact.

The rate is an index, not a probability. The numerator covers 8.5 years, the denominator one
service period, so values go above 1.0. Use it to compare stations, not on its own.

## Results

Interchanges merged back to station level:

| Station        | Raw | Adjusted | Change |  Trips |
|----------------|-----|----------|--------|--------|
| Spadina        |  13 |       32 |    −19 |  4,321 |
| Sheppard-Yonge |   9 |       24 |    −15 |  3,976 |
| Bloor-Yonge    |   1 |        8 |     −7 |  4,332 |
| St George      |   7 |       14 |     −7 |  4,326 |

Worst station-lines, 2018 onward:

| Station                     | Line | Index |
|-----------------------------|------|-------|
| Vaughan Metropolitan Centre |    1 |  1.41 |
| Finch                       |    1 |  1.31 |
| Kennedy                     |    2 |  1.28 |
| Kipling                     |    2 |  1.15 |
| Eglinton                    |    1 |  1.11 |
| Wilson                      |    1 |  1.02 |

Splitting interchanges also shows platforms differ: Bloor-Yonge's Line 1 side is 0.88
against 0.58 for Line 2. St George is 0.61 against 0.45. Reported as one station that gap
averages out.

The Line 1 extension opened December 2017, so in the full-range view those stations get a
full denominator against a partial window. Highway 407 moves 18 places between the two
views, Finch West 11. The 2018-onward numbers are the primary result.

## Data quality

2,178 distinct station spellings for 70 stations — `KENNEDY BD STATION`,
`KENNEDY SRT STATION`, `KENNEDY BD STATION (AP`. The top 74 cover 92% of rows, the top 150
98%, so the tail is nearly all one-offs and gets excluded rather than mapped.

```
263,023   raw
 -7,672   Line 3 (SRT), closed 2023, no GTFS denominator
 -1,282   segments ("UNION TO KING") and surface routes ("504 KING")
   -826   shuttle records and null Line
-15,186   no GTFS station: yards, carhouses, wyes, portals, line-level records
    -22   real station, line that doesn't serve it
─────────
238,035   analysed (90.5%)
```

Two ordering constraints. SRT has to go before the ` BD`/` SRT` suffix strip, or
`KENNEDY SRT` collapses into `KENNEDY` and inflates it ~20%. Segments have to go before the
split at `' STATION'`, which turns `UNION STATION TO KING` into `UNION`.

122 station-line pairs tag a line that doesn't serve the station — Kennedy is Line 2 only,
but 28 rows there say Line 1. At single-line stations the real line is knowable from the
station, so those are repaired: 626 rows recovered. At interchanges the field matters, so
the 22 bad rows there are dropped.

## Limitations

- One service period stands in for 8.5 years. Fine for comparing stations, not for reading
  the values directly.
- Trips aren't weighted by how often each service runs (weekday 29 days in the feed window,
  Sunday 6). Checked it — rank correlation 0.994, so it changes nothing.
- The confidence intervals are too narrow. Poisson assumes independence, but one incident
  generates several records.
- The metric counts incidents *logged at* a station, not caused by it. Terminals are where
  trains get held when something happens elsewhere, which may be why they top the list.
- ~250 rows carry a surface route number or a vehicle outside the 5000–6999 subway range.
  Only 8 have a non-zero delay, so they're left in.

## Running it

Needs `data/raw/delay/` and `data/raw/schedules/`. Run `explore.ipynb` top to bottom. Writes
`ttc.duckdb` and exports `output/station_final.csv` (per station-line, with coordinates) and
`output/rank_comparison.csv` (long format for a slope chart).

DuckDB tables persist across kernel restarts and shadow same-named DataFrames. If a number
won't change after an edit, check `SELECT table_name FROM duckdb_tables()` first.
