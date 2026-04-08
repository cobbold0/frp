# FRP Remote Access & Tunneling Setup Guide

## Overview
This guide covers:
- FRPC setup on Windows
- Running FRPC as a service using NSSM
- FRPS setup on Ubuntu VPS
- Remote SSH access to Windows
- Exposing a local Node.js app to the internet
- Using Nginx with a domain and HTTPS

---

## Architecture

Windows Machine (FRPC) → VPS (FRPS) → Internet → Nginx → Domain

---

## PART 1 — Setup FRPC on Windows

### 1. Download FRP
https://github.com/fatedier/frp/releases

Extract to:
C:\frp\

### 2. Create frpc.toml

```
serverAddr = "YOUR_VPS_IP"
serverPort = 7000

auth.method = "token"
auth.token = "mysecuretoken123"

[[proxies]]
name = "desktop-ssh"
type = "tcp"
localIP = "127.0.0.1"
localPort = 22
remotePort = 6001

[[proxies]]
name = "node-app"
type = "tcp"
localIP = "127.0.0.1"
localPort = 3000
remotePort = 8080
```

---

## PART 2 — Run FRPC as a Service (NSSM)

Download NSSM:
https://nssm.cc/download

Install:
```
nssm install frpc
```

Path:
C:\frp\frpc.exe

Arguments:
-c C:\frp\frpc.toml

Start:
```
nssm start frpc
```

---

## PART 3 — Setup FRPS on Ubuntu VPS

```
sudo apt update
sudo apt install wget tar -y

cd /opt
wget https://github.com/fatedier/frp/releases/download/v0.58.1/frp_0.58.1_linux_amd64.tar.gz
tar -xzf frp_0.58.1_linux_amd64.tar.gz
mv frp_0.58.1_linux_amd64 frp
cd frp
```

Create frps.toml:

```
bindPort = 7000

auth.method = "token"
auth.token = "mysecuretoken123"

webServer.addr = "0.0.0.0"
webServer.port = 7500
webServer.user = "admin"
webServer.password = "admin123"
```

Run:
```
./frps -c frps.toml
```

---

## PART 4 — Run FRPS as a Service

Create:
```
sudo nano /etc/systemd/system/frps.service
```

```
[Unit]
Description=FRP Server
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/frp
ExecStart=/opt/frp/frps -c /opt/frp/frps.toml
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Enable:
```
sudo systemctl daemon-reload
sudo systemctl enable frps
sudo systemctl start frps
```

---

## PART 5 — Firewall

```
sudo ufw allow 7000/tcp
sudo ufw allow 6001/tcp
sudo ufw allow 8080/tcp
sudo ufw allow 7500/tcp
```

---

## SCENARIO 1 — SSH into Windows

```
ssh -p 6001 USER@YOUR_VPS_IP
```

Enable SSH on Windows:
```
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
```

---

## SCENARIO 2 — Access Node App

Local:
http://127.0.0.1:3000

Public:
http://YOUR_VPS_IP:8080

---

## PART 6 — Domain Setup

DNS:
app.stephenandcobbold.com → YOUR_VPS_IP

---

## PART 7 — Install Nginx

```
sudo apt install nginx -y
sudo ufw allow 'Nginx Full'
```

---

## PART 8 — Nginx Config

```
server {
    listen 80;
    server_name app.stephenandcobbold.com;

    location / {
        proxy_pass http://127.0.0.1:8080;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
    }
}
```

Enable:
```
sudo ln -s /etc/nginx/sites-available/app.stephenandcobbold.com /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## PART 9 — HTTPS

```
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d app.stephenandcobbold.com
```

---

## FINAL

Use Cases:
https://github.com/cobbold0/frp/blob/main/frp_use_cases.md

SSH:
ssh -p 6001 USER@YOUR_VPS_IP

Web:
https://app.stephenandcobbold.com
