# Файл bot.py
import os
import asyncio
import logging
from aiogram import Bot, Dispatcher, types, F
from aiogram.filters import CommandStart
from aiogram.types import FSInputFile
import yt_dlp

# Вставьте ваш токен или передайте его через переменную окружения на Hugging Face
BOT_TOKEN = os.getenv("BOT_TOKEN", "YOUR_TELEGRAM_BOT_TOKEN_HERE")

# Настройка логов
logging.basicConfig(level=logging.INFO)

bot = Bot(token=BOT_TOKEN)
dp = Dispatcher()

# Семафор: максимум 2 одновременных скачивания
DOWNLOAD_SEMAPHORE = asyncio.Semaphore(2)

# Лимит на размер файла Telegram (50 МБ в байтах)
MAX_FILE_SIZE = 50 * 1024 * 1024

@dp.message(CommandStart())
async def start_cmd(message: types.Message):
    await message.answer(
        "Привет! Отправь мне ссылку на видео с YouTube, TikTok или Instagram Reels, и я скачаю его для тебя."
    )

def download_video_sync(url: str, output_path: str) -> dict:
    """Синхронная функция скачивания через yt-dlp."""
    ydl_opts = {
        'format': 'b[filesize<=50M]/mp4/best[filesize<=50M]/best',
        'outtmpl': output_path,
        'quiet': True,
        'no_warnings': True,
        'max_filesize': MAX_FILE_SIZE,
    }
    with yt_dlp.YoutubeDL(ydl_opts) as ydl:
        return ydl.extract_info(url, download=True)

@dp.message(F.text.startswith("http://") | F.text.startswith("https://"))
async def handle_video_link(message: types.Message):
    url = message.text.strip()
    status_msg = await message.answer("⏳ Видео добавлено в очередь...")

    async with DOWNLOAD_SEMAPHORE:
        await status_msg.edit_text("📥 Скачиваю видео, подождите...")
        
        file_id = f"video_{message.from_user.id}_{message.message_id}.mp4"
        output_template = f"/tmp/{file_id}"

        try:
            # Запуск скачивания в отдельном потоке, чтобы не блокировать бота
            loop = asyncio.get_running_loop()
            await loop.run_in_executor(None, download_video_sync, url, output_template)

            # Проверка существования и размера файла
            if not os.path.exists(output_template):
                await status_msg.edit_text("❌ Не удалось скачать видео или формат не поддерживается.")
                return

            file_size = os.path.getsize(output_template)

            if file_size > MAX_FILE_SIZE:
                await status_msg.edit_text(
                    "⚠️ **Ошибка:** Размер скачанного видео превышает 50 МБ. "
                    "Telegram API не позволяет отправлять файлы такого размера через ботов."
                )
            else:
                await status_msg.edit_text("📤 Отправляю видео...")
                video_file = FSInputFile(output_template)
                await message.answer_video(video=video_file)
                await status_msg.delete()

        except yt_dlp.utils.FileTooLargeError:
            await status_msg.edit_text("⚠️ **Ошибка:** Файл превышает лимит 50 МБ и не был скачан.")
        except Exception as e:
            logging.error(f"Ошибка при обработке ссылки: {e}")
            await status_msg.edit_text("❌ Произошла ошибка при скачивании видео. Проверьте ссылку.")
        
        finally:
            # Очистка диска: удаляем файл в любом случае
            if os.path.exists(output_template):
                os.remove(output_template)

async def main():
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
    
# файл requirements.txt
aiogram==3.15.0
yt-dlp

# Файл Dockerfile
FROM python:3.11-slim

# Установка ffmpeg (необходим для сборки и конвертации видео в yt-dlp)
RUN apt-get update && apt-get install -y --no-install-recommends \
    ffmpeg \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Копируем зависимости и устанавливаем их
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Копируем остальной код
COPY . .

# Запуск бота
CMD ["python", "bot.py"]
