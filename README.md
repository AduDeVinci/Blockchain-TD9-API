TD09 - Automating Trading  
Auriane DUPIN, Clara-Belle GININES, Maxence VAYRE  
  
import requests  
import sqlite3  
import time  
import logging  
from datetime import datetime  
import pandas as pd  
import hashlib  
import hmac  
  
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')  
  
pd.set_option('display.max_columns', None)  
pd.set_option('display.precision', 2)  
# Binance API base URL  
BASE_URL = 'https://api.binance.com/api/v3'  
  
# Replace with your actual Binance API key and secret  
API_KEY = 'phEld4naaQBjgvmqxjFFWWcSpuHGFMG3lfxIzHSrbvhsPzR4lVtwQx4H3jMk85SC'  
SECRET_KEY = 'secret_key'  
  
# SQLite database setup  
DB_FILE = 'binance_data.db'  
# Function to create a SQLite table for candle data  
def create_candles_table():  
    conn = sqlite3.connect(DB_FILE)  
    cursor = conn.cursor()  
    cursor.execute('''  
        CREATE TABLE IF NOT EXISTS candles (  
            pair TEXT,  
            timestamp TEXT,  
            open REAL,  
            high REAL,  
            low REAL,  
            close REAL,  
            volume REAL,  
            PRIMARY KEY (pair, timestamp)  
        )  
    ''')  
                     
    conn.commit()  
    conn.close()  
    print("Candles table created.")  
  
# Function to create other required tables  
def create_other_tables():  
    conn = sqlite3.connect(DB_FILE)  
    cursor = conn.cursor()  
      
    # Create a table for full data set  
    cursor.execute('''  
        CREATE TABLE IF NOT EXISTS full_data_set (  
            Id INTEGER PRIMARY KEY,  
            uuid TEXT,  
            traded_crypto REAL,  
            price REAL,  
            created_at_int INT,  
            side TEXT  
        )  
    ''')  
      
    # Create a table for keeping track of updates  
    cursor.execute('''  
        CREATE TABLE IF NOT EXISTS last_checks (  
            Id INTEGER PRIMARY KEY,  
            exchange TEXT,  
            trading_pair TEXT,  
            duration TEXT,  
            table_name TEXT,  
            last_check INT,  
            startdate INT,  
            last_id INT  
        )  
    ''')  
      
    conn.commit()  
    conn.close()  
# Function to refresh trade data and store it in the database  
def refreshDataCandle(pair='BTCUSDT', duration='5m'):  
    url = f"{BASE_URL}/klines?symbol={pair}&interval={duration}"  
    logging.info(f"Requesting data from {url}")  
      
    response = requests.get(url)  
    logging.info(f"Response status code: {response.status_code}")  
      
    if response.ok:  
        logging.info("Data retrieved successfully!")  
        data = response.json()  
  
        try:   
            conn = sqlite3.connect(DB_FILE)  
            cursor = conn.cursor()  
  
            for candle in data:  
                open_time, open, high, low, close, volume, close_time, *_ = candle  
                timestamp = datetime.fromtimestamp(open_time / 1000).strftime('%Y-%m-%d %H:%M:%S')  
  
                cursor.execute('''  
                    INSERT OR REPLACE INTO candles (pair, timestamp, open, high, low, close, volume)  
                    VALUES (?, ?, ?, ?, ?, ?, ?)  
                ''', (pair, timestamp, open, high, low, close, volume))  
  
            conn.commit()  
        finally:  
            conn.close()  
            logging.info(f"Data refreshed for {pair} with duration {duration}")  
    else:  
        logging.error(f"Failed to retrieve data: {response.text}")  
      
      
def refreshData(pair='BTCUSD'):  
    url = f"{BASE_URL}/klines?symbol={pair}&interval=1d"   
    response = requests.get(url)  
    data = response.json()  
  
    conn = sqlite3.connect(DB_FILE)  
    cursor = conn.cursor()  
  
    for candle in data:  
        open_time, open, high, low, close, volume, close_time, *_ = candle  
        timestamp = datetime.fromtimestamp(open_time/1000).strftime('%Y-%m-%d %H:%M:%S')  
  
        cursor.execute('SELECT * FROM my_table WHERE date=? AND pair=?', (timestamp, pair))  
        if cursor.fetchone() is None:  
            cursor.execute('''  
                INSERT INTO my_table (date, high, low, open, close, volume, pair)  
                VALUES (?, ?, ?, ?, ?, ?, ?)  
            ''', (timestamp, high, low, open, close, volume, pair))  
  
    conn.commit()  
    conn.close()  
    print(f"Data refreshed for {pair}")  
def display_all_cryptos(data):  
    df = pd.DataFrame(data)  
    pd.options.display.float_format = '{:.2f}'.format  
    with pd.option_context('display.max_rows', None):  
        print(df.to_string(index=False))  
  
def display_order_book(order_book):  
    bids_df = pd.DataFrame(order_book['bids'], columns=['Price', 'Quantity'])  
    asks_df = pd.DataFrame(order_book['asks'], columns=['Price', 'Quantity'])  
  
    print("Bids:\n", bids_df)  
    print("\nAsks:\n", asks_df)  
def get_all_cryptos():  
    url = f"{BASE_URL}/ticker/price"  
    response = requests.get(url)  
    return response.json()  
  
def getDepth(direction='ask', pair='BTCUSDT'):
    url = f"{BASE_URL}/depth?symbol={pair}&limit=5"
    response = requests.get(url)
    depth = response.json()
    top_price = depth['asks'][0][0] if direction == 'ask' else depth['bids'][0][0]
    return top_price

def getOrderBook(pair='BTCUSDT'):  
    url = f"{BASE_URL}/depth?symbol={pair}&limit=10"  
    response = requests.get(url)  
    return response.json()  
def createOrder(API_KEY, SECRET_KEY, direction, quantity, pair='BTCUSDT', orderType='LimitOrder'):  
    api_url = 'https://api.binance.com/api/v3/order'  
      
    params = {  
        'symbol': pair,  
        'side': direction,  
        'type': orderType,  
        'quantity': str(quantity),  # Convert the quantity to a string  
        'timestamp': int(time.time() * 1000)  
    }  
      
    query_string = '&'.join([f'{k}={v}' for k, v in params.items()])  
    signature = hmac.new(SECRET_KEY.encode(), query_string.encode(), hashlib.sha256).hexdigest()  
      
    headers = {  
        'X-MBX-APIKEY': API_KEY  
    }  
      
    params['signature'] = signature  
      
    # Send a POST request to create the order  
    response = requests.post(api_url, params=params, headers=headers)  
      
    if response.status_code == 200:  
        print(f"Order created successfully: {response.json()}")  
    else:  
        print(f"Error creating order: {response.text}")  
def cancelOrder(API_KEY, SECRET_KEY, orderId):  
    api_url = 'https://api.binance.com/api/v3/order'  
      
    params = {  
        'symbol': 'BTCUSD',  
        'orderId': str(orderId),  
        'timestamp': int(time.time() * 1000)  
    }  
      
    query_string = '&'.join([f'{k}={v}' for k, v in params.items()])  
    signature = hmac.new(SECRET_KEY.encode(), query_string.encode(), hashlib.sha256).hexdigest()  
      
    headers = {  
        'X-MBX-APIKEY': API_KEY  
    }  
      
    params['signature'] = signature  
      
    # Send a DELETE request to cancel the order  
    response = requests.delete(api_url, params=params, headers=headers)  
      
    if response.status_code == 200:  
        print(f"Order canceled successfully: {response.json()}")  
    else:  
        print(f"Error canceling order: {response.text}")  
if __name__ == "__main__":  
    create_candles_table()  
    create_other_tables()  
    print("Other tables created.")  
    refreshDataCandle(pair='BTCUSDT', duration='5m')  
    print("Candle data refreshed.")  
2023-12-11 22:23:31,687 - INFO - Requesting data from https://api.binance.com/api/v3/klines?symbol=BTCUSDT&interval=5m  
Candles table created.  
Other tables created.  
2023-12-11 22:23:31,975 - INFO - Response status code: 200  
2023-12-11 22:23:31,977 - INFO - Data retrieved successfully!  
2023-12-11 22:23:31,994 - INFO - Data refreshed for BTCUSDT with duration 5m  
Candle data refreshed.  
print("List of all available cryptocurrencies:")  
crypto_data = get_all_cryptos()  
display_all_cryptos(crypto_data)  
List of all available cryptocurrencies:  
       symbol             price  
       ETHBTC        0.05377000  
       LTCBTC        0.00176900  
       BNBBTC        0.00591000  
       NEOBTC        0.00028560  
      QTUMETH        0.00140300  
       EOSETH        0.00034440  
       SNTETH        0.00001877  
       BNTETH        0.00032690  
       BCCBTC        0.07908100  
       GASBTC        0.00017530  
       BNBETH        0.10990000  
      BTCUSDT    41063.15000000  
      ETHUSDT     2207.60000000  
       HSRBTC        0.00041400  
       OAXETH        0.00017780  
       DNTETH        0.00002801  
       MCOETH        0.00577200  
      ...  
        
top_ask_price = getDepth(direction='ask', pair='BTCUSDT')  
print(f"Top ask price for BTCUSDT: {top_ask_price}")  
      
order_book_data = getOrderBook('BTCUSDT')  
display_order_book(order_book_data)  
Top ask price for BTCUSDT: 41063.16000000  
Bids:  
             Price    Quantity  
0  41063.15000000  2.12158000  
1  41062.13000000  0.28000000  
2  41062.00000000  0.22220000  
3  41061.99000000  0.90000000  
4  41061.11000000  0.22477000  
5  41061.10000000  0.28000000  
6  41061.09000000  0.37461000  
7  41060.03000000  0.01906000  
8  41060.00000000  0.22220000  
9  41059.44000000  0.02884000  
  
Asks:  
             Price    Quantity  
0  41063.16000000  0.58698000  
1  41063.22000000  0.28020000  
2  41064.00000000  0.22220000  
3  41065.10000000  0.43205000  
4  41065.11000000  0.72000000  
5  41065.21000000  0.00631000  
6  41066.00000000  0.22240000  
7  41066.01000000  0.04048000  
8  41066.44000000  0.00688000  
9  41066.84000000  0.29246000  
direction = 'buy'  
quantity = 0.1  
pair = 'BTCUSDT'   
orderType = 'LimitOrder'  
  
createOrder(API_KEY, SECRET_KEY, direction, quantity, pair, orderType)  
order_id = '123456789'  
cancelOrder(API_KEY, SECRET_KEY, order_id)
Error creating order: {"code":-2015,"msg":"Invalid API-key, IP, or permissions for action."}
Error canceling order: {"code":-2015,"msg":"Invalid API-key, IP, or permissions for action."}
