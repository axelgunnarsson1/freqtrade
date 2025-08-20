# FreqAI Model Hyperparameter Optimization Plan

This document clarifies the correct approach for optimizing the internal hyperparameters of a FreqAI model (e.g., learning rate, network architecture), which requires retraining the model on every optimization trial.

---

## The Challenge: Standard `hyperopt` vs. Model Training

A crucial distinction exists in `freqtrade`'s optimization capabilities:

*   **Standard `freqtrade hyperopt`:** This tool is designed to optimize **strategy parameters** (e.g., buy/sell signal thresholds, stop-loss values). It uses a single, pre-trained FreqAI model and runs fast backtests to tune how the strategy *uses* the model's predictions. It **does not** retrain the AI model on each trial.

*   **Model Hyperparameter Optimization:** This is the process of finding the best internal parameters for the machine learning model itself. This **requires** a new model to be trained from scratch for every trial (i.e., for each new set of hyperparameters).

The standard `hyperopt` command is not suitable for model hyperparameter optimization because it would be incredibly time-consuming.

---

## The Plan: A Custom `optuna` Orchestration Script

To correctly optimize a FreqAI model's hyperparameters, we will create a custom Python script that uses `optuna` to orchestrate the process, leveraging `freqtrade`'s powerful backtesting engine for evaluation.

### 1. Create a Custom Optimization Script

*   **Action:** Create a new Python script, for example, at `user_data/scripts/optimize_ai_model.py`. This script will manage the entire `optuna` study.

### 2. Define the `optuna` Objective Function

*   **Action:** Inside the script, define an `objective` function for `optuna`. For each trial, this function will perform the following steps:

    a.  **Get Hyperparameters:** Use `trial.suggest_*` methods to get a new set of model hyperparameters (e.g., learning rate, number of layers, dropout rate).

    b.  **Create a Temporary Configuration:**
        *   Load the base `config.json` file into a Python dictionary.
        *   Inject the suggested hyperparameters from `optuna` into the `freqai.model_training_parameters` section of this dictionary.
        *   Save this modified dictionary to a temporary config file (e.g., `temp_config.json`).

    c.  **Run `freqtrade backtesting` as a Subprocess:**
        *   Use Python's `subprocess` module to execute the `freqtrade backtesting` command.
        *   Point the command to the temporary config file (`--config temp_config.json`).
        *   This will force FreqAI to train a new model using the hyperparameters for this specific trial and then run a full backtest.
        *   Ensure the backtest results are saved to a predictable JSON file (e.g., using the `--export-filename` option).

    d.  **Parse the Results:**
        *   Once the subprocess is complete, load the JSON file containing the backtest results.

    e.  **Return the Metric to `optuna`:**
        *   Extract the desired performance metric (e.g., Sortino ratio, total profit) from the results.
        *   Return this value to `optuna` so it can guide the optimization process.

### 3. Run the Study

*   **Action:** Execute the custom optimization script from the command line to begin the study.

    ```bash
    python user_data/scripts/optimize_ai_model.py
    ```

This approach provides a robust and correct way to perform hyperparameter optimization on FreqAI models, ensuring that each trial properly evaluates a newly trained model.
