*   Configuración de red
*   Instalación de Zammad
*   Instalación de Chatwoot
*   Integración OAuth con Microsoft 365
*   Middleware actualizado (versión final estable)
*   Correcciones aplicadas durante el proceso
*   Trigger
*   Macro
*   Ajustes de notas
*   Errores resueltos

***

# 🟦 **GUÍA DEFINITIVA CHATWOOT ↔ ZAMMAD (2026)**

*Integración completa, limpia y funcional sin emails, sin firmas, sin prefijos.*

***

# 🧩 **1. CONFIGURACIÓN DE RED (AMBOS SERVIDORES)**

## 🔹 **Zammad**

IP: `192.168.136.120`

## 🔹 **Chatwoot**

IP: `192.168.136.121`

## 🔧 **Netplan (en ambos):**

Archivo:

    sudo nano /etc/netplan/50-cloud-init.yaml

### Zammad:

```yaml
addresses:
  - 192.168.136.120/24
routes:
  - to: default
    via: 192.168.136.1
nameservers:
  addresses: [8.8.8.8,1.1.1.1]
```

### Chatwoot:

```yaml
addresses:
  - 192.168.136.121/24
routes:
  - to: default
    via: 192.168.136.1
nameservers:
  addresses: [8.8.8.8,1.1.1.1]
```

Aplicar:

    sudo netplan apply

***

# 🟦 **2. INSTALACIÓN DE ZAMMAD (NATIVO)**

**Servidor:** `192.168.136.120`

### 2.1 Actualizar sistema

```bash
sudo apt update && sudo apt upgrade -y
sudo reboot
```

### 2.2 Dependencias + Nginx

```bash
sudo apt install curl gnupg apt-transport-https ca-certificates lsb-release nginx postgresql redis-server -y
```

### 2.3 Instalar Elasticsearch

```bash
curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elastic.gpg
echo "deb [signed-by=/usr/share/keyrings/elastic.gpg] https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-7.x.list

sudo apt update
sudo apt install elasticsearch -y
```

Editar:

    sudo nano /etc/elasticsearch/elasticsearch.yml

Asegurar:

    network.host: localhost

Activar:

```bash
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch
```

### 2.4 Instalar Zammad

```bash
curl -fsSL https://dl.packager.io/srv/zammad/zammad/key | sudo gpg --dearmor -o /usr/share/keyrings/zammad.gpg
echo "deb [signed-by=/usr/share/keyrings/zammad.gpg] https://dl.packager.io/srv/deb/zammad/zammad/stable/ubuntu 22.04 main" | sudo tee /etc/apt/sources.list.d/zammad.list

sudo apt update
sudo apt install zammad -y
```

Inicializar base:

```bash
sudo zammad run rake db:create
sudo zammad run rake db:migrate
sudo zammad run rake db:seed
sudo systemctl restart zammad
```

### 2.5 Acceso local

    http://192.168.136.120

***

# 🟦 **3. EXPONER ZAMMAD CON NGROK**

Instalar:

```bash
curl -s https://ngrok-agent.s3.amazonaws.com/ngrok.asc | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null
echo "deb https://ngrok-agent.s3.amazonaws.com buster main" | sudo tee /etc/apt/sources.list.d/ngrok.list
sudo apt update
sudo apt install ngrok -y
```

Configurar token:

```bash
ngrok config add-authtoken TU_TOKEN
```

Exponer:

```bash
ngrok http 80
```

***

# 🟦 **4. INSTALACIÓN DE CHATWOOT (DOCKER)**

**Servidor:** `192.168.136.121`

### 4.1 Instalar Docker

```bash
sudo apt update
sudo apt install ca-certificates curl gnupg -y
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu noble stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER
```

Cerrar sesión y volver a entrar.

### 4.2 Crear proyecto

```bash
sudo mkdir -p /srv/chatwoot
cd /srv/chatwoot
nano docker-compose.yml
```

### 4.3 Contenido:

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

***

# 🟦 **5. ACCESO A CHATWOOT**

    http://192.168.136.121:3000

***

# 🟦 **6. CAMBIAR CONTRASEÑA ADMIN DE CHATWOOT**

```bash
cd /srv/chatwoot
docker compose exec chatwoot bundle exec rails c
User.pluck(:email)
user = User.find_by(email: "andrea.chamorro@creatorsco.com")
user.password = "Pruebas1?"
user.password_confirmation = "Pruebas1?"
user.save!
```

***

# 🟦 **7. PROBLEMAS COMUNES**

Si Chatwoot queda bloqueado:

```bash
docker compose down
docker compose up -d --force-recreate
```

***

# 🟦 **8. INTEGRACIÓN MICROSOFT 365 (IMAP/SMTP OAuth)**

*(Resumen, igual que tu guía original pero ordenado)*

### 8.1 Requisitos

*   Cuenta M365
*   Acceso Azure
*   Zammad con HTTPS (ngrok o dominio)

### 8.2 Configurar URL base en Zammad

**Admin → Sistema → Configuración → URL**

Debe contener el dominio HTTPS usado por ngrok:

    https://subdominio.ngrok-free.dev

### 8.3 Crear aplicación en Azure (Entra ID)

*   Nuevo registro
*   URI de redirección:

<!---->

    https://TU_DOMINIO/api/v1/external_credentials/microsoft365/callback

### 8.4 Permisos delegados:

*   IMAP.AccessAsUser.All
*   SMTP.Send
*   offline\_access
*   User.Read

Conceder consentimiento.

### 8.5 Configurar en Zammad

**Admin → Channels → Email → Microsoft 365**

*   Tenant ID
*   Client ID
*   Client Secret

Autenticar.

***

# 🟦 **9. MIDDLEWARE NODE.JS (VERSIÓN FINAL)**

### 9.1 Crear proyecto

```bash
mkdir ~/chat-integration
cd ~/chat-integration
npm init -y
npm install express axios dotenv fs
```

### 9.2 `.env`

    PORT=4000
    ZAMMAD_URL=http://localhost:8080
    ZAMMAD_TOKEN=TU_TOKEN
    CHATWOOT_URL=http://192.168.136.121:3000
    CHATWOOT_TOKEN=TU_TOKEN
    ACCOUNT_ID=1

### 9.3 `package.json`

Agregar:

```json
"type": "module"
```

### 9.4 `index.js` (versión estable)
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     index.js
// index.js

```
import express from "express";
import axios from "axios";
import dotenv from "dotenv";
import fs from "fs";

dotenv.config();

const app = express();
app.use(express.json());

/* =========================================================
   Utilidad de limpieza: sin HTML, sin [zammad]/[chatwoot], sin firma
   ========================================================= */
function cleanBody(text = "") {
  let t = String(text);

  // Decodifica entidades básicas
  t = t.replace(/&lt;/g, "<").replace(/&gt;/g, ">").replace(/&amp;/g, "&");

  // Elimina etiquetas HTML
  t = t.replace(/<[^>]*>/g, "");

  // Elimina prefijos tipo [zammad] o [chatwoot] al inicio
  t = t.replace(/^\s*\[(zammad|chatwoot)\]\s*/i, "");

  // Corta tras separador de firma clásico
  t = t.split("\n-- ")[0];

  // Limpieza de espacios
  t = t.replace(/\s+\n/g, "\n").trim();

  return t;
}

/* =========================================================
   PERSISTENCIA conversationMap  (con JSON en disco)
   ========================================================= */
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
  } catch {
    // noop
  }
}

/* =========================================================
   CHATWOOT → ZAMMAD  (solo mensajes del cliente)
   ========================================================= */
app.post("/chatwoot", async (req, res) => {
  try {
    const event = req.body?.event;
    if (event !== "message_created") return res.sendStatus(200);

    const messageType = req.body?.message_type; // incoming / outgoing
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

    // 1) Si no hay ticket asociado, crearlo
    if (!ticketId) {
      // Crear / buscar usuario
      let customerId;
      try {
        const newUser = await axios.post(
          `${process.env.ZAMMAD_URL}/api/v1/users`,
          { firstname: name, lastname: "-", email, role_ids: [3] },
          {
            headers: {
              Authorization: `Token token=${process.env.ZAMMAD_TOKEN}`,
              "Content-Type": "application/json",
            },
          }
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
            type: "web",      // artículo de origen web (NO email)
            internal: false,  // público
          },
        },
        {
          headers: {
            Authorization: `Token token=${process.env.ZAMMAD_TOKEN}`,
            "Content-Type": "application/json",
          },
        }
      );

      ticketId = ticket.data.id;
      conversationMap[conversationId] = ticketId;
      saveMap();

      return res.sendStatus(200);
    }

    // 2) Si ya existe, añadir artículo público
    await axios.post(
      `${process.env.ZAMMAD_URL}/api/v1/ticket_articles`,
      {
        ticket_id: ticketId,
        subject: "Mensaje desde Chatwoot",
        body: content,
        type: "web",     // mantenemos "web" para entradas de chat
        internal: false, // público
      },
      {
        headers: {
          Authorization: `Token token=${process.env.ZAMMAD_TOKEN}`,
          "Content-Type": "application/json",
        },
      }
    );

    return res.sendStatus(200);
  } catch (error) {
    console.error("❌ Error Chatwoot → Zammad:", error.response?.data || error.message);
    return res.sendStatus(500);
  }
});

/* =========================================================
   ZAMMAD → CHATWOOT  (solo notas públicas de agente)
   ========================================================= */
app.post("/zammad", async (req, res) => {
  try {
    const article = req.body?.article;
    const ticket = req.body?.ticket;
    if (!article || !ticket) return res.sendStatus(200);

    // Filtros robustos (por si el trigger es laxo)
    const isNote = (article.type || "").toLowerCase() === "note";
    const isPublic = article.internal === false;
    const fromAgent =
      (article.sender || "").toLowerCase() === "agent" ||
      (article.sender_name || "").toLowerCase() === "agent";

    if (!(isNote && isPublic && fromAgent)) return res.sendStatus(200);

    const ticketId = ticket.id;

    // Recuperar conversación asociada
    let conversationId = Object.keys(conversationMap).find(
      (key) => conversationMap[key] === ticketId
    );

    // Fallback: si guardaste chatwoot_id como custom field en el ticket
    if (!conversationId) {
      const cf = ticket.custom_fields || ticket.preferences || {};
      if (cf.chatwoot_id) {
        conversationId = cf.chatwoot_id;
        conversationMap[conversationId] = ticketId;
        saveMap();
      }
    }

    if (!conversationId) {
      console.log("⚠ No se encontró conversación asociada a este ticket.");
      return res.sendStatus(200);
    }

    const content = cleanBody(article.body || "");
    if (!content) return res.sendStatus(200);

    await axios.post(
      `${process.env.CHATWOOT_URL}/api/v1/accounts/${process.env.ACCOUNT_ID}/conversations/${conversationId}/messages`,
      {
        content,                 // ← SIN prefijo [zammad]
        message_type: "outgoing",
        private: false,
      },
      {
        headers: {
          api_access_token: process.env.CHATWOOT_TOKEN,
          "Content-Type": "application/json",
        },
      }
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

# 🟦 **10. NGINX — REVERSE PROXY PARA WEBHOOKS**

En la config de Zammad:

    location /webhook/chatwoot {
        proxy_pass http://127.0.0.1:4000/chatwoot;
    }

    location /webhook/zammad {
        proxy_pass http://127.0.0.1:4000/zammad;
    }

Reiniciar Nginx:

    sudo systemctl restart nginx

***

# 🟦 **11. CONFIGURAR WEBHOOKS**

### 🔹 En Chatwoot

**Settings → Integrations → Webhooks**

    URL: https://TU_DOMINIO/webhook/chatwoot
    Evento: message_created

### 🔹 En Zammad

**Admin → Webhooks**

    URL: https://TU_DOMINIO/webhook/zammad
    Evento: Ticket update

***

# 🟦 **12. TRIGGER PARA ENVIAR NOTAS A CHATWOOT (VERSIÓN CORRECTA)**

**Admin → Triggers**

Condiciones:

    Artículo → Tipo → es → nota
    Artículo → Visibilidad → es → público
    Artículo → Remitente → es → agente

Acciones:

    Notificar vía → Webhook → (tu webhook)

***

# 🟦 **13. AJUSTES IMPORTANTES EN ZAMMAD**

### 13.1 Hacer notas públicas por defecto

**Admin → Ajustes de Ticket**  
Opción:

    Nota – visibilidad por defecto → público

***

# 🟦 **14. MACRO “RESPONDER AL CHAT”**

**Admin → Objetos → Macros → Nueva**

*   **Nombre:** Responder al Chat
*   **Acción:**
        Artículo:
           Tipo → nota
           Visibilidad → público
           Asunto → (vacío)
           Cuerpo → (vacío)
*   **Una vez completado:** mantenerse en la pestaña
*   **Grupo:** Users
*   **Activo:** Sí

⚡ Esta macro abre directamente **nota pública**, lista para enviar al chat.

***

# 🟦 **15. RESULTADO FINAL DEL LAB**

| Servicio     | IP              | Acceso                                 |
| ------------ | --------------- | -------------------------------------- |
| **Zammad**   | 192.168.136.120 | <https://TU-SUBDOMINIO.ngrok-free.dev> |
| **Chatwoot** | 192.168.136.121 | <http://192.168.136.121:3000>          |

***

# 🟦 **16. FLUJO FINAL FUNCIONAL**

✔ Cliente escribe en Chatwoot → llega a Zammad como `type: web`  
✔ Agente pulsa **Responder al Chat** → nota pública → llega a Chatwoot  
✔ Sin correos  
✔ Sin firmas  
✔ Sin `[zammad]`  
✔ Sin loops  
✔ Persistencia conversationMap.json  
✔ Compatible con reinicios

***

Si quieres, puedo generar también:

✅ Versión PDF  
✅ Versión más corta  
✅ Versión con índice  
✅ Versión con imágenes y diagramas  
✅ Versión para tu repositorio GitHub exactamente con el estilo que usas

¿Quieres alguna de esas?
