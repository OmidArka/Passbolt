# 🔐 راهنمای کامل نصب و پیکربندی Passbolt روی Debian 13 (Trixie)

> Passbolt یک مدیر رمز عبور متن‌باز و خودمیزبان است که با رمزنگاری end-to-end مبتنی بر OpenPGP، امنیت بالا و کنترل دسترسی پیشرفته‌ای را برای تیم‌ها و سازمان‌ها فراهم می‌کند.

---

## 📋 فهرست مطالب

1. [انتخاب سیستم‌عامل](#انتخاب-سیستم‌عامل)
2. [پیش‌نیازها](#پیش‌نیازها)
3. [آماده‌سازی سرور](#آماده‌سازی-سرور)
4. [نصب Passbolt](#نصب-passbolt)
5. [پیکربندی MariaDB](#پیکربندی-mariadb)
6. [پیکربندی Nginx و SSL](#پیکربندی-nginx-و-ssl)
7. [پیکربندی کلید GPG](#پیکربندی-کلید-gpg)
8. [راه‌اندازی وب](#راه‌اندازی-وب)
9. [مدیریت و نگهداری](#مدیریت-و-نگهداری)
10. [پشتیبان‌گیری](#پشتیبان‌گیری)
11. [عیب‌یابی](#عیب‌یابی)

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
- **دامنه یا IP استاتیک** برای دسترسی به سرور
- سرور **SMTP** فعال برای ارسال ایمیل
- سرویس **NTP** فعال برای جلوگیری از مشکلات احراز هویت GPG

> ⚠️ **مهم:** اسکریپت نصب Passbolt ممکن است داده‌های موجود را خراب کند. حتماً از سرور تمیز استفاده کنید.

---

## آماده‌سازی سرور

### به‌روزرسانی سیستم

```bash
sudo apt update && sudo apt upgrade -y
```

### نصب ابزارهای پایه

```bash
sudo apt install -y curl gnupg2 ca-certificates ufw
```

### تنظیم timezone

```bash
sudo timedatectl set-timezone Asia/Tehran
```

### فعال‌سازی NTP

```bash
sudo apt install -y systemd-timesyncd
sudo timedatectl set-ntp true
timedatectl status
```

### تنظیم فایروال

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status
```

---

## نصب Passbolt

### قدم ۱ — دانلود اسکریپت نصب repository

```bash
curl -LO https://download.passbolt.com/ce/installer/passbolt-repo-setup.ce.sh
```

### قدم ۲ — دانلود فایل SHA512 برای تأیید اصالت

```bash
curl -LO https://github.com/passbolt/passbolt-dep-scripts/releases/latest/download/passbolt-ce-SHA512SUM.txt
```

### قدم ۳ — تأیید checksum و اجرای اسکریپت

```bash
sha512sum -c passbolt-ce-SHA512SUM.txt && sudo bash ./passbolt-repo-setup.ce.sh \
  || echo "Bad checksum. Aborting" && rm -f passbolt-repo-setup.ce.sh
```

> 🔒 **نکته امنیتی:** همیشه قبل از اجرای اسکریپت از منابع خارجی، checksum را تأیید کنید.

### قدم ۴ — نصب پکیج Passbolt CE

```bash
sudo apt install passbolt-ce-server
```

در طی نصب، یک **wizard تعاملی** باز می‌شود که مراحل بعدی را راهنمایی می‌کند.

---

## پیکربندی MariaDB

پکیج Passbolt به‌طور خودکار `mariadb-server` را نصب می‌کند. در طی wizard اطلاعات زیر خواسته می‌شود:

```
نام کاربری ادمین MariaDB  →  root
پسورد ادمین MariaDB       →  (معمولاً خالی در نصب تازه)
نام کاربری Passbolt       →  passbolt_user
پسورد کاربر Passbolt      →  [یک پسورد قوی انتخاب کنید]
نام دیتابیس               →  passbolt
```

> 💡 این مقادیر را یادداشت کنید — در مرحله پیکربندی وب دوباره لازم می‌شوند.

### تنظیم دستی دیتابیس (اختیاری)

اگر ترجیح می‌دهید دیتابیس را دستی بسازید:

```bash
sudo mysql_secure_installation
sudo mysql -u root -p
```

```sql
CREATE DATABASE passbolt CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'passbolt_user'@'localhost' IDENTIFIED BY 'StrongPassword123!';
GRANT ALL PRIVILEGES ON passbolt.* TO 'passbolt_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

---

## پیکربندی Nginx و SSL

در طی wizard دو گزینه دارید:

### گزینه A — Let's Encrypt (توصیه‌شده برای دامنه واقعی)

گزینه **Auto** را در wizard انتخاب کنید. نیاز به دامنه واقعی و باز بودن پورت 80/443 دارید.

```bash
# اگر بعداً نیاز به تمدید دستی داشتید:
sudo certbot renew --dry-run
sudo systemctl enable certbot.timer
```

### گزینه B — گواهی SSL دستی (برای شبکه داخلی / Intranet)

گزینه **Manual** را انتخاب کنید، سپس:

```bash
# نصب Certbot
sudo apt install -y certbot python3-certbot-nginx

# دریافت گواهی
sudo certbot --nginx -d passbolt.example.com
```

یا اگر گواهی از CA داخلی دارید، مسیر فایل‌های `.crt` و `.key` را مستقیم وارد کنید.

### پیکربندی دستی Nginx (در صورت نیاز)

```nginx
server {
    listen 80;
    server_name passbolt.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name passbolt.example.com;

    ssl_certificate     /etc/letsencrypt/live/passbolt.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/passbolt.example.com/privkey.pem;

    root /var/www/passbolt/webroot;
    index index.php;

    location / {
        try_files $uri /index.php$is_args$args;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/passbolt /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

---

## پیکربندی کلید GPG

> ⚠️ از آنجا که GnuPG 2.2.0+ به‌طور پیش‌فرض از ECC استفاده می‌کند که ممکن است در همه محیط‌ها به‌درستی کار نکند، استفاده از **RSA 3072-bit** توصیه می‌شود.

### تولید کلید GPG (روش batch — توصیه‌شده)

```bash
gpg --batch --no-tty --gen-key <<EOF
  Key-Type: RSA
  Key-Length: 3072
  Key-Usage: sign,cert
  Subkey-Type: RSA
  Subkey-Usage: encrypt
  Subkey-Length: 3072
  Name-Real: Passbolt Server
  Name-Email: admin@yourdomain.com
  Expire-Date: 0
  %no-protection
EOF
```

### مشاهده و صادرکردن کلید

```bash
# مشاهده کلیدها
gpg --list-keys

# صادرکردن کلید خصوصی و کپی برای Passbolt
gpg --export-secret-keys --armor YOUR_KEY_ID | sudo tee /etc/passbolt/gpg/serverkey.asc

# تنظیم مجوزها
sudo chown www-data:www-data /etc/passbolt/gpg/serverkey.asc
sudo chmod 600 /etc/passbolt/gpg/serverkey.asc
```

> 💡 اگر wizard نصب این مرحله را خودکار انجام داد، نیازی به اجرای دستی نیست.

---

## راه‌اندازی وب

بعد از اتمام نصب، مرورگر را باز کنید:

```
https://passbolt.example.com
```

### مراحل Wizard وب

| مرحله | توضیح |
|---|---|
| **Healthcheck** | بررسی سلامت محیط — مشکلات را رفع کنید، سپس ادامه دهید |
| **Database** | اطلاعات MariaDB که در مرحله نصب ثبت کردید |
| **GPG Key** | کلید سرور را generate یا import کنید |
| **Email** | تنظیمات SMTP برای ارسال ایمیل |
| **Admin User** | ایجاد اولین حساب مدیر سیستم |

پس از اتمام، می‌توانید وارد Passbolt شوید و شروع به مدیریت رمزهای عبور کنید.

---

## مدیریت و نگهداری

### فعال‌سازی و راه‌اندازی سرویس‌ها

```bash
sudo systemctl enable --now nginx php8.2-fpm mariadb
```

### بررسی وضعیت سرویس‌ها

```bash
sudo systemctl status nginx
sudo systemctl status php8.2-fpm
sudo systemctl status mariadb
```

### مشاهده log ها

```bash
# لاگ Nginx
sudo tail -f /var/log/nginx/error.log
sudo tail -f /var/log/nginx/access.log

# لاگ Passbolt
sudo tail -f /var/www/passbolt/logs/error.log
```

### آپدیت Passbolt

```bash
sudo apt update && sudo apt upgrade passbolt-ce-server
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
sudo crontab -e
```

اضافه کنید:

```cron
0 2 * * * mysqldump -u passbolt_user -pStrongPassword123! passbolt > /backup/passbolt-db-$(date +\%F).sql
0 2 * * * cp /etc/passbolt/gpg/serverkey.asc /backup/passbolt-gpg-$(date +\%F).asc
```

---

## عیب‌یابی

### خطای GPG در احراز هویت

```bash
# بررسی سرویس NTP
timedatectl status
sudo systemctl restart systemd-timesyncd
```

### خطای اتصال به دیتابیس

```bash
# تست اتصال
mysql -u passbolt_user -p passbolt -e "SELECT 1;"

# بررسی وضعیت MariaDB
sudo systemctl status mariadb
sudo journalctl -u mariadb --no-pager -n 50
```

### خطای permission در فایل‌ها

```bash
sudo chown -R www-data:www-data /var/www/passbolt
sudo chmod -R 755 /var/www/passbolt
sudo chmod 600 /etc/passbolt/gpg/serverkey.asc
```

### بررسی سلامت کلی Passbolt

```bash
sudo -u www-data /var/www/passbolt/bin/cake passbolt healthcheck
```

---

## نکات امنیتی

- **آپدیت منظم** Debian و Passbolt را فراموش نکنید
- دسترسی SSH را با **کلید عمومی** انجام دهید، نه پسورد
- ورود مستقیم با **root را غیرفعال** کنید (`PermitRootLogin no` در `/etc/ssh/sshd_config`)
- فایروال را همیشه فعال نگه دارید
- کلید GPG و پیکربندی را در مکان **امن و جدا** از سرور نگهداری کنید
- از **HTTPS اجباری** مطمئن شوید (redirect از 80 به 443)
mysql -u root passbolt -e "SELECT token FROM authentication_tokens WHERE user_id = (SELECT id FROM users WHERE username = '......@.....local') ORDER BY created DESC LIMIT 1;"
mysql -u root passbolt -e "SELECT id FROM users WHERE username = '.....@.....local';"
---

*آخرین به‌روزرسانی: ۲۰۲۵ | مبتنی بر مستندات رسمی Passbolt CE*
