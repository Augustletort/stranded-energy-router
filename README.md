# stranded-energy-router
Real-time router that turns stranded energy into profit.  Every 5 min it decides: mine Bitcoin (BTC = Money) or run AI inference (AI = Intelligence) — whichever pays more per kWh.  For wind curtailment, flare gas, negative prices &amp; future orbital solar.  Every wasted electron picks the winner.
import requests, time
from datetime import datetime

# === CONFIG ===
TELEGRAM_TOKEN = "YOUR_BOT_TOKEN"      # Get free at @BotFather
TELEGRAM_CHAT  = "@strandedenergy"     # Your public channel
# ==============

SOURCES = {
    "DK2": "https://dataportal.nordpoolgroup.com/api/DayAheadPrices?currency=EUR&market=DayAhead&deliveryArea=DK2",
    "ERCOT": "https://api.ercot.com/v1/curtailment",  # placeholder
}

ORACLES = {
    "BTC":  "https://api.nicehash.com/api/v2/mining/algorithms",
    "TAO":  "https://api.taostats.io/api/v1/subnet-rewards",
    "COCO": "https://api.cocoon.sh/v1/spot-price"
}

def send_telegram(msg):
    if TELEGRAM_TOKEN != "YOUR_BOT_TOKEN":
        requests.post(f"https://api.telegram.org/bot{TELEGRAM_TOKEN}/sendMessage",
                      data={"chat_id": TELEGRAM_CHAT, "text": msg, "parse_mode": "HTML"})

def get_dk_price():
    try:
        data = requests.get(SOURCES["DK2"], params={'date': datetime.now().strftime('%Y-%m-%d')}).json()
        hour = datetime.now().hour
        price = data['data']['Rows'][hour]['Columns'][0]['Value'].replace(',', '.')
        return float(price) / 1000  # €/kWh
    except:
        return 0.12

def get_profits():
    # BTC profitability (simplified)
    btc_usd_kwh = 0.09

    # AI inference (average TAO + Cocoon)
    try:
        tao = requests.get(ORACLES["TAO"]).json()
        cocoon = requests.get(ORACLES["COCO"]).json()
        ai_usd_kwh = (float(tao.get('avg_usd_per_wh', 2.2)) * 1000 +
                      float(cocoon.get('spot_usd_per_wh', 2.5)) * 1000) / 2
    except:
        ai_usd_kwh = 2.25

    price = get_dk_price()
    multiplier = 10 if price < 0 else 3 if price < 0.02 else 1
    return btc_usd_kwh * multiplier, ai_usd_kwh * multiplier

while True:
    btc, ai = get_profits()
    winner = "AI INFERENCE" if ai > btc * 1.1 else "BITCOIN MINING"
    profit = max(ai, btc)
    price = get_dk_price()

    msg = f"""GLOBAL STRANDED ENERGY SIGNAL
{winner}
${profit:.2f}/kWh
DK2 spot: €{price:.4f}/kWh
{datetime.utcnow().strftime('%Y-%m-%d %H:%M')} UTC

BTC = Money
AI = Intelligence
"""
    print(msg)
    send_telegram(msg)
    time.sleep(300)  # 5 min
