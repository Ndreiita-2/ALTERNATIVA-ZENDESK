Aquí tienes la guía completa actualizada para tu `README.md`, usando:

* Un solo puerto 80
* Nginx como reverse proxy
* Acceso por rutas:

  * /chatwoot
  * /odoo
  * /zammad
* Optimizado para 8 GB RAM

---

# Laboratorio Docker en Windows

Chatwoot + Odoo + Zammad
Puerto 80 único con Nginx

---

## 1. Requisitos

* Windows 10/11
* 8 GB RAM mínimo
* Virtualización activada en BIOS
* WSL2 habilitado
* Docker Desktop instalado

Tecnologías utilizadas:

* Docker Desktop
* Chatwoot
* Odoo
* Zammad
* Nginx

---

## 2. Activar WSL2

Abrir PowerShell como Administrador y ejecutar:

```powershell
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
```

Reiniciar el equipo.

Después ejecutar:

```powershell
wsl --set-default-version 2
```

---

## 3. Instalar Docker Desktop

Descargar desde:

[https://www.docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop)

Durante la instalación:

* Activar “Use WSL 2 instead of Hyper-V”

Verificar:

```powershell
docker --version
docker compose version
```

---

## 4. Crear estructura del laboratorio

Crear carpeta principal:

```powershell
mkdir C:\laboratorio
cd C:\laboratorio
```

Crear carpeta para Nginx:

```powershell
mkdir nginx
```

Estructura final:

```
C:\laboratorio
│
├── docker-compose.yml
└── nginx
    └── default.conf
```

---

## 5. docker-compose.yml completo

Crear archivo:

C:\laboratorio\docker-compose.yml

Contenido:

```yaml
version: "3.9"

services:

  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - chatwoot
      - odoo
      - zammad

  postgres_chatwoot:
    image: postgres:13
    environment:
      POSTGRES_USER: chatwoot
      POSTGRES_PASSWORD: chatwoot
      POSTGRES_DB: chatwoot_production
    volumes:
      - chatwoot_db:/var/lib/postgresql/data

  redis:
    image: redis:7

  chatwoot:
    image: chatwoot/chatwoot:latest
    environment:
      RAILS_ENV: production
      DATABASE_URL: postgres://chatwoot:chatwoot@postgres_chatwoot:5432/chatwoot_production
      REDIS_URL: redis://redis:6379
    depends_on:
      - postgres_chatwoot
      - redis

  postgres_odoo:
    image: postgres:15
    environment:
      POSTGRES_USER: odoo
      POSTGRES_PASSWORD: odoo
      POSTGRES_DB: postgres
    volumes:
      - odoo_db:/var/lib/postgresql/data

  odoo:
    image: odoo:17
    environment:
      HOST: postgres_odoo
      USER: odoo
      PASSWORD: odoo
    depends_on:
      - postgres_odoo
    volumes:
      - odoo_data:/var/lib/odoo

  zammad:
    image: zammad/zammad-docker-compose:latest

volumes:
  chatwoot_db:
  odoo_db:
  odoo_data:
```

---

## 6. Configuración Nginx

Crear archivo:

C:\laboratorio\nginx\default.conf

Contenido:

```nginx
server {
    listen 80;

    location /chatwoot/ {
        proxy_pass http://chatwoot:3000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /odoo/ {
        proxy_pass http://odoo:8069/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /zammad/ {
        proxy_pass http://zammad/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

---

## 7. Levantar el laboratorio

En C:\laboratorio ejecutar:

```powershell
docker compose down
docker compose up -d
```

Verificar contenedores:

```powershell
docker ps
```

---

## 8. Acceso a los sistemas

Todos por puerto 80:

Chatwoot
[http://localhost/chatwoot](http://localhost/chatwoot)

Odoo
[http://localhost/odoo](http://localhost/odoo)

Zammad
[http://localhost/zammad](http://localhost/zammad)

---

## 9. Configuración recomendada en Docker Desktop (8GB RAM)

Ir a:

Settings → Resources

Asignar:

* Memory: 6 GB
* CPU: 3 cores

No asignar más de 6 GB.

---

## 10. Detener laboratorio

```powershell
docker compose down
```

Reiniciar:

```powershell
docker compose up -d
```

---

## 11. Consideraciones importantes

* La primera ejecución puede tardar varios minutos.
* Si el sistema se vuelve lento, no utilices los tres sistemas al mismo tiempo.
* Todos los datos quedan guardados en volúmenes Docker.
* No es posible asignar IPs 192.168.x.x individuales a cada contenedor en Docker Desktop con WSL2.

---

Si quieres, puedo prepararte ahora:

* Versión optimizada para menor consumo de RAM
* Versión con dominios locales tipo empresa.local
* Versión con HTTPS y certificados
* Arquitectura simulando entorno empresarial real
