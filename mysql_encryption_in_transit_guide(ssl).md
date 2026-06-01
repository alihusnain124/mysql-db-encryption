# 🔐 MySQL Encryption in Transit (SSL/TLS) — Complete Beginner Guide

> A full, plain-language explanation of **encryption in transit** for MySQL:
> what it is, what every certificate/key file actually means, **why you don't
> always need it**, how to create the files, where to store them, and how to
> wire it into a **NestJS** app — for both **AWS RDS** and **your own server**.

---

## 📌 Table of Contents

1. [What is Encryption in Transit?](#1-what-is-encryption-in-transit)
2. [The Simple Mental Model — ID Cards & a Passport Office](#2-the-simple-mental-model--id-cards--a-passport-office)
3. [The 6 Files Explained (What Are All These?)](#3-the-6-files-explained-what-are-all-these)
4. [Why You DON'T Need This on the Same Machine](#4-why-you-dont-need-this-on-the-same-machine)
5. [Your Situation — Now vs Later](#5-your-situation--now-vs-later)
6. [Path A — AWS RDS (the easy one)](#6-path-a--aws-rds-the-easy-one)
7. [Path B — Your Own MySQL Server](#7-path-b--your-own-mysql-server)
8. [Where to Store the Certificates](#8-where-to-store-the-certificates)
9. [NestJS Setup (Works Both Modes)](#9-nestjs-setup-works-both-modes)
10. [Verify It's Actually Encrypted](#10-verify-its-actually-encrypted)
11. [Force SSL (Optional but Recommended)](#11-force-ssl-optional-but-recommended)
12. [Summary Tables & Checklist](#12-summary-tables--checklist)

---

## 1. What is Encryption in Transit?

**"In transit"** means data while it is **moving** from one place to another
through a network.

When your NestJS app talks to MySQL, two trips happen:

```
App   ──►  "SELECT * FROM users"        ──►  MySQL     (request travelling)
MySQL ──►  "Ali, 35202-..., 0300-..."   ──►  App       (results travelling)
```

If this travels over a network **without** SSL, anyone sitting on that network
can read it in plain text — like reading an open postcard:

```
WITHOUT SSL  →  name | cnic | phone   (hacker reads everything ❌)
WITH SSL     →  x8#Kp2$mQ9nL...        (hacker sees garbage ✅)
```

> ⚠️ Note: **HTTPS** (browser → backend) and **MySQL SSL** (backend → database)
> are two completely separate connections. Having HTTPS does **not** protect the
> app-to-database trip. This guide is only about the **backend → MySQL** trip.

---

## 2. The Simple Mental Model — ID Cards & a Passport Office

The whole certificate system is just **ID cards** and a **passport office**.

```
                 ┌────────────────────────────┐
                 │   Certificate Authority    │   ← the "passport office"
                 │     (trusted stamp office) │
                 └─────────────┬──────────────┘
                  signs        │        signs
            ┌──────────────────┴───────────────────┐
            ▼                                       ▼
   ┌──────────────────┐                  ┌──────────────────┐
   │ Database server  │ ◄── encrypted ──►│ Your app (NestJS)│
   │ proves it's real │      link        │ proves it's real │
   └──────────────────┘                  └──────────────────┘
```

Three simple words:

- **Key** = a **secret password**. Never share it.
- **Certificate (cert)** = a **public ID card** you are allowed to show.
- **CA (Certificate Authority)** = the **passport office** that stamps the ID
  cards, so everyone trusts them.

The CA stamps both ID cards. Because both sides trust the same office, the
database can prove "I'm the real DB" and the app can prove "I'm the real app" —
so nobody can secretly pretend to be either one.

---

## 3. The 6 Files Explained (What Are All These?)

Every identity (the server and the app) has **two matching pieces**: a secret
and a public ID.

| File | Plain meaning | Secret? | Lives on |
|------|---------------|---------|----------|
| `ca-key.pem` | The passport office's **secret stamp**. Used once to sign ID cards, then locked away. | 🔒 Yes | Kept safe / offline |
| `ca-cert.pem` | The passport office's **public ID**. Both sides use it to *check* an ID card is genuine. **Everyone needs this.** | 🌍 Public | Both machines |
| `server-key.pem` | The database's **secret password**. | 🔒 Yes | DB machine only |
| `server-cert.pem` | The database's **ID card** (stamped by CA). DB shows it to prove identity. | 🌍 Public | DB machine |
| `client-key.pem` | The app's **secret password**. | 🔒 Yes | App machine only |
| `client-cert.pem` | The app's **ID card** (stamped by CA). App shows it to prove identity. | 🌍 Public | App machine |

**Rule of thumb:**
`-key.pem` = always secret.
`-cert.pem` = the public ID you may hand out.

> 📝 The client cert + key (`client-*.pem`) are only needed if the database
> **forces** client ID checking (`REQUIRE X509`). For most setups — including
> AWS RDS — you only need `ca-cert.pem` on the app side.

---

## 4. Why You DON'T Need This on the Same Machine

ID cards and stamps exist to protect a conversation that **travels across a
network**, where a stranger could listen in or impersonate the database.

When the app and the database are on the **same machine**, the conversation
moves through `localhost` (`127.0.0.1`) — it never leaves the building.

```
┌─────────────────────────────────┐
│         YOUR SERVER             │
│                                 │
│   NestJS App                    │
│       │  localhost only         │
│       │  never touches network  │
│       ▼                         │
│   MySQL Database                │
│                                 │
│   No road for a hacker ✅       │
└─────────────────────────────────┘
```

No network road → no stranger to fool → nothing to scramble. **That is why
same-machine setups skip the whole certificate dance.**

---

## 5. Your Situation — Now vs Later

| Stage | Setup | Need MySQL SSL? | What to do |
|-------|-------|-----------------|------------|
| **Now** | App + DB on **same machine** | ❌ No | `DB_SSL=false`. Create/store **nothing**. |
| **Later** | DB on **AWS RDS** | ✅ Yes | Download 1 file → see [Path A](#6-path-a--aws-rds-the-easy-one) |
| **Later** | DB on **your own separate server** | ✅ Yes | Create the 6 files → see [Path B](#7-path-b--your-own-mysql-server) |

**The smart approach:** set up the NestJS code **now** so SSL can be switched on
with one `.env` setting later. On deploy day you just flip the switch — no code
rewrite. The config in [section 9](#9-nestjs-setup-works-both-modes) does exactly
that.

---

## 6. Path A — AWS RDS (the easy one)

With RDS, **Amazon is both the passport office AND the server**. They already
created every key and certificate for the server side. You do **not** run any
`openssl` commands.

You just download **one** public file (the CA bundle):

```bash
# Works for all AWS regions
wget https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem
```

Then store it on your **app machine** and point NestJS at it. That's the entire
setup — no client cert, no key generation.

```
RDS setup = download global-bundle.pem  →  point app's ssl.ca at it  →  done ✅
```

---

## 7. Path B — Your Own MySQL Server

This is the **only** time you create the files yourself. It happens in 3 stages.

### Stage 1 — Create the passport office (CA)

```bash
sudo mkdir -p /etc/mysql/ssl
cd /etc/mysql/ssl

# CA secret stamp
sudo openssl genrsa 2048 > ca-key.pem

# CA public ID (valid 10 years)
sudo openssl req -new -x509 -nodes -days 3650 \
  -key ca-key.pem -out ca-cert.pem -subj "/CN=MySQL-CA"
```

### Stage 2 — Create the database's ID card (server)

```bash
# Server secret + a signing request
sudo openssl req -newkey rsa:2048 -nodes \
  -keyout server-key.pem -out server-req.pem -subj "/CN=MySQL-Server"

# Office stamps it → server ID card
sudo openssl x509 -req -days 3650 -set_serial 01 \
  -in server-req.pem -out server-cert.pem \
  -CA ca-cert.pem -CAkey ca-key.pem
```

### Stage 3 — Create the app's ID card (client)

```bash
# Client secret + a signing request
sudo openssl req -newkey rsa:2048 -nodes \
  -keyout client-key.pem -out client-req.pem -subj "/CN=MySQL-Client"

# Office stamps it → client ID card
sudo openssl x509 -req -days 3650 -set_serial 01 \
  -in client-req.pem -out client-cert.pem \
  -CA ca-cert.pem -CAkey ca-key.pem

# Fix ownership/permissions
sudo chown mysql:mysql *.pem
sudo chmod 640 *-key.pem
sudo chmod 644 ca-cert.pem server-cert.pem client-cert.pem
```

### Tell MySQL to use the server certificates

Edit `/etc/mysql/my.cnf`:

```ini
[mysqld]
ssl-ca=/etc/mysql/ssl/ca-cert.pem
ssl-cert=/etc/mysql/ssl/server-cert.pem
ssl-key=/etc/mysql/ssl/server-key.pem
require_secure_transport=ON
```

```bash
sudo systemctl restart mysql
```

---

## 8. Where to Store the Certificates

| File | Database machine | App machine (NestJS) | Keep offline |
|------|------------------|----------------------|--------------|
| `ca-cert.pem` | ✅ `/etc/mysql/ssl/` | ✅ `/etc/ssl/mysql/` | — |
| `server-cert.pem` | ✅ `/etc/mysql/ssl/` | ❌ | — |
| `server-key.pem` | ✅ `/etc/mysql/ssl/` | ❌ | — |
| `client-cert.pem` | (created here) | ✅ `/etc/ssl/mysql/` *(only if X509)* | — |
| `client-key.pem` | (created here) | ✅ `/etc/ssl/mysql/` *(only if X509)* | — |
| `ca-key.pem` | ❌ not needed at runtime | ❌ | 🔒 ✅ store safely |

On the **app machine**, make one folder and put the file(s) there:

```bash
sudo mkdir -p /etc/ssl/mysql
# Put global-bundle.pem (RDS)  OR  ca-cert.pem (own server) here
sudo chmod 644 /etc/ssl/mysql/*.pem
```

> 🚫 **Never** commit certificate/key files to git / GitHub. Keep them on the
> server only, or in a secrets manager.

---

## 9. NestJS Setup (Works Both Modes)

This config works in **both** local dev (SSL off) and production (SSL on). You
control it entirely from `.env` — no code changes when you deploy.

### `src/config/database.config.ts`

```typescript
import { TypeOrmModuleOptions } from '@nestjs/typeorm';
import * as fs from 'fs';

export const databaseConfig = (): TypeOrmModuleOptions => {
  // Read the switch from .env. Default = OFF (safe for local dev).
  const useSsl = process.env.DB_SSL === 'true';

  return {
    type: 'mysql',
    host: process.env.DB_HOST ?? '127.0.0.1',
    port: Number(process.env.DB_PORT ?? 3306),
    username: process.env.DB_USERNAME,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_DATABASE,
    autoLoadEntities: true,
    synchronize: false, // keep false in production

    // ── Encryption in transit (SSL/TLS) ──
    ssl: useSsl
      ? {
          // CA proves the DB server is genuine.
          // RDS  -> global-bundle.pem   |   own server -> ca-cert.pem
          ca: fs.readFileSync(process.env.DB_SSL_CA as string),

          // Client cert+key: ONLY if your own MySQL forces X509.
          // RDS does NOT need these — leave them unset.
          ...(process.env.DB_SSL_CERT && {
            cert: fs.readFileSync(process.env.DB_SSL_CERT),
          }),
          ...(process.env.DB_SSL_KEY && {
            key: fs.readFileSync(process.env.DB_SSL_KEY),
          }),

          rejectUnauthorized: true, // verify the server cert (recommended)
        }
      : false, // SSL fully off for local same-machine dev
  };
};
```

### `src/app.module.ts`

```typescript
import { TypeOrmModule } from '@nestjs/typeorm';
import { databaseConfig } from './config/database.config';

@Module({
  imports: [
    TypeOrmModule.forRootAsync({ useFactory: databaseConfig }),
    // ...your other modules
  ],
})
export class AppModule {}
```

### `.env`

```env
# ===== LOCAL DEV (same machine — what you have NOW) =====
DB_HOST=127.0.0.1
DB_PORT=3306
DB_USERNAME=root
DB_PASSWORD=your_password
DB_DATABASE=your_db
DB_SSL=false                 # SSL off, no certificates needed

# ===== PRODUCTION later — RDS example =====
# DB_HOST=mydb.xxxxx.us-east-1.rds.amazonaws.com
# DB_SSL=true
# DB_SSL_CA=/etc/ssl/mysql/global-bundle.pem
# (no DB_SSL_CERT / DB_SSL_KEY needed for RDS)

# ===== PRODUCTION later — your own MySQL server example =====
# DB_HOST=10.0.0.5
# DB_SSL=true
# DB_SSL_CA=/etc/ssl/mysql/ca-cert.pem
# DB_SSL_CERT=/etc/ssl/mysql/client-cert.pem   # only if you force X509
# DB_SSL_KEY=/etc/ssl/mysql/client-key.pem     # only if you force X509
```

---

## 10. Verify It's Actually Encrypted

After deploying with `DB_SSL=true`, connect to MySQL and run:

```sql
SHOW STATUS LIKE 'Ssl_cipher';
```

If SSL is working, you'll see a cipher name such as:

```
TLS_AES_256_GCM_SHA384
```

If the value is **empty**, the connection is **not** encrypted yet.

---

## 11. Force SSL (Optional but Recommended)

By default MySQL still *allows* unencrypted connections. To make the database
**reject** any connection that isn't encrypted, force it per user:

```sql
-- Require SSL for the app's database user
ALTER USER 'appuser'@'%' REQUIRE SSL;

-- Stricter: also require a valid client certificate (X509)
-- ALTER USER 'appuser'@'%' REQUIRE X509;
```

> Only use `REQUIRE X509` if you've placed `client-cert.pem` + `client-key.pem`
> on the app machine and set them in `.env`.

---

## 12. Summary Tables & Checklist

### What protects what

| Protection | Trip it secures | Needed same machine? | Needed different machine? |
|------------|-----------------|----------------------|---------------------------|
| HTTPS | Browser → Backend | ✅ Always | ✅ Always |
| **MySQL SSL** (this guide) | Backend → MySQL | ❌ Not critical | ✅ Must have |
| TDE | MySQL → Disk | ✅ Always | ✅ Always |

### Deployment checklist

```
TODAY (same machine):
☑ DB_SSL=false in .env
☑ Create / store nothing
☑ App runs normally

DEPLOY DAY — AWS RDS:
☑ wget global-bundle.pem  → /etc/ssl/mysql/
☑ Set DB_HOST = RDS endpoint
☑ DB_SSL=true , DB_SSL_CA=/etc/ssl/mysql/global-bundle.pem
☑ Verify: SHOW STATUS LIKE 'Ssl_cipher';

DEPLOY DAY — your own DB server:
☑ Create 6 files (openssl, 3 stages)
☑ Server files → /etc/mysql/ssl/  on DB machine
☑ Copy ca-cert.pem (+ client files if X509) → /etc/ssl/mysql/ on app machine
☑ my.cnf points at server certs , restart mysql
☑ DB_SSL=true , set DB_SSL_CA (+ DB_SSL_CERT / DB_SSL_KEY if X509)
☑ Verify: SHOW STATUS LIKE 'Ssl_cipher';
☑ (Optional) ALTER USER ... REQUIRE SSL;
```

---

> 🔑 **One-line takeaway:** A *key* is a secret, a *cert* is a public ID, the
> *CA* stamps the IDs so both sides trust each other — and you only need any of
> it when the app and database talk **across a network**.
