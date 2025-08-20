# Project Migration Plan: From Custom ML Framework to Freqtrade

This document provides a detailed plan to migrate the concepts and components from the custom machine learning trading framework (previously in `combined_output.txt`) into the `freqtrade` platform. The goal is to create a self-contained guide that explains the mapping without needing to reference the original project's source code.

---

## 1. Machine Learning Integration (PPO Model)

*   **Previous Project:** The original project used a custom-built Proximal Policy Optimization (PPO) agent (`src/ppo_agent.py`) with a dedicated training script (`src/train_ppo.py`). The environment was defined in `src/environment_v2.py`. This setup was built on PyTorch and required manual orchestration.

*   **Freqtrade Equivalent (The Plan):**
    *   **Use FreqAI:** This is `freqtrade`'s dedicated module for integrating ML models. It handles data pipelines, training triggers, and prediction calls automatically.
    *   **Create a FreqAI Strategy:** In the `user_data/strategies/` folder, create a new strategy class that inherits from `IFreqaiStrategy`. This class will define the feature engineering and signal generation logic.
    *   **Create a Custom Model:** In a new file (e.g., `user_data/freqaimodels/PPOModel.py`), create a class that inherits from `IFreqaiModel`.
        *   Port the PyTorch model architecture (e.g., `ActorTransformer` or `ActorMLP` from `src/ppo_agent.py`) into this new class.
        *   Move the model training logic from `src/train_ppo.py` into the `train()` method of your `IFreqaiModel` class. FreqAI will provide the training data directly to this method.
        *   Implement the `predict()` method to generate buy/sell signals from your model's output.

---

## 2. Configuration and Experiment Management

*   **Previous Project:** Configuration was spread across multiple files: `settings.py` for API keys, `src/config_models.py` for defining experiment structure with Python dataclasses, and `src/experiment_manager.py` for handling the creation and deletion of experiment folders on the filesystem.

*   **Freqtrade Equivalent (The Plan):**
    *   **Centralize in `config.json`:** `freqtrade` uses a single JSON file for all configuration, which simplifies management.
    *   **ML Parameters:** Define all ML-specific parameters (feature settings, data split ratios, model hyperparameters) within the `"freqai": {}` block of `config.json`.
    *   **Experiment Separation:**
        *   Use the `identifier` key within the `freqai` config. FreqAI saves all models and data for a run under a folder named after this identifier, keeping experiments cleanly separated.
        *   To run entirely different experiments, you can simply use different config files and specify which one to use at runtime (e.g., `freqtrade trade --config experiment_2.json`).

---

## 3. Data and Feature Engineering

*   **Previous Project:** The `src/data_processor_alpaca.py` script was responsible for fetching historical data from the Alpaca API and calculating a predefined set of technical indicators using the `talib` library.

*   **Freqtrade Equivalent (The Plan):**
    *   **Data Download:** Use `freqtrade`'s built-in data download command: `freqtrade download-data`. This tool downloads and stores data for all pairs specified in your `config.json` from your chosen exchange.
    *   **Feature Calculation:** Implement all feature engineering logic inside the `populate_any_indicators()` method of your FreqAI strategy class. This method receives a dataframe and should return it with all the calculated features (indicators) that your model needs for training and prediction. This keeps the feature logic tightly coupled with the strategy that uses it.

---

## 4. Backtesting and Optimization

*   **Previous Project:** The framework used the `vectorbt` library for backtesting (seen in `run_final_evaluation`) and `optuna` for hyperparameter optimization. These processes were kicked off by `run_task.py` and visualized in a custom Streamlit dashboard.

*   **Freqtrade Equivalent (The Plan):**
    *   **Integrated Backtesting:** Use the `freqtrade backtesting` command. It provides a comprehensive performance report, including trade details, profit analysis, and key metrics like Sortino and Sharpe ratios, without needing any custom code.
    *   **Integrated Optimization:** Use the `freqtrade hyperopt` command. It leverages `optuna` internally to find the optimal parameters for your strategy's indicators and buy/sell logic, saving you the effort of building and maintaining your own optimization scripts.

---

## 5. Live Trading

*   **Previous Project:** A dedicated `live_trader` engine was built with its own state manager (`state_manager.py`), exchange connector (`binance_connector.py`), and main loop (`main.py`).

*   **Freqtrade Equivalent (The Plan):**
    *   **Use the Core Engine:** This is `freqtrade`'s primary function. The `freqtrade trade` command runs a robust, battle-tested trading bot.
    *   **Dry-Run Mode:** Always start with `freqtrade trade --dry-run`. This allows you to run your strategy with live data and simulate trades without risking real funds.
    *   **Live Mode:** Once you are confident in your strategy's performance, you can switch to live trading by removing the `--dry-run` flag.

---

## 6. UI / Dashboard

*   **Previous Project:** The project included two separate UIs: a multi-page Streamlit application (`research_hub/`) for managing experiments and a Plotly Dash application (`live_trader_dashboard.py`) for monitoring the live bot.

*   **Freqtrade Equivalent (The Plan):**
    *   **Use FreqUI:** `freqtrade` comes with a built-in, all-in-one web interface.
    *   **Enable the API Server:** Add the `"api_server": {}` configuration to your `config.json` and set `"enabled": true`.
    *   **Monitor and Control:** FreqUI allows you to monitor performance, view charts with trade history, inspect logs, and manually manage trades, replacing the need for custom-built dashboards.

---

This plan provides a clear path to leverage `freqtrade`'s powerful, integrated features while reusing the core machine learning logic from your previous project.
