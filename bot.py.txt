# ==============================================================================
# 🎬 MUKAMMAL TELEGRAM ANIME BOT (AIOGRAM 3 + AIOSQLITE + RENDER READY)
# ==============================================================================
# Mualliflik: Barcha funksiyalari to'liq yozilgan, qisqartirilmagan versiya.
# Texnologiyalar: Python 3.10+, aiogram 3.15+, aiosqlite, aiohttp (Render port uchun)
# ==============================================================================

import asyncio
import logging
import os
import secrets
import sys
from datetime import datetime, timedelta
from typing import Any, Callable, Dict, List, Optional

import aiosqlite
from aiohttp import web
from aiogram import BaseMiddleware, Bot, Dispatcher, F, Router
from aiogram.client.default import DefaultBotProperties
from aiogram.enums import ChatMemberStatus, ParseMode
from aiogram.filters import Command, CommandStart, StateFilter
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.types import (
    CallbackQuery,
    InlineKeyboardButton,
    InlineKeyboardMarkup,
    KeyboardButton,
    Message,
    ReplyKeyboardMarkup,
    ReplyKeyboardRemove,
    TelegramObject,
)

# ------------------------------------------------------------------------------
# 1. SOZLAMALAR VA KONFIGURATSIYA (CONFIG)
# ------------------------------------------------------------------------------
# DIQQAT: os.getenv() birinchi parametri o'zgaruvchi nomi ('BOT_TOKEN'),
# ikkinchi parametri esa standart qiymat bo'ladi.
BOT_TOKEN = os.getenv("BOT_TOKEN", "8732171636:AAEVjKlB7f-JIk4a9fTyCtEmDlBu9aL1AWs")
OWNER_ID = int(os.getenv("OWNER_ID", "8318241400"))
DATABASE_PATH = os.getenv("DATABASE_PATH", "anime_bot.db")
PORT = int(os.getenv("PORT", "10000"))  # Render web service porti

# 📢 KOD ICHIDAGI BOSHLANG'ICH KANALLAR SOZLAMASI
# Siz bu yerga o'z kanallaringizni yozib qo'yishingiz mumkin.
# Bot birinchi marta ishga tushganda bular avtomatik bazaga qo'shiladi!
DEFAULT_MANDATORY_CHANNELS = [
    {
        "channel_id": "", -1003852596133 # Kanalingiz ID raqami (masalan: -100...) yoki @username
        "title": "Anime Dunyosi | Rasmiy"
        "invite_link": "https://t.me/kagedub1
    }
]

DEFAULT_POST_CHANNELS = [
    {
        "channel_id": "-1003852596133",  # Post chiqariladigan kanal ID si
        "title": "Anime Premyeralar Kanali"
    }
]

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - [%(levelname)s] - %(name)s - %(message)s",
    handlers=[logging.StreamHandler(sys.stdout)],
)
logger = logging.getLogger("AnimeBot")

# Janrlar ro'yxati
POPULAR_GENRES = [
    "🔥 Jangari", "⚡ Shounen", "✨ Fantastika", "🎭 Drama",
    "😂 Komediya", "❤️ Romantika", "🌀 Isekai", "🗡️ Sarguzasht",
    "🧠 Triller", "🕵️ Detektiv", "🌸 Kundalik hayot", "💀 Dahshat",
    "👑 Tarixiy", "🏀 Sport", "🚀 Mexa", "🎓 Maktab"
]

# ------------------------------------------------------------------------------
# 2. MA'LUMOTLAR BAZASI (AIOSQLITE BILAN)
# ------------------------------------------------------------------------------
class Database:
    def __init__(self, db_path: str = DATABASE_PATH):
        self.db_path = db_path

    async def init(self):
        """Barcha jadvallarni va boshlang'ich ma'lumotlarni yaratadi"""
        async with aiosqlite.connect(self.db_path) as db:
            await db.execute("PRAGMA foreign_keys = ON;")
            
            # 1. Foydalanuvchilar jadvali
            await db.execute("""
                CREATE TABLE IF NOT EXISTS users (
                    user_id INTEGER PRIMARY KEY,
                    username TEXT,
                    full_name TEXT,
                    joined_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                    is_blocked INTEGER DEFAULT 0
                )
            """)

            # 2. Adminlar jadvali
            await db.execute("""
                CREATE TABLE IF NOT EXISTS admins (
                    user_id INTEGER PRIMARY KEY,
                    added_by INTEGER,
                    added_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                )
            """)

            # 3. Majburiy obuna kanallari
            await db.execute("""
                CREATE TABLE IF NOT EXISTS mandatory_channels (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    channel_id TEXT UNIQUE,
                    title TEXT,
                    invite_link TEXT
                )
            """)

            # 4. Post kanallari (anime postlari chiqadigan kanal)
            await db.execute("""
                CREATE TABLE IF NOT EXISTS post_channels (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    channel_id TEXT UNIQUE,
                    title TEXT
                )
            """)

            # 5. Animelar jadvali
            await db.execute("""
                CREATE TABLE IF NOT EXISTS animes (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    code INTEGER UNIQUE NOT NULL,
                    title TEXT NOT NULL,
                    original_title TEXT,
                    genres TEXT NOT NULL,
                    year INTEGER,
                    episodes_count INTEGER DEFAULT 12,
                    description TEXT,
                    poster_file_id TEXT,
                    is_vip INTEGER DEFAULT 0,
                    views INTEGER DEFAULT 0,
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                )
            """)

            # 6. Qismlar (Seriyalar) jadvali
            await db.execute("""
                CREATE TABLE IF NOT EXISTS episodes (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    anime_code INTEGER NOT NULL,
                    episode_number INTEGER NOT NULL,
                    video_file_id TEXT NOT NULL,
                    title TEXT,
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                    UNIQUE(anime_code, episode_number),
                    FOREIGN KEY(anime_code) REFERENCES animes(code) ON DELETE CASCADE
                )
            """)

            # 7. Sevimlilar (Tanlanganlar) jadvali
            await db.execute("""
                CREATE TABLE IF NOT EXISTS favorites (
                    user_id INTEGER NOT NULL,
                    anime_code INTEGER NOT NULL,
                    added_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                    PRIMARY KEY(user_id, anime_code),
                    FOREIGN KEY(anime_code) REFERENCES animes(code) ON DELETE CASCADE
                )
            """)

            # 8. VIP Foydalanuvchilar jadvali
            await db.execute("""
                CREATE TABLE IF NOT EXISTS vip_users (
                    user_id INTEGER PRIMARY KEY,
                    expires_at TIMESTAMP NOT NULL,
                    activated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                )
            """)

            # 9. VIP Promokodlar jadvali
            await db.execute("""
                CREATE TABLE IF NOT EXISTS vip_codes (
                    code TEXT PRIMARY KEY,
                    days INTEGER NOT NULL,
                    created_by INTEGER,
                    is_used INTEGER DEFAULT 0,
                    used_by INTEGER,
                    used_at TIMESTAMP,
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                )
            """)

            # 10. Zayavka / Takliflar jadvali
            await db.execute("""
                CREATE TABLE IF NOT EXISTS requests (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    user_id INTEGER NOT NULL,
                    username TEXT,
                    anime_title TEXT NOT NULL,
                    comment TEXT,
                    status TEXT DEFAULT 'kutilmoqda',
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                )
            """)

            # Boshlang'ich sinov animelarini kiritish (agar baza bo'sh bo'lsa)
            cursor = await db.execute("SELECT COUNT(*) FROM animes")
            row = await cursor.fetchone()
            if row and row[0] == 0:
                seed_animes = [
                    (101, "Solo Leveling (Yakkaxon daraja ko'tarish)", "Ore dake Level Up na Ken", "⚡ Shounen, ✨ Fantastika, 🔥 Jangari", 2024, 12, "Eng kuchsiz ovchi Jinwoo xavfli maxfiy g'orda o'lim yoqasida g'ayritabiiy tizimga ega bo'ladi.", "https://images.unsplash.com/photo-1578632767115-351597cf2477?w=800", 0),
                    (102, "Attack on Titan (Titanlar hujumi)", "Shingeki no Kyojin", "🔥 Jangari, 🎭 Drama, 🧠 Triller", 2013, 89, "Devorlar ortida odamxo'r titanlardan berkingan insoniyatning ozodlik uchun qonli kurashi.", "https://images.unsplash.com/photo-1534447677768-be436bb09401?w=800", 0),
                    (103, "Jujutsu Kaisen (Sehrli jang)", "Jujutsu Kaisen", "⚡ Shounen, 🔥 Jangari, 💀 Dahshat", 2020, 47, "Itadori Yuji qadimiy qudratli la'nat Sukunaning barmog'ini yutib yuboradi va sehrgarlar akademiyasiga kiradi.", "https://images.unsplash.com/photo-1607604276583-eef5d076aa5f?w=800", 1),
                    (104, "Demon Slayer (Iblislar qotili)", "Kimetsu no Yaiba", "⚡ Shounen, 🔥 Jangari, 🗡️ Sarguzasht", 2019, 55, "Tanjironing butun oilasini iblis qiradi, tirik qolgan singlisi Nezuko esa iblisga aylanadi.", "https://images.unsplash.com/photo-1563089145-599997674d42?w=800", 0),
                    (105, "Death Note (O'lim daftari)", "Death Note", "🧠 Triller, 🕵️ Detektiv, 🎭 Drama", 2006, 37, "Yagami Light nom yozilgan odam o'ladigan daftarga ega bo'ladi va yangi dunyo xudosi bo'lishga intiladi.", "https://images.unsplash.com/photo-1518709268805-4e9042af9f23?w=800", 0)
                ]
                await db.executemany("""
                    INSERT INTO animes (code, title, original_title, genres, year, episodes_count, description, poster_file_id, is_vip)
                    VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
                """, seed_animes)

            # Boshlang'ich majburiy obuna kanallarini kiritish (agar bo'sh bo'lsa)
            c_chan = await db.execute("SELECT COUNT(*) FROM mandatory_channels")
            if (await c_chan.fetchone())[0] == 0:
                for ch in DEFAULT_MANDATORY_CHANNELS:
                    await db.execute("""
                        INSERT OR IGNORE INTO mandatory_channels (channel_id, title, invite_link)
                        VALUES (?, ?, ?)
                    """, (ch["channel_id"], ch["title"], ch["invite_link"]))

            # Boshlang'ich post kanallarini kiritish (agar bo'sh bo'lsa)
            c_post = await db.execute("SELECT COUNT(*) FROM post_channels")
            if (await c_post.fetchone())[0] == 0:
                for ch in DEFAULT_POST_CHANNELS:
                    await db.execute("""
                        INSERT OR IGNORE INTO post_channels (channel_id, title)
                        VALUES (?, ?)
                    """, (ch["channel_id"], ch["title"]))

            await db.commit()
            logger.info("✅ Ma'lumotlar bazasi muvaffaqiyatli ishga tushirildi!")

    # --- FOYDALANUVCHILAR ---
    async def add_user(self, user_id: int, username: Optional[str], full_name: str):
        async with aiosqlite.connect(self.db_path) as db:
            await db.execute("""
                INSERT INTO users (user_id, username, full_name)
                VALUES (?, ?, ?)
                ON CONFLICT(user_id) DO UPDATE SET
                    username = excluded.username,
                    full_name = excluded.full_name
            """, (user_id, username, full_name))
            await db.commit()

    async def get_users_count(self) -> int:
        async with aiosqlite.connect(self.db_path) as db:
            async with db.execute("SELECT COUNT(*) FROM users") as cursor:
                res = await cursor.fetchone()
                return res[0] if res else 0

    async def get_all_user_ids(self) -> List[int]:
        async with aiosqlite.connect(self.db_path) as db:
            async with db.execute("SELECT user_id FROM users WHERE is_blocked = 0") as cursor:
                rows = await cursor.fetchall()
                return [r[0] for r in rows]

    # --- ADMIN VA OWNER HUQUQLARI ---
    async def is_admin(self, user_id: int) -> bool:
        if user_id == OWNER_ID:
            return True
        async with aiosqlite.connect(self.db_path) as db:
            async with db.execute("SELECT 1 FROM admins WHERE user_id = ?", (user_id,)) as cursor:
                return await cursor.fetchone() is not None

    async def add_admin(self, user_id: int, added_by: int) -> bool:
        try:
            async with aiosqlite.connect(self.db_path) as db:
                await db.execute("INSERT OR IGNORE INTO admins (user_id, added_by) VALUES (?, ?)", (user_id, added_by))
                await db.commit()
                return True
        except Exception:
            return False

    async def remove_admin(self, user_id: int) -> bool:
        async with aiosqlite.connect(self.db_path) as db:
            await db.execute("DELETE FROM admins WHERE user_id = ?", (user_id,))
            await db.commit()
            return True

    async def get_admins(self) -> List[Dict[str, Any]]:
        async with aiosqlite.connect(self.db_path) as db:
            db.row_factory = aiosqlite.Row
            async with db.execute("SELECT * FROM admins ORDER BY added_at DESC") as cursor:
                rows = await cursor.fetchall()
                return [dict(r) for r in rows]

    # --- MAJBURIR OBUNA KANALLARI ---
    async def get_mandatory_channels(self) -> List[Dict[str, Any]]:
        async with aiosqlite.connect(self.db_path) as db:
            db.row_factory = aiosqlite.Row
            async with db.execute("SELECT * FROM mandatory_channels") as cursor:
                rows = await cursor.fetchall()
                return [dict(r) for r in rows]

    async def add_mandatory_channel(self, channel_id: str, title: str, invite_link: str) -> bool:
        try:
            async with aiosqlite.connect(self.db_path) as db:
                await db.execute("""
                    INSERT INTO mandatory_channels (channel_id, title, invite_link)
                    VALUES (?, ?, ?)
                    ON CONFLICT(channel_id) DO UPDATE SET
                        title = excluded.title,
                        invite_link = excluded.invite_link
                """, (channel_id, title, invite_link))
                await db.commit()
                return True
        except Exception as e:
            logger.error(f"Kanal qo'shishda xato: {e}")
            return False

    async def remove_mandatory_channel(self, channel_id: str) -> bool:
        async with aiosqlite.connect(self.db_path) as db:
            await db.execute("DELETE FROM mandatory_channels WHERE channel_id = ?", (channel_id,))
            await db.commit()
            return True

    # --- POST KANALLARI ---
    async def get_post_channels(self) -> List[Dict[str, Any]]:
        async with aiosqlite.connect(self.db_path) as db:
            db.row_factory = aiosqlite.Row
            async with db.execute("SELECT * FROM post_channels") as cursor:
                rows = await cursor.fetchall()
                return [dict(r) for r in rows]

    async def add_post_channel(self, channel_id: str, title: str) -> bool:
        async with aiosqlite.connect(self.db_path) as db:
            await db.execute("INSERT OR REPLACE INTO post_channels (channel_id, title) VALUES (?, ?)", (channel_id, title))
            await db.commit()
            return True

    # --- ANIMELAR ---
    async def add_anime(self, code: int, title: str, original_title: str, genres: str,
                        year: int, episodes_count: int, description: str,
                        poster_file_id: str, is_vip: int = 0) -> bool:
        try:
            async with aiosqlite.connect(self.db_path) as db:
                await db.execute("""
                    INSERT INTO animes (code, title, original_title, genres, year, episodes_count, description, poster_file_id, is_vip)
                    VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
                """, (code, title, original_title, genres, year, episodes_count, description, poster_file_id, is_vip))
                await db.commit()
                return True
        except Exception as e:
            logger.error(f"Anime qo'shishda xato: {e}")
            return False

    async def get_anime_by_code(self, code: int) -> Optional[Dict[str, Any]]:
        async with aiosqlite.connect(self.db_path) as db:
            db.row_factory = aiosqlite.Row
            async with db.execute("SELECT * FROM animes WHERE code = ?", (code,)) as cursor:
                row = await cursor.fetchone()
                if row:
                    await db.execute("UPDATE animes SET views = views + 1 WHERE code = ?", (code,))
                    await db.commit()
                    return dict(row)
                return None

    async def search_animes(self, query: str, limit: int = 10) -> List[Dict[str, Any]]:
        async with aiosqlite.connect(self.db_path) as db:
            db.row_factory = aiosqlite.Row
            pattern = f"%{query}%"
            async with db.execute("""
                SELECT * FROM animes 
                WHERE title LIKE ? OR original_title LIKE ? OR genres LIKE ?
                ORDER BY views DESC LIMIT ?
            """, (pattern, pattern, pattern, limit)) as cursor:
                rows = await cursor.fetchall()
                return [dict(r) for r in rows]

    async def get_animes_by_genre(self, genre: str, limit: int = 8) -> List[Dict[str, Any]]:
        async with aiosqlite.connect(self.db_path) as db:
            db.row_factory = aiosqlite.Row
            clean_genre = genre.replace("🔥", "").replace("⚡", "").replace("✨", "").replace("🎭", "").strip()
            pattern = f"%{clean_genre}%"
            async with db.execute("""
                SELECT * FROM animes 
                WHERE genres LIKE ? 
                ORDER BY views DESC LIMIT ?
            """, (pattern, limit)) as cursor:
                rows = await cursor.fetchall()
                return [dict(r) for r in rows]

    async def get_random_anime(self) -> Optional[Dict[str, Any]]:
        async with aiosqlite.connect(self.db_path) as db:
            db.row_factory = aiosqlite.Row
            async with db.execute("SELECT * FROM animes ORDER BY RANDOM() LIMIT 1") as cursor:
                row = await cursor.fetchone()
                return dict(row) if row else None

    async def delete_anime(self, code: int) -> bool:
        async with aiosqlite.connect(self.db_path) as db:
            await db.execute("DELETE FROM animes WHERE code = ?", (code,))
            await db.commit()
            return True

    # --- SERIYALAR (QISMLAR) ---
    async def add_episode(self, anime_code: int, episode_number: int, video_file_id: str, title: str = "") -> bool:
        try:
            async with aiosqlite.connect(self.db_path) as db:
                await db.execute("""
                    INSERT INTO episodes (anime_code, episode_number, video_file_id, title)
                    VALUES (?, ?, ?, ?)
                    ON CONFLICT(anime_code, episode_number) DO UPDATE SET
                        video_file_id = excluded.video_file_id,
                        title = excluded.title
                """, (anime_code, episode_number, video_file_id, title))
                await db.commit()
                return True
        except Exception as e:
            logger.error(f"Qism qo'shishda xato: {e}")
            return False

    async def delete_episode(self, anime_code: int, episode_number: int) -> bool:
        async with aiosqlite.connect(self.db_path) as db:
            await db.execute("DELETE FROM episodes WHERE anime_code = ? AND episode_number = ?", (anime_code, episode_number))
            await db.commit()
            return True

    async def get_episodes(self, anime_code: int) -> List[Dict[str, Any]]:
        async with aiosqlite.connect(self.db_path) as db:
            db.row_factory = aiosqlite.Row
            async with db.execute("SELECT * FROM episodes WHERE anime_code = ? ORDER BY episode_number ASC", (anime_code,)) as cursor:
                rows = await cursor.fetchall()
                return [dict(r) for r in rows]

    async def get_episode(self, anime_code: int, episode_number: int) -> Optional[Dict[str, Any]]:
        async with aiosqlite.connect(self.db_path) as db:
            db.row_factory = aiosqlite.Row
            async with db.execute("SELECT * FROM episodes WHERE anime_code = ? AND episode_number = ?", (anime_code, episode_number)) as cursor:
                row = await cursor.fetchone()
                return dict(row) if row else None

    # --- SEVIMLILAR ---
    async def toggle_favorite(self, user_id: int, anime_code: int) -> bool:
        """True qaytsa qo'shildi, False qaytsa o'chirildi"""
        async with aiosqlite.connect(self.db_path) as db:
            async with db.execute("SELECT 1 FROM favorites WHERE user_id = ? AND anime_code = ?", (user_id, anime_code)) as cursor:
                exists = await cursor.fetchone()
            
            if exists:
                await db.execute("DELETE FROM favorites WHERE user_id = ? AND anime_code = ?", (user_id, anime_code))
                await db.commit()
                return False
            else:
                await db.execute("INSERT INTO favorites (user_id, anime_code) VALUES (?, ?)", (user_id, anime_code))
                await db.commit()
                return True

    async def is_favorite(self, user_id: int, anime_code: int) -> bool:
        async with aiosqlite.connect(self.db_path) as db:
            async with db.execute("SELECT 1 FROM favorites WHERE user_id = ? AND anime_code = ?", (user_id, anime_code)) as cursor:
                return await cursor.fetchone() is not None

    async def get_user_favorites(self, user_id: int) -> List[Dict[str, Any]]:
        async with aiosqlite.connect(self.db_path) as db:
            db.row_factory = aiosqlite.Row
            async with db.execute("""
                SELECT a.* FROM animes a
                JOIN favorites f ON a.code = f.anime_code
                WHERE f.user_id = ?
                ORDER BY f.added_at DESC
            """, (user_id,)) as cursor:
                rows = await cursor.fetchall()
                return [dict(r) for r in rows]

    # --- VIP TIZIMI ---
    async def is_vip(self, user_id: int) -> bool:
        if user_id == OWNER_ID:
            return True
        async with aiosqlite.connect(self.db_path) as db:
            async with db.execute("SELECT expires_at FROM vip_users WHERE user_id = ?", (user_id,)) as cursor:
                row = await cursor.fetchone()
                if not row:
                    return False
                expires_at = datetime.fromisoformat(row[0])
                if expires_at > datetime.utcnow():
                    return True
                else:
                    await db.execute("DELETE FROM vip_users WHERE user_id = ?", (user_id,))
                    await db.commit()
                    return False

    async def get_vip_info(self, user_id: int) -> Optional[datetime]:
        async with aiosqlite.connect(self.db_path) as db:
            async with db.execute("SELECT expires_at FROM vip_users WHERE user_id = ?", (user_id,)) as cursor:
                row = await cursor.fetchone()
                if row:
                    return datetime.fromisoformat(row[0])
                return None

    async def add_vip(self, user_id: int, days: int):
        new_expiry = datetime.utcnow() + timedelta(days=days)
        async with aiosqlite.connect(self.db_path) as db:
            await db.execute("""
                INSERT INTO vip_users (user_id, expires_at)
                VALUES (?, ?)
                ON CONFLICT(user_id) DO UPDATE SET
                    expires_at = datetime(expires_at, '+' || ? || ' days')
            """, (user_id, new_expiry.isoformat(), days))
            await db.commit()

    async def create_vip_code(self, days: int, created_by: int) -> str:
        code = f"VIP-{secrets.token_hex(4).upper()}-{days}D"
        async with aiosqlite.connect(self.db_path) as db:
            await db.execute("""
                INSERT INTO vip_codes (code, days, created_by)
                VALUES (?, ?, ?)
            """, (code, days, created_by))
            await db.commit()
        return code

    async def redeem_vip_code(self, user_id: int, code: str) -> Optional[int]:
        code = code.strip().upper()
        async with aiosqlite.connect(self.db_path) as db:
            db.row_factory = aiosqlite.Row
            async with db.execute("SELECT * FROM vip_codes WHERE code = ? AND is_used = 0", (code,)) as cursor:
                row = await cursor.fetchone()
                if not row:
                    return None
                days = row["days"]

            await db.execute("""
                UPDATE vip_codes SET is_used = 1, used_by = ?, used_at = CURRENT_TIMESTAMP
                WHERE code = ?
            """, (user_id, code))
            await db.commit()

        await self.add_vip(user_id, days)
        return days

    # --- ZAYAVKA / TAKLIFLAR ---
    async def add_request(self, user_id: int, username: Optional[str], title: str, comment: str) -> int:
        async with aiosqlite.connect(self.db_path) as db:
            cursor = await db.execute("""
                INSERT INTO requests (user_id, username, anime_title, comment)
                VALUES (?, ?, ?, ?)
            """, (user_id, username, title, comment))
            await db.commit()
            return cursor.lastrowid or 0

    # --- STATISTIKA ---
    async def get_stats(self) -> Dict[str, int]:
        async with aiosqlite.connect(self.db_path) as db:
            async with db.execute("SELECT COUNT(*) FROM users") as c1:
                u_cnt = (await c1.fetchone())[0]
            async with db.execute("SELECT COUNT(*) FROM animes") as c2:
                a_cnt = (await c2.fetchone())[0]
            async with db.execute("SELECT COUNT(*) FROM episodes") as c3:
                e_cnt = (await c3.fetchone())[0]
            async with db.execute("SELECT COUNT(*) FROM vip_users WHERE expires_at > CURRENT_TIMESTAMP") as c4:
                v_cnt = (await c4.fetchone())[0]
            async with db.execute("SELECT COUNT(*) FROM requests") as c5:
                r_cnt = (await c5.fetchone())[0]
            return {
                "users": u_cnt,
                "animes": a_cnt,
                "episodes": e_cnt,
                "vips": v_cnt,
                "requests": r_cnt
            }

db = Database()

# ------------------------------------------------------------------------------
# 3. FSM (FINITE STATE MACHINE) HOLATLARI
# ------------------------------------------------------------------------------
class SearchStates(StatesGroup):
    waiting_for_code = State()
    waiting_for_title = State()

class RequestStates(StatesGroup):
    waiting_for_title = State()
    waiting_for_comment = State()

class VipStates(StatesGroup):
    waiting_for_code = State()

class AdminAnimeStates(StatesGroup):
    waiting_for_code = State()
    waiting_for_title = State()
    waiting_for_original_title = State()
    waiting_for_genres = State()
    waiting_for_year = State()
    waiting_for_episodes_count = State()
    waiting_for_description = State()
    waiting_for_poster = State()
    waiting_for_vip_status = State()

class AdminEpisodeStates(StatesGroup):
    waiting_for_anime_code = State()
    waiting_for_episode_num = State()
    waiting_for_video = State()

class AdminPostStates(StatesGroup):
    waiting_for_anime_code = State()

class OwnerAdminStates(StatesGroup):
    waiting_for_admin_id = State()

class OwnerChannelStates(StatesGroup):
    waiting_for_channel_id = State()
    waiting_for_channel_title = State()
    waiting_for_invite_link = State()

class OwnerBroadcastStates(StatesGroup):
    waiting_for_message = State()

# ------------------------------------------------------------------------------
# 4. KLAVIATURALAR (REPLY VA INLINE)
# ------------------------------------------------------------------------------
def main_reply_keyboard(is_admin: bool = False, is_owner: bool = False) -> ReplyKeyboardMarkup:
    kb = [
        [KeyboardButton(text="🔍 Anime qidirish"), KeyboardButton(text="📂 Janrlar")],
        [KeyboardButton(text="🎲 Tasodifiy anime"), KeyboardButton(text="⭐ Sevimlilar")],
        [KeyboardButton(text="👑 VIP Kabinet"), KeyboardButton(text="✍️ Zayavka / Taklif")],
        [KeyboardButton(text="ℹ️ Ma'lumot va Yordam")]
    ]
    if is_owner:
        kb.append([KeyboardButton(text="👑 Egasi (Owner) Paneli"), KeyboardButton(text="⚙️ Admin Paneli")])
    elif is_admin:
        kb.append([KeyboardButton(text="⚙️ Admin Paneli")])

    return ReplyKeyboardMarkup(keyboard=kb, resize_keyboard=True)

def cancel_reply_keyboard() -> ReplyKeyboardMarkup:
    return ReplyKeyboardMarkup(
        keyboard=[[KeyboardButton(text="❌ Bekor qilish")]],
        resize_keyboard=True
    )

def search_type_inline_keyboard() -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[
        [
            InlineKeyboardButton(text="🔢 Kod orqali qidirish", callback_data="search_by_code"),
            InlineKeyboardButton(text="📝 Nomi orqali qidirish", callback_data="search_by_title")
        ]
    ])

def genres_inline_keyboard() -> InlineKeyboardMarkup:
    buttons = []
    row = []
    for i, genre in enumerate(POPULAR_GENRES, 1):
        clean_tag = genre.split()[-1]
        row.append(InlineKeyboardButton(text=genre, callback_data=f"genre_{clean_tag}"))
        if len(row) == 2:
            buttons.append(row)
            row = []
    if row:
        buttons.append(row)
    return InlineKeyboardMarkup(inline_keyboard=buttons)

def anime_detail_keyboard(anime_code: int, is_fav: bool, has_episodes: bool = True) -> InlineKeyboardMarkup:
    fav_icon = "❌ Sevimlilardan o'chirish" if is_fav else "⭐ Sevimlilarga saqlash"
    buttons = [
        [InlineKeyboardButton(text="🎬 Qismlarni ko'rish", callback_data=f"watch_{anime_code}")],
        [
            InlineKeyboardButton(text=fav_icon, callback_data=f"fav_toggle_{anime_code}"),
            InlineKeyboardButton(text="🎲 Boshqa anime", callback_data="random_anime")
        ],
        [InlineKeyboardButton(text="🔗 Do'stlarga ulashish", switch_inline_query=f"code_{anime_code}")]
    ]
    return InlineKeyboardMarkup(inline_keyboard=buttons)

def episodes_inline_keyboard(anime_code: int, episodes: List[Dict[str, Any]], page: int = 1) -> InlineKeyboardMarkup:
    per_page = 10
    total = len(episodes)
    start_idx = (page - 1) * per_page
    end_idx = start_idx + per_page
    page_episodes = episodes[start_idx:end_idx]

    buttons = []
    row = []
    for ep in page_episodes:
        ep_num = ep["episode_number"]
        row.append(InlineKeyboardButton(text=f"▶️ {ep_num}-qism", callback_data=f"play_{anime_code}_{ep_num}"))
        if len(row) == 2:
            buttons.append(row)
            row = []
    if row:
        buttons.append(row)

    # Navigatsiya tugmalari
    nav_row = []
    if page > 1:
        nav_row.append(InlineKeyboardButton(text="⬅️ Oldingi", callback_data=f"ep_page_{anime_code}_{page-1}"))
    if end_idx < total:
        nav_row.append(InlineKeyboardButton(text="Keyingi ➡️", callback_data=f"ep_page_{anime_code}_{page+1}"))
    
    if nav_row:
        buttons.append(nav_row)

    buttons.append([InlineKeyboardButton(text="⬅️ Animaga qaytish", callback_data=f"anime_{anime_code}")])
    return InlineKeyboardMarkup(inline_keyboard=buttons)

def subscription_inline_keyboard(channels: List[Dict[str, Any]]) -> InlineKeyboardMarkup:
    buttons = []
    for ch in channels:
        buttons.append([InlineKeyboardButton(text=f"📢 {ch['title']}", url=ch["invite_link"])])
    buttons.append([InlineKeyboardButton(text="✅ A'zo bo'ldim (Tekshirish)", callback_data="check_subscription")])
    return InlineKeyboardMarkup(inline_keyboard=buttons)

def admin_panel_inline_keyboard() -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[
        [
            InlineKeyboardButton(text="➕ Yangi Anime qo'shish", callback_data="adm_add_anime"),
            InlineKeyboardButton(text="🎬 Qism yuklash", callback_data="adm_add_episode")
        ],
        [
            InlineKeyboardButton(text="📢 Kanalga post qilish", callback_data="adm_post_channel"),
            InlineKeyboardButton(text="🗑️ Anime o'chirish", callback_data="adm_delete_anime")
        ],
        [
            InlineKeyboardButton(text="📊 Umumiy Statistika", callback_data="adm_stats")
        ]
    ])

def owner_panel_inline_keyboard() -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[
        [
            InlineKeyboardButton(text="👮 Yangi Admin qo'shish", callback_data="own_add_admin"),
            InlineKeyboardButton(text="📋 Adminlar ro'yxati", callback_data="own_list_admins")
        ],
        [
            InlineKeyboardButton(text="📢 Majburiy obuna kanallari", callback_data="own_channels"),
            InlineKeyboardButton(text="➕ Majburiy kanal qo'shish", callback_data="own_add_channel")
        ],
        [
            InlineKeyboardButton(text="🔑 VIP Promokod yaratish", callback_data="own_create_vip"),
            InlineKeyboardButton(text="📨 Barchaga xabarnoma (Broadcast)", callback_data="own_broadcast")
        ]
    ])

# ------------------------------------------------------------------------------
# 5. XAVFSIZLIK VA MAJBURIY OBUNA MIDDLEWARE
# ------------------------------------------------------------------------------
class MandatorySubscriptionMiddleware(BaseMiddleware):
    """
    Foydalanuvchi majburiy kanallarga a'zo bo'lmaguncha botdan foydalanishni cheklovchi xavfsizlik qatlami.
    Admin va Egasi (Owner) avtomatik tekshiruvdan o'tadi.
    """
    async def __call__(
        self,
        handler: Callable[[TelegramObject, Dict[str, Any]], Any],
        event: TelegramObject,
        data: Dict[str, Any]
    ) -> Any:
        bot: Bot = data["bot"]
        user = data.get("event_from_user")
        if not user:
            return await handler(event, data)

        # Admin yoki Owner bo'lsa tekshirmay o'tkazish
        if user.id == OWNER_ID or await db.is_admin(user.id):
            return await handler(event, data)

        # Agar callback_query aynan tekshirish tugmasi bo'lsa ruxsat berish
        if isinstance(event, CallbackQuery) and event.data == "check_subscription":
            return await handler(event, data)

        channels = await db.get_mandatory_channels()
        if not channels:
            return await handler(event, data)

        unsubscribed = []
        for ch in channels:
            try:
                member = await bot.get_chat_member(chat_id=ch["channel_id"], user_id=user.id)
                if member.status in [ChatMemberStatus.LEFT, ChatMemberStatus.KICKED]:
                    unsubscribed.append(ch)
            except Exception as e:
                logger.warning(f"Kanal obunasini tekshirishda xato ({ch['channel_id']}): {e}")
                # Agar bot kanal admini bo'lmasa yoki xato bo'lsa o'tkazib yuborish

        if unsubscribed:
            text = (
                "⚠️ <b>Diqqat! Botdan foydalanish uchun quyidagi rasmiy kanallarga a'zo bo'lishingiz shart:</b>\n\n"
                "<i>A'zo bo'lgach, «✅ A'zo bo'ldim (Tekshirish)» tugmasini bosing!</i>"
            )
            kb = subscription_inline_keyboard(unsubscribed)
            if isinstance(event, Message):
                await event.answer(text, reply_markup=kb, parse_mode=ParseMode.HTML)
            elif isinstance(event, CallbackQuery):
                await event.message.answer(text, reply_markup=kb, parse_mode=ParseMode.HTML)
                await event.answer("⚠️ Iltimos, avval kanallarga a'zo bo'ling!", show_alert=True)
            return

        return await handler(event, data)

# ------------------------------------------------------------------------------
# 6. ASOSIY FOYDALANUVCHI HANDLERLARI
# ------------------------------------------------------------------------------
router = Router()

@router.message(CommandStart())
async def cmd_start(message: Message, state: FSMContext):
    await state.clear()
    user = message.from_user
    await db.add_user(user.id, user.username, user.full_name)
    
    is_adm = await db.is_admin(user.id)
    is_own = (user.id == OWNER_ID)

    # Parametr tekshirish (masalan: /start code_101)
    args = message.text.split()
    if len(args) > 1 and args[1].startswith("code_"):
        try:
            code = int(args[1].replace("code_", ""))
            await show_anime_by_code(message, code, user.id)
            return
        except ValueError:
            pass

    welcome_text = (
        f"👋 <b>Assalomu alaykum, {user.full_name}!</b>\n\n"
        "🎬 <b>Eng sara va sifatli animelar olamiga xush kelibsiz!</b>\n"
        "Bu yerda siz sevimli animelaringizni kod yoki nom orqali qidirishingiz, "
        "HD sifatda tomosha qilishingiz va yangiliklardan birinchi bo'lib xabardor bo'lishingiz mumkin.\n\n"
        "👇 <i>Quyidagi menyudan kerakli bo'limni tanlang:</i>"
    )
    await message.answer(
        welcome_text,
        reply_markup=main_reply_keyboard(is_admin=is_adm, is_owner=is_own),
        parse_mode=ParseMode.HTML
    )

@router.callback_query(F.data == "check_subscription")
async def callback_check_subscription(callback: CallbackQuery, bot: Bot):
    user_id = callback.from_user.id
    channels = await db.get_mandatory_channels()
    unsubscribed = []

    for ch in channels:
        try:
            member = await bot.get_chat_member(chat_id=ch["channel_id"], user_id=user_id)
            if member.status in [ChatMemberStatus.LEFT, ChatMemberStatus.KICKED]:
                unsubscribed.append(ch)
        except Exception:
            pass

    if unsubscribed:
        await callback.answer("❌ Hali hamma kanallarga a'zo bo'lmadingiz!", show_alert=True)
    else:
        await callback.answer("✅ Rahmat! Obuna tasdiqlandi.", show_alert=False)
        try:
            await callback.message.delete()
        except Exception:
            pass
        is_adm = await db.is_admin(user_id)
        is_own = (user_id == OWNER_ID)
        await callback.message.answer(
            "🎉 <b>Tabriklaymiz, obuna tasdiqlandi!</b>\nAsosiy menyudan foydalanishingiz mumkin.",
            reply_markup=main_reply_keyboard(is_admin=is_adm, is_owner=is_own),
            parse_mode=ParseMode.HTML
        )

@router.message(F.text == "❌ Bekor qilish")
async def process_cancel(message: Message, state: FSMContext):
    await state.clear()
    is_adm = await db.is_admin(message.from_user.id)
    is_own = (message.from_user.id == OWNER_ID)
    await message.answer(
        "❌ Amaliyot bekor qilindi.",
        reply_markup=main_reply_keyboard(is_admin=is_adm, is_owner=is_own)
    )

# --- ANIME QIDIRISH ---
@router.message(F.text == "🔍 Anime qidirish")
async def menu_search(message: Message):
    text = (
        "🔍 <b>Anime qidirish usulini tanlang:</b>\n\n"
        "• <b>Kod orqali</b> — Anime unikal raqamini kiritib darhol topish.\n"
        "• <b>Nomi orqali</b> — O'zbekcha, Inglizcha yoki Yaponcha nomini yozish."
    )
    await message.answer(text, reply_markup=search_type_inline_keyboard(), parse_mode=ParseMode.HTML)

@router.callback_query(F.data == "search_by_code")
async def cb_search_by_code(callback: CallbackQuery, state: FSMContext):
    await state.set_state(SearchStates.waiting_for_code)
    await callback.message.answer(
        "🔢 <b>Anime kodini kiriting:</b>\n<i>(Masalan: 101, 102)</i>",
        reply_markup=cancel_reply_keyboard(),
        parse_mode=ParseMode.HTML
    )
    await callback.answer()

@router.message(SearchStates.waiting_for_code)
async def process_search_code(message: Message, state: FSMContext):
    if not message.text.isdigit():
        await message.answer("⚠️ Iltimos, faqat raqamdan iborat kod kiriting:")
        return

    code = int(message.text)
    await state.clear()
    await show_anime_by_code(message, code, message.from_user.id)

@router.callback_query(F.data == "search_by_title")
async def cb_search_by_title(callback: CallbackQuery, state: FSMContext):
    await state.set_state(SearchStates.waiting_for_title)
    await callback.message.answer(
        "📝 <b>Anime nomini yozing:</b>\n<i>(Masalan: Naruto, Solo Leveling, Titan)</i>",
        reply_markup=cancel_reply_keyboard(),
        parse_mode=ParseMode.HTML
    )
    await callback.answer()

@router.message(SearchStates.waiting_for_title)
async def process_search_title(message: Message, state: FSMContext):
    query = message.text.strip()
    await state.clear()
    results = await db.search_animes(query, limit=10)

    if not results:
        is_adm = await db.is_admin(message.from_user.id)
        is_own = (message.from_user.id == OWNER_ID)
        await message.answer(
            f"😔 Kechirasiz, «<b>{query}</b>» so'rovi bo'yicha hech qanday anime topilmadi.\n"
            "«✍️ Zayavka / Taklif» bo'limi orqali ushbu animeni qo'shishni so'rashingiz mumkin!",
            reply_markup=main_reply_keyboard(is_admin=is_adm, is_owner=is_own),
            parse_mode=ParseMode.HTML
        )
        return

    buttons = []
    for anime in results:
        vip_badge = "👑 " if anime["is_vip"] else "🎬 "
        title_btn = f"{vip_badge}[{anime['code']}] {anime['title'][:32]}"
        buttons.append([InlineKeyboardButton(text=title_btn, callback_data=f"anime_{anime['code']}")])

    kb = InlineKeyboardMarkup(inline_keyboard=buttons)
    await message.answer(
        f"🔎 «<b>{query}</b>» bo'yicha topilgan animelar:",
        reply_markup=kb,
        parse_mode=ParseMode.HTML
    )

# Raqamli to'g'ridan-to'g'ri kod kiritilganda avtomatik qidirish
@router.message(F.text.regexp(r"^\d+$"))
async def direct_code_handler(message: Message):
    code = int(message.text)
    await show_anime_by_code(message, code, message.from_user.id)

async def show_anime_by_code(message: Message, code: int, user_id: int):
    anime = await db.get_anime_by_code(code)
    if not anime:
        await message.answer(f"❌ <b>{code}</b> kodli anime ma'lumotlar bazasidan topilmadi!", parse_mode=ParseMode.HTML)
        return

    is_fav = await db.is_favorite(user_id, code)
    vip_status = "👑 VIP Eksklyuziv" if anime["is_vip"] else "🟢 Bepul"
    
    caption = (
        f"🎬 <b>{anime['title']}</b>\n"
        f"🇯🇵 <i>{anime['original_title'] or ''}</i>\n\n"
        f"🔢 <b>Kodi:</b> <code>{anime['code']}</code>\n"
        f"🎭 <b>Janrlar:</b> {anime['genres']}\n"
        f"📅 <b>Yili:</b> {anime['year']}\n"
        f"🎞️ <b>Qismlar:</b> {anime['episodes_count']} ta\n"
        f"👁️ <b>Ko'rishlar:</b> {anime['views']}\n"
        f"💎 <b>Holati:</b> {vip_status}\n\n"
        f"📖 <b>Tavsif:</b>\n{anime['description'] or 'Tavsif mavjud emas.'}"
    )

    kb = anime_detail_keyboard(code, is_fav)

    # Agar rasm file_id yoki URL mavjud bo'lsa rasm yuborish
    poster = anime.get("poster_file_id")
    if poster and (poster.startswith("http://") or poster.startswith("https://") or len(poster) > 20):
        try:
            await message.answer_photo(photo=poster, caption=caption, reply_markup=kb, parse_mode=ParseMode.HTML)
            return
        except Exception as e:
            logger.warning(f"Rasm yuborishda xato: {e}")

    await message.answer(caption, reply_markup=kb, parse_mode=ParseMode.HTML)

@router.callback_query(F.data.startswith("anime_"))
async def cb_show_anime(callback: CallbackQuery):
    code = int(callback.data.replace("anime_", ""))
    await show_anime_by_code(callback.message, code, callback.from_user.id)
    await callback.answer()

# --- JANRLAR ---
@router.message(F.text == "📂 Janrlar")
async def menu_genres(message: Message):
    text = (
        "📂 <b>Janrlar katalogi:</b>\n"
        "O'zingizga yoqqan yo'nalishdagi animelarni topish uchun quyidagi janrlardan birini tanlang:"
    )
    await message.answer(text, reply_markup=genres_inline_keyboard(), parse_mode=ParseMode.HTML)

@router.callback_query(F.data.startswith("genre_"))
async def cb_genre_selected(callback: CallbackQuery):
    genre_name = callback.data.replace("genre_", "")
    animes = await db.get_animes_by_genre(genre_name, limit=8)

    if not animes:
        await callback.answer(f"«{genre_name}» janrida hali animelar mavjud emas.", show_alert=True)
        return

    buttons = []
    for a in animes:
        buttons.append([InlineKeyboardButton(text=f"[{a['code']}] {a['title'][:32]}", callback_data=f"anime_{a['code']}")])
    buttons.append([InlineKeyboardButton(text="⬅️ Janrlarga qaytish", callback_data="back_to_genres")])

    kb = InlineKeyboardMarkup(inline_keyboard=buttons)
    await callback.message.edit_text(
        f"📂 <b>«{genre_name}» janridagi mashhur animelar:</b>",
        reply_markup=kb,
        parse_mode=ParseMode.HTML
    )
    await callback.answer()

@router.callback_query(F.data == "back_to_genres")
async def cb_back_to_genres(callback: CallbackQuery):
    await callback.message.edit_text(
        "📂 <b>Janrlar katalogi:</b>\nTanlang:",
        reply_markup=genres_inline_keyboard(),
        parse_mode=ParseMode.HTML
    )

# --- TASODIFIY ANIME ---
@router.message(F.text == "🎲 Tasodifiy anime")
async def menu_random_anime(message: Message):
    anime = await db.get_random_anime()
    if not anime:
        await message.answer("😔 Hozircha bazada animelar mavjud emas.")
        return
    await show_anime_by_code(message, anime["code"], message.from_user.id)

@router.callback_query(F.data == "random_anime")
async def cb_random_anime(callback: CallbackQuery):
    anime = await db.get_random_anime()
    if not anime:
        await callback.answer("Animelar topilmadi!", show_alert=True)
        return
    await show_anime_by_code(callback.message, anime["code"], callback.from_user.id)
    await callback.answer()

# --- SEVIMLILAR ---
@router.message(F.text == "⭐ Sevimlilar")
async def menu_favorites(message: Message):
    favs = await db.get_user_favorites(message.from_user.id)
    if not favs:
        await message.answer(
            "⭐ <b>Sevimlilar ro'yxatingiz bo'sh!</b>\n\n"
            "Anime kartochkasidagi «⭐ Sevimlilarga saqlash» tugmasini bosib shu yerga to'plashingiz mumkin.",
            parse_mode=ParseMode.HTML
        )
        return

    buttons = []
    for a in favs:
        buttons.append([InlineKeyboardButton(text=f"🎬 [{a['code']}] {a['title']}", callback_data=f"anime_{a['code']}")])

    kb = InlineKeyboardMarkup(inline_keyboard=buttons)
    await message.answer(f"⭐ <b>Sizning sevimli animelaringiz ({len(favs)} ta):</b>", reply_markup=kb, parse_mode=ParseMode.HTML)

@router.callback_query(F.data.startswith("fav_toggle_"))
async def cb_toggle_favorite(callback: CallbackQuery):
    code = int(callback.data.replace("fav_toggle_", ""))
    user_id = callback.from_user.id
    added = await db.toggle_favorite(user_id, code)

    status_text = "✅ Sevimlilarga qo'shildi!" if added else "❌ Sevimlilardan olib tashlandi!"
    await callback.answer(status_text, show_alert=False)

    # Tugmani yangilash
    anime = await db.get_anime_by_code(code)
    if anime:
        new_kb = anime_detail_keyboard(code, is_fav=added)
        try:
            await callback.message.edit_reply_markup(reply_markup=new_kb)
        except Exception:
            pass

# --- QISMLARNI KO'RISH VA TOMOSHA QILISH ---
@router.callback_query(F.data.startswith("watch_"))
async def cb_watch_episodes(callback: CallbackQuery):
    code = int(callback.data.replace("watch_", ""))
    user_id = callback.from_user.id

    anime = await db.get_anime_by_code(code)
    if not anime:
        await callback.answer("Anime topilmadi!", show_alert=True)
        return

    # VIP tekshiruvi
    if anime["is_vip"] and not await db.is_vip(user_id):
        text = (
            "👑 <b>Bu anime faqat VIP foydalanuvchilar uchun!</b>\n\n"
            "Ushbu animeni tomosha qilish uchun «👑 VIP Kabinet» orqali VIP obunaga ega bo'ling yoki promokodni faollashtiring."
        )
        await callback.message.answer(text, parse_mode=ParseMode.HTML)
        await callback.answer()
        return

    episodes = await db.get_episodes(code)
    if not episodes:
        await callback.answer("😔 Ushbu animening qismlari tez orada yuklanadi!", show_alert=True)
        return

    kb = episodes_inline_keyboard(code, episodes, page=1)
    await callback.message.answer(
        f"🎬 <b>«{anime['title']}» — Qismlar ro'yxati:</b>\nKerakli seriyani tanlang:",
        reply_markup=kb,
        parse_mode=ParseMode.HTML
    )
    await callback.answer()

@router.callback_query(F.data.startswith("ep_page_"))
async def cb_episodes_pagination(callback: CallbackQuery):
    parts = callback.data.split("_")
    code = int(parts[2])
    page = int(parts[3])

    episodes = await db.get_episodes(code)
    kb = episodes_inline_keyboard(code, episodes, page=page)
    await callback.message.edit_reply_markup(reply_markup=kb)
    await callback.answer()

@router.callback_query(F.data.startswith("play_"))
async def cb_play_episode(callback: CallbackQuery):
    parts = callback.data.split("_")
    code = int(parts[1])
    ep_num = int(parts[2])
    user_id = callback.from_user.id

    anime = await db.get_anime_by_code(code)
    if anime and anime["is_vip"] and not await db.is_vip(user_id):
        await callback.answer("👑 Faqat VIP foydalanuvchilar tomosha qilishi mumkin!", show_alert=True)
        return

    episode = await db.get_episode(code, ep_num)
    if not episode:
        await callback.answer("Bu qism topilmadi!", show_alert=True)
        return

    caption = (
        f"🎬 <b>{anime['title'] if anime else 'Anime'}</b>\n"
        f"🎞️ <b>{ep_num}-qism</b> {('— ' + episode['title']) if episode['title'] else ''}\n"
        f"🔢 Kodi: <code>{code}</code>\n\n"
        "🍿 <i>Yoqimli tomosha tilaymiz!</i>"
    )

    video_file = episode["video_file_id"]
    try:
        await callback.message.answer_video(video=video_file, caption=caption, parse_mode=ParseMode.HTML)
        await callback.answer(f"{ep_num}-qism yuklanmoqda...")
    except Exception as e:
        logger.error(f"Video yuborishda xato: {e}")
        await callback.message.answer(
            f"📹 <b>{anime['title'] if anime else 'Anime'} — {ep_num}-qism</b>\n\n"
            f"🔗 Video manzili / file_id: <code>{video_file}</code>",
            parse_mode=ParseMode.HTML
        )
        await callback.answer()

# --- VIP KABINET VA PROMO-KOD ---
@router.message(F.text == "👑 VIP Kabinet")
async def menu_vip_cabinet(message: Message):
    user_id = message.from_user.id
    is_vip_active = await db.is_vip(user_id)
    expiry = await db.get_vip_info(user_id)

    if is_vip_active and expiry:
        status_text = f"🟢 <b>Faol</b>\n📅 <b>Tugash muddati:</b> <code>{expiry.strftime('%Y-%m-%d %H:%M')}</code>"
    elif user_id == OWNER_ID:
        status_text = "👑 <b>Bot Egasi (Cheksiz VIP)</b>"
    else:
        status_text = "🔴 <b>Faol emas</b> (Oddiy foydalanuvchi)"

    text = (
        "👑 <b>VIP OBUNA KABINETI</b>\n\n"
        f"👤 Sizning holatingiz: {status_text}\n\n"
        "🌟 <b>VIP obunachi bo'lish afzalliklari:</b>\n"
        "• Barcha eksklyuziv va premyera animelarga cheksiz kirish\n"
        "• Maksimal 1080p Full HD video sifat\n"
        "• Reklamasiz va tezkor yuklash\n"
        "• Shaxsiy anime buyurtma qilish huquqi\n\n"
        "🔑 Agar sizda promokod bo'lsa, quyidagi tugmani bosib faollashtiring:"
    )

    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🔑 Promokodni faollashtirish", callback_data="activate_vip_code")],
        [InlineKeyboardButton(text="💳 VIP sotib olish (Admin)", url="https://t.me/rasulovyoutuber1")]
    ])
    await message.answer(text, reply_markup=kb, parse_mode=ParseMode.HTML)

@router.callback_query(F.data == "activate_vip_code")
async def cb_activate_vip(callback: CallbackQuery, state: FSMContext):
    await state.set_state(VipStates.waiting_for_code)
    await callback.message.answer(
        "🔑 <b>VIP promokodingizni yuboring:</b>\n<i>(Masalan: VIP-A1B2C3D4-30D)</i>",
        reply_markup=cancel_reply_keyboard(),
        parse_mode=ParseMode.HTML
    )
    await callback.answer()

@router.message(VipStates.waiting_for_code)
async def process_vip_code(message: Message, state: FSMContext):
    code = message.text.strip()
    await state.clear()

    days = await db.redeem_vip_code(message.from_user.id, code)
    is_adm = await db.is_admin(message.from_user.id)
    is_own = (message.from_user.id == OWNER_ID)

    if days:
        await message.answer(
            f"🎉 <b>Tabriklaymiz!</b>\n"
            f"VIP obuna muvaffaqiyatli <b>{days} kunga</b> faollashtirildi!",
            reply_markup=main_reply_keyboard(is_admin=is_adm, is_owner=is_own),
            parse_mode=ParseMode.HTML
        )
    else:
        await message.answer(
            "❌ <b>Noto'g'ri yoki allaqachon ishlatilgan VIP kod!</b>",
            reply_markup=main_reply_keyboard(is_admin=is_adm, is_owner=is_own),
            parse_mode=ParseMode.HTML
        )

# --- ZAYAVKA / TAKLIF YUBORISH ---
@router.message(F.text == "✍️ Zayavka / Taklif")
async def menu_request(message: Message, state: FSMContext):
    await state.set_state(RequestStates.waiting_for_title)
    text = (
        "✍️ <b>Botga qaysi animeni qo'shishimizni xohlaysiz?</b>\n\n"
        "Iltimos, animening aniq nomini yozib yuboring:"
    )
    await message.answer(text, reply_markup=cancel_reply_keyboard(), parse_mode=ParseMode.HTML)

@router.message(RequestStates.waiting_for_title)
async def process_request_title(message: Message, state: FSMContext):
    await state.update_data(anime_title=message.text.strip())
    await state.set_state(RequestStates.waiting_for_comment)
    await message.answer(
        "📝 Qo'shimcha izoh yoki taklifingiz bormi? (Masalan: qaysi studiya ovozi kerak):\n"
        "<i>(Agar yo'q bo'lsa, shunchaki «Yo'q» deb yozing)</i>",
        parse_mode=ParseMode.HTML
    )

@router.message(RequestStates.waiting_for_comment)
async def process_request_comment(message: Message, state: FSMContext, bot: Bot):
    data = await state.get_data()
    anime_title = data.get("anime_title")
    comment = message.text.strip()
    user = message.from_user

    await state.clear()
    req_id = await db.add_request(user.id, user.username, anime_title, comment)

    is_adm = await db.is_admin(user.id)
    is_own = (user.id == OWNER_ID)

    await message.answer(
        "✅ <b>Taklifingiz muvaffaqiyatli qabul qilindi!</b>\n"
        "Tez orada adminlar ko'rib chiqib, animeni bazaga qo'shadilar.",
        reply_markup=main_reply_keyboard(is_admin=is_adm, is_owner=is_own),
        parse_mode=ParseMode.HTML
    )

    # Egasi yoki adminlarga xabarnoma yuborish
    try:
        admin_notice = (
            f"🔔 <b>YANGI ANIME TAKLIFI! #{req_id}</b>\n\n"
            f"👤 Foydalanuvchi: {user.full_name} (@{user.username or 'yoq'}) [<code>{user.id}</code>]\n"
            f"🎬 Anime: <b>{anime_title}</b>\n"
            f"💬 Izoh: {comment}"
        )
        await bot.send_message(chat_id=OWNER_ID, text=admin_notice, parse_mode=ParseMode.HTML)
    except Exception as e:
        logger.warning(f"Adminlarga zayavkani jo'natishda xato: {e}")

# --- MA'LUMOT VA YORDAM ---
@router.message(F.text == "ℹ️ Ma'lumot va Yordam")
async def menu_info(message: Message):
    text = (
        "ℹ️ <b>ANIME BOT HAQIDA MA'LUMOT</b>\n\n"
        "🤖 Bu bot orqali siz sevimli animelaringizni bepul va VIP tarzda tomosha qilishingiz mumkin.\n\n"
        "📌 <b>Qanday qidirish mumkin?</b>\n"
        "1. Shunchaki anime kodini yozing (Masalan: <code>101</code>)\n"
        "2. «🔍 Anime qidirish» tugmasi orqali nomini qidiring\n"
        "3. Janrlar bo'yicha saralang\n\n"
        "👑 VIP obuna olish yoki hamkorlik uchun: @rasulovyoutuber1"
    )
    await message.answer(text, parse_mode=ParseMode.HTML)

# ------------------------------------------------------------------------------
# 7. ADMIN PANEL HANDLERLARI
# ------------------------------------------------------------------------------
@router.message(F.text == "⚙️ Admin Paneli")
async def menu_admin_panel(message: Message):
    if not (await db.is_admin(message.from_user.id)):
        await message.answer("⛔ Sizda admin huquqlari mavjud emas!")
        return

    text = (
        "⚙️ <b>ADMIN BOSHQARUV PANELI</b>\n\n"
        "Quyidagi amallardan birini tanlang:"
    )
    await message.answer(text, reply_markup=admin_panel_inline_keyboard(), parse_mode=ParseMode.HTML)

@router.callback_query(F.data == "adm_stats")
async def cb_admin_stats(callback: CallbackQuery):
    if not (await db.is_admin(callback.from_user.id)):
        await callback.answer("Ruxsat berilmagan!", show_alert=True)
        return

    stats = await db.get_stats()
    text = (
        "📊 <b>BOT STATISTIKASI:</b>\n\n"
        f"👥 Foydalanuvchilar soni: <b>{stats['users']} ta</b>\n"
        f"🎬 Animelar soni: <b>{stats['animes']} ta</b>\n"
        f"🎞️ Yuklangan qismlar: <b>{stats['episodes']} ta</b>\n"
        f"👑 Faol VIP a'zolar: <b>{stats['vips']} ta</b>\n"
        f"✍️ Jami zayavkalar: <b>{stats['requests']} ta</b>"
    )
    await callback.message.answer(text, parse_mode=ParseMode.HTML)
    await callback.answer()

# --- ANIME QO'SHISH VIZARDI (FSM) ---
@router.callback_query(F.data == "adm_add_anime")
async def cb_adm_add_anime(callback: CallbackQuery, state: FSMContext):
    if not (await db.is_admin(callback.from_user.id)):
        return
    await state.set_state(AdminAnimeStates.waiting_for_code)
    await callback.message.answer(
        "➕ <b>Yangi anime qo'shish (1/8):</b>\nUnikal kodni kiriting (faqat son, masalan: 106):",
        reply_markup=cancel_reply_keyboard(),
        parse_mode=ParseMode.HTML
    )
    await callback.answer()

@router.message(AdminAnimeStates.waiting_for_code)
async def process_adm_anime_code(message: Message, state: FSMContext):
    if not message.text.isdigit():
        await message.answer("⚠️ Faqat butun son kiriting:")
        return
    code = int(message.text)
    exists = await db.get_anime_by_code(code)
    if exists:
        await message.answer(f"⚠️ {code} kodi band! Boshqa kod kiriting:")
        return

    await state.update_data(code=code)
    await state.set_state(AdminAnimeStates.waiting_for_title)
    await message.answer("📝 <b>Anime nomini kiriting (2/8):</b>\n(Masalan: Bleach: Ming yillik qonli urush)", parse_mode=ParseMode.HTML)

@router.message(AdminAnimeStates.waiting_for_title)
async def process_adm_anime_title(message: Message, state: FSMContext):
    await state.update_data(title=message.text.strip())
    await state.set_state(AdminAnimeStates.waiting_for_original_title)
    await message.answer("🇯🇵 <b>Asl yoki inglizcha nomini kiriting (3/8):</b>\n(Masalan: Bleach: Thousand-Year Blood War)", parse_mode=ParseMode.HTML)

@router.message(AdminAnimeStates.waiting_for_original_title)
async def process_adm_anime_orig_title(message: Message, state: FSMContext):
    await state.update_data(original_title=message.text.strip())
    await state.set_state(AdminAnimeStates.waiting_for_genres)
    await message.answer("🎭 <b>Janrlarini kiriting (4/8):</b>\n(Vergul bilan ajrating, masalan: Jangari, Shounen, Sarguzasht)", parse_mode=ParseMode.HTML)

@router.message(AdminAnimeStates.waiting_for_genres)
async def process_adm_anime_genres(message: Message, state: FSMContext):
    await state.update_data(genres=message.text.strip())
    await state.set_state(AdminAnimeStates.waiting_for_year)
    await message.answer("📅 <b>Chiqarilgan yilini kiriting (5/8):</b>\n(Masalan: 2022)", parse_mode=ParseMode.HTML)

@router.message(AdminAnimeStates.waiting_for_year)
async def process_adm_anime_year(message: Message, state: FSMContext):
    year = int(message.text) if message.text.isdigit() else 2024
    await state.update_data(year=year)
    await state.set_state(AdminAnimeStates.waiting_for_episodes_count)
    await message.answer("🎞️ <b>Jami qismlar sonini kiriting (6/8):</b>\n(Masalan: 12 yoki 24)", parse_mode=ParseMode.HTML)

@router.message(AdminAnimeStates.waiting_for_episodes_count)
async def process_adm_anime_ep_count(message: Message, state: FSMContext):
    ep_count = int(message.text) if message.text.isdigit() else 12
    await state.update_data(episodes_count=ep_count)
    await state.set_state(AdminAnimeStates.waiting_for_description)
    await message.answer("📖 <b>Anime tavsifini (mazmunini) kiriting (7/8):</b>", parse_mode=ParseMode.HTML)

@router.message(AdminAnimeStates.waiting_for_description)
async def process_adm_anime_desc(message: Message, state: FSMContext):
    await state.update_data(description=message.text.strip())
    await state.set_state(AdminAnimeStates.waiting_for_poster)
    await message.answer(
        "🖼️ <b>Anime posterini yuboring (8/8):</b>\n"
        "Rasm yuboring yoki rasmning URL manzilini matn sifatida yuboring:",
        parse_mode=ParseMode.HTML
    )

@router.message(AdminAnimeStates.waiting_for_poster)
async def process_adm_anime_poster(message: Message, state: FSMContext):
    poster_id = ""
    if message.photo:
        poster_id = message.photo[-1].file_id
    elif message.text:
        poster_id = message.text.strip()
    else:
        await message.answer("⚠️ Iltimos, rasm yoki havola yuboring:")
        return

    data = await state.get_data()
    data["poster_file_id"] = poster_id

    # Saqlash
    success = await db.add_anime(
        code=data["code"],
        title=data["title"],
        original_title=data.get("original_title", ""),
        genres=data["genres"],
        year=data["year"],
        episodes_count=data["episodes_count"],
        description=data["description"],
        poster_file_id=poster_id,
        is_vip=0
    )

    await state.clear()
    is_adm = await db.is_admin(message.from_user.id)
    is_own = (message.from_user.id == OWNER_ID)

    if success:
        await message.answer(
            f"🎉 <b>Anime muvaffaqiyatli saqlandi!</b>\n\n"
            f"🎬 Nomi: <b>{data['title']}</b>\n"
            f"🔢 Kodi: <code>{data['code']}</code>\n\n"
            "Endi unga qismlar yuklashingiz yoki kanalga post qilishingiz mumkin.",
            reply_markup=main_reply_keyboard(is_admin=is_adm, is_owner=is_own),
            parse_mode=ParseMode.HTML
        )
    else:
        await message.answer("❌ Xatolik yuz berdi. Qayta urinib ko'ring.", reply_markup=main_reply_keyboard(is_adm, is_own))

# --- QISM (SERIYA) QO'SHISH ---
@router.callback_query(F.data == "adm_add_episode")
async def cb_adm_add_episode(callback: CallbackQuery, state: FSMContext):
    if not (await db.is_admin(callback.from_user.id)):
        return
    await state.set_state(AdminEpisodeStates.waiting_for_anime_code)
    await callback.message.answer(
        "🎬 <b>Qism yuklash (1/3):</b>\nAnime kodini kiriting:",
        reply_markup=cancel_reply_keyboard(),
        parse_mode=ParseMode.HTML
    )
    await callback.answer()

@router.message(AdminEpisodeStates.waiting_for_anime_code)
async def process_adm_ep_anime_code(message: Message, state: FSMContext):
    if not message.text.isdigit():
        await message.answer("Faqat son kiriting:")
        return
    code = int(message.text)
    anime = await db.get_anime_by_code(code)
    if not anime:
        await message.answer(f"❌ {code} kodli anime topilmadi. Qayta kiriting:")
        return

    await state.update_data(anime_code=code, anime_title=anime["title"])
    await state.set_state(AdminEpisodeStates.waiting_for_episode_num)
    await message.answer(f"🎞️ <b>«{anime['title']}»</b> uchun qism raqamini kiriting (masalan: 1, 2, 3...):", parse_mode=ParseMode.HTML)

@router.message(AdminEpisodeStates.waiting_for_episode_num)
async def process_adm_ep_num(message: Message, state: FSMContext):
    if not message.text.isdigit():
        await message.answer("Faqat raqam kiriting:")
        return
    ep_num = int(message.text)
    await state.update_data(episode_num=ep_num)
    await state.set_state(AdminEpisodeStates.waiting_for_video)
    await message.answer(
        f"📹 <b>{ep_num}-qism videosini yuboring:</b>\n"
        "(Telegram orqali video yuklang yoki video file_id sini matn ko'rinishida yuboring)",
        parse_mode=ParseMode.HTML
    )

@router.message(AdminEpisodeStates.waiting_for_video)
async def process_adm_ep_video(message: Message, state: FSMContext):
    video_id = ""
    if message.video:
        video_id = message.video.file_id
    elif message.document and message.document.mime_type and message.document.mime_type.startswith("video/"):
        video_id = message.document.file_id
    elif message.text:
        video_id = message.text.strip()
    else:
        await message.answer("⚠️ Iltimos, video fayl yoki video file_id sini yuboring:")
        return

    data = await state.get_data()
    code = data["anime_code"]
    ep_num = data["episode_num"]

    success = await db.add_episode(code, ep_num, video_id, title=f"{ep_num}-qism")
    await state.clear()
    is_adm = await db.is_admin(message.from_user.id)
    is_own = (message.from_user.id == OWNER_ID)

    if success:
        await message.answer(
            f"✅ <b>Qism muvaffaqiyatli saqlandi!</b>\n\n"
            f"🎬 Anime: <b>{data.get('anime_title', code)}</b>\n"
            f"🎞️ Qism: <b>{ep_num}-qism</b>",
            reply_markup=main_reply_keyboard(is_admin=is_adm, is_owner=is_own),
            parse_mode=ParseMode.HTML
        )
    else:
        await message.answer("❌ Saqlashda xatolik yuz berdi.", reply_markup=main_reply_keyboard(is_adm, is_own))

# --- KANALGA POST QILISH ---
@router.callback_query(F.data == "adm_post_channel")
async def cb_adm_post_channel(callback: CallbackQuery, state: FSMContext):
    if not (await db.is_admin(callback.from_user.id)):
        return
    channels = await db.get_post_channels()
    if not channels:
        await callback.message.answer(
            "⚠️ Hozircha post kanallari ulanmagan! Owner panelidan avval kanal qo'shing."
        )
        await callback.answer()
        return

    await state.set_state(AdminPostStates.waiting_for_anime_code)
    await callback.message.answer(
        "📢 <b>Kanalga post qilish:</b>\nKanalga chiqarmoqchi bo'lgan anime kodini kiriting:",
        reply_markup=cancel_reply_keyboard(),
        parse_mode=ParseMode.HTML
    )
    await callback.answer()

@router.message(AdminPostStates.waiting_for_anime_code)
async def process_adm_post_anime(message: Message, state: FSMContext, bot: Bot):
    if not message.text.isdigit():
        await message.answer("Faqat son kiriting:")
        return
    code = int(message.text)
    anime = await db.get_anime_by_code(code)
    if not anime:
        await message.answer("❌ Bunday kodli anime topilmadi!")
        return

    channels = await db.get_post_channels()
    me = await bot.get_me()

    post_caption = (
        f"🎬 <b>{anime['title']}</b>\n"
        f"🇯🇵 <i>{anime['original_title'] or ''}</i>\n\n"
        f"🔢 <b>Kodi:</b> <code>{anime['code']}</code>\n"
        f"🎭 <b>Janrlar:</b> {anime['genres']}\n"
        f"📅 <b>Yili:</b> {anime['year']}\n"
        f"🎞️ <b>Qismlar:</b> {anime['episodes_count']} ta\n\n"
        f"📖 <b>Mazmuni:</b>\n{anime['description'][:300]}...\n\n"
        f"🍿 <i>Animeni to'liq tomosha qilish uchun pastdagi tugmani bosing:</i>"
    )

    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🎬 Tomosha qilish (Botda)", url=f"https://t.me/{me.username}?start=code_{code}")]
    ])

    sent_count = 0
    for ch in channels:
        try:
            poster = anime.get("poster_file_id")
            if poster and (poster.startswith("http") or len(poster) > 20):
                await bot.send_photo(chat_id=ch["channel_id"], photo=poster, caption=post_caption, reply_markup=kb, parse_mode=ParseMode.HTML)
            else:
                await bot.send_message(chat_id=ch["channel_id"], text=post_caption, reply_markup=kb, parse_mode=ParseMode.HTML)
            sent_count += 1
        except Exception as e:
            logger.error(f"Kanalga post jo'natishda xatolik ({ch['channel_id']}): {e}")

    await state.clear()
    is_adm = await db.is_admin(message.from_user.id)
    is_own = (message.from_user.id == OWNER_ID)

    await message.answer(
        f"✅ <b>Post muvaffaqiyatli jo'natildi!</b>\n{sent_count} ta kanalga chiqarildi.",
        reply_markup=main_reply_keyboard(is_admin=is_adm, is_owner=is_own),
        parse_mode=ParseMode.HTML
    )

# ------------------------------------------------------------------------------
# 8. OWNER (EGASI) PANEL HANDLERLARI
# ------------------------------------------------------------------------------
@router.message(F.text == "👑 Egasi (Owner) Paneli")
async def menu_owner_panel(message: Message):
    if message.from_user.id != OWNER_ID:
        await message.answer("⛔ Bu bo'lim faqat bot egasi uchun!")
        return

    text = (
        "👑 <b>BOT EGASI (OWNER) BOSHQARUV MARKAZI</b>\n\n"
        "Barcha oliy darajadagi boshqaruv amallari:"
    )
    await message.answer(text, reply_markup=owner_panel_inline_keyboard(), parse_mode=ParseMode.HTML)

# --- ADMIN QO'SHISH ---
@router.callback_query(F.data == "own_add_admin")
async def cb_own_add_admin(callback: CallbackQuery, state: FSMContext):
    if callback.from_user.id != OWNER_ID:
        return
    await state.set_state(OwnerAdminStates.waiting_for_admin_id)
    await callback.message.answer(
        "👮 <b>Yangi admin qo'shish:</b>\n"
        "Foydalanuvchining Telegram ID raqamini kiriting (masalan: 987654321):",
        reply_markup=cancel_reply_keyboard(),
        parse_mode=ParseMode.HTML
    )
    await callback.answer()

@router.message(OwnerAdminStates.waiting_for_admin_id)
async def process_own_admin_id(message: Message, state: FSMContext):
    if not message.text.isdigit():
        await message.answer("⚠️ Faqat raqamli Telegram ID kiriting:")
        return

    new_admin_id = int(message.text)
    await db.add_admin(new_admin_id, message.from_user.id)
    await state.clear()

    await message.answer(
        f"✅ <b>Admin muvaffaqiyatli qo'shildi!</b>\nID: <code>{new_admin_id}</code>",
        reply_markup=main_reply_keyboard(is_admin=True, is_owner=True),
        parse_mode=ParseMode.HTML
    )

# --- ADMINLAR RO'YXATI ---
@router.callback_query(F.data == "own_list_admins")
async def cb_own_list_admins(callback: CallbackQuery):
    if callback.from_user.id != OWNER_ID:
        return

    admins = await db.get_admins()
    if not admins:
        await callback.message.answer("Hozircha qo'shimcha adminlar yo'q (Faqat siz).")
        await callback.answer()
        return

    text = "📋 <b>Mavjud Adminlar:</b>\n\n"
    buttons = []
    for adm in admins:
        text += f"• ID: <code>{adm['user_id']}</code> | Qo'shilgan: {adm['added_at']}\n"
        buttons.append([InlineKeyboardButton(text=f"🗑️ O'chirish ({adm['user_id']})", callback_data=f"del_adm_{adm['user_id']}")])

    kb = InlineKeyboardMarkup(inline_keyboard=buttons)
    await callback.message.answer(text, reply_markup=kb, parse_mode=ParseMode.HTML)
    await callback.answer()

@router.callback_query(F.data.startswith("del_adm_"))
async def cb_del_admin(callback: CallbackQuery):
    if callback.from_user.id != OWNER_ID:
        return
    admin_id = int(callback.data.replace("del_adm_", ""))
    await db.remove_admin(admin_id)
    await callback.answer(f"Admin {admin_id} o'chirildi!", show_alert=True)
    try:
        await callback.message.delete()
    except Exception:
        pass

# --- MAJBURIR OBUNA KANALLARINI SOZLASH ---
@router.callback_query(F.data == "own_channels")
async def cb_own_channels(callback: CallbackQuery):
    if callback.from_user.id != OWNER_ID:
        return

    channels = await db.get_mandatory_channels()
    if not channels:
        await callback.message.answer("Hozircha majburiy kanallar belgilanmagan.")
        await callback.answer()
        return

    text = "📢 <b>Majburiy obuna kanallari:</b>\n\n"
    buttons = []
    for ch in channels:
        text += f"• <b>{ch['title']}</b> (ID: <code>{ch['channel_id']}</code>)\nHavola: {ch['invite_link']}\n\n"
        buttons.append([InlineKeyboardButton(text=f"🗑️ O'chirish: {ch['title'][:20]}", callback_data=f"del_ch_{ch['channel_id']}")])

    kb = InlineKeyboardMarkup(inline_keyboard=buttons)
    await callback.message.answer(text, reply_markup=kb, parse_mode=ParseMode.HTML)
    await callback.answer()

@router.callback_query(F.data.startswith("del_ch_"))
async def cb_del_channel(callback: CallbackQuery):
    if callback.from_user.id != OWNER_ID:
        return
    ch_id = callback.data.replace("del_ch_", "")
    await db.remove_mandatory_channel(ch_id)
    await callback.answer("Kanal o'chirildi!", show_alert=True)
    try:
        await callback.message.delete()
    except Exception:
        pass

@router.callback_query(F.data == "own_add_channel")
async def cb_own_add_channel(callback: CallbackQuery, state: FSMContext):
    if callback.from_user.id != OWNER_ID:
        return
    await state.set_state(OwnerChannelStates.waiting_for_channel_id)
    text = (
        "➕ <b>Majburiy obuna kanalini qo'shish (1/3):</b>\n\n"
        "Kanalning ID raqamini yoki @username sini kiriting:\n"
        "<i>(Masalan: -1001234567890 yoki @anime_uzb)</i>\n\n"
        "⚠️ <b>Muhim:</b> Bot ushbu kanalda <b>Admin</b> bo'lishi shart!"
    )
    await callback.message.answer(text, reply_markup=cancel_reply_keyboard(), parse_mode=ParseMode.HTML)
    await callback.answer()

@router.message(OwnerChannelStates.waiting_for_channel_id)
async def process_channel_id(message: Message, state: FSMContext):
    await state.update_data(channel_id=message.text.strip())
    await state.set_state(OwnerChannelStates.waiting_for_channel_title)
    await message.answer("📝 Kanal nomini kiriting (2/3):\n(Masalan: Anime Dunyosi | Rasmiy)", parse_mode=ParseMode.HTML)

@router.message(OwnerChannelStates.waiting_for_channel_title)
async def process_channel_title(message: Message, state: FSMContext):
    await state.update_data(title=message.text.strip())
    await state.set_state(OwnerChannelStates.waiting_for_invite_link)
    await message.answer("🔗 Kanalga ulanish havolasini kiriting (3/3):\n(Masalan: https://t.me/anime_uzb)", parse_mode=ParseMode.HTML)

@router.message(OwnerChannelStates.waiting_for_invite_link)
async def process_channel_link(message: Message, state: FSMContext):
    data = await state.get_data()
    invite_link = message.text.strip()
    channel_id = data["channel_id"]
    title = data["title"]

    await db.add_mandatory_channel(channel_id, title, invite_link)
    await state.clear()

    await message.answer(
        f"✅ <b>Kanal muvaffaqiyatli saqlandi!</b>\n"
        f"📢 Nomi: <b>{title}</b>\n"
        f"🆔 ID: <code>{channel_id}</code>\n"
        f"🔗 Havola: {invite_link}",
        reply_markup=main_reply_keyboard(is_admin=True, is_owner=True),
        parse_mode=ParseMode.HTML
    )

# --- VIP PROMOKOD YARATISH ---
@router.callback_query(F.data == "own_create_vip")
async def cb_own_create_vip(callback: CallbackQuery):
    if callback.from_user.id != OWNER_ID:
        return

    kb = InlineKeyboardMarkup(inline_keyboard=[
        [
            InlineKeyboardButton(text="7 Kunlik", callback_data="gen_vip_7"),
            InlineKeyboardButton(text="30 Kunlik (1 oy)", callback_data="gen_vip_30")
        ],
        [
            InlineKeyboardButton(text="90 Kunlik (3 oy)", callback_data="gen_vip_90"),
            InlineKeyboardButton(text="365 Kunlik (1 yil)", callback_data="gen_vip_365")
        ]
    ])
    await callback.message.answer("🔑 <b>Qancha muddatli VIP kod yaratmoqchisiz?</b>", reply_markup=kb, parse_mode=ParseMode.HTML)
    await callback.answer()

@router.callback_query(F.data.startswith("gen_vip_"))
async def cb_gen_vip(callback: CallbackQuery):
    if callback.from_user.id != OWNER_ID:
        return
    days = int(callback.data.replace("gen_vip_", ""))
    code = await db.create_vip_code(days, callback.from_user.id)

    text = (
        "🎉 <b>YANGI VIP PROMOKOD YARATILDI!</b>\n\n"
        f"🔑 Promokod: <code>{code}</code>\n"
        f"⏳ Amal qilish muddati: <b>{days} kun</b>\n\n"
        "<i>Ushbu kodni foydalanuvchiga bering, u «👑 VIP Kabinet» orqali faollashtiradi.</i>"
    )
    await callback.message.answer(text, parse_mode=ParseMode.HTML)
    await callback.answer()

# --- BROADCAST (BARCHAGA XABARNOMA) ---
@router.callback_query(F.data == "own_broadcast")
async def cb_own_broadcast(callback: CallbackQuery, state: FSMContext):
    if callback.from_user.id != OWNER_ID:
        return
    await state.set_state(OwnerBroadcastStates.waiting_for_message)
    await callback.message.answer(
        "📨 <b>Barcha foydalanuvchilarga yuboriladigan xabarni yozing:</b>\n"
        "Matn, rasm yoki video yuborishingiz mumkin. Bekor qilish uchun «❌ Bekor qilish» tugmasini bosing.",
        reply_markup=cancel_reply_keyboard(),
        parse_mode=ParseMode.HTML
    )
    await callback.answer()

@router.message(OwnerBroadcastStates.waiting_for_message)
async def process_broadcast_message(message: Message, state: FSMContext, bot: Bot):
    await state.clear()
    users = await db.get_all_user_ids()
    await message.answer(f"⏳ <b>Xabarnoma tarqatilmoqda...</b>\nJami: {len(users)} ta foydalanuvchi.", parse_mode=ParseMode.HTML)

    success_count = 0
    fail_count = 0

    for uid in users:
        try:
            await message.copy_to(chat_id=uid)
            success_count += 1
            await asyncio.sleep(0.04)  # Telegram flood limitidan saqlanish
        except Exception:
            fail_count += 1

    await message.answer(
        f"✅ <b>Xabarnoma yakunlandi!</b>\n\n"
        f"📬 Yetkazildi: {success_count} ta\n"
        f"❌ Yetkazilmadi (bloklangan): {fail_count} ta",
        reply_markup=main_reply_keyboard(is_admin=True, is_owner=True),
        parse_mode=ParseMode.HTML
    )

# ------------------------------------------------------------------------------
# 9. RENDER CLOUD VA KEEP-ALIVE SERVERI
# ------------------------------------------------------------------------------
async def health_check_handler(request: web.Request) -> web.Response:
    """Render.com Web Service lari uchun 200 OK qaytaruvchi server"""
    return web.json_response({
        "status": "online",
        "service": "Aiogram 3 Anime Telegram Bot",
        "author": "@rasulovyoutuber1",
        "time": datetime.utcnow().isoformat()
    })

async def run_render_webserver():
    """Render portini (masalan, 10000) tinglab turadi"""
    app = web.Application()
    app.router.add_get("/", health_check_handler)
    app.router.add_get("/health", health_check_handler)
    
    runner = web.AppRunner(app)
    await runner.setup()
    site = web.TCPSite(runner, host="0.0.0.0", port=PORT)
    await site.start()
    logger.info(f"🌐 Render web server http://0.0.0.0:{PORT} manzilida ishga tushdi.")

# ------------------------------------------------------------------------------
# 10. ASOSIY ISHGA TUSHIRISH (MAIN ENTRYPOINT)
# ------------------------------------------------------------------------------
async def main():
    logger.info("🚀 Anime bot tizimi ishga tushirilmoqda...")
    
    # 1. Bazani sozlash
    await db.init()

    # 2. Bot va Dispatcher yaratish
    bot = Bot(token=BOT_TOKEN, default=DefaultBotProperties(parse_mode=ParseMode.HTML))
    dp = Dispatcher(storage=MemoryStorage())

    # 3. Middlewares ro'yxatdan o'tkazish
    dp.update.outer_middleware(MandatorySubscriptionMiddleware())

    # 4. Routerlarni ulash
    dp.include_router(router)

    # 5. Render web serverini parallel ishga tushirish (agar PORT mavjud bo'lsa)
    web_server_task = asyncio.create_task(run_render_webserver())

    # 6. Eski xabarlarni tozalash (polling oldidan)
    try:
        await bot.delete_webhook(drop_pending_updates=True)
        bot_info = await bot.get_me()
        logger.info(f"✅ Bot muvaffaqiyatli ishga tushdi: @{bot_info.username}")
    except Exception as e:
        logger.warning(f"Bot ma'lumotlarini olishda ogohlantirish (Token to'g'riligini tekshiring): {e}")

    # 7. Pollingni boshlash
    try:
        await dp.start_polling(bot)
    finally:
        web_server_task.cancel()
        await bot.session.close()

if __name__ == "__main__":
    try:
        asyncio.run(main())
    except (KeyboardInterrupt, SystemExit):
        logger.info("🛑 Bot to'xtatildi!")
