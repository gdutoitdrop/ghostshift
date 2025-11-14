from flask import Flask, request, jsonify
import telegram
import plaid
import stripe
import os
import json

app = Flask(__name__)

# === CONFIG ===
TELEGRAM_TOKEN = os.environ['TELEGRAM_TOKEN']
PLAID_CLIENT_ID = os.environ['PLAID_CLIENT_ID']
PLAID_SECRET = os.environ['PLAID_SECRET']
STRIPE_KEY = os.environ['STRIPE_KEY']

stripe.api_key = STRIPE_KEY
bot = telegram.Bot(token=TELEGRAM_TOKEN)

configuration = plaid.Configuration(
    host=plaid.Environment.Sandbox,
    api_key={'clientId': PLAID_CLIENT_ID, 'secret': PLAID_SECRET}
)
api_client = plaid.ApiClient(configuration)
plaid_client = plaid.PlaidApi(api_client)

# Simple in-memory DB
users = {}

@app.route('/')
def home():
    return "GhostShift LIVE (Render.com)"

@app.route('/webhook', methods=['POST'])
def webhook():
    data = request.json
    chat_id = data['message']['chat']['id']
    
    if 'start' in data['message'].get('text', '').lower():
        bot.send_message(chat_id, 
            "GhostShift ON\n"
            "1. /link → Connect bank\n"
            "2. Auto-split: 25% tax | 10% save | 65% yours\n"
            "Free beta!")
    
    elif 'link' in data['message'].get('text', '').lower():
        response = plaid_client.link_token_create({
            'user': {'client_user_id': str(chat_id)},
            'client_name': 'GhostShift',
            'products': ['transactions'],
            'country_codes': ['US'],
            'language': 'en'
        })
        link_token = response['link_token']
        users[str(chat_id)] = {'link_token': link_token}
        
        bot.send_message(chat_id, 
            f"Connect bank:\nhttps://cdn.plaid.com/link/v2/stable/link.html?key=PLAID_PUBLIC_KEY&link_token={link_token}&product=transactions")
    
    return jsonify(success=True)

# Plaid webhook
@app.route('/plaid', methods=['POST'])
def plaid_webhook():
    data = request.json
    if data.get('webhook_code') == 'DEFAULT_UPDATE':
        # Simulate split
        amount = 100.00
        tax = amount * 0.25
        save = amount * 0.10
        spend = amount * 0.65
        
        bot.send_message(123456789,  # ← CHANGE TO YOUR CHAT ID
            f"*New Deposit: ${amount:.2f}*\n\n"
            f"Tax → `${tax:.2f}`\n"
            f"Save → `${save:.2f}`\n"
            f"Yours → `${spend:.2f}`\n\n"
            f"_Auto-split in 0.2s._", parse_mode='Markdown')
    
    return jsonify(success=True)

# Set Telegram webhook on start
import threading
def set_webhook():
    url = f"https://{os.environ['RENDER_EXTERNAL_HOSTNAME']}.onrender.com/webhook"
    bot.set_webhook(url=url)

threading.Thread(target=set_webhook).start()

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=8080)# ghostshift
