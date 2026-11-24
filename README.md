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
