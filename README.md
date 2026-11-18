# n8n: Daily TSETMC Price + Option Chain → Postgres

[English](#english) | [فارسی](#فارسی)

---

## English

An n8n workflow that collects data from the Tehran Stock Exchange (TSETMC) through the [brsapi.ir](https://brsapi.ir) API and stores it in PostgreSQL:

- **Daily price** of one symbol (default: `اهرم`) → `market.daily_prices`
- **Option chain** whose underlying is that symbol → `market.option_contracts`

Both tables are written with **upserts**, so running the workflow several times never creates duplicate rows.

### How it works

```
Daily Trigger → Config ─┬→ Fetch Daily Price → Prepare Price Row → Upsert Daily Price (Postgres)
                        └→ Fetch Option Chain → Filter Target Underlying → Prepare Option Row → Upsert Option Contract (Postgres)
```

| Node | What it does |
|---|---|
| Daily Trigger | Runs Saturday–Wednesday at 15:00 (cron `0 15 * * 6,0,1,2,3`, timezone `Asia/Tehran`) |
| Config | Holds the `symbol` to track |
| Fetch Daily Price / Fetch Option Chain | `GET` requests to brsapi.ir (3 retries, 5 s apart) |
| Prepare Price Row / Prepare Option Row | Validate and convert values; fail with a clear error if the API response is unexpected |
| Filter Target Underlying | Keeps only option contracts whose underlying equals `symbol` |
| Upsert … (Postgres) | `INSERT … ON CONFLICT DO UPDATE` using query parameters (no string-built SQL) |

### Files

| File | Purpose |
|---|---|
| `tsetmc-daily-workflow.json` | The n8n workflow |
| `schema.sql` | Creates the `market` schema and both tables |

### Requirements

- An n8n instance (recent version; the workflow uses Postgres node v2.8 and HTTP Request node v4.5)
- A PostgreSQL database
- An API key from [brsapi.ir](https://brsapi.ir)

### Setup

1. Run `schema.sql` on your PostgreSQL database.
2. Import the workflow into n8n:
   - **⋯ → Import from File…**, or
   - **⋯ → Import from URL…** with  
     `https://raw.githubusercontent.com/USERNAME/n8n-tsetmc-daily-workflow/main/tsetmc-daily-workflow.json`
3. Create a **Postgres credential** and select it in both Postgres nodes.
4. Paste your API key into the `key` query parameter of **both** HTTP Request nodes (`Fetch Daily Price` and `Fetch Option Chain`), replacing `PASTE_YOUR_API_KEY_HERE`.
5. In the **Config** node, set `symbol` (Persian ticker, e.g. `اهرم`).
6. Open **Workflow Settings → Timezone** and set `Asia/Tehran`.
7. Execute each node once to test, then **activate** the workflow.

### Tables

**`market.daily_prices`** — primary key `(symbol, trade_date_jalali)`

| Column | Meaning |
|---|---|
| `trade_date_jalali`, `time` | Jalali date and time of the record |
| `tno`, `tvol`, `tval` | Number of trades, volume, value |
| `pmin`, `pmax` | Lowest / highest price of the day |
| `py`, `pf` | Yesterday's price / first price |
| `pl`, `plc`, `plp` | Last price, its change and percent change |
| `pc`, `pcc`, `pcp` | Closing price, its change and percent change |

**`market.option_contracts`** — primary key `isin`. Holds the latest snapshot of each option contract (prices, strike, expiration date, open interest, trade count/volume/value, underlying prices, and so on).

### Notes

- `market.option_contracts` stores only the **latest state** of each contract, not history. Expired contracts stay in the table.
- Public holidays are not excluded from the schedule; re-running is harmless because of the upserts.
- **Never commit your API key.** Keep the placeholder in the repository and set the real key only inside your own n8n instance.
- For failure alerts, attach an Error Workflow (Error Trigger → Telegram / Email).

### Disclaimer

This project is not affiliated with TSETMC or brsapi.ir. The data is provided for informational purposes only and is not financial advice.

---

## فارسی

<div dir="rtl">

یک ورک‌فلوی n8n که داده‌های بورس تهران (TSETMC) را از طریق API سرویس [brsapi.ir](https://brsapi.ir) می‌گیرد و در PostgreSQL ذخیره می‌کند:

- **قیمت روزانه** یک نماد (پیش‌فرض: `اهرم`) ← جدول `market.daily_prices`
- **زنجیره‌ی قراردادهای اختیار معامله** که دارایی پایه‌شان همان نماد است ← جدول `market.option_contracts`

هر دو جدول با **upsert** نوشته می‌شوند؛ پس اجرای چندباره‌ی ورک‌فلو هیچ‌وقت رکورد تکراری نمی‌سازد.

### نحوه‌ی کار

</div>

```
Daily Trigger → Config ─┬→ Fetch Daily Price → Prepare Price Row → Upsert Daily Price (Postgres)
                        └→ Fetch Option Chain → Filter Target Underlying → Prepare Option Row → Upsert Option Contract (Postgres)
```

<div dir="rtl">

| نود | کارکرد |
|---|---|
| Daily Trigger | شنبه تا چهارشنبه ساعت ۱۵ اجرا می‌شود (cron: `0 15 * * 6,0,1,2,3` و تایم‌زون `Asia/Tehran`) |
| Config | نمادی که باید دنبال شود (`symbol`) را نگه می‌دارد |
| Fetch Daily Price / Fetch Option Chain | درخواست `GET` به brsapi.ir (۳ بار تلاش مجدد، با فاصله‌ی ۵ ثانیه) |
| Prepare Price Row / Prepare Option Row | مقادیر را اعتبارسنجی و تبدیل می‌کنند؛ اگر پاسخ API غیرمنتظره باشد با خطای واضح متوقف می‌شوند |
| Filter Target Underlying | فقط قراردادهایی را نگه می‌دارد که دارایی پایه‌شان برابر `symbol` است |
| Upsert … (Postgres) | `INSERT … ON CONFLICT DO UPDATE` با پارامتر (بدون ساختن SQL با رشته‌چسبانی) |

### فایل‌ها

| فایل | کاربرد |
|---|---|
| `tsetmc-daily-workflow.json` | خود ورک‌فلوی n8n |
| `schema.sql` | اسکیمای `market` و هر دو جدول را می‌سازد |

### پیش‌نیازها

- یک n8n (نسخه‌ی جدید؛ ورک‌فلو از نود Postgres نسخه‌ی ۲.۸ و HTTP Request نسخه‌ی ۴.۵ استفاده می‌کند)
- یک دیتابیس PostgreSQL
- کلید API از [brsapi.ir](https://brsapi.ir)

### راه‌اندازی

1. فایل `schema.sql` را روی دیتابیس PostgreSQL اجرا کن.
2. ورک‌فلو را در n8n وارد کن:
   - **⋯ ← Import from File…** یا
   - **⋯ ← Import from URL…** با آدرس  
     `https://raw.githubusercontent.com/USERNAME/n8n-tsetmc-daily-workflow/main/tsetmc-daily-workflow.json`
3. یک **Credential پستگرس** بساز و در هر دو نود Postgres انتخابش کن.
4. کلید API خودت را در پارامتر `key` در **هر دو** نود HTTP Request (`Fetch Daily Price` و `Fetch Option Chain`) به‌جای `PASTE_YOUR_API_KEY_HERE` بگذار.
5. در نود **Config** مقدار `symbol` را (نماد فارسی، مثلاً `اهرم`) تنظیم کن.
6. از **Workflow Settings ← Timezone** گزینه‌ی `Asia/Tehran` را انتخاب کن.
7. هر نود را یک‌بار Execute کن تا تست شود، بعد ورک‌فلو را **Active** کن.

### جدول‌ها

**`market.daily_prices`** — کلید اصلی `(symbol, trade_date_jalali)`

| ستون | معنی |
|---|---|
| `trade_date_jalali`, `time` | تاریخ شمسی و ساعت رکورد |
| `tno`, `tvol`, `tval` | تعداد معاملات، حجم، ارزش |
| `pmin`, `pmax` | کمترین / بیشترین قیمت روز |
| `py`, `pf` | قیمت دیروز / اولین قیمت |
| `pl`, `plc`, `plp` | آخرین قیمت، تغییر و درصد تغییر آن |
| `pc`, `pcc`, `pcp` | قیمت پایانی، تغییر و درصد تغییر آن |

**`market.option_contracts`** — کلید اصلی `isin`. آخرین وضعیت هر قرارداد اختیار را نگه می‌دارد (قیمت‌ها، قیمت اعمال، تاریخ سررسید، موقعیت باز، تعداد/حجم/ارزش معاملات، قیمت‌های دارایی پایه و غیره).

### نکته‌ها

- جدول `market.option_contracts` فقط **آخرین وضعیت** هر قرارداد را نگه می‌دارد، نه تاریخچه. قراردادهای منقضی‌شده در جدول می‌مانند.
- تعطیلات رسمی از زمان‌بندی حذف نشده‌اند؛ اجرای دوباره به‌خاطر upsert مشکلی ایجاد نمی‌کند.


</div>
