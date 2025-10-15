import logging
from aiogram import Bot, Dispatcher, types, F
from aiogram.filters import Command
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.types import ReplyKeyboardMarkup, KeyboardButton, InlineKeyboardMarkup, InlineKeyboardButton, ReplyKeyboardRemove
from aiogram.utils.keyboard import ReplyKeyboardBuilder, InlineKeyboardBuilder
import sqlite3
from datetime import datetime, timedelta
import asyncio

# Настройка логирования
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Состояния для FSM
class ApplicationStates(StatesGroup):
    minecraft_name = State()
    interesting_info = State()

class RecruiterStates(StatesGroup):
    waiting_for_reply = State()

# ID создателя
CREATOR_ID = 6471613931

# Инициализация базы данных с обработкой блокировок
def init_db():
    conn = sqlite3.connect('applications.db', check_same_thread=False)
    cursor = conn.cursor()
    
    # Таблица для заявок
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS applications (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER NOT NULL,
            username TEXT,
            first_name TEXT,
            last_name TEXT,
            minecraft_name TEXT NOT NULL,
            interesting_info TEXT NOT NULL,
            status TEXT DEFAULT 'pending',
            response TEXT,
            responded_by INTEGER,
            response_date TIMESTAMP,
            application_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    
    # Таблица для отслеживания количества заявок
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS user_stats (
            user_id INTEGER PRIMARY KEY,
            application_count INTEGER DEFAULT 0,
            last_application_date TIMESTAMP
        )
    ''')
    
    # Таблица для ответственных за набор
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS recruiters (
            user_id INTEGER PRIMARY KEY,
            username TEXT,
            first_name TEXT,
            last_name TEXT,
            added_by INTEGER,
            added_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    
    # Таблица для банов (отказов)
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS bans (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER NOT NULL,
            banned_by INTEGER NOT NULL,
            reason TEXT,
            ban_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            ban_until TIMESTAMP,
            is_active BOOLEAN DEFAULT TRUE
        )
    ''')
    
    conn.commit()
    conn.close()

# Функция для безопасной работы с базой данных
def execute_db_query(query, params=(), fetch=False):
    conn = sqlite3.connect('applications.db', check_same_thread=False, timeout=30)
    conn.execute("PRAGMA busy_timeout = 30000")  # 30 секунд timeout
    cursor = conn.cursor()
    try:
        cursor.execute(query, params)
        if fetch:
            result = cursor.fetchall()
        else:
            result = cursor.lastrowid
        conn.commit()
        return result
    except Exception as e:
        conn.rollback()
        raise e
    finally:
        conn.close()

# Функция для проверки является ли пользователь создателем
def is_creator(user_id):
    return user_id == CREATOR_ID

# Функция для проверки является ли пользователь ответственным за набор
def is_recruiter(user_id):
    try:
        result = execute_db_query('SELECT 1 FROM recruiters WHERE user_id = ?', (user_id,), fetch=True)
        return len(result) > 0 or is_creator(user_id)
    except Exception as e:
        logger.error(f"Error checking recruiter status: {e}")
        return False

# Функция для добавления пользователя в ответственные
def add_user_as_recruiter(user_id, username, first_name, last_name, added_by):
    try:
        # Проверяем не добавлен ли уже
        existing = execute_db_query('SELECT 1 FROM recruiters WHERE user_id = ?', (user_id,), fetch=True)
        if existing:
            return False, "Этот пользователь уже является ответственным"
        
        # Добавляем в базу
        execute_db_query('''
            INSERT INTO recruiters (user_id, username, first_name, last_name, added_by)
            VALUES (?, ?, ?, ?, ?)
        ''', (user_id, username, first_name, last_name, added_by))
        
        return True, "Пользователь успешно добавлен в ответственные"
    except Exception as e:
        logger.error(f"Error adding recruiter: {e}")
        return False, "Ошибка при добавлении в ответственные"

# Функция для проверки бана пользователя
def is_user_banned(user_id):
    try:
        result = execute_db_query(
            'SELECT ban_until FROM bans WHERE user_id = ? AND is_active = TRUE AND ban_until > ?', 
            (user_id, datetime.now()), 
            fetch=True
        )
        if result:
            ban_until = datetime.fromisoformat(result[0][0])
            return True, ban_until
        return False, None
    except Exception as e:
        logger.error(f"Error checking user ban: {e}")
        return False, None

# Функция для проверки количества заявок
def check_application_limit(user_id):
    try:
        # Сначала проверяем бан
        banned, ban_until = is_user_banned(user_id)
        if banned:
            return False, f"❌ Вы забанены до {ban_until.strftime('%d.%m.%Y %H:%M')}"
        
        result = execute_db_query('SELECT application_count FROM user_stats WHERE user_id = ?', (user_id,), fetch=True)
        if result and result[0][0] >= 3:
            return False, "❌ Вы уже использовали все 3 доступные заявки."
        return True, ""
    except Exception as e:
        logger.error(f"Error checking application limit: {e}")
        return True, ""

# Функция для обновления статистики пользователя
def update_user_stats(user_id):
    try:
        # Сначала проверяем существующую запись
        existing = execute_db_query('SELECT application_count FROM user_stats WHERE user_id = ?', (user_id,), fetch=True)
        
        if existing:
            # Обновляем существующую запись
            execute_db_query(
                'UPDATE user_stats SET application_count = application_count + 1, last_application_date = ? WHERE user_id = ?',
                (datetime.now(), user_id)
            )
        else:
            # Создаем новую запись
            execute_db_query(
                'INSERT INTO user_stats (user_id, application_count, last_application_date) VALUES (?, 1, ?)',
                (user_id, datetime.now())
            )
    except Exception as e:
        logger.error(f"Error updating user stats: {e}")

# Функция для бана пользователя
def ban_user(user_id, banned_by, reason="Отказ в заявке"):
    try:
        ban_until = datetime.now() + timedelta(days=14)  # Бан на 2 недели
        execute_db_query('''
            INSERT INTO bans (user_id, banned_by, reason, ban_until) 
            VALUES (?, ?, ?, ?)
        ''', (user_id, banned_by, reason, ban_until))
        return True
    except Exception as e:
        logger.error(f"Error banning user: {e}")
        return False

# Главное меню с кнопками
def get_main_menu(user_id):
    builder = ReplyKeyboardBuilder()
    builder.add(KeyboardButton(text="📝 Подать заявку"))
    builder.add(KeyboardButton(text="📋 Мои заявки"))
    builder.add(KeyboardButton(text="ℹ️ Помощь"))
    
    if is_creator(user_id) or is_recruiter(user_id):
        builder.add(KeyboardButton(text="👨‍💼 Панель управления"))
    
    builder.adjust(2, 1)
    return builder.as_markup(resize_keyboard=True)

# Панель управления для админов/ответственных
def get_admin_panel(user_id):
    builder = ReplyKeyboardBuilder()
    builder.add(KeyboardButton(text="📨 Посмотреть заявки"))
    builder.add(KeyboardButton(text="📊 Статистика"))
    
    if is_creator(user_id):
        builder.add(KeyboardButton(text="👥 Список ответственных"))
    
    builder.add(KeyboardButton(text="⏪ Назад в меню"))
    builder.adjust(2, 1)
    return builder.as_markup(resize_keyboard=True)

# Клавиатура для управления заявкой (для создателя)
def get_application_control_keyboard_creator(application_id, user_id):
    builder = InlineKeyboardBuilder()
    builder.add(InlineKeyboardButton(text="✅ Принять", callback_data=f"accept_{application_id}"))
    builder.add(InlineKeyboardButton(text="❌ Отказать", callback_data=f"reject_{application_id}"))
    builder.add(InlineKeyboardButton(text="💬 Ответить", callback_data=f"reply_{application_id}"))
    builder.add(InlineKeyboardButton(text="👥 Сделать ответственным", callback_data=f"make_recruiter_{application_id}"))
    builder.add(InlineKeyboardButton(text="📋 Следующая", callback_data=f"next_{application_id}"))
    builder.adjust(2, 2, 1)
    return builder.as_markup()

# Клавиатура для управления заявкой (для обычных ответственных)
def get_application_control_keyboard(application_id):
    builder = InlineKeyboardBuilder()
    builder.add(InlineKeyboardButton(text="✅ Принять", callback_data=f"accept_{application_id}"))
    builder.add(InlineKeyboardButton(text="❌ Отказать", callback_data=f"reject_{application_id}"))
    builder.add(InlineKeyboardButton(text="💬 Ответить", callback_data=f"reply_{application_id}"))
    builder.add(InlineKeyboardButton(text="📋 Следующая", callback_data=f"next_{application_id}"))
    builder.adjust(2, 2)
    return builder.as_markup()

# Команда /start
async def cmd_start(message: types.Message):
    await message.answer(
        "👋 Добро пожаловать в бот для подачи заявок!\n\n"
        "Выберите действие:",
        reply_markup=get_main_menu(message.from_user.id)
    )

# Команда /help
async def cmd_help(message: types.Message):
    help_text = (
        "ℹ️ Помощь по боту:\n\n"
        "📝 Подать заявку - начать процесс подачи заявки на сервер\n"
        "📋 Мои заявки - посмотреть историю ваших заявок\n"
        "ℹ️ Помощь - показать это сообщение\n\n"
        "У каждого пользователя есть лимит в 3 заявки.\n"
        "При отказе в заявке вы не сможете подавать новые заявки 2 недели.\n"
        "После подачи заявки ожидайте ответа от ответственного за набор."
    )
    await message.answer(help_text)

# Начало подачи заявки
async def start_application(message: types.Message, state: FSMContext):
    user_id = message.from_user.id
    
    # Проверяем лимит заявок и бан
    can_apply, reason = check_application_limit(user_id)
    if not can_apply:
        await message.answer(reason)
        return
    
    await message.answer(
        "📝 Начинаем процесс подачи заявки!\n\n"
        "Введите ваш ник в Minecraft:",
        reply_markup=ReplyKeyboardRemove()
    )
    await state.set_state(ApplicationStates.minecraft_name)

# Получение ника в Minecraft
async def get_minecraft_name(message: types.Message, state: FSMContext):
    await state.update_data(minecraft_name=message.text)
    await message.answer(
        "Расскажите интересную информацию о себе "
        "(почему хотите присоединиться, ваши увлечения и т.д.):"
    )
    await state.set_state(ApplicationStates.interesting_info)

# Получение интересной информации и сохранение заявки
async def get_interesting_info(message: types.Message, state: FSMContext, bot: Bot):
    user_data = await state.get_data()
    user = message.from_user
    
    try:
        # Сохраняем заявку в базу данных
        application_id = execute_db_query('''
            INSERT INTO applications 
            (user_id, username, first_name, last_name, minecraft_name, interesting_info) 
            VALUES (?, ?, ?, ?, ?, ?)
        ''', (
            user.id,
            user.username,
            user.first_name,
            user.last_name,
            user_data['minecraft_name'],
            message.text
        ))
        
        # Обновляем статистику пользователя
        update_user_stats(user.id)
        
        # Получаем текущее количество заявок пользователя
        result = execute_db_query('SELECT application_count FROM user_stats WHERE user_id = ?', (user.id,), fetch=True)
        app_count = result[0][0] if result else 1
        
        # Отправляем уведомление ответственным
        await notify_recruiters(bot, application_id, user, user_data['minecraft_name'], message.text)
        
        await message.answer(
            f"✅ Заявка успешно подана!\n\n"
            f"📊 Статистика:\n"
            f"• Использовано заявок: {app_count}/3\n"
            f"• Осталось заявок: {3 - app_count}\n\n"
            f"Ожидайте ответа от ответственного за набор!",
            reply_markup=get_main_menu(user.id)
        )
        
    except Exception as e:
        logger.error(f"Error saving application: {e}")
        await message.answer(
            "❌ Произошла ошибка при сохранении заявки. Пожалуйста, попробуйте позже.",
            reply_markup=get_main_menu(user.id)
        )
    
    # Очищаем состояние
    await state.clear()

# Уведомление ответственных о новой заявке
async def notify_recruiters(bot: Bot, app_id: int, applicant, minecraft_name: str, interesting_info: str):
    try:
        recruiters = execute_db_query('SELECT user_id FROM recruiters', fetch=True)
        
        # Добавляем создателя в список уведомлений
        all_recruiters = [(CREATOR_ID,)] + recruiters
        
        for recruiter in all_recruiters:
            recruiter_id = recruiter[0]
            
            # Выбираем клавиатуру в зависимости от прав
            if recruiter_id == CREATOR_ID:
                keyboard = get_application_control_keyboard_creator(app_id, applicant.id)
            else:
                keyboard = get_application_control_keyboard(app_id)
            
            try:
                await bot.send_message(
                    chat_id=recruiter_id,
                    text=f"📨 Новая заявка #{app_id}\n\n"
                         f"👤 Айди человека в ТГ: {applicant.id}\n"
                         f"👤 Имя: {applicant.first_name or 'Не указано'}\n"
                         f"👤 Юзернейм: @{applicant.username or 'не указан'}\n"
                         f"📛 Ник в Minecraft: {minecraft_name}\n"
                         f"💬 Сообщение: {interesting_info}",
                    reply_markup=keyboard
                )
            except Exception as e:
                logger.error(f"Не удалось отправить уведомление пользователю {recruiter_id}: {e}")
    except Exception as e:
        logger.error(f"Error notifying recruiters: {e}")

# Просмотр заявок для ответственных
async def view_applications(message: types.Message):
    if not (is_creator(message.from_user.id) or is_recruiter(message.from_user.id)):
        await message.answer("❌ У вас нет доступа к этой функции")
        return
    
    try:
        # Получаем первую ожидающую заявку
        applications = execute_db_query('''
            SELECT id, user_id, username, first_name, minecraft_name, interesting_info, application_date 
            FROM applications 
            WHERE status = 'pending'
            ORDER BY application_date ASC 
            LIMIT 1
        ''', fetch=True)
        
        if not applications:
            await message.answer("📝 Нет ожидающих заявок.", reply_markup=get_admin_panel(message.from_user.id))
            return
        
        app = applications[0]
        app_id, user_id, username, first_name, minecraft_name, interesting_info, date = app
        
        response = f"📋 Заявка #{app_id}\n\n"
        response += f"👤 Пользователь:\n"
        response += f"   ID: {user_id}\n"
        response += f"   Имя: {first_name or 'Не указано'}\n"
        response += f"   Юзернейм: @{username or 'не указан'}\n\n"
        response += f"🎮 Ник в Minecraft: {minecraft_name}\n\n"
        response += f"💬 Информация:\n{interesting_info}\n\n"
        response += f"📅 Дата подачи: {date}"
        
        # Выбираем клавиатуру в зависимости от прав
        if is_creator(message.from_user.id):
            keyboard = get_application_control_keyboard_creator(app_id, user_id)
        else:
            keyboard = get_application_control_keyboard(app_id)
        
        await message.answer(
            response,
            reply_markup=keyboard
        )
        
    except Exception as e:
        logger.error(f"Error viewing applications: {e}")
        await message.answer("❌ Произошла ошибка при получении заявок", reply_markup=get_admin_panel(message.from_user.id))

# Обработка callback кнопок
async def handle_callback(callback: types.CallbackQuery, state: FSMContext, bot: Bot):
    user_id = callback.from_user.id
    
    if not (is_creator(user_id) or is_recruiter(user_id)):
        await callback.answer("❌ У вас нет прав для этого действия", show_alert=True)
        return
    
    data = callback.data
    
    try:
        # Разбираем callback data
        parts = data.split('_')
        action = parts[0]
        
        if action in ['accept', 'reject', 'reply', 'next']:
            app_id = int(parts[1])
            
            if action == 'accept':
                await handle_application_response(callback, bot, app_id, "accepted", "✅ Принята")
            elif action == 'reject':
                await handle_application_response(callback, bot, app_id, "rejected", "❌ Отклонена", ban_user=True)
            elif action == 'reply':
                await start_reply(callback, state, app_id)
            elif action == 'next':
                await show_next_application(callback, bot)
                
        elif action == 'make' and parts[1] == 'recruiter':
            app_id = int(parts[2])
            await make_user_recruiter(callback, bot, app_id)
        else:
            await callback.answer("❌ Неизвестное действие", show_alert=True)
            
    except (ValueError, IndexError) as e:
        logger.error(f"Error parsing callback data: {data}, error: {e}")
        await callback.answer("❌ Ошибка обработки запроса", show_alert=True)
    
    await callback.answer()

# Сделать пользователя ответственным
async def make_user_recruiter(callback: types.CallbackQuery, bot: Bot, app_id: int):
    if not is_creator(callback.from_user.id):
        await callback.answer("❌ Только создатель может добавлять ответственных", show_alert=True)
        return
    
    try:
        # Получаем информацию о заявке и пользователе
        result = execute_db_query('''
            SELECT user_id, username, first_name, last_name 
            FROM applications 
            WHERE id = ?
        ''', (app_id,), fetch=True)
        
        if not result:
            await callback.answer("❌ Заявка не найдена", show_alert=True)
            return
            
        applicant_id, username, first_name, last_name = result[0]
        
        # Добавляем пользователя в ответственные
        success, message = add_user_as_recruiter(applicant_id, username, first_name, last_name, callback.from_user.id)
        
        if success:
            # Уведомляем нового ответственного
            try:
                await bot.send_message(
                    chat_id=applicant_id,
                    text="🎉 Вас назначили ответственным за набор!\n\n"
                         "Теперь вы можете:\n"
                         "• Просматривать заявки\n"
                         "• Принимать/отклонять заявки\n"
                         "• Отвечать пользователям\n\n"
                         "Для начала работы нажмите /start"
                )
            except Exception as e:
                logger.error(f"Не удалось уведомить нового ответственного: {e}")
            
            await callback.message.edit_text(
                f"✅ Пользователь @{username or first_name} теперь ответственный!\n\n"
                f"📋 Заявка #{app_id} обработана."
            )
        else:
            await callback.answer(message, show_alert=True)
            
    except Exception as e:
        logger.error(f"Error making user recruiter: {e}")
        await callback.answer("❌ Ошибка при назначении ответственного", show_alert=True)

# Показать следующую заявку
async def show_next_application(callback: types.CallbackQuery, bot: Bot):
    try:
        # Получаем следующую ожидающую заявку
        applications = execute_db_query('''
            SELECT id, user_id, username, first_name, minecraft_name, interesting_info, application_date 
            FROM applications 
            WHERE status = 'pending'
            ORDER BY application_date ASC 
            LIMIT 1
        ''', fetch=True)
        
        if not applications:
            await callback.message.edit_text("📝 Больше нет ожидающих заявок.")
            return
        
        app = applications[0]
        app_id, user_id, username, first_name, minecraft_name, interesting_info, date = app
        
        response = f"📋 Заявка #{app_id}\n\n"
        response += f"👤 Пользователь:\n"
        response += f"   ID: {user_id}\n"
        response += f"   Имя: {first_name or 'Не указано'}\n"
        response += f"   Юзернейм: @{username or 'не указан'}\n\n"
        response += f"🎮 Ник в Minecraft: {minecraft_name}\n\n"
        response += f"💬 Информация:\n{interesting_info}\n\n"
        response += f"📅 Дата подачи: {date}"
        
        # Выбираем клавиатуру в зависимости от прав
        if is_creator(callback.from_user.id):
            keyboard = get_application_control_keyboard_creator(app_id, user_id)
        else:
            keyboard = get_application_control_keyboard(app_id)
        
        await callback.message.edit_text(
            response,
            reply_markup=keyboard
        )
        
    except Exception as e:
        logger.error(f"Error showing next application: {e}")
        await callback.message.edit_text("❌ Произошла ошибка при загрузке следующей заявки")

# Обработка ответа на заявку
async def handle_application_response(callback: types.CallbackQuery, bot: Bot, app_id: int, status: str, status_text: str, ban_user=False):
    user_id = callback.from_user.id
    
    try:
        # Обновляем статус заявки
        execute_db_query('''
            UPDATE applications 
            SET status = ?, responded_by = ?, response_date = ?
            WHERE id = ?
        ''', (status, user_id, datetime.now(), app_id))
        
        # Получаем информацию о заявке
        result = execute_db_query('SELECT user_id, minecraft_name FROM applications WHERE id = ?', (app_id,), fetch=True)
        if not result:
            await callback.message.edit_text("❌ Заявка не найдена")
            return
            
        applicant_id = result[0][0]
        minecraft_name = result[0][1]
        
        # Если отказ - баним пользователя на 2 недели
        if ban_user:
            ban_success = ban_user(applicant_id, user_id, "Отказ в заявке")
            if ban_success:
                ban_message = "\n🚫 Вы забанены на 2 недели и не можете подавать новые заявки."
            else:
                ban_message = "\n⚠️ Не удалось установить бан."
        else:
            ban_message = ""
        
        # Уведомляем applicant
        try:
            await bot.send_message(
                chat_id=applicant_id,
                text=f"📢 Статус вашей заявки изменен!\n\n"
                     f"📛 Ник: {minecraft_name}\n"
                     f"📊 Статус: {status_text}{ban_message}\n"
                     f"👤 Ответственный: @{callback.from_user.username if callback.from_user.username else 'не указан'}"
            )
        except Exception as e:
            logger.error(f"Не удалось уведомить applicant {applicant_id}: {e}")
        
        await callback.message.edit_text(
            f"✅ Заявка #{app_id} {status_text.lower()}\n"
            f"👤 Applicant: {applicant_id}\n"
            f"📛 Ник: {minecraft_name}"
            f"{' - ЗАБАНЕН на 2 недели' if ban_user else ''}"
        )
        
    except Exception as e:
        logger.error(f"Error handling application response: {e}")
        await callback.message.edit_text("❌ Произошла ошибка при обработке заявки")

# Начало ответа на заявку
async def start_reply(callback: types.CallbackQuery, state: FSMContext, app_id: int):
    await state.update_data(replying_to=app_id, reply_message_id=callback.message.message_id)
    await callback.message.edit_text(
        f"💬 Введите ответ для заявки #{app_id}:\n"
        f"Сообщение будет отправлено пользователю."
    )
    await state.set_state(RecruiterStates.waiting_for_reply)

# Обработка ответов от ответственных
async def handle_recruiter_reply(message: types.Message, state: FSMContext, bot: Bot):
    user_data = await state.get_data()
    app_id = user_data.get('replying_to')
    
    if not app_id:
        await state.clear()
        return
    
    reply_text = message.text
    
    try:
        # Получаем информацию о заявке
        result = execute_db_query('SELECT user_id, minecraft_name FROM applications WHERE id = ?', (app_id,), fetch=True)
        if not result:
            await message.answer("❌ Заявка не найдена")
            await state.clear()
            return
            
        applicant_id = result[0][0]
        minecraft_name = result[0][1]
        
        # Обновляем заявку
        execute_db_query('''
            UPDATE applications 
            SET response = ?, responded_by = ?, response_date = ?
            WHERE id = ?
        ''', (reply_text, message.from_user.id, datetime.now(), app_id))
        
        # Отправляем ответ applicant
        try:
            await bot.send_message(
                chat_id=applicant_id,
                text=f"💌 Ответ на вашу заявку:\n\n"
                     f"📛 Ваш ник: {minecraft_name}\n"
                     f"💬 Сообщение: {reply_text}\n\n"
                     f"👤 Ответственный: @{message.from_user.username if message.from_user.username else 'не указан'}"
            )
        except Exception as e:
            logger.error(f"Не удалось отправить ответ applicant {applicant_id}: {e}")
            await message.answer("❌ Не удалось отправить ответ пользователю")
            return
        
        await message.answer("✅ Ответ успешно отправлен!", reply_markup=get_admin_panel(message.from_user.id))
        
    except Exception as e:
        logger.error(f"Error handling recruiter reply: {e}")
        await message.answer("❌ Произошла ошибка при отправке ответа")
    
    await state.clear()

# Панель управления
async def admin_panel(message: types.Message):
    if not (is_creator(message.from_user.id) or is_recruiter(message.from_user.id)):
        await message.answer("❌ У вас нет доступа к панели управления")
        return
    
    await message.answer(
        "👨‍💼 Панель управления\n\n"
        "Выберите действие:",
        reply_markup=get_admin_panel(message.from_user.id)
    )

# Показать список ответственных
async def show_recruiters_list(message: types.Message):
    if not is_creator(message.from_user.id):
        await message.answer("❌ Только создатель может просматривать список ответственных")
        return
    
    try:
        recruiters = execute_db_query('SELECT * FROM recruiters', fetch=True)
        
        if not recruiters:
            response = "📝 Список ответственных пуст"
        else:
            response = "👥 Список ответственных:\n\n"
            for recruiter in recruiters:
                response += f"• ID: {recruiter[0]}\n"
                response += f"  Имя: {recruiter[2] or 'Не указано'}\n"
                response += f"  Юзернейм: @{recruiter[1] or 'не указан'}\n\n"
        
        await message.answer(response, reply_markup=get_admin_panel(message.from_user.id))
    except Exception as e:
        logger.error(f"Error managing recruiters: {e}")
        await message.answer("❌ Произошла ошибка при получении списка ответственных", reply_markup=get_admin_panel(message.from_user.id))

# Просмотр статистики
async def view_statistics(message: types.Message):
    if not (is_creator(message.from_user.id) or is_recruiter(message.from_user.id)):
        return
    
    try:
        # Общая статистика
        total_apps = execute_db_query('SELECT COUNT(*) FROM applications', fetch=True)[0][0]
        pending_apps = execute_db_query('SELECT COUNT(*) FROM applications WHERE status = "pending"', fetch=True)[0][0]
        accepted_apps = execute_db_query('SELECT COUNT(*) FROM applications WHERE status = "accepted"', fetch=True)[0][0]
        rejected_apps = execute_db_query('SELECT COUNT(*) FROM applications WHERE status = "rejected"', fetch=True)[0][0]
        banned_users = execute_db_query('SELECT COUNT(*) FROM bans WHERE is_active = TRUE AND ban_until > ?', (datetime.now(),), fetch=True)[0][0]
        total_recruiters = execute_db_query('SELECT COUNT(*) FROM recruiters', fetch=True)[0][0]
        
        response = "📊 Статистика:\n\n"
        response += f"📨 Всего заявок: {total_apps}\n"
        response += f"⏳ Ожидают: {pending_apps}\n"
        response += f"✅ Принято: {accepted_apps}\n"
        response += f"❌ Отклонено: {rejected_apps}\n"
        response += f"🚫 Забанено: {banned_users}\n"
        response += f"👥 Ответственных: {total_recruiters}"
        
        await message.answer(response, reply_markup=get_admin_panel(message.from_user.id))
    except Exception as e:
        logger.error(f"Error viewing statistics: {e}")
        await message.answer("❌ Произошла ошибка при получении статистики", reply_markup=get_admin_panel(message.from_user.id))

# Просмотр своих заявок
async def my_applications(message: types.Message):
    user_id = message.from_user.id
    
    try:
        # Получаем статистику
        stats = execute_db_query('SELECT application_count FROM user_stats WHERE user_id = ?', (user_id,), fetch=True)
        
        if not stats or stats[0][0] == 0:
            await message.answer("У вас пока нет поданых заявок.", reply_markup=get_main_menu(user_id))
            return
        
        app_count = stats[0][0]
        
        # Получаем последние заявки
        applications = execute_db_query('''
            SELECT minecraft_name, interesting_info, status, response, application_date 
            FROM applications 
            WHERE user_id = ? 
            ORDER BY application_date DESC
        ''', (user_id,), fetch=True)
        
        response = f"📋 Ваши заявки ({app_count}/3):\n\n"
        
        for i, app in enumerate(applications, 1):
            minecraft_name, interesting_info, status, response_text, date = app
            status_text = "✅ Принята" if status == "accepted" else "❌ Отклонена" if status == "rejected" else "⏳ На рассмотрении"
            
            response += f"📄 Заявка #{i}:\n"
            response += f"   🎮 Ник: {minecraft_name}\n"
            response += f"   💬 Инфо: {interesting_info}\n"
            response += f"   📊 Статус: {status_text}\n"
            
            if response_text:
                response += f"   💌 Ответ: {response_text}\n"
            
            response += f"   📅 Дата: {date}\n\n"
        
        await message.answer(response, reply_markup=get_main_menu(user_id))
    except Exception as e:
        logger.error(f"Error viewing user applications: {e}")
        await message.answer("❌ Произошла ошибка при получении ваших заявок", reply_markup=get_main_menu(user_id))

# Возврат в главное меню
async def back_to_main_menu(message: types.Message, state: FSMContext):
    await state.clear()
    await message.answer(
        "Главное меню:",
        reply_markup=get_main_menu(message.from_user.id)
    )

# Основная функция
async def main():
    # Инициализация базы данных
    init_db()
    
    # Токен бота
    TOKEN = "8355580515:AAGXkvDqf7xmqlp1Q_rUNLWCEwD1DX-m8pM"
    
    bot = Bot(token=TOKEN)
    dp = Dispatcher()
    
    # Регистрация обработчиков команд
    dp.message.register(cmd_start, Command("start"))
    dp.message.register(cmd_help, Command("help"))
    
    # Регистрация обработчиков кнопок навигации
    dp.message.register(back_to_main_menu, F.text == "⏪ Назад в меню")
    
    # Регистрация обработчиков основных кнопок
    dp.message.register(start_application, F.text == "📝 Подать заявку")
    dp.message.register(my_applications, F.text == "📋 Мои заявки")
    dp.message.register(cmd_help, F.text == "ℹ️ Помощь")
    dp.message.register(admin_panel, F.text == "👨‍💼 Панель управления")
    dp.message.register(show_recruiters_list, F.text == "👥 Список ответственных")
    dp.message.register(view_applications, F.text == "📨 Посмотреть заявки")
    dp.message.register(view_statistics, F.text == "📊 Статистика")
    
    # Регистрация обработчиков состояний
    dp.message.register(get_minecraft_name, ApplicationStates.minecraft_name)
    dp.message.register(get_interesting_info, ApplicationStates.interesting_info)
    dp.message.register(handle_recruiter_reply, RecruiterStates.waiting_for_reply)
    
    # Регистрация обработчиков callback
    dp.callback_query.register(handle_callback)
    
    # Запуск бота
    print("Бот запущен...")
    await dp.start_polling(bot)

if __name__ == '__main__':
    asyncio.run(main())