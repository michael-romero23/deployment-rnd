Setting up Harbor on Docker is a great way to manage your own private container registry. Harbor isn't just a simple image storage tool; it’s an entire ecosystem that includes vulnerability scanning, identity management, and image signing.

Because Harbor consists of several microservices (Postgres, Redis, Nginx, etc.), it is almost always deployed using **Docker Compose**.

---

## 🛠 Prerequisites

Before you start, ensure your environment meets these requirements:

* **OS:** Linux (Ubuntu or CentOS recommended).
* **Docker:** Engine version 17.06.0+ or higher.
* **Docker Compose:** Version 1.18.0+ or v2.
* **Hardware:** Minimum 2 CPUs, 4GB RAM, and 40GB storage.

---

## 🚀 Step-by-Step Setup

### 1. Download the Harbor Installer

Go to the [Harbor Releases page](https://github.com/goharbor/harbor/releases) and grab the "online installer" or "offline installer" (offline is larger but easier if your server has slow internet).

```bash
# Download the online installer (example for v2.10.0)
wget https://github.com/goharbor/harbor/releases/download/v2.10.0/harbor-online-installer-v2.10.0.tgz

# Extract it
tar xzvf harbor-online-installer-v2.10.0.tgz
cd harbor

```

### 2. Configure Harbor (`harbor.yml`)

The package includes a template file. You need to copy and edit it to define your hostname and access settings.

```bash
cp harbor.yml.tmpl harbor.yml
nano harbor.yml

```

**Key changes to make:**

* **hostname:** Set this to your server's IP or domain (e.g., `harbor.mycompany.com`).
* **http:** Ensure the port is `80`.
* **https (Optional):** If you don't have SSL certificates yet, you can comment out the entire `https` section to use plain HTTP for testing. *Note: Docker requires special configuration to push to "insecure" HTTP registries.*
* **harbor_admin_password:** Change this from the default `Harbor12345`.

### 3. Run the Installation Script

Harbor provides a `prepare` script that generates the `docker-compose.yml` file based on your `harbor.yml`, and an `install.sh` script to pull images and start the services.

```bash
# Run the installer (you can add --with-trivy for vulnerability scanning)
sudo ./install.sh --with-trivy

```

### 4. Verify the Setup

Once the script finishes, you should see a message saying "Harbor has been installed and started successfully." You can verify the running containers:

```bash
docker compose ps

```

---

## 🔓 Accessing Harbor

1. Open your browser and go to `http://your-server-ip`.
2. Log in with username **admin** and the password you set in `harbor.yml`.
3. To push an image from your terminal:
```bash
docker login your-server-ip
docker tag my-image:latest your-server-ip/library/my-image:latest
docker push your-server-ip/library/my-image:latest

```



> **Pro Tip:** If you didn't set up HTTPS, your `docker login` will fail unless you add your Harbor IP to the `insecure-registries` list in `/etc/docker/daemon.json` on your local machine.

Would you like me to show you how to configure the `insecure-registries` setting or how to generate self-signed SSL certificates for a production-like setup?


That error is a classic "caught in the middle" moment. It happens because Harbor’s configuration template defaults to **HTTPS**, but you haven't provided the paths to your SSL certificate and private key.

You have two ways to fix this depending on whether you want a quick test environment or a secure setup.

---

### Option 1: The Quick Fix (Disable HTTPS)

If you are just testing Harbor locally or on a private network and don't want to deal with SSL yet, you can disable the HTTPS check entirely.

1. Open your `harbor.yml` file.
2. **Comment out** the entire `https` section by adding a `#` to the start of each line.
3. Ensure the `port` under the `http` section is set to `80`.

```yaml
# http related config
http:
  port: 80

# https related config
# https:              <-- Comment this out
#   port: 443         <-- Comment this out
#   certificate: ...  <-- Comment this out
#   private_key: ...  <-- Comment this out

```

4. Run the installer again: `sudo ./install.sh`

---

### Option 2: The Right Way (Configure SSL)

If you intend to use this for production or want to avoid "Insecure Registry" errors in Docker, you need to provide the certificates.

1. **Generate or Locate your certs:** If you have files like `server.crt` and `server.key`, move them to a directory (e.g., `/data/cert/`).
2. **Update `harbor.yml`:** Point the configuration to those specific file paths.

```yaml
https:
  port: 443
  certificate: /data/cert/server.crt
  private_key: /data/cert/server.key

```

3. **Run the installer:** `sudo ./install.sh`

---

### Why this happens

Harbor's internal configuration validator checks your `harbor.yml` before it allows the installation to proceed. If it sees the `https:` key is active, it treats `certificate` and `private_key` as **mandatory fields**.

### ⚠️ A Note on "Insecure Registries"

If you choose **Option 1**, Docker will refuse to connect to your Harbor instance by default because it's not encrypted. You will need to edit `/etc/docker/daemon.json` on your client machine:

```json
{
  "insecure-registries" : ["your-harbor-ip-or-domain"]
}

```

Then restart Docker: `sudo systemctl restart docker`.

**Which path would you like to take? I can help you generate a self-signed certificate if you want to keep HTTPS enabled.**