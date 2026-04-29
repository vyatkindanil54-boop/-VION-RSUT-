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

# ===== НАСТРОЙКИ =====
logging.basicConfig(
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s", level=logging.INFO
)
logger = logging.getLogger(__name__)

TOKEN = os.environ.get("TELEGRAM_BOT_TOKEN", "8783547450:AAF5k94P1Q6Jv7gK8b5gfMGv5fMZQTy3yIQ")
GROUP_CHAT_ID = int(os.environ.get("GROUP_CHAT_ID", -1003906994235))

# ===== КНОПКИ МЕНЮ =====
MENU_BUTTONS = [
    ["📢 Жалоба", "⚠️ Проблема"],
    ["❓ Вопрос", "🎓 Апелляция"],
    ["🛒 Магазин"]
]
# Текст кнопок для фильтра (уберем emoji, но можно и с ними)
CATEGORY_TEXTS = ["Жалоба", "Проблема", "Вопрос", "Апелляция", "Магазин"]
# Сопоставление текста кнопки и названия категории (убираем emoji)
BUTTON_TO_CATEGORY = {
    "📢 Жалоба": "Жалоба",
    "⚠️ Проблема": "Проблема",
    "❓ Вопрос": "Вопрос",
    "🎓 Апелляция": "Апелляция",
    "🛒 Магазин": "Магазин"
}

# ===== КЛАВИАТУРА =====
def get_main_menu() -> ReplyKeyboardMarkup:
    """Возвращает клавиатуру главного меню."""
    return ReplyKeyboardMarkup(MENU_BUTTONS, resize_keyboard=True, one_time_keyboard=False)

# ===== СОСТОЯНИЯ РАЗГОВОРА =====
WAITING_NICK = 1
WAITING_ISSUE = 2

# ===== СВЯЗЬ СООБЩЕНИЕ В ГРУППЕ -> USER_ID =====
reply_mapping = {}

# ===== ОТПРАВКА ЗАЯВКИ В ГРУППУ =====
async def send_to_group(user_id: int, category: str, nick: str, issue: str, context: ContextTypes.DEFAULT_TYPE):
    text = (
        f"📩 <b>Новая заявка</b>\n"
        f"<b>Категория:</b> {category}\n"
        f"<b>Ник:</b> {nick}\n"
        f"<b>User ID:</b> <code>{user_id}</code>\n"
        f"<b>Суть вопроса:</b>\n{issue}"
    )
    msg = await context.bot.send_message(chat_id=GROUP_CHAT_ID, text=text, parse_mode="HTML")
    reply_mapping[(GROUP_CHAT_ID, msg.message_id)] = user_id
    logger.info(f"Заявка от {user_id} отправлена в группу (msg_id={msg.message_id})")

# ===== /start =====
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    """Приветствие и показ главного меню."""
    await update.message.reply_text(
        "👋 Привет! Я бот для обращений в поддержку.\n"
        "Выберите категорию, нажав на кнопку ниже, и следуйте инструкциям.",
        reply_markup=get_main_menu()
    )
    return ConversationHandler.END

# ===== ОБРАБОТЧИК НАЖАТИЯ КНОПКИ КАТЕГОРИИ (вход в диалог) =====
async def category_button_handler(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    """Сохраняет выбранную категорию и запрашивает ник."""
    text = update.message.text.strip()
    category = BUTTON_TO_CATEGORY.get(text, text)  # если вдруг текст без emoji
    context.user_data["category"] = category
    # Убираем основную клавиатуру на время ввода (можно и оставить, но лучше скрыть)
    await update.message.reply_text(
        f"📌 Категория: <b>{category}</b>\n\nВведите ваш ник (или @username):",
        parse_mode="HTML",
        reply_markup=ReplyKeyboardMarkup([[]], resize_keyboard=True)  # скрыть клавиатуру
    )
    return WAITING_NICK

# ===== ПОЛУЧЕНИЕ НИКА =====
async def nick_received(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    nick = update.message.text.strip()
    context.user_data["nick"] = nick
    await update.message.reply_text("Теперь подробно опишите суть вашего вопроса или проблемы:")
    return WAITING_ISSUE

# ===== ПОЛУЧЕНИЕ СУТИ -> ОТПРАВКА =====
async def issue_received(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    issue = update.message.text.strip()
    user_data = context.user_data
    category = user_data.get("category", "Не указано")
    nick = user_data.get("nick", "Не указан")
    user_id = update.effective_user.id

    await send_to_group(user_id, category, nick, issue, context)
    # Возвращаем главное меню
    await update.message.reply_text(
        "✅ Ваша заявка принята! Ожидайте ответа от администрации.\n"
        "Вы можете создать новую заявку, выбрав категорию ниже.",
        reply_markup=get_main_menu()
    )
    user_data.clear()
    return ConversationHandler.END

# ===== ОБРАБОТКА ОТВЕТОВ В ГРУППЕ =====
async def group_reply(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    if update.effective_chat.id != GROUP_CHAT_ID:
        return

    message = update.message
    if not message.reply_to_message:
        return

    if message.reply_to_message.from_user.id != context.bot.id:
        return

    original_msg_id = message.reply_to_message.message_id
    user_id = reply_mapping.get((GROUP_CHAT_ID, original_msg_id))
    if not user_id:
        logger.warning(f"Не найден user_id для сообщения {original_msg_id}")
        return

    try:
        await context.bot.copy_message(
            chat_id=user_id,
            from_chat_id=GROUP_CHAT_ID,
            message_id=message.message_id,
        )
        await message.reply_text("✅ Ответ отправлен пользователю.", quote=False)
    except Exception as e:
        logger.error(f"Ошибка при отправке ответа пользователю {user_id}: {e}")
        await message.reply_text(f"❌ Ошибка: {e}", quote=False)

# ===== НЕИЗВЕСТНЫЕ КОМАНДЫ =====
async def unknown_command(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    await update.message.reply_text(
        "Неизвестная команда. Воспользуйтесь кнопками меню или введите /start.",
        reply_markup=get_main_menu()
    )

# ===== ЗАПУСК =====
def main():
    app = Application.builder().token(TOKEN).build()

    # ConversationHandler: вход по нажатию кнопки категории
    conv_handler = ConversationHandler(
        entry_points=[
            MessageHandler(
                filters.TEXT & filters.Regex(
                    r'^(📢 Жалоба|⚠️ Проблема|❓ Вопрос|🎓 Апелляция|🛒 Магазин)$'
                ),
                category_button_handler
            )
        ],
        states={
            WAITING_NICK: [MessageHandler(filters.TEXT & ~filters.COMMAND, nick_received)],
            WAITING_ISSUE: [MessageHandler(filters.TEXT & ~filters.COMMAND, issue_received)],
        },
        fallbacks=[],
        allow_reentry=True  # чтобы можно было начать новую заявку из любого состояния
    )

    app.add_handler(CommandHandler("start", start))
    app.add_handler(conv_handler)
    # Ответы в группе
    app.add_handler(MessageHandler(filters.REPLY & filters.Chat(GROUP_CHAT_ID), group_reply))
    # Неизвестные команды
    app.add_handler(MessageHandler(filters.COMMAND, unknown_command))
    # Любой другой текст – можно показать меню
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, lambda u, c: u.message.reply_text(
        "Используйте кнопки меню для создания обращения.", reply_markup=get_main_menu()
    )))

    logger.info("🤖 Бот запущен с меню-кнопками.")
    app.run_polling(allowed_updates=Update.ALL_TYPES)

if __name__ == "__main__":
    main()
