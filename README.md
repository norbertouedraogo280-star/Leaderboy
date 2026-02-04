import requests
import time
import hmac
import hashlib
import os
import sys
from datetime import datetime

# ================== CONFIGURATION PARAMÈTRES ==================
API_KEY = 'YQL8N4sxGb6YF3RmfhaQIv2MMNuoB3AcQqf7x1YaVzARKoGb1TKjumwUVNZDW3af'
API_SECRET = 'si08ii320XMByW4VY1VRt5zRJNnB3QrYBJc3QkDOdKHLZGKxyTo5CHxz7nd4CuQ0'

URL = "https://fapi.binance.com"
RISQUE_USDT = 0.50          # Risque par trade
SEUIL_TRAILLING_PNL = 0.40   # Activation SLC à +0.40$ de profit
STRAT_IMPULSE = 0.0018       # Setup Stratégie 0.18%
CALLBACK_TRAIL = 0.15        # Écart du suiveur
MARKETS = ['BTCUSDT', 'ETHUSDT', 'SOLUSDT', 'BNBUSDT']

def api_call(method, endpoint, params={}):
    params['timestamp'] = int(time.time() * 1000)
    query = '&'.join([f"{k}={v}" for k, v in params.items()])
    signature = hmac.new(API_SECRET.encode(), query.encode(), hashlib.sha256).hexdigest()
    full_url = f"{URL}{endpoint}?{query}&signature={signature}"
    headers = {'X-MBX-APIKEY': API_KEY}
    try:
        r = requests.request(method, full_url, headers=headers, timeout=3)
        return r.json()
    except: return None

def get_precisions(symbol):
    info = requests.get(f"{URL}/fapi/v1/exchangeInfo").json()
    for s in info['symbols']:
        if s['symbol'] == symbol:
            return s['quantityPrecision'], s['pricePrecision']
    return 2, 2

def run_kilian_ultimate():
    os.system('cls' if os.name == 'nt' else 'clear')
    print("🚀 SYSTÈME KILIAN ARMÉ : SCAN + AUTO SL/TP ACTIVÉ")
    
    while True:
        try:
            pos_data = api_call('GET', '/fapi/v2/positionRisk')
            acc_data = api_call('GET', '/fapi/v2/account')
            if not pos_data or not acc_data: continue

            active = [p for p in pos_data if float(p['positionAmt']) != 0]
            
            # Rafraîchissement propre
            sys.stdout.write("\033[H") 
            print("=============================================")
            print(f"🕒 {datetime.now().strftime('%H:%M:%S')} | SOLDE : {float(acc_data['totalWalletBalance']):.2f} USDT")
            print("=============================================")

            if active:
                # ---------------------------------------------------
                # [MODE GESTION] - COMMANDE LE TRAILING À +0.40$
                # ---------------------------------------------------
                p = active[0]
                sym, qty, entry = p['symbol'], float(p['positionAmt']), float(p['entryPrice'])
                pnl, curr_p = float(p['unRealizedProfit']), float(p['markPrice'])
                
                orders = api_call('GET', '/fapi/v1/openOrders', {'symbol': sym})
                sl_p = next((float(o.get('stopPrice', 0)) for o in orders if 'STOP' in o['type']), 0.0)
                tp_p = next((float(o.get('price', 0)) for o in orders if o['type'] == 'LIMIT'), 0.0)
                is_trail = any(o['type'] == 'TRAILING_STOP_MARKET' for o in orders)

                print(f"📈 GESTION ACTIVE : {sym}")
                print(f"💰 PNL : {pnl:+.2f} USDT")
                print(f"🛡️  SL : {sl_p:.2f} | 🎯 TP : {tp_p:.2f}")
                print(f"🏁 PRIX ACTUEL : {curr_p:.2f}")
                
                if pnl >= SEUIL_TRAILLING_PNL and not is_trail:
                    print("\n⚡ PNL > 0.40$ : PASSAGE EN TRAILING STOP...")
                    api_call('DELETE', '/fapi/v1/allOpenOrders', {'symbol': sym})
                    api_call('POST', '/fapi/v1/order', {
                        'symbol': sym, 'side': 'SELL' if qty > 0 else 'BUY',
                        'type': 'TRAILING_STOP_MARKET', 'quantity': abs(qty),
                        'callbackRate': CALLBACK_TRAIL, 'reduceOnly': 'true'
                    })

            else:
                # ---------------------------------------------------
                # [MODE SCAN LIVE] - ENTRÉE + SL + TP AUTOMATIQUE
                # ---------------------------------------------------
                print("🔎 SCAN LIVE (STRATÉGIE 0.18%) :")
                for sym in MARKETS:
                    k = requests.get(f"{URL}/fapi/v1/klines?symbol={sym}&interval=1m&limit=1").json()
                    cp, op = float(k[0][4]), float(k[0][1])
                    change = (cp - op) / op
                    print(f"  > {sym:.<10} : {cp:>10.2f} ({change:+.4f}%)")

                    if abs(change) >= STRAT_IMPULSE:
                        side = "BUY" if change > 0 else "SELL"
                        q_p, p_p = get_precisions(sym)
                        
                        # Calcul lot pour risque 0.50$ (Distance SL environ 0.6%)
                        lot = round(RISQUE_USDT / (cp * 0.006), q_p)
                        
                        if lot > 0:
                            print(f"\n🚀 SIGNAL DÉTECTÉ ! EXÉCUTION COMBO {side} {sym}")
                            # 1. Position Marché
                            api_call('POST', '/fapi/v1/leverage', {'symbol': sym, 'leverage': 20})
                            api_call('POST', '/fapi/v1/order', {'symbol': sym, 'side': side, 'type': 'MARKET', 'quantity': lot})
                            
                            # 2. Placement AUTOMATIQUE du SL et TP
                            sl_price = round(cp * 0.994 if side == "BUY" else cp * 1.006, p_p)
                            tp_price = round(cp * 1.018 if side == "BUY" else cp * 0.982, p_p)
                            
                            api_call('POST', '/fapi/v1/order', {'symbol': sym, 'side': 'SELL' if side == "BUY" else 'BUY', 'type': 'STOP_MARKET', 'stopPrice': sl_price, 'quantity': lot, 'reduceOnly': 'true'})
                            api_call('POST', '/fapi/v1/order', {'symbol': sym, 'side': 'SELL' if side == "BUY" else 'BUY', 'type': 'LIMIT', 'price': tp_price, 'quantity': lot, 'timeInForce': 'GTC', 'reduceOnly': 'true'})
                            
                            print(f"✅ ORDRES PLACÉS : SL @ {sl_price} | TP @ {tp_price}")
                            time.sleep(2)
                            break

            print("=============================================")
            time.sleep(1.5)

        except Exception: time.sleep(2)

if __name__ == "__main__":
    run_kilian_ultimate()
