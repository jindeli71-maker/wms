# WMS - Website Monitoring System — Installation Guide

Core PHP (no framework) + MySQL. Works on XAMPP (Windows, local dev) and cPanel shared hosting.

## 1. Upload files

Upload the whole `wms/` folder to your server (or `htdocs/wms` for XAMPP).

## 2. Create the database

1. In phpMyAdmin (or cPanel → MySQL Databases), create a database, e.g. `wms_db`, and a
   MySQL user with full privileges on it.
2. Import `database.sql` (phpMyAdmin → Import, or `mysql -u user -p wms_db < database.sql`).

## 3. Configure the app

Edit `config/config.php`:

```php
define('DB_HOST', 'localhost');
define('DB_NAME', 'wms_db');
define('DB_USER', 'your_db_user');
define('DB_PASS', 'your_db_password');

define('TELEGRAM_BOT_TOKEN', 'your_bot_token_here');
define('APP_URL', 'https://yourdomain.com/wms');
define('CRON_SECRET', 'a-long-random-string');

// SMTP — required for Forgot Password emails (create an email in cPanel first)
define('SMTP_HOST', 'mail.yourdomain.com');
define('SMTP_PORT', 587);
define('SMTP_ENCRYPTION', 'tls');
define('SMTP_USER', 'noreply@yourdomain.com');
define('SMTP_PASS', 'your_email_password');
define('MAIL_FROM_EMAIL', 'noreply@yourdomain.com');
define('MAIL_FROM_NAME', 'WMS Monitor');
```

## 3b. Password reset email (SMTP)

Forgot Password sends a real email via SMTP. On **cPanel**:

1. Go to **Email Accounts** → create e.g. `noreply@yourdomain.com`.
2. Put that account’s host / user / password into the `SMTP_*` settings above.
3. Typical cPanel values:
   - Host: `mail.yourdomain.com` (or the hostname shown in “Connect Devices”)
   - Port: `587` with `tls`, or `465` with `ssl`
4. Set `APP_URL` to your live site URL so the reset link in the email is correct.

On local XAMPP, use a real SMTP provider (cPanel mailbox, Gmail App Password, etc.) — PHP `mail()` alone usually will not deliver.

## 4. Create your admin account

Visit `https://yourdomain.com/wms/install.php` in your browser and fill in the form to
create your first admin username/password.

**Important:** delete `install.php` immediately afterwards — it only allows creating an
account when the `admins` table is empty, but removing the file is the safest practice.

## 5. Set up the Telegram Bot (BotFather)

1. Open Telegram and search for **@BotFather**.
2. Send `/newbot` and follow the prompts (choose a name and a username ending in `bot`).
3. BotFather gives you a token like `123456789:AAExampleTokenTextGoesHere`. Paste it into
   `TELEGRAM_BOT_TOKEN` in `config/config.php`.
4. To find a **chat_id** for yourself or a group:
   - Message **@userinfobot** on Telegram — it replies with your numeric chat_id.
   - For a group: add your bot to the group, send any message in the group, then visit
     `https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates` in a browser and look for
     `"chat":{"id": ...}` in the JSON response.
5. In the WMS admin panel, go to **Alert Contacts** and add the contact's name + chat_id.
   Use "Send Test" to confirm the bot can message them (the contact must have started a
   chat with the bot first, i.e. pressed **Start**).

## 6. Set up the Cron Job (runs every minute)

### cPanel shared hosting
Go to **cPanel → Cron Jobs** and add a job that runs every minute:

```
* * * * * /usr/local/bin/php /home/yourusername/public_html/wms/cron/check_monitors.php your-cron-secret >> /dev/null 2>&1
```

(Use the PHP path shown by `which php` in cPanel's Terminal if different, e.g. `/usr/bin/php`.)

### Linux server crontab
```
* * * * * php /var/www/wms/cron/check_monitors.php your-cron-secret >> /dev/null 2>&1
```

### Windows / XAMPP (local testing)
Use **Task Scheduler** to run every minute:
```
"C:\xampp\php\php.exe" "C:\xampp\htdocs\wms\cron\check_monitors.php" your-cron-secret
```
Or trigger it manually in a browser during development:
```
http://localhost/wms/cron/check_monitors.php?key=your-cron-secret
```

The `key` / CLI argument must match `CRON_SECRET` in `config/config.php`, or the script
refuses to run when called over HTTP.

## 7. Log in

Go to `https://yourdomain.com/wms/login.php` and log in with the admin account you created
in step 4. Add your first monitor from the dashboard.

## Notes

- `check_interval` is the minimum seconds between checks per monitor; the cron itself should
  still run every minute so it can pick up monitors as soon as they become due.
- Response times are in milliseconds; the "ping" monitor type uses a TCP socket check
  (`fsockopen`) since ICMP ping is normally blocked on shared hosting.
- Uptime % is calculated from `monitor_logs` over the last 24h / 7d / 30d.
