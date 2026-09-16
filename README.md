import os
import asyncio
import yt_dlp
from dotenv import load_dotenv
from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, MessageHandler, filters, ContextTypes

load_dotenv()

BOT_TOKEN = os.environ['BOT_TOKEN']
MAX_FILE_SIZE_MB = 50

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "👋 Привет! Отправь мне ссылку на видео (YouTube, TikTok, Instagram, Facebook), и я скачаю его.\n\n"
        "⚠️ Ограничение: файлы до 50 МБ."
    )

async def download_video(update: Update, context: ContextTypes.DEFAULT_TYPE):
    url = update.message.text
    if not url.startswith("http"):
        await update.message.reply_text("Пожалуйста, отправь ссылку на видео.")
        return

    msg = await update.message.reply_text("⏳ Скачиваю видео, подожди немного...")

    ydl_opts = {
        'outtmpl': 'downloads/%(id)s.%(ext)s',
        'format': f'best[filesize<{MAX_FILE_SIZE_MB}M]/best[height<=720]',
        'quiet': True,
        'no_warnings': True,
    }

    try:
        def download():
            with yt_dlp.YoutubeDL(ydl_opts) as ydl:
                info = ydl.extract_info(url, download=True)
                return ydl.prepare_filename(info), info.get('title', 'Видео')

        filename, title = await asyncio.to_thread(download)

        await msg.edit_text("📤 Отправляю файл...")
        
        with open(filename, 'rb') as video:
            await update.message.reply_video(video, caption=title[:1024])

        os.remove(filename)

    except Exception as e:
        await msg.edit_text(f"❌ Ошибка: {str(e)[:200]}")

if __name__ == '__main__':
    if not os.path.exists('downloads'):
        os.makedirs('downloads')
        
    app = ApplicationBuilder().token(BOT_TOKEN).build()
    app.add_handler(CommandHandler('start', start))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, download_video))
    print('Бот запущен и работает...')
    app.run_polling()
