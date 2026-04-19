import requests
import pandas as pd
import numpy as np
import time
from datetime import datetime

# ── AYARLAR ──────────────────────────────────────────────
TELEGRAM_TOKEN = "BURAYA_TOKEN_YAZ"
CHAT_ID        = "BURAYA_CHAT_ID_YAZ"
INTERVAL       = "15m"
T3_LENGTH      = 8
T3_FACTOR      = 0.7
CHECK_EVERY    = 60 * 14  # 14 dakikada bir kontrol (15m mum kapanmadan önce)
# ─────────────────────────────────────────────────────────

def send_telegram(msg):
    url = f"https://api.telegram.org/bot{TELEGRAM_TOKEN}/sendMessage"
    requests.post(url, data={"chat_id": CHAT_ID, "text": msg, "parse_mode": "HTML"})

def get_margin_symbols():
    url = "https://api.binance.com/api/v3/exchangeInfo"
    data = requests.get(url).json()
    margin_url = "https://api.binance.com/sapi/v1/margin/allPairs"
    try:
        margin_data = requests.get(margin_url).json()
        symbols = [x["symbol"] for x in margin_data if x["symbol"].endswith("USDT")]
    except:
        # fallback: tüm aktif USDT pariteler
        symbols = [
            s["symbol"] for s in data["symbols"]
            if s["quoteAsset"] == "USDT" and s["status"] == "TRADING"
        ]
    return symbols

def get_klines(symbol, interval="15m", limit=100):
    url = "https://api.binance.com/api/v3/klines"
    params = {"symbol": symbol, "interval": interval, "limit": limit}
    r = requests.get(url, params=params)
    if r.status_code != 200:
        return None
    data = r.json()
    closes = [float(x[4]) for x in data]
    highs  = [float(x[2]) for x in data]
    lows   = [float(x[3]) for x in data]
    return closes, highs, lows

def ema(series, period):
    s = pd.Series(series)
    return s.ewm(span=period, adjust=False).mean().tolist()

def calc_t3(closes, highs, lows, length=8, factor=0.7):
    # hlcc4 = (high + low + close + close) / 4
    src = [(highs[i] + lows[i] + closes[i] + closes[i]) / 4 for i in range(len(closes))]
    e1 = ema(src, length)
    e2 = ema(e1, length)
    e3 = ema(e2, length)
    e4 = ema(e3, length)
    e5 = ema(e4, length)
    e6 = ema(e5, length)
    a  = factor
    c1 = -a**3
    c2 = 3*a**2 + 3*a**3
    c3 = -6*a**2 - 3*a - 3*a**3
    c4 = 1 + 3*a + a**3 + 3*a**2
    t3 = [c1*e6[i] + c2*e5[i] + c3*e4[i] + c4*e3[i] for i in range(len(src))]
    return t3

def check_signal(t3):
    # Son 3 değere bak
    prev2 = t3[-3]
    prev1 = t3[-2]
    curr  = t3[-1]
    # BUY: T3 yükselmeye başladı (önceki düşüyordu)
    if curr > prev1 and prev1 <= prev2:
        return "BUY"
    # SELL: T3 düşmeye başladı (önceki yükseliyordu)
    if curr < prev1 and prev1 >= prev2:
        return "SELL"
    return None

def format_message(symbol, signal, price, t3_value):
    emoji = "🟢" if signal == "BUY" else "🔴"
    direction = "ALIŞ" if signal == "BUY" else "SATIŞ"
    now = datetime.utcnow().strftime("%Y-%m-%d %H:%M UTC")
    msg = (
        f"{emoji} <b>T3 {direction} SİNYALİ</b>\n"
        f"📌 Parite: <b>{symbol}</b>\n"
        f"💰 Fiyat: <b>{price}</b>\n"
        f"⏱ Zaman: 15m\n"
        f"📊 T3 Değeri: {round(t3_value, 6)}\n"
        f"🕐 {now}"
    )
    return msg

def main():
    print("T3 Scanner başlatıldı...")
    send_telegram("🚀 T3 Scanner başlatıldı!\nBinance margin pariteleri 15m taranıyor.")
    
    # Önceki sinyalleri takip et (aynı sinyali tekrar gönderme)
    last_signal = {}

    while True:
        try:
            symbols = get_margin_symbols()
            print(f"{len(symbols)} parite taranıyor...")
            
            for symbol in symbols:
                try:
                    result = get_klines(symbol, INTERVAL)
                    if result is None:
                        continue
                    closes, highs, lows = result
                    t3 = calc_t3(closes, highs, lows, T3_LENGTH, T3_FACTOR)
                    signal = check_signal(t3)
                    
                    if signal:
                        # Aynı sinyali tekrar gönderme
                        if last_signal.get(symbol) != signal:
                            price = closes[-1]
                            msg = format_message(symbol, signal, price, t3[-1])
                            send_telegram(msg)
                            last_signal[symbol] = signal
                            print(f"Sinyal: {symbol} {signal}")
                    
                    time.sleep(0.1)  # API rate limit için
                    
                except Exception as e:
                    print(f"Hata {symbol}: {e}")
                    continue

        except Exception as e:
            print(f"Genel hata: {e}")

        print(f"Tarama tamamlandı. {CHECK_EVERY} saniye bekleniyor...")
        time.sleep(CHECK_EVERY)

if __name__ == "__main__":
    main()
