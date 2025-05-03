# README

## Code Structure

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

### Strategy Overview

This project implements a Smart Order Router based on the static Cont & Kukanov cost model. The code evaluates six execution strategies:

**Baseline Strategy**
* **Best Ask Strategy**: Selects the lowest ask price at each snapshot and buys from that venue.
* **TWAP**: Executes a fixed amount every 60 seconds, but may lead to underfills toward the end.
* **VWAP**: Allocates order sizes proportionally based on ask sizes across venues. Since the dataset only contains one venue, VWAP behaves similarly to Best Ask.

**My Strategy based on the Allocator**
* **Strategy 1**: Attempts to call the allocator at every snapshot. If the allocator fails to return a valid split (due to total ask size < order size), no trade is made. This conservative strategy avoids overtrading and achieves a lower cost than Best Ask.
* **Strategy 2**: Places orders every 15 seconds (and every 1 second in the last minute). When the market depth is insufficient, it falls back to buying from the best-priced venue. This is inspired by TWAP but adapted to the high-frequency nature of the dataset (microsecond-level granularity) and the small ask sizes.
* **Strategy 3**: Tracks the price of each trade and increases order size if a better price appears. The order size increment is capped at 10% of the total to mitigate slippage and market impact. In the last 2 minutes, it switches to fixed-interval trades. This strategy may suffer in rising markets due to delayed execution.
  

### Comments on Strategies
**Strategy 1:**
 - A weird thing is that the task specifies that we should "attempt to execute as many shares as the allocator tells you" at each snapshot and "beats the best-ask baseline by at least a few basis points". However, the allocator only returns a valid split when the total ask size across venues is greater than or equal to the order size. When it's not, the strategy faces a choice: either buy available shares or skip the snapshot.
 - If we always buy available shares, the behavior becomes indistinguishable from a Best-Ask strategy—especially because the dataset contains only one exchange. Therefore, Strategy 1 is designed to skip execution if the allocator returns no valid split. This conservative approach avoids overtrading and achieves a lower total cost than Best Ask under the test conditions.

**Strategy 2 (Periodic allocation with fallback)**:
- This strategy resembles TWAP but with a significantly increased interval between executions. The rationale is twofold:

   - The dataset is high-frequency, with timestamp granularity in the nanosecond range, so placing orders at lower frequency (e.g., every 15 seconds, and every 1 second in the final minute) avoids unnecessary noise.

   - The available ask sizes (ask_sz_00) are relatively small, so frequent small-volume trades help in gradually filling the order. This strategy can perform well in sideways or oscillating markets, where cost averaging is advantageous.

**Strategy 3 (Price-sensitive adaptive sizing)**:
 - This strategy tracks the execution price and increases the buy quantity if a lower price appears. The additional order size is capped at 10% of the total to control slippage and market impact. In the final two minutes, it switches to fixed-interval buying.
 - However, due to the upward-trending price pattern in the dataset, the strategy experienced significant underfills early on, forcing larger purchases at higher prices later—resulting in higher overall cost. While this approach might perform better in volatile or declining markets, it struggled under trending conditions.

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
* **Allocator Optimization**: The current brute-force allocator enumerates in steps of 100. This leads to issues if order size isn't a multiple of 100. In production, stochastic gradient descent (SGD) as discussed in the paper would be more efficient.
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
