<div align="center">

# attapy

**کتابخانهٔ async و بدون وابستگی پایتون برای Bot API سروش‌پلاس**
**Async, zero-dependency Python library for the Soroush Plus Bot API**

`pip install attapy`

</div>

---

## فهرست

1. [معرفی](#معرفی)
2. [نصب](#نصب)
3. [شروع سریع](#شروع-سریع)
4. [آدرس سرور (base_url)](#آدرس-سرور-base_url)
5. [مفاهیم اصلی](#مفاهیم-اصلی)
6. [هندلرها و اولویت](#هندلرها-و-اولویت)
7. [فیلترها](#فیلترها)
8. [آبجکت‌ها (Message, Chat, User, CallbackQuery)](#آبجکتها)
9. [دکمه‌ها و صفحه‌کلید](#دکمهها-و-صفحهکلید)
10. [ارسال فایل و رسانه](#ارسال-فایل-و-رسانه)
11. [مرجع کامل متدهای Robot](#مرجع-کامل-متدهای-robot)
12. [خطاها](#خطاها)
13. [اجرا، توقف و polling](#اجرا-توقف-و-polling)
14. [دستورهای آماده (Recipes)](#دستورهای-آماده-recipes)
15. [استقرار روی سرور](#استقرار-روی-سرور)
16. [🤖 راهنمای ویژهٔ هوش مصنوعی (AI Guide)](#-راهنمای-ویژهٔ-هوش-مصنوعی-ai-guide)
17. [عیب‌یابی](#عیبیابی)
18. [وضعیت پروژه و لایسنس](#وضعیت-پروژه-و-لایسنس)

---

## معرفی

`attapy` یک کتابخانهٔ **asyncio** برای ساخت ربات‌های **رسمی** سروش‌پلاس (با توکن ربات و Bot API) است.

| ویژگی | توضیح |
|---|---|
| 🪶 بدون وابستگی | فقط کتابخانهٔ استاندارد پایتون. نصب روی Termux، cPanel و هر جای دیگر بدون کامپایل و بدون دردسر |
| ⚡ async | همهٔ متدهای شبکه `async` هستند و هندلرها می‌توانند `async def` باشند |
| 🧩 هندلر دکوراتوری | `@bot.on_message(...)` و `@bot.on_callback_query(...)` |
| 🔎 فیلترهای قابل ترکیب | `filters.private & filters.command("start")` و `|` و `~` |
| ⌨️ ساخت دکمه | `Button.inline`، `Button.keyboard` و سازندهٔ زنجیره‌ای |
| 📎 ارسال فایل | از file_id، URL، مسیر فایل محلی، `bytes` یا file object |
| 🛡️ خطاهای مشخص | `ChatNotFound`، `FloodWait`، `Unauthorized`، ... و ثبت هندلر خطا |
| 🔁 polling پایدار | تلاش دوباره روی خطای شبکه، رعایت `retry_after` |

> این کتابخانه برای **ربات رسمی با توکن** است، نه برای ورود با شمارهٔ تلفن (self-bot).

---

## نصب

```bash
pip install attapy
```

پیش‌نیاز: Python 3.9 یا بالاتر. وابستگی دیگری نصب نمی‌شود.

نصب نسخهٔ توسعه از سورس:

```bash
git clone <آدرس-مخزن-شما>
cd attapy
pip install -e .
```

---

## شروع سریع

```python
from attapy import Robot

bot = Robot("TOKEN_RABAT")          # توکنی که از بات‌ساز سروش‌پلاس گرفته‌اید


@bot.on_message(commands=["start"])
async def start(message):
    await message.reply("سلام 👋")


@bot.on_message()                    # همهٔ پیام‌های دیگر
async def echo(message):
    if message.text:
        await message.reply(message.text)


bot.run()                            # تا Ctrl+C اجرا می‌شود
```

توکن را **هرگز** داخل کد یا مخزن عمومی نگذارید؛ از متغیر محیطی استفاده کنید:

```python
import os
bot = Robot(os.environ["BOT_TOKEN"])
```

---

## آدرس سرور (base_url)

طبق نمونهٔ مستندات سروش‌پلاس، هر متد با این الگو فراخوانی می‌شود:

```
https://api.splus.ir/bot<TOKEN>/<METHOD_NAME>        مثال: https://api.splus.ir/botABC123/sendMessage
```

`attapy` همین الگو را به‌صورت پیش‌فرض استفاده می‌کند، پس معمولاً **نیازی به تنظیم چیزی نیست.** برای دانلود فایل از الگوی رایج `https://api.splus.ir/file/bot<TOKEN>/<file_path>` استفاده می‌شود.

اگر سروش‌پلاس آدرس را تغییر داد، یا برای تست روی سرور دیگر:

```python
bot = Robot(token, base_url="https://ADDRESS-JADID")
```

```bash
export ATTAPY_BASE_URL="https://ADDRESS-JADID"
```

اگر هنگام اجرا خطای `NetworkError` دیدید، اول اینترنت/فیلتر و بعد همین آدرس را بررسی کنید.

---

## مفاهیم اصلی

```
سرور سروش‌پلاس ──getUpdates──▶ Robot.poll() ──▶ process_update(raw)
                                                  │
                          ┌───────────────────────┴──────────────────────┐
                    key = "message"                              key = "callback_query"
                          │                                              │
                Message(...) ← dispatch                         Callback(...) ← dispatch
                          │                                              │
        اولین هندلر منطبق (بر اساس priority)          اولین هندلر منطبق (بر اساس data/filter)
```

نکته‌های اصلی:

- **اولین هندلر منطبق اجرا می‌شود و بقیه نادیده گرفته می‌شوند.** ترتیب: اول `priority` (عدد کمتر = زودتر)، بعد ترتیب ثبت.
- خطای داخل یک هندلر ربات را **نمی‌اندازد**؛ اول به `bot.errors` می‌رسد، اگر هندلری نبود در لاگ ثبت می‌شود.
- آبجکت‌ها (Message و ...) هر فیلد ناشناختهٔ سرور را هم با نقطه می‌دهند؛ فیلدی که وجود ندارد `None` برمی‌گرداند (خطا نمی‌دهد).
- پارامترهایی که `None` باشند **ارسال نمی‌شوند**.

---

## هندلرها و اولویت

### `on_message`

```python
@bot.on_message(filter=None, commands=None, priority=HandlerPriority.NORMAL)
async def handler(message): ...
```

| پارامتر | توضیح |
|---|---|
| `commands` | لیست یا رشته؛ مثل `["start", "help"]`. `/start@botname` هم تشخیص داده می‌شود. متن بعد از دستور در `message.command_args` است |
| `filter` | یک `Filter` از `attapy.filters` (یا ترکیب آن‌ها) |
| `priority` | `HandlerPriority.HIGHEST(0) / HIGH(25) / NORMAL(50) / LOW(75) / LOWEST(100)` یا هر عدد |

اگر هم `commands` و هم `filter` بدهید، **هر دو** باید برقرار باشند.

```python
from attapy import HandlerPriority

@bot.on_message(commands=["ban"], priority=HandlerPriority.HIGH)
async def ban(message):
    user = message.command_args           # متن بعد از /ban
    await message.reply(f"کاربر {user} مسدود شد")
```

### `on_callback_query`

```python
@bot.on_callback_query(data=None, filter=None, priority=HandlerPriority.NORMAL)
async def handler(callback): ...
```

`data` می‌تواند باشد:

| مقدار | رفتار |
|---|---|
| `None` | همهٔ callbackها |
| `"vote:yes"` | فقط برابر دقیق |
| `"vote:*"` | هر چیزی که با `vote:` شروع شود |
| `["a", "b"]` | هرکدام از لیست |
| `callable` | `lambda data: data.startswith("x")` |

### هندلر همگام (sync)

هندلر می‌تواند معمولی (`def`) هم باشد، ولی چون `message.reply` یک coroutine است، تقریباً همیشه `async def` لازم دارید.

---

## فیلترها

```python
from attapy import filters
```

| فیلتر | توضیح |
|---|---|
| `filters.command("start")` / `command(["a","b"], prefixes="/")` | متن با آن دستور شروع شود |
| `filters.regex(pattern, flags=0)` | `re.search` روی `text` یا `caption`؛ نتیجه در `message.match` |
| `filters.user(123)` / `user([1,2])` | فرستنده در لیست باشد |
| `filters.chat(-100123)` / `chat([..])` | چت در لیست باشد |
| `filters.private` | چت خصوصی |
| `filters.group` / `filters.supergroup` / `filters.channel` | نوع چت |
| `filters.text` | پیام متن دارد |
| `filters.create(func, name=None)` | از یک تابع ساده فیلتر بسازید |

### ترکیب

```python
admins = filters.user([111, 222])

@bot.on_message(filter=filters.private & admins & filters.command("stats"))
async def stats(m): ...

@bot.on_message(filter=filters.group | filters.supergroup)
async def in_group(m): ...

@bot.on_message(filter=~filters.private)       # هر چیزی به جز چت خصوصی
async def not_private(m): ...
```

### فیلتر دلخواه

```python
has_photo = filters.create(lambda m: bool(m.photo), "has_photo")

@bot.on_message(filter=has_photo)
async def got_photo(m):
    await m.reply("عکس گرفتم 📷")
```

> تابع فیلتر باید **همگام** (sync) و سریع باشد و `True/False` برگرداند.

---

## آبجکت‌ها

### `Message`

| ویژگی | توضیح |
|---|---|
| `message.id` / `message.message_id` | شناسهٔ پیام |
| `message.chat` | آبجکت `Chat` |
| `message.chat_id` | میان‌بر برای `chat.id` |
| `message.from_user` | آبجکت `User` (ممکن است `None` باشد) |
| `message.text` / `message.caption` | متن / توضیح رسانه |
| `message.reply_to_message` | پیامی که به آن ریپلای شده (`Message` یا `None`) |
| `message.command_args` | متن بعد از دستور (فقط وقتی با `commands=` هندلر ثبت شده) |
| `message.match` | نتیجهٔ `regex` (فقط بعد از فیلتر regex) |
| `message.photo`, `message.document`, ... | داده‌ی خام همان فیلد (dict/list) یا `None` |
| `message.to_dict()` | دیکشنری خام کامل |

متدها (همه `async`):

```python
await m.reply(text, **kwargs)                      # به صورت ریپلای به همین پیام
await m.reply_photo(photo, caption=None, **kw)
await m.reply_video / reply_audio / reply_voice / reply_document / reply_animation(...)
await m.reply_sticker(sticker) ; await m.reply_video_note(video_note)
await m.reply_contact(phone_number, first_name, last_name=None)
await m.reply_location(latitude, longitude)
await m.reply_dice(emoji=None)
await m.edit_text(text, **kw) ; await m.edit_caption(caption) ; await m.edit_reply_markup(markup)
await m.delete()
await m.forward(to_chat_id)
await m.pin() ; await m.unpin()
```

### `Chat`

`id`, `type` (`private` / `group` / `supergroup` / `channel`), `title`, `username`، و propertyهای `is_private`، `is_group`.

### `User`

`id`, `is_bot`, `first_name`, `last_name`, `username`, `language_code`، و propertyهای `full_name` و `mention`.

### `CallbackQuery` / `Callback`

| | |
|---|---|
| `cb.id` | شناسهٔ callback |
| `cb.data` | همان `callback_data` دکمه |
| `cb.from_user` | کاربری که دکمه را زد |
| `cb.message` | پیامی که دکمه روی آن بود (`Message`) |
| `await cb.answer(text=None, show_alert=False, url=None, cache_time=None)` | پاسخ به کلیک (برای قطع شدن حالت loading دکمه **همیشه** صدا بزنید) |

---

## دکمه‌ها و صفحه‌کلید

```python
from attapy import Button
```

| متد | خروجی |
|---|---|
| `Button.inline(text, callback_data=None, url=None)` | دکمهٔ شیشه‌ای |
| `Button.keyboard(text, request_contact=False, request_location=False)` | دکمهٔ صفحه‌کلید معمولی |
| `Button.row(*buttons)` | چند دکمه در یک ردیف |
| `Button.inline_markup(rows)` | صفحه‌کلید شیشه‌ای کامل |
| `Button.keyboard_markup(rows, resize_keyboard=True, one_time_keyboard=False, input_field_placeholder=None, selective=None)` | صفحه‌کلید معمولی کامل |
| `Button.remove_keyboard(selective=None)` | حذف صفحه‌کلید معمولی |
| `Button.force_reply(input_field_placeholder=None, selective=None)` | وادار کردن کاربر به ریپلای |
| `Button.builder(inline=False)` | سازندهٔ زنجیره‌ای |

### سازندهٔ زنجیره‌ای

```python
kb = (
    Button.builder(inline=True)
    .add(Button.inline("بله", "yes"), Button.inline("خیر", "no"))   # ردیف ۱
    .new_row()
    .add(Button.inline("سایت", url="https://splus.ir"))             # ردیف ۲
    .build()
)
await message.reply("ادامه بدهیم؟", reply_markup=kb)
```

`reply_markup` هم دیکشنری می‌پذیرد و هم خود builder (بدون `.build()`).

### بدون builder

```python
kb = Button.inline_markup([
    [Button.inline("A", "a"), Button.inline("B", "b")],
    [Button.inline("C", "c")],
])
```

### دکمهٔ درخواست شماره/موقعیت

```python
kb = Button.keyboard_markup([
    [Button.keyboard("📞 ارسال شماره", request_contact=True)],
    [Button.keyboard("📍 ارسال موقعیت", request_location=True)],
], one_time_keyboard=True)
```

---

## ارسال فایل و رسانه

همهٔ متدهای `send_photo / send_video / send_audio / send_voice / send_document / send_animation / send_sticker / send_video_note` یک آرگومان رسانه می‌گیرند که می‌تواند باشد:

| نوع مقدار | رفتار |
|---|---|
| `"AgACAgQ..."` (file_id) | بدون آپلود، مستقیم ارسال می‌شود |
| `"https://..."` | سرور خودش دانلود می‌کند |
| `"photo.jpg"` (مسیر فایل موجود) | آپلود multipart |
| `b"..."` (bytes) | آپلود multipart |
| file object (`open(..., "rb")`) | آپلود multipart |

```python
await bot.send_photo(chat_id, "cat.jpg", caption="گربه")
await bot.send_document(chat_id, open("report.pdf", "rb"))
await bot.send_voice(chat_id, "https://example.com/a.ogg")
```

دانلود فایل ورودی:

```python
info = await bot.get_file(message.document["file_id"])   # {"file_id", "file_path", ...}
await bot.download_file(info, "saved.bin")               # info یا خود file_path
```

> `send_media_group` فقط file_id یا URL می‌پذیرد (آپلود فایل محلی در آن پشتیبانی نمی‌شود).

---

## مرجع کامل متدهای Robot

### ساخت

```python
Robot(token, base_url=None, parse_mode=None, timeout=60)
```

- `parse_mode`: مقدار پیش‌فرض برای همهٔ پیام‌ها و کپشن‌ها (مثلاً `"HTML"` یا `"MarkdownV2"`). اگر در یک فراخوانی `parse_mode=` بدهید، همان استفاده می‌شود.
- `timeout`: timeout پیش‌فرض درخواست‌ها (ثانیه).

### ارسال و دریافت

| متد | endpoint | توضیح |
|---|---|---|
| `get_me()` | `getMe` | اطلاعات خود ربات |
| `send_message(chat_id, text, parse_mode=None, entities=None, reply_markup=None, disable_web_page_preview=None, reply_to_message_id=None, allow_sending_without_reply=None, **kw)` | `sendMessage` | پیام متنی |
| `forward_message(chat_id, from_chat_id, message_id)` | `forwardMessage` | فوروارد |
| `copy_message(chat_id, from_chat_id, message_id, caption=None, parse_mode=None, caption_entities=None, reply_markup=None)` | `copyMessage` | کپی بدون برچسب فوروارد |
| `send_photo(chat_id, photo, caption=None, parse_mode=None, caption_entities=None, reply_markup=None)` | `sendPhoto` | عکس |
| `send_audio(chat_id, audio, caption=None, parse_mode=None, duration=None, performer=None, title=None, reply_markup=None)` | `sendAudio` | موزیک |
| `send_document(chat_id, document, caption=None, parse_mode=None, reply_markup=None)` | `sendDocument` | فایل |
| `send_video(chat_id, video, caption=None, parse_mode=None, duration=None, width=None, height=None, supports_streaming=None, reply_markup=None)` | `sendVideo` | ویدیو |
| `send_animation(chat_id, animation, caption=None, parse_mode=None, duration=None, width=None, height=None, reply_markup=None)` | `sendAnimation` | گیف |
| `send_voice(chat_id, voice, caption=None, parse_mode=None, duration=None, reply_markup=None)` | `sendVoice` | پیام صوتی |
| `send_video_note(chat_id, video_note, duration=None, length=None, reply_markup=None)` | `sendVideoNote` | ویدیو مسیج |
| `send_sticker(chat_id, sticker, reply_markup=None)` | `sendSticker` | استیکر |
| `send_media_group(chat_id, media)` | `sendMediaGroup` | آلبوم |
| `send_location(chat_id, latitude, longitude, horizontal_accuracy=None, reply_markup=None)` | `sendLocation` | موقعیت |
| `send_contact(chat_id, phone_number, first_name, last_name=None, vcard=None, reply_markup=None)` | `sendContact` | مخاطب |
| `send_dice(chat_id, emoji=None, reply_markup=None)` | `sendDice` | تاس |
| `send_chat_action(chat_id, action)` | `sendChatAction` | «در حال تایپ...» و مشابه |
| `get_updates(offset=None, limit=None, timeout=None, allowed_updates=None)` | `getUpdates` | دریافت دستی آپدیت |

### ویرایش و مدیریت

| متد | endpoint |
|---|---|
| `edit_message_text(text, chat_id=None, message_id=None, inline_message_id=None, parse_mode=None, entities=None, reply_markup=None, disable_web_page_preview=None)` | `editMessageText` |
| `edit_message_caption(chat_id=None, message_id=None, inline_message_id=None, caption=None, parse_mode=None, caption_entities=None, reply_markup=None)` | `editMessageCaption` |
| `edit_message_media(media, chat_id=None, message_id=None, inline_message_id=None, reply_markup=None)` | `editMessageMedia` |
| `edit_message_reply_markup(chat_id=None, message_id=None, inline_message_id=None, reply_markup=None)` | `editMessageReplyMarkup` |
| `delete_message(chat_id, message_id)` | `deleteMessage` |
| `answer_callback_query(callback_query_id, text=None, show_alert=False, url=None, cache_time=None)` | `answerCallbackQuery` |
| `pin_chat_message(chat_id, message_id)` / `unpin_chat_message(chat_id, message_id=None)` | `pinChatMessage` / `unpinChatMessage` |

### اطلاعات و فایل

| متد | endpoint |
|---|---|
| `get_chat(chat_id)` | `getChat` |
| `get_user_profile_photos(user_id, offset=None, limit=None)` | `getUserProfilePhotos` |
| `get_sticker_set(name)` | `getStickerSet` |
| `get_file(file_id)` | `getFile` |
| `download_file(file_path, destination)` | دانلود از `/file/bot<token>/<path>` |
| `file_url(file_path)` | فقط آدرس دانلود را می‌سازد (غیر async) |

### دستورهای ربات و webhook

| متد | endpoint |
|---|---|
| `set_my_commands(commands, scope=None, language_code=None)` | `setMyCommands` — `commands` لیست دیکشنری `{"command","description"}` یا لیست تاپل `("start","شروع")` |
| `get_my_commands(scope=None, language_code=None)` | `getMyCommands` |
| `delete_my_commands(scope=None, language_code=None)` | `deleteMyCommands` |
| `set_webhook(url, certificate=None, ip_address=None, max_connections=None, allowed_updates=None, drop_pending_updates=None)` | `setWebhook` |
| `delete_webhook(drop_pending_updates=None)` | `deleteWebhook` |
| `get_webhook_info()` | `getWebhookInfo` |
| `log_out()` | `logOut` |
| `close_bot()` | متد Bot API با نام `close` |

### چرخهٔ اجرا

| متد | توضیح |
|---|---|
| `run(interval=1.0, timeout=30)` | **همگام**؛ polling را تا Ctrl+C اجرا می‌کند |
| `await poll(interval=1.0, timeout=30, limit=100)` | حلقهٔ polling (داخل event loop خودتان) |
| `await start_polling(interval=1.0, timeout=30)` | polling را در یک task پس‌زمینه شروع و task را برمی‌گرداند |
| `stop()` | حلقهٔ polling را متوقف می‌کند |
| `await close()` | polling را می‌بندد |
| `await process_update(raw_dict)` | یک آپدیت خام را به هندلرها می‌دهد (برای webhook یا تست) |
| `async with Robot(token) as bot:` | start/close خودکار |

> تمام متدهای بالا `async` هستند مگر `run`, `stop`, `file_url`, `on_message`, `on_callback_query`.

---

## خطاها

```python
from attapy.errors import *
```

```
AttapyError                      ← پایهٔ همه (نام قدیمی PysrooshError هم کار می‌کند)
├── NetworkError                 اینترنت/DNS/timeout/آدرس base_url اشتباه
├── ValidationError              ورودی نامعتبر
├── SessionError → SessionExpiredError
├── UnsupportedByBackendError
└── SoroushPlusAPIError          خطای برگشتی از سرور (method, description, error_code, parameters)
    └── RPCError
        ├── BadRequest           (400)
        │   ├── ChatNotFound
        │   ├── MessageNotFound
        │   └── UserNotFound
        ├── AuthError
        │   └── Unauthorized     (401) توکن اشتباه
        └── FloodWait            (429) → e.retry_after (ثانیه)
```

### گرفتن خطا در فراخوانی

```python
from attapy.errors import ChatNotFound, FloodWait
import asyncio

try:
    await bot.send_message(chat_id, "سلام")
except ChatNotFound:
    print("چت پیدا نشد")
except FloodWait as e:
    await asyncio.sleep(e.retry_after)
```

### هندلر عمومی خطا

```python
@bot.errors.catch(ValueError)               # فقط ValueError
async def on_value_error(error, message):
    await message.reply("ورودی نامعتبر")

@bot.errors.catch()                         # همهٔ خطاها
async def on_any(error, *args):
    print("خطا:", error)
```

اولین هندلر منطبق اجرا می‌شود. `bot.errors.add(func, *exceptions)` همان کار را بدون دکوراتور انجام می‌دهد.

### رفتار polling در خطا

| خطا | رفتار |
|---|---|
| `FloodWait` | `retry_after` ثانیه صبر و ادامه |
| `NetworkError` و سایر خطاهای API | لاگ هشدار، ۳ ثانیه صبر و تلاش دوباره |
| `Unauthorized` | polling متوقف می‌شود و خطا بالا می‌آید (توکن اشتباه) |

---

## اجرا، توقف و polling

### ساده

```python
bot.run()
```

### داخل برنامهٔ async دیگر

```python
import asyncio
from attapy import Robot

async def main():
    async with Robot(TOKEN) as bot:
        @bot.on_message(commands=["start"])
        async def s(m): await m.reply("hi")

        task = await bot.start_polling()
        await asyncio.sleep(3600)       # کار دیگر شما
        bot.stop()
        await task

asyncio.run(main())
```

### لاگ

```python
import logging
logging.basicConfig(level=logging.INFO)     # logger با نام "attapy"
```

---

## دستورهای آماده (Recipes)

### ۱) فقط ادمین‌ها

```python
ADMINS = filters.user([111111, 222222])

@bot.on_message(filter=ADMINS & filters.command("panel"))
async def panel(m): await m.reply("پنل مدیریت")
```

### ۲) حالت‌های چندمرحله‌ای ساده (FSM سبک)

```python
state = {}        # user_id -> مرحله

@bot.on_message(commands=["register"])
async def reg(m):
    state[m.from_user.id] = "name"
    await m.reply("اسمت؟")

@bot.on_message(filter=filters.create(lambda m: m.from_user and state.get(m.from_user.id) == "name"))
async def got_name(m):
    state.pop(m.from_user.id)
    await m.reply(f"سلام {m.text}!")
```

> هندلر دوم باید **بعد از** دستورها ثبت شود یا اولویت بالاتر بگیرد.

### ۳) ارسال پیام همگانی با رعایت محدودیت

```python
import asyncio
from attapy.errors import FloodWait, SoroushPlusAPIError

async def broadcast(bot, chat_ids, text):
    for cid in chat_ids:
        try:
            await bot.send_message(cid, text)
        except FloodWait as e:
            await asyncio.sleep(e.retry_after)
            await bot.send_message(cid, text)
        except SoroushPlusAPIError:
            pass                      # کاربری که ربات را بلاک کرده
        await asyncio.sleep(0.05)
```

### ۴) دستور با آرگومان

```python
@bot.on_message(commands=["echo"])
async def echo(m):
    if not m.command_args:
        return await m.reply("مثال: /echo سلام")
    await m.reply(m.command_args)
```

### ۵) ویرایش پیام بعد از کلیک

```python
@bot.on_callback_query(data="like")
async def like(cb):
    await cb.answer("❤️")
    await cb.message.edit_text("ممنون از نظرتان!", reply_markup=None)
```

### ۶) رجیستر کردن دستورها در منو

```python
async def main():
    await bot.set_my_commands([("start", "شروع"), ("help", "راهنما")])
```

### ۷) ثبت وب‌هوک و پردازش دستی

```python
await bot.set_webhook("https://example.com/hook")
# در وب‌سرور خودتان بعد از دریافت JSON:
await bot.process_update(update_dict)
```

---

## استقرار روی سرور

ربات با polling باید **همیشه روشن** باشد.

| محیط | روش |
|---|---|
| Termux (موبایل) | `pkg install python && pip install attapy && python bot.py` (با `termux-wake-lock` تا خاموش نشود) |
| سرور لینوکس/VPS | `systemd` یا `screen`/`tmux` |
| هاست اشتراکی cPanel | معمولاً پروسهٔ همیشه‌روشن اجازه ندارد؛ در صورت پشتیبانی هاست «Setup Python App» یا از وب‌هوک استفاده کنید |

نمونهٔ `systemd`:

```ini
[Unit]
Description=attapy bot
After=network-online.target

[Service]
WorkingDirectory=/home/bot
Environment=BOT_TOKEN=xxxxx
ExecStart=/usr/bin/python3 /home/bot/bot.py
Restart=always

[Install]
WantedBy=multi-user.target
```

---

## 🤖 راهنمای ویژهٔ هوش مصنوعی (AI Guide)

این بخش برای **مدل‌های هوش مصنوعی** (ChatGPT, Claude, Gemini, Copilot, ...) و برای کسانی است که با آن‌ها کد می‌نویسند. اگر از هوش مصنوعی می‌خواهید با `attapy` ربات بنویسد، **این بخش را همراه درخواست به آن بدهید.**

### قوانین طلایی (برای AI)

1. فقط از `from attapy import Robot, Button, filters, HandlerPriority` و `from attapy.errors import ...` استفاده کن. **هیچ چیز دیگری از `attapy` وجود ندارد.** نام‌های ساختگی مثل `Dispatcher`، `Bot(...)`، `Router`، `executor`، `types.Message` از aiogram در این کتابخانه **نیستند**.
2. نقطهٔ ورود همیشه `Robot(token)` و در انتها `bot.run()` است. کلاس `Bot`/`Client` وجود ندارد.
3. هندلرها **دکوراتور** هستند: `@bot.on_message(...)` و `@bot.on_callback_query(...)`. امضای هندلر فقط **یک** آرگومان دارد: `(message)` یا `(callback)`. **نه** `(client, message)`.
4. همهٔ متدهای شبکه `async` هستند؛ حتماً `await` کن. هندلرها `async def` باشند.
5. پاسخ سریع: `await message.reply("...")`. پاسخ به کلیک دکمه: `await cb.answer()`.
6. اولین هندلر منطبق اجرا می‌شود؛ هندلر عمومی `@bot.on_message()` را **آخر** ثبت کن.
7. توکن را از `os.environ["BOT_TOKEN"]` بخوان؛ هرگز داخل کد ننویس.
8. برای فایل: `send_*` مسیر محلی، bytes، file object، file_id یا URL می‌پذیرد.
9. `reply_markup` را با `Button.builder(inline=True)...build()` یا `Button.inline_markup([...])` بساز.
10. تابع فیلتر دلخواه **sync** است: `filters.create(lambda m: ...)`.
11. وابستگی اضافه (aiohttp, requests, pydantic) نصب نکن؛ لازم نیست.
12. این کتابخانه برای **ربات رسمی با توکن** است؛ ورود با شماره تلفن و session ندارد.

### قالب استاندارد (کپی کن و تغییر بده)

```python
import os
from attapy import Robot, Button, filters, HandlerPriority
from attapy.errors import FloodWait, SoroushPlusAPIError

bot = Robot(os.environ["BOT_TOKEN"])


@bot.on_message(commands=["start"])
async def start(message):
    kb = (
        Button.builder(inline=True)
        .add(Button.inline("شروع", "go"))
        .build()
    )
    await message.reply("سلام!", reply_markup=kb)


@bot.on_callback_query(data="go")
async def go(cb):
    await cb.answer("انجام شد")
    await cb.message.edit_text("شروع شد ✅")


@bot.errors.catch()
async def on_error(error, *args):
    print("خطا:", repr(error))


if __name__ == "__main__":
    bot.run()
```

### اشتباهات رایج (و درستشان)

| ❌ اشتباه | ✅ درست |
|---|---|
| `bot = Bot(token); dp = Dispatcher()` | `bot = Robot(token)` |
| `@dp.message(Command("start"))` | `@bot.on_message(commands=["start"])` |
| `async def h(client, message)` | `async def h(message)` |
| `message.reply("x")` بدون await | `await message.reply("x")` |
| `executor.start_polling(...)` | `bot.run()` |
| `message.from_user.first_name` وقتی ممکن است `None` باشد | `message.from_user.full_name if message.from_user else "کاربر"` |
| `message.text.startswith(...)` بدون بررسی | `(message.text or "").startswith(...)` |
| `callback_query.answer()` را فراموش کردن | همیشه `await cb.answer()` |
| ثبت `@bot.on_message()` عمومی قبل از دستورها | بعد از دستورها ثبت کن یا به دستورها `priority=HandlerPriority.HIGH` بده |
| `Button.inline("x")` بدون `callback_data` و `url` | حتماً یکی از `callback_data` یا `url` |

### پرامپت آمادهٔ سیستم (System Prompt) برای AI

این متن را در ابتدای گفتگو با هوش مصنوعی بچسبانید:

```text
You are an expert in the Python library "attapy" (pip install attapy), an async,
zero-dependency client for the Soroush Plus official Bot API. Rules:
- Entry point: from attapy import Robot, Button, filters, HandlerPriority
  bot = Robot(os.environ["BOT_TOKEN"]); ...; bot.run()
- Handlers are decorators with ONE argument:
  @bot.on_message(filter=None, commands=None, priority=HandlerPriority.NORMAL) async def h(message)
  @bot.on_callback_query(data=None, filter=None, priority=...) async def h(cb)
- First matching handler wins; register catch-all handlers last.
- Message: id, chat(.id,.type), from_user(.id,.full_name,.username), text, caption,
  reply_to_message, command_args, to_dict(); await message.reply/reply_photo/edit_text/delete/forward/pin.
- CallbackQuery: data, from_user, message; await cb.answer(text=None, show_alert=False).
- Filters: filters.command, regex, user, chat, private, group, supergroup, channel, text, create(func);
  combine with & | ~.
- Buttons: Button.inline(text, callback_data=None, url=None), Button.keyboard(...),
  Button.builder(inline=True).add(...).new_row().build(), Button.inline_markup(rows), Button.keyboard_markup(rows).
- Robot methods (all async, snake_case of the Bot API): get_me, send_message, send_photo, send_document,
  send_video, send_audio, send_voice, send_animation, send_sticker, send_video_note, send_media_group,
  send_location, send_contact, send_dice, send_chat_action, forward_message, copy_message,
  edit_message_text/caption/media/reply_markup, delete_message, answer_callback_query,
  pin_chat_message, unpin_chat_message, get_chat, get_file, download_file, get_user_profile_photos,
  get_sticker_set, set_my_commands, get_my_commands, delete_my_commands, set_webhook, delete_webhook,
  get_webhook_info, get_updates, log_out, close_bot.
- Errors in attapy.errors: AttapyError, NetworkError, SoroushPlusAPIError, RPCError, BadRequest,
  ChatNotFound, MessageNotFound, UserNotFound, Unauthorized, AuthError, FloodWait(retry_after), ValidationError.
- Never invent classes (no Dispatcher, Bot, Router, executor). Never use aiohttp/requests. Always await.
- Read the token from the environment. Output complete runnable code.
```

### خلاصهٔ API برای مدل‌ها (Cheat-sheet)

```text
Robot(token, base_url=None, parse_mode=None, timeout=60)
  decorators : on_message(filter,commands,priority) | on_callback_query(data,filter,priority)
  lifecycle  : run() | poll() | start_polling() | stop() | close() | process_update(dict)
  errors     : bot.errors.catch(*exc) | bot.errors.add(func,*exc)
  helpers    : file_url(path)

Message    : id chat chat_id from_user text caption reply_to_message command_args match to_dict()
             reply reply_photo reply_video reply_audio reply_voice reply_document reply_animation
             reply_sticker reply_video_note reply_contact reply_location reply_dice
             edit_text edit_caption edit_reply_markup delete forward pin unpin
Chat       : id type title username is_private is_group
User       : id is_bot first_name last_name username language_code full_name mention
Callback   : id data from_user message answer(text,show_alert,url,cache_time)

filters    : command regex user chat private group supergroup channel text create   (& | ~)
Button     : inline keyboard row inline_markup keyboard_markup remove_keyboard force_reply builder
HandlerPriority: HIGHEST=0 HIGH=25 NORMAL=50 LOW=75 LOWEST=100
```

---

## عیب‌یابی

| مشکل | علت و راه‌حل |
|---|---|
| `NetworkError: Network error ...` | اینترنت/فیلتر، یا `base_url` اشتباه. آدرس رسمی را بررسی کنید |
| `Unauthorized` | توکن اشتباه یا باطل شده است |
| ربات پیام می‌گیرد ولی جواب نمی‌دهد | هندلر ثبت نشده یا فیلتر برقرار نیست؛ لاگ `INFO` را روشن کنید. یادتان باشد اولین هندلر منطبق اجرا می‌شود |
| `RuntimeWarning: coroutine ... was never awaited` | جلوی یک متد async `await` نگذاشته‌اید |
| خطا در هندلر دیده نمی‌شود | خطاها در لاگ `attapy` ثبت می‌شوند؛ `logging.basicConfig(level=logging.INFO)` یا `bot.errors.catch()` بگذارید |
| `FloodWait` زیاد | سرعت ارسال را کم کنید و `retry_after` را رعایت کنید |
| دکمه همیشه در حالت loading است | `await cb.answer()` را صدا نزده‌اید |
| فایل محلی ارسال نمی‌شود | مسیر فایل وجود ندارد؛ در این صورت رشته به‌عنوان file_id فرستاده می‌شود. مسیر را بررسی کنید |

---

## وضعیت پروژه و لایسنس

- **نسخه 0.1.0 (Beta).** تست‌های خودکار با یک سرور جعلی محلی انجام شده‌اند (ارسال، polling، هندلرها، فیلترها، دکمه‌ها، آپلود/دانلود، خطاها). **با سرور واقعی سروش‌پلاس تست نشده است؛** اگر رفتاری فرق داشت، لطفاً Issue باز کنید.
- نام متدها و endpointها از الگوی Bot API سروش‌پلاس پیروی می‌کنند؛ در دسترس بودن هر متد به سرور بستگی دارد.
- اجرای تست‌ها: `python tests/test_attapy.py`
- لایسنس: **MIT** — © 2026 Samyar

مشارکت (Pull Request) و گزارش باگ (Issue) پذیرفته می‌شود.
