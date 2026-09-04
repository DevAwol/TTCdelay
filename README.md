# TTC Station Reliability

Which subway stations are actually unreliable, and which just look bad because they're busy?

By raw delay count Bloor-Yonge is worst. It's also an interchange, so twice as many trains
stop there. Dividing incidents by scheduled trips drops it to 8th, along with the other
three interchanges. Everything else moves at most two places.

After adjusting, the worst stations are terminals: Vaughan Metropolitan Centre, Finch,
Kennedy, Kipling.

DuckDB + pandas in `explore.ipynb`, exported to CSV for Tableau.

## Data

| Source     | Coverage                                   | Use         |
|------------|--------------------------------------------|-------------|
| Delay data | 263,023 incidents, 2014-01-01 – 2026-06-30 | numerator   |
| GTFS feed  | summer 2026 service period                 | denominator |
| Ridership  | 1985–2019                                  | unused      |

Ridership would be the obvious denominator but the series ends in 2019, so it misses COVID
and the service changes after it.

The ten delay workbooks contain 61 sheets — five of them are one sheet per month.
`pd.read_excel(path)` returns only the first sheet with no error or warning, which drops
76,107 rows (29%), mostly 2017–2021. `sheet_name=None` is required.

## Method

Scheduled trips per station and line, from `stop_times` joined to `trips` on
`route_id IN (1,2,4)`. GTFS lists Bloor-Yonge as two stations ("Bloor" on Line 1, "Yonge" on
Line 2), merged here. 70 stations, 74 station-lines.

Interchanges are split by line because the platforms are physically separate. The delay feed
records which line each incident was on, and `Bound` agrees with it about 99% of the time
(Line 1 runs north-south, Lines 2 and 4 east-west).

`delay_rate = incidents / scheduled trips`, with p50 and p95 delay minutes for severity.
Rows with `Min Delay = 0` are dropped, 64% of the data — logged events with no measurable
service impact.

The rate is a relative index. The numerator covers 8.5 years and the denominator one service
period, so values go above 1.0 and only mean something compared across stations.

## Results

Interchanges merged back to station level:

| Station        | Raw | Adjusted | Change |  Trips |
|----------------|----:|---------:|-------:|-------:|
| Spadina        |  13 |       32 |    −19 |  4,321 |
| Sheppard-Yonge |   9 |       24 |    −15 |  3,976 |
| Bloor-Yonge    |   1 |        8 |     −7 |  4,332 |
| St George      |   7 |       14 |     −7 |  4,326 |

Worst station-lines, 2018 onward:

| Station                     | Line | Index |
|-----------------------------|-----:|------:|
| Vaughan Metropolitan Centre |    1 |  1.41 |
| Finch                       |    1 |  1.31 |
| Kennedy                     |    2 |  1.28 |
| Kipling                     |    2 |  1.15 |
| Eglinton                    |    1 |  1.11 |
| Wilson                      |    1 |  1.02 |

Splitting interchanges also shows the platforms differ. Bloor-Yonge's Line 1 side is 0.88
against 0.58 for Line 2; St George is 0.61 against 0.45. Merged into one station that gap
averages out.

The Line 1 extension opened December 2017, so in the full-range view those stations carry a
full denominator against a partial window. Highway 407 moves 18 places between the two
views and Finch West 11, so the 2018-onward numbers are the primary result.

## Data quality

2,178 distinct station spellings for 70 stations, including `KENNEDY BD STATION`,
`KENNEDY SRT STATION` and `KENNEDY BD STATION (AP`. The top 74 cover 92% of rows and the top
150 cover 98%, so the tail is almost entirely one-offs and is excluded instead of mapped.

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

The filters have to run in that order. SRT goes before the ` BD`/` SRT` suffix strip,
otherwise `KENNEDY SRT` collapses into `KENNEDY` and inflates it by about 20%. Segments go
before the split at `' STATION'`, which turns `UNION STATION TO KING` into `UNION`.

122 station-line pairs tag a line that doesn't serve the station. Kennedy is Line 2 only but
28 rows there say Line 1. Where a station has one line the correct value comes from the
station, so those rows are repaired: 626 recovered. Interchanges have no such fallback, so
the 22 bad rows there are dropped.

## Limitations

- One service period stands in for 8.5 years, so the values only support comparison between
  stations.
- Trips aren't weighted by how often each service runs — a weekday pattern covers 29 days of
  the feed window, a Sunday one 6. Tested at 0.994 rank correlation, so it changes nothing
  here.
- The confidence intervals are too narrow. The bootstrap assumes Poisson counts, but one
  incident often generates several records, so the data is overdispersed.
- The metric counts incidents recorded at a station. Terminals are where trains get held
  when something happens elsewhere on the line, and turnbacks and crew changes concentrate
  there, which may be part of why they rank highest.
- About 250 rows carry a surface route number or a vehicle outside the 5000–6999 subway
  range. Eight have a non-zero delay, so they're left in.

## Running it

Needs `data/raw/delay/` and `data/raw/schedules/`. Run `explore.ipynb` top to bottom. It
writes `ttc.duckdb` and exports `output/station_final.csv` (one row per station-line, with
coordinates) and `output/rank_comparison.csv` (long format for a slope chart).

DuckDB tables persist across kernel restarts and take precedence over a same-named pandas
DataFrame. If a number won't change after an edit, check
`SELECT table_name FROM duckdb_tables()` before debugging the Python.
