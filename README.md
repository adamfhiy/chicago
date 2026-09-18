# Chicago Taxi Trips

A Dataform pipeline over `bigquery-public-data.chicago_taxi_trips.taxi_trips`,
answering three questions about tipping, driver hours and public holidays, plus
two further insights.

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
| `mart_shift_length_distribution` | Shift length buckets, supports Q2's threshold and Q4b |
| `mart_q3_daily_trips` | Daily trip counts joined to `dim_date` |
| `mart_q3_holiday_comparison` | One row per holiday per year against a matched baseline |
| `mart_q3_holiday_summary` | Averaged across years, with spread |
| `mart_q4_insights_idle_time` | Q4a |

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
duplicates, the assertion still passed, and Q2 and Q3 reproduced. All figures in
this README are from after the republish.

### Cash tips are not recorded

| Payment type | % of all trips | % of trips tipped |
| --- | --- | --- |
| Credit Card | 39.93% | 94% |
| Mobile | 1.83% | 93% |
| Cash | 54.77% | 0% |
| Prcard | 1.57% | 6% |
| No Charge | 0.63% | 18% |
| Pcard | 0.01% | 7% |
| Way2ride | 0.00% | 100% |

Nine in ten card passengers tip. Effectively no cash passenger does. Same city,
same cabs. That is a recording gap, not a behavioural one.

Prepaid cards behave like cash, so only Credit Card and Mobile are treated as
reliable.

---

## Q1: top 100 tip earners, last 3 months

Vehicles ranked by total tips over the last three months in the data, restricted
to Credit Card and Mobile payments.

The top 100 took $406,165 in tips across 59,234 trips, at a 21.53% tip rate.

**Assumption on wording.** The question says "tip earners" but clarifies "earn
more money than others". Tips are taken as the money in question, so the ranking
is on total tips.

**Cash is excluded** because it is 54.77% of trips and records no tips at all.
Including it would rank vehicles on payment mix rather than on tipping.

**Tip rate and trips are reported alongside** the total, because total tips
tracks trip count closely. The raw leaderboard is therefore closer to "busiest
vehicles" than "best tipped", and the rate column makes a quieter but better
tipped vehicle visible.

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

152 vehicles of 3,405 passed all rules, 4.5%. The top 100 by total hours are
reported.

### The answer

The top three vehicles:

| Taxi ID | Hours | Shifts | Long shifts | Over 24 h | Avg shift |
| --- | --- | --- | --- | --- | --- |
| `6aefbdce` | 5,489.5 | 265 | 79.25% | 60 | 20.72 h |
| `d40dae7e` | 5,398.5 | 304 | 85.20% | 40 | 17.76 h |
| `6436b1ea` | 5,374.75 | 295 | 64.07% | 54 | 18.22 h |

Across the whole top 100: 31,230 shifts, averaging 13.65 hours each, of which
2,016 ran past 24 hours.

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
fatigue audit, because a medallion running 5,489 hours has drivers behind it
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

**Business value.** Holidays with tight variance can be planned on the average,
so supply can be cut confidently. New Year's Eve and St Patrick's Day must be
planned by weekday instead, because the holiday itself predicts nothing.

---

## Q4a: cabs wait longest when there is no one to pick up

Idle time is the gap between one trip ending and the next beginning, capped at
the 8 hour shift break since anything longer is a new shift rather than waiting.

| Hour | p90 wait | Median wait | Share of trips |
| --- | --- | --- | --- |
| 12AM | 165 min | 30 min | 1.79% |
| 1AM | 150 min | 30 min | 1.13% |
| 2AM | 135 min | 30 min | 0.68% |
| 3AM | 150 min | 30 min | 0.50% |
| 4AM | 255 min | 30 min | 0.59% |
| 5AM | 300 min | 30 min | 0.98% |
| 6AM | 195 min | 30 min | 1.82% |
| 7AM | 90 min | 15 min | 3.29% |
| 8AM | 75 min | 15 min | 4.87% |

Worst-case wait peaks at 300 minutes at 5AM and bottoms at 75 minutes at 8AM.
The 3AM to 5AM block carries 2.07% of all trips.

The morning block from 7AM to 10AM is the only window where median wait halves
to 15 minutes. The evening peak is busier but slower: 5PM carries the most trips
of any hour at 7.15%, yet its p90 wait is 135 minutes.

**Business value.** A cab working 3AM to 6AM chases roughly 2% of the day's
fares and can wait up to five hours between them. The same car in the 7AM to
10AM window waits 75 minutes and picks up five times the volume. Shifting
overnight shift starts two hours later raises revenue per vehicle hour without
adding a single car. It also cuts fatigue risk, since the longest shifts in Q2
run straight through these dead hours.

---

## Q4b: a shift stops earning after 14 hours

Trips per shift climb steadily with shift length, then stop. From the 14 to 16
hour band onward the figure sits at 13.5 to 13.9 and goes no higher.

| Shift band | Trips per shift | Gain over previous band |
| --- | --- | --- |
| 08-10h | 8.9 | +1.8 |
| 10-12h | 10.9 | +2.0 |
| 12-14h | 12.6 | +1.7 |
| 14-16h | 13.7 | +1.1 |
| 16-18h | 13.7 | 0.0 |
| 18-20h | 13.6 | -0.1 |
| 20-22h | 13.5 | -0.1 |
| 22-24h | 13.9 | +0.4 |

Up to 14 hours, two more hours on the road buys roughly two more fares. After
that the return is zero, and between 18 and 22 hours it is slightly negative.

Trips per hour tells the same story from the other side. It holds near 1.0
through 12 hours, then falls to 0.60 by hour 23.

This is an average rather than a hard cap. Individual shifts go above and below
it. The precise claim is that the average stops rising after 14 hours.

**Shifts over 24 hours are excluded** from this analysis. That bucket averages
28.4 trips, roughly double the ceiling, because those are shared medallions
rather than one person working through. Including them would hide the pattern.

**Business value.** There is a practical revenue cap per vehicle per shift and it
is reached at 14 hours. Every hour past that carries fuel, wear and fatigue risk
with no fare to offset it. Capping shifts at 14 to 16 hours would remove the
fatigue exposure at almost no revenue cost. Growth has to come from more
vehicles or better positioning, not longer shifts.

---

## Limitations

- `taxi_id` is a medallion, not a driver
- Cash tips are unrecorded, so Q1 measures card tips only
- Q2 and Q4b cover 2023 only and are not comparable across years
- 26,927 rows (0.01%) have no `trip_end_timestamp`. These are imputed as
  `trip_start + trip_seconds` rather than dropped, because a null end time
  returns a null gap in the shift logic, which reads as a new shift and would
  split one shift into several
- Timestamps are rounded to 15 minutes, producing small apparent overlaps
  between consecutive trips. This also explains why median idle time sits at
  exactly 15 or 30 minutes
- Shift length runs from first pickup to last dropoff, so it is a floor on hours
  worked, not a ceiling
- Idle time is capped at 8 hours by definition, since a longer gap ends the shift
- The stddev 15 cutoff separating reliable from volatile holidays is a judgement
  call, chosen because it sits in a natural gap in the data
- Not all trips are reported to the City, so the source is close to but not a
  complete census
