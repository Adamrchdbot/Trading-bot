"""
Trading Bot - Stratégie Golden Cross (MA50 / MA200)
Broker  : Alpaca (paper trading gratuit)
Contrôle: Telegram Bot
"""

import time
import logging
import threading
from datetime import datetime, timedelta

# pip install alpaca-trade-api pandas numpy python-telegram-bot
import alpaca_trade_api as tradeapi
import pandas as pd
from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, ContextTypes

# ══════════════════════════════════════════
#  CONFIGURATION — à remplir
# ══════════════════════════════════════════
ALPACA_KEY    = "VOTRE_ALPACA_KEY"
ALPACA_SECRET = "VOTRE_ALPACA_SECRET"
BASE_URL      = "https://paper-api.alpaca.markets"

TELEGRAM_TOKEN   = "NOUVEAU_TOKEN_APRES_REVOCATION"  # ⚠️ Mets ton nouveau token ici
TELEGRAM_CHAT_ID = "7781677105"                       # ✅ Ton Chat ID

# Paramètres de la stratégie
SYMBOLS        = ["SPY", "QQQ", "AAPL"]
MA_COURT       = 50
MA_LONG        = 200
RISK_PAR_TRADE = 0.02       # 2 % du portefeuille par trade
CHECK_INTERVAL = 60 * 60    # Vérification toutes les heures

# ══════════════════════════════════════════
#  ÉTAT GLOBAL DU BOT
# ══════════════════════════════════════════
bot_running = True
tg_app      = None

# ══════════════════════════════════════════
#  LOGGING
# ══════════════════════════════════════════
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    handlers=[
        logging.FileHandler("trading_bot.log"),
        logging.StreamHandler()
    ]
)
log = logging.getLogger(__name__)


# ══════════════════════════════════════════
#  TELEGRAM — Envoi de messages
# ══════════════════════════════════════════
def tg_send(message: str):
    """Envoie un message Telegram depuis n'importe quel thread."""
    if tg_app is None:
        return
    import asyncio
    async def _send():
        await tg_app.bot.send_message(
            chat_id=TELEGRAM_CHAT_ID,
            text=message,
            parse_mode="Markdown"
        )
    asyncio.run_coroutine_threadsafe(_send(), tg_app.loop)


# ══════════════════════════════════════════
#  TELEGRAM — Commandes
# ══════════════════════════════════════════
async def cmd_start(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    msg = (
        "🤖 *GoldenBot actif !*\n\n"
        "Commandes disponibles :\n"
        "`/status`    — État du bot & portefeuille\n"
        "`/positions` — Positions ouvertes\n"
        "`/pause`     — Mettre en pause\n"
        "`/resume`    — Reprendre\n"
        "`/stop`      — Arrêter le bot\n"
    )
    await update.message.reply_text(msg, parse_mode="Markdown")


async def cmd_status(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    global bot_running
    try:
        api     = connect()
        account = api.get_account()
        equity  = float(account.equity)
        cash    = float(account.cash)
        pnl     = float(account.equity) - float(account.last_equity)
        pnl_pct = (pnl / float(account.last_equity)) * 100
        etat    = "✅ En cours" if bot_running else "⏸ En pause"
        arrow   = "▲" if pnl >= 0 else "▼"

        msg = (
            f"📊 *Statut du bot*\n\n"
            f"État         : {etat}\n"
            f"Portefeuille : `${equity:,.2f}`\n"
            f"Liquidités   : `${cash:,.2f}`\n"
            f"P&L du jour  : `{arrow} ${abs(pnl):,.2f} ({pnl_pct:+.2f}%)`\n\n"
            f"Stratégie    : MA{MA_COURT} × MA{MA_LONG}\n"
            f"Symbols      : {', '.join(SYMBOLS)}\n"
            f"Risque/trade : {RISK_PAR_TRADE*100:.0f}%"
        )
    except Exception as e:
        msg = f"❌ Erreur statut : {e}"
    await update.message.reply_text(msg, parse_mode="Markdown")


async def cmd_positions(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    try:
        api       = connect()
        positions = api.list_positions()
        if not positions:
            await update.message.reply_text("📭 Aucune position ouverte.")
            return

        lines = ["📂 *Positions ouvertes :*\n"]
        for p in positions:
            pnl     = float(p.unrealized_pl)
            pnl_pct = float(p.unrealized_plpc) * 100
            arrow   = "▲" if pnl >= 0 else "▼"
            lines.append(
                f"• *{p.symbol}* — {p.qty} action(s) @ `${float(p.avg_entry_price):.2f}`\n"
                f"  Actuel : `${float(p.current_price):.2f}` | "
                f"P&L : `{arrow} ${abs(pnl):.2f} ({pnl_pct:+.2f}%)`"
            )
        await update.message.reply_text("\n".join(lines), parse_mode="Markdown")
    except Exception as e:
        await update.message.reply_text(f"❌ Erreur : {e}")


async def cmd_pause(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    global bot_running
    bot_running = False
    log.info("⏸ Bot mis en pause via Telegram")
    await update.message.reply_text("⏸ *Bot mis en pause.* Aucun ordre ne sera passé.", parse_mode="Markdown")


async def cmd_resume(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    global bot_running
    bot_running = True
    log.info("▶️ Bot repris via Telegram")
    await update.message.reply_text("▶️ *Bot repris !* Surveillance active.", parse_mode="Markdown")


async def cmd_stop(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    global bot_running
    bot_running = False
    await update.message.reply_text("🛑 *Bot arrêté.* Relancez le script pour redémarrer.", parse_mode="Markdown")
    log.warning("🛑 Bot arrêté via Telegram")


# ══════════════════════════════════════════
#  ALPACA — Connexion
# ══════════════════════════════════════════
def connect():
    return tradeapi.REST(ALPACA_KEY, ALPACA_SECRET, BASE_URL, api_version="v2")


# ══════════════════════════════════════════
#  DONNÉES & INDICATEURS
# ══════════════════════════════════════════
def get_bars(api, symbol: str, days: int = 300) -> pd.DataFrame:
    end   = datetime.now()
    start = end - timedelta(days=days)
    return api.get_bars(
        symbol,
        tradeapi.rest.TimeFrame.Day,
        start.strftime("%Y-%m-%d"),
        end.strftime("%Y-%m-%d"),
        adjustment="raw"
    ).df


def compute_signals(df: pd.DataFrame) -> pd.DataFrame:
    df[f"MA{MA_COURT}"] = df["close"].rolling(MA_COURT).mean()
    df[f"MA{MA_LONG}"]  = df["close"].rolling(MA_LONG).mean()
    df["signal"]    = 0
    df.loc[df[f"MA{MA_COURT}"] > df[f"MA{MA_LONG}"], "signal"] =  1
    df.loc[df[f"MA{MA_COURT}"] < df[f"MA{MA_LONG}"], "signal"] = -1
    df["crossover"] = df["signal"].diff()
    return df


# ══════════════════════════════════════════
#  POSITIONS & ORDRES
# ══════════════════════════════════════════
def get_position(api, symbol: str):
    try:
        return api.get_position(symbol)
    except Exception:
        return None


def calc_qty(api, price: float) -> int:
    equity = float(api.get_account().equity)
    return max(int(equity * RISK_PAR_TRADE / price), 1)


def buy(api, symbol: str, price: float):
    qty = calc_qty(api, price)
    log.info(f"🟢 ACHAT  {symbol} — {qty} action(s) @ ~${price:.2f}")
    api.submit_order(symbol=symbol, qty=qty, side="buy", type="market", time_in_force="day")
    tg_send(
        f"🟢 *ACHAT exécuté*\n"
        f"Symbole  : `{symbol}`\n"
        f"Quantité : `{qty}` action(s)\n"
        f"Prix ≈   : `${price:.2f}`\n"
        f"⚡ _Golden Cross détecté_"
    )


def sell(api, symbol: str, qty: int, price: float):
    log.info(f"🔴 VENTE  {symbol} — {qty} action(s) @ ~${price:.2f}")
    api.submit_order(symbol=symbol, qty=qty, side="sell", type="market", time_in_force="day")
    tg_send(
        f"🔴 *VENTE exécutée*\n"
        f"Symbole  : `{symbol}`\n"
        f"Quantité : `{qty}` action(s)\n"
        f"Prix ≈   : `${price:.2f}`\n"
        f"⚡ _Death Cross détecté_"
    )


# ══════════════════════════════════════════
#  LOGIQUE PRINCIPALE
# ══════════════════════════════════════════
def run_strategy(api, symbol: str):
    log.info(f"📊 Analyse : {symbol}")
    try:
        df = get_bars(api, symbol)
        if len(df) < MA_LONG:
            log.warning(f"Pas assez de données pour {symbol}")
            return

        df       = compute_signals(df)
        last     = df.iloc[-1]
        price    = last["close"]
        cross    = last["crossover"]
        signal   = last["signal"]
        ma_court = last[f"MA{MA_COURT}"]
        ma_long  = last[f"MA{MA_LONG}"]

        log.info(
            f"  {symbol} | Prix: ${price:.2f} | "
            f"MA{MA_COURT}: ${ma_court:.2f} | MA{MA_LONG}: ${ma_long:.2f} | "
            f"Signal: {'HAUSSIER 📈' if signal == 1 else 'BAISSIER 📉'}"
        )

        position = get_position(api, symbol)
        if cross == 2 and position is None:
            buy(api, symbol, price)
        elif cross == -2 and position is not None:
            sell(api, symbol, int(position.qty), price)
        else:
            log.info("  → Pas de croisement. En attente.")

    except Exception as e:
        log.error(f"Erreur sur {symbol} : {e}")
        tg_send(f"⚠️ Erreur sur `{symbol}` : {e}")


def is_market_open(api) -> bool:
    return api.get_clock().is_open


# ══════════════════════════════════════════
#  BOUCLE DE TRADING (thread séparé)
# ══════════════════════════════════════════
def trading_loop():
    api = connect()
    tg_send("🤖 *GoldenBot démarré !*\nEnvoyez `/start` pour voir les commandes.")
    log.info("🤖 Trading Bot démarré — Stratégie Golden Cross")

    while True:
        if bot_running:
            if is_market_open(api):
                log.info("── Marché ouvert — Vérification des signaux ──")
                for symbol in SYMBOLS:
                    run_strategy(api, symbol)
                    time.sleep(1)
            else:
                log.info("⏳ Marché fermé — Prochain check dans 1h")
        else:
            log.info("⏸ Bot en pause — attente /resume")

        time.sleep(CHECK_INTERVAL)


# ══════════════════════════════════════════
#  POINT D'ENTRÉE
# ══════════════════════════════════════════
def main():
    global tg_app

    # Lancer le trading dans un thread séparé
    t = threading.Thread(target=trading_loop, daemon=True)
    t.start()

    # Bot Telegram (bloquant dans le thread principal)
    app = ApplicationBuilder().token(TELEGRAM_TOKEN).build()
    tg_app = app

    app.add_handler(CommandHandler("start",     cmd_start))
    app.add_handler(CommandHandler("status",    cmd_status))
    app.add_handler(CommandHandler("positions", cmd_positions))
    app.add_handler(CommandHandler("pause",     cmd_pause))
    app.add_handler(CommandHandler("resume",    cmd_resume))
    app.add_handler(CommandHandler("stop",      cmd_stop))

    log.info("📱 Bot Telegram prêt — En écoute des commandes...")
    app.run_polling()


if __name__ == "__main__":
    main()