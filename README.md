# telegram-bot
Telegram bot for book recommendations using PostgreSQL and Python programming, developed with the Telebot library.
import logging
import psycopg2
import telebot
from telebot import types
# =========================
# 1. Настройки и подключение к библиотекам
# =========================
TOKEN = '7807458780:AAGANyypHuFMebYWXeRlMv3HEpDV2rYn80Y'
bot = telebot.TeleBot(TOKEN)

# =========================
# 2. Подключение к базе данных PostgreSQL
# =========================
try:
    conn = psycopg2.connect(
        dbname="library_bot",
        user="amina",
        password="_auMW{\zp.,+wS<=_4k7uNgE)gr_Jp",
        host="localhost",
        port="5432"
    )
    cursor = conn.cursor()
    print('Connected to the PostgreSQL database')
except Exception as e:
    print(f"Database connection error: {e}")
    exit()

# =========================
# 3. Логирование
# =========================
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Хранение данных о текущем состоянии пользователя
user_data = {}

# =========================
# 4. Команды и функции
# =========================
# Приветственное сообщение
@bot.message_handler(commands=['start'])
def cmd_start(message):
    # Разметка клавиатуры
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add(types.KeyboardButton('📚 Рекомендации по жанрам'), types.KeyboardButton('⭐ Рандомная книга'))
    markup.add(types.KeyboardButton('🔍 Найти книгу по автору'), types.KeyboardButton('❓ Помощь'))

    # Отправка изображения и текстового сообщения
    bot.send_photo(
        message.chat.id,
        open(r"C:\Users\aktae\OneDrive\Рабочий стол\image.jpg", 'rb'),  # Используем правильный путь
        caption="Привет! Я — Библиотечный помощник 📚. Я помогу тебе выбрать книгу для чтения. "
                "Для начала выбери одну из опций:",
        reply_markup=markup
    )


#-------------------------------
@bot.message_handler(commands=['info'])
def send_info(message):
    bot.send_message(
        message.chat.id,
        "Привет! Я бот для поиска книг. Вот что я умею:\n"
        "- *Рандомная книга* - Я могу порекомендовать случайную книгу.\n"
        "- *Найти книгу по автору* - Могу помочь найти книгу по автору из моей базы.\n"
        "Просто выбери одну из команд и наслаждайся!\n\n"
        "Автор бота: Ваше имя или ник\n"
        "Если возникнут вопросы, просто напишите мне!"
    )




@bot.message_handler(commands=['randombook'])
def random_book(message):
    cursor.execute("SELECT title, author, description, genre FROM books ORDER BY RANDOM() LIMIT 1")
    book = cursor.fetchone()

    if book:
        bot.send_message(
            message.chat.id,
            f"✨ *Рандомная книга* ✨\n\n"
            f"📝 **Название:** *{book[0]}*\n"
            f"✍️ **Автор:** _{book[1]}_\n"
            f"📖 **Описание:**\n_{book[2]}_\n"
            f"🎭 **Жанр:** _{book[3]}_\n\n"
            "📚 _Надеемся, что вам понравится эта книга!_\n\n"
            "Желаем приятного чтения! 😄"
            , parse_mode="Markdown"
        )
    else:
        bot.send_message(message.chat.id, "Не удалось найти случайную книгу. 😞")





# Обработчик команды /help
@bot.message_handler(commands=['help'])
def cmd_help(message):
    help_text = (
        "Вот несколько команд, которые я могу предложить:\n"
        "/help — получить список команд.\n"
        "/start — начать общение с ботом.\n"
        "/info — информация о боте.\n"
        "/randombook — получить случайную книгу.\n"
        "/findbook — найти книгу по автору.\n"
    )
    bot.send_message(message.chat.id, help_text)






@bot.message_handler(commands=['findbook'])
def search_book_by_author(message):
    bot.send_message(message.chat.id, "Выберите автора из списка ниже:")

    # Получаем список авторов из базы данных
    cursor.execute("SELECT DISTINCT author FROM books")
    authors = cursor.fetchall()
    print(f"Авторы: {authors}")  # Для отладки

    # Создаем инлайн кнопки для авторов
    markup = types.InlineKeyboardMarkup()

    for author in authors:
        author_name = author[0]
        markup.add(
            types.InlineKeyboardButton(author_name, callback_data=f"author_{author_name}"))  # Передаем имя автора в callback

    bot.send_message(message.chat.id, "Выберите автора:", reply_markup=markup)

# Обработка выбора жанра (инлайн-кнопки)
@bot.message_handler(func=lambda message: message.text == '📚 Рекомендации по жанрам')
def recommend_genre(message):
    markup = types.InlineKeyboardMarkup()
    cursor.execute("SELECT DISTINCT genre FROM books")
    genres = cursor.fetchall()

    # Добавление инлайн-кнопок для каждого жанра
    for genre in genres:
        genre_name = genre[0]
        markup.add(types.InlineKeyboardButton(genre_name, callback_data=genre_name))

    bot.send_message(message.chat.id, "Выбери жанр, который тебе интересен:", reply_markup=markup)

# Обработка нажатия на кнопку с автором
@bot.callback_query_handler(func=lambda call: call.data.startswith('author_'))
def send_books_by_author(call):
    author_name = call.data.replace('author_', '')  # Извлекаем имя автора

    # Получаем книги по автору без пагинации
    books = get_books_by_author(author_name)

    if books:
        response = f"🔍 Книги автора '{author_name}':\n\n"
        for book in books:
            response += f"📚 *{book[0]}*:\n{book[1]}\n\n"
        bot.send_message(call.message.chat.id, response, parse_mode="Markdown")
    else:
        bot.send_message(call.message.chat.id, f"Книги автора '{author_name}' не найдены. 😞")

    # Кнопки для возвращения в главное меню
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add(types.KeyboardButton('🏠 Главное меню'))
    bot.send_message(call.message.chat.id, "Что хотите сделать дальше?", reply_markup=markup)


# Обработка выбора жанра из инлайн-кнопок
@bot.callback_query_handler(func=lambda call: call.data != 'main_menu' and call.data != 'choose_genre')
def send_books_by_genre(call):
    genre = call.data
    cursor.execute("SELECT title, author, description FROM books WHERE genre = %s", (genre,))
    books = cursor.fetchall()

    if books:
        user_data[call.message.chat.id] = {
            'genre': genre,
            'books': books,
            'current_index': 0  # Начальный индекс книги
        }

        # Отправляем первую книгу с улучшенным форматированием (Markdown)
        book = books[0]
        bot.send_message(
            call.message.chat.id,
            f"✨ *Рекомендуемая книга в жанре* _{genre}_ ✨\n\n"
            f"📝 **Название:** *{book[0]}*\n"
            f"✍️ **Автор:** _{book[1]}_\n"
            f"📖 **Описание:**\n_{book[2]}_\n\n"
            "📚 _Надеемся, что вам понравится эта книга!_\n\n"
            "Желаем приятного чтения! 😄"
            , parse_mode="Markdown"
        )
    else:
        bot.send_message(call.message.chat.id, f"В жанре *{genre}* пока нет книг. 😞", parse_mode="Markdown")

    # Кнопки для следующего действия
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add(types.KeyboardButton('➡️ Следующая книга'))
    markup.add(types.KeyboardButton('🔄 Вернуться к жанрам'))
    markup.add(types.KeyboardButton('🏠 Главное меню'))

    bot.send_message(call.message.chat.id, "Что хотите сделать дальше?", reply_markup=markup)


# Обработка кнопки "Следующая книга"
@bot.message_handler(func=lambda message: message.text == '➡️ Следующая книга')
def next_book(message):
    user_info = user_data.get(message.chat.id)
    if user_info:
        current_index = user_info['current_index']
        books = user_info['books']

        # Проверяем, есть ли еще книги
        if current_index + 1 < len(books):
            user_info['current_index'] += 1
            book = books[user_info['current_index']]
            bot.send_message(message.chat.id, f"Следующая книга 📖:\n\n{book[0]} — {book[1]}: {book[2]}")
        else:
            bot.send_message(message.chat.id, "Это последняя книга в жанре. 😅")

        # Вложенные кнопки для следующего действия
        markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
        markup.add(types.KeyboardButton('➡️ Следующая книга'))
        markup.add(types.KeyboardButton('🔄 Вернуться к жанрам'))
        markup.add(types.KeyboardButton('🏠 Главное меню'))

        bot.send_message(message.chat.id, "Что хотите сделать дальше?", reply_markup=markup)
    else:
        bot.send_message(message.chat.id, "Ошибка: Вы не выбрали жанр. ⚠️")


# Обработка возврата к выбору жанра
@bot.message_handler(func=lambda message: message.text == '🔄 Вернуться к жанрам')
def back_to_genre_selection(message):
    recommend_genre(message)


# Обработка возврата в главное меню
@bot.message_handler(func=lambda message: message.text == '🏠 Главное меню')
def back_to_main_menu(message):
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add(types.KeyboardButton('📚 Рекомендации по жанрам'))
    markup.add(types.KeyboardButton('⭐ Рандомная книга'))
    markup.add(types.KeyboardButton('🔍 Найти книгу по автору'))
    markup.add(types.KeyboardButton('❓ Помощь'))

    bot.send_message(message.chat.id, "Вы вернулись в главное меню. 😊", reply_markup=markup)








# Функция для получения книг по автору (без пагинации)
def get_books_by_author(author_name):
    cursor.execute("SELECT title, description FROM books WHERE author = %s", (author_name,))
    return cursor.fetchall()


# Обработка кнопки "Найти книгу по автору"
@bot.message_handler(func=lambda message: message.text == '🔍 Найти книгу по автору')
def search_book_by_author(message):
    bot.send_message(message.chat.id, "Выберите автора из списка ниже:")

    # Получаем список авторов из базы данных
    cursor.execute("SELECT DISTINCT author FROM books")
    authors = cursor.fetchall()
    print(f"Авторы: {authors}")  # Добавьте это для отладки

    # Создаем инлайн кнопки для авторов
    markup = types.InlineKeyboardMarkup()

    for author in authors:
        author_name = author[0]
        markup.add(
            types.InlineKeyboardButton(author_name, callback_data=f"author_{author_name}"))  # Убрали пагинацию

    bot.send_message(message.chat.id, "Выберите автора:", reply_markup=markup)


#-------------------------------------------------------------------------

@bot.message_handler(func=lambda message: message.text == '❓ Помощь')
def show_help(message):
    help_text = (
        "Привет! Я — Библиотечный помощник 📚. Вот как ты можешь меня использовать:\n\n"
        "📚 Рекомендации по жанрам — выбери жанр, и я предложу книги в этом жанре.\n"
        "⭐ Популярные книги — увидишь список самых популярных книг.\n"
        "🔍 Найти книгу по автору — введи имя автора, и я покажу все его книги.\n\n"
        "Если хочешь вернуться в главное меню, просто нажми кнопку '🏠 Главное меню'.\n\n"
        "Если книги не найдены или не могу отобразить все данные, это может означать, что база данных пуста или произошла ошибка."
    )
    bot.send_message(message.chat.id, help_text)

    # Кнопки для возврата в главное меню и дополнительных команд
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add(types.KeyboardButton('🔧 Дополнительные команды'))
    markup.add(types.KeyboardButton('🔙 Назад'))
    bot.send_message(message.chat.id, "Что бы вы хотели сделать дальше?", reply_markup=markup)

#----------------------------------------------------------
# Обработка кнопки "⭐ Рандомная книга"
@bot.message_handler(func=lambda message: message.text == '⭐ Рандомная книга')
def random_book(message):
    cursor.execute("SELECT title, author, description, genre FROM books ORDER BY RANDOM() LIMIT 1")
    book = cursor.fetchone()

    if book:
        bot.send_message(
            message.chat.id,
            f"✨ *Рандомная книга* ✨\n\n"
            f"📝 **Название:** *{book[0]}*\n"
            f"✍️ **Автор:** _{book[1]}_\n"
            f"📖 **Описание:**\n_{book[2]}_\n"
            f"🎭 **Жанр:** _{book[3]}_\n\n"
            "📚 _Надеемся, что вам понравится эта книга!_\n\n"
            "Желаем приятного чтения! 😄"
            , parse_mode="Markdown"
        )
    else:
        bot.send_message(message.chat.id, "Не удалось найти случайную книгу. 😞")


# Обработка кнопки "🔧 Дополнительные команды"
@bot.message_handler(func=lambda message: message.text == '🔧 Дополнительные команды')
def show_additional_commands(message):
    commands_text = (
        "Вот несколько команд, которые я могу предложить:\n\n"
        "/help — получить список команд.\n"
        "/start — начать общение с ботом.\n"
        "/info — информация о боте.\n"
        "/randombook — получить случайную книгу.\n"
        "/findbook — найти книгу по автору.\n"
    )
    bot.send_message(message.chat.id, commands_text)

    # Кнопки для возврата в главное меню или назад
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add(types.KeyboardButton('🏠 Главное меню'))
    bot.send_message(message.chat.id, "Что бы вы хотели сделать дальше?", reply_markup=markup)

# Обработка кнопки "🔙 Назад"
@bot.message_handler(func=lambda message: message.text == '🔙 Назад')
def back_to_previous_menu(message):
    # Возвращаем пользователя к основным опциям меню
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add(types.KeyboardButton('📚 Рекомендации по жанрам'), types.KeyboardButton('⭐ Рандомная книга'))
    markup.add(types.KeyboardButton('🔍 Найти книгу по автору'), types.KeyboardButton('❓ Помощь'))
    bot.send_message(message.chat.id, "Вы вернулись в главное меню. 😊", reply_markup=markup)


# Обработка неизвестных команд
@bot.message_handler(func=lambda message: True)
def unknown_command(message):
    bot.send_message(message.chat.id,
                     "Извините, я не понял вашу команду. Пожалуйста, выберите одну из доступных опций. 😊")

    # Отправляем пользователю основные кнопки для выбора
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add(types.KeyboardButton('📚 Рекомендации по жанрам'), types.KeyboardButton('⭐ Рандомная книга'))
    markup.add(types.KeyboardButton('🔍 Найти книгу по автору'), types.KeyboardButton('❓ Помощь'))
    bot.send_message(message.chat.id, "Выберите одну из опций:", reply_markup=markup)


@bot.message_handler(func=lambda message: True)
def handle_unknown(message):
    bot.send_message(message.chat.id, "Извините, я не понимаю эту команду. Попробуйте снова.")

# =========================
# 6. Запуск бота
# =========================
if __name__ == '__main__':
    bot.polling(none_stop=True)
