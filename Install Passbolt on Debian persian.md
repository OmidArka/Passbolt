راهنمای نصب Passbolt روی Debian 11/12

این راهنما نحوه نصب و پیکربندی Passbolt را روی Debian 11/12 به‌صورت گام‌به‌گام توضیح می‌دهد. Passbolt یک مدیر رمز عبور متن‌باز و خودمیزبان است که امنیت بالا و کنترل دسترسی پیشرفته‌ای را برای تیم‌ها و سازمان‌ها فراهم می‌کند.
۱. پیش‌نیازها
۱.۱. به‌روزرسانی سیستم

قبل از نصب، مخازن و بسته‌های سیستم را به‌روز کنید:

sudo apt update && sudo apt upgrade -y

۱.۲. نصب نرم‌افزارهای موردنیاز

Passbolt به بسته‌های زیر نیاز دارد:

    Nginx (برای وب‌سرور)
    MariaDB (به‌عنوان دیتابیس)
    PHP 8.2 و ماژول‌های موردنیاز
    GnuPG (برای رمزنگاری)

نصب این بسته‌ها را اجرا کنید:

sudo apt install -y nginx mariadb-server php php-cli php-fpm php-mysql php-ldap php-gd php-curl php-zip php-xml php-mbstring gnupg unzip

۱.۳. تنظیم MariaDB

پس از نصب، امنیت دیتابیس را افزایش دهید:

sudo mysql_secure_installation

سپس یک دیتابیس و کاربر برای Passbolt ایجاد کنید:

sudo mysql -u root -p

و دستورات زیر را اجرا کنید:

CREATE DATABASE passbolt;
CREATE USER 'passbolt_user'@'localhost' IDENTIFIED BY 'StrongPassword';
GRANT ALL PRIVILEGES ON passbolt.* TO 'passbolt_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;

۲. نصب و راه‌اندازی Passbolt
۲.۱. دانلود و نصب مخزن رسمی Passbolt

sudo apt install -y curl gnupg
curl -fsSL https://download.passbolt.com/pub.key | gpg --dearmor | sudo tee /usr/share/keyrings/passbolt.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/passbolt.gpg] https://download.passbolt.com/ce/debian bookworm main" | sudo tee /etc/apt/sources.list.d/passbolt.list

به‌روزرسانی لیست مخازن و نصب Passbolt:

sudo apt update
sudo apt install -y passbolt-ce-server

۳. پیکربندی Passbolt
۳.۱. ایجاد کلید GPG برای رمزنگاری

gpg --full-generate-key

    گزینه RSA (4096-bit) را انتخاب کنید.
    کلید را بدون تاریخ انقضا تنظیم کنید.
    نام و ایمیل مرتبط با سرور را وارد کنید.

دریافت کلید GPG:

gpg --list-keys

کپی کلید و ثبت آن در Passbolt:

gpg --export-secret-keys --armor YOUR_KEY_ID | sudo tee /etc/passbolt/gpg/serverkey.asc
sudo chown www-data:www-data /etc/passbolt/gpg/serverkey.asc
sudo chmod 600 /etc/passbolt/gpg/serverkey.asc

۳.۲. پیکربندی Passbolt با Nginx

ویرایش فایل تنظیمات:

sudo nano /etc/nginx/sites-available/passbolt

و اضافه کردن تنظیمات زیر (دامنه را تغییر دهید):

server {
    listen 80;
    server_name passbolt.example.com;
    root /var/www/passbolt/webroot;

    location / {
        try_files $uri /index.php$is_args$args;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}

فعال‌سازی تنظیمات:

sudo ln -s /etc/nginx/sites-available/passbolt /etc/nginx/sites-enabled/
sudo systemctl restart nginx

۳.۳. تنظیم گواهی SSL (Let’s Encrypt)

sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d passbolt.example.com

فعال‌سازی تمدید خودکار گواهی:

sudo systemctl enable certbot.timer

۴. اجرای نصب وب Passbolt

اکنون می‌توانید نصب Passbolt را از طریق مرورگر انجام دهید:

🔗 باز کردن مرورگر و رفتن به:

https://passbolt.example.com

🛠 مراحل:

    تأیید اتصال به دیتابیس
    وارد کردن کلید GPG
    ایجاد حساب مدیر
    ورود به Passbolt و مدیریت رمزهای عبور

۵. مدیریت و نگهداری
۵.۱. راه‌اندازی سرویس Passbolt

sudo systemctl enable --now nginx php8.2-fpm mariadb

۵.۲. بررسی وضعیت سرویس‌ها

sudo systemctl status nginx php8.2-fpm mariadb

۵.۳. تهیه نسخه پشتیبان از پایگاه داده و کلید GPG

mysqldump -u passbolt_user -p passbolt > /backup/passbolt-db.sql
cp /etc/passbolt/gpg/serverkey.asc /backup/passbolt-gpg.asc

نتیجه‌گیری

با موفقیت Passbolt را روی Debian 11/12 نصب و پیکربندی کردید. این سیستم به تیم‌ها کمک می‌کند رمزهای عبور را با امنیت بالا مدیریت کنند و دسترسی‌ها را کنترل نمایند.

📖 مستندات رسمی: help.passbolt.com
🚀 مخزن گیت‌هاب: github.com/passbolt
