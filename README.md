#!/usr/bin/env python3
# bot_final.py - бот «Космос»
# =====================================================
# УНИВЕРСАЛЬНАЯ ВЕРСИЯ (Pydroid3 + Сервер + Railway)
#
# Настройки берутся из переменных окружения (os.getenv).
# Если переменных нет - используются значения по умолчанию
# из этого файла, поэтому код работает и на телефоне.
#
# Переменные окружения (задаются на сервере):
#   TOKEN          - токен бота
#   ADMIN_USER_ID  - твой Telegram ID (создатель)
#   ADMIN_KEY      - пароль админ-панели
#   PUBG_API_KEY   - ключ PUBG API (необязательно)
#   PUBG_REGION    - регион: steam / xbox / psn / kakao
#   MAINTENANCE    - стартовый режим тех. работ: True/False
# =====================================================

import csv
import io
import json
import logging
import os
import pathlib
import re
import sqlite3
import time
from datetime import datetime

import requests
from gtts import gTTS
import telebot
from telebot import types

# ====================== ПУТИ ======================
BASE_DIR = pathlib.Path(__file__).resolve().parent
DB_PATH = BASE_DIR / "bot_data.db"
CSV_PATH = BASE_DIR / "players.csv"
ADMIN_CONF_PATH = BASE_DIR / "admin.json"
MESSAGES_PATH = BASE_DIR / "messages.json"

# ====================== НАСТРОЙКИ ======================
# Берём из переменных окружения; если нет - используем значения по умолчанию
TOKEN = os.getenv("TOKEN", "8970636002:AAEvmmDDztqgdXkDYzr4SA83Bu4uNzopDig")
ADMIN_USER_ID = int(os.getenv("ADMIN_USER_ID", "5290886402"))
MAX_WARNINGS = 3   # после скольких предупреждений кикать и заносить в ЧС
WELCOME_TEXT = "👋 Добро пожаловать в группу, {name}!"
GOODBYE_TEXT = "👋 {name} покинул(а) группу."

# Стартовый режим тех. работ (можно задать через переменную окружения)
MAINTENANCE = os.getenv("MAINTENANCE", "False").lower() in ("1", "true", "да", "on")
# =======================================================

# ====================== АДМИН-ПАРОЛЬ (admin.json) ======================
def load_admin_key() -> str:
    """Читает пароль из admin.json; если файла нет - создаёт с ключом по умолчанию."""
    default_key = os.getenv("ADMIN_KEY", "trial_password_12345")
    if not ADMIN_CONF_PATH.is_file():
        try:
            with open(ADMIN_CONF_PATH, "w", encoding="utf-8") as f:
                json.dump({"ADMIN_KEY": default_key}, f, ensure_ascii=False, indent=2)
        except Exception as e:
            logging.error("Не удалось создать admin.json: %s", e)
        return default_key
    try:
        with open(ADMIN_CONF_PATH, "r", encoding="utf-8") as f:
            return json.load(f).get("ADMIN_KEY", default_key)
    except Exception as e:
        logging.error("Не удалось прочитать admin.json: %s", e)
        return default_key


def save_admin_key(new_key: str) -> bool:
    """Сохраняет новый пароль в admin.json."""
    try:
        with open(ADMIN_CONF_PATH, "w", encoding="utf-8") as f:
            json.dump({"ADMIN_KEY": new_key}, f, ensure_ascii=False, indent=2)
        return True
    except Exception as e:
        logging.error("Не удалось сохранить admin.json: %s", e)
        return False


ADMIN_KEY = load_admin_key()
# =======================================================

# ====================== СООБЩЕНИЯ (messages.json) ======================
def load_messages() -> dict:
    """Загружает переопределённые тексты сообщений из messages.json."""
    if not MESSAGES_PATH.is_file():
        return {}
    try:
        with open(MESSAGES_PATH, "r", encoding="utf-8") as f:
            return json.load(f)
    except Exception as e:
        logging.error("Не удалось прочитать messages.json: %s", e)
        return {}


def save_messages(data: dict) -> bool:
    """Сохраняет переопределённые тексты сообщений в messages.json."""
    try:
        with open(MESSAGES_PATH, "w", encoding="utf-8") as f:
            json.dump(data, f, ensure_ascii=False, indent=2)
        return True
    except Exception as e:
        logging.error("Не удалось сохранить messages.json: %s", e)
        return False


messages = load_messages()
# =======================================================

# ====================== PUBG API ======================
PUBG_API_KEY = os.getenv(
    "PUBG_API_KEY",
    "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9."
    "eyJqdGkiOiIyMDY3MGYyMC1iODA4LTAxM2YtNDlhZC01YWE1ZDU5NzBlYjEiLCJpc3MiOiJnYW1lbG9ja2VyIiwiaWF0IjoxNzkz"
    "NTI2ODU3LCJwdWIiOiJibHVlaG9sZSIsInRpdGxlIjoicHViZyIsImFwcCI6Imtvc21vcy11Yy1ib3QifQ."
    "QA8EnN--CTwe-0U_OciwkSnAbb8uIXohQl89UKq3p3Q",
)
PUBG_REGION = os.getenv("PUBG_REGION", "steam")   # steam / xbox / psn / kakao
# =======================================================

# ====================== ЛОГИРОВАНИЕ ======================
logging.basicConfig(
    format="%(asctime)s - %(levelname)s - %(message)s",
    level=logging.INFO,
)

bot = telebot.TeleBot(TOKEN)


def now_str() -> str:
    """Текущее время для логов (ЧЧ:ММ:СС)."""
    return datetime.now().strftime("%H:%M:%S")
# =======================================================

# ====================== РЕЖИМ ТЕХНИЧЕСКИХ РАБОТ ======================
def is_under_maintenance(message) -> bool:
    """True, если включён режим и пользователь НЕ администратор."""
    return MAINTENANCE and message.from_user.id != ADMIN_USER_ID


def maintenance_reply(message):
    """Ответ, который видит обычный пользователь во время тех. работ."""
    bot.reply_to(
        message,
        messages.get(
            "maintenance_text",
            "🤖 Бот в разработке. Идут технические работы. Попробуйте позже.",
        ),
    )


# ВАЖНО: этот обработчик должен быть ПЕРВЫМ - перехватывает ВСЁ при тех. работах
@bot.message_handler(
    func=lambda m: is_under_maintenance(m),
    content_types=[
        "text", "audio", "document", "photo", "sticker", "video",
        "video_note", "voice", "location", "contact", "new_chat_members",
        "left_chat_member", "new_chat_title", "new_chat_photo",
        "delete_chat_photo", "group_chat_created", "supergroup_chat_created",
        "channel_chat_created", "pinned_message", "invoice",
        "successful_payment",
    ],
)
def maintenance_block(message):
    maintenance_reply(message)
# =======================================================

# ------------------- БАЗА ДАННЫХ -------------------
def init_db():
    """Создаёт все таблицы, если их ещё нет, и импортирует players.csv."""
    with sqlite3.connect(DB_PATH) as conn:
        conn.executescript(
            """
            CREATE TABLE IF NOT EXISTS players (
                id TEXT PRIMARY KEY,
                nickname TEXT NOT NULL
            );
            CREATE TABLE IF NOT EXISTS prices (
                item TEXT PRIMARY KEY,
                price TEXT NOT NULL
            );
            CREATE TABLE IF NOT EXISTS msg_counter (
                chat_id INTEGER,
                user_id INTEGER,
                cnt INTEGER,
                PRIMARY KEY (chat_id, user_id)
            );
            CREATE TABLE IF NOT EXISTS chat_lock (
                chat_id INTEGER PRIMARY KEY,
                locked INTEGER
            );
            CREATE TABLE IF NOT EXISTS actions (
                action TEXT PRIMARY KEY
            );
            CREATE TABLE IF NOT EXISTS bot_groups (
                chat_id INTEGER PRIMARY KEY,
                title TEXT
            );
            CREATE TABLE IF NOT EXISTS users (
                user_id INTEGER PRIMARY KEY,
                username TEXT,
                first_name TEXT
            );
            CREATE TABLE IF NOT EXISTS warnings (
                user_id INTEGER PRIMARY KEY,
                cnt INTEGER NOT NULL DEFAULT 0
            );
            CREATE TABLE IF NOT EXISTS blacklist (
                user_id INTEGER PRIMARY KEY
            );
            CREATE TABLE IF NOT EXISTS uc_balance (
                user_id INTEGER PRIMARY KEY,
                balance INTEGER NOT NULL DEFAULT 0
            );
            """
        )
        # Импорт CSV-игроков (если файл существует)
        if CSV_PATH.is_file():
            try:
                with CSV_PATH.open(newline="", encoding="utf-8") as f:
                    rows = [(r["id"], r["nickname"]) for r in csv.DictReader(f)]
                conn.executemany(
                    "INSERT OR REPLACE INTO players (id, nickname) VALUES (?, ?)",
                    rows,
                )
                logging.info("Импортировано %d игроков", len(rows))
            except Exception as e:
                logging.error("Ошибка CSV: %s", e)
        conn.commit()


# ------------------- ЦЕНЫ -------------------
def load_prices():
    """Возвращает словарь {название: цена} из БД."""
    with sqlite3.connect(DB_PATH) as conn:
        cur = conn.cursor()
        cur.execute("SELECT item, price FROM prices")
        return {item: price for item, price in cur.fetchall()}


def save_price(item, price):
    """Сохраняет или обновляет позицию прайса."""
    with sqlite3.connect(DB_PATH) as conn:
        conn.execute(
            "INSERT OR REPLACE INTO prices (item, price) VALUES (?, ?)",
            (item, price),
        )
        conn.commit()


def delete_price(item):
    """Удаляет позицию прайса. Возвращает True, если удалена."""
    with sqlite3.connect(DB_PATH) as conn:
        cur = conn.execute("DELETE FROM prices WHERE item = ?", (item,))
        conn.commit()
        return cur.rowcount > 0


def get_nickname(player_id):
    """Возвращает ник игрока по ID из players.csv/БД."""
    with sqlite3.connect(DB_PATH) as conn:
        cur = conn.cursor()
        cur.execute("SELECT nickname FROM players WHERE id = ?", (player_id,))
        row = cur.fetchone()
    return row[0] if row else None


# ------------------- СЧЁТЧИК СООБЩЕНИЙ -------------------
def inc_msg_counter(chat_id, user_id):
    """Увеличивает счётчик сообщений пользователя в чате на 1."""
    with sqlite3.connect(DB_PATH) as conn:
        cur = conn.cursor()
        cur.execute(
            "SELECT cnt FROM msg_counter WHERE chat_id = ? AND user_id = ?",
            (chat_id, user_id),
        )
        row = cur.fetchone()
        if row:
            cur.execute(
                "UPDATE msg_counter SET cnt = cnt + 1 WHERE chat_id = ? AND user_id = ?",
                (chat_id, user_id),
            )
        else:
            cur.execute(
                "INSERT INTO msg_counter (chat_id, user_id, cnt) VALUES (?, ?, 1)",
                (chat_id, user_id),
            )
        conn.commit()


def get_msg_count(chat_id, user_id):
    """Возвращает количество сообщений пользователя в чате."""
    with sqlite3.connect(DB_PATH) as conn:
        cur = conn.cursor()
        cur.execute(
            "SELECT cnt FROM msg_counter WHERE chat_id = ? AND user_id = ?",
            (chat_id, user_id),
        )
        row = cur.fetchone()
    return row[0] if row else 0


# ------------------- БЛОКИРОВКА ЧАТА -------------------
def set_chat_lock(chat_id, locked):
    """Устанавливает состояние блокировки чата (True/False)."""
    with sqlite3.connect(DB_PATH) as conn:
        conn.execute(
            "INSERT OR REPLACE INTO chat_lock (chat_id, locked) VALUES (?, ?)",
            (chat_id, int(locked)),
        )
        conn.commit()


def is_chat_locked(chat_id):
    """True, если чат заблокирован."""
    with sqlite3.connect(DB_PATH) as conn:
        cur = conn.cursor()
        cur.execute("SELECT locked FROM chat_lock WHERE chat_id = ?", (chat_id,))
        row = cur.fetchone()
    return bool(row[0]) if row else False


# ------------------- ПРАВА -------------------
def is_group_admin(chat_id, user_id):
    """True, если user_id является администратором чата."""
    try:
        admins = bot.get_chat_administrators(chat_id)
        return any(a.user.id == user_id for a in admins)
    except Exception:
        return False


def is_admin(user_id, key):
    """True, если user_id - создатель бота и key совпадает с паролем."""
    return user_id == ADMIN_USER_ID and key == ADMIN_KEY


def bot_can_restrict(chat_id):
    """True, если у бота есть право ограничивать/банить участников чата."""
    try:
        me = bot.get_chat_member(chat_id, bot.get_me().id)
        if me.status == "administrator":
            return bool(me.can_restrict_members)
        return False
    except Exception:
        return False


def bot_is_admin(chat_id):
    """True, если бот является администратором чата."""
    try:
        me = bot.get_chat_member(chat_id, bot.get_me().id)
        return me.status in ("administrator", "creator")
    except Exception:
        return False


# ---------- ДЕЙСТВИЯ (/do) ----------
def save_action(action_name):
    """Сохраняет новое действие для команды /do."""
    with sqlite3.connect(DB_PATH) as conn:
        conn.execute(
            "INSERT OR REPLACE INTO actions (action) VALUES (?)", (action_name,)
        )
        conn.commit()


def get_actions():
    """Возвращает список всех действий."""
    with sqlite3.connect(DB_PATH) as conn:
        cur = conn.cursor()
        cur.execute("SELECT action FROM actions")
        return [row[0] for row in cur.fetchall()]


def delete_action(action_name):
    """Удаляет действие. Возвращает True, если удалено."""
    with sqlite3.connect(DB_PATH) as conn:
        cur = conn.execute("DELETE FROM actions WHERE action = ?", (action_name,))
        conn.commit()
        return cur.rowcount > 0


# ---------- ГРУППЫ ----------
def save_group(chat_id, title):
    """Запоминает группу, где присутствует бот."""
    with sqlite3.connect(DB_PATH) as conn:
        conn.execute(
            "INSERT OR REPLACE INTO bot_groups (chat_id, title) VALUES (?, ?)",
            (chat_id, title),
        )
        conn.commit()


def get_all_groups():
    """Возвращает список (chat_id, title) всех запомненных групп."""
    with sqlite3.connect(DB_PATH) as conn:
        cur = conn.cursor()
        cur.execute("SELECT chat_id, title FROM bot_groups")
        return cur.fetchall()


# ---------- ЗАПОМИНАНИЕ ПОЛЬЗОВАТЕЛЕЙ (для бана/ЧС по @username) ----------
def remember_user(user_id, username, first_name):
    """Сохраняет пользователя в таблицу users (ник, имя)."""
    with sqlite3.connect(DB_PATH) as conn:
        conn.execute(
            "INSERT OR REPLACE INTO users (user_id, username, first_name) VALUES (?, ?, ?)",
            (user_id, username or "", first_name or ""),
        )
        conn.commit()


def get_user_by_username(username):
    """Ищет user_id по @username среди запомненных пользователей."""
    with sqlite3.connect(DB_PATH) as conn:
        cur = conn.cursor()
        cur.execute(
            "SELECT user_id FROM users WHERE LOWER(username) = LOWER(?)",
            (username.lstrip("@"),),
        )
        row = cur.fetchone()
    return row[0] if row else None


# ---------- БАН ----------
def ban_user(chat_id, user_id, caller_name):
    """
    Банит пользователя в чате.
    Сначала проверяет права бота, чтобы дать понятную ошибку.
    Возвращает (успех, текст).
    """
    if not bot_can_restrict(chat_id):
        return False, (
            "У бота нет прав банить участников. "
            "Сделайте бота администратором группы и включите право «Банить участников»."
        )
    try:
        bot.ban_chat_member(chat_id, user_id)
        return True, "🔨 " + caller_name + " банит пользователя."
    except AttributeError:
        bot.kick_chat_member(chat_id, user_id)
        return True, "🔨 " + caller_name + " банит пользователя."
    except Exception as e:
        return False, "❌ Не удалось забанить: " + str(e)


def kick_user(chat_id, user_id):
    """Просто кикает пользователя (без бана). Возвращает True при успехе."""
    try:
        bot.kick_chat_member(chat_id, user_id)
        return True
    except Exception as e:
        logging.error("Не удалось кикнуть %s из чата %s: %s", user_id, chat_id, e)
        return False


# ---------- ПРЕДУПРЕЖДЕНИЯ ----------
def add_warning(user_id):
    """Увеличивает счётчик предупреждений и возвращает новое значение."""
    with sqlite3.connect(DB_PATH) as conn:
        cur = conn.cursor()
        cur.execute("SELECT cnt FROM warnings WHERE user_id = ?", (user_id,))
        row = cur.fetchone()
        if row:
            new_cnt = row[0] + 1
            conn.execute("UPDATE warnings SET cnt = ? WHERE user_id = ?", (new_cnt, user_id))
        else:
            new_cnt = 1
            conn.execute("INSERT INTO warnings (user_id, cnt) VALUES (?, ?)", (user_id, new_cnt))
        conn.commit()
        return new_cnt


def get_warnings(user_id):
    """Возвращает текущее количество предупреждений (0, если нет записей)."""
    with sqlite3.connect(DB_PATH) as conn:
        cur = conn.execute("SELECT cnt FROM warnings WHERE user_id = ?", (user_id,))
        row = cur.fetchone()
    return row[0] if row else 0


def reset_warnings(user_id):
    """Обнуляет предупреждения пользователя."""
    with sqlite3.connect(DB_PATH) as conn:
        conn.execute("DELETE FROM warnings WHERE user_id = ?", (user_id,))
        conn.commit()


# ---------- ЧЁРНЫЙ СПИСОК ----------
def is_blacklisted(user_id):
    """True, если пользователь находится в чёрном списке."""
    with sqlite3.connect(DB_PATH) as conn:
        cur = conn.execute("SELECT 1 FROM blacklist WHERE user_id = ?", (user_id,))
        return cur.fetchone() is not None


def add_to_blacklist(user_id):
    """Добавляет пользователя в чёрный список. False, если уже в нём."""
    if is_blacklisted(user_id):
        return False
    with sqlite3.connect(DB_PATH) as conn:
        conn.execute("INSERT INTO blacklist (user_id) VALUES (?)", (user_id,))
        conn.commit()
    return True


def remove_from_blacklist(user_id):
    """Удаляет пользователя из чёрного списка. True, если был удалён."""
    with sqlite3.connect(DB_PATH) as conn:
        cur = conn.execute("DELETE FROM blacklist WHERE user_id = ?", (user_id,))
        conn.commit()
        return cur.rowcount > 0


def get_blacklist():
    """Возвращает список всех user_id, находящихся в чёрном списке."""
    with sqlite3.connect(DB_PATH) as conn:
        cur = conn.execute("SELECT user_id FROM blacklist")
        return [row[0] for row in cur.fetchall()]


# ---------- БАЛАНС UC ----------
def get_uc_balance(user_id):
    """Возвращает текущий баланс UC (0, если записи нет)."""
    with sqlite3.connect(DB_PATH) as conn:
        cur = conn.execute("SELECT balance FROM uc_balance WHERE user_id = ?", (user_id,))
        row = cur.fetchone()
    return row[0] if row else 0


def set_uc_balance(user_id, amount):
    """Устанавливает точный баланс UC пользователя."""
    with sqlite3.connect(DB_PATH) as conn:
        conn.execute(
            "INSERT OR REPLACE INTO uc_balance (user_id, balance) VALUES (?, ?)",
            (user_id, amount),
        )
        conn.commit()


def add_uc(user_id, amount):
    """Прибавляет amount к балансу UC (не даёт уйти в минус)."""
    new_balance = get_uc_balance(user_id) + amount
    if new_balance < 0:
        new_balance = 0
    set_uc_balance(user_id, new_balance)
    return new_balance


def top_uc(limit=10):
    """Возвращает список (user_id, balance), отсортированный по убыванию."""
    with sqlite3.connect(DB_PATH) as conn:
        cur = conn.execute(
            "SELECT user_id, balance FROM uc_balance ORDER BY balance DESC LIMIT ?",
            (limit,),
        )
        return cur.fetchall()


# ---------- ПОМОЩНИК: ОПРЕДЕЛЕНИЕ ЦЕЛИ (для пред/ЧС/бана) ----------
def resolve_target_user(message, chat_id):
    """
    Пытается определить цель команды:
    1) ответ (reply) на сообщение пользователя;
    2) @username в тексте;
    3) числовой user_id в тексте.
    Возвращает объект User или None.
    """
    # 1) Ответ на сообщение
    if message.reply_to_message and message.reply_to_message.from_user:
        return message.reply_to_message.from_user

    # 2) @username
    m = re.search(r"@([A-Za-z0-9_]+)", message.text or "")
    if m:
        uid = get_user_by_username(m.group(1))
        if uid:
            try:
                return bot.get_chat_member(chat_id, uid).user
            except Exception:
                return None

    # 3) Числовой ID (5+ цифр)
    m_id = re.search(r"(\d{5,})", message.text or "")
    if m_id:
        try:
            return bot.get_chat_member(chat_id, int(m_id.group(1))).user
        except Exception:
            return None

    return None


def user_display_name(user) -> str:
    """Красивое имя пользователя для сообщений."""
    return user.first_name or user.username or str(user.id)


# ---------- PUBG API ----------
def get_pubg_nickname(player_id):
    """Возвращает ник PUBG по ID игрока через официальный API."""
    url = f"https://api.pubg.com/shards/{PUBG_REGION}/players/{player_id}"
    headers = {
        "Authorization": f"Bearer {PUBG_API_KEY}",
        "Accept": "application/vnd.api+json",
    }
    try:
        resp = requests.get(url, headers=headers, timeout=10)
        if resp.status_code == 200:
            return resp.json()["data"]["attributes"]["name"]
        logging.warning("PUBG API: HTTP %s", resp.status_code)
        return None
    except Exception as e:
        logging.error("PUBG API error: %s", e)
        return None


# ---------- ПОГОДА (wttr.in, без API-ключа) ----------
def get_weather_text(city: str):
    """
    Запрашивает погоду у wttr.in.
    Возвращает кортеж (текст_для_голоса, текст_для_экран) или None.
    """
    url = f"https://wttr.in/{requests.utils.quote(city)}"
    params = {"format": "j1", "lang": "ru"}
    try:
        resp = requests.get(url, params=params, timeout=10)
        if resp.status_code != 200:
            logging.warning("wttr.in error %s - %s", resp.status_code, resp.text)
            return None
        data = resp.json()
        cur = data["current_condition"][0]
        temp = round(float(cur["temp_C"]))
        desc = cur.get("lang_ru", [{}])[0].get("value") or cur["weatherDesc"][0]["value"]
        city_name = city.capitalize()
        speak = f"В городе {city_name} сейчас {temp} градуса по Цельсию, {desc}."
        display = f"🌤 {city_name}: {temp}°C, {desc}"
        return speak, display
    except Exception as e:
        logging.error("Weather request failed: %s", e)
        return None


def text_to_voice(text: str) -> io.BytesIO:
    """Превращает текст в голосовое сообщение (gTTS, русский)."""
    tts = gTTS(text=text, lang="ru")
    voice_io = io.BytesIO()
    tts.write_to_fp(voice_io)
    voice_io.seek(0)
    return voice_io


# ------------------- КОМАНДА /start -------------------
@bot.message_handler(commands=["start"])
def cmd_start(message):
    """Приветствие. Создателю показывает админ-команды, остальным - базовые."""
    if message.from_user.id == ADMIN_USER_ID:
        bot.reply_to(
            message,
            "👑 Привет, мой создатель! Я готов к работе.\n\n"
            "🧰 <b>Твои админ-возможности:</b>\n"
            "/admin - админ-панель (цены, UC, чёрный список, тех. работы)\n"
            "/check - полная проверка систем\n"
            "/maintenance on|off - режим тех. работ\n"
            "/do <действие> <цель>\n"
            "/lockchat - заблокировать чат\n"
            "/my_groups - список групп\n"
            "/send <chat_id> <текст> - отправка в группу\n"
            "/setmsg <ключ> <текст> - изменить сообщение бота\n"
            "/setadmin <старый> <новый> - сменить пароль админки\n"
            "/adduc <id> <сумма> - пополнить UC\n"
            "/takeuc <id> <сумма> - снять UC\n"
            "/myuc - свой баланс UC\n"
            "/topuc - топ по UC\n\n"
            "🛡️ <b>Команды в группе:</b>\n"
            "Космос бан - банит участника (админы)\n"
            "Космос пред - выдаёт пред (3 преда = кик + ЧС)\n"
            "Космос добавить в черный список (админы)\n"
            "Космос убрать из черного списка (админы)\n"
            "Космос черный список - показать ЧС\n"
            "Космос айди <ID> - ник PUBG\n"
            "Космос какая погода <город> - голосовой прогноз\n"
            "космос - приветствие",
            parse_mode="HTML",
        )
    else:
        bot.reply_to(
            message,
            "👋 Привет! Я бот «Космос».\n"
            "Команды:\n"
            "/price - прайс-лист\n"
            "/player <ID> - ник игрока\n"
            "/myuc - мой баланс UC\n"
            "/admins - админы группы\n"
            "Космос айди <ID> - ник PUBG\n"
            "Космос какая погода <город> - голосовой прогноз\n"
            "космос - приветствие",
        )


# ------------------- КОМАНДА /help -------------------
@bot.message_handler(commands=["help"])
def cmd_help(message):
    """Справка по всем командам бота."""
    if message.from_user.id == ADMIN_USER_ID:
        text = (
            "📖 <b>Справка по боту «Космос»</b>\n\n"
            "<b>Базовые команды:</b>\n"
            "/start - приветствие\n"
            "/help - эта справка\n"
            "/price - прайс-лист\n"
            "/player <ID> - ник игрока\n"
            "/myuc - мой баланс UC\n"
            "/topuc - топ по UC\n"
            "/admins - админы группы\n\n"
            "<b>Команды создателя:</b>\n"
            "/admin - админ-панель\n"
            "/check - полная диагностика всех систем\n"
            "/maintenance on|off - тех. работы\n"
            "/do <действие> <цель> - действие из списка\n"
            "/lockchat - блокировка чата\n"
            "/my_groups - список групп\n"
            "/send <chat_id> <текст> - отправка в группу\n"
            "/setmsg <ключ> <текст> - изменить сообщение\n"
            "/listmsg - показать изменённые сообщения\n"
            "/setadmin <старый> <новый> - смена пароля\n"
            "/adduc <id> <сумма> - пополнить UC\n"
            "/takeuc <id> <сумма> - снять UC\n\n"
            "<b>Команды в группе (админы группы):</b>\n"
            "Космос бан @ник/ID - бан\n"
            "Космос пред @ник/ID - предупреждение\n"
            "Космос добавить в черный список - добавить в ЧС + кик\n"
            "Космос убрать из черного списка - убрать из ЧС\n\n"
            "<b>Команды для всех:</b>\n"
            "Космос черный список - просмотр ЧС (только создатель)\n"
            "Космос айди <ID> - ник PUBG\n"
            "Космос какая погода <город> - голосовой прогноз\n"
            "космос - приветствие\n\n"
            "💡 Как указать цель: ответьте на сообщение, "
            "напишите @username или цифровой user_id."
        )
    else:
        text = (
            "📖 <b>Справка по боту «Космос»</b>\n\n"
            "/price - прайс-лист\n"
            "/player <ID> - ник игрока\n"
            "/myuc - мой баланс UC\n"
            "/topuc - топ UC\n"
            "/admins - админы группы\n\n"
            "Также работают текстовые команды:\n"
            "космос - приветствие\n"
            "Космос айди <ID> - ник PUBG\n"
            "Космос какая погода <город> - погода голосом"
        )
    bot.reply_to(message, text, parse_mode="HTML")


# ------------------- КОМАНДА /price -------------------
@bot.message_handler(commands=["price"])
def cmd_price(message):
    """Показывает прайс-лист UC."""
    prices = load_prices()
    if not prices:
        bot.reply_to(message, messages.get("price_empty", "⚠️ Прайс-лист пуст. Добавьте через /admin."))
        return
    lines = [f"{item}: {price}" for item, price in prices.items()]
    bot.reply_to(message, messages.get("price_header", "📊 Прайс-лист UC:") + "\n" + "\n".join(lines))


# ------------------- КОМАНДА /player -------------------
@bot.message_handler(commands=["player"])
def cmd_player(message):
    """Показывает ник игрока по ID из players.csv."""
    parts = message.text.split()
    if len(parts) < 2:
        bot.reply_to(message, "❗ Пример: /player 12345")
        return
    nick = get_nickname(parts[1])
    bot.reply_to(message, f"🧑💻 {nick}" if nick else messages.get("player_not_found", "⚠️ Игрок не найден."))


# ------------------- КОМАНДА /do -------------------
@bot.message_handler(commands=["do"])
def cmd_do(message):
    """Выполняет действие из списка: /do <действие> <цель> (создатель)."""
    if message.from_user.id != ADMIN_USER_ID:
        bot.reply_to(message, "🚫 Эта команда только для создателя бота.")
        return
    parts = message.text.split(maxsplit=2)
    if len(parts) < 3:
        bot.reply_to(
            message,
            "❗ Формат: /do <действие> <цель>\n"
            "Доступные действия: " + (", ".join(get_actions()) or "(пока пусто, добавьте через /admin)"),
        )
        return
    action = parts[1].strip().lower()
    target = parts[2].strip()
    if action not in get_actions():
        bot.reply_to(message, f"❌ Действие «{action}» не найдено.")
        return
    nick = message.from_user.first_name or message.from_user.username or "Кирюша"
    bot.send_message(message.chat.id, f"{nick} {action} {target}")


# ------------------- СПИСОК ГРУПП -------------------
@bot.message_handler(commands=["my_groups"])
def cmd_my_groups(message):
    """Показывает все группы, где бот запомнил себя (создатель)."""
    if message.from_user.id != ADMIN_USER_ID:
        bot.reply_to(message, "🚫 Эта команда только для создателя бота.")
        return
    groups = get_all_groups()
    if not groups:
        bot.reply_to(message, "ℹ️ Бот пока не запомнил ни одной группы.")
        return
    lines = [f"🏷 {title or 'Без названия'}\n   ID: {chat_id}" for chat_id, title in groups]
    bot.reply_to(message, "📋 Группы, где я есть:\n\n" + "\n\n".join(lines))


# ------------------- ОТПРАВКА В ГРУППУ -------------------
@bot.message_handler(commands=["send"])
def cmd_send(message):
    """Отправляет сообщение в указанный чат (создатель)."""
    if message.from_user.id != ADMIN_USER_ID:
        bot.reply_to(message, "🚫 Эта команда только для создателя бота.")
        return
    parts = message.text.split(maxsplit=2)
    if len(parts) < 3:
        bot.reply_to(message, "❗ Формат: /send <chat_id> <текст>\nУзнать chat_id: /my_groups")
        return
    try:
        chat_id = int(parts[1])
    except ValueError:
        bot.reply_to(message, "❗ chat_id должен быть числом.")
        return
    try:
        bot.send_message(chat_id, parts[2].strip())
        bot.reply_to(message, f"✅ Сообщение отправлено в чат {chat_id}.")
    except Exception as e:
        bot.reply_to(message, f"❌ Не удалось отправить: {e}")


# ------------------- АДМИНЫ ГРУППЫ -------------------
@bot.message_handler(commands=["admins"])
def cmd_admins(message):
    """Показывает список администраторов группы."""
    try:
        admins = bot.get_chat_administrators(message.chat.id)
    except Exception:
        bot.reply_to(message, "⚠️ Не удалось получить список админов.")
        return
    names = []
    for a in admins:
        u = a.user
        name = u.first_name or u.username or str(u.id)
        if u.username:
            name = "@" + u.username
        names.append(name)
    bot.reply_to(message, "🛡️ Админы группы:\n" + "\n".join(names))


# ------------------- БЛОКИРОВКА ЧАТА -------------------
@bot.message_handler(commands=["lockchat"])
def cmd_lockchat(message):
    """Включает/выключает режим «только админы» в группе."""
    if message.chat.type not in ("group", "supergroup"):
        bot.reply_to(message, "❗ Команда работает только в группах.")
        return
    if not is_group_admin(message.chat.id, message.from_user.id) and message.from_user.id != ADMIN_USER_ID:
        bot.reply_to(message, "🚫 Только админы группы могут блокировать чат.")
        return
    locked = is_chat_locked(message.chat.id)
    set_chat_lock(message.chat.id, not locked)
    state = "🔒 заблокирован (писать могут только админы)" if not locked else "🔓 разблокирован"
    bot.reply_to(message, f"Чат {state}.")


# ------------------- АДМИН-ПАНЕЛЬ -------------------
@bot.message_handler(commands=["admin"])
def cmd_admin(message):
    """Открывает админ-панель (только создатель)."""
    if message.from_user.id != ADMIN_USER_ID:
        bot.reply_to(message, "🚫 Эта команда только для создателя бота.")
        return
    markup = types.InlineKeyboardMarkup()
    markup.add(types.InlineKeyboardButton("📈 Добавить цену", callback_data="add_price"))
    markup.add(types.InlineKeyboardButton("🗑️ Удалить цену", callback_data="del_price"))
    markup.add(types.InlineKeyboardButton("🔐 Сменить ключ", callback_data="set_key"))
    markup.add(types.InlineKeyboardButton("📄 Показать цены", callback_data="show_prices"))
    markup.add(types.InlineKeyboardButton("➕ Добавить действие", callback_data="add_action"))
    markup.add(types.InlineKeyboardButton("🗑️ Удалить действие", callback_data="del_action"))
    markup.add(types.InlineKeyboardButton("📜 Список действий", callback_data="list_actions"))
    markup.add(types.InlineKeyboardButton("💰 Добавить UC", callback_data="add_uc"))
    markup.add(types.InlineKeyboardButton("💸 Снять UC", callback_data="take_uc"))
    markup.add(types.InlineKeyboardButton("📊 Топ UC", callback_data="top_uc"))
    markup.add(types.InlineKeyboardButton("🖤 Показать ЧС", callback_data="show_blacklist"))
    markup.add(types.InlineKeyboardButton("🚧 Технические работы", callback_data="toggle_maintenance"))
    state = "ВКЛ" if MAINTENANCE else "ВЫКЛ"
    bot.reply_to(message, f"🛠️ Админ-панель:\n🚧 Тех. работы: {state}", reply_markup=markup)


@bot.callback_query_handler(func=lambda call: True)
def callback_handler(call):
    """Обрабатывает нажатия кнопок админ-панели (только создатель)."""
    if call.from_user.id != ADMIN_USER_ID:
        bot.answer_callback_query(call.id, "🚫 Не ваш запрос.")
        return

    if call.data == "toggle_maintenance":
        global MAINTENANCE
        MAINTENANCE = not MAINTENANCE
        state = "ВКЛЮЧЁН" if MAINTENANCE else "ВЫКЛЮЧЕН"
        bot.send_message(
            call.message.chat.id,
            f"🚧 Режим технических работ {state}.\n"
            + ("Бот отвечает «В разработке» всем, кроме тебя." if MAINTENANCE else "Бот снова полностью работает."),
        )

    elif call.data == "add_price":
        msg = bot.send_message(call.message.chat.id, "Введите название и цену через пробел.\nПример: item1 100₽")
        bot.register_next_step_handler(msg, step_add_price)

    elif call.data == "del_price":
        msg = bot.send_message(call.message.chat.id, "Введите название позиции для удаления.\nПример: item1")
        bot.register_next_step_handler(msg, step_del_price)

    elif call.data == "set_key":
        msg = bot.send_message(call.message.chat.id, "Введите старый ключ и новый через пробел.\nПример: trial_password_12345 myNewKey456")
        bot.register_next_step_handler(msg, step_set_key)

    elif call.data == "show_prices":
        prices = load_prices()
        if not prices:
            bot.send_message(call.message.chat.id, "⚠️ Прайс-лист пуст.")
        else:
            lines = [f"{item}: {price}" for item, price in prices.items()]
            bot.send_message(call.message.chat.id, "📊 Прайс-лист UC:\n" + "\n".join(lines))

    elif call.data == "add_action":
        msg = bot.send_message(call.message.chat.id, "Введите действие (в форме вывода).\nПример: выебал, поцеловал, обнял")
        bot.register_next_step_handler(msg, step_add_action)

    elif call.data == "del_action":
        msg = bot.send_message(call.message.chat.id, "Введите действие, которое нужно удалить.\nДоступны: " + ", ".join(get_actions()))
        bot.register_next_step_handler(msg, step_del_action)

    elif call.data == "list_actions":
        actions = get_actions()
        if not actions:
            bot.send_message(call.message.chat.id, "📭 Действий пока нет.")
        else:
            bot.send_message(call.message.chat.id, "📜 Действия:\n" + "\n".join(actions))

    elif call.data == "add_uc":
        msg = bot.send_message(call.message.chat.id, "Введите: <user_id> <сумма>\nПример: 12345678 5000")
        bot.register_next_step_handler(msg, step_add_uc)

    elif call.data == "take_uc":
        msg = bot.send_message(call.message.chat.id, "Введите: <user_id> <сумма>\nПример: 12345678 2000")
        bot.register_next_step_handler(msg, step_take_uc)

    elif call.data == "top_uc":
        rows = top_uc(10)
        if not rows:
            bot.send_message(call.message.chat.id, "📊 Пока никто не имеет UC.")
        else:
            lines = []
            for i, (uid, bal) in enumerate(rows, start=1):
                try:
                    user = bot.get_chat_member(call.message.chat.id, uid).user
                    name = user.first_name or user.username or str(uid)
                except Exception:
                    name = str(uid)
                lines.append(f"{i}. {name}: {bal} UC")
            bot.send_message(call.message.chat.id, "🏆 Топ-10 по UC:\n" + "\n".join(lines))

    elif call.data == "show_blacklist":
        bl_ids = get_blacklist()
        if not bl_ids:
            bot.send_message(call.message.chat.id, "🖤 Чёрный список пуст.")
        else:
            lines = []
            for uid in bl_ids:
                try:
                    user = bot.get_chat_member(call.message.chat.id, uid).user
                    name = user.first_name or user.username or str(uid)
                except Exception:
                    name = str(uid)
                lines.append(f"• {name} (ID: {uid})")
            bot.send_message(call.message.chat.id, "🖤 Чёрный список:\n" + "\n".join(lines))

    bot.answer_callback_query(call.id)


# ------------------- ШАГИ ДИАЛОГА (next_step_handler) -------------------
def step_add_price(message):
    """Шаг: принять название и цену."""
    parts = message.text.split(maxsplit=1)
    if len(parts) != 2:
        bot.reply_to(message, "❗ Формат: название цена")
        return
    save_price(parts[0], parts[1])
    bot.reply_to(message, f"✅ {parts[0]} -> {parts[1]} сохранено.")


def step_del_price(message):
    """Шаг: принять название позиции для удаления."""
    item = message.text.strip()
    if delete_price(item):
        bot.reply_to(message, f"🗑️ {item} удалён.")
    else:
        bot.reply_to(message, f"⚠️ {item} не найден.")


def step_set_key(message):
    """Шаг: принять старый и новый пароли админки."""
    global ADMIN_KEY
    parts = message.text.split(maxsplit=1)
    if len(parts) != 2:
        bot.reply_to(message, "❗ Нужно два ключа: старый новый")
        return
    old_key, new_key = parts
    if not is_admin(message.from_user.id, old_key):
        bot.reply_to(message, "🚫 Неверный текущий ключ.")
        return
    if save_admin_key(new_key):
        ADMIN_KEY = new_key
        bot.reply_to(message, f"🔐 Ключ изменён на: {new_key}")
    else:
        bot.reply_to(message, "❌ Не удалось сохранить новый ключ.")


def step_add_action(message):
    """Шаг: принять новое действие."""
    action = message.text.strip().lower()
    if not action:
        bot.reply_to(message, "❗ Введите действие.")
        return
    save_action(action)
    bot.reply_to(message, f"✅ Действие «{action}» добавлено.\nТеперь используйте: /do {action} <цель>")


def step_del_action(message):
    """Шаг: принять действие для удаления."""
    action = message.text.strip().lower()
    if not action:
        bot.reply_to(message, "❗ Введите действие.")
        return
    if delete_action(action):
        bot.reply_to(message, f"🗑️ Действие «{action}» удалено.")
    else:
        bot.reply_to(message, f"⚠️ Действие «{action}» не найдено.")


def step_add_uc(message):
    """Шаг: начислить UC (user_id сумма)."""
    parts = message.text.split()
    if len(parts) != 2:
        bot.reply_to(message, "❗ Формат: <user_id> <сумма>")
        return
    try:
        uid = int(parts[0])
        amount = int(parts[1])
    except ValueError:
        bot.reply_to(message, "❗ Оба параметра должны быть целыми числами.")
        return
    if amount <= 0:
        bot.reply_to(message, "❗ Сумма должна быть положительной.")
        return
    new_bal = add_uc(uid, amount)
    bot.reply_to(message, f"✅ {amount} UC начислено пользователю {uid}. Баланс: {new_bal} UC.")


def step_take_uc(message):
    """Шаг: снять UC (user_id сумма)."""
    parts = message.text.split()
    if len(parts) != 2:
        bot.reply_to(message, "❗ Формат: <user_id> <сумма>")
        return
    try:
        uid = int(parts[0])
        amount = int(parts[1])
    except ValueError:
        bot.reply_to(message, "❗ Оба параметра должны быть целыми числами.")
        return
    if amount <= 0:
        bot.reply_to(message, "❗ Сумма должна быть положительной.")
        return
    cur = get_uc_balance(uid)
    if cur < amount:
        bot.reply_to(message, f"⚠️ У пользователя {uid} только {cur} UC.")
        return
    new_bal = add_uc(uid, -amount)
    bot.reply_to(message, f"✅ Снято {amount} UC у {uid}. Баланс: {new_bal} UC.")


# ------------------- КОМАНДЫ UC -------------------
@bot.message_handler(commands=["adduc"])
def cmd_adduc(message):
    """Пополняет UC пользователю (только создатель)."""
    if message.from_user.id != ADMIN_USER_ID:
        bot.reply_to(message, "🚫 Только создатель бота может пополнять UC.")
        return
    parts = message.text.split()
    if len(parts) != 3:
        bot.reply_to(message, "❗ Формат: /adduc <user_id> <сумма>")
        return
    try:
        target_id = int(parts[1])
        amount = int(parts[2])
    except ValueError:
        bot.reply_to(message, "❗ user_id и сумма должны быть целыми числами.")
        return
    if amount <= 0:
        bot.reply_to(message, "❗ Сумма должна быть положительной.")
        return
    new_balance = add_uc(target_id, amount)
    bot.reply_to(message, f"✅ Пользователю {target_id} начислено {amount} UC. Новый баланс: {new_balance} UC.")


@bot.message_handler(commands=["takeuc"])
def cmd_takeuc(message):
    """Снимает UC у пользователя (только создатель)."""
    if message.from_user.id != ADMIN_USER_ID:
        bot.reply_to(message, "🚫 Только создатель бота может снимать UC.")
        return
    parts = message.text.split()
    if len(parts) != 3:
        bot.reply_to(message, "❗ Формат: /takeuc <user_id> <сумма>")
        return
    try:
        target_id = int(parts[1])
        amount = int(parts[2])
    except ValueError:
        bot.reply_to(message, "❗ user_id и сумма должны быть целыми числами.")
        return
    if amount <= 0:
        bot.reply_to(message, "❗ Сумма должна быть положительной.")
        return
    current = get_uc_balance(target_id)
    if current < amount:
        bot.reply_to(message, f"⚠️ У пользователя {target_id} всего {current} UC, снять нельзя.")
        return
    new_balance = add_uc(target_id, -amount)
    bot.reply_to(message, f"✅ Снято {amount} UC у {target_id}. Новый баланс: {new_balance} UC.")


@bot.message_handler(commands=["myuc"])
def cmd_myuc(message):
    """Показывает свой баланс UC."""
    bal = get_uc_balance(message.from_user.id)
    bot.reply_to(message, f"💰 У тебя {bal} UC.")


@bot.message_handler(commands=["topuc"])
def cmd_topuc(message):
    """Показывает топ-10 по UC."""
    rows = top_uc(10)
    if not rows:
        bot.reply_to(message, "📊 Пока никто не имеет UC.")
        return
    lines = []
    for i, (uid, bal) in enumerate(rows, start=1):
        try:
            user = bot.get_chat_member(message.chat.id, uid).user
            name = user.first_name or user.username or str(uid)
        except Exception:
            name = str(uid)
        lines.append(f"{i}. {name}: {bal} UC")
    bot.reply_to(message, "🏆 Топ-10 по UC:\n" + "\n".join(lines))


# ------------------- КОМАНДА «КОСМОС АЙДИ» -------------------
@bot.message_handler(func=lambda m: "космос айди" in (m.text or "").lower())
def cmd_kosmos_id(message):
    """Ищет ник PUBG по ID игрока через официальный API."""
    parts = (message.text or "").split(maxsplit=2)
    if len(parts) < 2:
        bot.reply_to(message, "❗ Формат: Космос айди <PUBG player ID>")
        return
    player_id = parts[1].strip()
    nickname = get_pubg_nickname(player_id)
    if nickname:
        bot.reply_to(message, f"🔎 Игрок с ID {player_id} → Ник: {nickname}")
    else:
        bot.reply_to(message, f"⚠️ Не удалось найти ник по ID {player_id}.\nРегион сейчас: {PUBG_REGION}.")


# ------------------- КОМАНДА «КОСМОС КАКАЯ ПОГОДА <город>» -------------------
@bot.message_handler(func=lambda m: "космос какая погода" in (m.text or "").lower())
def cmd_kosmos_weather(message):
    """Отправляет прогноз погоды текстом и голосом."""
    pattern = re.compile(r"космос\s+какая\s+погода\s*(.*)", re.IGNORECASE)
    m = pattern.search(message.text or "")
    city = m.group(1).strip() if m and m.group(1).strip() else ""
    if not city:
        bot.reply_to(message, "❗ Формат: Космос какая погода <город>")
        return
    result = get_weather_text(city)
    if not result:
        bot.reply_to(message, f"⚠️ Не удалось получить погоду для «{city}». Проверьте название города.")
        return
    speak, display = result
    try:
        voice_io = text_to_voice(speak)
        bot.send_voice(message.chat.id, voice=voice_io)
    except Exception as e:
        logging.error("Не удалось отправить голос: %s", e)
        bot.reply_to(message, display)
        return
    bot.send_message(message.chat.id, display)


# ------------------- КОМАНДА «КОСМОС» (приветствие) -------------------
@bot.message_handler(func=lambda m: re.fullmatch(r"космос[!.]*", (m.text or "").strip().lower()))
def cmd_kosmos_greeting(message):
    """Простое приветствие по слову «космос»."""
    if message.from_user.id == ADMIN_USER_ID:
        bot.reply_to(message, "Я тут господин 👑")
    else:
        bot.reply_to(message, "Космос тут ✅")


# ------------------- НОВЫЙ УЧАСТНИК / УХОД -------------------
@bot.message_handler(content_types=["new_chat_members"])
def on_new_members(message):
    """
    Обрабатывает вступление новых участников.
    Если участник в чёрном списке - мгновенно кикается.
    """
    chat_id = message.chat.id
    title = message.chat.title or ""

    for member in message.new_chat_members:
        # Сам бот добавился - просто запоминаем группу
        if member.is_bot and member.id == bot.get_me().id:
            save_group(chat_id, title)
            continue

        # Участник в чёрном списке - кикаем сразу
        if is_blacklisted(member.id):
            logging.info("ЧС-участник %s попытался войти в чат %s", member.id, chat_id)
            try:
                bot.send_message(
                    chat_id,
                    f"🚫 {member.first_name or 'Пользователь'} в чёрном списке и был удалён.",
                )
            except Exception as e:
                logging.error("Не удалось отправить уведомление: %s", e)
            kick_user(chat_id, member.id)
            continue

        # Запоминаем обычного нового участника
        remember_user(member.id, member.username, member.first_name)
        try:
            welcome = messages.get("welcome_text", WELCOME_TEXT).format(name=member.first_name or "друг")
            bot.send_message(chat_id, welcome)
        except Exception as e:
            logging.error("Не удалось отправить приветствие: %s", e)


@bot.message_handler(content_types=["left_chat_member"])
def on_left_member(message):
    """Обрабатывает выход участника из группы."""
    user = message.left_chat_member
    if user and not user.is_bot:
        logging.info("Пользователь %s покинул чат %s", user.id, message.chat.id)
        try:
            goodbye = messages.get("goodbye_text", GOODBYE_TEXT).format(name=user.first_name or "друг")
            bot.send_message(message.chat.id, goodbye)
        except Exception as e:
            logging.error("Не удалось отправить прощание: %s", e)


# ------------------- РЕЖИМ ТЕХНИЧЕСКИХ РАБОТ (команда) -------------------
@bot.message_handler(commands=["maintenance"])
def cmd_maintenance(message):
    """/maintenance on|off - включить/выключить режим тех. работ (создатель)."""
    global MAINTENANCE
    if message.from_user.id != ADMIN_USER_ID:
        bot.reply_to(message, "🚫 Только создатель может управлять тех. работами.")
        return
    parts = message.text.split(maxsplit=1)
    if len(parts) == 1:
        state = "ВКЛЮЧЁН" if MAINTENANCE else "ВЫКЛЮЧЕН"
        bot.reply_to(message, f"🚧 Режим технических работ: {state}.")
        return
    arg = parts[1].strip().lower()
    if arg in ("on", "вкл", "1", "да"):
        MAINTENANCE = True
        bot.reply_to(message, "🚧 Режим технических работ ВКЛЮЧЁН. Бот отвечает «В разработке» всем, кроме тебя.")
    elif arg in ("off", "выкл", "0", "нет"):
        MAINTENANCE = False
        bot.reply_to(message, "✅ Режим технических работ ВЫКЛЮЧЕН. Бот снова работает.")
    else:
        bot.reply_to(message, "❗ Формат: /maintenance on | off")


# ------------------- СМЕНА ПАРОЛЯ АДМИНКИ -------------------
@bot.message_handler(commands=["setadmin"])
def cmd_setadmin(message):
    """/setadmin <старый_ключ> <новый_ключ> - смена пароля без перезапуска."""
    global ADMIN_KEY
    if message.from_user.id != ADMIN_USER_ID:
        bot.reply_to(message, "🚫 Только создатель может менять пароль.")
        return
    parts = message.text.split(maxsplit=2)
    if len(parts) != 3:
        bot.reply_to(message, "❗ Формат: /setadmin <старый_ключ> <новый_ключ>")
        return
    old_key, new_key = parts[1], parts[2]
    if not is_admin(message.from_user.id, old_key):
        bot.reply_to(message, "🚫 Неверный текущий ключ.")
        return
    if save_admin_key(new_key):
        ADMIN_KEY = new_key
        bot.reply_to(message, f"✅ Пароль изменён на: {new_key}")
    else:
        bot.reply_to(message, "❌ Ошибка при записи нового пароля.")


# ------------------- РЕДАКТИРОВАНИЕ СООБЩЕНИЙ -------------------
@bot.message_handler(commands=["setmsg"])
def cmd_setmsg(message):
    """/setmsg <ключ> <текст> - изменить любое сообщение бота (создатель)."""
    if message.from_user.id != ADMIN_USER_ID:
        bot.reply_to(message, "🚫 Только создатель может менять сообщения.")
        return
    parts = message.text.split(maxsplit=2)
    if len(parts) < 3:
        bot.reply_to(
            message,
            "❗ Формат: /setmsg <ключ> <текст>\n"
            "Ключи: price_empty, price_header, player_not_found,\n"
            "greeting_user, weather_error, maintenance_text,\n"
            "welcome_text, goodbye_text",
        )
        return
    key, new_text = parts[1], parts[2]
    messages[key] = new_text
    save_messages(messages)
    bot.reply_to(message, f"✅ Сообщение «{key}» обновлено.")


@bot.message_handler(commands=["listmsg"])
def cmd_listmsg(message):
    """Показывает все переопределённые сообщения (создатель)."""
    if message.from_user.id != ADMIN_USER_ID:
        bot.reply_to(message, "🚫 Только создатель.")
        return
    if not messages:
        bot.reply_to(message, "📭 Пока нет изменённых сообщений.")
        return
    lines = [f"{k}: {v}" for k, v in messages.items()]
    bot.reply_to(message, "📝 Сообщения:\n" + "\n".join(lines))


# ------------------- КОМАНДА «КОСМОС ПРЕД» -------------------
@bot.message_handler(func=lambda m: re.search(r"космос\s+пред", (m.text or ""), re.IGNORECASE))
def cmd_kosmos_pred(message):
    """
    Выдаёт предупреждение (только админы группы или создатель).
    После MAX_WARNINGS предупреждений - кик + чёрный список.
    """
    chat_id = message.chat.id
    admin_id = message.from_user.id

    # Проверка прав: админ группы или создатель бота
    if not is_group_admin(chat_id, admin_id) and admin_id != ADMIN_USER_ID:
        bot.reply_to(message, "🚫 Только админы группы могут выдавать предупреждения.")
        return

    # Определяем цель
    target_user = resolve_target_user(message, chat_id)
    if not target_user:
        bot.reply_to(
            message,
            "❗ Не удалось определить, кого предупреждать.\n"
            "Ответьте на сообщение нарушителя, укажите @username или ID.",
        )
        return

    # Нельзя выдавать пред админу группы или создателю
    if target_user.id == ADMIN_USER_ID:
        bot.reply_to(message, "🚫 Нельзя выдавать предупреждение создателю бота.")
        return
    if is_group_admin(chat_id, target_user.id):
        bot.reply_to(message, "🚫 Нельзя выдавать предупреждение администратору группы.")
        return

    new_cnt = add_warning(target_user.id)
    name = user_display_name(target_user)
    bot.reply_to(message, f"⚠️ {name}, тебе выдано предупреждение №{new_cnt} из {MAX_WARNINGS}.")

    # Достигнут лимит - кикаем и заносим в чёрный список
    if new_cnt >= MAX_WARNINGS:
        caller = message.from_user.first_name or message.from_user.username or "Админ"
        ok, txt = ban_user(chat_id, target_user.id, caller + f" ({MAX_WARNINGS} преда)")
        add_to_blacklist(target_user.id)
        reset_warnings(target_user.id)
        if ok:
            bot.reply_to(message, f"🚫 {txt} Пользователь {name} добавлен в чёрный список.")
        else:
            bot.reply_to(message, f"🖤 {name} добавлен в чёрный список (кикнуть не удалось: {txt})")


# ------------------- КОМАНДА «КОСМОС ДОБАВИТЬ В ЧЕРНЫЙ СПИСОК» -------------------
@bot.message_handler(func=lambda m: re.search(r"космос\s+добавить\s+в\s+черный\s+список", (m.text or ""), re.IGNORECASE))
def cmd_kosmos_blacklist_add(message):
    """
    Добавляет пользователя в чёрный список и кикает его (только админы группы/создатель).
    Формат: «Космос добавить в черный список» (ответом), либо с @username или ID.
    """
    chat_id = message.chat.id
    admin_id = message.from_user.id

    # Только админ группы или создатель
    if admin_id != ADMIN_USER_ID and not is_group_admin(chat_id, admin_id):
        bot.reply_to(message, "🚫 Только админы группы могут добавлять в чёрный список.")
        return

    # Определяем цель
    target_user = resolve_target_user(message, chat_id)
    if not target_user:
        bot.reply_to(
            message,
            "❗ Не удалось определить, кого добавить.\n"
            "Ответьте на сообщение пользователя или укажите @username / ID.",
        )
        return

    # Защита: нельзя добавить создателя или админа группы
    if target_user.id == ADMIN_USER_ID:
        bot.reply_to(message, "🚫 Нельзя добавить создателя бота в чёрный список.")
        return
    if is_group_admin(chat_id, target_user.id):
        bot.reply_to(message, "🚫 Нельзя добавить администратора группы в чёрный список.")
        return

    name = user_display_name(target_user)
    added = add_to_blacklist(target_user.id)

    if added:
        # Кикаем человека из группы
        if bot_can_restrict(chat_id):
            kick_user(chat_id, target_user.id)
            bot.reply_to(message, f"🖤 {name} добавлен(а) в чёрный список и удалён(а) из группы.")
        else:
            bot.reply_to(
                message,
                f"🖤 {name} добавлен(а) в чёрный список.\n"
                "⚠️ У бота нет прав банить - сделайте бота админом с правом «Банить участников», "
                "иначе человек сможет вернуться.",
            )
    else:
        bot.reply_to(message, f"ℹ️ {name} уже в чёрном списке.")


# ------------------- КОМАНДА «КОСМОС УБРАТЬ ИЗ ЧЕРНОГО СПИСКА» -------------------
@bot.message_handler(func=lambda m: re.search(r"космос\s+убрать\s+из\s+черного\s+списка", (m.text or ""), re.IGNORECASE))
def cmd_kosmos_blacklist_remove(message):
    """
    Убирает пользователя из чёрного списка (только админы группы/создатель).
    Можно указать ID даже если человек уже не в группе.
    """
    chat_id = message.chat.id
    admin_id = message.from_user.id

    # Только админ группы или создатель
    if admin_id != ADMIN_USER_ID and not is_group_admin(chat_id, admin_id):
        bot.reply_to(message, "🚫 Только админы группы могут убирать из чёрного списка.")
        return

    # Сначала пробуем определить цель (reply/@username/ID в группе)
    target_user = resolve_target_user(message, chat_id)

    if target_user:
        uid = target_user.id
        name = user_display_name(target_user)
    else:
        # Иначе ищем чистый ID в тексте (даже если человек вышел из группы)
        m_id = re.search(r"(\d{5,})", message.text or "")
        if not m_id:
            bot.reply_to(
                message,
                "❗ Не удалось определить, кого убрать.\n"
                "Ответьте на сообщение, укажите @username или ID пользователя.",
            )
            return
        uid = int(m_id.group(1))
        name = str(uid)

    removed = remove_from_blacklist(uid)
    if removed:
        bot.reply_to(message, f"✅ {name} убран(а) из чёрного списка.")
    else:
        bot.reply_to(message, f"ℹ️ {name} не был(а) в чёрном списке.")


# ------------------- КОМАНДА «КОСМОС ЧЕРНЫЙ СПИСОК» -------------------
@bot.message_handler(func=lambda m: re.search(r"космос\s+черный\s+список", (m.text or ""), re.IGNORECASE))
def cmd_kosmos_blacklist_show(message):
    """Показывает чёрный список (только создатель бота)."""
    if message.from_user.id != ADMIN_USER_ID:
        bot.reply_to(message, "🚫 Эта команда доступна только создателю бота.")
        return
    bl_ids = get_blacklist()
    if not bl_ids:
        bot.reply_to(message, "🖤 Чёрный список пуст.")
        return
    lines = []
    for uid in bl_ids:
        try:
            user = bot.get_chat_member(message.chat.id, uid).user
            name = user.first_name or user.username or str(uid)
        except Exception:
            name = str(uid)
        lines.append(f"• {name} (ID: {uid})")
    bot.reply_to(message, "🖤 Чёрный список:\n" + "\n".join(lines))


# ====================== СИСТЕМА ДИАГНОСТИКИ (для господина) ======================
# Запуск: /check  или  «Космос проверка»  (только создатель бота)

SPINNER_FRAMES = ["⏳", "⏳.", "⏳..", "⏳..."]


def _run_step(fn):
    """Выполняет один шаг диагностики.
    Возвращает ('ok'|'warn'|'fail', пояснение)."""
    try:
        note = fn()
        if note:
            return "warn", note
        return "ok", None
    except Exception as e:
        return "fail", f"{type(e).__name__}: {e}"


def check_database():
    init_db()


def check_prices():
    save_price("__test_price__", "100")
    assert load_prices().get("__test_price__") == "100", "цена не сохранилась"
    save_price("__test_price__", "150")
    assert load_prices().get("__test_price__") == "150", "цена не обновилась"
    assert delete_price("__test_price__") is True, "цена не удалилась"
    assert delete_price("__test_price__") is False, "повторное удаление цены"


def check_actions():
    save_action("__test_action__")
    assert "__test_action__" in get_actions(), "действие не сохранилось"
    assert delete_action("__test_action__") is True, "действие не удалилось"
    assert delete_action("__test_action__") is False, "повторное удаление действия"


def check_permissions():
    assert is_admin(ADMIN_USER_ID, ADMIN_KEY), "права господина не работают"
    assert not is_admin(123456, ADMIN_KEY), "посторонний получил права"


def check_chat_lock():
    set_chat_lock(999001, True)
    assert is_chat_locked(999001) is True, "блокировка не включилась"
    set_chat_lock(999001, False)
    assert is_chat_locked(999001) is False, "блокировка не выключилась"


def check_msg_counter():
    inc_msg_counter(999002, 999003)
    assert get_msg_count(999002, 999003) >= 1, "счётчик не работает"


def check_warnings():
    uid = 999000111
    reset_warnings(uid)
    assert get_warnings(uid) == 0, "преды не обнулились"
    assert add_warning(uid) == 1, "1-й пред"
    assert add_warning(uid) == 2, "2-й пред"
    assert add_warning(uid) == 3, "3-й пред"
    reset_warnings(uid)


def check_blacklist():
    uid = 999000111
    remove_from_blacklist(uid)
    assert not is_blacklisted(uid), "ЧС не очищен"
    assert add_to_blacklist(uid) is True, "добавление в ЧС"
    assert is_blacklisted(uid), "проверка ЧС"
    assert add_to_blacklist(uid) is False, "двойное добавление в ЧС"
    assert remove_from_blacklist(uid) is True, "удаление из ЧС"


def check_uc():
    uid = 999000111
    set_uc_balance(uid, 0)
    assert get_uc_balance(uid) == 0, "баланс не обнулился"
    add_uc(uid, 1000)
    assert get_uc_balance(uid) == 1000, "начисление UC"
    add_uc(uid, -500)
    assert get_uc_balance(uid) == 500, "снятие UC"
    add_uc(uid, -99999)
    assert get_uc_balance(uid) == 0, "баланс ушёл в минус"
    set_uc_balance(uid, 0)


def check_pubg():
    if get_pubg_nickname("00000000000000000000000000000000") is None:
        return "API недоступен (нет сети?)"


def check_weather():
    if get_weather_text("Москва") is None:
        return "Сервис погоды недоступен"


def _clean_test_data():
    """Убирает тестовые записи, чтобы не засорять базу."""
    for fn in (
        lambda: delete_price("__test_price__"),
        lambda: delete_action("__test_action__"),
    ):
        try:
            fn()
        except Exception:
            pass
    try:
        set_uc_balance(999000111, 0)
    except Exception:
        pass
    try:
        remove_from_blacklist(999000111)
    except Exception:
        pass
    try:
        reset_warnings(999000111)
    except Exception:
        pass


def _progress_bar(percent: float, width: int = 12) -> str:
    filled = int(percent * width)
    return "▰" * filled + "▱" * (width - filled)


def _build_check_text(i: int, total: int, lines: list) -> str:
    bar = _progress_bar(i / total)
    return (
        f"🛰️ <b>Космическая диагностика</b>\n"
        f"<code>{bar}</code> {i}/{total}\n\n"
        "┌───────────────\n"
        + "\n".join(lines)
        + "\n└───────────────"
    )


def typewriter(chat_id, text, parse_mode=None):
    """Эффект печатающей машинки: текст появляется по буквам."""
    msg = bot.send_message(chat_id, "▍", parse_mode=parse_mode)
    partial = ""
    for ch in text:
        partial += ch
        try:
            bot.edit_message_text(partial + "▍", chat_id, msg.message_id, parse_mode=parse_mode)
        except Exception:
            pass
        time.sleep(0.02)
    try:
        bot.edit_message_text(text, chat_id, msg.message_id, parse_mode=parse_mode)
    except Exception:
        pass


def run_system_check(message):
    """Красивый пошаговый самотест всех систем бота."""
    chat_id = message.chat.id
    steps = [
        ("База данных", check_database),
        ("Прайс-лист", check_prices),
        ("Действия (/do)", check_actions),
        ("Права доступа", check_permissions),
        ("Блокировка чата", check_chat_lock),
        ("Счётчик сообщений", check_msg_counter),
        ("Предупреждения", check_warnings),
        ("Чёрный список", check_blacklist),
        ("Баланс UC", check_uc),
        ("PUBG API", check_pubg),
        ("Погода (wttr.in)", check_weather),
    ]
    total = len(steps)

    bot.send_chat_action(chat_id, "typing")
    time.sleep(0.8)
    status_msg = bot.send_message(
        chat_id,
        f"🛰️ <b>Космическая диагностика</b>\n"
        f"<code>{_progress_bar(0)}</code> 0/{total}\n\n"
        f"Систем к проверке: <b>{total}</b>",
        parse_mode="HTML",
    )
    time.sleep(1.0)

    completed = []
    results = []   # (имя, 'ok'|'warn'|'fail', пояснение)

    for i, (name, fn) in enumerate(steps, 1):
        # анимация «печатаем» текущий шаг
        for frame in SPINNER_FRAMES:
            bot.send_chat_action(chat_id, "typing")
            lines = list(completed) + [f"   {name}  {frame}"]
            try:
                bot.edit_message_text(
                    _build_check_text(i, total, lines),
                    chat_id, status_msg.message_id, parse_mode="HTML",
                )
            except Exception:
                pass
            time.sleep(0.35)

        # выполняем проверку
        result, note = _run_step(fn)

        if result == "ok":
            line = f"   {name}  ✅"
        elif result == "warn":
            line = f"   {name}  ⚠️ <i>{note}</i>"
        else:
            line = f"   {name}  ❌ <i>{note}</i>"

        completed.append(line)
        results.append((name, result, note))

        try:
            bot.edit_message_text(
                _build_check_text(i, total, completed),
                chat_id, status_msg.message_id, parse_mode="HTML",
            )
        except Exception:
            pass
        bot.send_chat_action(chat_id, "typing")
        time.sleep(1.2)

    # ---------- итог ----------
    ok_count = sum(1 for _, r, _ in results if r == "ok")
    warn_count = sum(1 for _, r, _ in results if r == "warn")
    fail_count = sum(1 for _, r, _ in results if r == "fail")

    if fail_count == 0:
        final_text = (
            "👑 <b>Диагностика завершена, господин!</b>\n\n"
            f"✅ Систем проверено: <b>{total}</b> - все исправны.\n"
        )
        if warn_count:
            final_text += f"⚠️ Предупреждений: {warn_count}\n"
        final_text += "\n🤖 <b>Бот готов к работе, господин!</b> 🚀"
        typewriter(chat_id, final_text, parse_mode="HTML")
    else:
        err_lines = "\n".join(
            f"{i}. <b>{n}</b> - {nt}"
            for i, (n, r, nt) in enumerate(
                [(n, r, nt) for n, r, nt in results if r == "fail"], 1
            )
        )
        bot.send_message(
            chat_id,
            "👑 <b>Диагностика завершена, господин!</b>\n\n"
            f"❌ Найдено проблем: <b>{fail_count}</b>\n{err_lines}\n\n"
            "⚠️ Бот работает, но требуется вмешательство господина.",
            parse_mode="HTML",
        )

    _clean_test_data()


# ------------------- ЗАПУСК ДИАГНОСТИКИ -------------------
@bot.message_handler(commands=["check"])
def cmd_check(message):
    """Запуск полной диагностики (только создатель)."""
    if message.from_user.id != ADMIN_USER_ID:
        bot.reply_to(message, "🚫 Эта команда доступна только господину.")
        return
    run_system_check(message)


@bot.message_handler(func=lambda m: re.search(r"космос\s+провер", (m.text or "").lower()))
def cmd_kosmos_check(message):
    """Текстовая команда «Космос проверка» (только создатель)."""
    if message.from_user.id != ADMIN_USER_ID:
        bot.reply_to(message, "🚫 Эта команда доступна только господину.")
        return
    run_system_check(message)


# ------------------- ОБРАБОТКА СООБЩЕНИЙ В ГРУППАХ -------------------
@bot.message_handler(func=lambda m: m.chat.type in ("group", "supergroup"), content_types=["text"])
def group_text_handler(message):
    chat_id = message.chat.id
    user_id = message.from_user.id
    text = (message.text or "").lower()

    # 0) Запоминаем группу и пользователя
    save_group(chat_id, message.chat.title or "")
    if not message.from_user.is_bot:
        remember_user(user_id, message.from_user.username, message.from_user.first_name)

    # 0.1) Если пользователь в чёрном списке - удаляем сообщение и кикаем
    if is_blacklisted(user_id):
        try:
            bot.delete_message(chat_id, message.message_id)
        except Exception as e:
            logging.debug("Не удалось удалить сообщение ЧС-пользователя: %s", e)
        kick_user(chat_id, user_id)
        return

    # 1) Блокировка чата (только админы пишут)
    if is_chat_locked(chat_id) and not is_group_admin(chat_id, user_id) and user_id != ADMIN_USER_ID:
        try:
            bot.delete_message(chat_id, message.message_id)
        except Exception as e:
            logging.debug("Не удалось удалить сообщение в заблокированном чате: %s", e)
        bot.send_message(chat_id, f"🔒 {message.from_user.first_name}, чат заблокирован. Писать могут только админы.")
        return

    # 2) Счётчик сообщений
    if not message.from_user.is_bot:
        inc_msg_counter(chat_id, user_id)

    # 3) БАН «космос бан»
    if "космос бан" in text:
        if user_id != ADMIN_USER_ID and not is_group_admin(chat_id, user_id):
            bot.reply_to(message, "🚫 Только админы группы могут банить.")
            return
        target_user = None
        if message.reply_to_message and message.reply_to_message.from_user:
            target_user = message.reply_to_message.from_user
        else:
            m = re.search(r"@([A-Za-z0-9_]+)", message.text or "")
            if m:
                uid = get_user_by_username(m.group(1))
                if uid is None:
                    bot.reply_to(message, f"❌ Не нашёл @{m.group(1)}. Он должен написать хотя бы одно сообщение в чате, чтобы бот его запомнил.")
                    return
                try:
                    chat_member = bot.get_chat_member(chat_id, uid)
                    target_user = chat_member.user
                except Exception:
                    bot.reply_to(message, f"❌ Не нашёл пользователя @{m.group(1)} в этой группе.")
                    return
            else:
                m_id = re.search(r"(\d{5,})", message.text or "")
                if m_id:
                    try:
                        chat_member = bot.get_chat_member(chat_id, int(m_id.group(1)))
                        target_user = chat_member.user
                    except Exception:
                        bot.reply_to(message, f"❌ Не нашёл пользователя с ID {m_id.group(1)} в группе.")
                        return
                else:
                    bot.reply_to(
                        message,
                        "❗ Как банить?\n"
                        "1. Ответьте на сообщение нарушителя и напишите «Космос бан»\n"
                        "2. Или: Космос бан @username (если он писал в чат)\n"
                        "3. Или: Космос бан 123456789 (ID)",
                    )
                    return
        if not target_user:
            bot.reply_to(message, "❌ Не удалось определить, кого банить.")
            return
        if is_group_admin(chat_id, target_user.id) or target_user.id == ADMIN_USER_ID:
            bot.reply_to(message, "🚫 Нельзя забанить администратора группы или создателя бота.")
            return
        caller = message.from_user.first_name or message.from_user.username or "Админ"
        ok, msg = ban_user(chat_id, target_user.id, caller)
        bot.reply_to(message, msg + (f" ({target_user.first_name})" if ok else ""))
        return

    # 4) Кто создатель
    if "кто твой создатель" in text or "кто тебя создал" in text:
        if user_id == ADMIN_USER_ID:
            bot.reply_to(message, "Мой создатель - KOSMOS (это ты, Кирюша 👑)")
        else:
            bot.reply_to(message, "Мой создатель - KOSMOS")
        return

    # 5) Кто админы
    if "кто админы" in text or "кто админ" in text or "админы в группе" in text:
        try:
            admins = bot.get_chat_administrators(chat_id)
        except Exception:
            bot.reply_to(message, "⚠️ Не удалось получить список админов.")
            return
        names = []
        for a in admins:
            u = a.user
            name = u.first_name or u.username or str(u.id)
            if u.username:
                name = "@" + u.username
            names.append(name)
        bot.reply_to(message, "🛡️ Админы группы:\n" + "\n".join(names))
        return

    # 6) Счётчик сообщений
    if "сколько я написал" in text or "мои сообщения" in text or "счётчик" in text:
        cnt = get_msg_count(chat_id, user_id)
        bot.reply_to(message, f"📊 {message.from_user.first_name}, ты написал(а) {cnt} сообщений.")
        return

    # 7) Прайс-лист в группе
    if "прайс лист" in text and ("uc" in text or "космос" in text):
        prices = load_prices()
        if not prices:
            bot.reply_to(message, "⚠️ Прайс-лист UC пуст. Добавьте через админ-панель.")
        else:
            lines = [f"{item}: {price}" for item, price in prices.items()]
            bot.reply_to(message, "📊 Прайс-лист UC:\n" + "\n".join(lines))
        return


# ------------------- АВТОТЕСТ -------------------
def run_self_test():
    """Проверяет работоспособность всех функций БД и логики."""
    print("=== САМОТЕСТ БОТА ===")
    init_db()
    print("[OK] База данных готова")

    # Тест цен
    test_item = "__test_item__"
    save_price(test_item, "100₽")
    assert load_prices().get(test_item) == "100₽", "Цены не сохранились"
    save_price(test_item, "150₽")
    assert load_prices().get(test_item) == "150₽", "Обновление цены не сработало"
    assert delete_price(test_item) is True, "Удалить цену не удалось"
    assert delete_price(test_item) is False, "Повторное удаление должно вернуть False"
    print("[OK] Цены работают")

    # Тест действий
    test_action = "__test_action__"
    save_action(test_action)
    assert test_action in get_actions(), "Действие не сохранилось"
    assert delete_action(test_action) is True, "Не удалось удалить действие"
    assert delete_action(test_action) is False, "Повторное удаление действия должно вернуть False"
    print("[OK] Действия работают")

    # Тест прав
    assert is_admin(ADMIN_USER_ID, ADMIN_KEY), "Права администратора не прошли"
    assert not is_admin(123456, ADMIN_KEY), "Неправильный пользователь считается админом"
    print("[OK] Проверка прав прошла")

    # Тест счётчика сообщений
    set_chat_lock(123456, True)
    assert is_chat_locked(123456) is True, "Блокировка чата не включилась"
    set_chat_lock(123456, False)
    assert is_chat_locked(123456) is False, "Блокировка чата не выключилась"
    print("[OK] Блокировка чата работает")

    # Тест предупреждений
    test_uid = 999999999
    reset_warnings(test_uid)
    assert get_warnings(test_uid) == 0, "Предупреждения не обнулились"
    assert add_warning(test_uid) == 1, "Первое пред не сработало"
    assert add_warning(test_uid) == 2, "Второе пред не сработало"
    assert add_warning(test_uid) == 3, "Третье пред не сработало"
    reset_warnings(test_uid)
    print("[OK] Предупреждения работают")

    # Тест чёрного списка
    remove_from_blacklist(test_uid)
    assert not is_blacklisted(test_uid), "Чёрный список должен быть пуст"
    assert add_to_blacklist(test_uid) is True, "Добавление в чёрный список не сработало"
    assert is_blacklisted(test_uid), "Проверка чёрного списка не сработала"
    assert add_to_blacklist(test_uid) is False, "Повторное добавление должно вернуть False"
    assert remove_from_blacklist(test_uid) is True, "Удаление из чёрного списка не сработало"
    assert not is_blacklisted(test_uid), "Чёрный список должен быть пуст после удаления"
    print("[OK] Чёрный список работает")

    # Тест UC
    set_uc_balance(test_uid, 0)
    assert get_uc_balance(test_uid) == 0, "Баланс должен быть 0"
    add_uc(test_uid, 5000)
    assert get_uc_balance(test_uid) == 5000, "Пополнение UC не сработало"
    add_uc(test_uid, -2000)
    assert get_uc_balance(test_uid) == 3000, "Снятие UC не сработало"
    add_uc(test_uid, -99999)
    assert get_uc_balance(test_uid) == 0, "Баланс не должен уходить в минус"
    set_uc_balance(test_uid, 0)
    print("[OK] UC работает")

    print("=== ВСЕ ТЕСТЫ ПРОЙДЕНЫ ===")


# ------------------- ЗАПУСК -------------------
if __name__ == "__main__":
    run_self_test()

    # Устанавливаем команды в UI (если бот имеет права)
    try:
        bot.set_my_commands(
            [
                types.BotCommand("start", "Запуск бота"),
                types.BotCommand("help", "Справка"),
                types.BotCommand("price", "Прайс-лист"),
                types.BotCommand("player", "Ник по ID"),
                types.BotCommand("myuc", "Мой баланс UC"),
                types.BotCommand("topuc", "Топ UC"),
                types.BotCommand("admins", "Админы группы"),
                types.BotCommand("lockchat", "Блокировать чат"),
                types.BotCommand("my_groups", "Список групп"),
                types.BotCommand("send", "Отправить в группу"),
                types.BotCommand("admin", "Админ-панель"),
                types.BotCommand("check", "Проверка систем"),
                types.BotCommand("setadmin", "Сменить пароль"),
                types.BotCommand("setmsg", "Изменить сообщение"),
                types.BotCommand("maintenance", "Тех. работы on/off"),
            ]
        )
    except Exception as e:
        logging.warning("Не удалось установить команды: %s", e)

    print("🚀 Бот «Космос» запущен и готов к работе")

    # Устойчивый цикл: при обрыве сети бот сам переподключится через 5 сек
    while True:
        try:
            bot.infinity_polling(skip_pending=True, timeout=60, long_polling_timeout=60)
        except Exception as e:
            logging.error("Соединение потеряно: %s", e)
            logging.info("Повторная попытка через 5 секунд...")
            time.sleep(5)
