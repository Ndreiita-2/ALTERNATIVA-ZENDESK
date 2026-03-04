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

```
http://192.168.136.120
```

---

## 🌍 Exponer Zammad con Ngrok

Proyecto: **ngrok**

Instalar:

```bash
curl -s https://ngrok-agent.s3.amazonaws.com/ngrok.asc | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null
echo "deb https://ngrok-agent.s3.amazonaws.com buster main" | sudo tee /etc/apt/sources.list.d/ngrok.list
sudo apt update
sudo apt install ngrok -y
```

Configurar token:

```bash
ngrok config add-authtoken TU_TOKEN_AQUI
```

Exponer:

```bash
ngrok http 80
```

Acceso remoto:

```
https://abcd-1234.ngrok-free.app
```

Contraseña: adminAA123?

---

# 🚀 INSTALACIÓN CHATWOOT (DOCKER)

Servidor: **192.168.136.121**

Proyecto oficial: **Chatwoot**

---

## 1️⃣ Instalar Docker

```bash
sudo apt update
sudo apt install ca-certificates curl gnupg -y
```

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu noble stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
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
    image: chatwoot/chatwoot:latest
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

```
http://192.168.136.121:3000
```

# 🔐 CAMBIAR CONTRASEÑA DEL USUARIO ADMIN EN CHATWOOT

## 1️⃣ Ir al directorio

```bash
cd /srv/chatwoot
```

---

## 2️⃣ Entrar a la consola Rails

```bash
docker compose exec chatwoot bundle exec rails c
```

---

## 3️⃣ Verificar usuario existente

```ruby
User.pluck(:email)
```

Debe mostrar:

```
["andrea.chamorro@creatorsco.com"]
```

---

## 4️⃣ Cambiar contraseña

```ruby
user = User.find_by(email: "andrea.chamorro@creatorsco.com")
user.password = "Pruebas1?"
user.password_confirmation = "Pruebas1?"
user.save!
```

---

# ✅ Ahora puedes entrar con:

Email:

```
andrea.chamorro@creatorsco.com
```

Contraseña:

```
Pruebas1?
```

Archivo PID bloqueado después de reiniciar, solución:
```
docker compose down
```
```
docker compose up -d --force-recreate
```
---

# 🧩 RESULTADO FINAL DEL LAB

| Servicio | IP              | Acceso                                                               |
| -------- | --------------- | -------------------------------------------------------------------- |
| Zammad   | 192.168.136.120 | [https://polygalaceous-alaysia-unsilently.ngrok-free.dev/](https://polygalaceous-alaysia-unsilently.ngrok-free.dev/) |
| Chatwoot | 192.168.136.121 | [http://192.168.136.121:3000/](http://192.168.136.121:3000/) |
