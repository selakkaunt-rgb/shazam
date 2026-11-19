#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
🎵 MUSIQA + SHAZAM TELEGRAM BOTI — bitta fayl (main.py)

Nima qiladi
  * 🔗 Havoladan yuklash   — YouTube, Shorts, Instagram, TikTok... -> [🎞 Video] [🎧 Audio]
  * 🔍 Qidiruv             — nom / ijrochi / qo'shiq matni -> 10 tadan natija (3 sahifa) -> mp3
  * 🎤 Shazam              — ovozli xabar, audio, video yuborsangiz qo'shiqni topadi
  * 👥 Guruhda ovozli chat — /play, /skip, /pause, /resume, /stop, /queue, /shazam

Kutubxonalar (pip install):
    pip install "pyrofork>=2.3.35,<3" "py-tgcalls>=2.2.0,<3" "yt-dlp[default]" \
                "shazamio>=0.7,<1" "aiohttp>=3.9" "python-dotenv>=1.0"
    pip install tgcrypto-pyrofork          # ixtiyoriy, faqat tezlik uchun
Tizimda kerak:
    ffmpeg   (majburiy)
    deno     (YouTube uchun yt-dlp talab qiladi, https://deno.com)
Python 3.10 yoki yangiroq.

Environment o'zgaruvchilari (.env fayl yoki hosting sozlamalari):
    API_ID, API_HASH, BOT_TOKEN, SESSION_STRING          — majburiy
    MAX_FILE_MB (100), PORT (8080), COOKIES_FILE         — ixtiyoriy

Ishga tushirish:
    python main.py              — botni ishga tushiradi
    python main.py session      — yordamchi akkaunt uchun SESSION_STRING yaratadi (bir marta)

Bitta jarayonda ishlaydi:
  * bot (BOT_TOKEN)                    — buyruqlar, tugmalar, fayl yuborish
  * yordamchi akkaunt (SESSION_STRING) — guruhdagi ovozli chatda ijro (py-tgcalls)
  * kichik aiohttp server              — Render / UptimeRobot "uyg'oq tutish" so'rovlari uchun
"""
import sys

if sys.version_info < (3, 10):
    sys.exit("Bu bot Python 3.10 yoki yangiroq versiyani talab qiladi.")

import asyncio
import contextlib
import functools
import glob
import html
import logging
import math
import os
import random
import re
import secrets
import shutil
import tempfile
from collections import OrderedDict
from dataclasses import dataclass, field
from typing import Any, Callable, Iterator

import aiohttp
from aiohttp import web
from pyrogram import Client, enums, filters, idle, raw
from pyrogram.errors import (
    FloodWait,
    InviteHashExpired,
    InviteHashInvalid,
    MessageNotModified,
    RPCError,
    UserAlreadyParticipant,
    UserBannedInChannel,
    UserNotParticipant,
)
from pyrogram.handlers import CallbackQueryHandler, MessageHandler
from pyrogram.types import (
    CallbackQuery,
    ChatMember,
    ChatPrivileges,
    InlineKeyboardButton,
    InlineKeyboardMarkup,
    Message,
)
from pytgcalls import PyTgCalls
from pytgcalls.exceptions import AlreadyJoinedError, NoActiveGroupCall
from pytgcalls.types import AudioQuality, ChatUpdate, MediaStream, Update
from pytgcalls.types.stream import StreamEnded
from shazamio import Shazam
from yt_dlp import YoutubeDL

try:  # python-dotenv ixtiyoriy: hostingda o'zgaruvchilar to'g'ridan-to'g'ri beriladi
    from dotenv import load_dotenv

    load_dotenv()
except ImportError:
    pass

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)s %(name)s: %(message)s",
)
# Kutubxonalarning ortiqcha loglarini kamaytiramiz (UptimeRobot har daqiqa so'rov yuboradi)
logging.getLogger("pyrogram").setLevel(logging.WARNING)
logging.getLogger("aiohttp.access").setLevel(logging.WARNING)

log = logging.getLogger("musiqa-bot")


# =====================================================================================
# 1) SOZLAMALAR
# =====================================================================================
def _text(name: str) -> str:
    """Majburiy matnli o'zgaruvchini o'qiydi."""
    value = os.getenv(name, "").strip()
    if not value:
        raise RuntimeError(
            f"{name} o'zgaruvchisi o'rnatilmagan (.env faylini yoki hosting sozlamalarini tekshiring)"
        )
    return value


def _number(name: str, default: int | None = None) -> int:
    """Butun sonli o'zgaruvchini o'qiydi (default bo'lmasa majburiy)."""
    raw_value = os.getenv(name, "").strip()
    if not raw_value:
        if default is None:
            raise RuntimeError(f"{name} o'zgaruvchisi o'rnatilmagan")
        return default
    try:
        return int(raw_value)
    except ValueError:
        raise RuntimeError(f"{name} butun son bo'lishi kerak, hozir: {raw_value!r}") from None


def _prepare_cookies() -> str | None:
    """
    COOKIES_FILE ni vaqtinchalik yozish mumkin bo'lgan joyga nusxalaydi.
    Sabab: yt-dlp cookie faylni qayta yozishga urinadi, hosting esa
    (masalan, Render Secret Files) faylni faqat o'qish rejimida beradi.
    """
    source = os.getenv("COOKIES_FILE", "").strip()
    if not source:
        return None
    if not os.path.isfile(source):
        log.warning("COOKIES_FILE topilmadi: %s (cookies ishlatilmaydi)", source)
        return None
    target = os.path.join(tempfile.gettempdir(), "yt_cookies.txt")
    try:
        shutil.copyfile(source, target)
        return target
    except OSError as exc:
        log.warning("Cookies nusxalanmadi (%s), asl fayl ishlatiladi", exc)
        return source


class Config:
    """Barcha sozlamalar. Ishga tushishda Config.load() bir marta to'ldiradi."""

    API_ID: int = 0
    API_HASH: str = ""
    BOT_TOKEN: str = ""
    SESSION_STRING: str = ""
    MAX_FILE_MB: int = 100
    MAX_FILE_BYTES: int = 100 * 1024 * 1024
    PORT: int = 8080
    COOKIES_FILE: str | None = None

    @classmethod
    def load(cls) -> None:
        cls.API_ID = _number("API_ID")
        cls.API_HASH = _text("API_HASH")
        cls.BOT_TOKEN = _text("BOT_TOKEN")
        cls.SESSION_STRING = _text("SESSION_STRING")
        cls.MAX_FILE_MB = max(1, _number("MAX_FILE_MB", 100))
        cls.MAX_FILE_BYTES = cls.MAX_FILE_MB * 1024 * 1024
        cls.PORT = _number("PORT", 8080)
        cls.COOKIES_FILE = _prepare_cookies()


# =====================================================================================
# 2) UMUMIY YORDAMCHI FUNKSIYALAR
# =====================================================================================
URL_RE = re.compile(r"https?://[^\s<>\"']+", re.IGNORECASE)
ERR_GENERIC = "❌ Kutilmagan xatolik yuz berdi. Birozdan keyin qayta urinib ko'ring."


# ---------------------------------------------------------------- matn yordamchilari
def esc(text: Any) -> str:
    """HTML parse_mode uchun matnni xavfsiz qiladi."""
    return html.escape(str(text or ""), quote=False)


def clip(text: str, limit: int) -> str:
    """Matnni belgilangan uzunlikkacha qisqartiradi."""
    text = (text or "").strip()
    return text if len(text) <= limit else text[: limit - 1].rstrip() + "…"


def fmt_duration(seconds: Any) -> str:
    """Soniyalarni 3:53 yoki 1:02:05 ko'rinishiga o'tkazadi."""
    try:
        total = int(seconds or 0)
    except (TypeError, ValueError):
        total = 0
    hours, rest = divmod(total, 3600)
    minutes, secs = divmod(rest, 60)
    return f"{hours}:{minutes:02d}:{secs:02d}" if hours else f"{minutes}:{secs:02d}"


def find_url(text: str | None) -> str | None:
    """Matndagi birinchi havolani qaytaradi (oxiridagi tinish belgilarisiz)."""
    match = URL_RE.search(text or "")
    return match.group(0).rstrip(".,;:!?)]}") if match else None


# ---------------------------------------------------------------- vaqtinchalik saqlash
class Store:
    """Inline tugmalar uchun qisqa token -> ma'lumot saqlovchi (eskilari avtomatik o'chadi)."""

    def __init__(self, limit: int = 500) -> None:
        self._data: OrderedDict[str, Any] = OrderedDict()
        self._limit = limit

    def put(self, value: Any) -> str:
        token = secrets.token_hex(4)
        self._data[token] = value
        while len(self._data) > self._limit:
            self._data.popitem(last=False)
        return token

    def get(self, token: str) -> Any | None:
        value = self._data.get(token)
        if value is not None:
            self._data.move_to_end(token)
        return value

    def pop(self, token: str) -> None:
        self._data.pop(token, None)


@contextlib.contextmanager
def temp_dir() -> Iterator[str]:
    """Vaqtinchalik papka yaratadi va chiqishda (xatolik bo'lsa ham) o'chiradi."""
    path = tempfile.mkdtemp(prefix="musicbot_")
    try:
        yield path
    finally:
        shutil.rmtree(path, ignore_errors=True)


# ---------------------------------------------------------------- bloklovchi ishlar
async def run_blocking(func: Callable[..., Any], *args: Any, **kwargs: Any) -> Any:
    """Bloklovchi funksiyani (yt-dlp va h.k.) executor'da ishga tushiradi."""
    loop = asyncio.get_running_loop()
    return await loop.run_in_executor(None, functools.partial(func, *args, **kwargs))


async def run_ffmpeg(*args: str, timeout: int = 180) -> bool:
    """ffmpeg'ni async subprocess orqali ishga tushiradi. Muvaffaqiyat bo'lsa True."""
    proc = await asyncio.create_subprocess_exec(
        "ffmpeg", "-hide_banner", "-loglevel", "error", *args,
        stdout=asyncio.subprocess.DEVNULL,
        stderr=asyncio.subprocess.PIPE,
    )
    try:
        _, err = await asyncio.wait_for(proc.communicate(), timeout=timeout)
    except asyncio.TimeoutError:
        proc.kill()
        await proc.wait()
        log.warning("ffmpeg vaqti tugadi")
        return False
    if proc.returncode != 0:
        log.warning("ffmpeg xatosi: %s", err.decode(errors="ignore").strip()[-300:])
        return False
    return True


async def fetch_image(url: str | None, folder: str, max_side: int, name: str) -> str | None:
    """
    Rasmni (thumbnail) yuklab, ffmpeg bilan JPEG ga o'tkazadi va kichraytiradi.
    Muvaffaqiyatsiz bo'lsa None qaytaradi (rasm ixtiyoriy, xatolik yuborishga xalaqit bermaydi).
    """
    if not url or not url.startswith("http"):
        return None
    source = os.path.join(folder, f"{name}.src")
    target = os.path.join(folder, f"{name}.jpg")
    headers = {
        "User-Agent": "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 "
                      "(KHTML, like Gecko) Chrome/124.0 Safari/537.36"
    }
    try:
        timeout = aiohttp.ClientTimeout(total=20)
        async with aiohttp.ClientSession(timeout=timeout, headers=headers) as session:
            async with session.get(url) as resp:
                if resp.status != 200:
                    return None
                data = await resp.read()
        if not data or len(data) > 15 * 1024 * 1024:
            return None
        with open(source, "wb") as fh:
            fh.write(data)
        ok = await run_ffmpeg(
            "-y", "-i", source,
            "-vf", f"scale='min({max_side},iw)':-2",
            "-frames:v", "1", "-q:v", "4", target,
            timeout=60,
        )
        return target if ok and os.path.isfile(target) else None
    except Exception as exc:  # rasm yuklanmasa ham ish davom etadi
        log.info("Rasm olinmadi (%s): %s", url[:80], exc)
        return None


# ---------------------------------------------------------------- xavfsiz handlerlar
def safe_message(func: Callable[..., Any]) -> Callable[..., Any]:
    """Xabar handleri xatosini ushlaydi, foydalanuvchiga o'zbekcha aytadi, siklni to'xtatmaydi."""

    @functools.wraps(func)
    async def wrapper(client: Any, message: Any) -> None:
        try:
            await func(client, message)
        except Exception:
            log.exception("Handler xatosi: %s", getattr(func, "__name__", "?"))
            with contextlib.suppress(Exception):
                await message.reply_text(ERR_GENERIC)

    return wrapper


def safe_callback(func: Callable[..., Any]) -> Callable[..., Any]:
    """Inline tugma handleri uchun xuddi shunday himoya."""

    @functools.wraps(func)
    async def wrapper(client: Any, query: Any) -> None:
        try:
            await func(client, query)
        except Exception:
            log.exception("Callback xatosi: %s", getattr(func, "__name__", "?"))
            with contextlib.suppress(Exception):
                await query.answer(ERR_GENERIC, show_alert=True)

    return wrapper


# =====================================================================================
# 3) YT-DLP: qidiruv, ma'lumot olish, yuklab olish, oqim havolasi
# =====================================================================================
# Bir vaqtda nechta yuklash bo'lishi mumkin (bepul hostingda RAM/CPU tejash uchun)
_SLOTS = asyncio.Semaphore(3)
# Video sifati: avval 720p, hajm katta bo'lsa pastroqlari
VIDEO_HEIGHTS = (720, 480, 360)


class TooLargeError(Exception):
    """Fayl MAX_FILE_MB chegarasidan oshdi."""


class NotFoundError(Exception):
    """Qidiruv hech narsa topmadi."""


@dataclass
class SearchItem:
    id: str
    title: str
    url: str
    duration: int
    uploader: str


@dataclass
class AudioResult:
    path: str
    title: str
    performer: str
    duration: int
    thumbnail: str | None


@dataclass
class VideoResult:
    path: str
    title: str
    duration: int
    width: int
    height: int
    thumbnail: str | None


# ---------------------------------------------------------------- umumiy sozlamalar
def _common_opts() -> dict[str, Any]:
    opts: dict[str, Any] = {
        "quiet": True,
        "no_warnings": True,
        "noprogress": True,
        "socket_timeout": 30,
        "retries": 3,
        "extractor_retries": 2,
        # Oddiy veb-sahifalarni "video" deb tekshirib vaqt yo'qotmaslik uchun generic o'chirilgan
        "allowed_extractors": ["default", "-generic"],
    }
    if Config.COOKIES_FILE:
        opts["cookiefile"] = Config.COOKIES_FILE
    return opts


def _media_opts() -> dict[str, Any]:
    """Bitta video/audio uchun sozlamalar (pleylist bo'lsa faqat birinchisi)."""
    return {**_common_opts(), "noplaylist": True, "playlist_items": "1"}


def _first_entry(info: dict[str, Any] | None) -> dict[str, Any]:
    """Pleylist/karusel qaytsa, birinchi elementni oladi."""
    if not info:
        raise RuntimeError("Ma'lumot olinmadi")
    if info.get("_type") == "playlist":
        entries = [e for e in (info.get("entries") or []) if e]
        if not entries:
            raise RuntimeError("Hech narsa topilmadi")
        return entries[0]
    return info


def _clean_performer(name: str) -> str:
    return re.sub(r"\s*-\s*Topic$", "", name or "").strip()


# ---------------------------------------------------------------- qidiruv
def _search_sync(query: str, count: int) -> list[SearchItem]:
    opts = {**_common_opts(), "extract_flat": True, "skip_download": True}
    with YoutubeDL(opts) as ydl:
        data = ydl.extract_info(f"ytsearch{count}:{query}", download=False)
    items: list[SearchItem] = []
    for entry in (data or {}).get("entries") or []:
        if not entry:
            continue
        vid, title, duration = entry.get("id"), entry.get("title"), entry.get("duration")
        if not vid or not title or not duration:  # jonli efirlar va bo'sh natijalar o'tkaziladi
            continue
        url = entry.get("url") or entry.get("webpage_url") or ""
        if not url.startswith("http"):
            url = f"https://www.youtube.com/watch?v={vid}"
        uploader = entry.get("channel") or entry.get("uploader") or ""
        items.append(SearchItem(vid, title, url, int(duration), uploader))
    return items


async def search_raw(query: str, count: int) -> list[SearchItem]:
    """YouTube'dan `ytsearch{count}:` bilan qidiradi."""
    return await run_blocking(_search_sync, query, count)


async def search_many(query: str, count: int = 30) -> list[SearchItem]:
    """
    Foydalanuvchi qidiruvi uchun (3 sahifa x 10 ta natija).
    Natija kam bo'lsa (masalan, qo'shiq matnidan parcha yozilganda) so'rovga
    "lyrics" qo'shib, yana qidiriladi.
    """
    items = await search_raw(query, count)
    if len(items) < 10:
        try:
            extra = await search_raw(f"{query} lyrics", count)
        except Exception as exc:
            log.info("Qo'shimcha qidiruv ishlamadi: %s", exc)
            extra = []
        seen = {item.id for item in items}
        items += [item for item in extra if item.id not in seen]
    return items[:count]


# ---------------------------------------------------------------- ma'lumot / oqim
def _info_sync(url: str) -> dict[str, Any]:
    opts = {**_media_opts(), "skip_download": True, "ignore_no_formats_error": True}
    with YoutubeDL(opts) as ydl:
        return _first_entry(ydl.extract_info(url, download=False))


async def get_info(url: str) -> dict[str, Any]:
    """Havola haqida ma'lumot (sarlavha, rasm, davomiylik...)."""
    return await run_blocking(_info_sync, url)


def _stream_sync(url: str) -> str:
    opts = {**_media_opts(), "format": "bestaudio/best", "skip_download": True}
    with YoutubeDL(opts) as ydl:
        info = _first_entry(ydl.extract_info(url, download=False))
    stream = info.get("url")
    if not stream:
        candidates = [
            f for f in (info.get("formats") or [])
            if f.get("url") and f.get("acodec") not in (None, "none")
        ]
        if candidates:
            stream = candidates[-1]["url"]
    if not stream:
        raise RuntimeError("Audio oqim havolasi topilmadi")
    return stream


async def get_stream_url(url: str) -> str:
    """Ovozli chat uchun eng yaxshi audio oqim URL'i (har safar yangidan olinadi)."""
    return await run_blocking(_stream_sync, url)


# ---------------------------------------------------------------- audio yuklash
def _audio_sync(url: str, out_dir: str) -> tuple[dict[str, Any], str]:
    opts = {
        **_media_opts(),
        "format": "bestaudio/best",
        "outtmpl": os.path.join(out_dir, "audio.%(ext)s"),
        "max_filesize": Config.MAX_FILE_BYTES,
        "postprocessors": [
            {"key": "FFmpegExtractAudio", "preferredcodec": "mp3", "preferredquality": "192"}
        ],
    }
    with YoutubeDL(opts) as ydl:
        info = _first_entry(ydl.extract_info(url, download=True))
    path = os.path.join(out_dir, "audio.mp3")
    if not os.path.isfile(path):
        size = info.get("filesize") or info.get("filesize_approx") or 0
        if size > Config.MAX_FILE_BYTES:
            raise TooLargeError()
        raise RuntimeError("Audio fayl yaratilmadi")
    return info, path


async def download_audio(url: str, out_dir: str) -> AudioResult:
    """Eng yaxshi audioni yuklab, mp3 ga o'tkazadi."""
    async with _SLOTS:
        info, path = await run_blocking(_audio_sync, url, out_dir)
    if os.path.getsize(path) > Config.MAX_FILE_BYTES:
        raise TooLargeError()
    return AudioResult(
        path=path,
        title=(info.get("track") or info.get("title") or "Audio")[:100],
        performer=_clean_performer(
            info.get("artist") or info.get("creator") or info.get("uploader")
            or info.get("channel") or ""
        )[:100],
        duration=int(info.get("duration") or 0),
        thumbnail=info.get("thumbnail"),
    )


# ---------------------------------------------------------------- video yuklash
def _video_sync(url: str, out_dir: str, height: int) -> tuple[dict[str, Any], str | None]:
    # avc1+m4a Telegram'da eng ishonchli ijro etiladi; "?" — balandligi noma'lum formatlarga ruxsat
    fmt = (
        f"bv*[height<=?{height}][vcodec^=avc1]+ba[acodec^=mp4a]/"
        f"bv*[height<=?{height}][ext=mp4]+ba[ext=m4a]/"
        f"b[height<=?{height}][ext=mp4]/"
        f"bv*[height<=?{height}]+ba/"
        f"b[height<=?{height}]/b"
    )
    opts = {
        **_media_opts(),
        "format": fmt,
        "merge_output_format": "mp4",
        "outtmpl": os.path.join(out_dir, "video.%(ext)s"),
        "max_filesize": Config.MAX_FILE_BYTES,
    }
    with YoutubeDL(opts) as ydl:
        info = _first_entry(ydl.extract_info(url, download=True))
    path: str | None = None
    for item in info.get("requested_downloads") or []:
        candidate = item.get("filepath")
        if candidate and os.path.isfile(candidate):
            path = candidate
            break
    if not path:
        found = [
            p for p in glob.glob(os.path.join(out_dir, "video.*"))
            if not p.endswith((".part", ".ytdl"))
        ]
        path = max(found, key=os.path.getsize) if found else None
    return info, path


async def download_video(url: str, out_dir: str) -> VideoResult:
    """Videoni (maks. 720p, MAX_FILE_MB ichida) mp4 qilib yuklaydi."""
    async with _SLOTS:
        for index, height in enumerate(VIDEO_HEIGHTS):
            sub = os.path.join(out_dir, f"v{height}")
            os.makedirs(sub, exist_ok=True)
            info, path = await run_blocking(_video_sync, url, sub, height)
            if path and os.path.getsize(path) <= Config.MAX_FILE_BYTES:
                return VideoResult(
                    path=path,
                    title=info.get("title") or "Video",
                    duration=int(info.get("duration") or 0),
                    width=int(info.get("width") or 0),
                    height=int(info.get("height") or 0),
                    thumbnail=info.get("thumbnail"),
                )
            if path:
                os.remove(path)  # katta fayl: pastroq sifat bilan qayta urinamiz
            next_height = VIDEO_HEIGHTS[index + 1] if index + 1 < len(VIDEO_HEIGHTS) else None
            got = int(info.get("height") or 0)
            if next_height is None or (got and got <= next_height):
                break  # pastroq sifat bo'lmaydi, urinishning ma'nosi yo'q
    raise TooLargeError()


# ---------------------------------------------------------------- xatolarni o'zbekchalashtirish
def friendly_error(exc: BaseException) -> str:
    """yt-dlp va boshqa xatolarni foydalanuvchiga tushunarli o'zbekcha matnga aylantiradi."""
    if isinstance(exc, TooLargeError):
        return f"❌ Fayl hajmi {Config.MAX_FILE_MB} MB chegarasidan oshib ketdi."
    if isinstance(exc, NotFoundError):
        return "❌ Qo'shiq YouTube'da topilmadi."
    text = str(exc).lower()
    if "not a bot" in text or "sign in" in text or "cookies" in text:
        return ("❌ YouTube server so'rovini blokladi. Administrator cookies faylini "
                "(COOKIES_FILE) sozlashi kerak.")
    if "private" in text:
        return "❌ Bu video yopiq (private), yuklab bo'lmaydi."
    if "unsupported url" in text:
        return "❌ Bu havolani qo'llab-quvvatlab bo'lmaydi."
    if "country" in text or "geo" in text:
        return "❌ Bu video sizning mintaqangizda mavjud emas."
    if "unavailable" in text or "removed" in text or "not available" in text:
        return "❌ Video mavjud emas yoki o'chirib tashlangan."
    return "❌ Yuklab bo'lmadi. Havolani tekshirib, qayta urinib ko'ring."


# =====================================================================================
# 4) HANDLERLAR: havoladan yuklash, qidiruv, Shazam
# =====================================================================================
PAGE_SIZE = 10
MAX_PAGES = 3

START_TEXT = (
    "👋 <b>Salom! Men musiqa botiman.</b>\n\n"
    "🔗 <b>Havola</b> yuboring (YouTube, Shorts, Instagram, TikTok...) — video yoki audio qilib yuklab beraman.\n"
    "🔍 <b>Qo'shiq nomi</b>, ijrochi yoki qo'shiq matnidan parcha yozing — qidirib, mp3 yuboraman.\n"
    "🎤 <b>Ovozli xabar</b>, audio yoki video yuboring — qaysi qo'shiq ekanini topib beraman.\n\n"
    "👥 <b>Guruhda:</b>\n"
    "/play qo'shiq nomi — ovozli chatda ijro etish\n"
    "/skip, /pause, /resume, /stop, /queue — boshqaruv\n"
    "/shazam — ovozli xabar, audio yoki videoga reply qilib yozing"
)

# Inline tugmalar uchun vaqtinchalik ma'lumotlar (token -> ma'lumot)
SEARCHES = Store(300)
LINKS = Store(500)
SHAZAMS = Store(500)
# Bir xil tugma ketma-ket bosilganda ikki marta yuklamaslik uchun
BUSY: set[tuple[int, str]] = set()

_shazam: Shazam | None = None


@dataclass
class SearchSession:
    query: str
    results: list[SearchItem]


def _button(text: str, data: str) -> InlineKeyboardButton:
    return InlineKeyboardButton(text, callback_data=data)


def _get_shazam() -> Shazam:
    global _shazam
    if _shazam is None:
        _shazam = Shazam()
    return _shazam


# ---------------------------------------------------------------- filtrlar
async def _is_link_message(_, __, message: Message) -> bool:
    text = message.text or message.caption or ""
    return bool(URL_RE.search(text)) and not text.lstrip().startswith("/")


async def _is_search_text(_, __, message: Message) -> bool:
    text = (message.text or "").strip()
    return bool(text) and not text.startswith("/") and not URL_RE.search(text)


LINK_FILTER = filters.create(_is_link_message, "LinkFilter")
SEARCH_FILTER = filters.create(_is_search_text, "SearchFilter")
MEDIA_FILTER = filters.voice | filters.audio | filters.video | filters.video_note


# ---------------------------------------------------------------- yuborish yordamchilari
async def send_mp3(anchor: Message, url: str) -> None:
    """Havoladan eng yaxshi audioni mp3 qilib yuklaydi va `anchor` xabariga javoban yuboradi."""
    with temp_dir() as folder:
        result = await download_audio(url, folder)
        thumb = await fetch_image(result.thumbnail, folder, 320, "thumb")
        await anchor.reply_audio(
            result.path,
            title=result.title,
            performer=result.performer,
            duration=result.duration,
            thumb=thumb,
        )


async def send_video(anchor: Message, url: str) -> None:
    """Videoni (maks. 720p) yuklab, video qilib yuboradi."""
    with temp_dir() as folder:
        result = await download_video(url, folder)
        thumb = await fetch_image(result.thumbnail, folder, 320, "thumb")
        await anchor.reply_video(
            result.path,
            caption=f"<b>{esc(clip(result.title, 900))}</b>",
            duration=result.duration,
            width=result.width,
            height=result.height,
            thumb=thumb,
            supports_streaming=True,
        )


async def _deliver(query: CallbackQuery, key: str, job) -> None:
    """Holat xabarini ko'rsatib, `job()` ni bajaradi; takroriy bosishlardan himoyalaydi."""
    busy_key = (query.message.chat.id, key)
    if busy_key in BUSY:
        await query.answer("⏳ Bu allaqachon yuklanmoqda...")
        return
    BUSY.add(busy_key)
    status = None
    try:
        await query.answer("⏳ Yuklanmoqda...")
        status = await query.message.reply_text("⏳ Yuklanmoqda...")
        await job()
        with contextlib.suppress(Exception):
            await status.delete()
    except Exception as exc:
        log.warning("Yuklashda xatolik (%s): %s", key, exc, exc_info=not isinstance(exc, TooLargeError))
        text = friendly_error(exc)
        if status is not None:
            with contextlib.suppress(Exception):
                await status.edit_text(text)
        else:
            with contextlib.suppress(Exception):
                await query.message.reply_text(text)
    finally:
        BUSY.discard(busy_key)


# ---------------------------------------------------------------- /start, /help
@safe_message
async def cmd_start(client: Client, message: Message) -> None:
    await message.reply_text(START_TEXT)


# ---------------------------------------------------------------- 1) havoladan yuklash
@safe_message
async def on_link(client: Client, message: Message) -> None:
    url = find_url(message.text or message.caption)
    if not url:
        return
    private = message.chat.type == enums.ChatType.PRIVATE
    # Guruhda har bir havola uchun "kutilmoqda" xabari chiqarib, chatni ifloslantirmaymiz
    status = await message.reply_text("⏳ Havola tekshirilmoqda...") if private else None
    try:
        info = await get_info(url)
    except Exception as exc:
        log.info("Havola tekshirilmadi (%s): %s", url[:80], exc)
        if status:
            await status.edit_text(friendly_error(exc))
        return

    title = info.get("title") or "Nomsiz"
    token = LINKS.put({"url": url, "title": title})
    caption = f"<b>{esc(clip(title, 300))}</b>\n\nYuklab olish formatlari ↓"
    keyboard = InlineKeyboardMarkup(
        [[_button("🎞 Video", f"v:{token}"), _button("🎧 Audio", f"a:{token}")]]
    )
    with temp_dir() as folder:
        photo = await fetch_image(info.get("thumbnail"), folder, 1280, "preview")
        if photo:
            await message.reply_photo(photo, caption=caption, reply_markup=keyboard)
        else:
            await message.reply_text(caption, reply_markup=keyboard)
    if status:
        with contextlib.suppress(Exception):
            await status.delete()


@safe_callback
async def on_link_button(client: Client, query: CallbackQuery) -> None:
    kind, _, token = (query.data or "").partition(":")
    data = LINKS.get(token)
    if not data:
        await query.answer("⌛ Havola eskirgan. Uni qaytadan yuboring.", show_alert=True)
        return
    anchor = query.message
    if kind == "v":
        await _deliver(query, query.data, lambda: send_video(anchor, data["url"]))
    else:
        await _deliver(query, query.data, lambda: send_mp3(anchor, data["url"]))


# ---------------------------------------------------------------- 2) nom / ijrochi / matn bo'yicha qidiruv
def render_page(token: str, session: SearchSession, page: int) -> tuple[str, InlineKeyboardMarkup]:
    """Qidiruv natijalarining bitta sahifasi: matn + klaviatura."""
    start = page * PAGE_SIZE
    chunk = session.results[start:start + PAGE_SIZE]
    lines = [f"🔍 {esc(clip(session.query, 100))}", ""]
    for number, item in enumerate(chunk, 1):
        lines.append(f"{number}. {esc(clip(item.title, 60))} <b>{fmt_duration(item.duration)}</b>")

    numbers = [_button(str(n), f"s:{token}:{start + n - 1}") for n in range(1, len(chunk) + 1)]
    rows = [numbers[:5]]
    if numbers[5:]:
        rows.append(numbers[5:])
    rows.append([
        _button("⬅️", f"p:{token}:{page - 1}"),
        _button("❌", f"x:{token}"),
        _button("➡️", f"p:{token}:{page + 1}"),
    ])
    return "\n".join(lines), InlineKeyboardMarkup(rows)


@safe_message
async def on_search_text(client: Client, message: Message) -> None:
    query = clip((message.text or "").strip(), 200)
    if len(query) < 2:
        return
    status = await message.reply_text("🔎 Qidirilmoqda...")
    try:
        results = await search_many(query, PAGE_SIZE * MAX_PAGES)
    except Exception as exc:
        log.warning("Qidiruv xatosi: %s", exc)
        await status.edit_text(friendly_error(exc))
        return
    if not results:
        await status.edit_text("❌ Hech narsa topilmadi. Boshqacha yozib ko'ring.")
        return
    session = SearchSession(query, results[:PAGE_SIZE * MAX_PAGES])
    token = SEARCHES.put(session)
    text, keyboard = render_page(token, session, 0)
    await status.edit_text(text, reply_markup=keyboard)


@safe_callback
async def on_search_button(client: Client, query: CallbackQuery) -> None:
    parts = (query.data or "").split(":")
    if len(parts) < 2:
        await query.answer()
        return
    action, token = parts[0], parts[1]
    session: SearchSession | None = SEARCHES.get(token)
    if session is None:
        await query.answer("⌛ Qidiruv natijalari eskirgan. Qaytadan yozing.", show_alert=True)
        return

    if action == "x":  # yopish
        SEARCHES.pop(token)
        await query.answer()
        with contextlib.suppress(Exception):
            await query.message.delete()
        return

    if action == "p":  # sahifalash
        page = int(parts[2])
        pages = max(1, math.ceil(len(session.results) / PAGE_SIZE))
        if page < 0:
            await query.answer("Bu birinchi sahifa")
            return
        if page >= pages:
            await query.answer("Bu oxirgi sahifa")
            return
        text, keyboard = render_page(token, session, page)
        await query.answer()
        with contextlib.suppress(MessageNotModified):
            await query.message.edit_text(text, reply_markup=keyboard)
        return

    if action == "s":  # natija tanlandi -> mp3
        index = int(parts[2])
        if not 0 <= index < len(session.results):
            await query.answer("❌ Noto'g'ri tanlov", show_alert=True)
            return
        item = session.results[index]
        anchor = query.message
        await _deliver(query, query.data, lambda: send_mp3(anchor, item.url))


# ---------------------------------------------------------------- 3) Shazam
def _pick_media(message: Message | None):
    if message is None:
        return None
    return message.voice or message.audio or message.video or message.video_note


async def recognize_and_reply(anchor: Message, source: Message) -> None:
    """`source` dagi mediadan qo'shiqni topib, `anchor` xabariga javob beradi."""
    media = _pick_media(source)
    if media is None:
        await anchor.reply_text("❗ Ovozli xabar, audio yoki video kerak.")
        return
    if (getattr(media, "file_size", 0) or 0) > Config.MAX_FILE_BYTES:
        await anchor.reply_text(f"❌ Fayl juda katta (maksimum {Config.MAX_FILE_MB} MB).")
        return

    status = await anchor.reply_text("🎧 Qo'shiq aniqlanmoqda, biroz kuting...")
    try:
        with temp_dir() as folder:
            downloaded = await source.download(file_name=folder + os.sep)
            if not downloaded:
                await status.edit_text("❌ Faylni yuklab olib bo'lmadi.")
                return
            # Shazam uchun faqat birinchi 20 soniya yetarli
            sample = os.path.join(folder, "sample.wav")
            cut_ok = await run_ffmpeg(
                "-y", "-i", downloaded, "-t", "20", "-vn", "-ac", "1", "-ar", "16000", sample,
                timeout=120,
            )
            if not cut_ok or not os.path.isfile(sample):
                await status.edit_text("❌ Audioni o'qib bo'lmadi. Boshqa fayl yuborib ko'ring.")
                return
            try:
                result = await asyncio.wait_for(_get_shazam().recognize(sample), timeout=60)
            except Exception as exc:
                log.warning("Shazam xatosi: %s", exc)
                await status.edit_text("❌ Shazam xizmati bilan bog'lanib bo'lmadi. Keyinroq urinib ko'ring.")
                return

            track = (result or {}).get("track")
            if not track:
                await status.edit_text(
                    "😔 Afsuski, qo'shiq topilmadi. Musiqa aniqroq eshitiladigan, "
                    "boshqa parchani yuborib ko'ring."
                )
                return

            title = track.get("title") or "Noma'lum"
            artist = track.get("subtitle") or "Noma'lum ijrochi"
            images = track.get("images") or {}
            cover = images.get("coverarthq") or images.get("coverart") or images.get("background")
            token = SHAZAMS.put(f"{artist} - {title}")
            caption = f"🎵 <b>{esc(title)}</b>\n👤 {esc(artist)}"
            keyboard = InlineKeyboardMarkup([[_button("⬇️ Yuklab olish", f"z:{token}")]])
            photo = await fetch_image(cover, folder, 1000, "cover")
            if photo:
                await anchor.reply_photo(photo, caption=caption, reply_markup=keyboard)
            else:
                await anchor.reply_text(caption, reply_markup=keyboard)
        with contextlib.suppress(Exception):
            await status.delete()
    except Exception:
        log.exception("Shazam jarayonida xatolik")
        with contextlib.suppress(Exception):
            await status.edit_text("❌ Qo'shiqni aniqlashda xatolik yuz berdi.")


@safe_message
async def on_private_media(client: Client, message: Message) -> None:
    await recognize_and_reply(message, message)


@safe_message
async def cmd_shazam(client: Client, message: Message) -> None:
    source = message.reply_to_message
    if _pick_media(source) is None:
        await message.reply_text(
            "❗ /shazam buyrug'ini ovozli xabar, audio yoki videoga reply qilib yozing."
        )
        return
    await recognize_and_reply(message, source)


@safe_callback
async def on_shazam_button(client: Client, query: CallbackQuery) -> None:
    token = (query.data or "").partition(":")[2]
    wanted = SHAZAMS.get(token)
    if not wanted:
        await query.answer("⌛ Natija eskirgan. Qo'shiqni qaytadan aniqlang.", show_alert=True)
        return
    anchor = query.message

    async def job() -> None:
        items = await search_raw(wanted, 5)
        if not items:
            raise NotFoundError()
        # Birinchi "oddiy" davomiylikdagi natijani olamiz (soatlik miks emas)
        item = next((i for i in items if i.duration <= 900), items[0])
        await send_mp3(anchor, item.url)

    await _deliver(query, query.data, job)


# ---------------------------------------------------------------- ro'yxatdan o'tkazish
def register_handlers(bot: Client) -> None:
    """Havola, qidiruv va Shazam handlerlarini botga ulaydi."""
    bot.add_handler(MessageHandler(cmd_start, filters.command(["start", "help"])))
    bot.add_handler(MessageHandler(on_private_media, filters.private & MEDIA_FILTER))
    bot.add_handler(MessageHandler(cmd_shazam, filters.command("shazam") & filters.group))
    bot.add_handler(MessageHandler(on_link, LINK_FILTER))
    bot.add_handler(MessageHandler(on_search_text, filters.private & SEARCH_FILTER))

    bot.add_handler(CallbackQueryHandler(on_link_button, filters.regex(r"^[va]:")))
    bot.add_handler(CallbackQueryHandler(on_search_button, filters.regex(r"^[spx]:")))
    bot.add_handler(CallbackQueryHandler(on_shazam_button, filters.regex(r"^z:")))


# =====================================================================================
# 5) GURUHDAGI OVOZLI CHAT: yordamchi akkaunt + py-tgcalls 2.x
# =====================================================================================
# Video oqimi kerak emas. py-tgcalls 2.x da bayroq MediaStream.Flags ichida (ba'zi
# versiyalarda MediaStream.IGNORE nomi bilan ham mavjud), shuning uchun ikkalasini qo'llaymiz.
VIDEO_IGNORE = getattr(MediaStream, "Flags", MediaStream).IGNORE

MSG_BOT_NOT_ADMIN = (
    "❌ Men bu guruhda admin emasman. Meni barcha admin huquqlari bilan "
    "(jumladan «Yangi adminlar qo'shish» va «Video chatlarni boshqarish») admin qiling."
)
MSG_NO_INVITE = (
    "❌ Yordamchi akkauntni guruhga taklif qilish huquqim yo'q. "
    "Menga «Foydalanuvchilarni taklif qilish» huquqini bering."
)
MSG_NO_PROMOTE = (
    "❌ Yordamchi akkauntga admin berish huquqim yo'q. Menga «Yangi adminlar qo'shish» "
    "va «Video chatlarni boshqarish» huquqlarini bering."
)
MSG_CALL_FAIL = (
    "❌ Ovozli chat yaratib bo'lmadi. Guruhda ovozli chatni qo'lda yoqib, "
    "qayta urinib ko'ring."
)
MSG_NOT_FOUND = "❌ Qo'shiq topilmadi. Nomini boshqacha yozib ko'ring."
MSG_ASSISTANT_BANNED = (
    "❌ Yordamchi akkaunt bu guruhdan ban qilingan va men uni unban qila olmadim. "
    "Uni qo'lda unban qiling."
)
MSG_PLAY_FAIL = "❌ Qo'shiqni ijro etib bo'lmadi. Birozdan keyin qayta urinib ko'ring."
MSG_NOTHING = "❌ Hozir hech narsa ijro etilmayapti."


class VoiceError(Exception):
    """Matni to'g'ridan-to'g'ri foydalanuvchiga ko'rsatiladigan xatolik."""


@dataclass
class Track:
    title: str
    url: str
    duration: int


@dataclass
class ChatPlayer:
    queue: list[Track] = field(default_factory=list)  # queue[0] — hozir ijro etilayotgani
    paused: bool = False
    lock: asyncio.Lock = field(default_factory=asyncio.Lock)


def _can(member: ChatMember, right: str) -> bool:
    """Admin ma'lum huquqqa egami (egasi barcha huquqqa ega)."""
    if member.status == enums.ChatMemberStatus.OWNER:
        return True
    privileges = getattr(member, "privileges", None)
    return bool(privileges and getattr(privileges, right, False))


def _is_audio_end(update: StreamEnded) -> bool:
    """Faqat audio oqimi tugaganini hisobga olamiz (video oqimi yo'q)."""
    kind = getattr(StreamEnded, "Type", None)
    stream_type = getattr(update, "stream_type", None)
    if kind is None or stream_type is None:
        return True
    return stream_type != getattr(kind, "VIDEO", object())


class VoiceManager:
    """Har bir guruh uchun alohida navbat va ijroni boshqaradi."""

    def __init__(self, bot: Client, assistant: Client, call: PyTgCalls) -> None:
        self.bot = bot
        self.assistant = assistant
        self.call = call
        self.assistant_id = 0
        self.players: dict[int, ChatPlayer] = {}
        # Qo'shiq tugashi va ovozli chatdan chiqib ketish hodisalari
        self.call.on_update()(self._on_update)

    async def init(self) -> None:
        """Yordamchi akkaunt ishga tushgandan keyin chaqiriladi."""
        me = await self.assistant.get_me()
        self.assistant_id = me.id
        log.info("Yordamchi akkaunt: %s (%s)", me.first_name, me.id)

    def player(self, chat_id: int) -> ChatPlayer:
        return self.players.setdefault(chat_id, ChatPlayer())

    async def _say(self, chat_id: int, text: str) -> None:
        try:
            await self.bot.send_message(chat_id, text)
        except Exception as exc:
            log.warning("Xabar yuborilmadi (%s): %s", chat_id, exc)

    # ------------------------------------------------------------ py-tgcalls hodisalari
    async def _on_update(self, _, update: Update) -> None:
        try:
            if isinstance(update, StreamEnded):
                if _is_audio_end(update):
                    await self._track_ended(update.chat_id)
            elif isinstance(update, ChatUpdate):
                # Yordamchi akkaunt chatdan chiqib ketsa (chat yopildi, chiqarib yuborildi...)
                # eski navbat qolib ketmasligi uchun tozalaymiz. Qulf olinmaydi: leave_call
                # o'zimiz qulf ichida chaqirishimiz mumkin.
                if update.status & ChatUpdate.Status.LEFT_CALL:
                    player = self.players.get(update.chat_id)
                    if player:
                        player.queue.clear()
                        player.paused = False
        except Exception:
            log.exception("py-tgcalls hodisasini qayta ishlashda xatolik")

    async def _track_ended(self, chat_id: int) -> None:
        player = self.players.get(chat_id)
        if not player or not player.queue:
            return
        async with player.lock:
            if player.queue:
                player.queue.pop(0)
            await self._play_head(chat_id)

    # ------------------------------------------------------------ tayyorgarlik
    async def _bot_member(self, chat_id: int) -> ChatMember:
        try:
            me = await self.bot.get_chat_member(chat_id, "me")
        except RPCError:
            raise VoiceError(MSG_BOT_NOT_ADMIN) from None
        if me.status not in (enums.ChatMemberStatus.ADMINISTRATOR, enums.ChatMemberStatus.OWNER):
            raise VoiceError(MSG_BOT_NOT_ADMIN)
        return me

    async def ensure_ready(self, chat_id: int) -> None:
        """Yordamchi akkaunt guruhda ekanini va faol ovozli chat borligini ta'minlaydi."""
        bot_member = await self._bot_member(chat_id)
        await self._ensure_assistant_in_chat(chat_id, bot_member)
        await self._warm_peer(chat_id)
        if not await self._has_call(chat_id):
            await self._create_call(chat_id, bot_member)

    async def _ensure_assistant_in_chat(self, chat_id: int, bot_member: ChatMember) -> None:
        status = None
        in_chat = False
        try:
            member = await self.bot.get_chat_member(chat_id, self.assistant_id)
            status = member.status
            in_chat = status in (
                enums.ChatMemberStatus.MEMBER,
                enums.ChatMemberStatus.ADMINISTRATOR,
                enums.ChatMemberStatus.OWNER,
            ) or (status == enums.ChatMemberStatus.RESTRICTED and bool(member.is_member))
        except UserNotParticipant:
            in_chat = False
        if in_chat:
            return

        # Ban qilingan bo'lsa — avval unban
        if status == enums.ChatMemberStatus.BANNED:
            try:
                await self.bot.unban_chat_member(chat_id, self.assistant_id)
            except RPCError:
                raise VoiceError(MSG_ASSISTANT_BANNED) from None

        if not _can(bot_member, "can_invite_users"):
            raise VoiceError(MSG_NO_INVITE)
        try:
            link = await self.bot.create_chat_invite_link(
                chat_id, name="Musiqa yordamchisi", member_limit=1
            )
        except RPCError:
            raise VoiceError(MSG_NO_INVITE) from None

        try:
            await self.assistant.join_chat(link.invite_link)
        except UserAlreadyParticipant:
            pass
        except UserBannedInChannel:
            raise VoiceError(MSG_ASSISTANT_BANNED) from None
        except FloodWait as exc:
            raise VoiceError(
                f"❌ Yordamchi akkaunt vaqtincha cheklangan. {exc.value} soniyadan keyin qayta urinib ko'ring."
            ) from None
        except (InviteHashExpired, InviteHashInvalid):
            raise VoiceError(MSG_NO_INVITE) from None
        except RPCError:
            log.exception("Yordamchi akkaunt guruhga qo'shila olmadi")
            raise VoiceError(MSG_NO_INVITE) from None
        finally:
            # Bir martalik havolani darhol bekor qilamiz
            with contextlib.suppress(Exception):
                await self.bot.revoke_chat_invite_link(chat_id, link.invite_link)
        await asyncio.sleep(1)

    async def _warm_peer(self, chat_id: int) -> None:
        """
        Yordamchi sessiya xotirada saqlanadi (in_memory), shuning uchun qayta ishga tushganda
        guruhni "tanimasligi" mumkin. Shunda dialoglarni bir marta yuklab, keshni to'ldiramiz.
        """
        with contextlib.suppress(Exception):
            await self.assistant.resolve_peer(chat_id)
            return
        try:
            async for _ in self.assistant.get_dialogs(limit=500):
                pass
        except Exception as exc:
            log.warning("Dialoglarni yuklab bo'lmadi: %s", exc)
        try:
            await self.assistant.resolve_peer(chat_id)
        except Exception:
            raise VoiceError(MSG_CALL_FAIL) from None

    async def _has_call(self, chat_id: int) -> bool:
        """Guruhda faol ovozli chat bormi."""
        try:
            peer = await self.assistant.resolve_peer(chat_id)
            if isinstance(peer, raw.types.InputPeerChannel):
                full = await self.assistant.invoke(
                    raw.functions.channels.GetFullChannel(
                        channel=raw.types.InputChannel(
                            channel_id=peer.channel_id, access_hash=peer.access_hash
                        )
                    )
                )
            elif isinstance(peer, raw.types.InputPeerChat):
                full = await self.assistant.invoke(
                    raw.functions.messages.GetFullChat(chat_id=peer.chat_id)
                )
            else:
                return False
        except RPCError:
            log.exception("Guruh ma'lumotini olib bo'lmadi")
            raise VoiceError(MSG_CALL_FAIL) from None
        return getattr(full.full_chat, "call", None) is not None

    async def _create_call(self, chat_id: int, bot_member: ChatMember | None = None) -> None:
        """Ovozli chat yaratadi: bot yordamchiga huquq beradi, yordamchi `CreateGroupCall` chaqiradi."""
        if bot_member is None:
            bot_member = await self._bot_member(chat_id)
        try:
            assistant_member = await self.bot.get_chat_member(chat_id, self.assistant_id)
        except RPCError:
            raise VoiceError(MSG_CALL_FAIL) from None

        if not _can(assistant_member, "can_manage_video_chats"):
            if not _can(bot_member, "can_promote_members"):
                raise VoiceError(MSG_NO_PROMOTE)
            try:
                await self.bot.promote_chat_member(
                    chat_id,
                    self.assistant_id,
                    privileges=ChatPrivileges(can_manage_chat=True, can_manage_video_chats=True),
                )
            except RPCError:
                log.exception("Yordamchiga admin berib bo'lmadi")
                raise VoiceError(MSG_NO_PROMOTE) from None
            await asyncio.sleep(1)

        try:
            peer = await self.assistant.resolve_peer(chat_id)
            await self.assistant.invoke(
                raw.functions.phone.CreateGroupCall(
                    peer=peer, random_id=random.randint(1, 2**31 - 1)
                )
            )
        except RPCError:
            log.exception("Ovozli chat yaratilmadi")
            raise VoiceError(MSG_CALL_FAIL) from None
        await asyncio.sleep(1)

    # ------------------------------------------------------------ ijro
    async def _leave(self, chat_id: int) -> None:
        with contextlib.suppress(Exception):
            await self.call.leave_call(chat_id)

    async def _play_track(self, chat_id: int, track: Track) -> None:
        """Eng yaxshi audio URL'ni olib, ovozli chatga uzatadi."""
        stream_url = await get_stream_url(track.url)

        def make_stream() -> MediaStream:
            return MediaStream(
                stream_url,
                audio_parameters=AudioQuality.HIGH,
                video_flags=VIDEO_IGNORE,
            )

        try:
            await self.call.play(chat_id, make_stream())
        except NoActiveGroupCall:
            # Ovozli chat bu orada yopilgan bo'lishi mumkin: qayta yaratib, yana urinamiz
            await self._create_call(chat_id)
            await self.call.play(chat_id, make_stream())
        except AlreadyJoinedError:
            await self._leave(chat_id)
            await asyncio.sleep(1)
            await self.call.play(chat_id, make_stream())

    async def _play_head(self, chat_id: int) -> None:
        """
        Navbat boshidagi qo'shiqni ijro etadi (qulf ushlab turilgan holda chaqiriladi).
        Xatolik bo'lsa keyingisiga o'tadi, navbat bo'sh qolsa ovozli chatdan chiqadi.
        """
        player = self.player(chat_id)
        while player.queue:
            track = player.queue[0]
            try:
                await self._play_track(chat_id, track)
            except Exception:
                log.exception("Qo'shiq ijro etilmadi: %s", track.title)
                await self._say(chat_id, f"❌ Ijro etib bo'lmadi: <b>{esc(clip(track.title, 80))}</b>")
                player.queue.pop(0)
                continue
            player.paused = False
            await self._say(chat_id, f"🎶 Hozir ijro etilmoqda: <b>{esc(track.title)}</b>")
            return
        player.paused = False
        await self._leave(chat_id)
        await self._say(chat_id, "✅ Navbat tugadi, ovozli chatdan chiqdim.")

    async def _resolve(self, query: str) -> Track:
        """Matn yoki havoladan qo'shiqni topadi."""
        try:
            if query.lower().startswith(("http://", "https://")):
                url = find_url(query) or query
                info = await get_info(url)
                return Track(
                    title=info.get("title") or "Nomsiz",
                    url=info.get("webpage_url") or url,
                    duration=int(info.get("duration") or 0),
                )
            items = await search_raw(query, 5)
        except Exception as exc:
            log.info("Qo'shiq topilmadi (%s): %s", query[:60], exc)
            raise VoiceError(MSG_NOT_FOUND) from None
        if not items:
            raise VoiceError(MSG_NOT_FOUND)
        first = items[0]
        return Track(first.title, first.url, first.duration)

    # ------------------------------------------------------------ buyruqlar
    async def cmd_play(self, client: Client, message: Message) -> None:
        chat_id = message.chat.id
        query = " ".join(message.command[1:]).strip()
        if not query:
            await message.reply_text(
                "❗ Foydalanish: /play qo'shiq nomi yoki ijrochi - qo'shiq\n"
                "YouTube havolasini ham yuborishingiz mumkin."
            )
            return
        status = await message.reply_text("🔎 Qidirilmoqda...")
        player = self.player(chat_id)
        try:
            track = await self._resolve(query)
            async with player.lock:
                if player.queue:
                    player.queue.append(track)
                    await status.edit_text(
                        f"➕ Navbatga qo'shildi (#{len(player.queue)}): <b>{esc(track.title)}</b>"
                    )
                    return
                await status.edit_text("⏳ Ovozli chatga ulanmoqda...")
                await self.ensure_ready(chat_id)
                player.queue.append(track)
                try:
                    await self._play_track(chat_id, track)
                except Exception:
                    log.exception("Ijro boshlanmadi (chat %s)", chat_id)
                    player.queue.clear()
                    await self._leave(chat_id)
                    raise VoiceError(MSG_PLAY_FAIL) from None
                player.paused = False
        except VoiceError as exc:
            await status.edit_text(str(exc))
            return
        except Exception:
            log.exception("/play xatosi (chat %s)", chat_id)
            await status.edit_text(MSG_PLAY_FAIL)
            return
        await status.edit_text(f"🎶 Hozir ijro etilmoqda: <b>{esc(track.title)}</b>")

    async def cmd_skip(self, client: Client, message: Message) -> None:
        chat_id = message.chat.id
        player = self.players.get(chat_id)
        if not player or not player.queue:
            await message.reply_text(MSG_NOTHING)
            return
        async with player.lock:
            if not player.queue:
                await message.reply_text(MSG_NOTHING)
                return
            skipped = player.queue.pop(0)
            await message.reply_text(f"⏭ O'tkazib yuborildi: <b>{esc(clip(skipped.title, 80))}</b>")
            await self._play_head(chat_id)

    async def cmd_pause(self, client: Client, message: Message) -> None:
        chat_id = message.chat.id
        player = self.players.get(chat_id)
        if not player or not player.queue:
            await message.reply_text(MSG_NOTHING)
            return
        if player.paused:
            await message.reply_text("⏸ Ijro allaqachon to'xtatilgan. Davom ettirish: /resume")
            return
        try:
            await self.call.pause(chat_id)
        except Exception:
            log.exception("Pauza xatosi")
            await message.reply_text("❌ Pauza qilib bo'lmadi.")
            return
        player.paused = True
        await message.reply_text("⏸ Pauza qilindi. Davom ettirish: /resume")

    async def cmd_resume(self, client: Client, message: Message) -> None:
        chat_id = message.chat.id
        player = self.players.get(chat_id)
        if not player or not player.queue:
            await message.reply_text(MSG_NOTHING)
            return
        if not player.paused:
            await message.reply_text("▶️ Ijro to'xtatilmagan.")
            return
        try:
            await self.call.resume(chat_id)
        except Exception:
            log.exception("Davom ettirish xatosi")
            await message.reply_text("❌ Davom ettirib bo'lmadi.")
            return
        player.paused = False
        await message.reply_text("▶️ Ijro davom etmoqda.")

    async def cmd_stop(self, client: Client, message: Message) -> None:
        chat_id = message.chat.id
        player = self.players.get(chat_id)
        if not player or not player.queue:
            await message.reply_text(MSG_NOTHING)
            return
        async with player.lock:
            player.queue.clear()
            player.paused = False
            await self._leave(chat_id)
        await message.reply_text("⏹ Ijro to'xtatildi, navbat tozalandi.")

    async def cmd_queue(self, client: Client, message: Message) -> None:
        player = self.players.get(message.chat.id)
        if not player or not player.queue:
            await message.reply_text("📭 Navbat bo'sh.")
            return
        lines = ["📜 <b>Navbat:</b>"]
        for number, track in enumerate(player.queue[:15], 1):
            mark = "▶️" if number == 1 else "▫️"
            duration = fmt_duration(track.duration) if track.duration else "—"
            lines.append(f"{mark} {number}. {esc(clip(track.title, 60))} <b>{duration}</b>")
        if len(player.queue) > 15:
            lines.append(f"... va yana {len(player.queue) - 15} ta")
        await message.reply_text("\n".join(lines))

    async def cmd_private(self, client: Client, message: Message) -> None:
        await message.reply_text(
            "ℹ️ Bu buyruq faqat guruhlarda ishlaydi (ovozli chatda ijro uchun).\n"
            "Shaxsiy chatda qo'shiq nomini yozing — men topib, mp3 yuboraman."
        )

    # ------------------------------------------------------------ ro'yxatdan o'tkazish
    def register(self) -> None:
        """Ovozli chat buyruqlarini botga ulaydi."""
        table = [
            (["play", "p", "ijro"], self.cmd_play),
            (["skip"], self.cmd_skip),
            (["pause"], self.cmd_pause),
            (["resume"], self.cmd_resume),
            (["stop"], self.cmd_stop),
            (["queue"], self.cmd_queue),
        ]
        all_commands: list[str] = []
        for names, func in table:
            all_commands += names
            self.bot.add_handler(
                MessageHandler(safe_message(func), filters.command(names) & filters.group)
            )
        self.bot.add_handler(
            MessageHandler(
                safe_message(self.cmd_private),
                filters.command(all_commands) & filters.private,
            )
        )


# =====================================================================================
# 6) SESSION_STRING YARATISH (python main.py session)
# =====================================================================================
async def generate_session() -> None:
    """
    Yordamchi akkaunt uchun SESSION_STRING yaratadi.
    Faqat bir marta, o'z kompyuteringizda ishga tushiring:
        python main.py session
    Telefon raqam, Telegram yuborgan kod (va bo'lsa, ikki bosqichli parol) so'raladi.
    """
    api_id = os.getenv("API_ID", "").strip() or input("API_ID: ").strip()
    api_hash = os.getenv("API_HASH", "").strip() or input("API_HASH: ").strip()

    print("\nYordamchi akkaunt telefon raqamini xalqaro formatda kiriting (masalan, +998901234567).")
    print("Telegram shu akkauntga kod yuboradi.\n")

    async with Client(
        "session_generator",
        api_id=int(api_id),
        api_hash=api_hash,
        in_memory=True,
    ) as app:
        session = await app.export_session_string()
        me = await app.get_me()

    print("\n" + "=" * 60)
    print(f"Akkaunt: {me.first_name} (id: {me.id})")
    print("SESSION_STRING (shuni .env yoki hosting sozlamalariga qo'ying):\n")
    print(session)
    print("\n" + "=" * 60)
    print("DIQQAT: bu qator akkauntga to'liq kirish beradi. Hech kimga ko'rsatmang!")
    print("Va uni bir vaqtda faqat bitta joyda (kompyuter YOKI hosting) ishlating.")


# =====================================================================================
# 7) ASOSIY ISHGA TUSHIRISH
# =====================================================================================
async def start_health_server(port: int) -> web.AppRunner:
    """GET / va GET /health ga "OK" qaytaradi. Hosting portni ochiq deb ko'rishi uchun kerak."""

    async def ok(_: web.Request) -> web.Response:
        return web.Response(text="OK")

    app = web.Application()
    app.router.add_get("/", ok)
    app.router.add_get("/health", ok)
    runner = web.AppRunner(app, access_log=None)
    await runner.setup()
    await web.TCPSite(runner, "0.0.0.0", port).start()
    log.info("Health endpoint ishga tushdi (port %s)", port)
    return runner


async def run_bot() -> None:
    # Avval port ochiladi: hosting servisni "tirik" deb tezroq tan oladi
    runner = await start_health_server(Config.PORT)

    # Handlerlar matnida <b>...</b> ishlatilgan, shuning uchun parse_mode=HTML
    bot = Client(
        "music_bot",
        api_id=Config.API_ID,
        api_hash=Config.API_HASH,
        bot_token=Config.BOT_TOKEN,
        in_memory=True,
        parse_mode=enums.ParseMode.HTML,
        workers=16,
    )
    assistant = Client(
        "assistant",
        api_id=Config.API_ID,
        api_hash=Config.API_HASH,
        session_string=Config.SESSION_STRING,
        in_memory=True,
        workers=4,
    )
    call = PyTgCalls(assistant)
    voice = VoiceManager(bot, assistant, call)

    register_handlers(bot)   # havola, qidiruv, Shazam
    voice.register()         # /play, /skip, /pause, /resume, /stop, /queue

    started: list[Client] = []
    try:
        await bot.start()
        started.append(bot)
        await assistant.start()
        started.append(assistant)
        await call.start()
        await voice.init()

        me = await bot.get_me()
        log.info("Bot tayyor: @%s", me.username)
        await idle()
    except Exception:
        log.exception(
            "Ishga tushishda xatolik. API_ID, API_HASH, BOT_TOKEN va SESSION_STRING "
            "to'g'riligini tekshiring (SESSION_STRING eskirgan bo'lsa "
            "`python main.py session` bilan yangisini oling)."
        )
        raise
    finally:
        for client in reversed(started):
            with contextlib.suppress(Exception):
                await client.stop()
        with contextlib.suppress(Exception):
            await runner.cleanup()


def main() -> None:
    mode = sys.argv[1].lower().lstrip("-") if len(sys.argv) > 1 else ""
    if mode in ("session", "gen_session", "gen-session"):
        with contextlib.suppress(KeyboardInterrupt):
            asyncio.run(generate_session())
        return

    try:
        Config.load()
    except RuntimeError as exc:
        log.error("%s", exc)
        sys.exit(1)
    with contextlib.suppress(KeyboardInterrupt):
        asyncio.run(run_bot())


if __name__ == "__main__":
    main()
