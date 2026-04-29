#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import logging
import os
from telegram import Update, ReplyKeyboardMarkup, KeyboardButton
from telegram.ext import (
    Application,
    CommandHandler,
    MessageHandler,
    filters,
    ConversationHandler,
    ContextTypes,
)

# ===== ЛОГИРОВАНИЕ (подробное, чтобы видеть ошибки) =====
logging.basicConfig(
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s", level=logging.INFO
)
logger = logging.getLogger(__name__)

# ===== ТОКЕН И ID ГРУППЫ =====
TOKEN = os.environ.get("TELEGRAM_BOT_TOKEN", "8783547450:AAF5k94P1Q6Jv7gK8b5gfMGv5fMZQTy3yIQ")
GROUP_CHAT_ID = int(os.environ.get("GROUP_CHAT_ID", -1003906994235))

# ===== КНОПКИ МЕНЮ =====
MENU_BUTTONS = [
    ["📢 Жалоба", "⚠️ Проблема"],
    ["❓ Вопрос", "🎓 Апелляция"],
    ["🛒 Магазин"]
]
CATEGORY_BUTTONS = [btn for row in MENU_BUTTONS for btn in row]  # плоский список
BUTTON_TO_CATEGORY = {
    "📢 Жалоба": "Жалоба",
    "⚠️ Проблема": "Проблема",
    "❓ Вопрос": "Вопрос",
    "🎓 Апелляция": "Апелляция",
    "🛒 Магазин": "Магазин"
}

# ===== ВСПОМОГАТЕЛЬНЫЕ ФУНКЦИИ =====
def get_main_menu() -> ReplyKeyboardMarkup:
    return ReplyKeyboardMarkup(MENU_BUTTONS, resize_keyboard=True)

# ===== СОСТОЯНИЯ ДИАЛОГА =====
WAITING_NICK = 1
WAITING_ISSUE = 2

# ===== СВЯЗЬ СООБЩЕНИЙ =====
reply_mapping = {}

async def send_to_group(user_id: int, category: str, nick: str, issue: str, context: ContextTypes.DEFAULT_TYPE):
    """Отправляет заявку в группу и сохраняет информацию для ответа."""
    text = (
        f"📩 <b>Новая заявка</b>\n"
        f"<b>Категория:</b> {category}\n"
        f"<b>Ник:</b> {nick}\n"
        f"<b>User ID:</b> <code>{user_id}</code>\n"
        f"<b>Суть вопроса:</b>\n{issue}"
    )
    try:
        msg = await context.bot.send_message(chat_id=GROUP_CHAT_ID, text=text, parse_mode="HTML")
        reply_mapping[(GROUP_CHAT_ID, msg.message_id)] = user_id
        logger.info(f"✅ Заявка от {user_id} отправлена в группу, message_id={msg.message_id}")
        return True
    except Exception as e:
        logger.error(f"❌ Ошибка отправки в группу: {e}")
        return False

# ===== КОМАНДА /start =====
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    await update.message.reply_text(
        "👋 Привет! Я бот поддержки.\nВыбери категорию обращения:",
        reply_markup=get_main_menu()
    )
    return ConversationHandler.END

# ===== ОБРАБОТЧИК НАЖАТИЯ КНОПКИ КАТЕГОРИИ (НАДЁЖНЫЙ ФИЛЬТР) =====
async def category_button(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    """Вызывается при нажатии на кнопку категории. Запрашивает ник."""
    text = update.message.text
    category = BUTTON_TO_CATEGORY.get(text, text)
    context.user_data["category"] = category
    logger.info(f"Пользователь {update.effective_user.id} выбрал категорию: {category}")
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
    await update.message.reply_text("Теперь опишите суть обращения:")
    return WAITING_ISSUE

# ===== ПОЛУЧЕНИЕ СУТИ И ОТПРАВКА =====
async def issue_received(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    issue = update.message.text.strip()
    user_data = context.user_data
    category = user_data.get("category", "Не указана")
    nick = user_data.get("nick", "Не указан")
    user_id = update.effective_user.id

    success = await send_to_group(user_id, category, nick, issue, context)
    if success:
        await update.message.reply_text(
            "✅ Ваша заявка успешно создана! Ожидайте ответа администрации.\n"
            "Можете создать новую заявку, выбрав категорию.",
            reply_markup=get_main_menu()
        )
    else:
        await update.message.reply_text(
            "❌ Произошла ошибка при отправке заявки. Попробуйте позже или свяжитесь с администрацией напрямую.",
            reply_markup=get_main_menu()
        )
    user_data.clear()
    return ConversationHandler.END

# ===== ОТВЕТ ИЗ ГРУППЫ =====
async def group_reply(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    if update.effective_chat.id != GROUP_CHAT_ID:
        return
    msg = update.message
    if not msg.reply_to_message or msg.reply_to_message.from_user.id != context.bot.id:
        return

    original_msg_id = msg.reply_to_message.message_id
    user_id = reply_mapping.get((GROUP_CHAT_ID, original_msg_id))
    if not user_id:
        logger.warning(f"Не найден user_id для сообщения {original_msg_id}")
        return

    try:
        await context.bot.copy_message(chat_id=user_id, from_chat_id=GROUP_CHAT_ID, message_id=msg.message_id)
        await msg.reply_text("✅ Ответ отправлен пользователю.", quote=False)
    except Exception as e:
        logger.error(f"Ошибка пересылки ответа: {e}")
        await msg.reply_text(f"❌ Ошибка: {e}", quote=False)

# ===== ПРОЧИЕ СООБЩЕНИЯ =====
async def unknown_text(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    await update.message.reply_text("Используйте кнопки меню для создания обращения.", reply_markup=get_main_menu())

# ===== ЗАПУСК =====
def main():
    app = Application.builder().token(TOKEN).build()

    # ConversationHandler с входом по тексту кнопок (точное совпадение)
    conv_handler = ConversationHandler(
        entry_points=[
            MessageHandler(filters.Text(CATEGORY_BUTTONS), category_button)
        ],
        states={
            WAITING_NICK: [MessageHandler(filters.TEXT & ~filters.COMMAND, nick_received)],
            WAITING_ISSUE: [MessageHandler(filters.TEXT & ~filters.COMMAND, issue_received)],
        },
        fallbacks=[],
        allow_reentry=True,
    )

    app.add_handler(CommandHandler("start", start))
    app.add_handler(conv_handler)
    # Ответы в группе
    app.add_handler(MessageHandler(filters.REPLY & filters.Chat(GROUP_CHAT_ID), group_reply))
    # Неизвестные команды и обычные сообщения
    app.add_handler(MessageHandler(filters.COMMAND, lambda u, c: u.message.reply_text("Неизвестная команда. Введите /start")))
    app.add_handler(MessageHandler(filters.TEXT, unknown_text))

    logger.info("🤖 Бот запущен.")
    app.run_polling(allowed_updates=Update.ALL_TYPES)

if __name__ == "__main__":
    main()
