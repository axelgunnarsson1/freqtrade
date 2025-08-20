# Hyperopt Migration Plan: From Custom Optuna to Freqtrade

This document details the plan for migrating the `optuna`-based hyperparameter optimization process from the previous custom framework to `freqtrade`'s integrated `hyperopt` module.

---

## 1. Defining the Search Space

*   **Previous Project:** The search space was defined in a configuration file (`src/config_models.py`) and parameters were suggested to `optuna` via a helper function (`_prepare_params_from_trial` in `run_task.py`).

*   **Freqtrade Equivalent (The Plan):**
    *   Define the search space directly within the strategy file using `Parameter` objects (`IntParameter`, `DecimalParameter`, etc.). This co-locates the tunable parameters with the strategy logic that uses them.
    *   `hyperopt` will automatically detect these `Parameter` objects and use them to define the study's search space.

    **Example:**
    ```python
    from freqtrade.strategy import (IStrategy, IntParameter, DecimalParameter)

    class MyStrategy(IStrategy):
        # Define a parameter for the RSI buy signal
        buy_rsi = IntParameter(low=10, high=50, default=30, space="buy")

        # ... strategy logic ...
    ```

---

## 2. The Objective Function (Loss Function)

*   **Previous Project:** The `objective` function in `run_task.py` was responsible for running a backtest for each trial and returning the Sortino Ratio, which `optuna` was configured to maximize.

*   **Freqtrade Equivalent (The Plan):**
    *   Create a custom **Loss Function** class that inherits from `IHyperOptLoss`.
    *   `hyperopt` runs a full backtest for each trial and passes the results dataframe to this class.
    *   The `hyperopt_loss_function` method must return a single float value that `hyperopt` will **minimize**.
    *   To maximize a metric like the Sortino Ratio, we simply return its negative value.

    **Example (`user_data/hyperopts/SortinoLoss.py`):**
    ```python
    import numpy as np
    from pandas import DataFrame
    from freqtrade.optimize.hyperopt import IHyperOptLoss

    class SortinoLoss(IHyperOptLoss):
        @staticmethod
        def hyperopt_loss_function(results: DataFrame, **kwargs) -> float:
            # The loss is the negative of the Sortino Ratio.
            # Hyperopt will minimize this, effectively maximizing Sortino.
            sortino = results['sortino'].mean()
            return -sortino if np.isfinite(sortino) else 100.0
    ```

---

## 3. Running the Optimization Study

*   **Previous Project:** The optimization was started by running the `run_task.py` script from the command line.

*   **Freqtrade Equivalent (The Plan):**
    *   Use the built-in `freqtrade hyperopt` command. This command handles the entire workflow: creating the `optuna` study, connecting to the database, running trials, and saving results.

    **Example Command:**
    ```bash
    freqtrade hyperopt \
        --config config.json \
        --strategy MyStrategy \
        --hyperopt-loss SortinoLoss \
        --epochs 100 \
        --spaces buy \
        --timerange 20240101-20240630
    ```
    *   `--hyperopt-loss SortinoLoss`: Specifies the custom loss function to use.
    *   `--epochs 100`: Sets the number of trials for `optuna` to run.
    *   `--spaces buy`: Tells `hyperopt` to only tune parameters in the "buy" space.

---

## 4. Cross-Validation Approach

*   **Previous Project:** A custom `CombPurgedKFoldCV` was used to run cross-validation within the `optuna` objective function to prevent overfitting.

*   **Freqtrade Equivalent (The Plan):**
    *   `freqtrade`'s design inherently prevents lookahead bias by processing data chronologically.
    *   The standard validation methodology is to run `hyperopt` on an "in-sample" dataset (e.g., the first 6 months of the year) to find the best parameters.
    *   Then, run the `backtesting` command with these optimal parameters on a separate, "out-of-sample" dataset (e.g., the next 3 months) that the optimizer has never seen.
    *   This in-sample/out-of-sample validation achieves the same core goal as cross-validation: ensuring the strategy is robust and not just fitted to a specific period.
