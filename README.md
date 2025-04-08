# -import ccxt
import time
import pandas as pd

# Borsa ve API bilgilerinin ayarlanması
exchange = ccxt.binance({
    'apiKey': 'SİZİN_API_KEY',
    'secret': 'SİZİN_API_SECRET',
    'enableRateLimit': True,  # API istek sınırlarına uyum
})

# İşlem yapılacak sembol ve zaman dilimi (örneğin 1 dakikalık grafik)
symbol = 'BTC/USDT'
timeframe = '1m'
limit = 100  # geçmiş veriden kaç kandle çalışılacak

# Basit Hareketli Ortalama (SMA) hesaplama fonksiyonu
def calculate_sma(data, window):
    return data.rolling(window=window).mean()

def fetch_ohlcv(symbol, timeframe, limit):
    """Borsadan geçmiş OHLCV verisini getirir."""
    ohlcv = exchange.fetch_ohlcv(symbol, timeframe=timeframe, limit=limit)
    df = pd.DataFrame(ohlcv, columns=['timestamp', 'open', 'high', 'low', 'close', 'volume'])
    df['timestamp'] = pd.to_datetime(df['timestamp'], unit='ms')
    return df

def check_buy_signal(df, short_window=5, long_window=20):
    """Kısa ve uzun SMA karşılaştırması ile alım sinyali üretir.
    Basit mantık: kısa dönem SMA, uzun dönem SMA'yı yukarı keserse alım yap.
    """
    df['SMA_short'] = calculate_sma(df['close'], window=short_window)
    df['SMA_long'] = calculate_sma(df['close'], window=long_window)
    
    # En son iki kandle bakarak kesişim kontrolü
    if len(df) < long_window + 1:
        return False  # yeterli veri yoksa sinyal verme
    
    prev_short = df['SMA_short'].iloc[-2]
    prev_long = df['SMA_long'].iloc[-2]
    curr_short = df['SMA_short'].iloc[-1]
    curr_long = df['SMA_long'].iloc[-1]

    if (prev_short < prev_long) and (curr_short > curr_long):
        return True
    return False

def execute_buy(symbol, amount):
    """Alım emrini çalıştırır.
    Bu örnekte gerçek emir gönderimi yer almakta; canlı ortamda test net kullanmanız önerilir.
    """
    try:
        order = exchange.create_market_buy_order(symbol, amount)
        print("Alım emri gönderildi:", order)
    except Exception as e:
        print("Alım emri gönderilirken hata:", e)

def main():
    # İşlem miktarını belirleyin (örneğin, 0.001 BTC)
    amount = 0.001
    while True:
        try:
            df = fetch_ohlcv(symbol, timeframe, limit)
            if check_buy_signal(df):
                print("Alım sinyali tespit edildi. İşlem gerçekleştiriliyor...")
                execute_buy(symbol, amount)
            else:
                print("Alım sinyali yok. Bir sonraki kontrol bekleniyor...")
        except Exception as e:
            print("Veri çekme veya işlem sırasında hata oluştu:", e)
        
        # Her döngüde örneğin 1 dakika bekle (timeframe'e uygun)
        time.sleep(60)

if __name__ == "__main__":

   
