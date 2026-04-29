#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import logging
import os
from telegram import Update, ReplyKeyboardMarkup
from telegram.ext import (
    Application,
    CommandHandler,
    MessageHandler,
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
GROUP_CHAT_ID = int(os.environ.get("GROUP_CHAT_ID", -1003906994235))

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
# Ключ: (chat_id, message_id)  ->  данные заявки
reply_mapping = {}
# Активная (открытая) заявка пользователя: user_id -> ключ в reply_mapping
active_requests = {}
# Последняя закрытая заявка: user_id -> ключ в reply_mapping
closed_requests = {}

# ===== СОСТОЯНИЯ ДИАЛОГА =====
WAITING_NICK = 1
WAITING_ISSUE = 2

# ===== ОТПРАВКА В ГРУППУ =====
async def send_to_group(user_id: int, category: str, nick: str, issue: str, context: ContextTypes.DEFAULT_TYPE):
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
        reply_mapping[key] = {
            "user_id": user_id,
            "category": category,
            "nick": nick,
            "issue": issue,
            "status": "open"
        }
        active_requests[user_id] = key
        logger.info(f"Заявка от user_id={user_id} отправлена в группу, msg_id={msg.message_id}")
        return True, key
    except Exception as e:
        logger.error(f"Ошибка отправки в группу: {e}")
        return False, None

# ===== /start =====
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "👋 Привет! Я бот поддержки.\nВыберите категорию обращения, нажав на кнопку:",
        reply_markup=get_main_menu()
    )

# ===== /status =====
async def status(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    # Проверяем открытую заявку
    if user_id in active_requests:
        key = active_requests[user_id]
        data = reply_mapping[key]
        status_text = (
            f"📋 <b>Статус вашей заявки</b>\n"
            f"<b>Статус:</b> 🟡 Открыта (ожидает ответа)\n"
            f"<b>Категория:</b> {data['category']}\n"
            f"<b>Ник:</b> {data['nick']}\n"
            f"<b>Суть:</b> {data['issue']}\n"
        )
        await update.message.reply_text(status_text, parse_mode="HTML")
    # Проверяем последнюю закрытую
    elif user_id in closed_requests:
        key = closed_requests[user_id]
        data = reply_mapping[key]
        status_text = (
            f"📋 <b>Статус последней заявки</b>\n"
            f"<b>Статус:</b> 🟢 Закрыта (ответ отправлен)\n"
            f"<b>Категория:</b> {data['category']}\n"
            f"<b>Ник:</b> {data['nick']}\n"
            f"<b>Суть:</b> {data['issue']}\n"
        )
        await update.message.reply_text(status_text, parse_mode="HTML")
    else:
        await update.message.reply_text("У вас пока нет заявок.")

# ===== ОБРАБОТЧИК ВЫБОРА КАТЕГОРИИ (НАЧАЛО ДИАЛОГА) =====
async def category_button(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    """Нажата кнопка категории. Проверяем, нет ли открытой заявки."""
    user_id = update.effective_user.id
    if user_id in active_requests:
        await update.message.reply_text(
            "❌ У вас уже есть открытая заявка. Дождитесь ответа администрации.\n"
            "Проверить статус – /status",
            reply_markup=get_main_menu()
        )
        return ConversationHandler.END

    text = update.message.text
    category = BUTTON_TO_CATEGORY.get(text, text)
    context.user_data["category"] = category
    logger.info(f"Пользователь {user_id} выбрал категорию: {category}")
    await update.message.reply_text(
        f"📌 Категория: <b>{category}</b>\n\nВведите ваш ник (или @username):",
        parse_mode="HTML",
        reply_markup=ReplyKeyboardMarkup([[]], resize_keyboard=True)  # скрываем меню
    )
    return WAITING_NICK

# ===== ПОЛУЧЕНИЕ НИКА =====
async def nick_received(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    nick = update.message.text.strip()
    context.user_data["nick"] = nick
    logger.info(f"Пользователь {update.effective_user.id} ввёл ник: {nick}")
    await update.message.reply_text("Теперь подробно опишите суть обращения:")
    return WAITING_ISSUE

# ===== ПОЛУЧЕНИЕ СУТИ, ОТПРАВКА В ГРУППУ =====
async def issue_received(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    issue = update.message.text.strip()
    user_data = context.user_data
    category = user_data.get("category", "Не указана")
    nick = user_data.get("nick", "Не указан")
    user_id = update.effective_user.id

    success, key = await send_to_group(user_id, category, nick, issue, context)
    if success:
        await update.message.reply_text(
            "✅ Ваша заявка успешно создана! Ожидайте ответа администрации.\n"
            "Отслеживать статус можно командой /status",
            reply_markup=get_main_menu()
        )
    else:
        await update.message.reply_text(
            "❌ Произошла ошибка при отправке заявки. Попробуйте позже.",
            reply_markup=get_main_menu()
        )
    user_data.clear()
    return ConversationHandler.END

# ===== ОТВЕТ МОДЕРАТОРА В ГРУППЕ =====
async def group_reply(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_chat.id != GROUP_CHAT_ID:
        return
    msg = update.message
    if not msg.reply_to_message:
        return
    if msg.reply_to_message.from_user.id != context.bot.id:
        return

    original_msg_id = msg.reply_to_message.message_id
    key = (GROUP_CHAT_ID, original_msg_id)
    request_data = reply_mapping.get(key)

    if not request_data:
        logger.warning(f"Не найдена заявка для msg_id={original_msg_id}")
        return

    user_id = request_data["user_id"]
    # Отправляем ответ пользователю
    try:
        await context.bot.copy_message(
            chat_id=user_id,
            from_chat_id=GROUP_CHAT_ID,
            message_id=msg.message_id,
        )
        # Обновляем статус заявки
        request_data["status"] = "closed"
        closed_requests[user_id] = key
        if user_id in active_requests and active_requests[user_id] == key:
            del active_requests[user_id]
        await msg.reply_text("✅ Ответ отправлен пользователю, заявка закрыта.", quote=False)
        logger.info(f"Заявка {original_msg_id} закрыта, ответ отправлен пользователю {user_id}")
    except Exception as e:
        logger.error(f"Ошибка пересылки ответа пользователю {user_id}: {e}")
        await msg.reply_text(f"❌ Не удалось отправить ответ: {e}", quote=False)

# ===== ПРОЧИЕ СООБЩЕНИЯ (НЕ КОМАНДЫ, НЕ КНОПКИ МЕНЮ) =====
async def unknown_text(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "Используйте кнопки меню для создания обращения или команду /status.",
        reply_markup=get_main_menu()
    )

# ===== ОБРАБОТЧИК НЕИЗВЕСТНЫХ КОМАНД =====
async def unknown_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Неизвестная команда. Доступны: /start, /status")

# ===== ЗАПУСК ПРИЛОЖЕНИЯ =====
def main():
    app = Application.builder().token(TOKEN).build()

    # ConversationHandler для сбора заявки
    conv_handler = ConversationHandler(
        entry_points=[
            MessageHandler(filters.Text(CATEGORY_TEXTS), category_button)
        ],
        states={
            WAITING_NICK: [MessageHandler(filters.TEXT & ~filters.COMMAND, nick_received)],
            WAITING_ISSUE: [MessageHandler(filters.TEXT & ~filters.COMMAND, issue_received)],
        },
        fallbacks=[],
        allow_reentry=True,
    )

    app.add_handler(conv_handler)
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("status", status))
    # Ответы в группе (только целевая группа, реплай на сообщения бота)
    app.add_handler(MessageHandler(filters.REPLY & filters.Chat(GROUP_CHAT_ID), group_reply))
    # Неизвестные команды
    app.add_handler(MessageHandler(filters.COMMAND, unknown_command))
    # Любой другой текст
    app.add_handler(MessageHandler(filters.TEXT, unknown_text))

    logger.info("🤖 Бот запущен и ожидает сообщений.")
    app.run_polling(allowed_updates=Update.ALL_TYPES)

if __name__ == "__main__":
    main()
