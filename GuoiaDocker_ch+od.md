# 🚀 Infraestructura Docker: Chatwoot + Odoo + Zammad

Guía completa para desplegar:

- Servidor 1: Chatwoot + Odoo
- Servidor 2: Zammad

Optimizado para laboratorio o pequeño entorno productivo.

---

# 🖥️ SERVIDOR 1 — Chatwoot + Odoo

## ✅ Requisitos recomendados

- Ubuntu 22.04 LTS
- 6 GB RAM mínimo (5 GB funciona pero justo)
- 60 GB disco mínimo
- 2 vCPU

---

# 🔧 1. Instalar Docker

```bash
sudo apt update
sudo apt install docker.io docker-compose-plugin git -y
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER
````

Cerrar sesión y volver a entrar.

---

# 📁 2. Crear estructura

```bash
sudo mkdir -p /srv/apps
cd /srv/apps
nano docker-compose.yml
```

---

# 🧩 3. docker-compose.yml

```yaml
services:

  postgres:
    image: pgvector/pgvector:pg15
    restart: unless-stopped
    environment:
      POSTGRES_USER: odoo
      POSTGRES_PASSWORD: odoopass
      POSTGRES_DB: postgres
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
      SECRET_KEY_BASE: "CAMBIAR_POR_CLAVE_LARGA_Y_SEGURA"
      FRONTEND_URL: "http://IP_SERVIDOR:3000"
      REDIS_URL: redis://redis:6379
      POSTGRES_HOST: postgres
      POSTGRES_USERNAME: odoo
      POSTGRES_PASSWORD: odoopass
      POSTGRES_DATABASE: chatwoot
    ports:
      - "3000:3000"

  odoo:
    image: odoo:17
    restart: unless-stopped
    depends_on:
      - postgres
    ports:
      - "8069:8069"
    environment:
      HOST: postgres
      USER: odoo
      PASSWORD: odoopass
      DB_NAME: odoo

volumes:
  postgres_data:
```

---

# 🚀 4. Levantar servicios

```bash
docker compose up -d
```

Esperar 1 minuto.

---

# 🧠 5. Inicializar base de Odoo (OBLIGATORIO)

```bash
docker exec -it apps-odoo-1 \
odoo -d odoo -i base \
--db_host=postgres \
--db_user=odoo \
--db_password=odoopass \
--stop-after-init
```

Luego:

```bash
docker compose restart odoo
```

---

# 🌐 Accesos

* Chatwoot → http://IP_SERVIDOR:3000
* Odoo → http://IP_SERVIDOR:8069

---

# 🔑 Credenciales Base

PostgreSQL:

* Usuario: `odoo`
* Password: `odoopass`

Chatwoot:

* Se crea admin en primer inicio

Odoo:

* Crear base y usuario admin al acceder

---

# 🛠️ Problemas Comunes

## ❌ Error 500 en Odoo

Base no inicializada.

Solución:

```bash
odoo -d odoo -i base --stop-after-init
```

---

## ❌ password authentication failed

Las credenciales no coinciden entre servicios.

Verificar:

* POSTGRES_USER
* POSTGRES_PASSWORD
* Variables en Odoo

---

## ❌ Odoo usa base chatwoot

Falta variable:

```
DB_NAME: odoo
```

---

---



Archivo PID bloqueado después de reiniciar, solución: 

docker compose down 

docker compose up -d --force-recreate 
