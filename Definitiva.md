# 🖥️ CONFIGURACIÓN DE RED (AMBOS SERVIDORES)

| Zammad                  | Chatwoot                |
| ----------------------- | ----------------------- |
| IP: **192.168.136.120** | IP: **192.168.136.121** |

Archivo netplan en ambos:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

---

## 🔹 Zammad (192.168.136.120)

```yaml
      addresses:
        - 192.168.136.120/24
      routes:
        - to: default
          via: 192.168.136.1
      nameservers:
        addresses: [8.8.8.8,1.1.1.1]
```

---

## 🔹 Chatwoot (192.168.136.121)

```yaml
      addresses:
        - 192.168.136.121/24
      routes:
        - to: default
          via: 192.168.136.1
      nameservers:
        addresses: [8.8.8.8,1.1.1.1]
```

Aplicar en ambos:

```bash
sudo netplan apply
```

---

# 🚀 INSTALACIÓN ZAMMAD (NATIVO)

Servidor: **192.168.136.120**

Proyecto oficial: **Zammad**

---

## 1️⃣ Actualizar

```bash
sudo apt update && sudo apt upgrade -y
sudo reboot
```

---

## 2️⃣ Instalar dependencias + Nginx

```bash
sudo apt install curl gnupg apt-transport-https ca-certificates lsb-release nginx postgresql redis-server -y
```

---

## 3️⃣ Instalar Elasticsearch

```bash
curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elastic.gpg

echo "deb [signed-by=/usr/share/keyrings/elastic.gpg] https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-7.x.list

sudo apt update
sudo apt install elasticsearch -y
```

Editar:

```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
```

Asegurar:

```
network.host: localhost
```

Activar:

```bash
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch
```

---

## 4️⃣ Instalar Zammad

```bash
curl -fsSL https://dl.packager.io/srv/zammad/zammad/key | sudo gpg --dearmor -o /usr/share/keyrings/zammad.gpg

echo "deb [signed-by=/usr/share/keyrings/zammad.gpg] https://dl.packager.io/srv/deb/zammad/zammad/stable/ubuntu 22.04 main" | sudo tee /etc/apt/sources.list.d/zammad.list

sudo apt update
sudo apt install zammad -y
```

Inicializar:

```bash
sudo zammad run rake db:create
sudo zammad run rake db:migrate
sudo zammad run rake db:seed
```

Reiniciar:

```bash
sudo systemctl restart zammad
```

---

## 🔐 ACCESO LOCAL ZAMMAD

Servidor local:

```
http://192.168.136.120
```

Admin (creado en primer acceso):

```
admin@zammad.local
Admin123!
```

---

## 🌍 Exponer Zammad con Ngrok

Instalar:

```bash
curl -s https://ngrok-agent.s3.amazonaws.com/ngrok.asc | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null

echo "deb https://ngrok-agent.s3.amazonaws.com buster main" | sudo tee /etc/apt/sources.list.d/ngrok.list

sudo apt update
sudo apt install ngrok -y
```

Crear cuenta en:

[https://dashboard.ngrok.com/](https://dashboard.ngrok.com/)

Configurar token:

```bash
ngrok config add-authtoken TU_TOKEN_AQUI
```

Exponer servicio:

```bash
ngrok http 80
```

---

## 🔗 ACCESO REMOTO ZAMMAD

Ngrok generará algo como:

```
https://abcd-1234.ngrok-free.app
```

Ese será el acceso HTTPS público.

---

# 🚀 INSTALACIÓN CHATWOOT (DOCKER)

Servidor: **192.168.136.121**

Proyecto oficial: **Chatwoot**

## 1️⃣ Instalar Docker + Nginx + Certbot (Ubuntu 24.04 Noble)

```bash
sudo apt update
sudo apt install ca-certificates curl gnupg nginx certbot python3-certbot-nginx -y
```

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  noble stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

```bash
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER
```

Cerrar sesión y volver a entrar.

---

## 2️⃣ Crear proyecto

```bash
sudo mkdir -p /srv/chatwoot
cd /srv/chatwoot
nano docker-compose.yml
```

---

## 3️⃣ docker-compose.yml

```yaml
services:

  postgres:
    image: pgvector/pgvector:pg15
    restart: unless-stopped
    environment:
      POSTGRES_USER: chatwoot
      POSTGRES_PASSWORD: ChatwootDB2026!
      POSTGRES_DB: chatwoot
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7
    restart: unless-stopped

  chatwoot:
    image: chatwoot/chatwoot:v3.7.0
    restart: unless-stopped
    depends_on:
      - postgres
      - redis
    command: >
      sh -c "bundle exec rails db:chatwoot_prepare &&
             bundle exec rails s -p 3000 -b 0.0.0.0"
    environment:
      RAILS_ENV: production
      SECRET_KEY_BASE: "CAMBIAR_POR_CLAVE"
      FRONTEND_URL: "http://192.168.136.121:3000"
      REDIS_URL: redis://redis:6379
      POSTGRES_HOST: postgres
      POSTGRES_USERNAME: chatwoot
      POSTGRES_PASSWORD: ChatwootDB2026!
      POSTGRES_DATABASE: chatwoot
    ports:
      - "3000:3000"

volumes:
  postgres_data:
```

Generar clave:

```bash
openssl rand -hex 64
```

Levantar:

```bash
docker compose up -d
```

---

## 🔐 ACCESO LOCAL CHATWOOT

Servidor local:

```
http://192.168.136.121:3000
```

Admin manual:

```bash
docker compose exec chatwoot bundle exec rails console
```

```
User.create!(
  name: "Admin",
  email: "admin@chatwoot.local",
  password: "Admin123!",
  password_confirmation: "Admin123!",
  confirmed_at: Time.now
)
```

---

# 🌍 EXPONER ZAMMAD Y CHATWOOT CON UN SOLO NGROK

Servidor que expone al exterior: **192.168.136.120 (Zammad)**

Proyecto utilizado: **Ngrok**

Arquitectura:

* `/` → Zammad (local 127.0.0.1:3000)
* `/chat` → Chatwoot (192.168.136.121:3000)

---

## 1️⃣ Modificar Nginx en Zammad

Editar:

```bash
sudo nano /etc/nginx/sites-available/zammad
```

Dejar el bloque `server` así:

```nginx
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /chat/ {
        proxy_pass http://192.168.136.121:3000/;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Aplicar cambios:

```bash
sudo nginx -t
sudo systemctl restart nginx
```

---

## 2️⃣ Ajustar Chatwoot para trabajar bajo /chat

En el servidor **192.168.136.121**

Editar:

```bash
cd /srv/chatwoot
nano docker-compose.yml
```

Modificar la variable:

```yaml
FRONTEND_URL: "http://192.168.136.120/chat"
```

Recrear contenedores:

```bash
docker compose down
docker compose up -d
```

---

## 3️⃣ Levantar túnel Ngrok (solo en Zammad)

En **192.168.136.120**

```bash
ngrok http 80
```

Ngrok generará algo como:

```
https://abcd-1234.ngrok-free.app
```

---

## 🔗 ACCESO REMOTO FINAL

Zammad:

```
https://abcd-1234.ngrok-free.app/
```

Chatwoot:

```
https://abcd-1234.ngrok-free.app/chat
```

---

# 🧩 RESULTADO FINAL DEL LAB

| Servicio | IP              | Acceso público HTTPS (Ngrok)                                                   |
| -------- | --------------- | ------------------------------------------------------------------------------ |
| Zammad   | 192.168.136.120 | [https://abcd-1234.ngrok-free.app](https://abcd-1234.ngrok-free.app)           |
| Chatwoot | 192.168.136.121 | [https://abcd-1234.ngrok-free.app/chat](https://abcd-1234.ngrok-free.app/chat) |

---
