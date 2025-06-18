import requests
import time
import uuid
import json
import os
import schedule
import smtplib
from flask import Flask, redirect
from datetime import datetime, timedelta
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText

# -------- CONFIG --------
EMAIL_SENDER = "coinsniper7@gmail.com"
EMAIL_PASSWORD = "fahfdhnzuvrprqvq"
EMAIL_RECEIVER = "coinsniper7@gmail.com"
SMTP_SERVER = "smtp.gmail.com"
SMTP_PORT = 587
SCAN_INTERVAL = 60
MONITOR_INTERVAL = 300


SEEN_FILE = "seen_coins.json"
COIN_EXPIRATION_DAYS = 180
TICKER_URL = "https://api.bybit.com/v5/market/tickers?category=spot"
COIN_LIST_URL = "https://api.bybit.com/v5/market/coin/list"
BASE_URL = os.getenv("PUBLIC_HOST", "http://localhost:5000")
# -------- UTILS --------
def load_seen_coins():
    if not os.path.exists(SEEN_FILE):
        return {}
    try:
        with open(SEEN_FILE, "r") as f:
            return json.load(f)
    except (json.JSONDecodeError, IOError):
        return {}

def save_seen_coins(data):
    with open(SEEN_FILE, "w") as f:
        json.dump(data, f, indent=2)

def cleanup_old_coins(data):
    now = datetime.utcnow()
    return {
        k: v for k, v in data.items()
        if (now - datetime.strptime(v["listed_at"], "%Y-%m-%dT%H:%M:%SZ")) < timedelta(days=COIN_EXPIRATION_DAYS)
    }

def format_star_rating(score):
    return "★" * score + "☆" * (5 - score)

def get_price_data():
    try:
        res = requests.get(TICKER_URL)
        data = res.json().get("result", {}).get("list", [])
        return {entry["symbol"]: entry for entry in data}
    except:
        return {}

# -------- EMAIL --------
def send_coin_email(coin_data, update=False):
    subject = f"🔁 Update: {coin_data['name']}" if update else f"🚀 Νέο Meme Coin: {coin_data['name']}"
    msg = MIMEMultipart("alternative")
    msg["Subject"] = subject
    msg["From"] = EMAIL_SENDER
    msg["To"] = EMAIL_RECEIVER

    html = f"""
    <html>
        <body>
            <h2>{coin_data['name']}</h2>
            <p><b>Τιμή εκκίνησης:</b> {coin_data['start_price']}<br>
            <b>Τρέχουσα τιμή:</b> {coin_data['current_price']}<br>
            <b>Μεταβολή χρόνου:</b> {coin_data['time_diff']} sec<br>
            <b>Spikes:</b> {', '.join(map(str, coin_data['spikes']))}<br>
            <b>Αξιολόγηση:</b> {format_star_rating(coin_data['rating'])}</p>
            {"<p><a href='"+BASE_URL+"/accept/"+coin_data['id']+"'><button>✅ Αποδοχή</button></a> <a href='"+BASE_URL+"/reject/"+coin_data['id']+"'><button>❌ Απόρριψη</button></a></p>" if not update else ""}
        </body>
    </html>
    """
    msg.attach(MIMEText(html, "html"))
    try:
        with smtplib.SMTP(SMTP_SERVER, SMTP_PORT) as server:
            server.starttls()
            server.login(EMAIL_SENDER, EMAIL_PASSWORD)
            server.send_message(msg)
    except Exception as e:
        print(f"Email error: {e}")

# -------- COIN SCAN --------


def get_new_listings():
    try:
        res = requests.get(COIN_LIST_URL)
        return res.json().get("result", {}).get("rows", [])
    except:
        return []
def analyze_coin(coin, price_data):
    now = datetime.utcnow()
    symbol = coin["name"] + "USDT"
    ticker = price_data.get(symbol)
    if not ticker:
        return None
    try:
        listed_time = datetime.strptime(coin["launchTime"], "%Y-%m-%dT%H:%M:%SZ")
        time_diff = (now - listed_time).total_seconds()
        if 50 < time_diff < 20 * 60:
            price = float(ticker["lastPrice"])
            volume = float(ticker.get("turnover24h", 0))
            return {
                "id": str(uuid.uuid4()),
                "name": coin["name"],
                "listed_at": coin["launchTime"],
                "status": "none",
                "start_price": round(price, 6),
                "current_price": round(price, 6),
                "time_diff": int(time_diff),
                "spikes": [round(price * (1 + i / 10), 6) for i in range(1, 5)],
                "rating": min(5, max(1, int((volume / 50000) + 1)))
            }
    except:
        return None
    return None

def scan_coins():
    seen = cleanup_old_coins(load_seen_coins())
    listings = get_new_listings()
    price_data = get_price_data()
    for coin in listings:
        if coin["name"] in seen:
            continue
        analyzed = analyze_coin(coin, price_data)
        if analyzed:
            seen[analyzed["name"]] = analyzed
            send_coin_email(analyzed)
    save_seen_coins(seen)

# -------- MONITOR ACCEPTED --------
def monitor_accepted_coins():
    seen = load_seen_coins()
    price_data = get_price_data()
    changed = False
    for name, coin in seen.items():
        if coin["status"] != "accepted":
            continue
        symbol = coin["name"] + "USDT"
        ticker = price_data.get(symbol)
        if not ticker:
            continue
        new_price = float(ticker["lastPrice"])
        delta = abs(new_price - coin["current_price"]) / coin["current_price"]
        if delta > 0.015:


            coin["current_price"] = round(new_price, 6)
            send_coin_email(coin, update=True)
            changed = True
    if changed:
        save_seen_coins(seen)
# -------- FLASK --------
app = Flask(__name__)

@app.route('/')
def index():
    return "Sniper Bot is running!"

@app.route('/accept/<coin_id>')
def accept_coin(coin_id):
    seen = load_seen_coins()
    for coin in seen.values():
        if coin["id"] == coin_id:
            coin["status"] = "accepted"
            save_seen_coins(seen)
            return f"✅ Coin accepted: {coin['name']}"
    return "Not found"

@app.route('/reject/<coin_id>')
def reject_coin(coin_id):
    seen = load_seen_coins()
    for coin in seen.values():
        if coin["id"] == coin_id:
            coin["status"] = "rejected"
            save_seen_coins(seen)
            return f"❌ Coin rejected: {coin['name']}"
    return "Not found"

# -------- MAIN --------
def run_scheduler():
    schedule.every(SCAN_INTERVAL).seconds.do(scan_coins)
    schedule.every(MONITOR_INTERVAL).seconds.do(monitor_accepted_coins)
    while True:
        schedule.run_pending()
        time.sleep(1)


if __name__ == "__main__":
    import threading
    threading.Thread(target=run_scheduler).start()
    app.run(host="0.0.0.0", port=5000)

