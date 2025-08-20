# FreqAI Reinforcement Learning Demonstration Notes

This document provides a brief overview of the files created for the FreqAI Reinforcement Learning demonstration.

## Files

### 1. RL Model: `user_data/freqaimodels/MyCoolRLModel.py`

This file contains the core Reinforcement Learning model logic. It defines a custom environment `MyRLEnv` where you can implement your own `calculate_reward` function. This function is the heart of your RL strategy, as it determines what actions the agent is rewarded or penalized for. The provided example includes logic to reward entering trades and penalize holding positions for too long.

### 2. Strategy: `user_data/strategies/MyRLStrategy.py`

This is the strategy file that works with the `MyCoolRLModel`. It's a simplified strategy that doesn't require complex indicators for entries and exits. Instead, it listens for actions from the RL agent.

-   **Action 1**: Enter a long position.
-   **Action 2**: Exit a long position.
-   **Action 3**: Enter a short position.
-   **Action 4**: Exit a short position.
-   **Action 0**: Do nothing (neutral).

### 3. Configuration: `user_data/config-rl.json`

This is a sample configuration file to run the RL strategy. It's set up for dry-run on the Binance exchange and includes the necessary `freqai` and `rl_config` sections to enable the `MyCoolRLModel`.

## How to Run

To run the Reinforcement Learning bot, use the following command from the root of your `freqtrade` directory:

```bash
freqtrade trade --freqaimodel MyCoolRLModel --strategy MyRLStrategy --config user_data/config-rl.json
```

## Monitoring with Tensorboard

You can monitor the training progress and performance of your RL model using Tensorboard. Run the following command in a separate terminal:

```bash
tensorboard --logdir user_data/models/MyCoolRLModel
```

Then, open your web browser and navigate to `http://localhost:6006` to view the Tensorboard dashboard.

## How to Backtest

Backtesting a FreqAI reinforcement learning strategy requires a pre-trained model.

### 1. Train a Model

First, you need to train a model by running the bot in trade mode (dry-run or live). This will save a trained model file.

```bash
freqtrade trade --freqaimodel MyCoolRLModel --strategy MyRLStrategy --config user_data/config-rl.json
```

Let it run long enough to save at least one model.

### 2. Run the Backtest

Once a model is saved, you can run a backtest using the following command:

```bash
freqtrade backtesting --strategy MyRLStrategy --config user_data/config-rl.json
```

FreqAI will automatically load the latest trained model for the `MyCoolRLModel` identifier.
