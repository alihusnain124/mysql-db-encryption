# 🔐 MySQL Database Encryption Guide (TDE - Transparent Data Encryption)

> **Complete step-by-step guide** to encrypt your entire MySQL database using TDE (Transparent Data Encryption) for both local and production environments.

---

## 📌 Table of Contents

1. [What is TDE?](#what-is-tde)
2. [TDE vs Column-Level Encryption](#tde-vs-column-level-encryption)
3. [Does Data Look Encrypted in MySQL?](#does-data-look-encrypted-in-mysql)
4. [Prerequisites](#prerequisites)
5. [Step 1 — Login to MySQL Correctly](#step-1--login-to-mysql-correctly)
6. [Step 2 — Check MySQL Version](#step-2--check-mysql-version)
7. [Step 3 — Edit MySQL Config File](#step-3--edit-mysql-config-file)
8. [Step 4 — Create Keyring Directory](#step-4--create-keyring-directory)
9. [Step 5 — Restart MySQL](#step-5--restart-mysql)
10. [Step 6 — Verify Keyring Plugin](#step-6--verify-keyring-plugin)
11. [Step 7 — Encrypt Your Database](#step-7--encrypt-your-database)
12. [Step 8 — Verify Encryption](#step-8--verify-encryption)
13. [Step 9 — Production Setup with Docker (docker-compose.yml)](#step-9--production-setup-with-docker-composeyml)
14. [Step 10 — Backup Your Keyring File](#step-10--backup-your-keyring-file)
15. [Common Mistakes](#common-mistakes)
16. [Security Layering Strategy](#security-layering-strategy)

---

## 🔍 What is TDE?

**Transparent Data Encryption (TDE)** encrypts your MySQL data **at the disk level**. It means:

- ✅ Your `.ibd` files on disk are **fully encrypted**
- ✅ Your application code does **NOT need to change**
- ✅ You and your app see **normal readable data** when querying
- ✅ If someone **steals your hard drive**, data is completely **unreadable**

> The word **"Transparent"** means encryption/decryption happens automatically in the background — you don't notice it at all.

---

## ⚖️ TDE vs Column-Level Encryption

| Feature | TDE (Whole Database) | Column-Level AES |
|---|---|---|
| **Scope** | Entire database | Specific columns only |
| **Data visible in MySQL?** | ✅ Yes, looks normal | ❌ No, looks garbled/binary |
| **Code changes needed?** | ❌ None | ✅ Yes, app must encrypt/decrypt |
| **Performance impact** | ~3–5% | Higher per query |
| **Use case** | Encrypt everything | Extra sensitive fields (e.g. passwords) |

> **Conclusion:** For encrypting the whole database → **TDE is the right and complete solution.**

---

## 👁️ Does Data Look Encrypted in MySQL?

| Method | What you see when you open MySQL Workbench / phpMyAdmin |
|---|---|
| **TDE** | ✅ Data looks **completely normal and readable** |
| **Column AES_ENCRYPT()** | ❌ Data shows as **binary/garbled** |
| **Disk Encryption (LUKS)** | ✅ Data looks **normal inside MySQL** |

> **TDE encrypts the physical files on disk only.** Inside MySQL everything looks and works exactly as before.

---

## ✅ Prerequisites

- MySQL **8.0 or higher**
- Linux-based system (Ubuntu/Debian recommended)
- `sudo` / root access
- MySQL root password

---

## Step 1 — Login to MySQL Correctly

> ⚠️ **Important:** SQL commands must be run **inside MySQL**, not in the Linux terminal (bash).

```bash
# Run this in your Linux terminal
mysql -u root -p

# Enter your MySQL root password when prompted
# You will see the MySQL prompt:
# mysql>
```

> You are now inside MySQL and can run SQL commands.

---

## Step 2 — Check MySQL Version

Run this **inside MySQL** (`mysql>` prompt):

```sql
SELECT VERSION();
```

**Expected output:**
```
+-----------+
| VERSION() |
+-----------+
| 8.0.xx    |
+-----------+
```

> You need **8.0+** for full TDE support. If you are on 5.7, upgrade first.

---

## Step 3 — Edit MySQL Config File

Exit MySQL first, then find the correct config file:

```bash
# Exit MySQL
exit

# Step 1 — Find your config file location
sudo find /etc -name "my.cnf" 2>/dev/null
```

You will see two results:
```
/etc/mysql/my.cnf          ← main file (usually just has !includedir)
/etc/alternatives/my.cnf   ← just a shortcut, ignore this
```

```bash
# Step 2 — Check what is inside /etc/mysql/my.cnf
sudo nano /etc/mysql/my.cnf
```

> ⚠️ **Important:** If `/etc/mysql/my.cnf` only has comments and `!includedir /etc/mysql/conf.d/` with NO `[mysqld]` section — this means MySQL reads config from the `conf.d` folder instead. This is common on Ubuntu.

```bash
# Step 3 — Check conf.d folder
ls /etc/mysql/conf.d/
# You will see: mysql.cnf  mysqldump.cnf
```

```bash
# Step 4 — Open mysql.cnf (this is the correct file to edit)
sudo nano /etc/mysql/conf.d/mysql.cnf
```

> ⚠️ **Important:** You will see `[mysql]` at the top — do NOT remove it. Add a NEW `[mysqld]` section BELOW the existing `[mysql]` section.

Your file should look like this after editing:

```ini
[mysql]
# whatever was already here — DO NOT touch this section

[mysqld]
# ─── Keyring / Encryption Settings ──────────────────────
early-plugin-load=keyring_file.so
keyring_file_data=/var/lib/mysql-keyring/keyring

# Encrypt all new tables by default
default_table_encryption=ON

# Encrypt binary logs
binlog_encryption=ON

# Encrypt redo and undo logs
innodb_redo_log_encrypt=ON
innodb_undo_log_encrypt=ON
```

> ⚠️ **Common Mistake:** If you put encryption settings under `[mysql]` instead of `[mysqld]` you will get error: `unknown variable 'early-plugin-load=keyring_file.so'`

Save the file:
- Press `Ctrl + X`
- Press `Y`
- Press `Enter`

---

## Step 4 — Create Keyring Directory

```bash
# Create the keyring folder
sudo mkdir -p /var/lib/mysql-keyring

# Give MySQL ownership
sudo chown mysql:mysql /var/lib/mysql-keyring

# Set secure permissions
sudo chmod 750 /var/lib/mysql-keyring
```

---

## Step 5 — Restart MySQL

```bash
sudo systemctl restart mysql

# Confirm MySQL is running properly
sudo systemctl status mysql
```

**Expected output:**
```
● mysql.service - MySQL Community Server
   Active: active (running)
```

---

## Step 6 — Verify Keyring Plugin

Login to MySQL again and verify the plugin is loaded:

```bash
mysql -u root -p
```

```sql
-- Check keyring plugin status
SELECT PLUGIN_NAME, PLUGIN_STATUS
FROM information_schema.PLUGINS
WHERE PLUGIN_NAME LIKE '%keyring%';
```

**Expected output:**
```
+--------------+---------------+
| PLUGIN_NAME  | PLUGIN_STATUS |
+--------------+---------------+
| keyring_file | ACTIVE        |
+--------------+---------------+
```

> If status is **ACTIVE** — you are good to go. ✅

---

## Step 7 — Encrypt Your Database

### 7a — Select Your Database First

> ⚠️ **Important:** Always run `USE` command first — otherwise you will get `no database selected` error.

```sql
USE your_database_name;
```

### 7b — Encrypt the Database Schema

```sql
ALTER DATABASE your_database_name ENCRYPTION='Y';
```

### 7c — Auto-Generate ALTER Commands for All Tables

```sql
-- Run this to get ALTER statements for all your tables
SELECT CONCAT('ALTER TABLE ', TABLE_NAME, " ENCRYPTION='Y';")
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'your_database_name'
AND ENGINE = 'InnoDB';
```

This will output something like:

```sql
ALTER TABLE users ENCRYPTION='Y';
ALTER TABLE orders ENCRYPTION='Y';
ALTER TABLE payments ENCRYPTION='Y';
ALTER TABLE products ENCRYPTION='Y';
-- ...and so on
```

**Copy all the output and run it.** Each table will be encrypted one by one.

---

## Step 8 — Verify Encryption

```sql
-- Check encryption status of all tables
SELECT
    TABLE_NAME,
    CREATE_OPTIONS
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'your_database_name';
```

**Expected output:**
```
+------------------+----------------+
| TABLE_NAME       | CREATE_OPTIONS |
+------------------+----------------+
| users            | ENCRYPTION="Y" |
| orders           | ENCRYPTION="Y" |
| payments         | ENCRYPTION="Y" |
+------------------+----------------+
```

> If `CREATE_OPTIONS` shows `ENCRYPTION="Y"` for all tables — **encryption is complete!** ✅

---

## Step 9 — Production Setup with docker-compose.yml

If your production database runs inside **Docker**, update your `docker-compose.yml` like this:

### `docker-compose.yml`

```yaml
version: '3.8'

services:
  db:
    image: mysql:8.0
    container_name: mysql_prod
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql
      - ./mysql/my.cnf:/etc/mysql/conf.d/my.cnf      # Custom config
      - ./mysql/keyring:/var/lib/mysql-keyring        # Keyring volume
    ports:
      - "3306:3306"
    command: >
      --early-plugin-load=keyring_file.so
      --keyring_file_data=/var/lib/mysql-keyring/keyring
      --default_table_encryption=ON
      --binlog_encryption=ON
      --innodb_redo_log_encrypt=ON
      --innodb_undo_log_encrypt=ON

volumes:
  mysql_data:
```

### `./mysql/my.cnf`

Create this file in your project:

```ini
[mysqld]
early-plugin-load=keyring_file.so
keyring_file_data=/var/lib/mysql-keyring/keyring
default_table_encryption=ON
binlog_encryption=ON
innodb_redo_log_encrypt=ON
innodb_undo_log_encrypt=ON
```

### Folder Structure

```
your-project/
├── docker-compose.yml
├── mysql/
│   ├── my.cnf          ← MySQL encryption config
│   └── keyring/        ← Keyring files stored here
├── .env                ← DB credentials (never commit this)
└── ...
```

### `.env` file (never push to GitHub)

```env
DB_ROOT_PASSWORD=your_strong_root_password
DB_NAME=your_database_name
DB_USER=your_db_user
DB_PASSWORD=your_db_password
```

> ⚠️ Add `.env` to your `.gitignore` file!

```bash
echo ".env" >> .gitignore
```

---

## Step 10 — Backup Your Keyring File

> ⚠️ **CRITICAL WARNING:** If you lose the keyring file, your encrypted data is **permanently unrecoverable** — even with a full SQL dump backup.

```bash
# Copy keyring to a safe separate location
sudo cp /var/lib/mysql-keyring/keyring /secure-location/keyring.backup

# Set secure permissions on backup
chmod 600 /secure-location/keyring.backup
```

**Recommended keyring storage options:**

| Option | Description |
|---|---|
| **AWS Secrets Manager** | Cloud-based, highly secure |
| **HashiCorp Vault** | Enterprise-grade key management |
| **Separate encrypted USB** | Physical offline backup |
| **Different server** | Never same server as database |

---

## ⚠️ Common Mistakes

| Mistake | What happens | Fix |
|---|---|---|
| Running SQL in bash terminal | `syntax error near unexpected token` | Login with `mysql -u root -p` first |
| Adding config under `[mysql]` instead of `[mysqld]` | `unknown variable 'early-plugin-load'` error | Add new `[mysqld]` section below existing `[mysql]` |
| Editing `/etc/mysql/my.cnf` when it has no `[mysqld]` | Settings ignored | Edit `/etc/mysql/conf.d/mysql.cnf` instead |
| Forgetting `USE database_name;` before ALTER TABLE | `no database selected` error | Always run `USE your_db;` first |
| Losing keyring file | Data permanently lost | Always backup keyring separately |
| Storing keyring on same server | Defeats encryption purpose | Use separate server or secret manager |
| Not backing up before encrypting | Risk of data loss | Always run `mysqldump` before encrypting |
| Forgetting to encrypt existing tables | Old tables stay unencrypted | Run ALTER TABLE on all existing tables |

---

## 🛡️ Security Layering Strategy

```
┌──────────────────────────────────────────┐
│         Your Application (Backend)       │  ← Input validation, auth ✅
├──────────────────────────────────────────┤
│      Column-Level AES (Optional)         │  ← Extra sensitive fields only
├──────────────────────────────────────────┤
│    TDE — InnoDB Tablespace Encryption    │  ← Whole database ✅ (this guide)
├──────────────────────────────────────────┤
│      Disk / Volume Encryption            │  ← OS level (LUKS on Linux)
└──────────────────────────────────────────┘
         ↑ More layers = more secure
```

---

## 📋 Quick Reference — Commands Summary

```bash
# 1. Login to MySQL
mysql -u root -p

# 2. Check version (inside MySQL)
SELECT VERSION();

# 3. Find correct config file
sudo find /etc -name "my.cnf" 2>/dev/null
ls /etc/mysql/conf.d/

# 4. Edit correct config file (Ubuntu — conf.d folder)
sudo nano /etc/mysql/conf.d/mysql.cnf
# Add [mysqld] section BELOW existing [mysql] section

# 5. Create keyring folder
sudo mkdir -p /var/lib/mysql-keyring
sudo chown mysql:mysql /var/lib/mysql-keyring
sudo chmod 750 /var/lib/mysql-keyring

# 6. Restart MySQL
sudo systemctl restart mysql

# 7. Verify keyring (inside MySQL)
SELECT PLUGIN_NAME, PLUGIN_STATUS FROM information_schema.PLUGINS WHERE PLUGIN_NAME LIKE '%keyring%';

# 8. Select database first (inside MySQL)
USE your_db;

# 9. Encrypt database (inside MySQL)
ALTER DATABASE your_db ENCRYPTION='Y';

# 10. Encrypt all tables — generate commands (inside MySQL)
SELECT CONCAT('ALTER TABLE ', TABLE_NAME, " ENCRYPTION='Y';")
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'your_db' AND ENGINE = 'InnoDB';
-- Copy output and run all ALTER TABLE commands

# 11. Verify (inside MySQL)
SELECT TABLE_NAME, CREATE_OPTIONS FROM information_schema.TABLES WHERE TABLE_SCHEMA = 'your_db';

# 12. Backup keyring
sudo cp /var/lib/mysql-keyring/keyring /secure-location/keyring.backup
```

---

## 📚 References

- [MySQL 8.0 InnoDB Encryption Docs](https://dev.mysql.com/doc/refman/8.0/en/innodb-data-encryption.html)
- [MySQL Keyring Plugin](https://dev.mysql.com/doc/refman/8.0/en/keyring.html)
- [HashiCorp Vault](https://www.vaultproject.io/)
- [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/)

---

> 📝 **Author:** Ali Husnain
> 🗓️ **Last Updated:** 2026
> 🔒 **Purpose:** MySQL TDE Encryption — Complete Production Guide
