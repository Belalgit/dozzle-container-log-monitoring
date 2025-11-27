# **Dozzle Centralized Log Viewer – Development Deployment Guide**

This project provides a **secure development-ready deployment** of **Dozzle**, a lightweight real-time log viewer for Docker.
It enables a central Dozzle UI server to read logs from multiple remote Docker hosts using a **docker-socket-proxy**, protected behind **Nginx + SSL + Basic Auth**.

---

## **🔰 Development URL**

**[https://dozzle.example.com](https://dozzle.example.com)**

---

## **🏗 Architecture Overview**

```
                           Internet (SSL + Browser)
                                      │
                            https://dozzle.example.com
                                      │
                             NGINX + Basic Auth (Server A)
                                      │
                              Dozzle UI (Docker, 8080)
                                      │
                       DOZZLE_REMOTE_HOST (Multiple Hosts)
                                      │
                          TCP 2375 (Read-only Docker API)
                                      │
                  docker-socket-proxy (Server B/Application Host)
```

### **Server Roles**

* **Server A / Dozzle Server** → Dozzle UI + Nginx + Basic Auth
* **Server B / Application Server** → docker-socket-proxy (exposes docker API read-only for logs)

---

## **📌 Step 1 — Server A Setup (Dozzle UI Server)**

### **1. Install System Packages**

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install vim zip unzip nginx curl -y
sudo systemctl enable nginx
```

### **2. Install Docker & Docker Compose**

```bash
sudo apt install docker.io -y

sudo curl -L \
"https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" \
-o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose
sudo usermod -aG docker $USER
newgrp docker
sudo systemctl start docker
sudo systemctl enable docker
```

---

## **📁 Project Directory Structure**

```
/var/www/dozzle.example.com/
│── docker-compose.yml
│── hosts.env
```

---

## **📄 hosts.env**

Example for multiple Docker servers:

```
DOZZLE_REMOTE_HOST=tcp://10.10.1.220:2375|Server-B,tcp://10.10.1.221:2375|Server-C
```

Single host:

```
DOZZLE_REMOTE_HOST=tcp://10.10.1.220:2375|Server-Test-Docker
```

---

## **📦 Docker Compose – Dozzle UI**

Download the repo **docker-compose.yml**

Run:

```bash
sudo docker-compose up -d --build
```

Check:

```
http://<server-ip>:8080
```

---

## **🌐 Nginx Reverse Proxy Configuration**

Location: `/etc/nginx/sites-available/dozzle.example.com`

```nginx
server {
    server_name dozzle.example.com;

    auth_basic "Restricted Zone";
    auth_basic_user_file /etc/nginx/.dozzle_htpasswd;

    location / {
        proxy_pass http://127.0.0.1:8080/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    error_log /var/log/nginx/dozzle_error.log;
    access_log /var/log/nginx/dozzle_access.log;
}
```

Enable site:

```bash
sudo nginx -t
sudo systemctl restart nginx
```

---

## **🔐 SSL Setup (Let’s Encrypt)**

```bash
sudo certbot --nginx -d dozzle.example.com
sudo certbot certificates
sudo systemctl restart nginx
```

---

## **🛡 Basic Authentication Setup**

### **Create htpasswd**

```bash
sudo apt install apache2-utils -y
sudo htpasswd -c /etc/nginx/.dozzle_htpasswd admin
```

### **Add more users**

```bash
sudo htpasswd /etc/nginx/.dozzle_htpasswd devops1
```

### **Delete user**

```bash
sudo htpasswd -D /etc/nginx/.dozzle_htpasswd devops1
```

---

# **📌 Step 3 — Server B / Application Server Setup (docker-socket-proxy)**

Directory:

```
/var/www/dozzle-agent/
```

### **docker-compose.yml**

```yaml
version: "3.8"

services:
  docker-agent:
    image: tecnativa/docker-socket-proxy
    container_name: dozzle-agent
    privileged: true
    environment:
      CONTAINERS: 1
      INFO: 1
      IMAGES: 1
      VERSION: 1
      EVENTS: 1
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    ports:
      - "2375:2375"
    restart: unless-stopped
```

Deploy:

```bash
sudo docker-compose up -d --build
```

Ensure security group allows:

```
TCP 2375 → only from Dozzle UI server (Server A)
```

---

## **📈 Final Verification**

* Browse: **[https://dozzle.example.com/](https://dozzle.example.com/)**
* Authenticate using Basic Auth
* Logs from multiple servers should appear

---

## **📁 Repository Contents**

```
README.md
dozzle-ui/docker-compose.yml
dozzle-ui/hosts.env
dozzle-agent/docker-compose.yml
nginx-config/dozzle.example.com
```

---

## **📝 License**

This project is for internal deployment within JatriTech.
Not intended for public redistribution.

---
