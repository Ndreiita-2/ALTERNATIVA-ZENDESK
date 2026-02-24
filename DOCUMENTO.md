# ALTERNATIVA A ZENDESK – STACK OPEN SOURCE

## 1. Objetivo

Diseñar una solución open-source que permita reemplazar Zendesk integrando:

* Web
* Email
* Redes sociales (Instagram, Facebook, TikTok)
* Chat en vivo
* Telefonía (Asterisk)
* Integración con Odoo
* Escalabilidad futura

---

# 2. Software Analizado

---

# Chatwoot

## Descripción

Chatwoot es una plataforma omnicanal open-source orientada a la gestión de conversaciones desde múltiples canales en una sola interfaz.

Cubre:

* Instagram
* Facebook Messenger
* WhatsApp
* Chat web
* Email
* API REST

## Ventajas

* Interfaz moderna
* Integraciones sociales sólidas
* API bien documentada
* Docker oficial estable

## Limitaciones

* Ticketing menos avanzado que Zammad
* No incluye telefonía avanzada nativa

---

## Instalación Chatwoot (Docker – Producción)

### Clonar repositorio

```bash
git clone https://github.com/chatwoot/chatwoot.git
cd chatwoot
```

### Configurar entorno

```bash
cp .env.example .env
```

Editar `.env`:

```env
POSTGRES_PASSWORD=TuPasswordSeguro
REDIS_PASSWORD=
```

### Configurar PostgreSQL en docker-compose.production.yaml

```yaml
postgres:
  image: pgvector/pgvector:pg16
  restart: always
  ports:
    - '127.0.0.1:5432:5432'
  volumes:
    - postgres_data:/var/lib/postgresql/data
  environment:
    - POSTGRES_DB=chatwoot
    - POSTGRES_USER=postgres
    - POSTGRES_PASSWORD=TuPasswordSeguro
```

La contraseña debe coincidir con `.env`.

---

### Levantar servicios

```bash
docker compose -f docker-compose.production.yaml up -d
```

---

### Problema: PostgreSQL se reiniciaba

Error:

```
Database is uninitialized and superuser password is not specified
```

Solución:

```bash
docker compose -f docker-compose.production.yaml down
docker volume rm chatwoot_postgres_data
docker compose -f docker-compose.production.yaml up -d
```

---

### Problema: Sidekiq se reiniciaba

Solución:

```bash
docker compose -f docker-compose.production.yaml run --rm rails bundle exec rails db:chatwoot_prepare
docker compose -f docker-compose.production.yaml restart
```

---

### Acceso

```
http://IP_SERVIDOR:3000
```

Se recomienda usar Nginx para exponer en puerto 80.

---

# Zammad

## Descripción

Zammad es un sistema de ticketing open-source robusto orientado a soporte estructurado y gestión avanzada de incidencias.

Cubre:

* Email
* Tickets
* SLA
* Automatizaciones
* Chat web básico

## Ventajas

* Ticketing profesional
* Gestión SLA
* Integración IMAP/SMTP estable
* Automatización avanzada

## Limitaciones

* Omnicanal más limitado que Chatwoot
* Interfaz menos moderna

---

## Instalación Zammad (Ubuntu)

```bash
wget -qO - https://dl.packager.io/srv/zammad/zammad/key | sudo apt-key add -
sudo wget -O /etc/apt/sources.list.d/zammad.list \
https://dl.packager.io/srv/zammad/zammad/stable/installer/ubuntu/24.04.repo
sudo apt update
sudo apt install zammad -y
```

### Acceso

```
http://IP_SERVIDOR
```

---

## Configuración Microsoft 365

### IMAP

```
Host: outlook.office365.com
Puerto: 993
SSL: Activado
```

### SMTP

```
Host: smtp.office365.com
Puerto: 587
TLS: Activado
```

---

## Activar Chat Web

Ruta:

```
Admin → Channels → Chat
```

Script básico:

```html
<script src="http://IP/assets/chat/chat.js"></script>
```

---

# FreeScout

## Descripción

FreeScout es una alternativa ligera a HelpScout enfocada principalmente en gestión de email.

Cubre:

* Email
* Tickets básicos
* Soporte multiusuario

## Ventajas

* Muy ligero
* Bajo consumo de recursos
* Fácil instalación

## Limitaciones

* No es omnicanal
* Funcionalidad inferior a Zammad

---

## Instalación FreeScout

### Instalar dependencias

```bash
sudo apt install nginx mysql-server git
sudo apt install php php-fpm php-mysql php-mbstring php-xml php-imap php-zip php-gd php-curl php-intl
```

### Crear base de datos

```sql
CREATE DATABASE freescout;
CREATE USER 'freescout'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON freescout.* TO 'freescout'@'localhost';
```

### Clonar aplicación

```bash
git clone https://github.com/freescout-help-desk/freescout /var/www/freescout
```

### Instalación vía navegador

```
http://IP_SERVIDOR/install
```

---

# Odoo

## Descripción

Odoo es un ERP open-source completo que incluye CRM, facturación, inventario y múltiples módulos empresariales.

Cubre:

* CRM
* Facturación
* Inventario
* Proyectos
* Helpdesk básico (módulo)

## Ventajas

* ERP completo
* CRM potente
* Escalable
* Modular

## Limitaciones

* No es un sistema de soporte puro
* Requiere más recursos

---

## Instalación Odoo (Repositorio Oficial)

### Agregar repositorio

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://nightly.odoo.com/odoo.key | sudo gpg --dearmor -o /etc/apt/keyrings/odoo.gpg
echo "deb [signed-by=/etc/apt/keyrings/odoo.gpg] https://nightly.odoo.com/17.0/nightly/deb/ ./" | sudo tee /etc/apt/sources.list.d/odoo.list
```

### Instalar

```bash
sudo apt update
sudo apt install odoo -y
```

### Acceso

```
http://IP_SERVIDOR:8069
```

Se recomienda configurar Nginx para exponer por puerto 80.

---

# Arquitectura Recomendada

## Opción Profesional (Separada)

Servidor 1:

* Chatwoot (omnicanal)
* Zammad (ticketing)

Servidor 2:

* Odoo (ERP / CRM)

Servidor 3:

* Asterisk (telefonía)

Ventajas:

* Mejor rendimiento
* Mayor seguridad
* Escalabilidad futura

---

## Opción Todo en Uno

Un solo servidor con:

* Docker para Chatwoot
* Docker para Odoo
* Zammad nativo
* Nginx como reverse proxy

Ventajas:

* Menor coste
* Más simple de mantener

Desventajas:

* Mayor riesgo ante fallo
* Menor escalabilidad

---

# Conclusión

Para reemplazar Zendesk con una solución open-source completa:

* Chatwoot → Omnicanal
* Zammad → Ticketing profesional
* Odoo → CRM / ERP
* Asterisk → Telefonía

Ninguna herramienta individual reemplaza Zendesk completamente,
pero la combinación adecuada ofrece mayor flexibilidad y control.
