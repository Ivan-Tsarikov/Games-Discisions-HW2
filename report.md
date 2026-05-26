# Price-Change Game in Airline Competition: A Predictive Game-Theoretic Analysis

Course: Games and Decisions in Data Analysis and Modelling  
Homework Assignment #2  
Topic: Game theory  
Data source: Bureau of Transportation Statistics [DB1B Market data](https://www.transtats.bts.gov/DL_SelectFields.aspx?gnoyr_VQ=FHK&QO_fu146_anzr=b4vtv0%20n0q%20Qr56v0n6v10%20f748rB)  
Analysis notebook: [`HW_2/notebooks/02_price_change_predictive_game.ipynb`](notebooks/02_price_change_predictive_game.ipynb)
Authors: Tsarikov Ivan, Raimova Alina

---

## 1. Introduction

We chose the problem of strategic price competition between airlines on major U.S. domestic routes. Airline ticket prices are not chosen independently: if one carrier raises or lowers prices on a route, its competitors may react by changing their own prices in the same or opposite direction. Therefore, airline pricing is a natural real-life example of strategic interaction.

We've constructed an empirical normal-form game and compared its theoretical prediction with observed airline behaviour. Also, we modeled whether a carrier **increases** or **decreases** its passenger-weighted average fare relative to the previous quarter.

This choice is motivated by the exploratory analysis: for many routes, airlines tend to move prices in the same direction over time. A price-change game captures this dynamic more directly than a static `Low` / `High` price-level classification.



---

## 2. Data Description

We use the [DB1B Market data](https://www.transtats.bts.gov/DL_SelectFields.aspx?gnoyr_VQ=FHK&QO_fu146_anzr=b4vtv0%20n0q%20Qr56v0n6v10%20f748rB) from the Bureau of Transportation Statistics. The raw files are stored in [`HW_2/data/`](data/) as quarterly `T_DB1B_MARKET-*.csv` files (the archive can be sent upon request as it is too large for GitHub). They contain market-level observations with the following key fields:

- year and quarter;
- origin and destination airports;
- reporting carrier;
- passengers;
- market fare.

The raw CSV files were first combined into an analysis-ready parquet dataset in [`HW_2/cache/db1b_market_combined.parquet`](cache/db1b_market_combined.parquet). The dataset-preparation notebook is [`HW_2/notebooks/01_db1b_market_eda.ipynb`](notebooks/01_db1b_market_eda.ipynb), and the final predictive-game analysis is implemented in [`HW_2/notebooks/02_price_change_predictive_game.ipynb`](notebooks/02_price_change_predictive_game.ipynb).

The combined dataset covers:

| Quantity | Value |
|---|---:|
| Period | 2021Q1–2024Q4 |
| Number of quarters | 16 |
| Raw observations | 112,342,845 |
| Carriers | 25 |
| Airport-pair routes | 55,765 |
| City-market routes | 49,152 |
| Total passengers | 221,674,388 |
| Passenger-weighted average fare | approximately 207.76 |

A fare filter is applied before constructing the game:

$$
20 \leq \text{MARKET\_FARE} \leq 1000.
$$

This removes very low and extremely high fare records that are likely to be reporting artifacts or special cases. The exploratory analysis showed that the raw data contain fares equal to zero, very small fares below 20, and extreme fares above 1000. Since payoffs are revenue-based, these outliers could distort the estimated game.

---

## 3. Why Price Changes Instead of Price Levels?

An earlier modelling option was to classify prices as `Low` or `High` relative to each carrier's median fare. However, this approach mainly captures the **level** of prices and may reflect seasonality or common demand shocks rather than strategic moves.

For example, if demand rises in a quarter, both airlines may have high fares. If demand falls, both may have low fares. Such behaviour may be better interpreted as synchronized market movement rather than a one-shot choice of low or high price.

Therefore, the final model uses price changes:

- `Increase`: the carrier's passenger-weighted average fare increased relative to the previous quarter;
- `Decrease`: the carrier's fare did not increase relative to the previous quarter.

This strategy definition directly models the direction of pricing adjustment.

---

## 4. Game-Theoretic Model

### 4.1 Type of Game

The situation is modelled as a **static normal-form game**, estimated from repeated quarterly observations.

The real airline market is dynamic and repeated, but the normal-form representation is useful as a stage-game approximation. Each route-quarter observation is treated as one realization of the same strategic interaction between two carrier roles.

### 4.2 Players

The model pools observations across major routes. On each selected route, two player roles are defined using only the training period:

$$
N = \{Leader, Challenger\}.
$$

- `Leader` is the carrier with the largest passenger volume on the route in the training period.
- `Challenger` is the carrier with the second-largest passenger volume on the route in the training period.

Using roles instead of fixed carrier names allows us to pool many routes and obtain enough observations in each payoff-matrix cell.

### 4.3 Route Selection

Routes are selected using only training-period data. The baseline selection rules are:

- top 50 airport-pair routes by training passenger volume;
- route training passengers at least 50,000;
- both leader and challenger active in at least 10 training quarters;
- both leader and challenger active in at least 3 test quarters.

The selected-route summary is:

| Quantity | Value |
|---|---:|
| Selected routes | 50 |
| Total selected-route training passengers | 16,565,767 |
| Mean top-2 training share | 0.7264 |
| Minimum training active quarters per player | 10 |
| Minimum test active quarters per player | 3 |

The first selected routes include JFK-LAX, EWR-MCO, MCO-SJU, MCO-PHL, LAX-SFO, LGA-ORD, and LAX-ORD. These are large routes with repeated competition between two major carriers.

### 4.4 Strategies

Each player has two pure strategies:

$$
S_{Leader} = S_{Challenger} = \{Decrease, Increase\}.
$$

Let

$$
\bar p_{i,r,t}
$$

be the passenger-weighted average fare of player $i$ on route $r$ in quarter $t$. The percentage price change is

$$
\Delta p_{i,r,t} = \frac{\bar p_{i,r,t}}{\bar p_{i,r,t-1}} - 1.
$$

The observed strategy is defined as

$$
s_{i,r,t} =
\begin{cases}
Increase, & \Delta p_{i,r,t} > 0, \\
Decrease, & \Delta p_{i,r,t} \leq 0.
\end{cases}
$$

The first quarter for each route-carrier is used only as a lag and is not a game observation.

### 4.5 Payoffs

The raw revenue proxy is

$$
Revenue_{i,r,t} = \sum_k Passengers_{i,r,t,k} \cdot Fare_{i,r,t,k}.
$$

Since routes have different sizes, absolute revenue is not comparable across routes. Therefore, payoffs are normalized by each route-carrier's average training revenue:

$$
u_{i,r,t} = \frac{Revenue_{i,r,t}}{\overline{Revenue}_{i,r,train}}.
$$

Here,

$$
\overline{Revenue}_{i,r,train}
$$

is the average revenue of carrier $i$ on route $r$ during the training period.

Interpretation:

- $u_{i,r,t} = 1.10$: revenue is 10% above the player's normal training-period level;
- $u_{i,r,t} = 0.90$: revenue is 10% below the player's normal training-period level.

The payoff for each strategy profile is the average normalized revenue over all training observations with that profile:

$$
u_i(s_i,s_j) = \frac{1}{|T(s_i,s_j)|}\sum_{(r,t) \in T(s_i,s_j)} u_{i,r,t}.
$$

---

## 5. Train/Test Design

To make the comparison predictive rather than purely descriptive, the data are split by time:

| Sample | Period | Purpose |
|---|---|---|
| Training | 2021Q2–2023Q4 | Estimate payoff matrix and Nash equilibrium |
| Test | 2024Q1–2024Q4 | Compare theoretical prediction with empirical behaviour |

The split gives:

| Quantity | Value |
|---|---:|
| Training observations | 550 |
| Test observations | 200 |
| Number of routes | 50 |

This avoids estimating the game and evaluating it on exactly the same observations.

---

## 6. Estimated Training Game

The estimated payoff matrix from the training period is:

| Leader / Challenger | Challenger: Decrease | Challenger: Increase |
|---|---:|---:|
| Leader: Decrease | (0.989, 0.967), n=162 | (1.047, 1.040), n=39 |
| Leader: Increase | (1.005, 1.018), n=68 | (0.999, 1.009), n=281 |

Each cell contains:

$$
(Leader\ payoff, Challenger\ payoff),\ n,
$$

where $n$ is the number of training observations in that cell.

The minimum cell count is 39, so the pooled model avoids the sparse-cell problem that occurred in single-route games.

---

## 7. Solving the Game

A pure-strategy Nash equilibrium is a strategy profile $s^*=(s_i^*,s_j^*)$ such that no player can improve its payoff by unilaterally deviating:

$$
u_i(s_i^*,s_j^*) \geq \nu_i(s_i',s_j^*) \quad \forall s_i' \in S_i.
$$

### 7.1 Best Responses

For the Leader:

| If Challenger chooses | Leader best response | Payoff |
|---|---|---:|
| Decrease | Increase | 1.0050 |
| Increase | Decrease | 1.0471 |

For the Challenger:

| If Leader chooses | Challenger best response | Payoff |
|---|---|---:|
| Decrease | Increase | 1.0402 |
| Increase | Decrease | 1.0176 |

### 7.2 Nash Equilibria

The pure Nash equilibria are:

$$
(Decrease, Increase)
$$

and

$$
(Increase, Decrease).
$$

Thus, the static stage game predicts **asymmetric price changes**: one player increases while the other decreases.

---

## 8. Out-of-Sample Empirical Comparison

The Nash equilibria estimated on 2021–2023 data are used as theoretical predictions for 2024.

The predicted Nash-equilibrium profiles are:

$$
Decrease-Increase
$$

and

$$
Increase-Decrease.
$$

The 2024 test comparison is:

| Metric | Value |
|---|---:|
| Test observations | 200 |
| Share matching predicted NE profiles | 0.225 |

The train/test distribution of observed profiles is:

| Profile | Train share | Test share | Predicted NE? |
|---|---:|---:|---|
| Decrease-Decrease | 0.2945 | 0.3750 | no |
| Decrease-Increase | 0.0709 | 0.1000 | yes |
| Increase-Decrease | 0.1236 | 0.1250 | yes |
| Increase-Increase | 0.5109 | 0.4000 | no |

Only 22.5% of 2024 observations match the Nash-equilibrium profiles. Therefore, the static normal-form prediction only partially coincides with empirical behaviour.

---

## 9. Main Empirical Deviation: Synchronized Price Movements

The main empirical result is that observed price movements are mostly synchronized i.e. the diagonal profiles:

$$
Decrease-Decrease
$$

and

$$
Increase-Increase.
$$

Therefore, Nesh equilibrium belongs to asymmetric profiles:

$$
Decrease-Increase
$$

and

$$
Increase-Decrease.
$$

The synchronized shares are:

| Sample | Synchronized share | Asymmetric share |
|---|---:|---:|
| Train 2021–2023 | 0.8055 | 0.1945 |
| Test 2024 | 0.7750 | 0.2250 |

This is the key empirical deviation from the static Nash prediction. The estimated game predicts asymmetric price changes, but real airline behaviour is dominated by synchronized price changes.

This means that airlines often move prices in the same direction: both increase fares or both decrease fares. Such behaviour may reflect:

- common demand shocks;
- fuel and cost shocks;
- seasonality;
- capacity constraints;
- or even repeated interaction and tacit coordination. 

All in all, it seems like airline market is rather simple and streight-forward in its pricing behaviour.

---

## 10. Role-Level Comparison

The assignment asks which player behaves closer to the theoretical prediction. In this role-based pooled game, direct player-level best-response comparison is not very informative because the best-response structure is symmetric: both roles are predicted to move in the opposite direction from the opponent.

Instead, we compare how often each role chooses the `Increase` strategy:

| Role | Train Increase share | Test Increase share |
|---|---:|---:|
| Leader | 0.6345 | 0.5250 |
| Challenger | 0.5818 | 0.5000 |

Leaders increase prices slightly more often than challengers in both train and test samples. However, both roles show broadly similar behaviour. The model therefore does not provide strong evidence that one role is substantially closer to the theoretical prediction than the other.

---

## 11. Interpretation

The theoretical normal-form game predicts asymmetric price movements. This is intuitive from a strategic perspective: if one carrier increases prices, the other may benefit from decreasing or not increasing prices to attract passengers; if one carrier decreases, the other may benefit from increasing if demand remains strong enough.

However, the empirical data show that synchronized movements are much more common. The most frequent profile is `Increase-Increase`, followed by `Decrease-Decrease`. This suggests that real airline pricing is not well described by a one-shot static game alone.

The most plausible interpretation is that airlines face strong common market forces. If demand rises, both carriers may increase prices. If demand falls, both may decrease prices. In addition, because airlines interact repeatedly over time, they may avoid aggressive unilateral deviations that could trigger future price competition.

Therefore, the static game is useful as a benchmark, but the empirical behaviour points toward a richer repeated or dynamic environment.

---

## 12. Limitations

The analysis has several important limitations.

### 12.1 Revenue Is Not Profit

Payoffs are based on revenue, not profit. True profit would require costs such as fuel, aircraft utilization, crew, airport fees, and load factors. These data are not available in DB1B Market.

### 12.2 Quarterly Aggregation

Airlines change prices much more frequently than once per quarter. Quarterly average fares hide within-quarter pricing dynamics, booking-time effects, and short-run reactions.

### 12.3 Common Shocks

Synchronized price movements may be caused by common shocks rather than strategic coordination. Demand, seasonality, fuel prices, and macroeconomic conditions can affect all carriers at the same time.

### 12.4 Role-Based Pooling

The model pools different routes and different airlines using the roles `Leader` and `Challenger`. This improves sample size but abstracts from carrier-specific business models and route-specific features.

### 12.5 Strategy Simplification

The binary strategy set `Increase` / `Decrease` is simple and interpretable, but real pricing decisions are continuous and multidimensional. Airlines choose many fares across booking classes, days, and passenger types.


---

## 13. Practical Conclusions

1. The predictive game estimated on 2021–2023 data only partially explains 2024 behaviour.
2. The Nash-equilibrium prediction is asymmetric price movement, but the empirical data are dominated by synchronized movements.
3. This suggests that airline pricing on major routes is strongly influenced by common market conditions and repeated interaction.
4. The strongest empirical finding is not that airlines follow the static Nash equilibrium, but that they often adjust prices in the same direction.

---

## 14. Finial Conclusion

We've constructed an empirical normal-form game of airline price changes using BTS DB1B Market data. The game was estimated on 2021–2023 observations from 50 major U.S. domestic routes and tested on 2024 data.

The estimated game predicts two pure Nash equilibria:

$$
(Decrease, Increase)
$$

and

$$
(Increase, Decrease).
$$

However, only 22.5% of 2024 observations match these predicted equilibrium profiles. Most observed behaviour is synchronized: both carriers increase or both decrease prices. The synchronized share is 80.55% in training data and 77.50% in test data.

Therefore, the theoretical static-game prediction does not fully coincide with empirical behaviour. The main conclusion is that airline price competition appears to be shaped less by isolated one-shot deviations and more by synchronized responses to common market forces and possibly repeated-interaction incentives.

