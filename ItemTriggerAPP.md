# مانیتورینگ پردازش Policy Enforcement در Zabbix

## ۱. هدف

هدف، مانیتورینگ وضعیت اجرای برنامه `pe_performance` روی سرور PE و نمایش هشدار در داشبورد Zabbix هنگام توقف برنامه است. پس از اجرای مجدد برنامه، هشدار باید به حالت برطرف‌شده برگردد.

## ۲. مشخصات محیط

| مؤلفه | مقدار |
|---|---|
| Zabbix Server | `zabbix-01` |
| IP سرور Zabbix | `10.10.10.21` |
| نام فنی هاست PE | `pe3.tbl.local` |
| نام نمایشی هاست | `PE-Dev` |
| IP سرور PE | `10.10.10.149` |
| نام پردازش | `pe_performance` |
| نام آیتم | `PE Process Status` |
| کلید آیتم | `pe.process.running` |
| نام Trigger | `PE Process Stopped` |
| فاصله جمع‌آوری داده | `30s` |

## ۳. تعریف UserParameter در Zabbix Agent

برنامه PE به‌صورت پردازش لینوکسی اجرا می‌شود و سرویس systemd نیست؛ بنابراین وضعیت آن با `pgrep` بررسی می‌شود.

فایل زیر روی سرور PE ایجاد شد:

`/etc/zabbix/zabbix_agentd.d/pe_process.conf`

محتوای فایل:

```bash
UserParameter=pe.process.running,pgrep -x pe_performance >/dev/null 2>&1 && echo 1 || echo 0
```

سپس Agent راه‌اندازی مجدد شد:

```bash
sudo systemctl restart zabbix-agent
```

## ۴. بررسی عملکرد Agent

از روی سرور Zabbix دستور زیر اجرا شد:

```bash
zabbix_get -s 10.10.10.149 -k pe.process.running
```

خروجی اولیه `1` بود؛ یعنی پردازش در آن لحظه اجرا می‌شد و Agent وضعیت آن را گزارش می‌کرد.

## ۵. ساخت Item در رابط Zabbix

مسیر:

`Data collection → Hosts → PE-Dev → Items → Create item`

تنظیمات:

| فیلد | مقدار |
|---|---|
| Name | `PE Process Status` |
| Type | `Zabbix agent` |
| Key | `pe.process.running` |
| Type of information | `Numeric (unsigned)` |
| Update interval | `30s` |
| History | `7d` |

این آیتم وضعیت پردازش را به‌صورت دوره‌ای دریافت می‌کند.

## ۶. ساخت Trigger برای توقف برنامه

مسیر:

`Data collection → Hosts → PE-Dev → Triggers → Create trigger`

تنظیمات:

| فیلد | مقدار |
|---|---|
| Name | `PE Process Stopped` |
| Severity | `High` |
| Problem expression | `last(/pe3.tbl.local/pe.process.running)=0` |
| Recovery mode | `Recovery expression` |
| Recovery expression | `last(/pe3.tbl.local/pe.process.running)=1` |

عملکرد Trigger:

- با توقف پردازش، مقدار آیتم `0` می‌شود و مشکل فعال می‌شود.
- با اجرای مجدد پردازش، مقدار آیتم `1` می‌شود و مشکل برطرف می‌شود.

## ۷. نمایش هشدار در داشبورد

ویجت `Problems` در داشبورد Zabbix برای نمایش مشکلات فعال استفاده شد. پس از توقف پردازش، هشدار `PE Process Stopped` در داشبورد نمایش داده شد.

برای دیدن تاریخچه بازیابی، می‌توان از تاریخچه رویدادها یا تنظیمات نمایش ویجت استفاده کرد؛ نمایش رخدادهای برطرف‌شده به تنظیمات و فیلترهای ویجت بستگی دارد.

## ۸. روش آزمایش

> توقف برنامه را فقط در محیط تست و با اطمینان از اینکه به ترافیک عملیاتی آسیب نمی‌زند انجام دهید.

۱. وضعیت پردازش را بررسی کنید:

```bash
pgrep -af pe_performance
```

۲. در محیط تست، پردازش را متوقف کنید.

۳. پس از به‌روزرسانی آیتم، انتظار می‌رود `PE Process Status` برابر `0` شود.

۴. Trigger باید فعال شود و هشدار در داشبورد نمایش داده شود.

۵. برنامه را دوباره اجرا کنید.

۶. پس از به‌روزرسانی آیتم به مقدار `1`، Trigger باید به حالت برطرف‌شده برگردد.

## ۹. محدودیت‌ها

- این روش فقط وجود پردازش `pe_performance` را بررسی می‌کند؛ اجرای پردازش به‌تنهایی تضمین نمی‌کند که PE واقعاً ترافیک را پردازش می‌کند.
- خاموش‌شدن کامل سرور PE با توقف خود پردازش متفاوت است. برای تشخیص خاموش‌شدن سرور، مانیتورینگ دسترس‌پذیری هاست و Trigger جداگانه لازم است.
- این مستند روی مانیتورینگ پردازش تمرکز دارد و آیتم‌ها یا تنظیمات SNMP را از خود Zabbix حذف نمی‌کند.

## ۱۰. نتیجه

مانیتورینگ وضعیت پردازش PE با استفاده از Zabbix Agent و یک آیتم سفارشی راه‌اندازی شد. Trigger توقف پردازش ایجاد شد و هشدار آن در داشبورد نمایش داده شد.
