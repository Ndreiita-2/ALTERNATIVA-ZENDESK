Aquí tienes la guía lista para copiar y pegar en tu `README.md` (formato Markdown limpio y estructurado).

---

# Laboratorio Docker en Windows

Chatwoot + Odoo + Zammad

## 1. Requisitos

* Windows 10/11
* 8 GB RAM mínimo
* Virtualización activada en BIOS
* WSL2 habilitado

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

* Activar opción **Use WSL 2 instead of Hyper-V**

Verificar instalación:

```powershell
docker --version
docker compose version
```

---

## 4. Crear estructura del laboratorio

Crear carpeta:

```powershell
mkdir C:\laboratorio
cd C:\laboratorio
```

---

## 5. Crear docker-compose.yml

Crear archivo:

```
C:\laboratorio\docker-compose.yml
```

Contenido:

```yaml
version: "3.9"

services:

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
    depends_on:
      - postgres_chatwoot
      - redis
    ports:
      - "3000:3000"
    environment:
      RAILS_ENV: production
      DATABASE_URL: postgres://chatwoot:chatwoot@postgres_chatwoot:5432/chatwoot_production
      REDIS_URL: redis://redis:6379

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
    depends_on:
      - postgres_odoo
    ports:
      - "8069:8069"
    environment:
      HOST: postgres_odoo
      USER: odoo
      PASSWORD: odoo
    volumes:
      - odoo_data:/var/lib/odoo

  zammad:
    image: zammad/zammad-docker-compose:latest
    ports:
      - "8080:80"

volumes:
  chatwoot_db:
  odoo_db:
  odoo_data:
```

---

## 6. Levantar el laboratorio

En la carpeta `C:\laboratorio` ejecutar:

```powershell
docker compose up -d
```

Verificar contenedores:

```powershell
docker ps
```

---

## 7. Acceso a los servicios

* Chatwoot: [http://localhost:3000](http://localhost:3000)
* Odoo: [http://localhost:8069](http://localhost:8069)
* Zammad: [http://localhost:8080](http://localhost:8080)

---

## 8. Configurar recursos en Docker Desktop

Ir a:

Settings → Resources

Asignar:

* Memory: 6 GB
* CPU: 3 o 4 cores

No asignar más de 6 GB si tu equipo tiene 8 GB totales.

---

## 9. Detener laboratorio

```powershell
docker compose down
```

Reiniciar:

```powershell
docker compose up -d
```

---

## 10. Notas importantes

* La primera ejecución puede tardar varios minutos.
* Si el sistema se pone lento, no uses los tres servicios al mismo tiempo.
* Los datos quedan guardados en los volúmenes Docker.

---

## Tecnologías utilizadas

* Docker Desktop
* Chatwoot
* Odoo
* Zammad

---

Si quieres, puedo prepararte una versión optimizada para bajo consumo de memoria o una versión tipo entorno productivo con Nginx reverse proxy.
