#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import logging
import os
from telegram import Update, ReplyKeyboardMarkup, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    Application,
    CommandHandler,
    MessageHandler,
    CallbackQueryHandler,
    filters,
    ConversationHandler,
    ContextTypes,
)

# ===== НАСТРОЙКИ =====
logging.basicConfig(
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s", level=logging.INFO
)
logger = logging.getLogger(__name__)

TOKEN = os.environ.get("TELEGRAM_BOT_TOKEN", "8783547450:AAF5k94P1Q6Jv7gK8b5gfMGv5fMZQTy3yIQ")
GROUP_CHAT_ID = int(os.environ.get("GROUP_CHAT_ID", -1003667867938))

# ===== КЛАВИАТУРА МЕНЮ =====
MENU_BUTTONS = [
    ["📢 Жалоба", "⚠️ Проблема"],
    ["❓ Вопрос", "🎓 Апелляция"],
    ["🛒 Магазин"]
]
CATEGORY_TEXTS = [btn for row in MENU_BUTTONS for btn in row]
BUTTON_TO_CATEGORY = {
    "📢 Жалоба": "Жалоба",
    "⚠️ Проблема": "Проблема",
    "❓ Вопрос": "Вопрос",
    "🎓 Апелляция": "Апелляция",
    "🛒 Магазин": "Магазин"
}

def get_main_menu():
    return ReplyKeyboardMarkup(MENU_BUTTONS, resize_keyboard=True)

# ===== ХРАНИЛИЩА ЗАЯВОК =====
requests_db = {}          # ключ: (chat_id, msg_id) -> dict
active_requests = {}      # user_id -> ключ заявки
closed_requests = {}      # user_id -> ключ последней закрытой

# ===== СОСТОЯНИЯ ДИАЛОГА =====
WAITING_NICK = 1
WAITING_ISSUE = 2

# ===== ЗАКРЫТИЕ ЗАЯВКИ (ТОЛЬКО ПО КНОПКЕ) =====
async def close_request(key, context: ContextTypes.DEFAULT_TYPE):
    data = requests_db.get(key)
    if not data or data["status"] != "open":
        return False
    data["status"] = "closed"
    user_id = data["user_id"]
    closed_requests[user_id] = key
    if user_id in active_requests and active_requests[user_id] == key:
        del active_requests[user_id]
    try:
        await context.bot.send_message(
            chat_id=user_id,
            text="🔒 Ваша заявка была закрыта администратором. Можете создать новую.",
        )
    except Exception as e:
        logger.warning(f"Не удалось уведомить пользователя {user_id}: {e}")
    logger.info(f"Заявка {key} закрыта.")
    return True

# ===== ОТПРАВКА ЗАЯВКИ В ГРУППУ =====
async def send_to_group(user_id, category, nick, issue, context):
    text = (
        f"📩 <b>Новая заявка</b>\n"
        f"<b>Категория:</b> {category}\n"
        f"<b>Ник:</b> {nick}\n"
        f"<b>User ID:</b> <code>{user_id}</code>\n"
        f"<b>Суть обращения:</b>\n{issue}"
    )
    try:
        msg = await context.bot.send_message(chat_id=GROUP_CHAT_ID, text=text, parse_mode="HTML")
        key = (GROUP_CHAT_ID, msg.message_id)
        requests_db[key] = {
            "user_id": user_id,
            "category": category,
            "nick": nick,
            "issue": issue,
            "status": "open"
        }
        active_requests[user_id] = key
        # Кнопка закрытия
        keyboard = [[InlineKeyboardButton("🔒 Закрыть заявку", callback_data=f"close_{msg.message_id}")]]
        await msg.edit_reply_markup(InlineKeyboardMarkup(keyboard))
        logger.info(f"Заявка {msg.message_id} от user_id={user_id} создана")
        return True, key
    except Exception as e:
        logger.error(f"Ошибка отправки в группу: {e}")
        return False, None

# ===== КОМАНДЫ =====
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "👋 Привет! Я бот поддержки.\nВыберите категорию обращения:",
        reply_markup=get_main_menu()
    )

async def status_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    if user_id in active_requests:
        key = active_requests[user_id]
        data = requests_db[key]
        text = (
            f"📋 <b>Статус вашей заявки</b>\n"
            f"<b>Статус:</b> 🟡 Открыта\n"
            f"<b>Категория:</b> {data['category']}\n"
            f"<b>Ник:</b> {data['nick']}\n"
            f"<b>Суть:</b> {data['issue']}"
        )
    elif user_id in closed_requests:
        key = closed_requests[user_id]
        data = requests_db[key]
        text = (
            f"📋 <b>Последняя заявка</b>\n"
            f"<b>Статус:</b> 🟢 Закрыта\n"
            f"<b>Категория:</b> {data['category']}\n"
            f"<b>Ник:</b> {data['nick']}\n"
            f"<b>Суть:</b> {data['issue']}"
        )
    else:
        text = "У вас пока нет заявок."
    await update.message.reply_text(text, parse_mode="HTML")

# ===== ДИАЛОГ СОЗДАНИЯ ЗАЯВКИ =====
async def category_button(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    user_id = update.effective_user.id
    if user_id in active_requests:
        await update.message.reply_text(
            "❌ У вас уже есть открытая заявка. Дождитесь её закрытия.\n"
            "Проверить статус: /status",
            reply_markup=get_main_menu()
        )
        return ConversationHandler.END

    text = update.message.text
    category = BUTTON_TO_CATEGORY.get(text, text)
    context.user_data["category"] = category
    await update.message.reply_text(
        f"📌 Категория: <b>{category}</b>\n\nВведите ваш ник (или @username):",
        parse_mode="HTML",
        reply_markup=ReplyKeyboardMarkup([[]], resize_keyboard=True)
    )
    return WAITING_NICK

async def nick_received(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    nick = update.message.text.strip()
    context.user_data["nick"] = nick
    await update.message.reply_text("Теперь подробно опишите суть обращения:")
    return WAITING_ISSUE

async def issue_received(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    issue = update.message.text.strip()
    user_data = context.user_data
    category = user_data.get("category", "Не указана")
    nick = user_data.get("nick", "Не указан")
    user_id = update.effective_user.id

    success, key = await send_to_group(user_id, category, nick, issue, context)
    if success:
        await update.message.reply_text(
            "✅ Ваша заявка успешно создана! Ожидайте ответа.\n"
            "Статус: /status",
            reply_markup=get_main_menu()
        )
    else:
        await update.message.reply_text(
            "❌ Ошибка при создании заявки. Попробуйте позже.",
            reply_markup=get_main_menu()
        )
    user_data.clear()
    return ConversationHandler.END

# ===== ОТВЕТ МОДЕРАТОРА В ГРУППЕ =====
async def group_reply(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Модератор отвечает на заявку реплаем -> префикс + копия пользователю."""
    if update.effective_chat.id != GROUP_CHAT_ID:
        return
    message = update.message
    if not message.reply_to_message or message.reply_to_message.from_user.id != context.bot.id:
        return

    original_msg_id = message.reply_to_message.message_id
    key = (GROUP_CHAT_ID, original_msg_id)
    data = requests_db.get(key)
    if not data:
        logger.warning(f"Заявка для msg_id={original_msg_id} не найдена")
        return

    user_id = data["user_id"]
    try:
        await context.bot.send_message(
            chat_id=user_id,
            text="👨‍💻 <b>Команда поддержки:</b>",
            parse_mode="HTML"
        )
        await context.bot.copy_message(
            chat_id=user_id,
            from_chat_id=GROUP_CHAT_ID,
            message_id=message.message_id,
        )
        await message.reply_text("✅ Ответ отправлен пользователю.", quote=False)
        logger.info(f"Ответ на заявку {original_msg_id} отправлен пользователю {user_id}")
    except Exception as e:
        logger.error(f"Ошибка пересылки ответа: {e}")
        await message.reply_text(f"❌ Ошибка: {e}", quote=False)

# ===== ОТВЕТ ПОЛЬЗОВАТЕЛЯ В ЛИЧКЕ =====
async def user_reply(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Пользователь отвечает на сообщение бота -> пересылается в группу."""
    if update.effective_chat.type != "private":
        return
    message = update.message
    if not message.reply_to_message or message.reply_to_message.from_user.id != context.bot.id:
        return

    user_id = update.effective_user.id
    if user_id not in active_requests:
        await message.reply_text("У вас нет открытых заявок для ответа.")
        return

    key = active_requests[user_id]
    original_msg_id = key[1]
    try:
        # Уведомление в группе
        await context.bot.send_message(
            chat_id=GROUP_CHAT_ID,
            text=f"📩 <b>Ответ пользователя</b> (ID <code>{user_id}</code>):",
            parse_mode="HTML",
            reply_to_message_id=original_msg_id,
        )
        # Само сообщение пользователя
        await context.bot.copy_message(
            chat_id=GROUP_CHAT_ID,
            from_chat_id=user_id,
            message_id=message.message_id,
            reply_to_message_id=original_msg_id,
        )
        logger.info(f"Ответ пользователя {user_id} переслан в группу к заявке {original_msg_id}")
    except Exception as e:
        logger.error(f"Ошибка пересылки ответа пользователя: {e}")
        await message.reply_text("❌ Не удалось отправить ответ.")

# ===== КНОПКА ЗАКРЫТИЯ ЗАЯВКИ =====
async def close_button(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    if not query.data.startswith("close_"):
        return
    try:
        msg_id = int(query.data.split("_")[1])
    except (IndexError, ValueError):
        return
    key = (GROUP_CHAT_ID, msg_id)
    if key not in requests_db:
        await query.edit_message_text("❌ Заявка не найдена.")
        return
    success = await close_request(key, context)
    if success:
        await query.edit_message_reply_markup(reply_markup=None)
        await query.edit_message_text(
            text=query.message.text + "\n\n🔒 <b>Заявка закрыта модератором.</b>",
            parse_mode="HTML",
            reply_markup=None
        )
    else:
        await query.edit_message_text("❌ Не удалось закрыть заявку.")

# ===== ПРОЧИЕ СООБЩЕНИЯ =====
async def unknown_text(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "Используйте кнопки меню для создания обращения или команду /status.",
        reply_markup=get_main_menu()
    )

async def unknown_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Неизвестная команда. Доступны: /start, /status")

# ===== ЗАПУСК =====
def main():
    app = Application.builder().token(TOKEN).build()

    conv_handler = ConversationHandler(
        entry_points=[MessageHandler(filters.Text(CATEGORY_TEXTS), category_button)],
        states={
            WAITING_NICK: [MessageHandler(filters.TEXT & ~filters.COMMAND, nick_received)],
            WAITING_ISSUE: [MessageHandler(filters.TEXT & ~filters.COMMAND, issue_received)],
        },
        fallbacks=[],
        allow_reentry=True,
    )

    app.add_handler(conv_handler)
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("status", status_command))

    # Ответы в группе (реплай на сообщения бота)
    app.add_handler(MessageHandler(filters.Chat(GROUP_CHAT_ID) & filters.REPLY, group_reply))

    # Ответы пользователей в личке (реплай на сообщения бота) – приоритет над обычным текстом
    app.add_handler(MessageHandler(filters.ChatType.PRIVATE & filters.REPLY, user_reply))

    app.add_handler(CallbackQueryHandler(close_button, pattern="^close_"))
    app.add_handler(MessageHandler(filters.COMMAND, unknown_command))
    app.add_handler(MessageHandler(filters.TEXT, unknown_text))

    logger.info("🤖 Бот запущен с поддержкой диалога и уведомлениями «Команда поддержки».")
    app.run_polling(allowed_updates=Update.ALL_TYPES)

if __name__ == "__main__":
    main()
