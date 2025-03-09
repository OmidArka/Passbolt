Passbolt Installation Guide on Debian 11/12

This guide provides a step-by-step installation and configuration process for Passbolt on Debian 11/12. Passbolt is an open-source, self-hosted password manager designed for teams and organizations, offering high-level security and granular access control.
1. Prerequisites
1.1. Update the System

Before starting the installation, update the system packages:

sudo apt update && sudo apt upgrade -y

1.2. Install Required Packages

Passbolt requires the following dependencies:

    Nginx (as the web server)
    MariaDB (as the database)
    PHP 8.2 and required extensions
    GnuPG (for encryption)

Install them using:

sudo apt install -y nginx mariadb-server php php-cli php-fpm php-mysql php-ldap php-gd php-curl php-zip php-xml php-mbstring gnupg unzip

1.3. Configure MariaDB

Secure the database installation:

sudo mysql_secure_installation

Then, create a database and user for Passbolt:

sudo mysql -u root -p

Run the following SQL commands:

CREATE DATABASE passbolt;
CREATE USER 'passbolt_user'@'localhost' IDENTIFIED BY 'StrongPassword';
GRANT ALL PRIVILEGES ON passbolt.* TO 'passbolt_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;

2. Install and Configure Passbolt
2.1. Download and Install Passbolt Repository

sudo apt install -y curl gnupg
curl -fsSL https://download.passbolt.com/pub.key | gpg --dearmor | sudo tee /usr/share/keyrings/passbolt.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/passbolt.gpg] https://download.passbolt.com/ce/debian bookworm main" | sudo tee /etc/apt/sources.list.d/passbolt.list

Update the package list and install Passbolt:

sudo apt update
sudo apt install -y passbolt-ce-server

3. Passbolt Configuration
3.1. Generate GPG Key for Encryption

gpg --full-generate-key

    Select RSA (4096-bit) encryption.
    Set no expiration date for the key.
    Enter your name and email associated with the server.

To list the generated keys:

gpg --list-keys

Export and register the key in Passbolt:

gpg --export-secret-keys --armor YOUR_KEY_ID | sudo tee /etc/passbolt/gpg/serverkey.asc
sudo chown www-data:www-data /etc/passbolt/gpg/serverkey.asc
sudo chmod 600 /etc/passbolt/gpg/serverkey.asc

3.2. Configure Passbolt with Nginx

Edit the Nginx configuration file:

sudo nano /etc/nginx/sites-available/passbolt

Add the following configuration (replace with your domain):

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

Enable the configuration:

sudo ln -s /etc/nginx/sites-available/passbolt /etc/nginx/sites-enabled/
sudo systemctl restart nginx

3.3. Secure with SSL (Let’s Encrypt)

sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d passbolt.example.com

Enable auto-renewal:

sudo systemctl enable certbot.timer

4. Completing Passbolt Web Setup

Now, complete the installation via the web interface:

🔗 Open your browser and go to:

https://passbolt.example.com

🛠 Steps:

    Database Configuration – Enter MariaDB credentials.
    GPG Key Configuration – Use the previously generated key.
    Admin Account Setup – Create the first admin user.
    Login and Start Managing Passwords 🚀

5. Managing and Maintenance
5.1. Enable and Start Passbolt Services

sudo systemctl enable --now nginx php8.2-fpm mariadb

5.2. Check Service Status

sudo systemctl status nginx php8.2-fpm mariadb

5.3. Backup Database and GPG Key

mysqldump -u passbolt_user -p passbolt > /backup/passbolt-db.sql
cp /etc/passbolt/gpg/serverkey.asc /backup/passbolt-gpg.asc

Conclusion

You have successfully installed and configured Passbolt on Debian 11/12. This setup provides a secure password management solution for teams, ensuring strong encryption and controlled access.

📖 Official Documentation: help.passbolt.com
🚀 GitHub Repository: github.com/passbolt
