https://www.youtube.com/watch?v=9LO_sy_TVG8&ab_channel=AnushkaTejeswi
Some challenges I faced was getting the actual stock data to enter into my program and have the stock prices change. Another main challenge we faced was getting the graph to update with the prices. Overall, we had a good time learning how to code this and watching everything come together.

Credit:
“C$50 Finance - CS50x.” Harvard.edu, 2020, cs50.harvard.edu/x/2020/tracks/web/finance/.

‌


import time
import pandas as pd
import matplotlib.pyplot as plt
from matplotlib.animation import FuncAnimation
import threading

try:
    import yfinance as yf
except ModuleNotFoundError:
    print("The 'yfinance' library is not installed. Please install it using 'pip install yfinance'")
    exit()

# Global variables for tracking transactions
portfolio = {}
total_profit_loss = 0.0
lock = threading.Lock()

# Function to get real-time stock data
def get_stock_data(ticker):
    stock = yf.Ticker(ticker)
    try:
        data = stock.history(period="1d", interval="1m")
        if data.empty:
            raise ValueError(f"No price data found for {ticker}. It may be delisted or unavailable.")
    except Exception as e:
        print(f"Error fetching data for {ticker}: {e}")
        data = pd.DataFrame()
    return data

# Function to perform analysis on stock data
def analyze_stock_data(data):
    # Simple analysis example: calculate moving average and check price trend
    data['Moving Average'] = data['Close'].rolling(window=5).mean()
    latest_close = data['Close'].iloc[-1]
    moving_average = data['Moving Average'].iloc[-1]
    
    if latest_close > moving_average:
        trend = "uptrend"
    else:
        trend = "downtrend"
    
    return latest_close, moving_average, trend

def buy_sell(stock_data, ticker):
    global portfolio, total_profit_loss
    with lock:
        print(f"Current price of {ticker}: {stock_data['Close'].iloc[-1]}")
        action = input("Would you like to 'buy' or 'sell' this stock? (or 'skip'): ").lower()
        if action == 'buy':
            amount = float(input("Enter the amount you want to buy (in dollars): "))
            price = stock_data['Close'].iloc[-1]
            quantity = amount / price
            if ticker in portfolio:
                portfolio[ticker]['quantity'] += quantity
                portfolio[ticker]['invested'] += amount
            else:
                portfolio[ticker] = {'quantity': quantity, 'invested': amount}
            print(f"You bought ${amount} worth of {ticker} at price {price}")
        elif action == 'sell':
            if ticker in portfolio and portfolio[ticker]['quantity'] > 0:
                amount = float(input("Enter the amount you want to sell (in dollars): "))
                price = stock_data['Close'].iloc[-1]
                quantity = amount / price
                if quantity <= portfolio[ticker]['quantity']:
                    average_price = portfolio[ticker]['invested'] / portfolio[ticker]['quantity']
                    profit_loss = round(quantity * (price - average_price), 6)
                    total_profit_loss += profit_loss
                    
                    portfolio[ticker]['quantity'] -= quantity
                    portfolio[ticker]['invested'] -= amount
                    
                    print(f"You sold ${amount} worth of {ticker} at price {price}")
                    print(f"Profit/Loss from this transaction: ${profit_loss:.6f}")
                else:
                    print("Not enough quantity to sell.")
            else:
                print("You do not own this stock.")
        else:
            print("Skipped trading operation.")
        
        print(f"Total Profit/Loss so far: ${total_profit_loss:.6f}")

def plot_real_time_graph(ticker):
    plt.style.use('ggplot')
    fig, ax = plt.subplots()
    times, prices = [], []

    def update(frame):
        data = get_stock_data(ticker)
        if not data.empty:
            current_time = pd.Timestamp.now()
            current_price = data['Close'].iloc[-1]
            
            times.append(current_time)
            prices.append(current_price)
            
            # Limit the number of points on the graph to avoid overcrowding
            if len(times) > 60:
                times.pop(0)
                prices.pop(0)
            
            # Clear and plot updated data
            ax.clear()
            ax.plot(times, prices, label=f'{ticker} Price', color='b', marker='o', linewidth=1.5)
            ax.set_title(f'Real-Time Stock Price for {ticker}')
            ax.set_xlabel('Time')
            ax.set_ylabel('Price (USD)')
            ax.legend()
            fig.autofmt_xdate()

            # Print analysis results
            latest_close, moving_average, trend = analyze_stock_data(data)
            print(f"Ticker: {ticker}")
            print(f"Latest Close Price: {latest_close}")
            print(f"5-Minute Moving Average: {moving_average}")
            print(f"Trend: {trend}")
            print("=" * 40)

    ani = FuncAnimation(fig, update, interval=2000, save_count=100, cache_frame_data=False)
    plt.show()

def trade(ticker):
    while True:
        data = get_stock_data(ticker)
        if not data.empty:
            buy_sell(data, ticker)
        time.sleep(10)

def main():
    tickers = input("Enter the stock ticker symbols separated by commas (e.g., AAPL, TSLA, MSFT): ").upper().split(',')
    try:
        for ticker in tickers:
            ticker = ticker.strip()
            threading.Thread(target=trade, args=(ticker,)).start()
        # Plotting graph in the main thread to avoid GUI issues
        plot_real_time_graph(tickers[0])
    except KeyboardInterrupt:
        print("\nStopped real-time stock analysis.")
        print(f"Final Total Profit/Loss: ${total_profit_loss:.6f}")

if __name__ == "__main__":
    main()
