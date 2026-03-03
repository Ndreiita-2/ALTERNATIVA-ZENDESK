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

```
http://192.168.136.121:3000
```

# 🌍 EXPONER CHATWOOT CON CLOUDLARE TUNNEL (INDEPENDIENTE)

Proyecto: **Cloudflare Tunnel**

---

## 📦 Instalar cloudflared (Ubuntu 24.04 / 22.04)

Eliminar repos anteriores si existen:

```bash
sudo rm -f /etc/apt/sources.list.d/cloudflare.list
sudo apt update
```

Descargar paquete oficial:

```bash
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
```

Instalar:

```bash
sudo dpkg -i cloudflared-linux-amd64.deb
```

Si hay dependencias pendientes:

```bash
sudo apt -f install -y
```

Verificar:

```bash
cloudflared --version
```

---

## 🔐 Login (opcional para túnel rápido no persistente no es necesario)

```bash
cloudflared tunnel login
```

---

## 🚀 Exponer Chatwoot

En el servidor donde está Chatwoot:

```bash
cloudflared tunnel --url http://localhost:3000
```

Generará algo como:

```
https://random-name.trycloudflare.com
```

ENESPERA

Reemplaza **todo** tu bloque por este:

---

# 🌍 EXPONER CHATWOOT CON DOMINIO FIJO (CLOUDFLARE GRATIS)

Proyecto: **Cloudflare Tunnel**

---

## 📦 Instalar cloudflared (Ubuntu 24.04 / 22.04)

```bash
sudo rm -f /etc/apt/sources.list.d/cloudflare.list
sudo apt update

wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared-linux-amd64.deb
sudo apt -f install -y

cloudflared --version
```

---

## 🔐 1️⃣ Login en Cloudflare

```bash
cloudflared tunnel login
```

Autoriza tu dominio en el navegador.

---

## 🧱 2️⃣ Crear túnel con nombre

```bash
cloudflared tunnel create chatwoot
```

Guarda el **Tunnel ID** que aparece.

---

## ⚙️ 3️⃣ Crear configuración

```bash
sudo nano /etc/cloudflared/config.yml
```

Contenido:

```yaml
tunnel: chatwoot
credentials-file: /home/USUARIO/.cloudflared/TUNNEL-ID.json

ingress:
  - hostname: chat.tudominio.com
    service: http://localhost:3000
  - service: http_status:404
```

Reemplazar:

* `USUARIO` por tu usuario Linux
* `TUNNEL-ID` por el ID generado
* `chat.tudominio.com` por tu subdominio real

---

## 🌐 4️⃣ Crear DNS automático

```bash
cloudflared tunnel route dns chatwoot chat.tudominio.com
```

---

## 🚀 5️⃣ Ejecutar túnel

```bash
cloudflared tunnel run chatwoot
```

Probar acceso:

```
https://chat.tudominio.com
```

---

## 🔄 6️⃣ Dejar como servicio permanente

```bash
sudo cloudflared service install
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
```

---

## 🔧 7️⃣ Ajustar Chatwoot

```bash
cd /srv/chatwoot
nano .env
```

Modificar:

```
FRONTEND_URL=https://chat.tudominio.com
```

Reiniciar:

```bash
docker compose down
docker compose up -d
```

---

## ✅ RESULTADO FINAL

Chatwoot disponible en:

```
https://chat.tudominio.com
```

URL fija
HTTPS automático
Gratis
Persistente
Sin depender de terminal abierta


ENESPERA

---

# 🔗 ACCESO REMOTO FINAL

Zammad:

```
https://abcd-1234.ngrok-free.app
```

Chatwoot:

```
https://random.trycloudflare.com
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
["admin@chatwoot.local"]
```

---

## 4️⃣ Cambiar contraseña

```ruby
user = User.find_by(email: "admin@chatwoot.local")
user.password = "NuevaPassword123!"
user.password_confirmation = "NuevaPassword123!"
user.save!
```

---

# ✅ Ahora puedes entrar con:

Email:

```
admin@chatwoot.local
```

Contraseña:

```
NuevaPassword123!
```

---

# 🧩 RESULTADO FINAL DEL LAB

| Servicio | IP              | Acceso público HTTPS                                                 |
| -------- | --------------- | -------------------------------------------------------------------- |
| Zammad   | 192.168.136.120 | [https://abcd-1234.ngrok-free.app](https://abcd-1234.ngrok-free.app) |
| Chatwoot | 192.168.136.121 | [https://random.trycloudflare.com](https://random.trycloudflare.com) |
