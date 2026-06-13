---
name: tg-premium-ui
description: Стиль и каркас Telegram-ботов на aiogram 3 с премиум-эмодзи. Используй, когда нужно сделать красивое меню на inline-кнопках, премиум-эмодзи в кнопках (icon_custom_emoji_id) и текстах (через тег tg-emoji), цветные кнопки через style (success/primary/danger), живые тексты без воды, конфиг переменными наверху файла (без ENV) и служебные файлы рядом со скриптом (BASE_DIR), а также обязательные админ-функции: подробная статистика, обязательная подписка (ОП), бан/разбан, управление балансом, рекламные ссылки с раздельной статистикой и рассылка с фото+текстом+URL-кнопкой. Содержит пак премиум-эмодзи с ID и описаниями для выбора иконок.
---

# Telegram Premium UI + админ-каркас

Скилл задаёт единый «дорогой» стиль Telegram-бота и обязательный набор админ-механик. Стек: **aiogram 3.x**, `ParseMode.HTML`, хранилище — SQLite (`aiosqlite`), состояния — FSM.

> Все ID премиум-эмодзи в этом скилле — из паков, приведённых в §13. Используй ТОЛЬКО их (не подставляй ID из других ботов). Иконку под кнопку/строку выбирай по описанию в таблице §13.

---

## 0. Железные правила (читать первым)

1. **Меню строится только на inline-кнопках** (`InlineKeyboardMarkup`) — они привязаны к сообщению. Меню НЕ строится на reply-кнопках (нижние кнопки) — это выглядит плохо.
2. **Исключение для reply-кнопки.** Один частный случай разрешён: reply-кнопка **«Выбрать пользователя»** (`request_users`) в админке — чтобы выбрать/«передать» боту конкретного пользователя (для бана, начисления баланса и т.п.). Это разовая служебная кнопка, она появляется на один шаг и сразу убирается (`ReplyKeyboardRemove`). На reply-кнопках НЕ должно быть построено само меню. По умолчанию можно и просто просить прислать ID числом — оба варианта ниже.
3. **Премиум-эмодзи вставляются прямо в код, без констант и словарей.** ID пишется литералом в `icon_custom_emoji_id="..."` (кнопки) и в `<tg-emoji emoji-id="...">фолбэк</tg-emoji>` (тексты). Не выноси их в переменные — вставляй на месте, выбирая по таблице §13.
4. **Цветные кнопки — через `style`, дозированно** (см. §4.1). Подсвечивай цветом только одну важную кнопку, не всю клавиатуру.
5. **Тексты — живые.** Без канцелярита и воды: короткие фразы, обращение на «ты», лёгкий азарт и продажа выгоды. Ключевые слова подчёркиваются `<u>...</u>`.
6. **Навигация на `edit_text`.** Экраны перерисовываются в одном сообщении, «Назад» всегда последним рядом.

---

## 1. Конфиг и инициализация

### 1.1 Правила конфига

- **Весь конфиг — наверху файла, обычными переменными.** Никакого `.env`, `os.getenv`, `dotenv` и внешних конфиг-файлов: все значения (токен, ID админов, тексты, лимиты, цены) лежат прямо в коде, в одном блоке `CONFIG` в начале. Это сознательный выбор — всё меняется в одном месте.
- **Делай конфиг богатым.** Выноси в переменные как можно больше: тексты, эмодзи целевых кнопок, цены/пакеты, лимиты, тайминги, ссылки, флаги вкл/выкл функций. Чем больше параметров наверху — тем меньше придётся лезть в тело кода.
- **Все служебные файлы создаются рядом со скриптом.** База данных и любые файлы (логи, выгрузки, сессии) строятся от `BASE_DIR = папка, где лежит сам файл`, через `os.path.join(BASE_DIR, ...)`. Так бот работает одинаково из любой рабочей директории (cron, systemd, ручной запуск).

### 1.2 Блок конфига (шаблон — наверху файла)

```python
import os, time, asyncio, contextlib
import aiosqlite
from aiogram import Bot, Dispatcher, F, Router, BaseMiddleware
from aiogram.enums import ParseMode
from aiogram.client.default import DefaultBotProperties

# ===================== CONFIG (всё меняется здесь) =====================
# --- Базовая директория: всё служебное создаётся РЯДОМ с этим файлом ---
BASE_DIR = os.path.dirname(os.path.abspath(__file__))

# --- Доступы ---
BOT_TOKEN = "123456:AA...your_token_here"      # токен бота от @BotFather
ADMINS    = [61916459]                          # ID администраторов
SUPPORT   = "@your_support"                     # контакт поддержки

# --- Пути к служебным файлам (всегда от BASE_DIR, рядом с файлом) ---
DB_PATH       = os.path.join(BASE_DIR, "bot.db")        # база данных SQLite
LOG_PATH      = os.path.join(BASE_DIR, "bot.log")       # лог-файл
EXPORT_DIR    = os.path.join(BASE_DIR, "exports")       # выгрузки/файлы юзерам

# --- Экономика / баланс ---
START_BALANCE   = 10        # стартовый баланс новому пользователю
REF_BONUS       = 2         # бонус за приглашённого друга
CURRENCY_NAME   = "генераций"   # как называется внутренняя «валюта» в текстах
STAR_USD_RATE   = 0.013     # курс 1 Telegram Star ≈ $ (для статистики)
STAR_PACKAGES   = [10, 15, 30, 50, 100, 300]   # пакеты для магазина Stars

# --- Лимиты и тайминги ---
BROADCAST_BATCH      = 25   # обновлять прогресс рассылки каждые N отправок
BROADCAST_SLEEP      = 1.0  # пауза между батчами рассылки, сек (анти-flood)
ACTION_COOLDOWN_SEC  = 3    # антиспам между действиями пользователя, сек

# --- Переключатели функций (вкл/выкл одной строкой) ---
FORCE_SUB_ENABLED   = True  # обязательная подписка вкл/выкл
REFERRALS_ENABLED   = True  # реферальная программа вкл/выкл
GIVE_START_BONUS    = True  # выдавать стартовый баланс новичкам

# --- Эмодзи целевых кнопок/заголовков (ID из паков §13, меняй тут) ---
EMOJI_TITLE   = "5310004295817515167"   # ⭐️ заголовок главного экрана
EMOJI_BUY     = "5310071245767726345"   # 👑 покупка/премиум
EMOJI_BALANCE = "5309815175522571667"   # 💳 баланс
EMOJI_BACK    = "5309979024229948963"   # ⬅️ назад

# --- Тексты (правятся здесь, без лазания по коду) ---
TXT_WELCOME = "Привет! Этот бот <u>находит</u> тебе свободный username."
# =======================================================================

bot = Bot(token=BOT_TOKEN,
          default=DefaultBotProperties(parse_mode=ParseMode.HTML))
dp = Dispatcher()
router = Router()
dp.include_router(router)

def is_admin(uid: int) -> bool:
    return uid in ADMINS
```

`parse_mode=HTML` ставится один раз глобально — HTML-теги работают во всех `answer`/`edit_text`. Список параметров выше — это ориентир: добавляй в `CONFIG` всё, что захочешь менять (другие тексты, цены, иконки, лимиты, флаги).

### 1.3 Создание БД рядом с файлом

`DB_PATH` уже указывает в папку скрипта (`BASE_DIR`), поэтому база создаётся именно там, а не в текущей рабочей директории:

```python
async def init_db():
    os.makedirs(EXPORT_DIR, exist_ok=True)        # папки для служебных файлов — рядом
    async with aiosqlite.connect(DB_PATH) as db:  # bot.db создаётся возле скрипта
        await db.execute("""CREATE TABLE IF NOT EXISTS users (...)""")
        # ... остальные таблицы из §5
        await db.commit()
```

---

## 2. Премиум-эмодзи — два механизма

### 2.1 Иконка кнопки → `icon_custom_emoji_id`
Текст кнопки чистый (без эмодзи), премиум-иконка встаёт слева. ID — строкой, литералом, выбран по таблице §13.

```python
from aiogram.types import InlineKeyboardMarkup, InlineKeyboardButton

InlineKeyboardButton(text="Информация", callback_data="menu_info",
                     icon_custom_emoji_id="5310104514584399566")   # ℹ️ инфо
```

### 2.2 Эмодзи в тексте → `<tg-emoji>`
Тег `<tg-emoji emoji-id="ЦИФРЫ">обычный_эмодзи</tg-emoji>`. Внутри — обычный эмодзи-фолбэк. Вставляется прямо в строку.

```python
'<tg-emoji emoji-id="5310004295817515167">⭐️</tg-emoji> <b>Главное меню</b>'
```

> Премиум-эмодзи отрисуются у всех пользователей только если у бота есть к ним доступ; иначе покажется фолбэк — поэтому фолбэк подбирай осмысленно.

---

## 3. Живые тексты + подчёркивания

Правила «голоса»:
- Первая строка — премиум-эмодзи + `<b>Заголовок</b>`, затем `\n\n`.
- Каждый смысловой пункт начинается с эмодзи, дальше жирная подпись: `эмодзи <b>Подпись:</b> значение`.
- Числа, ссылки, ID — в `<code>`.
- Второстепенное и сгруппированное — в `<blockquote>`; длинные FAQ — в `<blockquote expandable>`.
- Интрига/скрытое — в `<tg-spoiler>`.
- **Ключевые слова подчёркивай** `<u>...</u>` — подсвечиваем выгоду и важные слова.
- `\n\n` между блоками, `\n` внутри блока.

Подчёркивания в живом тексте:

```python
text = (
    f'<b>Встречай — </b><a href="{ref_link}"><b>Генератор Username</b></a><b>!</b>\n\n'
    f'<b>Сгенерируй себе свободный, <u>ценный</u> юзернейм:</b>\n'
    f'<b><tg-emoji emoji-id="5310147356883178344">🛡</tg-emoji> Каждому новичку — <u>пробный</u> пакет 👇</b>'
)
```

Живой vs «ИИшный» (ориентир по тону):

```text
Плохо (вода):  «Данный бот предоставляет возможность осуществить генерацию
                username в соответствии с заданными параметрами.»
Хорошо (живо):  «Этот бот <u>находит</u> тебе свободный username — для профиля
                или на <u>продажу</u>. Таких ни у кого нет.»
```

Пример главного экрана:

```python
def text_main(balance: int, sold_count: int) -> str:
    return (
        f'<tg-emoji emoji-id="5310004295817515167">⭐️</tg-emoji> <b>Генератор username</b>\n\n'
        f'<b><tg-emoji emoji-id="5310071245767726345">👑</tg-emoji> Бот уже сгенерировал — <i>{sold_count}</i> username</b>\n\n'
        f'<blockquote><b><tg-emoji emoji-id="5309946893579605926">👤</tg-emoji> Сделай себе <u>ценный</u> username — для профиля или на продажу.\n'
        f'<tg-emoji emoji-id="5310001027347403454">🔄</tg-emoji> Генерируем уникальные 5/6/7-значные, таких <u>ни у кого</u> нет.</b></blockquote>\n\n'
        f'<tg-emoji emoji-id="5309815175522571667">💳</tg-emoji> <b>Баланс:</b> <code>{balance}</code> юзернеймов'
    )
```

Нумерованные списки — обычными жирными цифрами (в паке нет премиум-цифр):

```python
'<blockquote><b>1.</b> Кидай ссылку в чаты.\n'
'<b>2.</b> Запости её в своём канале.\n'
'<b>3.</b> Отправь друзьям.</blockquote>'
```

---

## 4. Inline-клавиатуры

Правила раскладки:
- Главные действия — каждое отдельным рядом во всю ширину `[InlineKeyboardButton(...)]`.
- Короткие однотипные кнопки — по 2–3 в ряд; длинные метки — отдельным рядом.
- «Назад» — всегда последним рядом, одна и та же иконка во всём боте (⬅️ `5309979024229948963`).
- Динамические метки строятся в коде (суффиксы `[ПРОБНЫЕ]`, ` — {цена} ⭐`).

```python
def kb_main() -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="Генерация юзернейма", callback_data="menu_generate",
                              icon_custom_emoji_id="5310288244695389656")],   # 🔎 поиск
        [InlineKeyboardButton(text="Купить юзернеймы", callback_data="menu_buy",
                              icon_custom_emoji_id="5310071245767726345")],   # 👑 премиум
        [InlineKeyboardButton(text="Бесплатные за друзей!", callback_data="menu_ref",
                              icon_custom_emoji_id="5309844291105869907")],   # 👥 рефералы
        [InlineKeyboardButton(text="Информация", callback_data="menu_info",
                              icon_custom_emoji_id="5310104514584399566")],   # ℹ️ инфо
    ])

def kb_back(callback: str = "menu_main") -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="Назад", callback_data=callback,
                              icon_custom_emoji_id="5309979024229948963")]   # ⬅️ назад
    ])

@router.callback_query(F.data == "menu_info")
async def cb_info(call: CallbackQuery):
    await call.message.edit_text(text_info(), reply_markup=kb_back(),
                                 disable_web_page_preview=True)
    await call.answer()
```

### 4.1 Цветные кнопки — параметр `style`

У `InlineKeyboardButton` есть параметр `style`. Значения:
- `style='success'` — **зелёная** кнопка (подтверждение, запуск, «всё ок»).
- `style='primary'` — **синяя/голубая** кнопка (главное действие, акцент).
- `style='danger'` — **красная** кнопка (опасное/необратимое: удалить, отменить, забанить).

```python
inline_keyboard=[
    [InlineKeyboardButton(text="Запустить рассылку", callback_data="bc_launch", style='success')],
    [InlineKeyboardButton(text="Купить пакет",        callback_data="menu_buy", style='primary')],
    [InlineKeyboardButton(text="Удалить ссылку",      callback_data="lnk_del", style='danger')],
]
```

**Не злоупотребляй.** Если вся клавиатура цветная — выглядит плохо. Цветом выделяют ОДНУ важную кнопку на экране (например, «Запустить» — зелёным, «Удалить» — красным), остальные оставляй обычными. `style` и `icon_custom_emoji_id` можно сочетать.

---

## 5. Схема базы данных (минимум под все функции)

```sql
CREATE TABLE IF NOT EXISTS users (
    id          INTEGER PRIMARY KEY,
    balance     INTEGER NOT NULL DEFAULT 0,   -- внутренняя «валюта» бота (см. §8)
    referrer_id INTEGER,
    created_at  INTEGER NOT NULL DEFAULT 0,   -- регистрация (для статистики по времени)
    source_link TEXT,                          -- имя рекламной ссылки захода
    banned      INTEGER NOT NULL DEFAULT 0,
    ban_reason  TEXT
);
CREATE TABLE IF NOT EXISTS payments (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER, stars INTEGER NOT NULL DEFAULT 0, ts INTEGER NOT NULL DEFAULT 0
);
CREATE TABLE IF NOT EXISTS subscriptions (     -- каналы обязательной подписки (ОП)
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    channel_id TEXT NOT NULL, invite_link TEXT NOT NULL, title TEXT NOT NULL,
    is_active INTEGER DEFAULT 1, wave INTEGER NOT NULL DEFAULT 1,
    UNIQUE(channel_id, wave)
);
CREATE TABLE IF NOT EXISTS ad_links (
    name TEXT PRIMARY KEY, created_at INTEGER NOT NULL DEFAULT 0,
    clicks INTEGER NOT NULL DEFAULT 0
);
CREATE TABLE IF NOT EXISTS link_visits (
    link_name TEXT NOT NULL, user_id INTEGER NOT NULL,
    ts INTEGER NOT NULL DEFAULT 0, is_new INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (link_name, user_id)
);
```

---

## 6. Админ-меню (точка входа, только inline)

```python
@router.message(Command("admin"))
async def cmd_admin(message: Message, state: FSMContext):
    if not is_admin(message.from_user.id):
        return
    await state.clear()
    await message.answer(
        '<tg-emoji emoji-id="5309810008676915393">💬</tg-emoji> <b>Панель администратора</b>',
        reply_markup=kb_admin())

def kb_admin() -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="Статистика", callback_data="adm_stats",
                              icon_custom_emoji_id="5310048641354845866")],   # 📊
        [InlineKeyboardButton(text="Обязательная подписка", callback_data="adm_op",
                              icon_custom_emoji_id="5310244156856097218")],   # ✈️
        [InlineKeyboardButton(text="Управление балансом", callback_data="adm_balance",
                              icon_custom_emoji_id="5309815175522571667")],   # 💳
        [InlineKeyboardButton(text="Бан / Разбан", callback_data="adm_ban",
                              icon_custom_emoji_id="5309946064650917180")],   # 🚫
        [InlineKeyboardButton(text="Рекламные ссылки", callback_data="adm_links",
                              icon_custom_emoji_id="5310008912907360202")],   # 🔗
        [InlineKeyboardButton(text="Рассылка", callback_data="adm_broadcast",
                              icon_custom_emoji_id="5310011257959508085")],   # ✉️
    ])

def kb_admin_back() -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[[InlineKeyboardButton(
        text="Назад", callback_data="adm_main",
        icon_custom_emoji_id="5309979024229948963")]])   # ⬅️
```

---

## 7. ОБЯЗАТЕЛЬНАЯ: подробная статистика

Срезы: всего/органика/рефералы, финансы (Stars + $), ARPU, динамика час/сутки/неделя, рефералы, ОП. Блок = премиум-эмодзи + жирный заголовок + строки `• метка: <code>значение</code>`.

```python
STAR_USD_RATE = 0.013  # 1 Star ≈ $0.013

@router.callback_query(F.data == "adm_stats")
async def cb_adm_stats(call: CallbackQuery):
    if not is_admin(call.from_user.id):
        return await call.answer()

    users     = await count_users()
    referred  = await count_referred()
    organic   = users - referred
    referrers = await count_referrers()
    stars     = await sum_stars_all()
    payments  = await count_payments_all()
    usd       = stars * STAR_USD_RATE
    avg_check = (stars / payments) if payments else 0
    conv      = (payments / users * 100) if users else 0
    arpu      = (stars / users) if users else 0

    now = int(time.time())
    nu_h = await count_new_users_since(now - 3600)
    nu_d = await count_new_users_since(now - 86400)
    nu_w = await count_new_users_since(now - 604800)
    st_h, pay_h = await payments_since(now - 3600)
    st_d, pay_d = await payments_since(now - 86400)
    st_w, pay_w = await payments_since(now - 604800)

    subs_all    = await get_all_subscriptions()
    subs_active = sum(1 for s in subs_all if s["is_active"] == 1)

    text = (
        '<tg-emoji emoji-id="5310048641354845866">📊</tg-emoji> <b>Бизнес-статистика</b>\n\n'
        '<tg-emoji emoji-id="5309946893579605926">👤</tg-emoji> <b>Пользователи</b>\n'
        f'   • Всего: <code>{users}</code>\n'
        f'   • Органических: <code>{organic}</code>\n'
        f'   • По рефералам: <code>{referred}</code>\n\n'
        '<tg-emoji emoji-id="5309815175522571667">💳</tg-emoji> <b>Финансы (Stars)</b>\n'
        f'   • Заработано: <code>{stars}</code> ⭐ (<code>${usd:.2f}</code>)\n'
        f'   • Оплат: <code>{payments}</code>\n'
        f'   • Средний чек: <code>{avg_check:.1f}</code> ⭐\n'
        f'   • Конверсия: <code>{conv:.1f}%</code>\n'
        f'   • ARPU: <code>{arpu:.2f}</code> ⭐\n\n'
        '<tg-emoji emoji-id="5310001027347403454">🔄</tg-emoji> <b>Динамика</b>\n'
        f'   • Час: новых <code>{nu_h}</code>, оплат <code>{pay_h}</code> на <code>{st_h}</code> ⭐\n'
        f'   • Сутки: новых <code>{nu_d}</code>, оплат <code>{pay_d}</code> на <code>{st_d}</code> ⭐\n'
        f'   • Неделя: новых <code>{nu_w}</code>, оплат <code>{pay_w}</code> на <code>{st_w}</code> ⭐\n\n'
        '<tg-emoji emoji-id="5309844291105869907">👥</tg-emoji> <b>Рефералы</b>\n'
        f'   • Активных рефереров: <code>{referrers}</code>\n\n'
        '<tg-emoji emoji-id="5310244156856097218">✈️</tg-emoji> <b>Обязательная подписка</b>\n'
        f'   • Каналов: <code>{len(subs_all)}</code> (активных: <code>{subs_active}</code>)'
    )
    await call.message.edit_text(text, reply_markup=kb_admin_back())
    await call.answer()
```

Базовые запросы:

```python
async def count_users() -> int:
    async with aiosqlite.connect(DB_PATH) as db:
        async with db.execute("SELECT COUNT(*) FROM users WHERE banned=0") as c:
            return (await c.fetchone())[0]

async def count_new_users_since(ts: int) -> int:
    async with aiosqlite.connect(DB_PATH) as db:
        async with db.execute("SELECT COUNT(*) FROM users WHERE created_at>=?", (ts,)) as c:
            return (await c.fetchone())[0]

async def payments_since(ts: int) -> tuple[int, int]:
    async with aiosqlite.connect(DB_PATH) as db:
        async with db.execute(
            "SELECT COALESCE(SUM(stars),0), COUNT(*) FROM payments WHERE ts>=?", (ts,)) as c:
            r = await c.fetchone(); return (r[0], r[1])
```

---

## 8. ОБЯЗАТЕЛЬНАЯ: управление балансом (валюта под каждого бота)

`balance` — абстрактная внутренняя валюта; в каждом боте своя (генерации, монеты, попытки, лимиты). Под конкретного юзера админ может выставить точное значение или добавить N. Нужны другие счётчики (например, бонусные рефералы) — заводи отдельную колонку и такие же `add_/set_` функции.

```python
async def add_balance(user_id: int, amount: int):
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute("UPDATE users SET balance=balance+? WHERE id=?", (amount, user_id)); await db.commit()

async def set_balance(user_id: int, amount: int):
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute("UPDATE users SET balance=? WHERE id=?", (amount, user_id)); await db.commit()

async def spend_balance(user_id: int, amount: int):
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute("UPDATE users SET balance=MAX(balance-?,0) WHERE id=?", (amount, user_id)); await db.commit()
```

### Вариант A — ввод ID числом (полностью inline, по умолчанию)

```python
class AdminStates(StatesGroup):
    balance_user = State(); balance_amount = State()

@router.callback_query(F.data == "adm_balance")
async def cb_adm_balance(call: CallbackQuery, state: FSMContext):
    if not is_admin(call.from_user.id):
        return await call.answer()
    await state.set_state(AdminStates.balance_user)
    await call.message.edit_text(
        '<tg-emoji emoji-id="5309815175522571667">💳</tg-emoji> <b>Управление балансом</b>\n\n'
        'Пришли <b>ID пользователя</b> числом 👇',
        reply_markup=kb_admin_back())
    await call.answer()

@router.message(AdminStates.balance_user)
async def adm_balance_user(message: Message, state: FSMContext):
    if not is_admin(message.from_user.id):
        return
    try:
        target = int((message.text or "").strip())
    except ValueError:
        return await message.answer("Некорректный ID. Пришли число:")
    user = await get_user(target) or await create_user(target)
    await state.update_data(target=target); await state.set_state(AdminStates.balance_amount)
    await message.answer(f'Юзер: <code>{target}</code>\nТекущий баланс: <b>{user["balance"]}</b>\n\n'
                         'Пришли <b>новое значение</b> (число):', reply_markup=kb_admin_back())

@router.message(AdminStates.balance_amount)
async def adm_balance_amount(message: Message, state: FSMContext):
    if not is_admin(message.from_user.id):
        return
    try:
        amount = int((message.text or "").strip())
    except ValueError:
        return await message.answer("Некорректное число. Повтори:")
    data = await state.get_data(); await set_balance(data["target"], amount); await state.clear()
    await message.answer(
        f'<tg-emoji emoji-id="5310147356883178344">🛡</tg-emoji> Баланс <code>{data["target"]}</code> '
        f'установлен: <b>{amount}</b>', reply_markup=kb_admin_back())
```

### Вариант B — reply-кнопка «Выбрать пользователя» (разрешённое исключение)

Удобно, если админ хочет ткнуть в пользователя из списка чатов, а не вводить ID. Reply-кнопка показывается на один шаг и сразу убирается через `ReplyKeyboardRemove` — она НЕ часть меню.

```python
from aiogram.types import (ReplyKeyboardMarkup, KeyboardButton,
                           KeyboardButtonRequestUsers, ReplyKeyboardRemove)

# вместо edit_text выше — добавляем отдельным сообщением reply-кнопку выбора:
pick_kb = ReplyKeyboardMarkup(
    keyboard=[[KeyboardButton(
        text="👤 Выбрать пользователя",
        request_users=KeyboardButtonRequestUsers(request_id=1, user_is_bot=False, max_quantity=1),
    )]],
    resize_keyboard=True, one_time_keyboard=True,
    input_field_placeholder="Или пришли ID числом…")
await call.message.answer("Выбери пользователя 👇", reply_markup=pick_kb)

# ловим выбранного и сразу убираем reply-клавиатуру:
@router.message(AdminStates.balance_user, F.users_shared)
async def adm_balance_user_shared(message: Message, state: FSMContext):
    if not is_admin(message.from_user.id):
        return
    target = message.users_shared.user_ids[0]
    user = await get_user(target) or await create_user(target)
    await state.update_data(target=target); await state.set_state(AdminStates.balance_amount)
    await message.answer(f'Выбран <code>{target}</code>. Пришли новое значение баланса:',
                         reply_markup=ReplyKeyboardRemove())
```

---

## 9. ОБЯЗАТЕЛЬНАЯ: бан / разбан

Кэш забаненных в памяти + middleware, который глушит апдейты забаненных (кроме админов).

```python
BANNED_USERS: dict[int, str] = {}

def is_banned(uid: int) -> bool:
    return uid in BANNED_USERS

async def ban_user(uid: int, reason: str):
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute(
            "INSERT INTO users (id, banned, ban_reason, created_at) VALUES (?,1,?,?) "
            "ON CONFLICT(id) DO UPDATE SET banned=1, ban_reason=excluded.ban_reason",
            (uid, reason, int(time.time()))); await db.commit()
    BANNED_USERS[uid] = reason

async def unban_user(uid: int):
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute("UPDATE users SET banned=0, ban_reason=NULL WHERE id=?", (uid,)); await db.commit()
    BANNED_USERS.pop(uid, None)

class BanMiddleware(BaseMiddleware):
    async def __call__(self, handler, event, data):
        u = data.get("event_from_user")
        if u and is_banned(u.id) and not is_admin(u.id):
            return
        return await handler(event, data)

dp.update.middleware(BanMiddleware())
```

FSM-флоу (inline, ID числом, причина — текстом; забаненному уходит уведомление; «Разбанить» — красным `danger`):

```python
class AdminBan(StatesGroup):
    ban_user = State(); ban_reason = State()

@router.callback_query(F.data == "adm_ban")
async def cb_adm_ban(call: CallbackQuery, state: FSMContext):
    if not is_admin(call.from_user.id):
        return await call.answer()
    await state.set_state(AdminBan.ban_user)
    await call.message.edit_text(
        '<tg-emoji emoji-id="5309946064650917180">🚫</tg-emoji> <b>Бан / Разбан</b>\n\n'
        'Пришли <b>ID пользователя</b> числом 👇', reply_markup=kb_admin_back())
    await call.answer()

@router.message(AdminBan.ban_user)
async def adm_ban_user(message: Message, state: FSMContext):
    if not is_admin(message.from_user.id):
        return
    try:
        target = int((message.text or "").strip())
    except ValueError:
        return await message.answer("Некорректный ID. Пришли число:")
    if target in ADMINS:
        await state.clear()
        return await message.answer("Нельзя забанить админа.", reply_markup=kb_admin_back())
    await state.update_data(target=target)
    if is_banned(target):
        kb = InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="Разбанить", callback_data=f"unban:{target}", style='success')],
            [InlineKeyboardButton(text="Назад", callback_data="adm_main",
                                  icon_custom_emoji_id="5309979024229948963")]])
        await message.answer(f'Юзер <code>{target}</code>: <b>забанен</b>\nПричина: {BANNED_USERS[target]}',
                             reply_markup=kb)
    else:
        await state.set_state(AdminBan.ban_reason)
        await message.answer(f'Юзер <code>{target}</code> не забанен.\n\nПришли <b>причину бана</b> '
                             '(её увидит пользователь):', reply_markup=kb_admin_back())

@router.message(AdminBan.ban_reason)
async def adm_ban_reason(message: Message, state: FSMContext):
    if not is_admin(message.from_user.id):
        return
    reason = (message.text or "").strip()
    if not reason:
        return await message.answer("Причина не может быть пустой:")
    target = (await state.get_data())["target"]
    await ban_user(target, reason); await state.clear()
    with contextlib.suppress(Exception):
        await bot.send_message(target,
            '<tg-emoji emoji-id="5309946064650917180">🚫</tg-emoji> <b>Ты заблокирован в боте.</b>\n\n'
            f'<b>Причина:</b> {reason}\n\nПо вопросам: {SUPPORT}')
    await message.answer(f'<tg-emoji emoji-id="5310147356883178344">🛡</tg-emoji> '
                         f'Юзер <code>{target}</code> забанен.', reply_markup=kb_admin_back())

@router.callback_query(F.data.startswith("unban:"))
async def cb_unban(call: CallbackQuery, state: FSMContext):
    if not is_admin(call.from_user.id):
        return await call.answer()
    await state.clear(); target = int(call.data.split(":", 1)[1]); await unban_user(target)
    with contextlib.suppress(Exception):
        await bot.send_message(target, "🛡 Ты разблокирован. Можешь снова пользоваться ботом.")
    await call.message.edit_text(f'<tg-emoji emoji-id="5310147356883178344">🛡</tg-emoji> '
                                 f'Юзер <code>{target}</code> разбанен.', reply_markup=kb_admin_back())
    await call.answer("Разбанен")
```

---

## 10. ОБЯЗАТЕЛЬНАЯ: обязательная подписка (ОП)

Перед целевым действием проверяем подписку (или поданную заявку). Не подписан → кнопки каналов + «Проверить». Управление каналами — в админ-меню.

```python
async def get_active_subscriptions(wave: int = 1):
    async with aiosqlite.connect(DB_PATH) as db:
        db.row_factory = aiosqlite.Row
        async with db.execute("SELECT * FROM subscriptions WHERE is_active=1 AND wave=?", (wave,)) as c:
            return await c.fetchall()

async def get_not_subscribed(user_id: int, wave: int = 1):
    subs = await get_active_subscriptions(wave); not_ok = []
    for sub in subs:
        passed = False
        try:
            m = await bot.get_chat_member(sub["channel_id"], user_id)
            passed = m.status not in ("left", "kicked")
        except Exception:
            passed = False
        if not passed:
            not_ok.append(sub)
    return not_ok

def kb_subscriptions(subs, check_cb: str = "check_sub") -> InlineKeyboardMarkup:
    rows = [[InlineKeyboardButton(text="Подписаться", url=s["invite_link"],
                                  icon_custom_emoji_id="5310244156856097218")] for s in subs]   # ✈️
    rows.append([InlineKeyboardButton(text="Проверить подписку", callback_data=check_cb,
                                      style='success')])   # зелёная — главное действие экрана
    rows.append([InlineKeyboardButton(text="Назад", callback_data="menu_main",
                                      icon_custom_emoji_id="5309979024229948963")])
    return InlineKeyboardMarkup(inline_keyboard=rows)
```

Гейт перед действием:

```python
not_sub = await get_not_subscribed(user_id)
if not_sub:
    await call.message.edit_text(
        '<tg-emoji emoji-id="5309836186502587572">❗️</tg-emoji> <b>Доступ ограничен!</b>\n\n'
        'Подпишись на каналы ниже — и продолжим 👇',
        reply_markup=kb_subscriptions(not_sub))
    return
```

Управление каналами ОП в админке (FSM, всё inline): админ присылает по шагам название, инвайт-ссылку и ID/username канала (бот — админ в нём), запись падает в `subscriptions`; удаление — кнопкой `del_sub:{id}` (красная `danger`).

---

## 11. ОБЯЗАТЕЛЬНАЯ: рекламные ссылки с раздельной статистикой

У каждой ссылки своё имя и СВОЯ статистика (клики, уники, новые, оплаты этой когорты). Имя передаётся через deep-link `?start=ИМЯ` и пишется в `users.source_link` при первом заходе.

```python
@router.message(CommandStart(deep_link=True))
async def cmd_start_deeplink(message: Message, command: CommandObject, state: FSMContext):
    arg = command.args or ""
    source = arg if not arg.startswith("ref_") else None
    is_new = await get_user(message.from_user.id) is None
    if is_new:
        await create_user(message.from_user.id, source_link=source)
    if source and await get_ad_link(source):
        await register_link_visit(source, message.from_user.id, is_new)
    # ... показать главное меню

async def register_link_visit(link_name: str, user_id: int, is_new: bool):
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute("UPDATE ad_links SET clicks=clicks+1 WHERE name=?", (link_name,))
        await db.execute(
            "INSERT OR IGNORE INTO link_visits (link_name, user_id, ts, is_new) VALUES (?,?,?,?)",
            (link_name, user_id, int(time.time()), 1 if is_new else 0))
        await db.commit()

async def ad_link_full_stats(name: str) -> dict:
    async with aiosqlite.connect(DB_PATH) as db:
        async with db.execute("SELECT clicks FROM ad_links WHERE name=?", (name,)) as c:
            clicks = (await c.fetchone() or [0])[0]
        async with db.execute(
            "SELECT COUNT(*), COALESCE(SUM(is_new),0) FROM link_visits WHERE link_name=?", (name,)) as c:
            r = await c.fetchone(); visitors, new_users = r[0], r[1]
        async with db.execute("SELECT COUNT(*) FROM users WHERE source_link=?", (name,)) as c:
            registered = (await c.fetchone())[0]
        async with db.execute(
            "SELECT COALESCE(SUM(p.stars),0), COUNT(*) FROM payments p "
            "JOIN users u ON u.id=p.user_id WHERE u.source_link=?", (name,)) as c:
            r = await c.fetchone(); stars, pays = r[0], r[1]
    return {"clicks": clicks, "visitors": visitors, "new_users": new_users,
            "registered": registered, "stars": stars, "pays": pays}
```

Меню ссылок и карточка одной ссылки (inline; «Удалить» — красная `danger`):

```python
@router.callback_query(F.data == "adm_links")
async def cb_adm_links(call: CallbackQuery, state: FSMContext):
    if not is_admin(call.from_user.id):
        return await call.answer()
    links = await get_all_ad_links()
    rows = [[InlineKeyboardButton(text=f"🔗 {l['name']} — {l['clicks']} кликов",
             callback_data=f"lnk_v_{l['name']}",
             icon_custom_emoji_id="5310008912907360202")] for l in links]   # 🔗
    rows.append([InlineKeyboardButton(text="Создать ссылку", callback_data="adm_link_new",
                                      style='primary')])
    rows.append([InlineKeyboardButton(text="Назад", callback_data="adm_main",
                                      icon_custom_emoji_id="5309979024229948963")])
    await call.message.edit_text(
        '<tg-emoji emoji-id="5310008912907360202">🔗</tg-emoji> <b>Рекламные ссылки</b>\n\n'
        'У каждой ссылки — своя статистика переходов и оплат.',
        reply_markup=InlineKeyboardMarkup(inline_keyboard=rows))
    await call.answer()

@router.callback_query(F.data.startswith("lnk_v_"))
async def cb_link_view(call: CallbackQuery):
    if not is_admin(call.from_user.id):
        return await call.answer()
    name = call.data[len("lnk_v_"):]; s = await ad_link_full_stats(name); me = await bot.get_me()
    text = (
        f'<tg-emoji emoji-id="5310008912907360202">🔗</tg-emoji> <b>Ссылка:</b> <code>{name}</code>\n\n'
        f'<b>Ссылка для рекламы:</b>\n<code>https://t.me/{me.username}?start={name}</code>\n\n'
        f'<tg-emoji emoji-id="5310048641354845866">📊</tg-emoji> <b>Статистика</b>\n'
        f'   • Кликов: <code>{s["clicks"]}</code>\n'
        f'   • Уникальных: <code>{s["visitors"]}</code> (новых: <code>{s["new_users"]}</code>)\n'
        f'   • Зарегистрировано: <code>{s["registered"]}</code>\n'
        f'   • Оплат: <code>{s["pays"]}</code> на <code>{s["stars"]}</code> ⭐')
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="Удалить ссылку", callback_data=f"lnk_d_{name}", style='danger')],
        [InlineKeyboardButton(text="Назад", callback_data="adm_links",
                              icon_custom_emoji_id="5309979024229948963")]])
    await call.message.edit_text(text, reply_markup=kb, disable_web_page_preview=True)
    await call.answer()
```

---

## 12. ОБЯЗАТЕЛЬНАЯ: рассылка (фото + текст + URL-кнопка)

Админ присылает ЛЮБОЕ сообщение (фото с подписью, текст, видео), бот его запоминает и **копирует** каждому через `bot.copy_message` — фото и форматирование сохраняются. Опционально снизу вешается inline-кнопка с URL. Перед запуском — превью и подтверждение («Запустить» — зелёная `success`, «Отменить» — красная `danger`).

```python
class BroadcastStates(StatesGroup):
    wait_content = State(); ask_url_btn = State()
    wait_btn_text = State(); wait_btn_url = State(); confirm = State()

@router.callback_query(F.data == "adm_broadcast")
async def cb_adm_broadcast(call: CallbackQuery, state: FSMContext):
    if not is_admin(call.from_user.id):
        return await call.answer()
    await state.set_state(BroadcastStates.wait_content)
    await call.message.edit_text(
        '<tg-emoji emoji-id="5310011257959508085">✉️</tg-emoji> <b>Рассылка</b>\n\n'
        'Пришли сообщение для рассылки — <b>фото с подписью</b>, текст или видео 👇',
        reply_markup=kb_admin_back())
    await call.answer()

@router.message(BroadcastStates.wait_content)
async def bc_content(message: Message, state: FSMContext):
    if not is_admin(message.from_user.id):
        return
    await state.update_data(msg_id=message.message_id, chat_id=message.chat.id)
    await state.set_state(BroadcastStates.ask_url_btn)
    kb = InlineKeyboardMarkup(inline_keyboard=[[
        InlineKeyboardButton(text="Да", callback_data="bc_btn_yes"),
        InlineKeyboardButton(text="Нет", callback_data="bc_btn_no")]])
    await message.answer("Добавить URL-кнопку под пост?", reply_markup=kb)

@router.callback_query(BroadcastStates.ask_url_btn, F.data == "bc_btn_no")
async def bc_no_btn(call: CallbackQuery, state: FSMContext):
    await state.update_data(has_url_btn=False); await call.answer(); await bc_preview(call.message, state)

@router.callback_query(BroadcastStates.ask_url_btn, F.data == "bc_btn_yes")
async def bc_yes_btn(call: CallbackQuery, state: FSMContext):
    await state.set_state(BroadcastStates.wait_btn_text)
    await call.message.edit_text("Пришли <b>текст кнопки</b>:"); await call.answer()

@router.message(BroadcastStates.wait_btn_text)
async def bc_btn_text(message: Message, state: FSMContext):
    await state.update_data(btn_text=message.text.strip())
    await state.set_state(BroadcastStates.wait_btn_url)
    await message.answer("Теперь пришли <b>URL</b> (https://…):")

@router.message(BroadcastStates.wait_btn_url)
async def bc_btn_url(message: Message, state: FSMContext):
    url = message.text.strip()
    if not url.startswith(("http://", "https://")):
        return await message.answer("URL должен начинаться с http:// или https://. Повтори:")
    await state.update_data(btn_url=url, has_url_btn=True); await bc_preview(message, state)

def _bc_kb(data) -> InlineKeyboardMarkup | None:
    if data.get("has_url_btn"):
        return InlineKeyboardMarkup(inline_keyboard=[[
            InlineKeyboardButton(text=data["btn_text"], url=data["btn_url"])]])
    return None

async def bc_preview(msg: Message, state: FSMContext):
    data = await state.get_data()
    await msg.answer("<b>Превью рассылки:</b>")
    await bot.copy_message(chat_id=msg.chat.id, from_chat_id=data["chat_id"],
                           message_id=data["msg_id"], reply_markup=_bc_kb(data))
    await state.set_state(BroadcastStates.confirm)
    ctrl = InlineKeyboardMarkup(inline_keyboard=[[
        InlineKeyboardButton(text="Запустить", callback_data="bc_launch", style='success'),
        InlineKeyboardButton(text="Отменить", callback_data="bc_cancel", style='danger')]])
    await msg.answer("<b>Подтвердить запуск?</b>", reply_markup=ctrl)

@router.callback_query(BroadcastStates.confirm, F.data == "bc_cancel")
async def bc_cancel(call: CallbackQuery, state: FSMContext):
    await state.clear()
    await call.message.edit_text("Рассылка отменена.", reply_markup=kb_admin_back()); await call.answer()

@router.callback_query(BroadcastStates.confirm, F.data == "bc_launch")
async def bc_launch(call: CallbackQuery, state: FSMContext):
    data = await state.get_data(); kb = _bc_kb(data); await state.clear()
    user_ids = await get_all_user_ids(); sent = failed = 0
    status = await call.message.edit_text(f"📤 Рассылка... 0/{len(user_ids)}"); await call.answer()
    for i, uid in enumerate(user_ids, 1):
        try:
            await bot.copy_message(chat_id=uid, from_chat_id=data["chat_id"],
                                   message_id=data["msg_id"], reply_markup=kb)
            sent += 1
        except Exception:
            failed += 1
        if i % 25 == 0:                       # троттлинг от flood
            with contextlib.suppress(Exception):
                await status.edit_text(f"📤 Рассылка... {i}/{len(user_ids)}")
            await asyncio.sleep(1)
    await status.edit_text(
        f'<tg-emoji emoji-id="5310147356883178344">🛡</tg-emoji> <b>Рассылка завершена!</b>\n\n'
        f'Доставлено: <code>{sent}</code>\nОшибок: <code>{failed}</code>',
        reply_markup=kb_admin_back())
```

Почему `copy_message`, а не `send_photo`: админ один раз присылает готовый пост (фото + подпись с любым форматированием), и копия уходит каждому без пересборки — фото, текст и стиль сохраняются как есть, плюс своя URL-кнопка.

---

## 13. Пак премиум-эмодзи (вставлять литералом по описанию)

Бери ID и вставляй прямо в `icon_custom_emoji_id="..."` или `<tg-emoji emoji-id="...">фолбэк</tg-emoji>`. Иконку выбирай по колонке «Когда использовать». Не выноси в константы.

### Пак A (как на 1-й фотографии)
| Фолбэк | ID | Когда использовать |
|--------|----|--------------------|
| ⭐️ | 5310004295817515167 | Заголовок главного экрана, избранное, премиум |
| 🎁 | 5309871791781467459 | Подарок, бонус, акция |
| ℹ️ | 5310104514584399566 | Информация, «о боте» |
| ❗️ | 5309932110302170924 | Важно, предупреждение |
| ❓ | 5310193995933044872 | Помощь, FAQ, поддержка |
| 👤 | 5309946893579605926 | Профиль, аккаунт |
| ⬇️ | 5310199703944580853 | Скачать, загрузить |
| 📊 | 5310048641354845866 | Статистика, аналитика |
| 🔑 | 5310212099220198077 | Доступ, ключ, авторизация |
| 🔧 | 5309878852707701235 | Настройки, инструменты |
| 🤖 | 5309944754685890777 | Бот, автоматизация |
| 🔎 | 5310288244695389656 | Поиск, генерация/подбор |
| ⬅️ | 5309979024229948963 | Назад (основная кнопка навигации) |
| ◀️ | 5310004652299801676 | Назад, предыдущий |
| 🍔 | 5310142546519806123 | Главное меню (бургер) |
| 🗂 | 5309834545825083238 | Список, категории |
| 🚪 | 5310141932339483132 | Выход |
| 🌐 | 5309921647761839867 | Язык, сеть |
| 📁 | 5309748809687913390 | Папка, разделы |
| ⭐ | 5310041288370836771 | Закладка, отметить |
| 🔗 | 5312332103667438675 | Обновить/синхронизация |
| 🔗 | 5310290722891518666 | Реферальная ссылка |
| 🔒 | 5312008847248870505 | Приватность, блокировка |
| 👥 | 5309815527709889358 | Рефералы, друзья |
| 📨 | 5309920844602956189 | Сообщения, чат |
| 🎤 | 5310176837538697844 | Голос, аудио, запись |
| ⏲ | 5310058180477211356 | Таймер, история |
| 🔔 | 5309761368172287857 | Уведомления вкл |
| 🔕 | 5310276429240356580 | Уведомления выкл |
| 🆔 | 5310024172926161438 | ID, карточка пользователя |
| 💳 | 5309815175522571667 | Оплата, баланс, кошелёк |
| ❤️ | 5310302327893152390 | Лайк, поддержать |
| 👑 | 5310071245767726345 | Премиум, VIP, покупка |
| 👁 | 5310227423663510719 | Просмотр, видимость |
| 🏠 | 5310110360034889831 | Главная, домой |
| 🛡 | 5309854985574441263 | Защита, безопасность |

### Пак B (как на 2-й фотографии)
| Фолбэк | ID | Когда использовать |
|--------|----|--------------------|
| 💬 | 5309810008676915393 | Чат, сообщения, админ-панель, отзывы |
| 🔄 | 5310001027347403454 | Обновить, синхронизация, перезапуск, динамика |
| 🔔 | 5309761368172287857 | Уведомления |
| 🏳 | 5309966611774460686 | Жалоба, сдаться, сброс |
| ✋ | 5310136877162976288 | Стоп, ограничение, «стоп-действие» |
| ❗️ | 5309836186502587572 | Важно, ошибка, предупреждение |
| 🛡 | 5310147356883178344 | Проверено/успех, защита, «готово» (замена ✅) |
| 🐶 (@) | 5312018068543657325 | Юзернейм, упоминание, @-ник |
| 🚪 | 5310141932339483132 | Выход |
| 🗂 | 5309834545825083238 | Список, категории |
| ◀️ | 5310004652299801676 | Назад, предыдущий |
| ⬅️ | 5309804979270208139 | Назад |
| ⬅️ | 5309979024229948963 | Назад / отменить (основная кнопка навигации) |
| 👥 | 5309844291105869907 | Группа, участники, рефералы |
| 👤 | 5309901482890382924 | Пользователь, профиль |
| 🚫 | 5309946064650917180 | Бан, запрет, отмена |
| ℹ️ | 5310104514584399566 | Информация |
| ✈️ | 5310244156856097218 | Отправить, перейти, подписаться |
| ⭐️ | 5310004295817515167 | Избранное, премиум |
| 👤 | 5309946893579605926 | Профиль |
| 🔧 | 5309878852707701235 | Настройки |
| 🔎 | 5310288244695389656 | Поиск |
| ⭐ | 5310041288370836771 | Закладка, отметить |
| 🔗 | 5310008912907360202 | Ссылка |
| ↗️ | 5310300738755252965 | Внешняя ссылка, открыть, перейти наружу |
| ✉️ | 5310011257959508085 | Письмо, рассылка, сообщение |
| ➡️ | 5309944844880204293 | Далее, вперёд, продолжить |
| 👥 | 5309815527709889358 | Рефералы, друзья |
| 📤 | 5310052090213585287 | Отправить, поделиться, запустить рассылку |
| 🗂 | 5310021570175980504 | Документ, лог, список |

---

## 14. Чеклист

- [ ] Весь конфиг — наверху файла, обычными переменными; без `.env`/`os.getenv`. Параметров побольше (тексты, цены, лимиты, флаги, эмодзи).
- [ ] Все служебные файлы (БД, логи, выгрузки) строятся от `BASE_DIR` через `os.path.join` — создаются рядом со скриптом.
- [ ] Меню — только `InlineKeyboardMarkup`; reply-кнопки нет (кроме разовой `request_users` «Выбрать пользователя» в админке).
- [ ] `parse_mode=HTML` в `DefaultBotProperties`.
- [ ] Премиум-эмодзи только из паков §13, вставлены литералом, без констант.
- [ ] В тексте кнопки нет эмодзи, если задан `icon_custom_emoji_id`.
- [ ] `style` (success/primary/danger) — только на одной важной кнопке экрана, не на всей клавиатуре.
- [ ] Ключевые слова подчёркнуты `<u>...</u>`; тексты живые, без воды.
- [ ] Числа, ID, ссылки — в `<code>`.
- [ ] «Назад» последним рядом, единая иконка ⬅️ во всём боте.
- [ ] Есть все обязательные админ-функции: статистика, ОП, бан, баланс, ссылки с раздельной статистикой, рассылка фото+текст+URL-кнопка.
- [ ] Рассылка через `copy_message` + троттлинг каждые 25 отправок.
- [ ] Забаненные глушатся middleware; админы под бан не попадают.
