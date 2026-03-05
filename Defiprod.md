***

# **Guía de Integración Chatwoot ↔ Zammad (Producción) — 2026**

> **Objetivo**  
> Integrar el chat web (Chatwoot) con el sistema de tickets (Zammad) para que:
>
> *   Mensajes del **cliente** en Chatwoot creen/actualicen **tickets** en Zammad.
> *   Respuestas del **agente** en Zammad (notas públicas) se envíen de vuelta a Chatwoot como mensajes del agente.
> *   Sin correos, sin firmas, sin prefijos, sin loops.

***

## 📚 Índice

1.  \#arquitectura-y-requisitos
2.  \#red-y-direcciones-ip
3.  \#servidor-zammad-192168136120
    1.  \#instalación-y-dependencias
    2.  \#elasticsearch
    3.  \#instalación-zammad
    4.  \#exposición-con-ngrok
    5.  \#ajustes-base-y-token-inválido-tras-reinicio
4.  \#servidor-chatwoot-192168136121
    1.  \#docker-y-compose
    2.  \#despliegue-chatwoot
    3.  \#acceso-y-cambio-de-contraseña
5.  \#integración-microsoft-365-imapsmtp-oauth
6.  \#middleware-nodejs-chat-integration
    1.  \#estructura-y-env
    2.  \#indexjs-versión-final
    3.  \#ejecución-logs-y-persistencia
    4.  \#servicio-systemd-producción
7.  \#nginx--reverse-proxy-para-webhooks
8.  \#webhooks-y-trigger-en-zammad
9.  \#webhook-en-chatwoot
10. \#macro-responder-al-chat-en-zammad
11. \#ajuste-notas-públicas-por-defecto
12. \#pruebas-de-extremo-a-extremo
13. \#resolución-de-incidencias
14. \#checklist-para-paso-a-producción
15. \#apéndices-docker-del-middleware-seguridad-backups

***

## Arquitectura y requisitos

**Arquitectura final**

    Cliente (Chat web) → Chatwoot → Middleware Node → Zammad
    Cliente (Chat web) ← Chatwoot ← Middleware Node ← Zammad (nota pública)

**Requisitos mínimos**

*   Ubuntu 22.04/24.04 (servidores separados).
*   Acceso sudo.
*   PostgreSQL/Redis (incluidos por los despliegues).
*   Conectividad entre servidores (puertos: 3000 para Chatwoot, 80/443 para Zammad/nginx, 4000 para middleware).
*   Opción A (laboratorio): **ngrok** para exponer Zammad/middleware.
*   Opción B (producción): dominio propio + **TLS** (Let’s Encrypt / ACME) y firewall.

***

## Red y direcciones IP

**Zammad**: `192.168.136.120`  
**Chatwoot**: `192.168.136.121`

Configurar Netplan en ambos:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

**Zammad**

```yaml
addresses: [192.168.136.120/24]
routes:
  - to: default
    via: 192.168.136.1
nameservers:
  addresses: [8.8.8.8,1.1.1.1]
```

**Chatwoot**

```yaml
addresses: [192.168.136.121/24]
routes:
  - to: default
    via: 192.168.136.1
nameservers:
  addresses: [8.8.8.8,1.1.1.1]
```

Aplicar:

```bash
sudo netplan apply
```

***

## Servidor Zammad (192.168.136.120)

### Instalación y dependencias

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl gnupg apt-transport-https ca-certificates lsb-release nginx postgresql redis-server -y
```

### Elasticsearch

```bash
curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elastic.gpg
echo "deb [signed-by=/usr/share/keyrings/elastic.gpg] https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-7.x.list
sudo apt update && sudo apt install elasticsearch -y
```

Editar:

```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
# Añadir/asegurar:
network.host: localhost
```

Activar:

```bash
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch
```

### Instalación Zammad

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
sudo systemctl restart zammad
```

Acceso local: `http://192.168.136.120`

### Exposición con ngrok

> **Laboratorio**: permite URL pública rápida. En producción, usar dominio + TLS real.

Instalar **en el servidor de Zammad**:

```bash
curl -s https://ngrok-agent.s3.amazonaws.com/ngrok.asc | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null
echo "deb https://ngrok-agent.s3.amazonaws.com buster main" | sudo tee /etc/apt/sources.list.d/ngrok.list
sudo apt update && sudo apt install ngrok -y
ngrok config add-authtoken TU_TOKEN
ngrok http 80
```

> Guardar la URL pública mostrada por ngrok (p.ej. `https://xxxxx.ngrok-free.app`).

### Ajustes base y “token inválido” tras reinicio

Si pruebas con ngrok **cambiante**, Zammad puede esperar el **FQDN antiguo** y romper OAuth/webhooks.  
Para **pruebas locales** (no producción) puedes fijar temporalmente:

```bash
sudo zammad run rails c
Setting.set('fqdn', 'localhost')
Setting.set('http_type', 'http')
exit
sudo systemctl restart zammad
```

> **Producción**: restaurar a dominio real + `https`.  
> Cada vez que cambie el subdominio de ngrok, **actualiza**:
>
> 1.  Base URL en Zammad
> 2.  Redirect URI en Azure (si usas OAuth)
> 3.  Webhook en Chatwoot
> 4.  Webhook en Zammad

***

## Servidor Chatwoot (192.168.136.121)

### Docker y Compose

```bash
sudo apt update
sudo apt install ca-certificates curl gnupg -y
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu noble stable" | sudo tee /etc/apt/sources.list.d/docker.list >/dev/null
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
# Cerrar sesión y entrar de nuevo
```

### Despliegue Chatwoot

```bash
sudo mkdir -p /srv/chatwoot
cd /srv/chatwoot
nano docker-compose.yml
```

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
    depends_on: [postgres, redis]
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

Arrancar:

```bash
docker compose up -d
```

### Acceso y cambio de contraseña

*   UI: `http://192.168.136.121:3000`

Cambiar password de un usuario:

```bash
cd /srv/chatwoot
docker compose exec chatwoot bundle exec rails c
User.pluck(:email)
user = User.find_by(email: "andrea.chamorro@creatorsco.com")
user.password = "Pruebas1?"
user.password_confirmation = "Pruebas1?"
user.save!
```

**Problema de PID/lock**:

```bash
docker compose down
docker compose up -d --force-recreate
```

***

## Integración Microsoft 365 (IMAP/SMTP OAuth)

**Resumen operativo**:

1.  En Zammad, configura **Base URL** con la **URL pública** (ngrok o dominio).
2.  En **Azure Entra ID** → App Registration:
    *   Redirect URI (Web):  
        `https://TU_DOMINIO/api/v1/external_credentials/microsoft365/callback`
    *   Permisos delegados: `IMAP.AccessAsUser.All`, `SMTP.Send`, `offline_access`, `User.Read`
    *   Concede consentimiento de administrador.
3.  En Zammad → **Channels → Email (Microsoft 365)**: **Tenant ID**, **Client ID**, **Client Secret** → **Añadir cuenta**.
4.  Verifica **Entrante: activo** y **Saliente: activo**.

> Recomendaciones: siempre **HTTPS**, secretos rotados, monitorizar expiración del **client secret** y evitar reglas de borrado automático en el buzón.

***

## Middleware Node.js (`chat-integration`)

### Estructura y `.env`

```bash
mkdir -p ~/chat-integration
cd ~/chat-integration
npm init -y
npm install express axios dotenv fs
```

`package.json`:

```json
{
  "name": "chat-integration",
  "version": "1.0.0",
  "type": "module",
  "dependencies": {
    "axios": "^1.7.0",
    "dotenv": "^16.4.0",
    "express": "^4.19.0",
    "fs": "^0.0.1-security"
  }
}
```

`.env`:

    PORT=4000
    ZAMMAD_URL=http://localhost:8080
    ZAMMAD_TOKEN=TU_TOKEN_DE_ZAMMAD
    CHATWOOT_URL=http://192.168.136.121:3000
    CHATWOOT_TOKEN=TU_TOKEN_API_DE_CHATWOOT
    ACCOUNT_ID=1

> **Tokens correctos**
>
> *   Zammad: **Avatar → Profile → Token Access** (activar API en *Admin → Security → API* si no lo estaba).
> *   Chatwoot: **Profile Settings → Access Tokens** (no usar inbox token/webhook token).

### `index.js` (versión final)

> **Copia y pega todo el archivo**:

```js
// index.js
import express from "express";
import axios from "axios";
import dotenv from "dotenv";
import fs from "fs";

dotenv.config();

const app = express();
app.use(express.json());

/* ========= Limpieza de contenido ========= */
function cleanBody(text = "") {
  let t = String(text);
  t = t.replace(/&lt;/g, "<").replace(/&gt;/g, ">").replace(/&amp;/g, "&");
  t = t.replace(/<[^>]*>/g, "");                     // quita HTML
  t = t.replace(/^\s*\[(zammad|chatwoot)\]\s*/i, ""); // quita prefijos
  t = t.split("\n-- ")[0];                            // corta firma clásica
  t = t.replace(/\s+\n/g, "\n").trim();
  return t;
}

/* ========= Persistencia conversationMap ========= */
const MAP_FILE = "./conversationMap.json";
let conversationMap = {};
try {
  if (fs.existsSync(MAP_FILE)) {
    conversationMap = JSON.parse(fs.readFileSync(MAP_FILE, "utf8"));
  }
} catch {
  conversationMap = {};
}
function saveMap() {
  try {
    fs.writeFileSync(MAP_FILE, JSON.stringify(conversationMap, null, 2));
  } catch {}
}

/* ========= Chatwoot → Zammad (cliente) ========= */
app.post("/chatwoot", async (req, res) => {
  try {
    const event = req.body?.event;
    if (event !== "message_created") return res.sendStatus(200);

    const messageType = req.body?.message_type;
    if (messageType !== "incoming") return res.sendStatus(200); // evita loops

    const conversationId = req.body?.conversation?.id;
    const raw = req.body?.content || "";
    const content = cleanBody(raw);
    if (!conversationId || !content) return res.sendStatus(200);

    const contact = req.body?.sender || {};
    const email = contact?.email;
    const name = contact?.name || "Cliente";
    if (!email) return res.sendStatus(200);

    let ticketId = conversationMap[conversationId];

    // Crear ticket si no existe
    if (!ticketId) {
      // crear/buscar usuario
      let customerId;
      try {
        const newUser = await axios.post(
          `${process.env.ZAMMAD_URL}/api/v1/users`,
          { firstname: name, lastname: "-", email, role_ids: [3] },
          { headers: { Authorization: `Token token=${process.env.ZAMMAD_TOKEN}`, "Content-Type": "application/json" } }
        );
        customerId = newUser.data.id;
      } catch {
        const search = await axios.get(
          `${process.env.ZAMMAD_URL}/api/v1/users/search?query=${encodeURIComponent(email)}`,
          { headers: { Authorization: `Token token=${process.env.ZAMMAD_TOKEN}` } }
        );
        customerId = search.data?.[0]?.id;
      }

      const ticket = await axios.post(
        `${process.env.ZAMMAD_URL}/api/v1/tickets`,
        {
          title: `Chat #${conversationId} - ${name}`,
          group: "Users",
          customer_id: customerId,
          article: {
            subject: "Nuevo mensaje desde Chatwoot",
            body: content,
            type: "web",      // no email
            internal: false,  // público
          },
          // Si deseas persistir el vínculo también en Zammad:
          // custom_fields: { chatwoot_id: conversationId }
        },
        { headers: { Authorization: `Token token=${process.env.ZAMMAD_TOKEN}`, "Content-Type": "application/json" } }
      );

      ticketId = ticket.data.id;
      conversationMap[conversationId] = ticketId;
      saveMap();

      return res.sendStatus(200);
    }

    // Añadir artículo al ticket existente
    await axios.post(
      `${process.env.ZAMMAD_URL}/api/v1/ticket_articles`,
      { ticket_id: ticketId, subject: "Mensaje desde Chatwoot", body: content, type: "web", internal: false },
      { headers: { Authorization: `Token token=${process.env.ZAMMAD_TOKEN}`, "Content-Type": "application/json" } }
    );

    return res.sendStatus(200);
  } catch (error) {
    console.error("❌ Error Chatwoot → Zammad:", error.response?.data || error.message);
    return res.sendStatus(500);
  }
});

/* ========= Zammad → Chatwoot (agente) ========= */
app.post("/zammad", async (req, res) => {
  try {
    const article = req.body?.article;
    const ticket = req.body?.ticket;
    if (!article || !ticket) return res.sendStatus(200);

    // Filtrar solo notas públicas de agente
    const isNote = (article.type || "").toLowerCase() === "note";
    const isPublic = article.internal === false;
    const fromAgent =
      (article.sender || "").toLowerCase() === "agent" ||
      (article.sender_name || "").toLowerCase() === "agent";
    if (!(isNote && isPublic && fromAgent)) return res.sendStatus(200);

    const ticketId = ticket.id;

    // Buscar conversación asociada
    let conversationId = Object.keys(conversationMap).find((k) => conversationMap[k] === ticketId);
    if (!conversationId) {
      const cf = ticket.custom_fields || ticket.preferences || {};
      if (cf.chatwoot_id) {
        conversationId = cf.chatwoot_id;
        conversationMap[conversationId] = ticketId;
        saveMap();
      }
    }
    if (!conversationId) return res.sendStatus(200);

    const content = cleanBody(article.body || "");
    if (!content) return res.sendStatus(200);

    await axios.post(
      `${process.env.CHATWOOT_URL}/api/v1/accounts/${process.env.ACCOUNT_ID}/conversations/${conversationId}/messages`,
      { content, message_type: "outgoing", private: false },
      { headers: { api_access_token: process.env.CHATWOOT_TOKEN, "Content-Type": "application/json" } }
    );

    return res.sendStatus(200);
  } catch (error) {
    console.error("❌ Error Zammad → Chatwoot:", error.response?.data || error.message);
    return res.sendStatus(500);
  }
});

app.listen(process.env.PORT || 4000, () => {
  console.log("✅ Middleware activo en puerto", process.env.PORT || 4000);
});
```

### Ejecución, logs y persistencia

```bash
cd ~/chat-integration
node index.js
```

*   Persistencia en `conversationMap.json` (mismo directorio).
*   Logs: se muestran en stdout (puedes redirigirlos o usar systemd, ver más abajo).

### Servicio systemd (producción)

```bash
sudo tee /etc/systemd/system/chat-integration.service >/dev/null <<'UNIT'
[Unit]
Description=Chatwoot ↔ Zammad Middleware
After=network-online.target
Wants=network-online.target

[Service]
WorkingDirectory=/home/zm/chat-integration
Environment=NODE_ENV=production
ExecStart=/usr/bin/node /home/zm/chat-integration/index.js
Restart=always
RestartSec=3
User=zm
Group=zm
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
UNIT

sudo systemctl daemon-reload
sudo systemctl enable --now chat-integration
sudo systemctl status chat-integration
```

***

## Nginx / Reverse proxy para Webhooks

> Si expones el middleware por el mismo dominio de Zammad/ngrok, añade rutas:

```nginx
location /webhook/chatwoot {
    proxy_pass http://127.0.0.1:4000/chatwoot;
}
location /webhook/zammad {
    proxy_pass http://127.0.0.1:4000/zammad;
}
```

Reiniciar:

```bash
sudo systemctl restart nginx
```

> **URLs resultantes para webhooks**:  
> `https://TU_DOMINIO/webhook/chatwoot`  
> `https://TU_DOMINIO/webhook/zammad`

***

## Webhooks y Trigger en Zammad

### Webhook (saliente hacia middleware)

**Admin → Integrations → Webhooks → New**

*   URL: `https://TU_DOMINIO/webhook/zammad`
*   Método: `POST`
*   Content-Type: `application/json`

### Trigger (condiciones correctas)

**Admin → Triggers** → Nuevo

**Condiciones (todas deben cumplirse):**

*   `Artículo → Tipo → es → nota`
*   `Artículo → Visibilidad → es → público`
*   `Artículo → Remitente → es → agente`  ← *mejor que “Último contacto(agente)”*

**Acción:**

*   `Notify via → Webhook` → seleccionar el webhook creado.

> **Truco de diagnóstico**: añade temporalmente una acción “Añadir etiqueta: `enviado-a-chatwoot`” para verificar que el trigger se ejecuta.

***

## Webhook en Chatwoot

**Settings → Integrations → Webhooks**

*   URL: `https://TU_DOMINIO/webhook/chatwoot`
*   Evento: `message_created`

> El middleware filtra `message_type !== 'incoming'` para evitar loops.  
> El **API Token** de Chatwoot es el **de perfil** (no inbox token / no webhook token).

***

## Macro “Responder al Chat” en Zammad

**Admin → Objetos → Macros → Nueva**

*   **Nombre**: `Responder al Chat`
*   **Acciones**:
    *   `Artículo`:
        *   **Tipo**: `nota`
        *   **Visibilidad**: `público`
        *   **Asunto**: *(vacío)*
        *   **Cuerpo**: *(vacío)*
*   **Una vez completado**: `Mantenerse en la pestaña`
*   **Grupos**: seleccionar el grupo donde llegan los tickets (p.ej. `Users`)
*   **Activo**: `Sí`

> Esta macro **prepara** el formulario de nota (no la crea automáticamente), evitando el error  
> `The required 'perform' value for article.note, subject is missing!`

***

## Ajuste: notas públicas por defecto

**Admin → Ajustes de Ticket** (o “Ajustes → Ticket” según versión)

*   `Nota – visibilidad por defecto`: **público**
*   `Artículo – cuadro de confirmación de visibilidad`: a criterio (puede ser `no` para agilizar).

***

## Pruebas de extremo a extremo

1.  **Chatwoot → Zammad**
    *   Mensaje en el chat: “Hola prueba”
    *   Zammad crea/actualiza ticket con **artículo type: `web`, public**
    *   Aparece en la UI de Zammad.

2.  **Zammad → Chatwoot**
    *   En el ticket, **Aplicar macro → Responder al Chat**
    *   Escribir: “Hola, ¿en qué puedo ayudar?” → Guardar
    *   Mensaje llega a Chatwoot **sin firmas ni prefijos**.

3.  Middleware (logs):
    *   Debe mostrar recepciones en `/chatwoot` y `/zammad`.
    *   Sin errores `422` / `Invalid Access Token`.

***

## Resolución de incidencias

*   **No aparece la respuesta en Chatwoot**
    *   Verifica que el **artículo** en Zammad sea **nota pública** y que el **trigger** tenga las 3 condiciones de arriba.
    *   Añade acción temporal de **etiqueta** para ver si el trigger dispara.
    *   Revisa el **webhook** (URL correcta) y **logs** del middleware.

*   **Sigue saliendo `[zammad]`**
    *   Asegúrate de usar este `index.js` (limpia prefijos).
    *   Verifica que no haya triggers/macros añadiendo texto al **body**.

*   **Chatwoot no crea ticket**
    *   Comprueba que llega a `/chatwoot` con `event=message_created` y `message_type=incoming`.
    *   Token de Zammad correcto y API activada.

*   **Token inválido / cambios de ngrok**
    *   Actualiza Base URL en Zammad / Redirect URI en Azure / Webhooks.
    *   Para pruebas locales, `Setting.set('fqdn', 'localhost')` y `http_type` a `http` (no producción).

*   **Persistencia del vínculo al reiniciar**
    *   `conversationMap.json` se guarda en `~/chat-integration`.
    *   Opcional: crear campo **custom** `chatwoot_id` en ticket para persistencia doble.

***

## Checklist para paso a Producción

*   [ ] Dominios reales (ej. `zammad.midominio.com`, `integracion.midominio.com`)
*   [ ] **TLS** en Nginx (Let’s Encrypt / ACME)
*   [ ] Firewall (abrir solo 80/443; 3000 y 4000 no expuestos públicamente)
*   [ ] **systemd** para el middleware (auto-restart)
*   [ ] Backups:
    *   Zammad DB/attachments
    *   Chatwoot DB
    *   `conversationMap.json` (y/o campo `chatwoot_id`)
*   [ ] Monitoreo de logs y métricas básicas
*   [ ] Documentar tokens, secretos y rotación
*   [ ] Pruebas E2E con usuarios reales antes de “go-live”

***

## Apéndices (Docker del middleware, seguridad, backups)

**Docker (opcional del middleware):**

`Dockerfile`

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
ENV NODE_ENV=production
CMD ["node", "index.js"]
```

`docker-compose.yml`

```yaml
services:
  chat-integration:
    build: .
    environment:
      PORT: "4000"
      ZAMMAD_URL: "http://zammad-internal:8080"
      ZAMMAD_TOKEN: "XXXX"
      CHATWOOT_URL: "http://chatwoot-internal:3000"
      CHATWOOT_TOKEN: "YYYY"
      ACCOUNT_ID: "1"
    ports:
      - "4000:4000"
    volumes:
      - ./conversationMap.json:/app/conversationMap.json
    restart: unless-stopped
```

**Seguridad**

*   Tokens en `.env` local con permisos restrictivos.
*   No exponer puertos 3000/4000 a Internet; usar proxy con TLS.
*   Revisar CORS si se añaden endpoints nuevos.

**Backups**

*   Zammad: base de datos + `/opt/zammad` (adjuntos).
*   Chatwoot: base de datos `postgres_data`.
*   Middleware: `conversationMap.json` y, si se crea, la BD asociada.

***

### Estado final del laboratorio

| Servicio       | IP              | Acceso                                                               |
| -------------- | --------------- | -------------------------------------------------------------------- |
| **Zammad**     | 192.168.136.120 | `https://<subdominio>.ngrok-free.dev/` (LAB) / Dominio propio (PROD) |
| **Chatwoot**   | 192.168.136.121 | `http://192.168.136.121:3000/`                                       |
| **Middleware** | (en Zammad)     | `http://127.0.0.1:4000` (tras Nginx: `/webhook/...`)                 |

***
