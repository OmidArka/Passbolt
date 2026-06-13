# 🔐 راهنمای کامل نصب و پیکربندی Passbolt روی Debian 13

> Passbolt یک مدیر رمز عبور متن‌باز و خودمیزبان است که با رمزنگاری end-to-end مبتنی بر OpenPGP، امنیت بالا و کنترل دسترسی پیشرفته‌ای را برای تیم‌ها و سازمان‌ها فراهم می‌کند.

---

## 📋 فهرست مطالب

1. [انتخاب سیستم‌عامل](#انتخاب-سیستم‌عامل)
2. [پیش‌نیازها](#پیش‌نیازها)
3. [آماده‌سازی سرور](#آماده‌سازی-سرور)
4. [نصب Passbolt](#نصب-passbolt)
5. [پیکربندی MariaDB](#پیکربندی-mariadb)
6. [پیکربندی Nginx و SSL](#پیکربندی-nginx-و-ssl)
7. [راه‌اندازی وب](#راه‌اندازی-وب)
8. [ساخت ادمین بدون SMTP](#ساخت-ادمین-بدون-smtp)
9. [ساخت کاربر جدید بدون SMTP](#ساخت-کاربر-جدید-بدون-smtp)
10. [مدیریت و نگهداری](#مدیریت-و-نگهداری)
11. [پشتیبان‌گیری](#پشتیبان‌گیری)
12. [عیب‌یابی](#عیب‌یابی)

---

## انتخاب سیستم‌عامل

| معیار | Debian 13 (Trixie) | Ubuntu 24.04 LTS |
|---|---|---|
| پایداری سرور | ✅ بالاتر | خوب |
| مصرف منابع | ✅ کمتر | بیشتر |
| چرخه پشتیبانی | ✅ طولانی‌تر | 5 ساله |
| پشتیبانی رسمی Passbolt | ✅ کامل | ✅ کامل |
| مناسب برای Production | ✅ ایده‌آل | خوب |

> **✅ توصیه:** Debian 13 (Trixie) — سبک‌تر، پایدارتر و بهینه‌تر برای محیط سرور.

---

## پیش‌نیازها

### سخت‌افزار (حداقل توصیه‌شده)

- **CPU:** 2 هسته
- **RAM:** 2GB
- **دیسک:** 20GB

### نرم‌افزار و شبکه

- سرور **Debian 13** تمیز (بدون سرویس از پیش نصب‌شده)
- **IP استاتیک** برای دسترسی به سرور
- سرویس **NTP** فعال برای جلوگیری از مشکلات احراز هویت GPG

> ⚠️ **مهم:** اسکریپت نصب Passbolt ممکن است داده‌های موجود را خراب کند. حتماً از سرور تمیز استفاده کنید.

---

## آماده‌سازی سرور

### به‌روزرسانی سیستم

```bash
apt update && apt upgrade -y
```

### نصب ابزارهای پایه

```bash
apt install -y curl gnupg2 ca-certificates procps
```

### اضافه کردن PATH برای ابزارهای سیستمی

> ⚠️ در Debian 13 ممکن است `sysctl`، `nginx`، `ufw` و سایر ابزارها در PATH نباشند. این دستور را **همیشه اول** اجرا کنید:

```bash
export PATH=$PATH:/sbin:/usr/sbin
```

برای دائمی کردن:

```bash
echo 'export PATH=$PATH:/sbin:/usr/sbin' >> ~/.bashrc
source ~/.bashrc
```

### تنظیم timezone

```bash
timedatectl set-timezone Asia/Tehran
```

### فعال‌سازی NTP

```bash
apt install -y systemd-timesyncd
timedatectl set-ntp true
timedatectl status
```

### تنظیم فایروال

```bash
apt install -y ufw
export PATH=$PATH:/sbin:/usr/sbin
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw enable
ufw status
```

---

## نصب Passbolt

### قدم ۱ — دانلود اسکریپت نصب repository

```bash
curl -LO https://download.passbolt.com/ce/installer/passbolt-repo-setup.ce.sh
curl -LO https://github.com/passbolt/passbolt-dep-scripts/releases/latest/download/passbolt-ce-SHA512SUM.txt
```

### قدم ۲ — تأیید checksum و اجرای اسکریپت

```bash
export PATH=$PATH:/sbin:/usr/sbin
sha512sum -c passbolt-ce-SHA512SUM.txt && bash ./passbolt-repo-setup.ce.sh
```

> ⚠️ اگر خطای `sysctl: command not found` دیدید، دستور `export PATH` بالا را اجرا کنید و دوباره امتحان کنید.

> ⚠️ اگر اسکریپت به هر دلیلی fail شد و فایل پاک شد، دوباره هر دو فایل را دانلود کنید.

### قدم ۳ — نصب پکیج Passbolt CE

```bash
apt update
apt install passbolt-ce-server
```

در طی نصب، یک **wizard تعاملی** باز می‌شود.

---

## پیکربندی MariaDB

در طی wizard اطلاعات زیر خواسته می‌شود:

```
نام کاربری ادمین MariaDB  →  root
پسورد ادمین MariaDB       →  (معمولاً خالی در نصب تازه)
نام کاربری Passbolt       →  passbolt_user
پسورد کاربر Passbolt      →  [یک پسورد قوی انتخاب کنید]
نام دیتابیس               →  passbolt
```

> 💡 این مقادیر را یادداشت کنید.

### اگر یوزر دیتابیس درست ساخته نشد

```bash
mysql -u root
```

```sql
DROP USER IF EXISTS 'passbolt_user'@'localhost';
CREATE USER 'passbolt_user'@'localhost' IDENTIFIED BY 'StrongPass123!';
CREATE DATABASE IF NOT EXISTS passbolt CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
GRANT ALL PRIVILEGES ON passbolt.* TO 'passbolt_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

تست اتصال:

```bash
mysql -u passbolt_user -p passbolt
```

---

## پیکربندی Nginx و SSL

### انتخاب SSL در wizard

- اگر دامنه واقعی دارید → **Let's Encrypt**
- اگر شبکه داخلی (Intranet) دارید → **Manual** و مسیر فایل‌های cert و key را وارد کنید

### ساخت گواهی self-signed (برای شبکه داخلی)

```bash
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 \
  -keyout /etc/ssl/private/passbolt.key \
  -out /etc/ssl/certs/passbolt.crt \
  -subj "/CN=passbolt.kidc.local" \
  -addext "subjectAltName=DNS:passbolt.kidc.local,IP:YOUR_SERVER_IP"
```

IP سرور را جای `YOUR_SERVER_IP` بگذارید.

### پیکربندی دستی Nginx

اگر nginx فقط روی 80 listen می‌کرد و SSL نداشت:

```bash
nano /etc/nginx/sites-enabled/nginx-passbolt.conf
```

محتوا را با این جایگزین کنید:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name passbolt.kidc.local;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name passbolt.kidc.local;

    ssl_certificate     /etc/ssl/certs/passbolt.crt;
    ssl_certificate_key /etc/ssl/private/passbolt.key;

    client_body_buffer_size     100K;
    client_header_buffer_size   1K;
    client_max_body_size        5M;
    client_body_timeout   10;
    client_header_timeout 10;
    keepalive_timeout     5 5;
    send_timeout          10;

    root /usr/share/php/passbolt/webroot;
    index index.php;

    error_log /var/log/nginx/passbolt-error.log info;
    access_log /var/log/nginx/passbolt-access.log;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        try_files                $uri =404;
        include                  fastcgi_params;
        fastcgi_pass             unix:/run/php/php8.4-fpm.sock;
        fastcgi_index            index.php;
        fastcgi_intercept_errors on;
        fastcgi_split_path_info  ^(.+\.php)(.+)$;
        fastcgi_param            SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param            SERVER_NAME $http_host;
        fastcgi_param PHP_VALUE  "upload_max_filesize=5M \n post_max_size=5M";
    }

    location ~ /\.ht {
        deny all;
    }
}
```

```bash
# حذف default nginx
rm /etc/nginx/sites-enabled/default

export PATH=$PATH:/sbin:/usr/sbin
nginx -t && systemctl restart nginx
```

### تنظیم fullBaseUrl

اگر با IP دسترسی دارید (نه دامنه):

```bash
nano /etc/passbolt/passbolt.php
```

تغییر دهید:

```php
'fullBaseUrl' => 'https://YOUR_SERVER_IP',
```

اگر با دامنه دسترسی دارید، روی کامپیوتر کاربران فایل hosts را ویرایش کنید:

**Windows:** `C:\Windows\System32\drivers\etc\hosts`

```
YOUR_SERVER_IP    passbolt.kidc.local
```

---

## راه‌اندازی وب

در مرورگر باز کنید:

```
https://YOUR_SERVER_IP
```

یا

```
https://passbolt.kidc.local
```

> اخطار SSL را قبول کنید (Advanced → Accept Risk)

### مراحل Wizard وب

| مرحله | توضیح |
|---|---|
| **Healthcheck** | بررسی سلامت محیط — مشکلات را رفع کنید |
| **Database** | اطلاعات MariaDB که در مرحله نصب ثبت کردید |
| **GPG Key** | گزینه Create را انتخاب کنید |
| **Email (SMTP)** | اگر SMTP ندارید اطلاعات زیر را بزنید و رد شوید |
| **Admin User** | ایمیل ادمین را وارد کنید |

### تنظیمات SMTP بدون سرور ایمیل

```
Sender name:    Passbolt
Sender email:   admin@kidc.local
SMTP host:      localhost
Use TLS:        No
Port:           25
Authentication: None
Username:       (خالی)
Password:       (خالی)
```

> ⚠️ دکمه Send Test Email را **نزنید** — fail می‌شود. فقط Next بزنید.

---

## ساخت ادمین بدون SMTP

بعد از اتمام wizard وب، لینک activation ایمیل نمی‌شود. از command line اجرا کنید:

```bash
su -s /bin/bash www-data -c "/usr/share/php/passbolt/bin/cake passbolt register_user -u admin@kidc.local -f Admin -l User -r admin"
```

لینکی شبیه این تولید می‌شود:

```
https://YOUR_SERVER_IP/setup/start/USER_ID/TOKEN
```

این لینک را در مرورگر باز کنید.

> ⚠️ **Extension الزامی است** — قبل از باز کردن لینک، extension Passbolt را در مرورگر نصب کنید:
> - Chrome/Edge: [chromewebstore.google.com](https://chromewebstore.google.com/detail/passbolt/didegimhafipceonhjepacocaffmoppf)
> - Firefox: [addons.mozilla.org](https://addons.mozilla.org/en-US/firefox/addon/passbolt/)

### اگر یوزر قبلاً ساخته شده بود

```bash
# پاک کردن یوزر قدیمی
mysql -u root passbolt -e "DELETE FROM authentication_tokens WHERE user_id = (SELECT id FROM users WHERE username = 'admin@kidc.local');"
mysql -u root passbolt -e "DELETE FROM users WHERE username = 'admin@kidc.local';"

# ساخت مجدد
su -s /bin/bash www-data -c "/usr/share/php/passbolt/bin/cake passbolt register_user -u admin@kidc.local -f Admin -l User -r admin"
```

### پسفریز (Passphrase)

در طی setup از شما یک passphrase خواسته می‌شود:

> 🔒 این تنها پسورد مهمی است که باید حفظ کنید. اگر فراموش شود **هیچ راه بازیابی وجود ندارد.**

ویژگی‌های یک passphrase خوب:
- حداقل 12 کاراکتر
- ترکیب حروف بزرگ، کوچک، عدد و کاراکتر خاص

بعد از تأیید، یک **Recovery Kit (فایل PDF)** دانلود کنید و در مکان امن نگهداری کنید.

---

## ساخت کاربر جدید بدون SMTP

### قدم ۱ — از پنل ادمین یوزر بسازید

1. وارد Passbolt شوید
2. بالا سمت راست → آیکون پروفایل → **Administration**
3. از منو **Users** را انتخاب کنید
4. روی **+** کلیک کنید و اطلاعات کاربر را وارد کنید

### قدم ۲ — لینک activation را از دیتابیس بگیرید

```bash
# گرفتن USER_ID
mysql -u root passbolt -e "SELECT id FROM users WHERE username = 'noroozi@kidc.local';"

# گرفتن TOKEN
mysql -u root passbolt -e "SELECT token FROM authentication_tokens WHERE user_id = (SELECT id FROM users WHERE username = 'noroozi@kidc.local') ORDER BY created DESC LIMIT 1;"
```

### قدم ۳ — لینک را به کاربر بدهید

```
https://YOUR_SERVER_IP/setup/start/USER_ID/TOKEN
```

> کاربر باید extension Passbolt را نصب کرده باشد و همان مراحل setup را طی کند.

---

## مدیریت و نگهداری

### فعال‌سازی و راه‌اندازی سرویس‌ها

```bash
systemctl enable --now nginx php8.4-fpm mariadb
```

### بررسی وضعیت سرویس‌ها

```bash
systemctl status nginx
systemctl status php8.4-fpm
systemctl status mariadb
```

### مشاهده لاگ‌ها

```bash
tail -f /var/log/nginx/passbolt-error.log
tail -f /var/log/nginx/passbolt-access.log
```

### آپدیت Passbolt

```bash
apt update && apt upgrade passbolt-ce-server
```

### بررسی سلامت کلی

```bash
su -s /bin/bash www-data -c "/usr/share/php/passbolt/bin/cake passbolt healthcheck"
```

---

## پشتیبان‌گیری

### پشتیبان از دیتابیس

```bash
mysqldump -u passbolt_user -p passbolt > /backup/passbolt-db-$(date +%F).sql
```

### پشتیبان از کلید GPG

```bash
cp /etc/passbolt/gpg/serverkey.asc /backup/passbolt-gpg-$(date +%F).asc
```

### پشتیبان از فایل پیکربندی

```bash
cp /etc/passbolt/passbolt.php /backup/passbolt-config-$(date +%F).php
```

### اتوماتیک‌سازی با cron

```bash
crontab -e
```

اضافه کنید:

```cron
0 2 * * * mysqldump -u passbolt_user -pSTRONG_PASSWORD passbolt > /backup/passbolt-db-$(date +\%F).sql
0 2 * * * cp /etc/passbolt/gpg/serverkey.asc /backup/passbolt-gpg-$(date +\%F).asc
```

---

## عیب‌یابی

### خطای `command not found` برای nginx، ufw، sysctl

```bash
export PATH=$PATH:/sbin:/usr/sbin
```

### خطای IPv6 در اسکریپت نصب

```bash
export PATH=$PATH:/sbin:/usr/sbin
bash ./passbolt-repo-setup.ce.sh
```

### خطای `Unable to locate package passbolt-ce-server`

یعنی اسکریپت repo setup اجرا نشده:

```bash
curl -LO https://download.passbolt.com/ce/installer/passbolt-repo-setup.ce.sh
curl -LO https://github.com/passbolt/passbolt-dep-scripts/releases/latest/download/passbolt-ce-SHA512SUM.txt
export PATH=$PATH:/sbin:/usr/sbin
sha512sum -c passbolt-ce-SHA512SUM.txt && bash ./passbolt-repo-setup.ce.sh
apt update
apt install passbolt-ce-server
```

### خطای SSL در مرورگر

چون self-signed certificate است — در مرورگر روی **Advanced** و سپس **Accept Risk** کلیک کنید.

### صفحه سفید در مرورگر

```bash
# حذف default nginx
rm /etc/nginx/sites-enabled/default
export PATH=$PATH:/sbin:/usr/sbin
nginx -t && systemctl restart nginx
```

### خطای اتصال به دیتابیس در wizard وب

```bash
mysql -u root
```

```sql
DROP USER IF EXISTS 'passbolt_user'@'localhost';
CREATE USER 'passbolt_user'@'localhost' IDENTIFIED BY 'StrongPass123!';
GRANT ALL PRIVILEGES ON passbolt.* TO 'passbolt_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### لینک activation کار نمی‌کند (Not Found)

یعنی توکن منقضی شده:

```bash
mysql -u root passbolt -e "DELETE FROM authentication_tokens WHERE user_id = (SELECT id FROM users WHERE username = 'USER@kidc.local');"
mysql -u root passbolt -e "DELETE FROM users WHERE username = 'USER@kidc.local';"
su -s /bin/bash www-data -c "/usr/share/php/passbolt/bin/cake passbolt register_user -u USER@kidc.local -f Name -l Lastname -r admin"
```

### لیست کاربران

```bash
su -s /bin/bash www-data -c "/usr/share/php/passbolt/bin/cake passbolt users_index"
```

---

## نکات امنیتی

- **آپدیت منظم** Debian و Passbolt را فراموش نکنید
- دسترسی SSH را با **کلید عمومی** انجام دهید، نه پسورد
- ورود مستقیم با **root را غیرفعال** کنید:

```bash
nano /etc/ssh/sshd_config
# تغییر دهید:
PermitRootLogin no
```

- فایروال را همیشه فعال نگه دارید
- کلید GPG و Recovery Kit را در مکان **امن و جدا** از سرور نگهداری کنید

---

*آخرین به‌روزرسانی: ۲۰۲۶ | مبتنی بر تجربه عملی نصب روی Debian 13 (Trixie) + مستندات رسمی Passbolt CE v5.12*
