import asyncio
import logging
import os
from datetime import datetime
import json
import httpx
from aiohttp import web
from aiogram import Bot, Dispatcher, types
from aiogram.filters import Command
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from apscheduler.triggers.cron import CronTrigger

# НАСТРОЙКИ БОТА
BOT_TOKEN = "8650318030:AAF8a7FICg_Zs0iEm9QVDwMgonrNCxhleg8"
CHAT_ID = "YOUR_CHAT_ID"  # Можно узнать переслав сообщение в @userinfobot
POLLS_THREAD_ID = None  # ID топика "Голосовалки" (если есть разделы)
TIPS_THREAD_ID = None  # ID топика "Подумай об этом" (если есть разделы)

# Ключ Gemini API для генерации уникальных советов (Опционально)
# Если ключа нет, бот будет использовать встроенную базу профессиональных советов
GEMINI_API_KEY = "YOUR_GEMINI_API_KEY" 

logging.basicConfig(level=logging.INFO)
bot = Bot(token=BOT_TOKEN)
dp = Dispatcher()
scheduler = AsyncIOScheduler()

# Встроенная база баскетбольных статей на случай отсутствия ключа AI
BASKETBALL_TIPS_DATABASE = [
    {
        "title": "🏀 СЕКРЕТ ИДЕАЛЬНОГО БРОСКА: МЕХАНИКА И ТОЧКИ КОНТРОЛЯ",
        "text": "Каждый великий снайпер знает: стабильный бросок — это чистая физика и мышечная память. Локоть должен быть строго сонаправлен кольцу под углом 90 градусов. Не заводите мяч далеко за голову. Важнейшая точка контроля — фиксация расслабленной кисти («гусиная шея») после релиза мяча до его касания кольца. Это придает правильное обратное вращение."
    },
    {
        "title": "💪 РАЗВИТИЕ МЫШЦ КОРА (БАЗА ДЛЯ ПРЫЖКА И СТАБИЛЬНОСТИ)",
        "text": "Мышцы кора (пресс, косые мышцы, поясница) — это мост передачи энергии от ног к рукам. Без сильного кора вы будете терять баланс при броске в прыжке и контакте в воздухе. Добавьте в тренировки статическую планку (3 подхода по 1.5 мин), боковые планки и динамические скручивания «книжка». Это защитит поясницу от травм."
    },
    {
        "title": "⚡️ ДРИБЛИНГ СО СМЕНОЙ ТЕМПА (HESITATION MOVE)",
        "text": "Самый эффективный кроссовер — это не самый быстрый, а тот, который меняет темп. Научитесь усыплять бдительность защитника: ведите мяч высоко и медленно, выпрямляя корпус (имитируя подготовку к броску или передаче), а затем резко взрывайтесь вниз с низким ведением. Это заставит защитника подняться на носки."
    },
    {
        "title": "🛡️ ЗАЩИТА ОДИН НА ОДИН: РАБОТА НОГ И КОРПУСА",
        "text": "При защите на периметре никогда не скрещивайте ноги — двигайтесь приставными шагами в низкой стойке. Смотрите сопернику в область пояса (ее невозможно сымитировать финтом в отличие от головы или мяча). Держите одну руку на уровне его глаз для помехи броску, а вторую опустите ниже для перехвата передач."
    }
]

tip_counter = 0

async def generate_or_get_tip():
    global tip_counter
    if not GEMINI_API_KEY or GEMINI_API_KEY == "YOUR_GEMINI_API_KEY":
        # Используем встроенную базу по очереди
        tip = BASKETBALL_TIPS_DATABASE[tip_counter % len(BASKETBALL_TIPS_DATABASE)]
        tip_counter += 1
        return f"<b>{tip['title']}</b>\n\n{tip['text']}\n\n<i>Подумайте об этом! Стабильность создается в деталях.</i>"
    
    # Иначе генерируем через Gemini API
    try:
        url = "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent"
        headers = {"Content-Type": "application/json"}
        params = {"key": GEMINI_API_KEY}
        payload = {
            "contents": [{
                "parts": [{
                    "text": "Напиши профессиональную, полезную, мотивирующую статью о баскетболе (тактика, ОФП, дриблинг, бросок, защита) в формате HTML для Telegram. Используй теги <b>, <i>, <code>. Статья должна начинаться с яркого заголовка с эмодзи."
                }]
            }]
        }
        async with httpx.AsyncClient() as client:
            response = await client.post(url, json=payload, headers=headers, params=params, timeout=15.0)
            res_data = response.json()
            raw_text = res_data['candidates'][0]['content']['parts'][0]['text']
            return raw_text
    except Exception as e:
        logging.error(f"Ошибка вызова Gemini: {e}")
        tip = BASKETBALL_TIPS_DATABASE[0]
        return f"<b>{tip['title']}</b>\n\n{tip['text']}"

# ФУНКЦИИ ОТПРАВКИ ОПРОСОВ И СТАТЕЙ

async def send_custom_poll(question, options, emoji):
    try:
        poll_options = list(options) + [emoji]
        await bot.send_poll(
            chat_id=CHAT_ID,
            question=question,
            options=poll_options,
            is_anonymous=False, # Видим, кто придет на тренировку!
            message_thread_id=POLLS_THREAD_ID
        )
        logging.info(f"Опрос отправлен: {question}")
    except Exception as e:
        logging.error(f"Ошибка отправки опроса: {e}")

async def send_basketball_tip():
    try:
        text = await generate_or_get_tip()
        await bot.send_message(
            chat_id=CHAT_ID,
            text=text,
            parse_mode="HTML",
            message_thread_id=TIPS_THREAD_ID
        )
        logging.info("Полезный совет отправлен!")
    except Exception as e:
        logging.error(f"Ошибка отправки совета: {e}")

# ОПРЕДЕЛЕНИЕ РАСПИСАНИЯ ОТПРАВКИ

# 1. После Понедельника (например, во Вторник в 09:00) отправляем опрос на Четверг (21:00-22:30)
async def post_poll_for_thursday():
    await send_custom_poll(
        question="Академическая\nЧт 21:00-22:30",
        options=["Приду", "Не приду", "Не знаю"],
        emoji="🔥"
    )

# 2. После Четверга (в Пятницу в 09:00) отправляем два опроса на Субботу (9:00-11:00 утро и 18:00-20:00 вечер)
async def post_polls_for_saturday():
    # Опрос на утро
    await send_custom_poll(
        question="Академическая\nСб 9:00-11:00",
        options=["Приду", "Не приду", "Не знаю"],
        emoji="☀️"
    )
    await asyncio.sleep(2) # Небольшая пауза между опросами
    # Опрос на вечер
    await send_custom_poll(
        question="Академическая\nСб 18:00-20:00",
        options=["Приду", "Не приду", "Не знаю"],
        emoji="😎"
    )

# 3. После Субботы (в Воскресенье в 09:00) отправляем опрос на Понедельник (21:00-22:30)
async def post_poll_for_monday():
    await send_custom_poll(
        question="Академическая\nПн 21:00-22:30",
        options=["Приду", "Не приду", "Не знаю"],
        emoji="🤯"
    )

# РЕГИСТРАЦИЯ КОМАНД

@dp.message(Command("start"))
async def cmd_start(message: types.Message):
    await message.answer(
        "👋 <b>Привет, Никола!</b>\n\n"
        "Я специализированный баскетбольный бот-организатор для твоей группы.\n\n"
        "<b>Моё автоматическое расписание:</b>\n"
        "• <b>Вторник 09:00</b> — Опрос на Четверг (21:00)\n"
        "• <b>Пятница 09:00</b> — Опросы на Субботу Утро (9:00) и Вечер (18:00)\n"
        "• <b>Воскресенье 09:00</b> — Опрос на Понедельник (21:00)\n"
        "• <b>Раз в 2 дня в 12:00</b> — Публикация полезных статей в «Подумай об этом»!\n\n"
        "<b>Административные команды быстрого теста:</b>\n"
        "/poll_thursday — Отправить опрос на Четверг сейчас\n"
        "/poll_saturday — Отправить опросы на Субботу сейчас\n"
        "/poll_monday — Отправить опрос на Понедельник сейчас\n"
        "/send_tip — Сгенерировать и отправить статью о баскетболе сейчас",
        parse_mode="HTML"
    )

@dp.message(Command("poll_thursday"))
async def force_thursday(message: types.Message):
    await message.answer("🔄 Отправляю опрос на Четверг...")
    await post_poll_for_thursday()

@dp.message(Command("poll_saturday"))
async def force_saturday(message: types.Message):
    await message.answer("🔄 Отправляю опросы на Субботу...")
    await post_polls_for_saturday()

@dp.message(Command("poll_monday"))
async def force_monday(message: types.Message):
    await message.answer("🔄 Отправляю опрос на Понедельник...")
    await post_poll_for_monday()

@dp.message(Command("send_tip"))
async def force_tip(message: types.Message):
    await message.answer("🔄 Генерирую и отправляю полезный баскетбольный совет...")
    await send_basketball_tip()

# ЗАПУСК ПЛАНИРОВЩИКА С ДЕЛИКАТНЫМИ ТРИГГЕРАМИ
def setup_scheduler():
    # 1. Опрос на Четверг: запускается каждый ВТОРНИК в 09:00
    scheduler.add_job(post_poll_for_thursday, CronTrigger(day_of_week="tue", hour=9, minute=0))
    
    # 2. Опросы на Субботу: запускаются каждую ПЯТНИЦУ в 09:00
    scheduler.add_job(post_polls_for_saturday, CronTrigger(day_of_week="fri", hour=9, minute=0))
    
    # 3. Опрос на Понедельник: запускается каждое ВОСКРЕСЕНЬЕ в 09:00
    scheduler.add_job(post_poll_for_monday, CronTrigger(day_of_week="sun", hour=9, minute=0))
    
    # 4. Баскетбольные статьи: отправляются ЧЕРЕЗ ДЕНЬ (каждые 2 дня) в 12:00
    scheduler.add_job(send_basketball_tip, CronTrigger(interval="2d", hour=12, minute=0))
    
    scheduler.start()

# Простой веб-сервер для успешного прохождения проверки портов (Health Check) на Render
async def handle_ping(request):
    return web.Response(text="Basketball Bot is running live 24/7!")

async def start_web_server():
    app = web.Application()
    app.router.add_get("/", handle_ping)
    app.router.add_get("/health", handle_ping)
    runner = web.AppRunner(app)
    await runner.setup()
    port = int(os.environ.get("PORT", 8080))
    site = web.TCPSite(runner, "0.0.0.0", port)
    await site.start()
    logging.info(f"Веб-сервер заглушки запущен на порту {port}")

async def main():
    setup_scheduler()
    logging.info("Баскетбольный планировщик успешно запущен!")
    try:
        await start_web_server()
    except Exception as e:
        logging.error(f"Не удалось запустить веб-сервер: {e}")
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
