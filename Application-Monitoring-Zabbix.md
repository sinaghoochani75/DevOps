# مانیتورینگ یک اپلیکیشن در Zabbix

## ۱. هدف

هدف، مانیتورینگ وضعیت اجرای یک اپلیکیشن لینوکسی در Zabbix و نمایش هشدار در داشبورد هنگام توقف پردازش آن است. پس از اجرای مجدد برنامه، هشدار باید به حالت برطرف‌شده برگردد.

## ۲. مشخصات نمونه

| مؤلفه | مقدار نمونه |
|---|---|
| Zabbix Server | `zabbix-server` |
| نام فنی هاست | `app-server.example.local` |
| نام نمایشی هاست | `Application Server` |
| نام پردازش | `sample_app` |
| نام آیتم | `Application Process Status` |
| کلید آیتم | `app.process.running` |
| نام Trigger | `Application Process Stopped` |
| فاصله جمع‌آوری داده | `30s` |

> مقادیر بالا صرفاً نمونه هستند. نام هاست، نام پردازش و کلید آیتم را با مقادیر واقعی محیط خود جایگزین کنید.

## ۳. تعریف UserParameter در Zabbix Agent

اگر اپلیکیشن به‌صورت پردازش لینوکسی اجرا می‌شود و سرویس systemd نیست، می‌توان وضعیت آن را با `pgrep` بررسی کرد.

فایل زیر را روی سرور اپلیکیشن ایجاد کنید:

`/etc/zabbix/zabbix_agentd.d/app_process.conf`

محتوای فایل:

```bash
UserParameter=app.process.running,pgrep -x sample_app >/dev/null 2>&1 && echo 1 || echo 0
```

سپس Agent را راه‌اندازی مجدد کنید:

```bash
sudo systemctl restart zabbix-agent
```

اگر اپلیکیشن نام پردازش متفاوتی دارد، `sample_app` را با نام واقعی پردازش جایگزین کنید.

## ۴. بررسی عملکرد Agent

از روی سرور Zabbix دستور زیر را اجرا کنید:

```bash
zabbix_get -s <APP_SERVER_IP> -k app.process.running
```

- خروجی `1`: پردازش در حال اجراست.
- خروجی `0`: پردازش پیدا نشده است.

به‌جای `<APP_SERVER_IP>` آدرس سرور اپلیکیشن را قرار دهید.

## ۵. ساخت Item در رابط Zabbix

مسیر:

`Data collection → Hosts → Application Server → Items → Create item`

تنظیمات:

| فیلد | مقدار |
|---|---|
| Name | `Application Process Status` |
| Type | `Zabbix agent` |
| Key | `app.process.running` |
| Type of information | `Numeric (unsigned)` |
| Update interval | `30s` |
| History | `7d` |

این آیتم وضعیت پردازش را به‌صورت دوره‌ای دریافت می‌کند.

## ۶. ساخت Trigger برای توقف برنامه

مسیر:

`Data collection → Hosts → Application Server → Triggers → Create trigger`

تنظیمات:

| فیلد | مقدار |
|---|---|
| Name | `Application Process Stopped` |
| Severity | `High` |
| Problem expression | `last(/app-server.example.local/app.process.running)=0` |
| Recovery mode | `Recovery expression` |
| Recovery expression | `last(/app-server.example.local/app.process.running)=1` |

در عبارت‌ها، `app-server.example.local` باید دقیقاً با **Host name** هاست در Zabbix مطابقت داشته باشد؛ نام نمایشی الزاماً همان نام فنی نیست.

عملکرد Trigger:

- با توقف پردازش، مقدار آیتم `0` می‌شود و مشکل فعال می‌شود.
- با اجرای مجدد پردازش، مقدار آیتم `1` می‌شود و مشکل برطرف می‌شود.

## ۷. نمایش هشدار در داشبورد

۱. داشبورد موردنظر را باز کنید.
۲. وارد حالت ویرایش شوید.
۳. از **Add widget**، ویجت **Problems** را اضافه کنید.
۴. در صورت نیاز، هاست اپلیکیشن را در فیلتر ویجت انتخاب کنید.
۵. ویجت را اضافه و تغییرات داشبورد را ذخیره کنید.

پس از توقف پردازش، هشدار `Application Process Stopped` در ویجت مشکلات فعال نمایش داده می‌شود. برای مشاهده رویدادهای بازیابی‌شده، تنظیمات نمایش و فیلترهای ویجت یا تاریخچه رویدادها را بررسی کنید.

## ۸. روش آزمایش

> پردازش را فقط در محیط تست و پس از اطمینان از نداشتن اثر روی سرویس عملیاتی متوقف کنید.

۱. وضعیت پردازش را بررسی کنید:

```bash
pgrep -af sample_app
```

۲. در محیط تست، پردازش را متوقف کنید.

۳. پس از به‌روزرسانی آیتم، انتظار می‌رود `Application Process Status` برابر `0` شود.

۴. Trigger باید فعال شود و هشدار در داشبورد نمایش داده شود.

۵. برنامه را دوباره اجرا کنید.

۶. پس از به‌روزرسانی آیتم به مقدار `1`، Trigger باید به حالت برطرف‌شده برگردد.

## ۹. محدودیت‌ها

- این روش فقط وجود پردازش را بررسی می‌کند؛ اجرای پردازش به‌تنهایی تضمین نمی‌کند که اپلیکیشن از نظر عملکردی سالم است.
- خاموش‌شدن کامل سرور با توقف خود پردازش متفاوت است. برای تشخیص خاموش‌شدن سرور، مانیتورینگ دسترس‌پذیری هاست و Trigger جداگانه لازم است.
- این مستند درباره مانیتورینگ پردازش است و تنظیمات یا آیتم‌های دیگری را در Zabbix حذف نمی‌کند.

## ۱۰. نتیجه

با استفاده از Zabbix Agent، یک آیتم سفارشی برای وضعیت پردازش و یک Trigger برای توقف و بازیابی آن ایجاد می‌شود. هشدار توقف از طریق ویجت Problems در داشبورد قابل مشاهده است.
