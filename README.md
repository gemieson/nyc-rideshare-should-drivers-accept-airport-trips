# Should NYC Rideshare Drivers Accept Airport Dropoff Trips? A Regression and Simulation Analysis

### Overview 

Rideshare drivers widely regard airport dropoffs as lucrative due to longer distances and higher base fares, but the return trip uncertainty introduces real risk. After dropping off at an airport, a driver must either join the virtual queue and wait for a return trip or deadhead back to the city empty, losing both time and potential earnings. The true cost of an airport trip is therefore not just the fare but the opportunity cost of time spent waiting instead of completing city trips.

Using 1.9 million NYC TLC High Volume FHV trips from 2024 to 2025, a regression model and Monte Carlo simulation were built to quantify whether accepting an airport trip during a 3-hour shift is more profitable than staying in the city. Results suggest the airport premium is real but time sensitive, varying meaningfully by borough and hour of day.

### Data
- Source:  NYC TLC Trip Record Data, High Volume FHV (2024 - 2025)
- Size: 1.9 million observations
- **Key Variables**: (PULocationID, DOLocationID, pickup_datetime, borough, trip_duration, driver_pay)
- To keep trips representative of typical city travel, the sample was restricted to trips under two hours.

### Methodology

#### Linear Regression

Based on the aggregation, airport drop-offs show meaningfully higher average earnings than non-airport trips. 
<img src="images/aggregation.png" width="700">

This however, is confounded by trip length, time of day, and borough of origin, all of which independently influence driver earnings. In order to isolate these variables a linear regression of **log(driver_pay) ~ is_JFK + is_LGA + is_EWR + C(hour) + C(borough) + log(trip_duration)** was applied. Linear regression is appropriate here because driver pay is a continuous outcome and trip duration exhibits a roughly linear relationship with earnings. Log transformations were applied in an attempt to make errors of constant variance.

<img src="images/lin1.png" width="500">
<img src="images/lin2.png" width="500">
<img src="images/lin3.png" width="500">

The model explains roughly 87% of the variation in log driver pay, and all three airport indicators are positive, large, and statistically significant. Holding trip duration, hour, and borough constant, we interpret the airport coefficients using the formula $(e^{b} - 1) \times 100$ which gives the true percentage change in driver pay associated with each airport relative to a comparable non-airport trip. A JFK dropoff is associated with a 21.5% pay premium, LGA with 22.0%, and EWR with 49.2%. The log duration coefficient of 0.93 confirms that pay grows nearly proportionally with trip length. Hour and borough fixed effects behave as expected, with late night hours and Staten Island pickups commanding the highest premiums, and midday hours and Brooklyn pickups the lowest.

The normality assumption is violated, indicating a heavy right tail even after transformation. However, with 1.9 million observations and HC3 robust standard errors, coefficient estimates remain consistent and inference remains valid.

#### Simulation

The simulation models a three-hour shift and compares total expected earnings across three paths: staying in the city, taking a JFK dropoff, or taking an LGA dropoff. For each borough and hour combination, 10,000 shifts were simulated and the path with the highest mean earnings was recommended.

**Key Assumptions**
- Drivers always join the airport queue after dropoff with no immediate deadhead
- Return trip probability is proxied by the hourly pickup to dropoff ratio at each airport, capped at 1
- Wait time follows a log-normal distribution parameterized by the pickup to dropoff ratio
- Fare and trip duration inputs are averaged by borough and hour from the raw data

The city baseline applies the average local earnings rate over 180 minutes at 58% utilization. Airport paths accumulate earnings across three stages (outbound fare, wait time, and return trip), each drawn from distributions parameterized by the borough-airport-hour averages. If no return trip is secured, the driver deadheads back. Remaining time on each path is filled with local trips at borough-hour average rates, and the recommended path is the one with the highest mean earnings.

<img src="images/heatmap.png" width="700">

#### Results

JFK dominates recommendations across most boroughs and hours, particularly from 7pm to 2am, while LGA is competitive through the mid-day window, most clearly in Brooklyn and Staten Island, where LGA holds the recommendation almost continuously from around 7am to 6pm. Queens breaks this pattern, since rather than favoring an airport all day, it reverts to the city baseline during the late-morning to early-afternoon stretch (roughly 8:30am to 1:30pm), and even outside that window its airport advantage in dollar terms is consistently among the smallest of the five boroughs, often just $1 to $20, compared to $30 to $50+ elsewhere. The city baseline is otherwise only preferred during the overnight and early morning hours (roughly 3:00 to 6:00), when return trip probability is low and the wait cost outweighs the fare premium. Staten Island shows the highest airport premium in terms of pay, consistent with its far distance from the airports. Queens, by contrast, shows the weakest airport advantage overall, likely because drivers are already close to both airports and face a higher opportunity cost of leaving; the marginal benefit of detouring to an airport pickup is small when you're already near one. Overall, the airport premium is real but time sensitive, with the size of the benefit depending heavily on the borough's baseline proximity to JFK and LGA.


#### Prediction

The regression above is explanatory. A separate question is predictive — given what is knowable before a trip is accepted, how accurately can its pay be forecast? The R² of 0.87 does not answer this, since it measures fit on the same data used to estimate the coefficients.

The sample was split by time rather than at random, training on 2024 and holding out 2025. A random split would let the model learn from trips occurring after those it predicts. Mean absolute error was used as the primary metric because it is expressed in dollars and is less dominated by the heavy right tail than RMSE.

Three models were compared. The baseline predicts each trip as the average pay of training trips sharing its borough and hour, using no information about trip length. The log-linear specification was refit on the training period alone, with predictions converted back to dollars using the smearing correction $e^{\sigma^2/2}$. The third is a histogram-based gradient boosting regressor trained on raw pay with absolute-error loss, stopped early at 138 trees.

| Model | MAE | RMSE | Improvement over baseline |
|---|---|---|---|
| Borough-hour mean | $11.63 | $17.47 | — |
| Log-linear OLS | $3.98 | $8.17 | 65.8% |
| Gradient boosting | $3.56 | $7.41 | 69.4% |

Both models improve substantially on the baseline, though this is less impressive than it appears, since the baseline has no access to trip duration. The informative comparison is between the fitted models. Gradient boosting reduces error a further 11%, roughly forty cents per trip, attributable to interactions the additive specification cannot express.

Permutation importance, measured as the increase in test MAE when each feature is shuffled, confirms the dominance of trip length.

| Feature | MAE increase when shuffled |
|---|---|
| trip_duration_min | $11.79 |
| hour | $0.45 |
| borough | $0.21 |
| is_EWR | $0.14 |
| is_LGA | $0.11 |
| is_JFK | $0.10 |
| dow | $0.07 |

Duration accounts for essentially the entire advantage over the baseline. The airport indicators rank EWR, LGA, JFK, matching the ordering of the regression coefficients.

Their small predictive contribution reflects frequency rather than effect size. Airport dropoffs are 5.3% of the sample, so shuffling an airport flag barely moves error averaged across all trips even though the effect on any individual airport trip is large. The regression answers how much more an airport trip pays. Permutation importance answers how much knowing about airports helps predict an arbitrary trip.

Disaggregating error by trip type reveals the model's main weakness.

| Trip type | Mean absolute error | Median absolute error | n |
|---|---|---|---|
| Non-airport | $3.46 | $1.41 | 901,361 |
| Airport | $5.34 | $2.58 | 50,149 |

Error on airport trips is 54% higher, consistent with the heavier right tail and the greater variance in long-distance fares, where tolls, surge, and route variation add noise not captured by duration, hour, and borough. The model is least reliable on the trips this analysis is concerned with. The regression coefficients remain well identified across 1.9 million observations; the narrower point is that a point forecast for any particular airport trip carries more uncertainty than the headline error suggests.

### Limitations

Return-trip probability is proxied by the hourly ratio of pickups to dropoffs, capped at one. This is not a queue model — it ignores how many drivers are already waiting, which is the actual determinant of wait time.

Driver behavior is modeled as a single decision at the start of a shift with no re-optimization. Real drivers respond continuously to conditions, decline trips, and reposition. The three-hour horizon and 58% utilization are fixed inputs rather than estimated quantities.

The analysis observes pay but not costs. Fuel, tolls, depreciation, and the driver's own commute are excluded, all of which fall disproportionately on long airport trips, so the premiums reported here are gross.

The data covers 2024 and 2025 only. Queue dynamics depend on platform policy and driver supply, both of which change, so the borough-hour recommendations are descriptive of this period rather than stable rules.








