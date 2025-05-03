# README

## Code Structure

This project implements a Smart Order Router based on the static Cont & Kukanov cost model. The code evaluates six execution strategies:

* **Best Ask Strategy**: Selects the lowest ask price at each snapshot and buys from that venue.
* **TWAP**: Executes a fixed amount every 60 seconds, but may lead to underfills toward the end.
* **VWAP**: Allocates order sizes proportionally based on ask sizes across venues. Since the dataset only contains one venue, VWAP behaves similarly to Best Ask.
* **Strategy 1**: Attempts to call the allocator at every snapshot. If the allocator fails to return a valid split (due to total ask size < order size), no trade is made. This conservative strategy avoids overtrading and achieves a lower cost than Best Ask.
* **Strategy 2**: Places orders every 15 seconds (and every 1 second in the last minute). When the market depth is insufficient, it falls back to buying from the best-priced venue. This is inspired by TWAP but adapted to the high-frequency nature of the dataset (microsecond-level granularity) and the small ask sizes.
* **Strategy 3**: Tracks the price of each trade and increases order size if a better price appears. The order size increment is capped at 10% of the total to mitigate slippage and market impact. In the last 2 minutes, it switches to fixed-interval trades. This strategy may suffer in rising markets due to delayed execution.

### Function Overview

* **`compute_cost(...)`**: Calculates execution cost including price, underfill/overfill penalties, and market impact.
* **`allocate(...)`**: Brute-force allocator that finds the lowest-cost split of order size across venues in 100-share steps.
* **`execute_order_best_ask(...)`**: Executes trades at the venue with the best ask.
* **`execute_order_twap(...)`**: Time-weighted average price strategy.
* **`execute_order_vwap(...)`**: Volume-weighted average price strategy.
* **`execute_order_my_strategy1(...)`**: Allocator is used only when a full valid split is possible.
* **`execute_order_my_strategy2(...)`**: Time-based allocator and fallback hybrid strategy.
* **`execute_order_my_strategy3(...)`**: Price-tracking strategy with adaptive quantity and fallback schedule.
* **`grid_search_parameters(...)`**: Searches over hyperparameters to minimize cost for Strategy 1.
* **`plot_cum_costs(...)`**: Visualizes cumulative cost per strategy.
* **`load_data(...)`**: Parses and cleans the limit order book data.

## Parameter Choices

Based on the Cont & Kukanov model:

* **lambda\_over** penalizes overfills.
* **lambda\_under** penalizes underfills.
* **theta\_queue** penalizes execution risk and market impact.

Parameter grid used:

```python
lambda_o_vals = [0.05, 0.1, 0.2]
lambda_u_vals = [0.01, 0.05, 0.1]
theta_vals = [0.005, 0.01]
```

Rationale:

* Overfill is more severely penalized due to its potential to cause direct financial loss, while underfill only reduces potential gain. This reflects behavioral finance insights where loss aversion dominates.
* The market impact coefficient is kept low as discussed in Section 2.5 of the paper: "However, empirical studies show that market impact differences between limit and market orders are small."

## Suggested Improvements

* **Strategy Adaptation**: Strategy 1 performed best because prices trended upward during the test window. It captured lower prices earlier. However, in a sideways market, Strategy 2 or 3 could offer more stable cost averaging.
* **Dynamic Strategy Switching**: Future implementations could dynamically select strategies based on short-term market trend prediction.
* **Adaptive Parameters**: Parameters (like order size increment) could be scaled based on the remaining order size or volatility.
* **Allocator Optimization**: The current brute-force allocator enumerates in steps of 100. This leads to issues if order size isn't a multiple of 100. In production, stochastic gradient descent (SGD) or other continuous optimization methods would be more efficient.
* **Cost Function Fix**: In the `compute_cost` function, the line:

  ```python
  maker_rebate = max(split[i] - exe, 0) * venues[i]['rebate']
  ```

  wrongly applies the rebate to unexecuted shares. In reality, rebates are earned on executed maker orders, so this logic should be updated to:

  ```python
  maker_rebate = exe * venues[i]['rebate']
  ```

---

This project demonstrates practical challenges in implementing execution strategies using realistic market data and highlights areas where financial intuition and engineering constraints intersect.
