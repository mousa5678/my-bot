# my-bot
import asyncio
import aiohttp
import time
from datetime import datetime
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    ApplicationBuilder, CommandHandler, CallbackContext,
    CallbackQueryHandler, ContextTypes
)

TELEGRAM_BOT_TOKEN = "7958425763:AAFUTllfiVNdBMMqPzZTxWi7hZQbYEBM6ZA"
DEFAULT_MIN_ARBITRAGE_PERCENT = 0.7
MAX_SCAN_COINS = 300

coins_cache = None
coins_cache_time = 0
CACHE_SECONDS = 1200  # 20 min

LOCK_TIMES = {
    "refresh": 5,
    "scan": 30,
    "stopauto": 5,
    # currency_* = 3 (لكل عملة عند الضغط عليها)
}

def is_price_valid(price):
    return price is not None and price > 0.01 and price < 1000000

def is_arbitrage_valid(buy, sell):
    if not is_price_valid(buy) or not is_price_valid(sell):
        return False
    percent = (sell - buy) / buy * 100 if buy else 0
    if percent > 500 or percent < 0 or sell < 0.01 or buy < 0.01 or sell > buy * 10:
        return False
    return True

def build_locked_keyboard(wait_seconds):
    return InlineKeyboardMarkup(
        [[InlineKeyboardButton(f"⏳ يرجى الانتظار {wait_seconds} ثانية ...", callback_data=f"locked_{wait_seconds}")]]
    )

async def countdown_and_execute(query, seconds, final_text, final_markup, parse_mode="HTML"):
    for remaining in range(seconds, 0, -1):
        try:
            await query.edit_message_reply_markup(reply_markup=build_locked_keyboard(remaining))
        except Exception:
            pass
        await asyncio.sleep(1)
    try:
        await query.edit_message_text(final_text, reply_markup=final_markup, parse_mode=parse_mode)
    except Exception:
        pass

async def get_binance_usdt_symbols():
    url = "https://api.binance.com/api/v3/exchangeInfo"
    try:
        async with aiohttp.ClientSession() as session:
            async with session.get(url, timeout=20) as resp:
                data = await resp.json()
                return set(
                    s['symbol'][:-4] for s in data['symbols']
                    if s['quoteAsset'] == 'USDT' and s['status'] == 'TRADING'
                )
    except Exception:
        return set(['BTC', 'ETH', 'BNB', 'SOL', 'DOGE', 'ADA', 'XRP'])

async def get_all_coins():
    url = "https://api.coingecko.com/api/v3/coins/list"
    try:
        async with aiohttp.ClientSession() as session:
            async with session.get(url, timeout=45) as resp:
                data = await resp.json()
                return [{'id': c['id'], 'symbol': c['symbol'].upper(), 'name': c['name']} for c in data]
    except Exception:
        return [{'id': 'bitcoin', 'symbol': 'BTC', 'name': 'Bitcoin'}]

async def get_supported_coins(limit=MAX_SCAN_COINS):
    binance_symbols = await get_binance_usdt_symbols()
    all_coins = await get_all_coins()
    filtered = [c for c in all_coins if c['symbol'] in binance_symbols]
    seen = set()
    unique = []
    for c in filtered:
        key = (c['id'], c['symbol'])
        if key not in seen:
            unique.append(c)
            seen.add(key)
    return unique[:limit]

async def get_top_coins(limit=12):
    global coins_cache, coins_cache_time
    now = time.time()
    if coins_cache and (now - coins_cache_time) < CACHE_SECONDS:
        return coins_cache
    url = "https://api.coingecko.com/api/v3/coins/markets"
    params = {"vs_currency": "usd", "order": "market_cap_desc", "per_page": limit, "page": 1}
    try:
        async with aiohttp.ClientSession() as session:
            async with session.get(url, params=params, timeout=25) as resp:
                data = await resp.json()
                coins_cache = [
                    {'id': coin['id'], 'symbol': coin['symbol'].upper(), 'name': coin['name']}
                    for coin in data
                ]
                coins_cache_time = now
                return coins_cache
    except Exception:
        return [{'id': 'bitcoin', 'symbol': 'BTC', 'name': 'Bitcoin'}]

async def get_price_binance(session, symbol):
    url = f"https://api.binance.com/api/v3/ticker/price?symbol={symbol.upper()}USDT"
    try:
        async with session.get(url, timeout=6) as resp:
            data = await resp.json()
            price = float(data['price']) if 'price' in data else None
            return price if is_price_valid(price) else None
    except Exception:
        return None

async def get_price_gateio(session, symbol):
    url = f"https://api.gateio.ws/api/v4/spot/tickers?currency_pair={symbol.upper()}_USDT"
    try:
        async with session.get(url, timeout=6) as resp:
            data = await resp.json()
            if data and 'last' in data[0]:
                price = float(data[0]['last'])
                return price if is_price_valid(price) else None
            return None
    except Exception:
        return None

async def get_price_mexc(session, symbol):
    url = f"https://api.mexc.com/api/v3/ticker/price?symbol={symbol.upper()}USDT"
    try:
        async with session.get(url, timeout=6) as resp:
            data = await resp.json()
            price = float(data['price']) if 'price' in data else None
            return price if is_price_valid(price) else None
    except Exception:
        return None

async def get_prices_async(session, symbol):
    results = await asyncio.gather(
        get_price_binance(session, symbol),
        get_price_gateio(session, symbol),
        get_price_mexc(session, symbol)
    )
    return {'Binance': results[0], 'Gate.io': results[1], 'MEXC': results[2]}

def get_coins_list_title():
    return (
        "💠 <b>اختر عملة لمتابعة الأسعار:</b>\n"
        "اضغط على العملة المطلوبة من القائمة بالأسفل.\n"
        "استخدم الأزرار أسفل القائمة لإدارة الفحص."
    )

def build_keyboard(coins, selected_coin_id=None):
    buttons = []
    row = []
    for idx, c in enumerate(coins[:12]):
        text = f"{'✅ ' if c['id'] == selected_coin_id else ''}{c['name']} ({c['symbol']})"
        btn = InlineKeyboardButton(text=text, callback_data=f"currency_{c['id']}")
        row.append(btn)
        if (idx + 1) % 3 == 0:
            buttons.append(row)
            row = []
    if row:
        buttons.append(row)
    control_row = [
        InlineKeyboardButton("🔄 تحديث", callback_data="refresh"),
        InlineKeyboardButton("🛰️ مسح", callback_data="scan"),
        InlineKeyboardButton("⏹ إيقاف", callback_data="stopauto"),
    ]
    buttons.append(control_row)
    return InlineKeyboardMarkup(buttons)

def make_price_message(coin, prices, commission=0.002, usd_amount=100):
    symbol = coin['symbol']
    name = coin['name']
    cid = coin['id']
    now = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
    out = [f"💠 <b>تفاصيل أسعار {name} ({symbol})</b>\n<code>id: {cid}</code>"]
    out.append(f"⏰ <b>آخر تحديث:</b> <code>{now}</code>")
    out.append("\n<b>السعر الحالي في المنصات:</b>")
    for ex, p in prices.items():
        price_str = f"{p:.6f} $ ✅" if p else "غير متوفر ❌"
        out.append(f"• {ex}: <b>{price_str}</b>")
    available = {k: v for k, v in prices.items() if is_price_valid(v)}
    if len(available) >= 2:
        min_platform = min(available, key=available.get)
        max_platform = max(available, key=available.get)
        buy_price = available[min_platform]
        sell_price = available[max_platform]
        diff = sell_price - buy_price
        percent = (diff / buy_price) * 100 if buy_price else 0
        total_fee = commission * 3
        buy_with_fee = buy_price * (1 + commission)
        sell_with_fee = sell_price * (1 - commission)
        net_profit = (usd_amount / buy_with_fee) * sell_with_fee - usd_amount
        net_profit_after_fees = net_profit - (usd_amount * total_fee)
        profit_percent = (net_profit_after_fees / usd_amount) * 100 if usd_amount else 0
        if not is_arbitrage_valid(buy_price, sell_price):
            out.append("\n⚠️ بيانات الأسعار غير منطقية أو السوق غير نشط في إحدى المنصات.")
        else:
            out.append(
                f"\n🔰 <b>أفضل منصة للشراء:</b> {min_platform} <code>{buy_price:.6f} $</code>"
                f"\n💰 <b>أفضل منصة للبيع:</b> {max_platform} <code>{sell_price:.6f} $</code>"
                f"\n🔝 <b>الفرق النظري:</b> <code>{diff:.6f} $</code> <b>({percent:.2f}%)</b>"
            )
            out.append(
                f"\n💲 <b>الأرباح الصافية المقدرة لمبلغ 100$:</b> <code>{net_profit_after_fees:.2f} $</code> <b>({profit_percent:.2f}%)</b>\n"
                f"🧾 <i>(تم احتساب رسوم تقريبية إجمالية {(total_fee*100):.2f}%)</i>"
            )
            if profit_percent > 0.3:
                out.append("⚡ <b>فرصة أربيتراج جيدة</b> 🔥")
    else:
        out.append("\nلا توجد بيانات كافية للمقارنة بين المنصات.")
    return "\n".join(out)

async def smart_reply(query, text, reply_markup=None, parse_mode="HTML"):
    try:
        current_markup = query.message.reply_markup
        if query.message.text == text and current_markup == reply_markup:
            return
        await query.edit_message_text(text, reply_markup=reply_markup, parse_mode=parse_mode)
    except Exception as e:
        try:
            await query.message.reply_text(text, reply_markup=reply_markup, parse_mode=parse_mode)
        except Exception as e2:
            print("Both edit and reply failed:", e, e2)

async def handle_buttons(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    try:
        await query.answer()
    except Exception:
        pass

    data = query.data
    coins = context.chat_data.get("coins")
    if not coins:
        coins = await get_top_coins(12)
        context.chat_data["coins"] = coins

    text = ""
    reply_markup = None
    parse_mode = "HTML"

    if data.startswith("locked_"):
        seconds = int(data.split("_")[1])
        await query.answer(f"يرجى الانتظار {seconds} ثانية...", show_alert=True)
        return

    if data.startswith("currency_"):
        lock_time = 3
        await smart_reply(query, "⏳ يتم جلب الأسعار، يرجى الانتظار...", reply_markup=build_locked_keyboard(lock_time))
        coin_id = data.split("_", 1)[1]
        coin = next((c for c in context.chat_data.get("coins", []) if c['id'] == coin_id), None)
        if not coin:
            text = "حدث خطأ في تحديد العملة."
            reply_markup = build_keyboard(context.chat_data["coins"])
        else:
            context.chat_data["selected_coin_id"] = coin_id
            async with aiohttp.ClientSession() as session:
                prices = await get_prices_async(session, coin['symbol'])
            text = make_price_message(coin, prices)
            reply_markup = build_keyboard(context.chat_data["coins"], selected_coin_id=coin_id)
        await countdown_and_execute(query, lock_time, text, reply_markup)
        return

    elif data == "refresh":
        lock_time = LOCK_TIMES.get("refresh", 5)
        await smart_reply(query, "⏳ يتم تحديث الأسعار، يرجى الانتظار...", reply_markup=build_locked_keyboard(lock_time))
        coin_id = context.chat_data.get("selected_coin_id")
        if coin_id:
            coin = next((c for c in context.chat_data.get("coins", []) if c['id'] == coin_id), None)
            if not coin:
                text = "حدث خطأ في تحديد العملة."
                reply_markup = build_keyboard(context.chat_data["coins"])
            else:
                async with aiohttp.ClientSession() as session:
                    prices = await get_prices_async(session, coin['symbol'])
                text = make_price_message(coin, prices)
                reply_markup = build_keyboard(context.chat_data["coins"], selected_coin_id=coin_id)
        else:
            text = "يرجى اختيار العملة أولاً من الأزرار."
            reply_markup = build_keyboard(context.chat_data["coins"])
        await countdown_and_execute(query, lock_time, text, reply_markup)
        return

    elif data == "stopauto":
        lock_time = LOCK_TIMES.get("stopauto", 5)
        await smart_reply(query, "⏳ يتم الآن إيقاف التحديث التلقائي...", reply_markup=build_locked_keyboard(lock_time))
        old_job = context.chat_data.get('auto_job')
        if old_job:
            old_job.schedule_removal()
            context.chat_data['auto_job'] = None
            text = "تم إيقاف التحديث التلقائي."
            reply_markup = build_keyboard(context.chat_data["coins"])
        else:
            text = "لا يوجد تحديث تلقائي نشط حالياً."
            reply_markup = build_keyboard(context.chat_data["coins"])
        await countdown_and_execute(query, lock_time, text, reply_markup)
        return

    elif data == "scan":
        lock_time = LOCK_TIMES.get("scan", 30)
        await smart_reply(query, "⏳ يتم البحث عن أفضل فرص الأربيتراج، يرجى الانتظار...", reply_markup=build_locked_keyboard(lock_time))
        min_percent = context.chat_data.get('scan_min_percent', DEFAULT_MIN_ARBITRAGE_PERCENT)
        num_results = context.chat_data.get('scan_num_results', 1)
        # جلب النتائج أثناء العداد
        coins_list = await get_supported_coins(MAX_SCAN_COINS)
        # قد يستغرق ذلك وقتًا طويلاً حسب العدد
        # جلب النتائج أثناء العد
        scan_results = await scan_callback_prepare(context, coins_list, min_percent, num_results)
        await countdown_and_execute(query, lock_time, scan_results["text"], scan_results["reply_markup"])
        return

def get_coins_list_title():
    return (
        "💠 <b>اختر عملة لمتابعة الأسعار:</b>\n"
        "اضغط على العملة المطلوبة من القائمة بالأسفل.\n"
        "استخدم الأزرار أسفل القائمة لإدارة الفحص."
    )

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    sent_msg = await update.message.reply_text("جاري تجهيز القائمة، يرجى الانتظار ...", parse_mode="HTML")
    try:
        coins = await get_top_coins(12)
        context.chat_data["coins"] = coins
        keyboard = build_keyboard(coins)
        await sent_msg.edit_text(
            get_coins_list_title(),
            reply_markup=keyboard,
            parse_mode="HTML"
        )
    except Exception:
        await sent_msg.edit_text("حدث خطأ أثناء تحميل البيانات، حاول مرة أخرى بعد قليل.")

async def price(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not context.args:
        await update.message.reply_text("اكتب رمز العملة أو اسمها أو id مثل: /price btc eth sol أو /price tenx")
        return
    coins = await get_supported_coins(MAX_SCAN_COINS)
    messages = []
    for arg in context.args:
        arg_lower = arg.lower()
        matches = [c for c in coins if c['symbol'].lower() == arg_lower or c['id'] == arg_lower or c['name'].lower() == arg_lower]
        if len(matches) == 0:
            messages.append(f"لم يتم العثور على عملة مطابقة لـ {arg}")
        elif len(matches) == 1:
            coin = matches[0]
            async with aiohttp.ClientSession() as session:
                prices = await get_prices_async(session, coin['symbol'])
            msg = make_price_message(coin, prices)
            messages.append(msg)
        else:
            options = "\n".join([f"- {c['name']} ({c['symbol']}) [id: {c['id']}]" for c in matches])
            messages.append(f"وجدت أكثر من عملة برمز/اسم {arg}:\n{options}\nيرجى تحديد id في الأمر: /price <id>")
    await update.message.reply_text('\n\n' + ('-'*30) + '\n\n'.join(messages), parse_mode="HTML")

async def auto(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not context.args:
        await update.message.reply_text("اكتب رمز العملة أو اسمها أو id مثل: /auto btc eth sol")
        return
    coins = await get_supported_coins(MAX_SCAN_COINS)
    coin_infos = []
    for arg in context.args:
        arg_lower = arg.lower()
        matches = [c for c in coins if c['symbol'].lower() == arg_lower or c['id'] == arg_lower or c['name'].lower() == arg_lower]
        if len(matches) == 1:
            coin_infos.append(matches[0])
    chat_id = update.effective_chat.id

    old_job = context.chat_data.get('auto_job')
    if old_job:
        old_job.schedule_removal()

    async def send_prices(context: CallbackContext):
        messages = []
        async with aiohttp.ClientSession() as session:
            for coin in coin_infos:
                prices = await get_prices_async(session, coin['symbol'])
                msg = make_price_message(coin, prices)
                messages.append(msg)
        await context.bot.send_message(chat_id=chat_id, text='\n\n' + ('-'*30) + '\n\n'.join(messages), parse_mode="HTML")

    job = context.job_queue.run_repeating(send_prices, interval=60, first=0)
    context.chat_data['auto_job'] = job
    await update.message.reply_text(
        f"سيتم إرسال أسعار هذه العملات تلقائيًا كل دقيقة: {'، '.join([c['name'] for c in coin_infos])}.\nلإيقاف التحديث أرسل /stopauto"
    )

async def stopauto(update: Update, context: ContextTypes.DEFAULT_TYPE):
    old_job = context.chat_data.get('auto_job')
    if old_job:
        old_job.schedule_removal()
        context.chat_data['auto_job'] = None
        await update.message.reply_text("تم إيقاف التحديث التلقائي.")
    else:
        await update.message.reply_text("لا يوجد تحديث تلقائي نشط حالياً.")

def parse_scan_args(args):
    min_percent = DEFAULT_MIN_ARBITRAGE_PERCENT
    num_results = 1
    limit = MAX_SCAN_COINS
    for arg in args:
        if arg.replace('.', '', 1).isdigit():
            min_percent = float(arg)
        elif arg.isdigit():
            num_results = int(arg)
        elif arg.startswith("limit=") and arg[6:].isdigit():
            l = int(arg[6:])
            limit = min(l, MAX_SCAN_COINS)
    return min_percent, num_results, limit

async def scan(update: Update, context: ContextTypes.DEFAULT_TYPE):
    min_percent, num_results, limit = parse_scan_args(context.args)
    context.chat_data['scan_min_percent'] = min_percent
    context.chat_data['scan_num_results'] = num_results

    if limit > MAX_SCAN_COINS:
        await update.message.reply_text(f"الحد الأقصى لعدد العملات هو {MAX_SCAN_COINS}. سيتم الفحص على أول {MAX_SCAN_COINS} عملة فقط.")
        limit = MAX_SCAN_COINS
    elif limit > 200:
        await update.message.reply_text(f"تحذير: البحث في أكثر من 200 عملة قد يستغرق وقتًا طويلاً...")

    coins = await get_supported_coins(limit)
    await update.message.reply_text(
        f"جاري مسح {len(coins)} عملة مدعومة في بايننس بحثًا عن أفضل {num_results} فرصة أربيتراج بنسبة أكبر من {min_percent}% ..."
    )

    results = []
    async with aiohttp.ClientSession() as session:
        tasks = [get_prices_async(session, c['symbol']) for c in coins]
        all_prices = await asyncio.gather(*tasks)

    for coin, prices in zip(coins, all_prices):
        available = {k: v for k, v in prices.items() if is_price_valid(v)}
        if len(available) >= 2:
            min_platform = min(available, key=available.get)
            max_platform = max(available, key=available.get)
            buy_price = available[min_platform]
            sell_price = available[max_platform]
            if not is_arbitrage_valid(buy_price, sell_price):
                continue
            diff = sell_price - buy_price
            percent = (diff / buy_price) * 100 if buy_price else 0
            if percent > min_percent:
                results.append({
                    "name": coin['name'],
                    "symbol": coin['symbol'],
                    "id": coin['id'],
                    "buy_platform": min_platform,
                    "buy_price": buy_price,
                    "sell_platform": max_platform,
                    "sell_price": sell_price,
                    "diff": diff,
                    "percent": percent
                })

    results.sort(key=lambda x: x['percent'], reverse=True)
    results = results[:num_results]

    if results:
        msgs = []
        for res in results:
            msg = (
                f"🔍 فرصة أربيتراج:\n"
                f"العملة: {res['name']} ({res['symbol']}) [id: {res['id']}]\n"
                f"شراء من: {res['buy_platform']} بسعر {res['buy_price']:.6f} $\n"
                f"بيع في: {res['sell_platform']} بسعر {res['sell_price']:.6f} $\n"
                f"الفرق: {res['diff']:.6f} $ ({res['percent']:.2f}%)"
                "\n⚡ فرصة أربيتراج جيدة!" if res['percent'] >= min_percent else ""
            )
            msgs.append(msg)
        await update.message.reply_text(
            f"تم فحص {len(coins)} عملة.\n\n" +
            '\n\n'.join(msgs)
        )
    else:
        await update.message.reply_text(
            f"تم فحص {len(coins)} عملة.\nلم يتم العثور على فرص أربيتراج بهذه الشروط حالياً."
        )

# هذه الدالة تُجهز نتائج scan أثناء العد التنازلي
async def scan_callback_prepare(context, coins, min_percent, num_results):
    results = []
    async with aiohttp.ClientSession() as session:
        tasks = [get_prices_async(session, c['symbol']) for c in coins]
        all_prices = await asyncio.gather(*tasks)
    for coin, prices in zip(coins, all_prices):
        available = {k: v for k, v in prices.items() if is_price_valid(v)}
        if len(available) >= 2:
            min_platform = min(available, key=available.get)
            max_platform = max(available, key=available.get)
            buy_price = available[min_platform]
            sell_price = available[max_platform]
            if not is_arbitrage_valid(buy_price, sell_price):
                continue
            diff = sell_price - buy_price
            percent = (diff / buy_price) * 100 if buy_price else 0
            if percent > min_percent:
                results.append({
                    "name": coin['name'],
                    "symbol": coin['symbol'],
                    "id": coin['id'],
                    "buy_platform": min_platform,
                    "buy_price": buy_price,
                    "sell_platform": max_platform,
                    "sell_price": sell_price,
                    "diff": diff,
                    "percent": percent
                })
    results.sort(key=lambda x: x['percent'], reverse=True)
    results = results[:num_results]

    if results:
        ids_in_results = [r['id'] for r in results]
        new_coins_for_buttons = [c for c in coins if c['id'] in ids_in_results][:12]
        if len(new_coins_for_buttons) < 12:
            ids = set([c['id'] for c in new_coins_for_buttons])
            for c in coins:
                if c['id'] not in ids:
                    new_coins_for_buttons.append(c)
                if len(new_coins_for_buttons) == 12:
                    break
    else:
        new_coins_for_buttons = coins[:12]

    context.chat_data["coins"] = new_coins_for_buttons

    text = ""
    reply_markup = build_keyboard(new_coins_for_buttons)
    if results:
        msgs = []
        for res in results:
            msg = (
                f"🔍 فرصة أربيتراج:\n"
                f"العملة: {res['name']} ({res['symbol']}) [id: {res['id']}]\n"
                f"شراء من: {res['buy_platform']} بسعر {res['buy_price']:.6f} $\n"
                f"بيع في: {res['sell_platform']} بسعر {res['sell_price']:.6f} $\n"
                f"الفرق: {res['diff']:.6f} $ ({res['percent']:.2f}%)"
                "\n⚡ فرصة أربيتراج جيدة!" if res['percent'] >= min_percent else ""
            )
            msgs.append(msg)
        text = f"تم فحص {len(coins)} عملة.\n\n" + '\n\n'.join(msgs)
    else:
        text = (
            f"تم فحص {len(coins)} عملة.\nلم يتم العثور على فرص أربيتراج بهذه الشروط حالياً."
        )
    return {"text": text, "reply_markup": reply_markup}

async def error_handler(update, context):
    print(f"Exception: {context.error}")

if __name__ == '__main__':
    import logging
    logging.basicConfig(level=logging.INFO)
    app = ApplicationBuilder().token(TELEGRAM_BOT_TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("price", price))
    app.add_handler(CommandHandler("auto", auto))
    app.add_handler(CommandHandler("stopauto", stopauto))
    app.add_handler(CommandHandler("scan", scan))
    app.add_handler(CallbackQueryHandler(handle_buttons))
    app.add_error_handler(error_handler)
    print("Bot started...")
    app.run_polling()
