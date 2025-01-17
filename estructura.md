# Bot Trading Project Structure

## Índice

- [Root Directory](#root-directory)
- [Modules](#modules)
  - [config/](#config)
    - [settings.py](#settingspy)
    - [constants.py](#constantspy)
    - [strategies.json](#strategiesjson)
  - [data/](#data)
    - [fetching.py](#fetchingpy)
    - [processing.py](#processingpy)
  - [db/](#db)
    - [operations.py](#operationspy)
    - [schema.sql](#schemasql)
    - [backup.py](#backuppy)
    - [cleaning.py](#cleaningpy)
  - [strategies/](#strategies)
    - [core.py](#corepy)
    - [events/](#events)
      - [breakout.py](#breakoutpy)
    - [prompts.py](#promptspy)
    - [tests/](#tests)
  - [chatgpt/](#chatgpt)
    - [integration.py](#integrationpy)
    - [cache.py](#cachepy)
    - [prompts.py](#promptspy-1)
  - [trading/](#trading)
    - [execution.py](#executionpy)
    - [positions.py](#positionspy)
    - [risk_management.py](#risk_managementpy)
  - [gui/](#gui)
    - [dashboard.py](#dashboardpy)
    - [settings.py](#settingspy-1)
  - [backtesting/](#backtesting)
    - [simulation.py](#simulationpy)
    - [analysis.py](#analysispy)
    - [reports.py](#reportspy)
  - [utils/](#utils)
    - [logging.py](#loggingpy)
    - [api_helpers.py](#api_helperspy)
    - [error_handling.py](#error_handlingpy)

## Root Directory

```plaintext
bot_trading/
├── main.py
│   - Entry point to start the bot.
├── config/
├── data/
├── db/
├── strategies/
├── chatgpt/
├── trading/
├── gui/
├── backtesting/
└── utils/
```

---

## Modules

<details>
  <summary><strong>config/</strong></summary>

  ### settings.py

  ```plaintext
  - Global configurations, API keys, and parameters.
  - Stores critical settings for the bot such as:
      - API keys (Binance, ChatGPT, etc.).
      - Default trading pairs and initial parameters.
  ```

  ### constants.py

  ```plaintext
  - Common constants used throughout the project.
  - Examples include:
      - Default time intervals (1D, 4H, 1H, 15M).
      - Base URLs for APIs.
  ```

  ### strategies.json

  ```plaintext
  - (Optional) Predefined strategies file.
  - Allows users to load or save trading strategies dynamically.
  ```

</details>

<details>
  <summary><strong>data/</strong></summary>

  ### fetching.py

  ```plaintext
  - Fetches data from the API and WebSocket (1D, 4H, 1H, 15M).
  - Handles rate limits and monitoring to ensure compliance with Binance API policies.
  - At first will take BTC/USDT, later on addig more symbols. For 1.0.0 the pairs will be hardcoded and not selected. Future version may have dynamic and manual selection
  - Key Responsibilities:
      - Initialize connections to REST API and WebSocket.
      - Pull historical data for each symbol and timeframe.
      - Listen to live updates for time-sensitive strategies.
  - Output:
      - Provides WebSocket data directly to `processing.py` for real-time calculations (e.g., 15M indicators).
      - Stores API data in `db/operations.py` for further processing by `processing.py`.
  - Notes:
      - Fetching will store all temporalities (1D, 4H, 1H, 15M) in `db/candles`.
      - Adds one candle at a time for the corresponding timeframe.
      - Data for 15M is passed directly to `processing.py` for immediate calculations.
  ```

  ### processing.py

  ```plaintext
  - Processes raw data and calculates technical indicators:
      - **1D Indicators:**
          - EMA/SMA (14, 50): Identifies macro trends.
          - MACD: Confirms market momentum.
          - RSI (14): Detects overbought/oversold conditions in a broad context.
      - **4H Indicators:**
          - ATR (14): Measures volatility to define dynamic support/resistance levels.
          - Volume Average: Highlights accumulation/distribution zones.
          - RSI (14): Validates potential divergences.
      - **1H Indicators:**
          - EMA/SMA (5, 14): Tracks microtrends.
          - Volume: Confirms short-term accumulation/distribution.
      - **15M Indicators:**
          - ATR (5): Validates volatility for breakout detection.
          - Volume Spike: Flags anomalies in market activity.
  - Input:
      - **1D, 4H, 1H data:** Retrieved from `db/operations.py`.
      - **15M data:** Received directly from `fetching.py` via WebSocket.
  - Output:
      - Indicators calculated for all temporalities (1D, 4H, 1H, 15M) and stored in `db/indicators`.
      - Data flow:
          - **Primary:** Processed indicators saved to `db/indicators`.
          - **Secondary (real-time):** Certain indicators for 15M could flow directly to `strategies/` to reduce latency when necessary.
  ```

</details>

<details>
  <summary><strong>db/</strong></summary>

  ### operations.py

  ```plaintext
  - CRUD for database operations.
  - Handles storage of:
      - Candle data for different symbols and timeframes.
      - Precomputed indicators for historical analysis.
  - Supports efficient queries for `strategies` and `backtesting`.
  - Output:
      - Provides stored data (e.g., 1D, 4H, 1H candles) to `processing.py` for indicator calculation.
      - Supplies indicators from `db/indicators` to `strategies/` for event detection and decision-making.
  - Additional Logging:
      - Logs critical database operations such as failed inserts or updates.
      - Optionally integrates with `utils/logging.py` for centralized logging.
  ```

  ### schema.sql

  ```plaintext
  - Defines the initial database schema.
  - Includes:
      - Tables for each symbol and timeframe.
      - Indexes for quick lookups.
      - Triggers:
          - Prevent duplicate candles from being inserted.
          - Automate basic indicator calculations for simplicity in `processing.py`.
          - Ensure referential integrity between `candles` and `indicators`.
  - Recommendations:
      - Use foreign keys to maintain relationships between `candles` and `indicators`.
      - Create materialized views for complex queries in `strategies/`.
  ```

  ### backup.py

  ```plaintext
  - Automates backups and restoration of the database.
  - Key Features:
      - Scheduled exports to prevent data loss.
      - Cloud storage support (Google Cloud, AWS S3).
      - Rotational backup system to manage storage efficiently.
      - Logs backup activity in `metadata` table.
      - Notifications for failed backups to ensure reliability.
  ```

  ### cleaning.py

  ```plaintext
  - Removes duplicates and validates data integrity.
  - Ensures consistency in candle data and computed indicators.
  - Notes:
      - Relies on database triggers for primary deduplication.
      - Secondary cleaning performed to validate missing or incomplete data.
      - Example Logic:
          1. Identify gaps or inconsistencies in `candles` and request missing data from `fetching.py`.
          2. Validate indicator calculations and recompute if anomalies are found.
      - Future Improvement:
          - Integrate with `utils/logging.py` to record anomalies detected during cleaning.
  ```
</details>

<details>
  <summary><strong>db/</strong></summary>

  ### training.py

  ```plaintext
- Handles training of machine learning models (e.g., XGBoost).
- Key Responsibilities:
    - Load and preprocess historical data for BTC/USDT and ETH/USDT.
    - Train models to identify breakouts/rebotes and time to event completion.
    - Save trained models for deployment.
- Inputs:
    - Historical candles and indicators from `db/operations.py`.
    - Labels: Breakout/rebote and number of candles to event.
- Outputs:
    - Trained model saved locally for use by `predictions.py`.
    - Logs of training performance and validation metrics.
  ```

  ### predictions.sql

  ```plaintext
- Generates predictions based on trained ML models.
- Key Responsibilities:
    - Load pre-trained models.
    - Process live data from `data/processing.py` to classify events (e.g., breakout probability).
    - Integrate predictions into `strategies/` for decision-making.
- Inputs:
    - Real-time indicators and processed candles.
- Outputs:
    - Event classification (breakout/rebote).
    - Probability scores for each event type.
- Notes:
    - Predictions act as a filter to prioritize relevant events for `chatgpt/`.
  ```



</details>

<details>
  <summary><strong>strategies/</strong></summary>

  Input: from db/, just if necessary 15min indicators from processing.py

  ### core.py

  ```plaintext
  - Calculates support and resistance levels based on:
      - Historical highs and lows.
      - ATR to dynamically adjust levels.
  ```

  ### events/

  #### breakout.py

  ```plaintext
  - Detects breakout events:
      - Identifies when price breaks support or resistance levels.
      - Uses ATR, volume, and price action to confirm breakouts.
  ```

  #### (future events)

  ```plaintext
  - Each event (e.g., volume shifts, trend changes) will have its own module.
  ```

  ### prompts.py

  ```plaintext
  - Generates prompts for ChatGPT based on detected events and context.
  - Examples:
      - "The price is nearing resistance at $45000. RSI is 65. Should we expect a breakout?"
  - Centralized for easy modification of prompt templates.
  ```

  ### tests/

  ```plaintext
  - Unit tests for calculations and event detection.
  - Example Tests:
      - Ensure ATR calculations are consistent.
      - Validate breakout detection logic with edge cases.
  ```

</details>

<details>
  <summary><strong>chatgpt/</strong></summary>

  ### integration.py

  ```plaintext
  - Manages API interaction with ChatGPT for decision analysis.
  - Handles system prompts and context management to ensure:
      - Efficient responses from ChatGPT.
      - Contextual consistency across queries.
  ```

  ### cache.py

  ```plaintext
  - Optimizes requests by caching frequent responses.
  - Avoids redundant queries to reduce latency and cost.
  ```

  ### prompts.py

  ```plaintext
  - Templates and structures for dynamic prompts.
  - Supports integration with `strategies` to format detected events.
  ```

</details>

<details>
  <summary><strong>trading/</strong></summary>

  ### execution.py

  ```plaintext
  - Sends market orders (buy, sell) based on ChatGPT recommendations.
  - Handles order management:
      - Retry logic for failed orders.
      - Real-time status updates.
  ```

  ### positions.py

  ```plaintext
  - Records and monitors active positions.
  - Examples:
      - Track open trades and their profit/loss status.
      - Monitor average entry prices and exposure.
  ```

  ### risk_management.py

  ```plaintext
  - Implements:
      - Stop-loss: Limits potential losses.
      - Take-profit: Locks in profits at target levels.
      - Trailing stop-loss: Dynamically adjusts to market movements using ATR.
  - Fully configurable based on strategy risk tolerance.
  ```

</details>

<details>
  <summary><strong>gui/</strong></summary>

  ### dashboard.py

  ```plaintext
  - Displays:
      - Open/closed positions with profit/loss.
      - API rate and token monitoring for Binance.
  - Simple controls:
      - Enable/Disable bot trading.
      - Pause/Resume data fetching.
  ```

  ### settings.py

  ```plaintext
  - Allows editing bot configurations such as:
      - Adding new trading pairs.
      - Adjusting risk parameters.
  ```

</details>

<details>
  <summary><strong>backtesting/</strong></summary>

  ### simulation.py

  ```plaintext
  - Runs strategies on historical data to validate performance.
  - Supports multi-timeframe testing to ensure robustness.
  ```

  ### analysis.py

  ```plaintext
  - Analyzes backtesting results to highlight:
      - Maximum drawdown.
      - Sharpe ratio and win rate.
  ```

  ### reports.py

  ```plaintext
  - Generates detailed performance reports.
  - Output Formats:
      - CSV for data analysis.
      - PDF for easy sharing.
  ```

</details>

<details>
  <summary><strong>utils/</strong></summary>

  ### logging.py

  ```plaintext
  - Centralized logging management:
      - Tracks key events and errors.
      - Creates log files for auditing and debugging.
  ```

  ### api_helpers.py

  ```plaintext
  - Helper functions for Binance API interactions:
      - Constructing queries.
      - Parsing API responses.
  ```

  ### error_handling.py

  ```plaintext
  - Handles common errors across modules:
      - API rate limit exceptions.
      - Connection retries for WebSocket and REST API.
  ```

</details>
