# Chicago Taxi Trips

A Dataform pipeline over `bigquery-public-data.chicago_taxi_trips.taxi_trips`,
answering three questions about tipping, driver hours and public holidays, plus
two further insights.

**Dashboard:** https://datastudio.google.com/reporting/6606dde6-c28a-427b-8020-a064a7cbb46f
**BigQuery project:** `chicago-taxi-cab`, dataset `dbo`
**Dataform core:** 3.0.52, location US
**Repo:** `github.com/adamfhiy/chicago`, branch `dev`

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
| `stg_trips` | Cleaned, filtered and deduplicated. Incremental with a 7 day lookback, partitioned monthly on `trip_date`, clustered on `trip_date` and `taxi_id` |
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

Daily partitioning was not possible. Eleven years is 4,018 days against a 4,000
partition cap, so staging is partitioned monthly and clustered on date.

### What `stg_trips` does

**Shape.** Renames `unique_key` to `trip_key`, casts both timestamps to
`DATETIME`, derives `trip_date`, `trip_minutes` and an integer `date_key` in
`YYYYMMDD` form for joining to `dim_date`. Carries pickup and dropoff
coordinates through. Coalesces `trip_seconds`, `fare`, `tips` and `trip_total`
to 0.

**Incremental load.** A `pre_operations` block declares a `checkpoint`. On an
incremental run it is `MAX(trip_start_dt)` from the existing table minus 7 days;
on a full refresh it is 2013-01-01. The 7 day lookback picks up late-arriving
rows and corrections. `uniqueKey: ["trip_key"]` makes Dataform merge rather than
append, so reprocessed rows replace themselves instead of duplicating.

**Quality filter.** Rows are kept only where the trip ends at or after it starts
and runs 12 hours or less, and where `trip_seconds` is between 0 and 43,200.
That removes roughly 29,000 rows, 0.02% of the table. A null end timestamp is
kept, since those are imputed rather than dropped.

**End time.** A missing end timestamp is imputed as `trip_start + trip_seconds`.
A `CASE` also caps any trip whose derived duration exceeds 12 hours back to
`trip_start + trip_seconds`, guarding against a meter left running.

**Dedupe.** `ROW_NUMBER()` over `trip_key` ordered by start time, keeping the
first row, then an assertion on `trip_key` uniqueness.

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
| Cash | 55.97% | 0% |
| Credit Card | 39.87% | 94% |
| Mobile | 1.48% | 93% |
| Prcard | 1.24% | 6% |
| Unknown | 0.90% | 5% |
| No Charge | 0.48% | 17% |
| Dispute | 0.05% | 0% |
| Pcard | 0.01% | 6% |
| Split | 0.00% | 83% |
| Way2ride | 0.00% | 87% |
| Prepaid | 0.00% | 0% |

Nine in ten card passengers tip. Effectively no cash passenger does. Same city,
same cabs. That is a recording gap, not a behavioural one.

Prepaid cards behave like cash, so only Credit Card and Mobile are treated as
reliable. Split and Way2ride tip at card-like rates but round to 0.00% of trips,
too small to matter either way.

---

## Q1: top 100 tip earners, last 3 months

Vehicles ranked by total tips over the last three months in the data, restricted
to Credit Card and Mobile payments.

The top 100 took $406,160 in tips across 59,232 trips, at a 21.53% tip rate.

**Assumption on wording.** The question says "tip earners" but clarifies "earn
more money than others". Tips are taken as the money in question, so the ranking
is on total tips.

**Cash is excluded** because it is 55.97% of trips and records no tips at all.
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

Within 2023: 858,274 shifts.

### How shifts are derived

Sessionisation on trip gaps. A gap of 8 hours or more between consecutive trips
ends a shift, taken directly from the question. Every shift produced this way
therefore contains no 8 hour break by construction. Q4c tests that threshold
rather than assuming it.

### Qualifying rules

| Rule | Value | Basis |
| --- | --- | --- |
| Break that ends a shift | 8 hours | Given in the question, confirmed in Q4c |
| Long shift | 12 hours or more | Changeover convention, confirmed below |
| Minimum shifts | Median shifts per vehicle | Computed at runtime, not hardcoded |
| Share that must be long | 40% | Reading of "regularly" |
| Rank within qualifiers | Total hours | "work more hours than others" |

The minimum-shifts floor is derived in the model itself, as the 50th percentile
of shifts per vehicle via `APPROX_QUANTILES`, so it moves with the data rather
than being fixed. Without a volume floor a vehicle with three recorded shifts
would outrank a genuine full-time one.

`mart_overworked_taxis` also carries `total_hours_resolvable`, which sums only
shifts of 24 hours or less, and `implausible_shifts`, which counts those above
24. That keeps the shared-medallion problem visible in the output rather than
buried in a caveat.

### The answer

The top three vehicles:

| Taxi ID | Hours | Shifts | Long shifts | % long | Over 24 h | Avg shift |
| --- | --- | --- | --- | --- | --- | --- |
| `3ce4fc90` | 6,560.25 | 295 | 220 | 74.58% | 79 | 22.24 h |
| `2780ead1` | 6,042.75 | 360 | 162 | 45.00% | 79 | 16.79 h |
| `008dda45` | 6,035.75 | 364 | 147 | 40.38% | 97 | 16.58 h |

Across the whole top 100: 35,223 shifts, averaging 12.30 hours each, of which
2,608 ran past 24 hours.

### Why the long shift threshold is 12 hours

12 hours comes from the US taxi changeover convention, which is outside
knowledge. The distribution agrees with it independently:

| Shift length | Shifts |
| --- | --- |
| 00-02h | 115,400 |
| 02-04h | 98,200 |
| 04-06h | 119,800 |
| 06-08h | 134,400 |
| 08-10h | 137,500 |
| 10-12h | 120,600 |
| 12-14h | 76,500 |
| 14-16h | 30,600 |
| 16-18h | 6,500 |
| 18-20h | 1,800 |
| 20-22h | 1,100 |
| 22-24h | 1,200 |
| 24h+ | 14,600 |

Roughly 725,900 shifts finish inside 12 hours, 84.6%. About 132,300 run longer,
15.4%. The threshold lands near the longest sixth of shifts, so convention and
data point at the same region.

Buckets are 2 hours wide so bar height reads as density. The final bucket is
open-ended and is not comparable in width to the others.

### Caveat: this measures vehicles, not drivers

`taxi_id` is a medallion, not a driver.

14,600 shifts ran past 24 hours. The 24h+ bucket is a jump rather than a tail,
standing at 14,600 against 4,100 across the whole 18 to 24 hour range combined.
The distribution does not taper into it. Trips per hour also falls steadily with
shift length, from about 1.45 in the 02-04h band to roughly 0.65 by 24h+, which
is what sharing rather than endurance looks like.

These are shared vehicles handed between drivers without an 8 hour gap. The
answer measures vehicle utilisation. It remains the right starting point for a
fatigue audit, because a medallion running 6,560 hours has drivers behind it
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

| Holiday | Type | % vs baseline | Stddev | Years |
| --- | --- | --- | --- | --- |
| Christmas Day | federal | -71.08 | 9.75 | 11 |
| Thanksgiving Day | federal | -60.54 | 7.41 | 11 |
| Christmas Eve | cultural | -41.00 | 13.59 | 10 |
| Memorial Day | federal | -36.32 | 11.99 | 11 |
| Labor Day | federal | -34.35 | 14.10 | 11 |
| Thanksgiving Eve | cultural | -26.93 | 11.10 | 11 |
| Martin Luther King Jr. Day | federal | -18.36 | 9.94 | 11 |
| Presidents Day | federal | -18.28 | 5.82 | 11 |
| Halloween | cultural | -9.68 | 14.90 | 11 |
| Veterans Day | federal | -4.98 | 5.02 | 11 |
| Columbus Day | federal | -0.71 | 8.45 | 11 |

Largest fall is Christmas Day at -71.08%, smallest is Columbus Day at -0.71%.

### Results too volatile to average

Standard deviation above 15. These swing so widely between years that a mean
would report a number that never actually occurred, so the range is given
instead.

| Holiday | Worst year | Best year | Stddev | Years |
| --- | --- | --- | --- | --- |
| New Year's Eve | -48.50% | +88.89% | 39.51 | 10 |
| St Patrick's Day | -59.50% | +56.76% | 27.10 | 11 |
| New Year's Day | -57.20% | +21.67% | 25.62 | 11 |
| Independence Day | -59.32% | -3.05% | 19.22 | 11 |
| Juneteenth | -24.20% | +7.63% | 16.22 | 3 |

The 15 threshold is a judgement call. It sits in a natural gap in the data, since
nothing falls between 14.90 and 16.22.

Independence Day is the odd one in this group. It is volatile by the stddev rule,
but unlike the others it is negative in every single year, worst to best. The
size of the fall is unpredictable; the direction is not.

**Juneteenth has only 3 years** because it became a federal holiday in 2021. Its
figure is not comparable to holidays averaged across a decade.

### What the pattern means

**The size of the drop tracks how many offices actually close**, not whether the
day is officially federal. Christmas is federal and almost everything shuts, so
trips fall 71%. Columbus Day is equally federal but most private employers stay
open, so trips fall under 1%. Halloween is not a federal holiday at all and still
drops 10%.

**Chicago taxi demand is commuting, not leisure.** If cabs were mainly for going
out, holidays would be busy. They are empty instead.

**The volatile holidays are all social occasions** where the weekday decides
everything. St Patrick's Day averages slightly positive at +7.44% but ranges from
-59.50% to +56.76%, which is the clearest case of an average that describes no
actual year. On a Saturday it is a boom, on a Tuesday a normal working day. New
Year's Eve is the same story at a wider spread.

**Business value.** Holidays with tight variance can be planned on the average,
so supply can be cut confidently. New Year's Eve and St Patrick's Day must be
planned by weekday instead, because the holiday itself predicts nothing.

---

## Q4a: cabs wait longest when there is no one to pick up

Idle time is the gap between one trip ending and the next beginning, capped at
the 8 hour shift break since anything longer is a new shift rather than waiting.

| Hour | p90 wait | Avg wait | Share of trips | Total idle |
| --- | --- | --- | --- | --- |
| 12AM | 165 min | 64.0 min | 1.79% | 118.6K h |
| 1AM | 150 min | 58.5 min | 1.13% | 68.9K h |
| 2AM | 135 min | 55.3 min | 0.68% | 38.1K h |
| 3AM | 150 min | 60.2 min | 0.50% | 28.0K h |
| 4AM | 255 min | 80.3 min | 0.59% | 36.8K h |
| 5AM | 300 min | 87.1 min | 0.98% | 52.9K h |
| 6AM | 195 min | 65.5 min | 1.82% | 70.3K h |

Worst-case wait peaks at 300 minutes at 5AM and bottoms at 75 minutes. The 3AM to
5AM block carries 2.07% of all trips.

Average wait tracks the same shape as p90 and peaks in the same place, rising
from about 55 minutes at 2AM to 87 minutes at 5AM, so the pattern is not an
artefact of the tail.

**Business value.** A cab working 3AM to 6AM chases roughly 2% of the day's
fares and can wait up to five hours between them. The same car in the morning
peak waits 75 minutes at worst and picks up several times the volume. Shifting
overnight shift starts two hours later raises revenue per vehicle hour without
adding a single car. It also cuts fatigue risk, since the longest shifts in Q2
run straight through these dead hours.

---

## Q4b: a shift stops earning after 14 hours

Trips per shift climb steadily with shift length, then stop. The ceiling is 12.8
trips per shift, reached in the 16 to 18 hour band, and the gain from each extra
two hours has already collapsed before then.

| Shift band | Shifts | Trips per shift | Gain over previous band |
| --- | --- | --- | --- |
| 00-02h | 115,384 | 2.0 | n/a |
| 02-04h | 98,176 | 4.2 | +2.2 |
| 04-06h | 119,793 | 5.6 | +1.4 |
| 06-08h | 134,405 | 7.1 | +1.5 |
| 08-10h | 137,514 | 8.6 | +1.5 |
| 10-12h | 120,598 | 10.3 | +1.7 |
| 12-14h | 76,525 | 11.6 | +1.3 |
| 14-16h | 30,580 | 12.7 | +1.1 |
| 16-18h | 6,468 | 12.8 | +0.1 |

Up to 14 hours, two more hours on the road buys between 1.3 and 2.2 more fares.
The 14 to 16 hour band still buys 1.1. The 16 to 18 hour band buys 0.1, which is
nothing.

Trips per hour tells the same story from the other side, falling steadily from
about 1.45 in the 02-04h band to roughly 0.65 by 24h+.

This is an average rather than a hard cap. Individual shifts go above and below
it. The precise claim is that the average stops rising after the 14 to 16 hour
band.

**Business value.** There is a practical revenue cap per vehicle per shift and it
is reached by 16 hours. Every hour past that carries fuel, wear and fatigue risk
with no fare to offset it. Capping shifts at 14 to 16 hours would remove the
fatigue exposure at almost no revenue cost. Growth has to come from more
vehicles or better positioning, not longer shifts.

---

## Q4c: a 7 to 8 hour break is the right threshold

Q2 takes its 8 hour break straight from the question. This tests whether that
number is defensible.

### The gap distribution has two humps

Gaps between fares in 2023, under 24 hours, fall into two clearly separate
groups. The first is short turnarounds between fares, concentrated in the 0 to 2
hour bars which hold 4.03M, 954K and 384K gaps. The second is a broad hump
centred around 13 to 15 hours, which is overnight rest.

Between them, roughly 5 to 7 hours, sits the trough. Any threshold in that dip
separates turnaround from rest cleanly, which is exactly what a shift boundary
should do.

### Threshold sensitivity

| Break (h) | Shifts | Median (h) | p95 (h) | % over 24h | % under 2h |
| --- | --- | --- | --- | --- | --- |
| 3 | 1,123,380 | 4.00 | 12.5 | 0.36% | 29.94% |
| 4 | 983,277 | 5.50 | 13.5 | 0.58% | 21.26% |
| 5 | 927,065 | 6.50 | 14.0 | 0.83% | 17.75% |
| 6 | 896,548 | 6.75 | 14.5 | 1.10% | 15.96% |
| **7** | **875,923** | **7.25** | **14.8** | **1.46%** | **14.82%** |
| **8** | **858,274** | **7.25** | **14.8** | **2.08%** | **14.08%** |
| **9** | **837,935** | **7.50** | **15.3** | **3.18%** | **13.62%** |
| 10 | 811,194 | 7.50 | 23.8 | 4.88% | 13.35% |
| 11 | 773,892 | 7.25 | 31.8 | 7.43% | 13.30% |
| 12 | 725,572 | 7.25 | 35.8 | 10.82% | 13.38% |

### Why 7 to 9 is the stable band

**Below 7**, shifts fragment. At a 3 hour break nearly 30% of shifts are under 2
hours, which are turnarounds being counted as whole shifts.

**Above 9**, shifts start merging. p95 jumps from 15.3 hours at a 9 hour break to
23.8 at 10, and the share of shifts over 24 hours more than doubles from 3.18% to
4.88%, then keeps climbing to 10.82% by 12. Those are separate shifts being
glued together across a real overnight rest.

**Between 7 and 9** everything is flat. Median holds at 7.25 to 7.5 hours, p95 at
14.8 to 15.3, and shift count moves by only 4%. The answer is insensitive to the
exact choice inside that band.

The 8 hours given in the question sits in the middle of the stable band. Q2's
figures are not an artefact of that choice.

---

## Limitations

- `taxi_id` is a medallion, not a driver
- Cash tips are unrecorded, so Q1 measures card tips only
- Q2, Q4b and Q4c cover 2023 only and are not comparable across years
- Idle time is capped at 8 hours by definition, since a longer gap ends the shift
- The stddev 15 cutoff separating reliable from volatile holidays is a judgement
  call, chosen because it sits in a natural gap in the data
- Shift length runs from first pickup to last dropoff, so it is a floor on hours
  worked, not a ceiling
- Trips longer than 12 hours or with `trip_seconds` over 43,200 are dropped in
  staging, about 29,000 rows or 0.02%. These are meter errors rather than real
  trips, but the cut is a judgement call
- Timestamps are rounded to 15 minutes, producing small apparent overlaps
  between consecutive trips
- Not all trips are reported to the City, so the source is close to but not a
  complete census
