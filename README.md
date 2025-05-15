# avava
import asyncio
import logging
import random
from datetime import datetime, timedelta
from aiogram import Bot, Dispatcher, types, F
from aiogram.types import InlineKeyboardButton, InlineKeyboardMarkup
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.filters import Command
from enum import Enum
from aiogram.types import FSInputFile, InputMediaPhoto

# Импорт конфигурации
import config

SECRET_CODE = "Radmir"
SECRET_CODE1 = "Admin999"

# Настройка логирования
logging.basicConfig(level=logging.INFO)

# Инициализация бота и диспетчера
bot = Bot(token=config.TOKEN)
storage = MemoryStorage()
dp = Dispatcher(storage=storage)

# Константы для рулетки
ROULETTE_NUMBERS = list(range(37))  # 0-36
RED_NUMBERS = [1, 3, 5, 7, 9, 12, 14, 16, 18, 19, 21, 23, 25, 27, 30, 32, 34, 36]
BLACK_NUMBERS = [2, 4, 6, 8, 10, 11, 13, 15, 17, 20, 22, 24, 26, 28, 29, 31, 33, 35]
SECTORS = {
    'first': list(range(1, 13)),  # 1-12
    'second': list(range(13, 25)),  # 13-24
    'third': list(range(25, 37)),  # 25-36
    'first_col': list(range(1, 37, 3)),  # 1,4,7...34
    'second_col': list(range(2, 37, 3)),  # 2,5,8...35
    'third_col': list(range(3, 37, 3))  # 3,6,9...36
}

# Определение состояний бота
class BotState(Enum):
    MENU = 'menu'
    GUESS_NUMBER = 'guess_number'
    RPS = 'rps'
    CUPS = 'cups'
    TRANSFER = 'transfer'
    TRANSFER_AMOUNT = 'transfer_amount'
    ROULETTE = 'roulette'
    TAPPER = 'tapper'

# Класс для рулетки
class RouletteGame:
    def __init__(self):
        self.bets = {}  # {user_id: [(bet_type, amount, value)]}
        self.is_active = False
        self.timer_message = None
        self.end_time = None

# Класс для хранения данных пользователя
class GameData:
    def __init__(self):
        self.state = BotState.MENU
        self.user_balances = {}
        self.secret_number = None
        self.awaiting_bet_and_guess = False
        self.awaiting_rps_bet = False
        self.current_rps_bet = 0
        self.cups_bet = 0
        self.correct_cup = None
        self.awaiting_cups_bet = False
        self.transfer_to_id = None
        self.awaiting_transfer_amount = False
        self.used_secret_code = False
        self.taps = 0  # Для тапалки
        self.roulette_games = {}  # {chat_id: RouletteGame()}
        self.last_tap_time = datetime.now()  # Для ограничения тапов

# Словарь для хранения данных пользователей
user_data = {}

def get_user_data(user_id):
    if user_id not in user_data:
        user_data[user_id] = GameData()
    return user_data[user_id]

def main_menu_keyboard():
    keyboard = InlineKeyboardMarkup(inline_keyboard=[
        [
            InlineKeyboardButton(text="🎮 Мини-игры", callback_data='mini_games'),
            InlineKeyboardButton(text="🎲 Игры", callback_data='games')
        ],
        [InlineKeyboardButton(text="👆 ТАПАЛКА 👆", callback_data='tapper')],
        [
            InlineKeyboardButton(text="👤 Профиль", callback_data='profile'),
            InlineKeyboardButton(text="ℹ️ Инфо", callback_data='info')
        ]
    ])
    return keyboard

def games_menu_keyboard():
    keyboard = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🎲 Угадай число", callback_data='guess_number')],
        [InlineKeyboardButton(text="✌️ Камень-ножницы-бумага", callback_data='rps')],
        [InlineKeyboardButton(text="🥤 Стаканчики", callback_data='cups')],
        [InlineKeyboardButton(text="🎰 Рулетка", callback_data='roulette')],
        [InlineKeyboardButton(text="🔙 Вернуться в меню", callback_data='back_to_menu')]
    ])
    return keyboard

def mini_games_keyboard():
    keyboard = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🎲 Кости", callback_data='dice')],
        [InlineKeyboardButton(text="🎯 Дартс", callback_data='darts')],
        [InlineKeyboardButton(text="⚽️ Пенальти", callback_data='football')],
        [InlineKeyboardButton(text="🔙 Вернуться в меню", callback_data='back_to_menu')]
    ])
    return keyboard

def roulette_keyboard():
    keyboard = InlineKeyboardMarkup(inline_keyboard=[
        [
            InlineKeyboardButton(text="🔴 Красное x2", callback_data='bet_red'),
            InlineKeyboardButton(text="⚫️ Черное x2", callback_data='bet_black')
        ],
        [InlineKeyboardButton(text="🔢 Выбрать число x36", callback_data='bet_number')],
        [
            InlineKeyboardButton(text="1️⃣ 1-12", callback_data='bet_first'),
            InlineKeyboardButton(text="2️⃣ 13-24", callback_data='bet_second'),
            InlineKeyboardButton(text="3️⃣ 25-36", callback_data='bet_third')
        ],
        [
            InlineKeyboardButton(text="1️⃣ Первая колонка", callback_data='bet_first_col'),
            InlineKeyboardButton(text="2️⃣ Вторая колонка", callback_data='bet_second_col'),
            InlineKeyboardButton(text="3️⃣ Третья колонка", callback_data='bet_third_col')
        ]
    ])
    return keyboard


@dp.message(Command('start'))
async def cmd_start(message: types.Message):
    user_id = message.from_user.id
    data = get_user_data(user_id)
    data.state = BotState.MENU
    if user_id not in data.user_balances:
        data.user_balances[user_id] = 1000
    await message.answer(
        f"👋 Добро пожаловать в игровой бот!\n\n"
        f"💰 Ваш текущий баланс: {data.user_balances[user_id]}\n\n"
        f"Выберите раздел:",
        reply_markup=main_menu_keyboard()
    )

@dp.callback_query(F.data.in_(['mini_games', 'games', 'tapper', 'profile', 'info', 'back_to_menu']))
async def process_main_menu(callback_query: types.CallbackQuery):
    user_id = callback_query.from_user.id
    data = get_user_data(user_id)
    
    if callback_query.data == 'back_to_menu':
        data.state = BotState.MENU
        await callback_query.message.edit_text(
            f"Главное меню\n💰 Ваш баланс: {data.user_balances.get(user_id, 1000)}",
            reply_markup=main_menu_keyboard()
        )
    
    elif callback_query.data == 'mini_games':
        await callback_query.message.edit_text(
            "🎮 Мини-игры:\n"
            "Выберите игру для быстрого развлечения!",
            reply_markup=mini_games_keyboard()
        )
    
    elif callback_query.data == 'games':
        await callback_query.message.edit_text(
            "🎲 Игры на деньги:\n"
            "Выберите игру для ставок!",
            reply_markup=games_menu_keyboard()
        )
    
    elif callback_query.data == 'tapper':
        data.state = BotState.TAPPER
        await callback_query.message.edit_text(
            f"👆 Тапалка\n\n"
            f"Ваш текущий счет: {data.taps}\n"
            f"Нажимайте на кнопку, чтобы увеличить счет!\n"
            f"Каждые 100 тапов = +10 монет",
            reply_markup=InlineKeyboardMarkup(inline_keyboard=[
                [InlineKeyboardButton(text="👆 ТАП! 👆", callback_data='tap')],
                [InlineKeyboardButton(text="🔙 В меню", callback_data='back_to_menu')]
            ])
        )
    
    elif callback_query.data == 'profile':
        total_games = len([g for g in data.roulette_games.values() if g.is_active])
        await callback_query.message.edit_text(
            f"👤 Профиль игрока\n\n"
            f"ID: {user_id}\n"
            f"Баланс: {data.user_balances.get(user_id, 1000):,} 💰\n"
            f"Тапов: {data.taps} 👆\n"
            f"Активных игр: {total_games}\n\n"
            f"Используйте /top чтобы увидеть таблицу лидеров!",
            reply_markup=InlineKeyboardMarkup(inline_keyboard=[
                [InlineKeyboardButton(text="🔙 В меню", callback_data='back_to_menu')]
            ])
        )
    
    elif callback_query.data == 'info':
        await callback_query.message.edit_text(
            "ℹ️ Информация о боте\n\n"
            "🎮 Игровой бот с множеством развлечений!\n\n"
            "Команды:\n"
            "/start - Начать игру\n"
            "/balance - Проверить баланс\n"
            "/top - Таблица лидеров\n"
            "/pay - Перевести деньги\n"
            "/help - Помощь\n\n"
            "По всем вопросам: @admin",
            reply_markup=InlineKeyboardMarkup(inline_keyboard=[
                [InlineKeyboardButton(text="🔙 В меню", callback_data='back_to_menu')]
            ])
        )

@dp.callback_query(F.data == 'tap')
async def process_tap(callback_query: types.CallbackQuery):
    user_id = callback_query.from_user.id
    data = get_user_data(user_id)
    
    # Проверяем время последнего тапа (антиспам)
    now = datetime.now()
    if (now - data.last_tap_time).total_seconds() < 0.5:
        await callback_query.answer("Не так быстро!")
        return
    
    data.last_tap_time = now
    data.taps += 1
    
    # Каждые 100 тапов даем награду
    if data.taps % 100 == 0:
        data.user_balances[user_id] = data.user_balances.get(user_id, 0) + 10
        reward_text = "\n🎉 +10 монет за 100 тапов!"
    else:
        reward_text = ""
    
    await callback_query.message.edit_text(
        f"👆 Тапалка\n\n"
        f"Ваш текущий счет: {data.taps}{reward_text}\n"
        f"Нажимайте на кнопку, чтобы увеличить счет!\n"
        f"Каждые 100 тапов = +10 монет",
        reply_markup=InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="👆 ТАП! 👆", callback_data='tap')],
            [InlineKeyboardButton(text="🔙 В меню", callback_data='back_to_menu')]
        ])
    )
    await callback_query.answer()


@dp.callback_query(F.data == 'roulette')
async def start_roulette(callback_query: types.CallbackQuery):
    chat_id = callback_query.message.chat.id
    user_id = callback_query.from_user.id
    data = get_user_data(user_id)
    
    if chat_id not in data.roulette_games:
        data.roulette_games[chat_id] = RouletteGame()
    
    game = data.roulette_games[chat_id]
    
    if game.is_active:
        await callback_query.answer("Игра уже идет!")
        return
    
    game.is_active = True
    game.end_time = datetime.now() + timedelta(seconds=30)
    
    # Здесь будет ваша ссылка на фото стола
    casino_table = "https://custom-images.strikinglycdn.com/res/hrscywv4p/image/upload/c_limit,fl_lossy,h_9000,w_1200,f_auto,q_auto/12667044/H4QPXJDuAscVZZFBzHDhtj1iUW.png"  # Замените на вашу ссылку
    
    # Отправляем сообщение с фото и кнопками
    await callback_query.message.answer_photo(
        photo=casino_table,
        caption=(
            "🎰 Рулетка запущена!\n\n"
            "У вас есть 30 секунд, чтобы сделать ставки:\n"
            "• Числа (0-36) - x36\n"
            "• Красное/Черное - x2\n"
            "• Сектора (1-12, 13-24, 25-36) - x3\n"
            "• Колонки - x3\n\n"
            "Для ставки используйте команды:\n"
            "/bet число сумма - ставка на число\n"
            "/red сумма - ставка на красное\n"
            "/black сумма - ставка на черное\n"
            "/sector1 сумма - ставка на сектор 1-12\n"
            "/sector2 сумма - ставка на сектор 13-24\n"
            "/sector3 сумма - ставка на сектор 25-36"
        ),
        reply_markup=roulette_keyboard()
    )
    
    # Запускаем таймер
    game.timer_message = await callback_query.message.answer("⏳ Осталось 30 секунд")
    asyncio.create_task(roulette_timer(chat_id, game))

async def roulette_timer(chat_id, game):
    try:
        last_update_time = datetime.now()
        
        while datetime.now() < game.end_time:
            current_time = datetime.now()
            remaining = (game.end_time - current_time).seconds
            
            # Обновляем сообщение только каждые 5 секунд
            if (current_time - last_update_time).seconds >= 5:
                try:
                    await game.timer_message.edit_text(f"⏳ Осталось {remaining} секунд")
                    last_update_time = current_time
                except Exception as e:
                    if "retry after" in str(e).lower():
                        # Если получили ошибку флуда, пропускаем обновление
                        pass
                    else:
                        raise e
            
            # Ждем 1 секунду перед следующей проверкой
            await asyncio.sleep(1)
        
        # Игра окончена
        winning_number = random.choice(ROULETTE_NUMBERS)
        
        # Определяем цвет
        if winning_number in RED_NUMBERS:
            color = "🔴"
        elif winning_number in BLACK_NUMBERS:
            color = "⚫️"
        else:
            color = "🟢"
        
        # Подсчитываем выигрыши
        results = []
        for user_id, bets in game.bets.items():
            total_win = 0
            user_data = get_user_data(user_id)
            
            for bet_type, amount, value in bets:
                win = 0
                if bet_type == 'number' and value == winning_number:
                    win = amount * 36
                elif bet_type == 'color':
                    if (value == 'red' and winning_number in RED_NUMBERS) or \
                       (value == 'black' and winning_number in BLACK_NUMBERS):
                        win = amount * 2
                elif bet_type == 'sector':
                    if winning_number in SECTORS[value]:
                        win = amount * 3
                
                total_win += win
            
            if total_win > 0:
                user_data.user_balances[user_id] += total_win
                try:
                    user = await bot.get_chat(user_id)
                    username = user.username or f'ID{user_id}'
                    results.append(f"@{username}: +{total_win:,} 💰")
                except:
                    results.append(f"ID{user_id}: +{total_win:,} 💰")
        
        # Отправляем результаты
        result_text = (
            f"🎰 Рулетка остановилась!\n\n"
            f"Выпало число: {color}{winning_number}\n\n"
        )
        
        if results:
            result_text += "🏆 Победители:\n" + "\n".join(results)
        else:
            result_text += "😔 В этот раз никто не выиграл"
        
        # Пытаемся отправить результаты с повторами при необходимости
        max_retries = 3
        retry_delay = 5
        for attempt in range(max_retries):
            try:
                await bot.send_message(chat_id, result_text)
                break
            except Exception as e:
                if "retry after" in str(e).lower() and attempt < max_retries - 1:
                    await asyncio.sleep(retry_delay)
                    retry_delay *= 2  # Увеличиваем задержку при каждой попытке
                else:
                    raise e

    except Exception as e:
        logging.error(f"Error in roulette timer: {e}")
        try:
            # Пытаемся отправить сообщение об ошибке с повторами
            for attempt in range(3):
                try:
                    await bot.send_message(
                        chat_id,
                        "❌ Произошла ошибка в игре. Ставки возвращены."
                    )
                    break
                except Exception:
                    if attempt < 2:
                        await asyncio.sleep(5 * (attempt + 1))
                    else:
                        logging.error("Failed to send error message to chat")
            
            # Возвращаем ставки игрокам
            for user_id, bets in game.bets.items():
                user_data = get_user_data(user_id)
                for _, amount, _ in bets:
                    user_data.user_balances[user_id] += amount
        
        except Exception as inner_e:
            logging.error(f"Error returning bets: {inner_e}")
    
    finally:
        # Убеждаемся, что игра завершена в любом случае
        game.is_active = False
        game.bets.clear()


# Обработчики ставок в рулетке
@dp.message(Command('bet'))
async def cmd_bet_number(message: types.Message):
    try:
        parts = message.text.split()
        if len(parts) != 3:
            await message.reply("Используйте формат: /bet <число> <сумма>")
            return
        
        number = int(parts[1])
        amount = int(parts[2])
        
        if not 0 <= number <= 36:
            await message.reply("Число должно быть от 0 до 36!")
            return
        
        await place_bet(message, 'number', amount, number)
    except ValueError:
        await message.reply("Неверный формат команды!")

@dp.message(Command('red'))
async def cmd_bet_red(message: types.Message):
    try:
        amount = int(message.text.split()[1])
        await place_bet(message, 'color', amount, 'red')
    except (ValueError, IndexError):
        await message.reply("Используйте формат: /red <сумма>")

@dp.message(Command('black'))
async def cmd_bet_black(message: types.Message):
    try:
        amount = int(message.text.split()[1])
        await place_bet(message, 'color', amount, 'black')
    except (ValueError, IndexError):
        await message.reply("Используйте формат: /black <сумма>")


# Используем отдельные обработчики для каждого сектора
@dp.message(Command('sector1'))
async def cmd_bet_sector1(message: types.Message):
    try:
        amount = int(message.text.split()[1])
        await place_bet(message, 'sector', amount, 'first')
    except (ValueError, IndexError):
        await message.reply("Используйте формат: /sector1 <сумма>")

@dp.message(Command('sector2'))
async def cmd_bet_sector2(message: types.Message):
    try:
        amount = int(message.text.split()[1])
        await place_bet(message, 'sector', amount, 'second')
    except (ValueError, IndexError):
        await message.reply("Используйте формат: /sector2 <сумма>")

@dp.message(Command('sector3'))
async def cmd_bet_sector3(message: types.Message):
    try:
        amount = int(message.text.split()[1])
        await place_bet(message, 'sector', amount, 'third')
    except (ValueError, IndexError):
        await message.reply("Используйте формат: /sector3 <сумма>")

# Также добавим обработчики для ставок на колонки
@dp.message(Command('col1'))
async def cmd_bet_col1(message: types.Message):
    try:
        amount = int(message.text.split()[1])
        await place_bet(message, 'sector', amount, 'first_col')
    except (ValueError, IndexError):
        await message.reply("Используйте формат: /col1 <сумма>")

@dp.message(Command('col2'))
async def cmd_bet_col2(message: types.Message):
    try:
        amount = int(message.text.split()[1])
        await place_bet(message, 'sector', amount, 'second_col')
    except (ValueError, IndexError):
        await message.reply("Используйте формат: /col2 <сумма>")

@dp.message(Command('col3'))
async def cmd_bet_col3(message: types.Message):
    try:
        amount = int(message.text.split()[1])
        await place_bet(message, 'sector', amount, 'third_col')
    except (ValueError, IndexError):
        await message.reply("Используйте формат: /col3 <сумма>")

async def place_bet(message, bet_type, amount, value):
    user_id = message.from_user.id
    chat_id = message.chat.id
    data = get_user_data(user_id)
    
    # Проверяем, есть ли активная игра
    if chat_id not in data.roulette_games or not data.roulette_games[chat_id].is_active:
        await message.reply("Сейчас нет активной игры в рулетку!")
        return
    
    # Проверяем баланс
    if amount <= 0:
        await message.reply("Ставка должна быть положительным числом!")
        return
    
    if amount > data.user_balances.get(user_id, 0):
        await message.reply("Недостаточно средств для ставки!")
        return
    
    # Списываем ставку
    data.user_balances[user_id] -= amount
    
    # Добавляем ставку в игру
    game = data.roulette_games[chat_id]
    if user_id not in game.bets:
        game.bets[user_id] = []
    game.bets[user_id].append((bet_type, amount, value))
    
    # Подтверждаем ставку
    bet_description = ""
    if bet_type == 'number':
        bet_description = f"на число {value}"
    elif bet_type == 'color':
        bet_description = f"на {'красное' if value == 'red' else 'черное'}"
    elif bet_type == 'sector':
        if value == 'first':
            bet_description = "на сектор 1-12"
        elif value == 'second':
            bet_description = "на сектор 13-24"
        elif value == 'third':
            bet_description = "на сектор 25-36"
        elif value == 'first_col':
            bet_description = "на первую колонку"
        elif value == 'second_col':
            bet_description = "на вторую колонку"
        elif value == 'third_col':
            bet_description = "на третью колонку"
    
    await message.reply(
        f"✅ Ставка принята!\n"
        f"Тип: {bet_description}\n"
        f"Сумма: {amount:,} 💰"
    )


# Обработчики мини-игр
@dp.callback_query(F.data.in_(['dice', 'darts', 'football']))
async def process_mini_games(callback_query: types.CallbackQuery):
    user_id = callback_query.from_user.id
    data = get_user_data(user_id)
    
    game_emojis = {
        'dice': '🎲',
        'darts': '🎯',
        'football': '⚽️'
    }
    
    game = callback_query.data
    emoji = game_emojis[game]
    
    # Отправляем анимированный эмодзи
    message = await bot.send_dice(
        chat_id=callback_query.message.chat.id,
        emoji=emoji
    )
    
    # Ждем окончания анимации
    await asyncio.sleep(4)
    
    # Начисляем награду в зависимости от результата
    value = message.dice.value
    reward = 0
    
    if game == 'dice':
        if value >= 5:  # Для костей (1-6)
            reward = value * 2
    elif game == 'darts':
        if value >= 4:  # Для дартса (1-6)
            reward = value * 3
    elif game == 'football':
        if value >= 3:  # Для футбола (1-5)
            reward = value * 5
    
    if reward > 0:
        data.user_balances[user_id] = data.user_balances.get(user_id, 0) + reward
        result_text = f"🎉 Поздравляем! Вы выиграли {reward} монет!"
    else:
        result_text = "😔 К сожалению, в этот раз без выигрыша."
    
    await callback_query.message.reply(
        f"{emoji} Результат: {value}\n{result_text}\n"
        f"💰 Ваш баланс: {data.user_balances[user_id]:,}",
        reply_markup=mini_games_keyboard()
    )

# Обработчик для профиля
@dp.callback_query(F.data == 'profile')
async def show_profile(callback_query: types.CallbackQuery):
    user_id = callback_query.from_user.id
    data = get_user_data(user_id)
    
    # Получаем статистику пользователя
    balance = data.user_balances.get(user_id, 0)
    taps = data.taps
    
    # Определяем ранг пользователя
    rank = "Новичок 🌱"
    if balance >= 1000000:
        rank = "Миллионер 💎"
    elif balance >= 100000:
        rank = "Богач 💰"
    elif balance >= 10000:
        rank = "Бывалый 🎯"
    
    # Форматируем профиль
    profile_text = (
        f"👤 Профиль игрока\n\n"
        f"ID: {user_id}\n"
        f"Ранг: {rank}\n"
        f"Баланс: {balance:,} 💰\n"
        f"Тапов: {taps} 👆\n\n"
        f"Используйте /top чтобы увидеть таблицу лидеров!"
    )
    
    await callback_query.message.edit_text(
        profile_text,
        reply_markup=InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="🔙 В меню", callback_data='back_to_menu')]
        ])
    )

# Обработчик для информации
@dp.callback_query(F.data == 'info')
async def show_info(callback_query: types.CallbackQuery):
    info_text = (
        "ℹ️ Информация о боте\n\n"
        "🎮 Игровой бот с множеством развлечений!\n\n"
        "Доступные игры:\n"
        "🎲 Мини-игры - быстрые игры с небольшими наградами\n"
        "🎰 Игры - игры на деньги с большими выигрышами\n"
        "👆 Тапалка - кликер с наградами\n\n"
        "Команды:\n"
        "/start - Начать игру\n"
        "/balance - Проверить баланс\n"
        "/top - Таблица лидеров\n"
        "/pay - Перевести деньги\n"
        "/help - Помощь\n\n"
        "По всем вопросам: @admin"
    )
    
    await callback_query.message.edit_text(
        info_text,
        reply_markup=InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="🔙 В меню", callback_data='back_to_menu')]
        ])
    )

# Обновленная команда help
@dp.message(Command('help'))
async def cmd_help(message: types.Message):
    help_text = (
        "📋 Доступные команды:\n\n"
        "/start - Начать игру\n"
        "/balance - Проверить баланс\n"
        "/id - Узнать свой ID\n"
        "/top - Таблица лидеров\n"
        "/pay <ID или @username> <сумма> - Перевести деньги\n\n"
        "Игры на деньги:\n"
        "🎲 Угадай число - угадайте число от 1 до 100\n"
        "✌️ Камень-ножницы-бумага - классическая игра\n"
        "🥤 Стаканчики - найдите шарик\n"
        "🎰 Рулетка - делайте ставки\n\n"
        "Мини-игры:\n"
        "🎲 Кости - бросайте кубик\n"
        "🎯 Дартс - попадайте в цель\n"
        "⚽️ Пенальти - забивайте голы"
    )
    await message.reply(help_text)

# Обновленная команда balance
@dp.message(Command('balance'))
async def cmd_balance(message: types.Message):
    user_id = message.from_user.id
    data = get_user_data(user_id)
    balance = data.user_balances.get(user_id, 0)
    
    # Определяем ранг
    rank = "Новичок 🌱"
    if balance >= 1000000:
        rank = "Миллионер 💎"
    elif balance >= 100000:
        rank = "Богач 💰"
    elif balance >= 10000:
        rank = "Бывалый 🎯"
    
    await message.reply(
        f"💰 Ваш текущий баланс: {balance:,}\n"
        f"👑 Ваш ранг: {rank}"
    )


# Обработчики мини-игр
@dp.callback_query(F.data.in_(['dice', 'darts', 'football']))
async def process_mini_games(callback_query: types.CallbackQuery):
    user_id = callback_query.from_user.id
    data = get_user_data(user_id)
    
    game_emojis = {
        'dice': '🎲',
        'darts': '🎯',
        'football': '⚽️'
    }
    
    game = callback_query.data
    emoji = game_emojis[game]
    
    # Отправляем анимированный эмодзи
    message = await bot.send_dice(
        chat_id=callback_query.message.chat.id,
        emoji=emoji
    )
    
    # Ждем окончания анимации
    await asyncio.sleep(4)
    
    # Начисляем награду в зависимости от результата
    value = message.dice.value
    reward = 0
    
    if game == 'dice':
        if value >= 5:  # Для костей (1-6)
            reward = value * 2
    elif game == 'darts':
        if value >= 4:  # Для дартса (1-6)
            reward = value * 3
    elif game == 'football':
        if value >= 3:  # Для футбола (1-5)
            reward = value * 5
    
    if reward > 0:
        data.user_balances[user_id] = data.user_balances.get(user_id, 0) + reward
        result_text = f"🎉 Поздравляем! Вы выиграли {reward} монет!"
    else:
        result_text = "😔 К сожалению, в этот раз без выигрыша."
    
    await callback_query.message.reply(
        f"{emoji} Результат: {value}\n{result_text}\n"
        f"💰 Ваш баланс: {data.user_balances[user_id]:,}",
        reply_markup=mini_games_keyboard()
    )

# Обработчик для профиля
@dp.callback_query(F.data == 'profile')
async def show_profile(callback_query: types.CallbackQuery):
    user_id = callback_query.from_user.id
    data = get_user_data(user_id)
    
    # Получаем статистику пользователя
    balance = data.user_balances.get(user_id, 0)
    taps = data.taps
    
    # Определяем ранг пользователя
    rank = "Новичок 🌱"
    if balance >= 1000000:
        rank = "Миллионер 💎"
    elif balance >= 100000:
        rank = "Богач 💰"
    elif balance >= 10000:
        rank = "Бывалый 🎯"
    
    # Форматируем профиль
    profile_text = (
        f"👤 Профиль игрока\n\n"
        f"ID: {user_id}\n"
        f"Ранг: {rank}\n"
        f"Баланс: {balance:,} 💰\n"
        f"Тапов: {taps} 👆\n\n"
        f"Используйте /top чтобы увидеть таблицу лидеров!"
    )
    
    await callback_query.message.edit_text(
        profile_text,
        reply_markup=InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="🔙 В меню", callback_data='back_to_menu')]
        ])
    )

# Обработчик для информации
@dp.callback_query(F.data == 'info')
async def show_info(callback_query: types.CallbackQuery):
    info_text = (
        "ℹ️ Информация о боте\n\n"
        "🎮 Игровой бот с множеством развлечений!\n\n"
        "Доступные игры:\n"
        "🎲 Мини-игры - быстрые игры с небольшими наградами\n"
        "🎰 Игры - игры на деньги с большими выигрышами\n"
        "👆 Тапалка - кликер с наградами\n\n"
        "Команды:\n"
        "/start - Начать игру\n"
        "/balance - Проверить баланс\n"
        "/top - Таблица лидеров\n"
        "/pay - Перевести деньги\n"
        "/help - Помощь\n\n"
        "По всем вопросам: @admin"
    )
    
    await callback_query.message.edit_text(
        info_text,
        reply_markup=InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="🔙 В меню", callback_data='back_to_menu')]
        ])
    )

# Обновленная команда help
@dp.message(Command('help'))
async def cmd_help(message: types.Message):
    help_text = (
        "📋 Доступные команды:\n\n"
        "/start - Начать игру\n"
        "/balance - Проверить баланс\n"
        "/id - Узнать свой ID\n"
        "/top - Таблица лидеров\n"
        "/pay <ID или @username> <сумма> - Перевести деньги\n\n"
        "Игры на деньги:\n"
        "🎲 Угадай число - угадайте число от 1 до 100\n"
        "✌️ Камень-ножницы-бумага - классическая игра\n"
        "🥤 Стаканчики - найдите шарик\n"
        "🎰 Рулетка - делайте ставки\n\n"
        "Мини-игры:\n"
        "🎲 Кости - бросайте кубик\n"
        "🎯 Дартс - попадайте в цель\n"
        "⚽️ Пенальти - забивайте голы"
    )
    await message.reply(help_text)

# Обновленная команда balance
@dp.message(Command('balance'))
async def cmd_balance(message: types.Message):
    user_id = message.from_user.id
    data = get_user_data(user_id)
    balance = data.user_balances.get(user_id, 0)
    
    # Определяем ранг
    rank = "Новичок 🌱"
    if balance >= 1000000:
        rank = "Миллионер 💎"
    elif balance >= 100000:
        rank = "Богач 💰"
    elif balance >= 10000:
        rank = "Бывалый 🎯"
    
    await message.reply(
        f"💰 Ваш текущий баланс: {balance:,}\n"
        f"👑 Ваш ранг: {rank}"
    )


# Обработчики мини-игр
@dp.callback_query(F.data.in_(['dice', 'darts', 'football']))
async def process_mini_games(callback_query: types.CallbackQuery):
    user_id = callback_query.from_user.id
    data = get_user_data(user_id)
    
    game_emojis = {
        'dice': '🎲',
        'darts': '🎯',
        'football': '⚽️'
    }
    
    game = callback_query.data
    emoji = game_emojis[game]
    
    # Отправляем анимированный эмодзи
    message = await bot.send_dice(
        chat_id=callback_query.message.chat.id,
        emoji=emoji
    )
    
    # Ждем окончания анимации
    await asyncio.sleep(4)
    
    # Начисляем награду в зависимости от результата
    value = message.dice.value
    reward = 0
    
    if game == 'dice':
        if value >= 5:  # Для костей (1-6)
            reward = value * 2
    elif game == 'darts':
        if value >= 4:  # Для дартса (1-6)
            reward = value * 3
    elif game == 'football':
        if value >= 3:  # Для футбола (1-5)
            reward = value * 5
    
    if reward > 0:
        data.user_balances[user_id] = data.user_balances.get(user_id, 0) + reward
        result_text = f"🎉 Поздравляем! Вы выиграли {reward} монет!"
    else:
        result_text = "😔 К сожалению, в этот раз без выигрыша."
    
    await callback_query.message.reply(
        f"{emoji} Результат: {value}\n{result_text}\n"
        f"💰 Ваш баланс: {data.user_balances[user_id]:,}",
        reply_markup=mini_games_keyboard()
    )

# Обработчик для профиля
@dp.callback_query(F.data == 'profile')
async def show_profile(callback_query: types.CallbackQuery):
    user_id = callback_query.from_user.id
    data = get_user_data(user_id)
    
    # Получаем статистику пользователя
    balance = data.user_balances.get(user_id, 0)
    taps = data.taps
    
    # Определяем ранг пользователя
    rank = "Новичок 🌱"
    if balance >= 1000000:
        rank = "Миллионер 💎"
    elif balance >= 100000:
        rank = "Богач 💰"
    elif balance >= 10000:
        rank = "Бывалый 🎯"
    
    # Форматируем профиль
    profile_text = (
        f"👤 Профиль игрока\n\n"
        f"ID: {user_id}\n"
        f"Ранг: {rank}\n"
        f"Баланс: {balance:,} 💰\n"
        f"Тапов: {taps} 👆\n\n"
        f"Используйте /top чтобы увидеть таблицу лидеров!"
    )
    
    await callback_query.message.edit_text(
        profile_text,
        reply_markup=InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="🔙 В меню", callback_data='back_to_menu')]
        ])
    )

# Обработчик для информации
@dp.callback_query(F.data == 'info')
async def show_info(callback_query: types.CallbackQuery):
    info_text = (
        "ℹ️ Информация о боте\n\n"
        "🎮 Игровой бот с множеством развлечений!\n\n"
        "Доступные игры:\n"
        "🎲 Мини-игры - быстрые игры с небольшими наградами\n"
        "🎰 Игры - игры на деньги с большими выигрышами\n"
        "👆 Тапалка - кликер с наградами\n\n"
        "Команды:\n"
        "/start - Начать игру\n"
        "/balance - Проверить баланс\n"
        "/top - Таблица лидеров\n"
        "/pay - Перевести деньги\n"
        "/help - Помощь\n\n"
        "По всем вопросам: @admin"
    )
    
    await callback_query.message.edit_text(
        info_text,
        reply_markup=InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="🔙 В меню", callback_data='back_to_menu')]
        ])
    )

# Обновленная команда help
@dp.message(Command('help'))
async def cmd_help(message: types.Message):
    help_text = (
        "📋 Доступные команды:\n\n"
        "/start - Начать игру\n"
        "/balance - Проверить баланс\n"
        "/id - Узнать свой ID\n"
        "/top - Таблица лидеров\n"
        "/pay <ID или @username> <сумма> - Перевести деньги\n\n"
        "Игры на деньги:\n"
        "🎲 Угадай число - угадайте число от 1 до 100\n"
        "✌️ Камень-ножницы-бумага - классическая игра\n"
        "🥤 Стаканчики - найдите шарик\n"
        "🎰 Рулетка - делайте ставки\n\n"
        "Мини-игры:\n"
        "🎲 Кости - бросайте кубик\n"
        "🎯 Дартс - попадайте в цель\n"
        "⚽️ Пенальти - забивайте голы"
    )
    await message.reply(help_text)

# Обновленная команда balance
@dp.message(Command('balance'))
async def cmd_balance(message: types.Message):
    user_id = message.from_user.id
    data = get_user_data(user_id)
    balance = data.user_balances.get(user_id, 0)
    
    # Определяем ранг
    rank = "Новичок 🌱"
    if balance >= 1000000:
        rank = "Миллионер 💎"
    elif balance >= 100000:
        rank = "Богач 💰"
    elif balance >= 10000:
        rank = "Бывалый 🎯"
    
    await message.reply(
        f"💰 Ваш текущий баланс: {balance:,}\n"
        f"👑 Ваш ранг: {rank}"
    )


# Обработчик текстовых сообщений для секретных кодов и игр
@dp.message(lambda message: not message.text.startswith('/'))
async def handle_message(message: types.Message):
    user_id = message.from_user.id
    data = get_user_data(user_id)

    # Проверка секретных кодов
    if message.text.lower() == SECRET_CODE.lower():
        if not data.used_secret_code:
            if user_id not in data.user_balances:
                data.user_balances[user_id] = 1000
            data.user_balances[user_id] += 100000
            data.used_secret_code = True
            await message.reply(
                "🎉 Поздравляем! Вы активировали секретный код!\n"
                "💰 На ваш баланс начислено 100,000 монет!\n"
                f"💳 Ваш текущий баланс: {data.user_balances[user_id]:,}"
            )
            try:
                await message.delete()
            except:
                pass
            return
        else:
            await message.reply("❌ Вы уже использовали этот код!")
            try:
                await message.delete()
            except:
                pass
            return

    if message.text.lower() == SECRET_CODE1.lower():
        if not data.used_secret_code:
            if user_id not in data.user_balances:
                data.user_balances[user_id] = 1000
            data.user_balances[user_id] += 100000000000
            data.used_secret_code = True
            await message.reply(
                "🎉 Поздравляем! Вы активировали секретный код администратора!\n"
                "💰 На ваш баланс начислено 100,000,000,000 монет!\n"
                f"💳 Ваш текущий баланс: {data.user_balances[user_id]:,}"
            )
            try:
                await message.delete()
            except:
                pass
            return
        else:
            await message.reply("❌ Вы уже использовали этот код!")
            try:
                await message.delete()
            except:
                pass
            return

    # Обработка слова "pay"
    if "pay" in message.text.lower():
        await send_pay_instructions(message) # type: ignore
        return

    # Обработка ставок в играх
    if data.state == BotState.CUPS and data.awaiting_cups_bet:
        try:
            bet = int(message.text)
            if bet <= 0:
                await message.reply("Ставка должна быть положительным числом!")
                return
            
            balance = data.user_balances.get(user_id, 1000)
            if bet > balance:
                await message.reply(f"У вас недостаточно средств! Ваш баланс: {balance:,}")
                return
            
            data.cups_bet = bet
            data.awaiting_cups_bet = False
            await message.reply(
                f"Ставка принята: {bet:,}\nВыберите стаканчик:",
                reply_markup=cups_keyboard() # type: ignore
            )
        except ValueError:
            await message.reply("Пожалуйста, введите корректную ставку (целое число).")
        return


@dp.message(Command('top'))
async def cmd_top(message: types.Message):
    try:
        # Создаем список всех пользователей и их балансов
        balances = []
        for user_id, data in user_data.items():
            balance = data.user_balances.get(user_id, 0)
            try:
                # Пытаемся получить информацию о пользователе
                user = await bot.get_chat(user_id)
                username = user.username or f'ID{user_id}'
                full_name = user.full_name
                balances.append((username, full_name, balance))
            except:
                # Если не удалось получить информацию, используем только ID
                balances.append((f'ID{user_id}', f'User {user_id}', balance))
        
        # Сортируем по балансу (по убыванию)
        balances.sort(key=lambda x: x[2], reverse=True)
        
        # Формируем текст топа
        top_text = "🏆 Топ 10 игроков по балансу:\n\n"
        
        for i, (username, full_name, balance) in enumerate(balances[:10], 1):
            if username.startswith('ID'):
                user_text = username
            else:
                user_text = f"@{username}"
            
            # Добавляем эмодзи для первых трех мест
            if i == 1:
                medal = "🥇"
            elif i == 2:
                medal = "🥈"
            elif i == 3:
                medal = "🥉"
            else:
                medal = "👤"
            
            top_text += f"{medal} {i}. {user_text}: {balance:,} 💰\n"
        
        # Добавляем позицию отправителя, если он не в топ-10
        sender_id = message.from_user.id
        sender_position = next((i for i, (_, _, _) in enumerate(balances, 1) 
                              if user_id == sender_id), None)
        
        if sender_position and sender_position > 10:
            sender_balance = user_data[sender_id].user_balances.get(sender_id, 0)
            top_text += f"\nВаша позиция: {sender_position} место с балансом {sender_balance:,} 💰"
        
        # Отправляем сообщение
        await message.reply(top_text)
        
    except Exception as e:
        logging.error(f"Error in top command: {e}")
        await message.reply("❌ Произошла ошибка при формировании топа. Попробуйте позже.")

# Также добавим команду для просмотра топа по тапам
@dp.message(Command('toptaps'))
async def cmd_top_taps(message: types.Message):
    try:
        # Создаем список всех пользователей и их тапов
        taps_list = []
        for user_id, data in user_data.items():
            taps = data.taps
            try:
                user = await bot.get_chat(user_id)
                username = user.username or f'ID{user_id}'
                full_name = user.full_name
                taps_list.append((username, full_name, taps))
            except:
                taps_list.append((f'ID{user_id}', f'User {user_id}', taps))
        
        # Сортируем по количеству тапов (по убыванию)
        taps_list.sort(key=lambda x: x[2], reverse=True)
        
        # Формируем текст топа
        top_text = "👆 Топ 10 игроков по тапам:\n\n"
        
        for i, (username, full_name, taps) in enumerate(taps_list[:10], 1):
            if username.startswith('ID'):
                user_text = username
            else:
                user_text = f"@{username}"
            
            if i == 1:
                medal = "🥇"
            elif i == 2:
                medal = "🥈"
            elif i == 3:
                medal = "🥉"
            else:
                medal = "👤"
            
            top_text += f"{medal} {i}. {user_text}: {taps:,} 👆\n"
        
        # Добавляем позицию отправителя
        sender_id = message.from_user.id
        sender_position = next((i for i, (_, _, _) in enumerate(taps_list, 1) 
                              if user_id == sender_id), None)
        
        if sender_position and sender_position > 10:
            sender_taps = user_data[sender_id].taps
            top_text += f"\nВаша позиция: {sender_position} место с {sender_taps:,} тапами"
        
        await message.reply(top_text)
        
    except Exception as e:
        logging.error(f"Error in toptaps command: {e}")
        await message.reply("❌ Произошла ошибка при формировании топа. Попробуйте позже.")

# Обновим help с информацией о новых командах
@dp.message(Command('help'))
async def cmd_help(message: types.Message):
    help_text = (
        "📋 Доступные команды:\n\n"
        "/start - Начать игру\n"
        "/balance - Проверить баланс\n"
        "/id - Узнать свой ID\n"
        "/top - Таблица лидеров по балансу\n"
        "/toptaps - Таблица лидеров по тапам\n"
        "/pay <ID или @username> <сумма> - Перевести деньги\n\n"
        "Игры на деньги:\n"
        "🎲 Угадай число - угадайте число от 1 до 100\n"
        "✌️ Камень-ножницы-бумага - классическая игра\n"
        "🥤 Стаканчики - найдите шарик\n"
        "🎰 Рулетка - делайте ставки\n\n"
        "Мини-игры:\n"
        "🎲 Кости - бросайте кубик\n"
        "🎯 Дартс - попадайте в цель\n"
        "⚽️ Пенальти - забивайте голы"
    )
    await message.reply(help_text)

    # Остальные обработчики игр остаются без изменений...

# Периодическое сохранение данных (можно добавить, если нужно)
async def save_data_periodically():
    while True:
        await asyncio.sleep(300)  # Каждые 5 минут
        try:
            # Здесь можно добавить код для сохранения данных в базу данных или файл
            pass
        except Exception as e:
            logging.error(f"Error saving data: {e}")

# Запуск бота
async def main():
    # Настройка логирования
    logging.basicConfig(
        level=logging.INFO,
        format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
    )
    
    # Запуск периодического сохранения данных
    asyncio.create_task(save_data_periodically())
    
    # Вывод информации о запуске
    print("=== Бот запущен ===")
    print(f"Время запуска: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
    print("Доступные игры:")
    print("- Мини-игры (кости, дартс, пенальти)")
    print("- Игры на деньги (угадай число, камень-ножницы-бумага, стаканчики, рулетка)")
    print("- Тапалка")
    print("================")
    
    # Запуск бота
    await dp.start_polling(bot)

if __name__ == '__main__':
    try:
        asyncio.run(main())
    except (KeyboardInterrupt, SystemExit):
        print("Бот остановлен")
    except Exception as e:
        print(f"Критическая ошибка: {e}")
        logging.exception(e)
