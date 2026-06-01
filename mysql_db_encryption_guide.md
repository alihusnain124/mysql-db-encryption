# 🔓 How to Disable TDE (Transparent Data Encryption) — Complete Guide

> **Step-by-step guide** to safely disable TDE encryption from your MySQL database without losing any data.

---

## 📌 Table of Contents

1. [Before You Start — Important Warnings](#before-you-start--important-warnings)
2. [Step 1 — Take Full Backup](#step-1--take-full-backup)
3. [Step 2 — Login to MySQL](#step-2--login-to-mysql)
4. [Step 3 — Check Current Encryption Status](#step-3--check-current-encryption-status)
5. [Step 4 — Decrypt All Tables](#step-4--decrypt-all-tables)
6. [Step 5 — Decrypt Database Schema](#step-5--decrypt-database-schema)
7. [Step 6 — Verify Decryption Complete](#step-6--verify-decryption-complete)
8. [Step 7 — Remove Encryption from Config File](#step-7--remove-encryption-from-config-file)
9. [Step 8 — Restart MySQL](#step-8--restart-mysql)
10. [Step 9 — Final Verification](#step-9--final-verification)
11. [Disable TDE in Docker Setup](#disable-tde-in-docker-setup)
12. [Disable Only Specific Tables](#disable-only-specific-tables)
13. [Common Errors and Fixes](#common-errors-and-fixes)
14. [Quick Reference Commands](#quick-reference-commands)

---

## ⚠️ Before You Start — Important Warnings

```
┌─────────────────────────────────────────────────────────────┐
│                    READ THIS FIRST                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ✅ DO THIS BEFORE DISABLING TDE:                          │
│     1. Take a full database backup                          │
│     2. Keep your keyring file safe until fully done         │
│     3. Do this during low traffic time                      │
│     4. Verify backup works before proceeding                │
│                                                             │
│  ❌ NEVER DO THIS:                                         │
│     1. Delete keyring file before decrypting tables         │
│     2. Remove config before running ALTER TABLE             │
│     3. Skip the backup step                                 │
│                                                             │
│  IF YOU DELETE KEYRING BEFORE DECRYPTING:                  │
│     → MySQL cannot read encrypted tables                    │
│     → Data becomes permanently unreadable                   │
│     → No recovery possible ❌                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 1 — Take Full Backup

> ⚠️ **Never skip this step. Always backup before making any changes.**

```bash
# Full database backup
mysqldump -u root -p --all-databases > backup_before_disable_tde.sql

# Verify backup file was created and has data
ls -lh backup_before_disable_tde.sql

# Also backup your keyring file
sudo cp /var/lib/mysql-keyring/keyring ~/keyring_backup_safe.key

# Confirm both exist
ls -lh backup_before_disable_tde.sql
ls -lh ~/keyring_backup_safe.key
```

---

## Step 2 — Login to MySQL

```bash
# Open terminal and login to MySQL
mysql -u root -p

# Enter your root password
# You will see: mysql>
# Now you are inside MySQL
```

---

## Step 3 — Check Current Encryption Status

Before disabling, check which tables are currently encrypted:

```sql
-- Check all tables and their encryption status
SELECT
    TABLE_NAME,
    CREATE_OPTIONS
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'your_database_name';
```

**Output when tables ARE encrypted:**
```
+------------------+----------------+
| TABLE_NAME       | CREATE_OPTIONS |
+------------------+----------------+
| users            | ENCRYPTION="Y" |  ← encrypted
| orders           | ENCRYPTION="Y" |  ← encrypted
| payments         | ENCRYPTION="Y" |  ← encrypted
+------------------+----------------+
```

**Also check keyring plugin is active:**
```sql
SELECT PLUGIN_NAME, PLUGIN_STATUS
FROM information_schema.PLUGINS
WHERE PLUGIN_NAME LIKE '%keyring%';

-- Must show ACTIVE before proceeding
-- +──────────────+───────────────+
-- | keyring_file | ACTIVE        |
-- +──────────────+───────────────+
```

> ⚠️ If keyring is NOT active, do not proceed — fix keyring first or data will be lost.

---

## Step 4 — Decrypt All Tables

### Option A — Auto Generate Decrypt Commands (Recommended)

```sql
-- This generates ALTER TABLE commands for all your tables
SELECT CONCAT('ALTER TABLE ', TABLE_NAME, " ENCRYPTION='N';")
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'your_database_name'
AND ENGINE = 'InnoDB';
```

This will output something like:

```sql
ALTER TABLE users ENCRYPTION='N';
ALTER TABLE orders ENCRYPTION='N';
ALTER TABLE payments ENCRYPTION='N';
ALTER TABLE products ENCRYPTION='N';
ALTER TABLE categories ENCRYPTION='N';
```

**Copy all lines and run them one by one.**

---

### Option B — Decrypt Tables Manually One by One

```sql
ALTER TABLE users ENCRYPTION='N';
ALTER TABLE orders ENCRYPTION='N';
ALTER TABLE payments ENCRYPTION='N';
ALTER TABLE products ENCRYPTION='N';
-- add all your table names here
```

> Each ALTER TABLE command will take a few seconds depending on table size. This is normal.

---

## Step 5 — Decrypt Database Schema

After all tables are decrypted, decrypt the database schema itself:

```sql
ALTER DATABASE your_database_name ENCRYPTION='N';
```

---

## Step 6 — Verify Decryption Complete

```sql
-- Check all tables are now decrypted
SELECT
    TABLE_NAME,
    CREATE_OPTIONS
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'your_database_name';
```

**Expected output after successful decryption:**
```
+------------------+----------------+
| TABLE_NAME       | CREATE_OPTIONS |
+------------------+----------------+
| users            |                |  ← empty = decrypted ✅
| orders           |                |  ← empty = decrypted ✅
| payments         |                |  ← empty = decrypted ✅
+------------------+----------------+
```

> If `CREATE_OPTIONS` is **empty** for ALL tables — decryption is complete. ✅
> If any table still shows `ENCRYPTION="Y"` — run ALTER TABLE for that table again.

---

## Step 7 — Remove Encryption from Config File

Exit MySQL first, then find and edit the correct config file:

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
# Step 4 — Open the correct file to edit
sudo nano /etc/mysql/conf.d/mysql.cnf
```

> ⚠️ **Important:** You will see `[mysql]` at the top — do NOT remove it. Find the `[mysqld]` section below it and comment out or remove only the encryption lines.

**Your file should look like this after editing:**

```ini
[mysql]
# whatever was already here — DO NOT touch this section

[mysqld]
# ── REMOVE or COMMENT these encryption lines ────────────
# early-plugin-load=keyring_file.so
# keyring_file_data=/var/lib/mysql-keyring/keyring
# default_table_encryption=ON
# binlog_encryption=ON
# innodb_redo_log_encrypt=ON
# innodb_undo_log_encrypt=ON
```

> To comment a line — add `#` at the start of the line.
> To remove — delete the line completely.

Save and exit:
- Press `Ctrl + X`
- Press `Y`
- Press `Enter`

---

## Step 8 — Restart MySQL

```bash
sudo systemctl restart mysql

# Confirm MySQL started successfully
sudo systemctl status mysql
```

**Expected output:**
```
● mysql.service - MySQL Community Server
   Active: active (running)  ✅
```

---

## Step 9 — Final Verification

Login to MySQL again and confirm everything is working:

```bash
mysql -u root -p
```

```sql
-- Check keyring plugin is now gone (removed from config)
SELECT PLUGIN_NAME, PLUGIN_STATUS
FROM information_schema.PLUGINS
WHERE PLUGIN_NAME LIKE '%keyring%';
-- Should return empty result

-- Check all tables are decrypted
SELECT TABLE_NAME, CREATE_OPTIONS
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'your_database_name';
-- CREATE_OPTIONS should be empty for all tables

-- Check you can still read your data normally
SELECT * FROM users LIMIT 5;
-- Should show normal readable data ✅
```

> If data is readable and `CREATE_OPTIONS` is empty — **TDE is fully disabled!** ✅

---

## 🐳 Disable TDE in Docker Setup

If your MySQL runs inside Docker:

### Step 1 — Enter the Container and Decrypt Tables

```bash
# Enter running MySQL container
docker exec -it mysql_prod bash

# Login to MySQL inside container
mysql -u root -p
```

```sql
-- Generate decrypt commands for all tables
SELECT CONCAT('ALTER TABLE ', TABLE_NAME, " ENCRYPTION='N';")
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'your_database_name'
AND ENGINE = 'InnoDB';

-- Run all generated ALTER TABLE commands
ALTER TABLE users ENCRYPTION='N';
ALTER TABLE orders ENCRYPTION='N';
-- ... all tables

-- Decrypt database schema
ALTER DATABASE your_database_name ENCRYPTION='N';

-- Verify
SELECT TABLE_NAME, CREATE_OPTIONS
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'your_database_name';

-- Exit MySQL
exit

-- Exit container
exit
```

### Step 2 — Update docker-compose.yml

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
      - ./mysql/my.cnf:/etc/mysql/conf.d/my.cnf
    ports:
      - "3306:3306"
    # ── REMOVE these encryption lines ──────────────────
    # command: >
    #   --early-plugin-load=keyring_file.so
    #   --keyring_file_data=/var/lib/mysql-keyring/keyring
    #   --default_table_encryption=ON
    #   --binlog_encryption=ON
    #   --innodb_redo_log_encrypt=ON
    #   --innodb_undo_log_encrypt=ON

volumes:
  mysql_data:
```

### Step 3 — Update ./mysql/my.cnf

```ini
[mysql]
# whatever was already here — DO NOT touch this section

[mysqld]
# ── Comment out or remove all encryption lines ──────
# early-plugin-load=keyring_file.so
# keyring_file_data=/var/lib/mysql-keyring/keyring
# default_table_encryption=ON
# binlog_encryption=ON
# innodb_redo_log_encrypt=ON
# innodb_undo_log_encrypt=ON
```

### Step 4 — Restart Docker Container

```bash
# Restart container with new config
docker-compose down
docker-compose up -d

# Check container is running
docker ps

# Verify MySQL is healthy
docker logs mysql_prod
```

---

## 🎯 Disable Only Specific Tables

If you want to disable TDE on **some tables only** and keep others encrypted:

```sql
-- Decrypt only specific tables
ALTER TABLE logs ENCRYPTION='N';
ALTER TABLE sessions ENCRYPTION='N';

-- Keep sensitive tables encrypted (do not touch these)
-- ALTER TABLE users ENCRYPTION='N';    ← leave this
-- ALTER TABLE payments ENCRYPTION='N'; ← leave this
```

**Check mixed status:**
```sql
SELECT TABLE_NAME, CREATE_OPTIONS
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'your_database_name';

-- Example output:
-- +──────────+────────────────+
-- | users    | ENCRYPTION="Y" |  ← still encrypted ✅
-- | payments | ENCRYPTION="Y" |  ← still encrypted ✅
-- | logs     |                |  ← decrypted
-- | sessions |                |  ← decrypted
-- +──────────+────────────────+
```

---

## ❌ Common Errors and Fixes

### Error 1 — Keyring Plugin Not Active

```
ERROR: Can't initialize keyring plugin
```

**Fix:**
```bash
# Check keyring directory exists
ls -la /var/lib/mysql-keyring/

# If missing, restore keyring backup
sudo cp ~/keyring_backup_safe.key /var/lib/mysql-keyring/keyring
sudo chown mysql:mysql /var/lib/mysql-keyring/keyring
sudo chmod 640 /var/lib/mysql-keyring/keyring
sudo systemctl restart mysql
```

---

### Error 2 — Table Still Shows Encrypted After ALTER

```sql
-- Run this to check which tables are still encrypted
SELECT TABLE_NAME, CREATE_OPTIONS
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'your_database_name'
AND CREATE_OPTIONS LIKE '%ENCRYPTION%';

-- Run ALTER TABLE again for those specific tables
ALTER TABLE problematic_table ENCRYPTION='N';
```

---

### Error 3 — MySQL Won't Start After Config Change

```bash
# Check MySQL error log
sudo tail -50 /var/log/mysql/error.log

# Most common fix — restore config
sudo nano /etc/mysql/conf.d/mysql.cnf
# Make sure no syntax errors in config file
# Make sure encryption lines are commented out under [mysqld]

sudo systemctl restart mysql
```

---

### Error 4 — Access Denied

```
ERROR 1045: Access denied for user 'root'@'localhost'
```

**Fix:**
```bash
# Use sudo with MySQL
sudo mysql

# Or specify socket
mysql -u root -p --socket=/var/run/mysqld/mysqld.sock
```

---

## 📋 Quick Reference Commands

```bash
# ── TERMINAL COMMANDS ────────────────────────────────────

# Backup before disabling
mysqldump -u root -p --all-databases > backup_before_disable_tde.sql

# Backup keyring
sudo cp /var/lib/mysql-keyring/keyring ~/keyring_backup_safe.key

# Login to MySQL
mysql -u root -p

# Edit correct config file (Ubuntu — conf.d folder)
sudo nano /etc/mysql/conf.d/mysql.cnf

# Restart MySQL
sudo systemctl restart mysql

# Check MySQL status
sudo systemctl status mysql

# Check MySQL error log
sudo tail -50 /var/log/mysql/error.log

# Docker — enter container
docker exec -it mysql_prod bash
```

```sql
-- ── MYSQL COMMANDS (run inside mysql> prompt) ────────────

-- Check keyring active
SELECT PLUGIN_NAME, PLUGIN_STATUS
FROM information_schema.PLUGINS
WHERE PLUGIN_NAME LIKE '%keyring%';

-- Check encryption status of all tables
SELECT TABLE_NAME, CREATE_OPTIONS
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'your_database_name';

-- Auto generate decrypt commands
SELECT CONCAT('ALTER TABLE ', TABLE_NAME, " ENCRYPTION='N';")
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'your_database_name'
AND ENGINE = 'InnoDB';

-- Decrypt database schema
ALTER DATABASE your_database_name ENCRYPTION='N';

-- Check only encrypted tables
SELECT TABLE_NAME, CREATE_OPTIONS
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'your_database_name'
AND CREATE_OPTIONS LIKE '%ENCRYPTION%';

-- Verify data still readable
SELECT * FROM users LIMIT 5;
```

---

## ✅ Disable TDE Checklist

```
Before Starting:
☐ Take full mysqldump backup
☐ Backup keyring file separately
☐ Confirm keyring plugin is ACTIVE
☐ Plan for low traffic window

Decryption Steps:
☐ Login to MySQL (mysql -u root -p)
☐ Check current encryption status
☐ Generate ALTER TABLE commands for all tables
☐ Run all ALTER TABLE ENCRYPTION='N' commands
☐ Run ALTER DATABASE ENCRYPTION='N'
☐ Verify all tables show empty CREATE_OPTIONS

Config Cleanup:
☐ Exit MySQL
☐ Check /etc/mysql/my.cnf — if no [mysqld] section edit conf.d/mysql.cnf instead
☐ Open /etc/mysql/conf.d/mysql.cnf
☐ Comment out all encryption lines under [mysqld]
☐ DO NOT touch [mysql] section at top
☐ Restart MySQL
☐ Confirm MySQL started successfully

Final Check:
☐ Login to MySQL again
☐ Verify keyring plugin removed
☐ Verify all tables decrypted
☐ Run SELECT on tables — confirm data readable
☐ TDE successfully disabled ✅
```

---

> 📝 **Author:** Ali Husnain
> 🗓️ **Last Updated:** 2026
> 🔒 **Purpose:** MySQL TDE Disable — Complete Safe Guide
