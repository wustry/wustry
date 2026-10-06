import asyncio
import os
import re
import tempfile
from pathlib import Path

from aiogram import Bot, Dispatcher, F
from aiogram.filters import CommandStart
from aiogram.types import FSInputFile, Message
from yt_dlp import YoutubeDL

# BotFather bergan tokenni shu yerga qo'ying yoki BOT_TOKEN o'zgaruvchisiga yozing
TOKEN = os.getenv("8242967640:AAEdHIp4KpGQkMkeN_p84ohF50zV4JL76U8")

# Telegram bot orqali maksimal fayl hajmi: 50 MB
MAX_SIZE = 50 * 1024 * 1024

URL_RE = re.compile(r"https?://(?:www\.)?(?:youtube\.com|youtu\.be|instagram\.com)/\S+")

bot = Bot(TOKEN)
dp = Dispatcher()


def download(url: str, out_dir: str) -> list[Path]:
    opts = {
        "outtmpl": f"{out_dir}/%(id)s.%(ext)s",
        # 50 MB dan oshmaydigan eng yaxshi sifat
        "format": f"best[filesize<{MAX_SIZE}]/best[filesize_approx<{MAX_SIZE}]/worst",
        "noplaylist": True,
        "quiet": True,
        "max_filesize": MAX_SIZE,
        # Instagram uchun kerak bo'lsa: "cookiefile": "cookies.txt",
    }
    with YoutubeDL(opts) as ydl:
        ydl.extract_info(url, download=True)
    return sorted(Path(out_dir).glob("*"))


@dp.message(CommandStart())
async def start(m: Message):
    await m.answer(
        "Salom! 👋\nYouTube yoki Instagram havolasini yuboring, "
        "men videoni yoki rasmni yuklab beraman."
    )


@dp.message(F.text.regexp(URL_RE))
async def handle_link(m: Message):
    url = URL_RE.search(m.text).group(0)
    status = await m.answer("⏳ Yuklanmoqda...")
    try:
        with tempfile.TemporaryDirectory() as tmp:
            files = await asyncio.to_thread(download, url, tmp)
            if not files:
                await status.edit_text("❌ Hech narsa topilmadi.")
                return
            for f in files:
                if f.stat().st_size > MAX_SIZE:
                    await m.answer("❌ Fayl 50 MB dan katta, yubora olmayman.")
                    continue
                media = FSInputFile(f)
                ext = f.suffix.lower()
                if ext in {".jpg", ".jpeg", ".png", ".webp"}:
                    await m.answer_photo(media)
                else:
                    await m.answer_video(media)
        await status.delete()
    except Exception as e:
        await status.edit_text(f"❌ Xatolik: {str(e)[:200]}")


@dp.message()
async def other(m: Message):
    await m.answer("Iltimos, YouTube yoki Instagram havolasini yuboring.")


async def main():
    await dp.start_polling(bot)


if __name__ == "__main__":
    asyncio.run(main())

