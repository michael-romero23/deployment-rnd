Below is a **clean, production-style way to set up Harbor with HTTPS**, assuming you’re running it via **Docker / docker-compose** (which is the most common), and this works well with **Docker Desktop + WSL** too.

I’ll show **two options**:

1. **Self-signed cert (internal / dev)**
2. **Let’s Encrypt (public domain)**

---

# ✅ Prerequisites

* A **domain name** pointing to your Harbor server
  (e.g. `harbor.example.com`)
* Ports **80 & 443 open**
* Docker + docker-compose
* Harbor **offline installer** (`harbor-offline-installer.tgz`)

---

# OPTION 1 — HTTPS with Self-Signed Certificate (DEV / INTERNAL)

## 1️⃣ Create certificates

```bash
mkdir -p /data/cert
cd /data/cert
```

### Create CA

```bash
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -sha512 -days 3650 \
  -subj "/C=PH/ST=MetroManila/L=QC/O=MyOrg/OU=IT/CN=Harbor-CA" \
  -key ca.key -out ca.crt
```

### Create server cert

```bash
openssl genrsa -out harbor.key 4096
```

```bash
openssl req -sha512 -new \
  -subj "/C=PH/ST=MetroManila/L=QC/O=MyOrg/OU=IT/CN=asuw10.itsrvr01.com" \
  -key harbor.key -out harbor.csr
```

```bash
openssl x509 -req -sha512 -days 3650 \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -in harbor.csr -out harbor.crt
```

### Convert cert (Harbor requires this)

```bash
openssl x509 -inform PEM -in harbor.crt -out harbor.cert
```

---

## 2️⃣ Configure Harbor

Edit **`harbor.yml`**:

```yaml
hostname: harbor.example.com

http:
  port: 80

https:
  port: 443
  certificate: /data/cert/harbor.crt
  private_key: /data/cert/harbor.key

harbor_admin_password: StrongPassword123
```

---

## 3️⃣ Install / Apply Harbor

```bash
./prepare
./install.sh
```

(or `./install.sh --with-trivy` if you want scanning)

---

## 4️⃣ Trust the CA on clients (IMPORTANT)

### On Docker hosts (WSL / Linux):

```bash
sudo mkdir -p /etc/docker/certs.d/harbor.example.com
sudo cp ca.crt /etc/docker/certs.d/harbor.example.com/ca.crt
sudo systemctl restart docker
```

(For Docker Desktop: restart Docker Desktop instead)

---

## 5️⃣ Login test

```bash
docker login harbor.example.com
```

✅ Should succeed without TLS errors

---

# OPTION 2 — HTTPS with Let’s Encrypt (PRODUCTION)

⚠️ Requires **public domain & port 80 reachable**

## 1️⃣ Get cert using certbot

```bash
sudo apt install certbot -y
```

```bash
sudo certbot certonly --standalone -d harbor.example.com
```

Certs will be here:

```text
/etc/letsencrypt/live/harbor.example.com/
```

---

## 2️⃣ Update `harbor.yml`

```yaml
hostname: harbor.example.com

https:
  port: 443
  certificate: /etc/letsencrypt/live/harbor.example.com/fullchain.pem
  private_key: /etc/letsencrypt/live/harbor.example.com/privkey.pem
```

---

## 3️⃣ Reinstall / Apply

```bash
./prepare
./install.sh
```

---

## 4️⃣ Auto-renew certs

```bash
sudo crontab -e
```

```cron
0 3 * * * certbot renew --quiet && docker restart harbor-nginx
```

---

# 🔍 Common Problems & Fixes

### ❌ `x509: certificate signed by unknown authority`

➡ Client doesn’t trust CA
✔ Install CA cert on Docker host

---

### ❌ Docker push fails but UI works

➡ Docker uses **different cert trust path**
✔ Must be in:

```bash
/etc/docker/certs.d/<harbor-domain>/ca.crt
```

---

### ❌ Harbor UI loads but login fails

➡ `hostname` mismatch
✔ `hostname` MUST match cert CN exactly

---

# ✅ Recommended Production Setup

| Component | Recommendation                  |
| --------- | ------------------------------- |
| HTTPS     | Let’s Encrypt                   |
| Storage   | External volume                 |
| Auth      | LDAP / OIDC                     |
| Backup    | `/data` directory               |
| Exposure  | Reverse proxy (NGINX / Traefik) |

---

If you want, tell me:

* Is Harbor **local / internal / public**?
* Are you running it **on WSL, VM, or bare metal**?
* Do you want **NGINX / Traefik / HAProxy** in front?

I’ll tailor the exact config for your environment 🔐🚀


# Issue

This is a **classic Harbor HTTPS setup failure**, and the traceback tells us exactly where it breaks 👍
Harbor **cannot find or read your TLS private key / cert file**.

---

## 🔴 Root Cause (from the traceback)

This line is the key:

```
prepare_nginx_certs(config_dict['cert_key_path'], config_dict['cert_path'])
shutil.copy2(host_ngx_cert_key_path, real_key_path)
with open(src, 'rb') as fsrc:
```

➡ **Harbor is trying to copy the TLS key or cert**
➡ **The source file does NOT exist or is not readable**

In short:

> **The paths in `harbor.yml` do not point to real files (or permission denied)**

---

## ✅ Step-by-step Fix

### 1️⃣ Check what Harbor expects

Open your `harbor.yml`:

```bash
nano harbor.yml
```

Make sure this section is correct:

```yaml
hostname: harbor.example.com

https:
  port: 443
  certificate: /data/cert/harbor.crt
  private_key: /data/cert/harbor.key
```

⚠️ **These paths must exist on the HOST**, not inside a container.

---

### 2️⃣ Verify cert files actually exist

Run:

```bash
ls -l /data/cert
```

You MUST see:

```
harbor.crt
harbor.key
harbor.cert   (optional but recommended)
ca.crt        (if self-signed)
```

If `/data/cert` does NOT exist, Harbor will fail exactly like this.

---

### 3️⃣ Fix permissions (VERY common issue)

Harbor runs containers as **root**, but WSL file permissions still matter.

```bash
sudo chmod 600 /data/cert/harbor.key
sudo chmod 644 /data/cert/harbor.crt
sudo chmod 644 /data/cert/ca.crt
sudo chown root:root /data/cert/*
```

---

### 4️⃣ If you used Let’s Encrypt (IMPORTANT)

If your `harbor.yml` points to:

```yaml
certificate: /etc/letsencrypt/live/harbor.example.com/fullchain.pem
private_key: /etc/letsencrypt/live/harbor.example.com/privkey.pem
```

Then **Docker containers CANNOT read symlinks** used by Let’s Encrypt.

✔ **Fix: copy the files**

```bash
sudo mkdir -p /data/cert

sudo cp /etc/letsencrypt/live/harbor.example.com/fullchain.pem /data/cert/harbor.crt
sudo cp /etc/letsencrypt/live/harbor.example.com/privkey.pem /data/cert/harbor.key
```

Update `harbor.yml` to `/data/cert/*`.

---

### 5️⃣ Clean previous failed config (important)

From Harbor directory:

```bash
sudo docker compose down -v
sudo rm -rf /data/harbor/*
```

Then rerun:

```bash
./prepare
./install.sh
```

---

## 🔍 Quick Diagnostic Command

Run this and paste output if it still fails:

```bash
ls -l \
/data/cert/harbor.crt \
/data/cert/harbor.key
```

---

## ✅ Why this happens a lot in WSL + Docker Desktop

* Harbor runs in **containers**
* Cert paths must exist **on the Linux filesystem**
* Symlinks + permissions break silently
* Harbor doesn’t validate paths until `prepare`

---

## 🧠 Golden Rules for Harbor HTTPS

✔ Cert files must exist
✔ Paths must be absolute
✔ No symlinks
✔ Permissions readable
✔ `hostname` must match cert CN

---

If you want, tell me:

* Are you using **self-signed or Let’s Encrypt**
* Output of `ls -l /data/cert`
* Your **https section** in `harbor.yml` (redact secrets)

I’ll pinpoint the fix in one reply 🔧

# mkdir: cannot create directory ‘cert’: Permission denied

That’s expected — you’re hitting a **Linux permission issue**, not a Harbor issue 👍

You’re trying to create the cert directory in a location that **requires root privileges**.

---

## ✅ Fix: create the directory as root

If you’re following the Harbor docs, the cert path is **`/data/cert`**, which is **root-owned**.

Run:

```bash
sudo mkdir -p /data/cert
```

Then verify:

```bash
ls -ld /data /data/cert
```

You should see something like:

```
drwxr-xr-x root root /data
drwxr-xr-x root root /data/cert
```

---

## ✅ Copy / create certs correctly

### If you already have certs:

```bash
sudo cp harbor.crt harbor.key ca.crt /data/cert/
```

### Fix permissions (important):

```bash
sudo chmod 600 /data/cert/harbor.key
sudo chmod 644 /data/cert/harbor.crt
sudo chmod 644 /data/cert/ca.crt
sudo chown root:root /data/cert/*
```

---

## ✅ Update `harbor.yml`

Make sure it matches exactly:

```yaml
https:
  port: 443
  certificate: /data/cert/harbor.crt
  private_key: /data/cert/harbor.key
```

⚠️ Paths must be **absolute** and **exist**.

---

## 🧹 Clean failed attempt (recommended)

From your Harbor directory:

```bash
sudo docker compose down -v
sudo rm -rf /data/harbor/*
```

Then:

```bash
./prepare
./install.sh
```

---

## 🧠 Why this happens (especially on WSL)

* `/data` is **outside your home directory**
* Only root can create directories there
* Harbor expects certs in `/data` by default

---

## ✅ Quick sanity check before rerunning `prepare`

Run:

```bash
sudo ls -l /data/cert/harbor.crt /data/cert/harbor.key
```

If both files show up → you’re good to go.

---

If you want, paste:

* Your `https:` block from `harbor.yml`
* Output of `ls -l /data/cert`

I’ll verify everything before you rerun `install.sh` 🔐🚀


# Issue sudo install

Good, this error is **very specific** and easy to fix 👍
You’re hitting a **file ownership / permission mismatch** caused by running Harbor commands with **mixed sudo and non-sudo**.

---

## 🔴 What’s happening

```
open /home/jmlromero/harbor/common/config/core/env: permission denied
```

Harbor is trying to write to:

```
/home/jmlromero/harbor/common/config/...
```

But those files are now **owned by root**, because at some point you ran:

* `sudo ./prepare`
* or `sudo ./install.sh`

After that:

* Running `./prepare` **without sudo** → permission denied
* Running it **with sudo** → later causes other issues

---

## ✅ Correct Fix (clean & safe)

### 1️⃣ Reset ownership to your user

Run this **once**:

```bash
sudo chown -R jmlromero:jmlromero /home/jmlromero/harbor
```

Then verify:

```bash
ls -ld /home/jmlromero/harbor/common/config/core
```

Owner should be `jmlromero`.

---

### 2️⃣ Decide ONE way to run Harbor commands

You have **two valid options** — pick **ONE** and stick to it.

---

### ✅ OPTION A (RECOMMENDED): Run Harbor as your user

This is the **official and safest way**, especially on WSL.

```bash
cd /home/jmlromero/harbor
./prepare
./install.sh
```

✔ Cert files stay in `/data`
✔ No permission conflicts
✔ Easier upgrades

---

### ⚠️ OPTION B: Run Harbor fully as root (not recommended)

If you insist:

```bash
sudo chown -R root:root /home/jmlromero/harbor
sudo ./prepare
sudo ./install.sh
```

But **never mix** sudo and non-sudo again.

---

## 🧹 If problems persist (full reset)

If you want a **guaranteed clean state**:

```bash
sudo docker compose down -v
sudo rm -rf /data/harbor
sudo rm -rf /home/jmlromero/harbor/common
sudo chown -R jmlromero:jmlromero /home/jmlromero/harbor
```

Then rerun:

```bash
./prepare
./install.sh
```

---

## 🧠 Golden Rule for Harbor

> **Run prepare/install the same way every time**
> Mixing sudo = broken permissions

---

## ✅ Final sanity check

Before rerunning `prepare`, confirm:

```bash
whoami
ls -l common/config
```

You should be your user, and files writable.

---

If you want, paste:

* Output of `ls -l common/config/core`
* Tell me if you’re on **WSL + Docker Desktop** (likely)

I’ll confirm you’re 100% clean before proceeding 🚀

