import telebot
import yt_dlp
import re
import os

# اطلاعات شما:
BOT_TOKEN = "8151666014:AAEsM99c95wERxY60eOflrMPwX2c88ZHp_I"
ADMIN_ID = 6018651265
CHANNEL_USERNAME = "@llove_musicam"

bot = telebot.TeleBot(BOT_TOKEN)

def is_user_joined(chat_id):
    try:
        member = bot.get_chat_member(CHANNEL_USERNAME, chat_id)
        return member.status in ['member', 'creator', 'administrator']
    except:
        return False

@bot.message_handler(commands=['start'])
def start(message):
    if not is_user_joined(message.chat.id):
        bot.send_message(message.chat.id, f"برای استفاده از ربات باید ابتدا در کانال {CHANNEL_USERNAME} عضو شوید.")
        return
    bot.send_message(message.chat.id, "سَلاو! لینک ویدیو رو بفرست تا برات بفرستمش!")

@bot.message_handler(commands=['admin'])
def admin_panel(message):
    if message.from_user.id == ADMIN_ID:
        bot.send_message(message.chat.id, "پنل مدیریت فعال شد.\n(دستور خاصی خواستی اضافه کنم بگو)")
    else:
        bot.send_message(message.chat.id, "شما اجازه دسترسی به پنل رو ندارید.")

@bot.message_handler(func=lambda msg: True)
def download_video(message):
    if not is_user_joined(message.chat.id):
        bot.send_message(message.chat.id, f"لطفاً اول عضو کانال {CHANNEL_USERNAME} بشی.")
        return

    url = message.text.strip()
    if not re.match(r'https?://', url):
        bot.send_message(message.chat.id, "این لینک معتبر نیست! لطفاً یه لینک ویدیو بفرست.")
        return

    msg = bot.reply_to(message, "در حال بررسی لینک...")

    ydl_opts = {
        'quiet': True,
        'skip_download': True,
        'forcejson': True
    }

    try:
        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            info = ydl.extract_info(url, download=False)
            formats = info.get('formats', [info])
            text = "کیفیت مدنظر رو انتخاب کن:\n"
            buttons = []

            for f in formats:
                if f.get('ext') == 'mp4' and f.get('filesize') and f.get('format_note'):
                    size = round(f['filesize'] / 1024 / 1024, 2)
                    text += f"- {f['format_note']} ({size}MB)\n"
                    buttons.append(telebot.types.InlineKeyboardButton(
                        text=f"{f['format_note']} - {size}MB",
                        callback_data=f['url']
                    ))

            markup = telebot.types.InlineKeyboardMarkup()
            for btn in buttons[:10]:
                markup.add(btn)

            bot.edit_message_text(chat_id=msg.chat.id, message_id=msg.message_id, text=text, reply_markup=markup)

    except Exception as e:
        bot.edit_message_text(chat_id=msg.chat.id, message_id=msg.message_id, text=f"دانلود موفق نبود.\nخطا: {e}")

@bot.callback_query_handler(func=lambda call: True)
def callback_handler(call):
    url = call.data
    bot.send_message(call.message.chat.id, "در حال دانلود فایل...")

    try:
        ydl_opts = {
            'outtmpl': 'video.%(ext)s',
            'format': 'bestvideo+bestaudio/best',
            'quiet': True
        }

        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            info = ydl.extract_info(url, download=True)
            filename = ydl.prepare_filename(info)

        with open(filename, 'rb') as f:
            bot.send_video(call.message.chat.id, f)

        os.remove(filename)

    except Exception as e:
        bot.send_message(call.message.chat.id, f"مشکلی در ارسال ویدیو پیش اومد:\n{e}")

print("ربات با موفقیت راه‌اندازی شد.")
bot.infinity_polling()
