# Stock Analysis Bot (Google Sheets & FinBERT)

This Python script automates basic technical analysis and optional sentiment analysis for a list of stock tickers provided in a Google Sheet. It fetches historical stock data, calculates technical indicators, determines a BUY/SELL/KEEP recommendation, and writes the decision back to the Google Sheet. The script is designed to run within a Google Colab environment.

## Features

*   **Google Sheets Integration:** Reads stock tickers from a specified column and writes analysis results back to another column in the same sheet.
*   **Historical Data:** Uses `yfinance` to download historical stock price data.
*   **Technical Analysis:** Performs analysis based on Simple Moving Averages (SMA) and long-term price trends to generate a BUY/SELL/KEEP signal.
*   **Sentiment Analysis (Optional):** Integrates FinBERT (a BERT model fine-tuned for financial text) to analyze the sentiment of provided text (currently uses placeholder text) and potentially adjust the technical decision.
*   **Authentication:** Uses Google Colab's authentication mechanism to securely access Google Sheets.
*   **Error Handling:** Includes basic checks for missing data, insufficient data, and other potential errors during analysis.

## Requirements

*   **Environment:** Google Colab Notebook.
*   **Google Account:** Required for authentication to access Google Sheets.
*   **Python 3.x**
*   **Libraries:**
    *   `google-colab`
    *   `google-auth`
    *   `gspread`
    *   `yfinance`
    *   `pandas`
    *   `transformers`
    *   `torch`
    *   `numpy`
*   **Google Sheet:** A Google Sheet accessible by your authenticated Google account.

## Setup

1.  **Copy to Colab:** Copy the Python script into a new Google Colab notebook cell.
2.  **Install Libraries:** Run the following command in a Colab cell to install the necessary libraries:
    ```bash
    pip install gspread google-auth yfinance pandas transformers torch numpy
    ```
3.  **Configure Google Sheet:**
    *   Create a Google Sheet.
    *   In the first column (Column A), starting from the second row (A2), list the stock tickers you want to analyze (e.g., `AAPL`, `GOOGL`, `MSFT`).
    *   Make note of your Google Sheet's exact name and the specific tab (worksheet) name where your tickers are listed.
4.  **Update Script Configuration:**
    *   Modify the line `sheet = gc.open("Stock_values").worksheet("Sheet1")` in the script:
        *   Replace `"Stock_values"` with the exact name of your Google Sheet.
        *   Replace `"Sheet1"` with the exact name of the worksheet/tab containing your tickers.
    *   **Input Column:** The script reads tickers from Column A (index 1). This is hardcoded in `sheet.col_values(1)[1:]`.
    *   **Output Column:** The script writes decisions to Column R (index 18). This is hardcoded in `sheet.update_cell(i, 18, decision)`. Adjust these column indices if necessary.
    *   **(Optional) News Source:** The current script uses placeholder text for FinBERT (`f"Sample news about {quote}"`). For real sentiment analysis, you would need to replace this with code that fetches relevant, recent news headlines or articles for each ticker (e.g., using a news API).

## Usage

1.  **Run the Script:** Execute the Colab cell containing the Python script.
2.  **Authenticate:** You will be prompted to authenticate with your Google account. Follow the instructions to grant Colab access to your Google Drive (which includes Sheets).
3.  **Execution:** The script will iterate through each ticker in Column A:
    *   Fetch historical data.
    *   Perform technical analysis.
    *   Perform sentiment analysis (on placeholder text).
    *   Combine results into a final decision (BUY, SELL, KEEP, or ERROR).
    *   Write the decision to the corresponding row in Column R.
4.  **Check Results:** Once the script finishes, check Column R in your Google Sheet for the analysis results. The `time.sleep(1)` call is included to prevent hitting Google Sheets API rate limits, so processing a long list may take some time.

---

## Explanation of the Technical Analysis Logic (`get_technical_decision` function)

The technical analysis part of the script aims to provide a long-term investment recommendation (BUY, SELL, or KEEP) based on historical price movements, specifically targeting a 3-5 year investment window as mentioned in the function's docstring.

Here's a breakdown of the logic:

1.  **Data Fetching:**
    *   It uses `yfinance` to download the last **5 years** of stock data.
    *   Crucially, it requests **monthly** data (`interval="1mo"`). This means each data point represents the closing price at the end of a month. This choice significantly impacts the indicators calculated later, making them very long-term focused.

2.  **Data Validation:**
    *   Checks if any data was returned (`data.empty`).
    *   Checks if there are at least 50 data points (months) available (`len(prices) < 50`). This is necessary to calculate the 50-month SMA.

3.  **Indicator Calculation:**
    *   **Current Price (`current_price`):** The closing price of the most recent month available in the dataset.
    *   **50-Month Simple Moving Average (SMA50):** Calculates the average closing price over the *last 50 months*. An SMA smooths out price data to show the underlying trend over the specified period. A 50-month SMA represents a very long-term trend (over 4 years).
    *   **200-Month Simple Moving Average (SMA200):** Calculates the average closing price over the *last 200 months* (if 200 months of data exist). This represents an extremely long-term trend (over 16 years). *Note: Given 5 years (60 months) of data is downloaded, the SMA200 calculation will likely often result in `None` unless the stock has a very long history available via yfinance beyond the explicit 5y request.* The code *does* check `len(prices) >= 200` before calculating it.
    *   **3-Year Price (`three_year_price`):** The closing price from 36 months prior to the most recent month. This is used for a direct lookback comparison.

4.  **Decision Logic:**
    *   **Default:** The initial decision is set to `KEEP`.
    *   **Primary Rule (SMA Crossover - Long-Term):**
        *   This logic attempts to use the relationship between the SMA50, SMA200, and the current price. This is inspired by the "Golden Cross" (SMA50 crosses above SMA200 - bullish) and "Death Cross" (SMA50 crosses below SMA200 - bearish) concepts, but adapted for *monthly* data and adding a current price condition.
        *   **BUY Condition:** `sma50 > sma200 and current_price > sma50`. This requires:
            *   The shorter-term average (50-month) is above the longer-term average (200-month), indicating a potential long-term uptrend (monthly Golden Cross).
            *   The current price is also above the 50-month average, suggesting current strength reinforcing the uptrend.
        *   **SELL Condition:** `sma50 < sma200 and current_price < sma50`. This requires:
            *   The shorter-term average is below the longer-term average, indicating a potential long-term downtrend (monthly Death Cross).
            *   The current price is also below the 50-month average, suggesting current weakness reinforcing the downtrend.
        *   *Important Caveat:* Because this uses *monthly* data, these signals are extremely slow-moving and indicate very major, long-term shifts in trend. They will lag significantly compared to daily SMA crossovers. This part of the logic might only trigger for stocks with exceptionally long, established trends and sufficient history (for SMA200).
    *   **Secondary Rule (Fallback - 3-Year Trend):**
        *   This rule is only checked if the SMA Crossover logic resulted in `KEEP` (i.e., the conditions for BUY or SELL above were not met, or SMA200 was not available).
        *   It directly compares the current price to the price 3 years ago.
        *   **BUY Condition:** `current_price > three_year_price * 1.2`. If the stock has appreciated by more than 20% over the last 3 years, it's considered a BUY signal (indicating strong multi-year growth).
        *   **SELL Condition:** `current_price < three_year_price * 0.8`. If the stock has depreciated by more than 20% over the last 3 years, it's considered a SELL signal (indicating significant multi-year decline).

5.  **Final Output:** The function returns the calculated decision (`BUY`, `SELL`, `KEEP`) or an `ERROR` message if something went wrong (e.g., insufficient data).

**In summary:** The technical analysis uses very long-term monthly SMAs (if enough history exists) as the primary signal for major trend shifts. If those signals are inconclusive or unavailable, it falls back to checking if the stock has experienced significant (>20%) growth or decline over the past 3 years. This combination aims to identify stocks suitable for a long-term (multi-year) investment perspective based on sustained trends.

---

## Notes & Limitations

*   **Colab Dependency:** The script uses `google.colab.auth` and is intended for the Colab environment. Running it elsewhere would require modifying the authentication method.
*   **Placeholder Sentiment Analysis:** The FinBERT integration currently uses static placeholder text. To make it meaningful, you need to integrate a real news fetching mechanism.
*   **Technical Indicator Simplification:** The technical analysis uses a specific set of rules based on long-term monthly SMAs and a 3-year price trend. This is a simplified approach and may not be suitable for all trading strategies or market conditions. The use of *monthly* data for 50/200 period SMAs results in *very* long-term signals (approx 4 years / 16 years).
*   **API Rate Limits:** The `time.sleep(1)` helps manage Google Sheets API limits but might still be insufficient for very large sheets or under heavy API usage.
*   **No Financial Advice:** This script is for educational and illustrative purposes only. The generated signals are based on simple rules and should **not** be considered financial advice. Always conduct thorough research and consult with a qualified financial advisor before making investment decisions.
