# *Script M12H - Estratégia EMA5/21 com entrada em tempo real, uma operação por vela e STOP corrigido*

import requests
import time
import math
import logging
from binance.client import Client
from binance.exceptions import BinanceAPIException

# Configuração de logs
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('projeto_ouro_m12h.log'),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger('ProjetoOuro-M12H')

# Configuração do bot
M12H_CONFIG = {
    "TELEGRAM_TOKEN": "7786844337:AAFzs-iJsroeYlun6oNIekfSxFCnt93FDac",
    "TELEGRAM_CHAT_ID": "7684227726",
    "BINANCE_API_KEY": "1dLjdnMwNj6MygL3hygEfNI9YlTmcuiXQHQcr4mQT2Rd9CFgPrMAkjg7paShO53t",
    "BINANCE_API_SECRET": "3IpxLExa9Ue4MJ5DIFuxnM2o6cYXfL3bwTkKSx6NzQ3mXjUROduoZ8z7zRiDFctA",
    "SYMBOL": "1000PEPEUSDT",
    "LEVERAGE": 20,
    "TIMEFRAME": Client.KLINE_INTERVAL_1HOUR,
    "MIN_NOTIONAL": 5.0,
    "STRATEGY_NAME": "M12H-EMA5-21",
    "MARGEM_SEGURANCA": 1.10
}

def calculate_ema(prices, period):
    if len(prices) < period:
        return None
    multiplier = 2 / (period + 1)
    ema = sum(prices[:period]) / period
    for price in prices[period:]:
        ema = (price - ema) * multiplier + ema
    return ema

def adjust_quantity(qty, step_size=1000):
    return math.floor(qty / step_size) * step_size

def m12h_send_telegram(msg):
    try:
        requests.post(
            f"https://api.telegram.org/bot{M12H_CONFIG['TELEGRAM_TOKEN']}/sendMessage",
            json={"chat_id": M12H_CONFIG["TELEGRAM_CHAT_ID"], "text": f"🚨 [M12H-PEPE] {msg}"},
            timeout=5
        )
    except Exception as e:
        logger.error(f"[M12H] Erro Telegram: {e}")

def m12h_get_available_balance():
    try:
        account = M12H_CLIENT.futures_account_balance()
        balance = next((float(x['balance']) for x in account if x['asset'] == 'USDT'), 0.0)
        return balance
    except Exception as e:
        logger.error(f"[M12H] Erro ao obter saldo: {e}")
        return 0.0

def m12h_calculate_quantity(price):
    balance = m12h_get_available_balance()
    if balance <= 0:
        return 0
    margin_per_unit = (price / M12H_CONFIG["LEVERAGE"]) * M12H_CONFIG["MARGEM_SEGURANCA"]
    max_qty = balance / margin_per_unit
    qty = adjust_quantity(max_qty)
    notional = qty * price
    if notional < M12H_CONFIG["MIN_NOTIONAL"]:
        m12h_send_telegram(f"⚠️ Valor notional abaixo do mínimo: {notional:.2f} USDT")
        return 0
    return qty

def m12h_open_position(side, price):
    qty = m12h_calculate_quantity(price)
    if not qty:
        m12h_send_telegram("⚠️ Quantidade insuficiente para abrir posição.")
        return False
    try:
        M12H_CLIENT.futures_create_order(
            symbol=M12H_CONFIG["SYMBOL"],
            side=side,
            type="MARKET",
            quantity=qty
        )

        klines = M12H_CLIENT.futures_klines(
            symbol=M12H_CONFIG["SYMBOL"],
            interval=M12H_CONFIG["TIMEFRAME"],
            limit=51
        )
        closes = [float(k[4]) for k in klines]
        ema_21 = calculate_ema(closes, 21)
        if ema_21:
            stop_price = ema_21 * (1 - 0.0015) if side == "BUY" else ema_21 * (1 + 0.0015)
            stop_side = "SELL" if side == "BUY" else "BUY"
            try:
                M12H_CLIENT.futures_create_order(
                    symbol=M12H_CONFIG["SYMBOL"],
                    side=stop_side,
                    type="STOP_MARKET",
                    stopPrice=round(stop_price, 5),
                    closePosition=True,
                    timeInForce="GTC"
                )
                m12h_send_telegram(f"📉 Stop Loss {'abaixo' if side == 'BUY' else 'acima'} da EMA21 ({ema_21:.8f}) → {stop_price:.5f}")
            except BinanceAPIException as e:
                m12h_send_telegram(f"❌ ERRO AO CRIAR STOP: {e.message}")

        m12h_send_telegram(f"✅ {side} | {qty:.0f} PEPE | ${price:.8f}")
        return True
    except BinanceAPIException as e:
        m12h_send_telegram(f"❌ ERRO: {e.message}")
        return False

def m12h_close_position(side, price):
    try:
        positions = M12H_CLIENT.futures_position_information(symbol=M12H_CONFIG["SYMBOL"])
        position = next((p for p in positions if float(p['positionAmt']) != 0), None)
        if position:
            qty = abs(float(position['positionAmt']))
            M12H_CLIENT.futures_create_order(
                symbol=M12H_CONFIG["SYMBOL"],
                side=side,
                type="MARKET",
                quantity=qty,
                reduceOnly=True
            )
            m12h_send_telegram(f"🔒 Fechou {side} | {qty:.0f} PEPE | ${price:.8f}")
            return True
        return False
    except Exception as e:
        m12h_send_telegram(f"💥 Erro ao fechar: {str(e)}")
        return False

def m12h_run_strategy():
    logger.info("[M12H] 🤖 Iniciando estratégia M12H (1h EMA5/21 - Tempo Real)")
    m12h_send_telegram("⚡ Estratégia M12H ATIVADA")
    current_position = "none"
    previous_ema_5, previous_ema_21 = None, None
    last_kline_time = None
    operou_na_vela = False

    while True:
        try:
            positions = M12H_CLIENT.futures_position_information(symbol=M12H_CONFIG["SYMBOL"])
            position = next((p for p in positions if float(p['positionAmt']) != 0), None)
            if position:
                pos_amt = float(position['positionAmt'])
                side_to_close = "BUY" if pos_amt < 0 else "SELL"
                price = float(M12H_CLIENT.futures_mark_price(symbol=M12H_CONFIG["SYMBOL"])['markPrice'])
                m12h_send_telegram(f"⚠️ Posição pré-existente detectada: {pos_amt} PEPE. Fechando antes de operar.")
                try:
                    M12H_CLIENT.futures_create_order(
                        symbol=M12H_CONFIG["SYMBOL"],
                        side=side_to_close,
                        type="MARKET",
                        quantity=abs(pos_amt),
                        reduceOnly=True
                    )
                    m12h_send_telegram(f"🔒 Posição fechada automaticamente: {side_to_close} {abs(pos_amt):.0f} PEPE.")
                except Exception as e:
                    m12h_send_telegram(f"❌ Erro ao fechar posição pré-existente: {str(e)}")
                time.sleep(3)
                current_position = "none"

            klines = M12H_CLIENT.futures_klines(
                symbol=M12H_CONFIG["SYMBOL"],
                interval=M12H_CONFIG["TIMEFRAME"],
                limit=51
            )
            closes = [float(k[4]) for k in klines]
            ema_5 = calculate_ema(closes, 5)
            ema_21 = calculate_ema(closes, 21)
            price = float(M12H_CLIENT.futures_mark_price(symbol=M12H_CONFIG["SYMBOL"])['markPrice'])
            last_kline_close_time = klines[-1][6]

            logger.info(f"[M12H] EMA5: {ema_5:.8f} | EMA21: {ema_21:.8f} | Preço (tempo real): {price:.8f}")

            if last_kline_time != last_kline_close_time:
                last_kline_time = last_kline_close_time
                operou_na_vela = False

            if not operou_na_vela and previous_ema_5 is not None and previous_ema_21 is not None:
                cruzou_para_cima = previous_ema_5 <= previous_ema_21 and ema_5 > ema_21
                cruzou_para_baixo = previous_ema_5 >= previous_ema_21 and ema_5 < ema_21

                if cruzou_para_cima:
                    if current_position == "short":
                        if m12h_close_position("BUY", price):
                            current_position = "none"
                            time.sleep(1)
                    if current_position == "none":
                        if m12h_open_position("BUY", price):
                            current_position = "long"
                            operou_na_vela = True

                elif cruzou_para_baixo:
                    if current_position == "long":
                        if m12h_close_position("SELL", price):
                            current_position = "none"
                            time.sleep(1)
                    if current_position == "none":
                        if m12h_open_position("SELL", price):
                            current_position = "short"
                            operou_na_vela = True

            previous_ema_5, previous_ema_21 = ema_5, ema_21
            time.sleep(60)

        except KeyboardInterrupt:
            logger.info("[M12H] Estratégia interrompida manualmente")
            break
        except Exception as e:
            logger.error(f"[M12H] Erro: {e}")
            time.sleep(30)

if __name__ == "__main__":
    try:
        M12H_CLIENT = Client(M12H_CONFIG["BINANCE_API_KEY"], M12H_CONFIG["BINANCE_API_SECRET"])
        M12H_CLIENT.futures_change_leverage(
            symbol=M12H_CONFIG["SYMBOL"],
            leverage=M12H_CONFIG["LEVERAGE"]
        )
        m12h_run_strategy()
    except Exception as e:
        logger.critical(f"[M12H] ERRO INICIAL: {e}")
        m12h_send_telegram(f"☠️ FALHA CRÍTICA: {str(e)}")
