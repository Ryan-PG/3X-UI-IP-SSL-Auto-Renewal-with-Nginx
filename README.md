<div align="right">
  <a href="README.fa.md">🇮🇷 نسخه فارسی (Persian)</a>
</div>

# 🔐 3X-UI IP SSL Auto-Renewal with Nginx

<p align="center">
  <strong>Automated Let's Encrypt IP Certificates Without Stopping Nginx</strong>
</p>
<p align="center">
  <img src="https://img.shields.io/badge/3X--UI-Compatible-blue?style=flat-square" alt="3X-UI"/>
  <img src="https://img.shields.io/badge/Nginx-Reverse%20Proxy-green?style=flat-square" alt="Nginx"/>
  <img src="https://img.shields.io/badge/SSL-Let's%20Encrypt-brightgreen?style=flat-square" alt="Let's Encrypt"/>
  <img src="https://img.shields.io/badge/Certificate-Short--Lived-orange?style=flat-square" alt="Short-lived certificate"/>
</p>

---

## 📖 Overview

This guide explains how to obtain and automatically renew Let's Encrypt SSL certificates for public IP addresses using the built-in SSL Management feature of **3X-UI**, without stopping **Nginx** or interrupting other applications running on ports `80` and `443`.

---

## ⚠️ The Problem

3X-UI uses `acme.sh` in **Standalone mode** to obtain IP SSL certificates. Standalone mode requires an available HTTP port. However, if Nginx is already listening on port `80`, certificate issuance fails:

```text
tcp port 80 is already used by nginx
Please stop it first

```

Stopping Nginx is not a practical solution when multiple applications depend on it.

---

## 💡 The Solution

Instead of stopping Nginx, we configure it to forward ACME `HTTP-01` challenge requests to a separate local port where `acme.sh` runs. This allows both services to operate simultaneously without downtime.

---

## 🏗️ Architecture

```text
                       Internet
                          |
                          |
                    Public Port 80
                          |
                          v
                   +-------------+
                   |    Nginx    |
                   |   Port 80   |
                   +-------------+
                          |
                          |
              /.well-known/acme-challenge/
                          |
                          v
                   +-------------+
                   | 127.0.0.1   |
                   |   :18080    |
                   |             |
                   |   acme.sh   |
                   | Standalone  |
                   +-------------+
                          |
                          v
                    Let's Encrypt

```

| Component | Responsibility |
| --- | --- |
| **Nginx** | Handles public HTTP traffic on port `80` |
| **acme.sh** | Handles ACME HTTP-01 validation on port `18080` |
| **3X-UI** | Manages certificate issuance and renewal |
| **Let's Encrypt** | Issues and validates SSL certificates |

### How It Works

1. Let's Encrypt sends an HTTP-01 validation request to port `80`.
2. Nginx receives the request.
3. Nginx forwards the ACME challenge path to `127.0.0.1:18080`.
4. `acme.sh` responds to the validation request.
5. Let's Encrypt validates the server's IP ownership and issues the certificate.

> Normal HTTP traffic continues to be handled by your existing Nginx configuration.

---

## 📋 Requirements

Before starting, make sure you have:

* A Linux server with a public IPv4 address.
* 3X-UI installed and running.
* Nginx installed and listening on port `80`.
* Root or `sudo` access.
* Public TCP port `80` accessible from the internet.
* An available local TCP port for the ACME listener.

> This guide uses **`18080`** as the dedicated ACME listener port. You can choose another available port if necessary.

### 📌 About Let's Encrypt IP Certificates

Let's Encrypt supports certificates for public IP addresses using its **shortlived** certificate profile. These certificates have an approximately **160-hour (6-day and 16-hour)** validity period. Automatic renewal is essential to avoid certificate expiration.

---

## 🔍 Step 1: Check Existing Port Usage

Inspect the ports currently in use:

```bash
sudo ss -lntp | grep -E ':(80|443|8080|18080)\b'

```

**Example output:**

```text
LISTEN 0 511 0.0.0.0:80   0.0.0.0:* users:(("nginx",pid=29744,fd=7))
LISTEN 0 511 0.0.0.0:443  0.0.0.0:* users:(("nginx",pid=29744,fd=8))
LISTEN 0 8192 0.0.0.0:8080 0.0.0.0:* users:(("docker-proxy",pid=2030,fd=8))

```

In this example:

* Port `80` is occupied by Nginx.
* Port `443` is occupied by Nginx.
* Port `8080` is occupied by Docker.

Check whether port `18080` is available:

```bash
sudo ss -lntp | grep ':18080'

```

If there is no output, the port is currently free. **Do not stop existing services just to free a port.** Choose another available port instead.

---

## ⚙️ Step 2: Configure Nginx

Nginx must forward ACME challenge requests to the dedicated ACME listener port. Find the Nginx configuration responsible for handling requests sent directly to your server IP.

**Common configuration locations:**

* `/etc/nginx/sites-enabled/`
* `/etc/nginx/conf.d/`

To inspect the active configuration:

```bash
sudo nginx -T

```

Inside the appropriate HTTP `server` block listening on port `80`, add:

```nginx
location ^~ /.well-known/acme-challenge/ {
    proxy_pass [http://127.0.0.1:18080](http://127.0.0.1:18080);

    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}

```

### Example Configuration

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    server_name _;

    location ^~ /.well-known/acme-challenge/ {
        proxy_pass [http://127.0.0.1:18080](http://127.0.0.1:18080);

        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location / {
        # Keep your existing application routing here.
    }
}

```

**Important Notes:**

* If you already have a `default_server` block, modify it instead of creating another conflicting `default_server`.
* The ACME location must be accessible **without** HTTP-to-HTTPS redirection.
* Make sure the `server` block handles requests addressed directly to your public IP.
* Keep your existing application routing intact.

### Validate and Reload Nginx

```bash
sudo nginx -t

```

If the configuration is valid:

```bash
sudo systemctl reload nginx

```

Nginx will continue running without interrupting existing applications.

---

## 🔐 Step 3: Obtain the IP SSL Certificate

1. Open the 3X-UI management menu:
```bash
x-ui

```


2. Navigate to:
```text
20. SSL Certificate Management

```


3. Select:
```text
6. Get SSL for IP Address

```


4. Enter your actual public IPv4 address:
```text
Please enter your server's public IPv4 address:

```


5. When asked about IPv6, leave it empty if you do not want to include an IPv6 address.
6. Next, the script asks:
```text
Port to use for ACME HTTP-01 listener (default 80):

```


Enter:
```text
18080

```



**Expected output:**

```text
[INF] Using port 18080 to issue certificate
[INF] Port 18080 is free and ready for standalone validation.
[INF] Starting automatic SSL certificate generation...
[Tue ...] Standalone mode.

```

If everything is configured correctly, `acme.sh` starts its standalone listener on port `18080`, and Nginx forwards incoming ACME challenge requests from public port `80` to this listener.

---

## 🔄 Step 4: Verify Certificate Installation

After successful issuance, locate the certificate files. The default `acme.sh` directory is usually `/root/.acme.sh/`.

Find certificate-related files:

```bash
sudo find /root/.acme.sh -type f \( -name '*.conf' -o -name '*.cer' -o -name '*.key' \)

```

Search for the configuration associated with your public IP (replace `YOUR_SERVER_IP` with your actual IP):

```bash
sudo find /root/.acme.sh -type f -name '*.conf' | grep 'YOUR_SERVER_IP'

```

Inspect the relevant configuration:

```bash
sudo grep -E 'Le_HTTPPort|Le_ReloadCmd|Le_Webroot' /path/to/certificate.conf

```

Look for:

```text
Le_HTTPPort='18080'

```

> The exact directory structure may differ depending on the `acme.sh` installation and certificate type. Some installations use an `_ecc` subdirectory.

---

## 🔁 Step 5: Verify Automatic Renewal

Successful initial issuance does not automatically guarantee successful renewal. Verify the following three components:

### 5.1. Check the Saved ACME Port

The certificate configuration should contain:

```text
Le_HTTPPort='18080'

```

This indicates that `acme.sh` has saved the custom HTTP listener port for the certificate and will reuse it during subsequent renewal operations.

### 5.2. Check the Renewal Cron Job

Inspect the `root` user's scheduled jobs:

```bash
sudo crontab -l

```

You should find the scheduled `acme.sh` renewal command. If `acme.sh` was installed under another user, inspect that user's crontab as well. To locate the executable:

```bash
sudo find / -name acme.sh -type f 2>/dev/null

```

### 5.3. Verify Nginx Persistence

Check that the ACME challenge location remains in the active configuration:

```bash
sudo nginx -T | grep -A 10 'acme-challenge'

```

Make sure Nginx starts automatically after a reboot:

```bash
sudo systemctl is-enabled nginx

```

Expected output: `enabled`

### ⚠️ Renewal Considerations

During renewal:

1. The scheduled `acme.sh` job starts.
2. `acme.sh` uses the saved HTTP listener port (`18080`).
3. Nginx forwards the HTTP-01 challenge to that port.
4. Let's Encrypt validates the challenge.
5. `acme.sh` renews the certificate.
6. The configured certificate deployment or reload mechanism updates the consuming service.

> Make sure the certificate deployment/reload command is correctly configured for the panel or service using the certificate.

---

## 🛡️ Step 6: Firewall Configuration

The public HTTP port must remain accessible to Let's Encrypt.

| Port | Protocol | Access |
| --- | --- | --- |
| `80` | TCP | Public |
| `443` | TCP | Public (if HTTPS services are used) |
| `18080` | TCP | Local proxy access only |

**Do not expose port `18080` publicly.** Nginx connects to `127.0.0.1:18080`.

However, note that `acme.sh` Standalone may bind to `0.0.0.0` rather than just `localhost`. Verify the actual listener:

```bash
sudo ss -lntp | grep ':18080'

```

If it listens on all interfaces (`0.0.0.0`), block external access to port `18080` through your firewall while allowing local connections. There is no need to open this port in UFW, iptables, or your cloud firewall.

---

## 🧰 Troubleshooting

### ❌ Port 80 Is Already in Use

* **Error:**
```text
tcp port 80 is already used by nginx
Please stop it first

```


* **Cause:** Nginx is already listening on port `80`, so `acme.sh` cannot start its standalone listener there.
* **Solution:** Keep Nginx running, configure Nginx to forward ACME requests, use another local listener port (e.g., `18080`), and enter that port in the 3X-UI SSL Management menu.

### ❌ The Selected ACME Port Is Already in Use

* **Error:**
```text
tcp port 8080 is already used by docker-proxy
Please stop it first

```


* **Cause:** Another service, such as Docker, is already using the selected port.
* **Solution:** Find another available port (`sudo ss -lntp | grep ':18080'`), use it if free, and update the Nginx upstream accordingly.

### ❌ Let's Encrypt Cannot Validate the IP

* **Possible causes:**
* Public TCP port `80` is blocked.
* Nginx is not forwarding the challenge path.
* The wrong Nginx `server` block receives the request.
* The configured upstream port is incorrect.
* The entered IP does not match the incoming public IP.
* A firewall or network-level proxy interferes with the request.


* **Diagnostic commands:**
```bash
sudo nginx -t
sudo systemctl status nginx
sudo ss -lntp | grep -E ':(80|18080)\b'
sudo tail -n 100 /var/log/nginx/error.log

```



### ❌ Automatic Renewal Fails

Check the following:

1. The saved `Le_HTTPPort` value.
2. The `acme.sh` renewal Cron job.
3. Nginx availability.
4. The ACME challenge location in Nginx.
5. Public TCP port `80` accessibility.
6. The certificate deployment/reload command.

> Remember: successful initial issuance does not prove that automatic renewal is configured correctly.

---

## 📚 Useful Commands

| Purpose | Command |
| --- | --- |
| Check listening ports | `sudo ss -lntp` |
| Validate Nginx | `sudo nginx -t` |
| Reload Nginx | `sudo systemctl reload nginx` |
| Check Nginx status | `sudo systemctl status nginx` |
| Inspect Nginx configuration | `sudo nginx -T` |
| Check root Cron jobs | `sudo crontab -l` |
| Locate `acme.sh` | `sudo find / -name acme.sh -type f 2>/dev/null` |
| Locate ACME configurations | `sudo find /root/.acme.sh -name '*.conf'` |

---

## ✅ Final Checklist

Before considering the setup complete:

* [ ] The correct public IPv4 address is configured.
* [ ] Nginx listens on public TCP port `80`.
* [ ] The selected ACME port (`18080`) is available.
* [ ] Nginx forwards `/.well-known/acme-challenge/` to `127.0.0.1:18080`.
* [ ] Nginx configuration passes `nginx -t`.
* [ ] The IP certificate was successfully issued.
* [ ] The certificate configuration stores `Le_HTTPPort='18080'`.
* [ ] The `acme.sh` renewal Cron job is active.
* [ ] Nginx starts automatically after a reboot.
* [ ] External access to port `18080` is blocked.
* [ ] The certificate deployment/reload mechanism is configured.
* [ ] Other applications continue working on ports `80` and `443`.

---

## 🎯 Conclusion

With this architecture, Nginx remains responsible for public HTTP traffic, while 3X-UI and `acme.sh` handle certificate issuance and renewal through a dedicated listener port.

**The key idea:** Keep Nginx running. Forward ACME HTTP-01 challenge requests to a separate `acme.sh` listener instead of stopping your existing services. This approach allows multiple applications, reverse proxies, and Docker workloads to continue running while maintaining an automatically renewed Let's Encrypt IP SSL certificate.

> **Implementation note:** This setup assumes that the installed 3X-UI version persists the custom standalone HTTP port and reuses it during renewal. Always verify the saved `Le_HTTPPort` setting, renewal Cron job, and certificate deployment configuration after the first successful issuance.
