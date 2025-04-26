# Stock Analysis Bot (Google Sheets & FinBERT) - 1-3 Year Horizon

This Python script automates basic technical analysis and optional sentiment analysis for a list of stock tickers provided in a Google Sheet. It fetches historical stock data using a **weekly** interval, calculates technical indicators suitable for a **1-3 year investment outlook**, determines a BUY/SELL/KEEP recommendation, and writes the decision back to the Google Sheet. The script is designed to run within a Google Colab environment.

## Features

*   **Google Sheets Integration:** Reads stock tickers from a specified column and writes analysis results back to another column in the same sheet using efficient batch updates.
*   **Historical Data:** Uses `yfinance` to download 5 years of **weekly** historical stock price data.
*   **Technical Analysis (1-3 Year Focus):** Performs analysis based on **weekly** Simple Moving Averages (SMA 50w, 200w) and 1-year / 3-year price trends to generate a BUY/SELL/KEEP signal tailored for a medium-term outlook.
*   **Sentiment Analysis (Optional):** Integrates FinBERT (a BERT model fine-tuned for financial text) to analyze the sentiment of provided text (currently uses placeholder text). Sentiment can moderate technical signals (e.g., downgrade a BUY to KEEP if sentiment is negative).
*   **Authentication:** Uses Google Colab's authentication mechanism to securely access Google Sheets.
*   **Error Handling:** Includes checks for missing data, insufficient data, API errors, and other potential issues during analysis and sheet updates. Enhanced logging provides better visibility.

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
*   **Google Sheet:** A Google Sheet accessible by your authenticated Google account with write permissions.

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
    *   **Output Column:** The script writes decisions to Column R (index 18). This is hardcoded in `sheet.update_cells(cell_list)`. Adjust these column indices if necessary.
    *   **(Optional) News Source:** The current script uses placeholder text for FinBERT (`f"General market news today for {ticker}"`). For real sentiment analysis, you would need to replace this with code that fetches relevant, recent news headlines or articles for each ticker (e.g., using a news API).

## Usage

1.  **Run the Script:** Execute the Colab cell containing the Python script.
2.  **Authenticate:** You will be prompted to authenticate with your Google account. Follow the instructions to grant Colab access to your Google Drive (which includes Sheets).
3.  **Execution:** The script will iterate through each ticker in Column A:
    *   Fetch 5 years of weekly historical data.
    *   Perform technical analysis for a 1-3 year horizon.
    *   Perform sentiment analysis (on placeholder text).
    *   Combine results into a final decision (BUY, SELL, KEEP, or ERROR).
    *   Prepare results for batch writing.
4.  **Write Results:** After processing all tickers, the script writes all decisions to the corresponding rows in Column R in a single batch operation.
5.  **Check Results:** Once the script finishes ("Successfully wrote results to Google Sheet." or similar message), check Column R in your Google Sheet for the analysis results.

---

## Explanation of the Technical Analysis Logic (`get_technical_decision` function - 1-3 Year Horizon)

The technical analysis logic is designed to provide a BUY, SELL, or KEEP recommendation focused on a **1-3 year investment timeframe**, using **weekly** price data.

Here's a breakdown:

1.  **Data Fetching:**
    *   Downloads the last **5 years** of stock data to ensure sufficient history for calculations.
    *   Uses **weekly** intervals (`interval="1wk"`). Each data point represents the closing price for a week.

2.  **Data Validation:**
    *   Checks if data was returned.
    *   Requires at least **50 weeks** of data to calculate the 50-week SMA.

3.  **Indicator Calculation (Weekly):**
    *   **Current Price (`current_price`):** The closing price of the most recent week available.
    *   **50-Week Simple Moving Average (SMA50w):** The average closing price over the *last 50 weeks* (approx. 1 year). Represents the medium-term trend.
    *   **200-Week Simple Moving Average (SMA200w):** The average closing price over the *last 200 weeks* (approx. 4 years), if available. Represents the long-term underlying trend.
    *   **1-Year Price (`one_year_price`):** The closing price from 52 weeks prior.
    *   **3-Year Price (`three_year_price`):** The closing price from 156 weeks prior.

4.  **Decision Logic (Layered Approach):**
    *   **Default:** Starts with `KEEP`.
    *   **1. Primary Signal (Weekly SMA Crossover):** Checks the relationship between the 50w SMA, 200w SMA, and current price. This is akin to weekly Golden/Death Cross signals.
        *   **BUY:** If `sma50w > sma200w` (medium trend above long trend) AND `current_price > sma50w` (current strength).
        *   **SELL:** If `sma50w < sma200w` (medium trend below long trend) AND `current_price < sma50w` (current weakness).
    *   **2. Secondary Signal (1-Year Momentum):** If the primary signal results in `KEEP`, this checks recent performance.
        *   **BUY:** If `current_price > one_year_price * 1.15` (gained >15% in the last year).
        *   **SELL:** If `current_price < one_year_price * 0.85` (lost >15% in the last year).
    *   **3. Tertiary Signal (3-Year Confirmation):** If the decision is still `KEEP`, this looks for stronger, sustained trends over a longer period.
        *   **BUY:** If `current_price > three_year_price * 1.25` (gained >25% over 3 years).
        *   **SELL:** If `current_price < three_year_price * 0.80` (lost >20% over 3 years).

5.  **Final Output:** Returns the calculated decision (`BUY`, `SELL`, `KEEP`) or an `ERROR` message (e.g., `ERROR: No data`, `ERROR: Insufficient data (need 50w)`).

**Sentiment Integration (`main` function):**
*   After the technical decision is made, sentiment analysis is performed (currently on placeholder text).
*   If the technical decision was `KEEP`, positive sentiment changes it to `BUY`, and negative sentiment changes it to `SELL`.
*   If the technical decision was `BUY` but sentiment is `negative`, the decision is changed to `KEEP` (acting as a safety check).
*   If the technical decision was `SELL` but sentiment is `positive`, the decision is changed to `KEEP`.

**In summary:** The analysis prioritizes the medium-term trend (weekly SMA crossover) and then uses 1-year momentum and 3-year performance as secondary/tertiary signals for a 1-3 year investment outlook. Sentiment acts as a final layer of refinement or caution.

---

## Notes & Limitations

*   **Colab Dependency:** Uses `google.colab.auth`, intended for Colab. Running elsewhere requires changing the authentication method.
*   **Placeholder Sentiment Analysis:** FinBERT integration uses static text. **Replace with a real news source for meaningful sentiment analysis.**
*   **Technical Indicator Simplification:** Uses specific SMA crossover and percentage change rules. This is a defined strategy but may not capture all market nuances or suit all investment styles. Backtesting is recommended before relying on signals.
*   **Weekly Data:** Using weekly data smooths price action compared to daily but is more responsive than monthly data, suitable for the 1-3 year target horizon. Signals will still lag compared to shorter-term strategies.
*   **API Rate Limits:** Batch updates help manage Google Sheets API limits, but issues could still occur with extremely large lists or heavy API usage.
*   **No Financial Advice:** This script is for educational and illustrative purposes only. Generated signals are based on predefined rules and **should not be considered financial advice**. Always conduct thorough research and consult a qualified financial advisor before investing.
