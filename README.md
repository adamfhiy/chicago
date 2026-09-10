# Chicago Taxi Trips

A Dataform pipeline over `bigquery-public-data.chicago_taxi_trips.taxi_trips`,
answering three questions about tipping, driver hours and holidays.

**Dashboard:** [FILL]
**BigQuery project:** chicago-taxi-cab, dataset `dbo`

---

## The data

The source holds 212,211,265 rows covering 2013-01-01 to 2023-12-31. Only
126,879,213 of those rows are distinct. The extract is historical and no longer
refreshed, so every date window is anchored to `MAX(trip_date)` rather than
`CURRENT_DATE()`.

## Models

| Model | What it does |
| --- | --- |
| `taxi_trips` | Declaration pointing at the public table |
| `stg_trips` | Cleaned, deduplicated, incremental with a 7 day lookback |
| `dim_us_holiday` | Hardcoded holiday dates, federal and cultural |
| `dim_date` | Calendar joined to holidays |
| `mart_taxi_trip_3month` | Q1 |

Q2 and Q3 models pending in this repo.

---

## Two things the source gets wrong

### `unique_key` is not unique

The column name claims uniqueness. 212,211,265 rows contain 126,879,213 distinct
keys, so 85,332,052 rows are duplicates. Forty percent of the table.

Staging deduplicates with `ROW_NUMBER()` and asserts uniqueness on the output.
Without that, every tip total and trip count would be roughly doubled.

**The source changed mid-project.** An earlier reading of the same table gave
213,111,447 rows and 211,655,459 distinct keys, a 0.7% duplicate rate. The table
was republished at some point between builds, and duplicates jumped from 1.5
million to 85 million. The pipeline absorbed it without a code change: the dedupe
removed the new duplicates and the assertion still passed. Figures quoted for Q2
and Q3 below were measured before the republish and need re-verifying.

### Cash tips are not recorded

Measured on deduplicated staging:

| Payment type | Trips | % of trips tipped | Tip rate |
| --- | --- | --- | --- |
| Credit Card | 50,666,718 | 94.09% | 21.67% |
| Mobile | 2,328,046 | 93.14% | 19.84% |
| Cash | 69,488,171 | 0.10% | 0.04% |
| Prcard | 1,997,052 | 6.08% | 0.80% |
| Pcard | 12,337 | 6.70% | 2.72% |

Nine in ten card passengers tip. One in a thousand cash passengers does. Same
city, same cabs. That is a recording gap, not a behavioural one. Prepaid cards
behave like cash, so only Credit Card and Mobile are treated as reliable.

---

## Q1: top 100 tip earners, last 3 months

Vehicles ranked by total tips over the last three months in the data, restricted
to Credit Card and Mobile payments. Cash is excluded because 69.5 million cash
trips would enter the ranking contributing nothing.

Tip rate and tips per trip are reported alongside, because total tips tracks trip
count closely and the leaderboard is therefore closer to "busiest vehicles" than
"best tipped".

## Q2: top 100 overworkers

Shifts are derived by sessionisation. A gap of 8 hours or more between trips ends
a shift, taken from the question. Every shift produced this way therefore contains
no 8 hour break by construction.

| Rule | Value | Basis |
| --- | --- | --- |
| Break that ends a shift | 8 hours | Given in the question |
| Long shift | 12 hours | US taxi changeover convention; the longest 20% of shifts |
| Minimum shifts | 254 | Median shifts per vehicle |
| Share that must be long | 50% | Definition of "regularly" |

152 vehicles of 3,405 passed all three, 4.5%. *(Pre-republish figure.)*

Restricted to a single year. Trips per vehicle fell from 4,807 in 2014 to 1,908
in 2023, so mixing years compares vehicles operating in different conditions.

**Caveat with evidence.** `taxi_id` is a medallion, not a driver. Fleet wide,
2.33% of shifts ran over 24 hours; in the top 100 it was 13 to 23%. The longest
shift found was 113 hours, with continuous trips across three days and no gap
over four hours. That is a shared vehicle, not one person. The answer measures
vehicle utilisation. *(Pre-republish figures.)*

## Q3: do US holidays change trip volume?

Each holiday is compared against the same weekday within four weeks either side,
with other holidays excluded from the baseline. A flat average would confound
weekday, season, and the long decline in ridership.

*(Pre-republish figures.)*

| Holiday | Change vs baseline | Consistency |
| --- | --- | --- |
| Christmas Day | -71.8% | stddev 9.9 |
| Thanksgiving Day | -60.9% | stddev 7.5 |
| Christmas Eve | -43.2% | stddev 13.2 |
| Memorial Day | -37.3% | stddev 12.4 |
| Columbus Day | -2.5% | stddev 7.9 |
| New Years Eve | -2.3% | stddev 38.1 |
| St Patricks Day | +9.6% | stddev 28.2 |

Federal holidays suppress demand, and the size tracks how many offices actually
close rather than whether the day is officially a holiday. Columbus Day and
Veterans Day barely move because most people still work.

New Years Eve and St Patricks Day have standard deviations of 38 and 28, ranging
from roughly -60% to +89%, because the effect depends entirely on which weekday
they land on. No average is reported for those.

Holidays that fall on a weekend appear twice, on the true date and the observed
date. `is_shift` flags the moved one, and it is filtered out so each holiday
counts once per year.

---

## Limitations

- `taxi_id` is a medallion, not a driver
- Cash tips are unrecorded, so Q1 measures card tips only
- 26,927 rows (0.01%) have no `trip_end_timestamp`. These are imputed as
  `trip_start + trip_seconds` rather than dropped, because a null end time returns
  a null gap in the shift logic, which reads as a new shift and would split one
  shift into several
- Timestamps are rounded to 15 minutes, producing small apparent overlaps between
  consecutive trips
- Shift length runs from first pickup to last dropoff, so it is a floor on hours
  worked, not a ceiling
- Daily partitioning was not possible: 11 years is 4,018 days against a 4,000
  partition cap, so staging is partitioned monthly and clustered on date
- The source table was republished during the project and its contents changed
  substantially. Anything quoted here should be checked against a current run
