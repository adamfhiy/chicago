# Chicago Taxi Trips

A Dataform pipeline over `bigquery-public-data.chicago_taxi_trips.taxi_trips`,
answering three questions about tipping, driver hours and public holidays.

**Dashboard:** [FILL]
**BigQuery project:** `chicago-taxi-cab`, dataset `dbo`
**Repo:** `github.com/adamfhiy/chicago`

---

## The data

The source holds 212,211,265 rows covering 2013-01-01 to 2023-12-31. Only
126,879,213 of those rows are distinct.

The extract is historical and no longer refreshed, so every date window is
anchored to `MAX(trip_date)` rather than `CURRENT_DATE()`.

## Models

| Model | What it does |
| --- | --- |
| `taxi_trips` | Declaration pointing at the public table |
| `stg_trips` | Cleaned and deduplicated, incremental with a 7 day lookback, partitioned monthly on `trip_date`, clustered on `trip_date` and `taxi_id` |
| `dim_us_holiday` | Holiday dates, federal and cultural, with `is_shift` flagging moved observances |
| `dim_date` | Calendar from min to max `trip_date`, left joined to holidays |
| `mart_taxi_trip_3month` | Q1 answer |
| `int_trip_shift_model_1` | Trip level. `LAG` to get the gap to the previous trip, flags shift starts |
| `int_taxi_shift_model_2` | One row per shift |
| `mart_overworked_taxis` | Q2 answer |
| `mart_shift_length_distribution` | Shift length buckets, supports the 12 hour threshold |
| `mart_q3_daily_trips` | Daily trip counts joined to `dim_date` |
| `mart_q3_holiday_comparison` | One row per holiday per year against a matched baseline |
| `mart_q3_holiday_summary` | Averaged across years, with spread |
| `mart_insight_payment_tipping` | Insight 1 |
| `mart_insight_idle_time` | Insight 2 |

Assertions run on `stg_trips` for uniqueness on `trip_key`.

Daily partitioning was not possible. Eleven years is 4,018 days against a 4,000
partition cap, so staging is partitioned monthly and clustered on date.

---

## Two things the source gets wrong

### `unique_key` is not unique

The column name claims uniqueness. 212,211,265 rows contain 126,879,213 distinct
keys, so 85,332,052 rows are duplicates. Forty percent of the table.

Staging deduplicates with `ROW_NUMBER()` and asserts uniqueness on the output.
Without that, every tip total and trip count would be roughly doubled.

**The source was republished mid-project.** An earlier reading of the same table
gave 213,111,447 rows and 211,655,459 distinct keys, a 0.7% duplicate rate. The
table was republished between builds and duplicates jumped from 1.5 million to
85 million.

The pipeline absorbed it without a code change. The dedupe removed the new
duplicates, the assertion still passed, and Q2 and Q3 results reproduced. All
figures in this README are from after the republish.

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
city, same cabs. That is a recording gap, not a behavioural one.

Prepaid cards behave like cash, so only Credit Card and Mobile are treated as
reliable.

---

## Q1: top 100 tip earners, last 3 months

Vehicles ranked by total tips over the last three months in the data, restricted
to Credit Card and Mobile payments.

**Assumption on wording.** The question says "tip earners" but clarifies "earn
more money than others". Tips are taken as the money in question, so the ranking
is on total tips. Total fare revenue is reported alongside so the list can be
re-ranked on gross earnings if the other reading was intended.

**Cash is excluded** because 69.5 million cash trips would enter the ranking
contributing nothing, turning the leaderboard into a measure of payment mix.

**Tip rate and tips per trip are reported alongside** the total. Total tips
tracks trip count closely, so the raw leaderboard is closer to "busiest
vehicles" than "best tipped". The rate columns make a quieter but better tipped
vehicle visible.

---

## Q2: top 100 overworkers

### Scope

**2023 only.** Trips per vehicle fell from 4,807 in 2014 to 1,908 in 2023.
Mixing years would rank vehicles on which year they operated in rather than on
how hard they worked. 2023 is the most recent full year and matches the Q1
window.

A 2023 overworker would look ordinary in 2014. The list is relative to its own
year.

Within 2023: 742,950 shifts across 3,405 vehicles.

### How shifts are derived

Sessionisation on trip gaps. A gap of 8 hours or more between consecutive trips
ends a shift, taken directly from the question. Every shift produced this way
therefore contains no 8 hour break by construction.

### Qualifying rules

| Rule | Value | Basis |
| --- | --- | --- |
| Break that ends a shift | 8 hours | Given in the question |
| Long shift | 12 hours | Changeover convention, confirmed below |
| Minimum shifts | 254 | Median shifts per vehicle |
| Share that must be long | 50% | Reading of "regularly" |
| Rank within qualifiers | Total hours | "work more hours than others" |

152 vehicles of 3,405 passed all rules, 4.5%. The top 100 by hours are reported.

### Why the long shift threshold is 12 hours

12 hours comes from the US taxi changeover convention, which is outside
knowledge. The distribution agrees with it independently:

| Shift length | Shifts | Trips per shift |
| --- | --- | --- |
| 00-02h | 58,237 | 1.8 |
| 02-04h | 59,408 | 3.9 |
| 04-06h | 93,848 | 5.4 |
| 06-08h | 120,679 | 7.1 |
| 08-10h | 133,880 | 8.9 |
| 10-12h | 127,114 | 10.9 |
| 12-14h | 86,447 | 12.6 |
| 14-16h | 35,268 | 13.7 |
| 16-18h | 7,145 | 13.7 |
| 18-20h | 1,667 | 13.6 |
| 20-22h | 958 | 13.5 |
| 22-24h | 870 | 13.9 |
| 24h+ | 17,429 | 28.4 |

593,166 shifts finish inside 12 hours, 79.8%. 149,784 run longer, 20.2%. The
threshold lands almost exactly on the longest fifth of shifts, so convention and
data agree on the same number.

Buckets are 2 hours wide so bar height reads as density. The final bucket is
open-ended, covering 24 to 113 hours, and is not comparable in width to the
others. No shift had a null length.

### Long hours are not the same as hard work

Trips per shift stops rising at about 14 hours. A 22 hour shift takes 13.5
fares, the same as a 15 hour one. Those extra hours produce nothing.

This is the 8 hour rule showing through. A vehicle that works a normal day,
parks for 6 hours, then takes one late fare is recorded as a single long shift
because the gap never reached 8 hours. Those vehicles were switched on, not
busy.

Ranking on hours therefore rewards idle time. Trips are reported alongside hours
in the output so the two can be separated.

### Caveat: this measures vehicles, not drivers

`taxi_id` is a medallion, not a driver.

17,429 shifts across 1,838 vehicles ran past 24 hours, and they averaged 28.4
trips each. That is roughly one fare per hour sustained for more than a day, the
same productivity as a normal shift. No single driver works that way.

The 24h+ bucket is also a jump rather than a tail: 17,429 shifts against 3,495
in the whole 18 to 24 hour range combined. The distribution does not taper into
it.

The longest shift found was 113 hours, with continuous trips across three days
and no gap over four hours.

These are shared vehicles handed between drivers without an 8 hour gap. The
answer measures vehicle utilisation. It remains the right starting point for a
fatigue audit, because a medallion running 4,182 hours has drivers behind it
whose individual hours nobody is tracking.

---

## Q3: do US public holidays change trip volume?

**Yes, and the direction is almost always down.**

### Method

Each holiday is compared against nearby days that fall on the same weekday, four
weeks either side. Other holidays are excluded from that baseline.

Matching on weekday removes three confounders at once: the weekday effect,
seasonality, and the long-term decline in ridership. What is left is the holiday
itself.

Holidays falling at a weekend appear twice, on the true date and the observed
date. `is_shift` flags the moved one and it is filtered out, so each holiday
counts once per year.

The reported figure is the average of the yearly percentages, which weights each
year equally, rather than the percentage of the pooled totals.

### Reliable results

Standard deviation under 15, so the average is meaningful.

| Holiday | % vs baseline | Stddev | Years |
| --- | --- | --- | --- |
| Christmas Day | -72.46 | 9.81 | 11 |
| Thanksgiving Day | -60.76 | 7.78 | 11 |
| Christmas Eve | -43.32 | 12.95 | 10 |
| Labor Day | -31.66 | 11.93 | 10 |
| Thanksgiving Eve | -26.96 | 11.11 | 11 |
| Martin Luther King Jr. Day | -19.32 | 8.97 | 11 |
| Presidents Day | -18.81 | 6.14 | 11 |
| Halloween | -10.40 | 12.75 | 11 |
| Veterans Day | -4.42 | 3.91 | 11 |
| Columbus Day | -2.73 | 8.59 | 11 |

### Results too volatile to average

Standard deviation above 15. These swing so widely between years that a mean
would report a number that never actually occurred, so the range is given
instead.

| Holiday | Worst year | Best year | Stddev | Years |
| --- | --- | --- | --- | --- |
| New Year's Eve | -34.05% | +88.83% | 34.49 | 10 |
| St Patrick's Day | -59.47% | +63.03% | 28.39 | 11 |
| New Year's Day | -57.19% | +21.44% | 25.22 | 11 |
| Independence Day | -53.23% | +11.17% | 21.50 | 11 |
| Memorial Day | -51.74% | +8.63% | 16.76 | 11 |
| Juneteenth | -24.21% | +7.64% | 16.23 | 3 |

The 15 threshold is a judgement call. It sits in a natural gap in the data, since
nothing falls between 12.95 and 16.23.

**Juneteenth has only 3 years** because it became a federal holiday in 2021. Its
figure is not comparable to holidays averaged across a decade.

### What the pattern means

**The size of the drop tracks how many offices actually close**, not whether the
day is officially federal. Christmas is federal and almost everything shuts, so
trips fall 72%. Columbus Day is equally federal but most private employers stay
open, so trips fall 3%. Halloween is not a federal holiday at all and still
drops 10%.

**Chicago taxi demand is commuting, not leisure.** If cabs were mainly for going
out, holidays would be busy. They are empty instead.

**The volatile holidays are all social occasions** where the weekday decides
everything. St Patrick's Day on a Saturday is a boom, on a Tuesday it is a
normal working day.

**Business value.** Federal holidays with tight variance can be planned on the
average, so supply can be cut confidently. New Year's Eve and St Patrick's Day
must be planned by weekday instead, because the holiday itself predicts nothing.

---

## Bonus: two further insights

### Insight 1: cash tips are absent from the record

The payment table above is the evidence. Nine in ten card passengers tip, one in
a thousand cash passengers does, in the same city and the same cabs.

**Business value.** Cash is 55% of trip volume. Any driver earnings, tipping or
service quality metric built on this table is measuring card trips only, so a
fleet ranking drivers on recorded tips is ranking them on payment mix. Either
restrict those metrics to card trips or model cash tips separately before using
them in driver incentives.

### Insight 2: idle time peaks when volume is lowest

The 90th percentile wait for the next fare peaks at 300 minutes at 5am against
75 minutes at 8am, on roughly a fifteenth of the trip volume.

**Business value.** A vehicle on the road at 5am faces a worst case five hour
wait against roughly one hour at 8am. Moving overnight capacity toward the 7am
to 9am ramp raises revenue per vehicle hour without adding a single car. It also
overlaps with the fatigue exposure in Q2, since the longest shifts are the ones
running through these dead hours.

---

## Limitations

- `taxi_id` is a medallion, not a driver
- Cash tips are unrecorded, so Q1 measures card tips only
- Q2 covers 2023 only and is not comparable across years
- 26,927 rows (0.01%) have no `trip_end_timestamp`. These are imputed as
  `trip_start + trip_seconds` rather than dropped, because a null end time
  returns a null gap in the shift logic, which reads as a new shift and would
  split one shift into several
- Timestamps are rounded to 15 minutes, producing small apparent overlaps
  between consecutive trips
- Shift length runs from first pickup to last dropoff, so it is a floor on hours
  worked, not a ceiling
- The stddev 15 cutoff separating reliable from volatile holidays is a judgement
  call, chosen because it sits in a natural gap in the data
- Not all trips are reported to the City, so the source is close to but not a
  complete census
