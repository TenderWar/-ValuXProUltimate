# -ValuXProUltimate
import os
import asyncio
import requests
from datetime import datetime
from aiogram import Bot, Dispatcher, types
from aiogram.types import InlineKeyboardMarkup, InlineKeyboardButton
from aiogram.filters import Command
import matplotlib.pyplot as plt

TOKEN ="8347342137:AAHG6zS6YCGX0HWMDjt5T53GkSWRbiaKYuI"
bot = Bot(token=TOKEN)
dp = Dispatcher()

user_history = {}
user_favorites = {}

def get_rates(base="USD"):
    url = f"https://api.exchangerate-api.com/v4/latest/{base}"
    return requests.get(url).json()["rates"]

def convert(amount, from_cur, to_cur):
    rates = get_rates(from_cur)
    return amount * rates[to_cur]

def get_crypto(coin="bitcoin", currency="usd"):
    url = f"https://api.coingecko.com/api/v3/simple/price?ids={coin}&vs_currencies={currency}&include_24hr_change=true"
    data = requests.get(url).json()
    price = data[coin][currency]
    change = data[coin][f"{currency}_24h_change"]
    return price, change

def plot_crypto_history(prices, coin="BTC"):
    plt.figure(figsize=(4,3))
    plt.plot(prices, marker='o', linestyle='-', color='gold' if coin=="BTC" else "cyan")
    plt.title(f"{coin} last {len(prices)} hours")
    plt.xlabel("Hours")
    plt.ylabel("Price USD")
    file_path = f"{coin}_chart.png"
    plt.savefig(file_path)
    plt.close()
    return file_path

def main_menu():
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton("💱 Конвертація", callback_data="convert"),
         InlineKeyboardButton("📊 Курси USD", callback_data="rates")],
        [InlineKeyboardButton("🪙 Криптовалюти", callback_data="crypto"),
         InlineKeyboardButton("⭐ Улюблені", callback_data="fav")],
        [InlineKeyboardButton("📜 Історія", callback_data="history"),
         InlineKeyboardButton("📈 Графік крипти", callback_data="chart")],
        [InlineKeyboardButton("ℹ️ Допомога", callback_data="help")]
    ])

@dp.message(Command("start"))
async def start(message: types.Message):
    await message.answer(
        "💎 <b>ValuX Pro Ultimate – FULL CHAOS</b>\n\n"
        "🚀 Професійний конвертер валют та крипти\n"
        "Обери функцію 👇",
        parse_mode="HTML",
        reply_markup=main_menu()
    )

@dp.callback_query()
async def callbacks(callback: types.CallbackQuery):
    user_id = callback.from_user.id

    if callback.data == "help":
        await callback.message.edit_text(
            "📌 <b>Як користуватись:</b>\n\n"
            "1️⃣ Введи суму у форматі:\n<code>100 USD UAH</code>\n"
            "2️⃣ Або скористайся кнопками нижче 👇",
            parse_mode="HTML",
            reply_markup=main_menu()
        )

    elif callback.data == "rates":
        rates = get_rates("USD")
        text = (
            "📊 <b>Курси USD:</b>\n\n"
            f"🇺🇦 UAH: {rates['UAH']}\n"
            f"🇪🇺 EUR: {rates['EUR']}\n"
            f"🇵🇱 PLN: {rates['PLN']}\n"
            f"🇬🇧 GBP: {rates['GBP']}"
        )
        await callback.message.edit_text(text, parse_mode="HTML", reply_markup=main_menu())

    elif callback.data == "crypto":
        btc_price, btc_change = get_crypto("bitcoin")
        eth_price, eth_change = get_crypto("ethereum")
        text = (
            "🪙 <b>Криптовалюти:</b>\n\n"
            f"₿ BTC: ${btc_price:,.2f} {'📈' if btc_change>=0 else '📉'} {btc_change:.2f}%\n"
            f"Ξ ETH: ${eth_price:,.2f} {'📈' if eth_change>=0 else '📉'} {eth_change:.2f}%"
        )
        await callback.message.edit_text(text, parse_mode="HTML", reply_markup=main_menu())

    elif callback.data == "chart":
        prices = [get_crypto("bitcoin")[0] for _ in range(10)]
        file_path = plot_crypto_history(prices, "BTC")
        await callback.message.answer_photo(open(file_path, "rb"))
        await callback.message.answer("📈 Ось графік BTC", reply_markup=main_menu())

    elif callback.data == "history":
        history = user_history.get(user_id, [])
        if not history:
            text = "📜 Історія порожня."
        else:
            text = "📜 <b>Останні 5 конвертацій:</b>\n\n" + "\n".join(history[-5:])
        await callback.message.edit_text(text, parse_mode="HTML", reply_markup=main_menu())

    elif callback.data == "fav":
        fav = user_favorites.get(user_id)
        if fav:
            text = f"⭐ <b>Улюблена пара:</b> {fav}"
        else:
            text = "⭐ Немає обраних валют."
        await callback.message.edit_text(text, reply_markup=main_menu())

    elif callback.data == "convert":
        await callback.message.edit_text(
            "💱 Введи суму та валюти:\n<code>100 USD EUR</code>",
            parse_mode="HTML",
            reply_markup=main_menu()
        )

@dp.message()
async def handle_conversion(message: types.Message):
    user_id = message.from_user.id
    try:
        amount, from_cur, to_cur = message.text.split()
        amount = float(amount)
        result = convert(amount, from_cur.upper(), to_cur.upper())

        text = (
            f"💱 <b>{amount} {from_cur.upper()}</b> ➡️ <b>{result:.2f} {to_cur.upper()}</b>\n"
            f"🕒 {datetime.now().strftime('%H:%M:%S %d-%m-%Y')}"
        )

        user_history.setdefault(user_id, []).append(text)
        user_favorites[user_id] = f"{from_cur.upper()} → {to_cur.upper()}"

        await message.answer(text, parse_mode="HTML")
    except:
        await message.answer("⚠️ Формат: 100 USD UAH або 0.01 BTC USD")

async def main():
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
