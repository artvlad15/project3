# Импортируем необходимые классы.
import logging

from telegram import (
    ReplyKeyboardRemove,
    Update,
    ReplyKeyboardMarkup
)

from telegram.constants import ParseMode
from telegram.ext import (
    Application,
    CommandHandler,
    ContextTypes,
)

logging.basicConfig(
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s", level=logging.DEBUG
)
# set higher logging level for httpx to avoid all GET and POST requests being logged
logging.getLogger("httpx").setLevel(logging.WARNING)

logger = logging.getLogger(__name__)

TOTAL_VOTER_COUNT = 3


async def start(update, context):
    reply_keyboard = [['/seek_artists', '/seek_music', '/get_genres']]
    markup = ReplyKeyboardMarkup(reply_keyboard, one_time_keyboard=False)
    await update.message.reply_text("Здравствуйте! Меня зовут Клайв Клауд, я - меломан, эксперт в музыке, гений, "
                                    "миллиардер, плейбой и филантроп."
                                    " Я помогу вам разыскать исполнителей и музыку по вашим вкусам! "
                                    "Выберите команду, и давайте приступим!", reply_markup=markup)


async def close_keyboard(update, context):
    await update.message.reply_text(
        "Ok",
        reply_markup=ReplyKeyboardRemove()
    )


async def get_genres(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    questions = ["Rock", "Metal", "Pop", "Classical"]
    message = await context.bot.send_poll(
        update.effective_chat.id,
        "Выберите жанры, sorry not sorry for english da",
        questions,
        is_anonymous=False,
        allows_multiple_answers=True,
    )
    # Save some info about the poll the bot_data for later use in receive_poll_answer
    payload = {
        message.poll.id: {
            "questions": questions,
            "message_id": message.message_id,
            "chat_id": update.effective_chat.id,
            "answers": 0,
        }
    }
    context.bot_data.update(payload)


async def receive_poll_answer(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Summarize a users poll vote"""
    answer = update.poll_answer
    answered_poll = context.bot_data[answer.poll_id]
    try:
        questions = answered_poll["questions"]
    # this means this poll answer update is from an old poll, we can't do our answering then
    except KeyError:
        return
    selected_options = answer.option_ids
    res = []
    for question_id in selected_options:
        res.append(questions[question_id])
    await context.bot.send_message(
        answered_poll["chat_id"],
        f"{res}",
        parse_mode=ParseMode.HTML,
    )
    answered_poll["answers"] += 1
    # Close poll after three participants voted
    if answered_poll["answers"] == TOTAL_VOTER_COUNT:
        await context.bot.stop_poll(answered_poll["chat_id"], answered_poll["message_id"])


async def receive_poll(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """On receiving polls, reply to it by a closed poll copying the received poll"""
    actual_poll = update.effective_message.poll
    # Only need to set the question and options, since all other parameters don't matter for
    # a closed poll
    await update.effective_message.reply_poll(
        question=actual_poll.question,
        options=[o.text for o in actual_poll.options],
        # with is_closed true, the poll/quiz is immediately closed
        is_closed=True,
        reply_markup=ReplyKeyboardRemove(),
    )


async def seek_artist(update, context):
    await update.message.reply_text("Ой! Как пусто...")


async def seek_music(update, context):
    await update.message.reply_text("Ой! Как пусто...")


def main():
    # Создаём объект Application.
    # Вместо слова "TOKEN" надо разместить полученный от @BotFather токен
    application = Application.builder().token("7049722450:AAF7pOMtuMf6jyZrM73cWsxdCrVn28COqt8").build()

    # Создаём обработчик сообщений типа filters.TEXT
    # из описанной выше асинхронной функции echo()
    # После регистрации обработчика в приложении
    # эта асинхронная функция будет вызываться при получении сообщения
    # с типом "текст", т. е. текстовых сообщений.

    # Регистрируем обработчик в приложении.
    application.add_handler(CommandHandler("seek_artists", seek_artist))
    application.add_handler(CommandHandler("seek_music", seek_music))
    application.add_handler(CommandHandler("close", close_keyboard))
    application.add_handler(CommandHandler("get_genres", get_genres))
    application.add_handler(CommandHandler("start", start))

    # Запускаем приложение.
    application.run_polling()


# Запускаем функцию main() в случае запуска скрипта.
if __name__ == '__main__':
    main()
