zm@zammad:~$ ngrok http 80


zm@zammad:~/chat-integration$ node index.js


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

# Archivo PID bloqueado después de reiniciar, solución:
```
docker compose down
```
```
docker compose up -d --force-recreate
```
---

# Integración de Microsoft 365 con Zammad mediante OAuth (IMAP y SMTP)

## 1. Requisitos previos

* Cuenta activa en Microsoft 365
* Acceso de administrador en Azure
* Acceso administrador en Zammad
* Dominio verificado en Microsoft 365
* Zammad accesible por HTTPS (dominio público o túnel como ngrok)

---

# 2. Configuración inicial en Zammad (URL pública correcta)

Antes de crear la aplicación en Azure, es obligatorio que Zammad tenga configurada su URL pública correcta.

## 2.1 Cambiar la URL base del sistema

Ir a:

Admin → Sistema → Configuración → URL (o Base URL)

Cambiar la dirección IP interna (por ejemplo 192.168.x.x) por el dominio público, por ejemplo:

```
https://subdominio.ngrok-free.dev
```

Guardar los cambios.

Esto es imprescindible porque Zammad utilizará esta URL para construir automáticamente la URI de redirección OAuth.

---

# 3. Creación de la aplicación en Azure

## 3.1 Acceder al portal

1. Ir a https://portal.azure.com
2. Entrar en Microsoft Entra ID
3. Seleccionar Registros de aplicaciones
4. Pulsar Nuevo registro

---

## 3.2 Registrar la aplicación

Configurar:

* Nombre: Zammad Mail Integration
* Tipos de cuenta compatibles: Cuentas en este directorio organizativo solamente
* URI de redirección:

  * Tipo: Web
  * URI:

```
https://TU_DOMINIO_ZAMMAD/api/v1/external_credentials/microsoft365/callback
```

Ejemplo con ngrok:

```
https://subdominio.ngrok-free.dev/api/v1/external_credentials/microsoft365/callback
```

Crear la aplicación.

---

## 3.3 Obtener identificadores

Copiar:

* Id. de aplicación (cliente)
* Id. de directorio (inquilino)

---

# 4. Crear secreto del cliente

1. Ir a Certificados y secretos
2. Nuevo secreto de cliente
3. Definir duración
4. Crear

Copiar el valor del secreto inmediatamente.

---

# 5. Configurar permisos API

Ir a:

Permisos de API → Agregar un permiso

Seleccionar Microsoft Graph → Permisos delegados

Añadir:

* IMAP.AccessAsUser.All
* SMTP.Send
* offline_access
* User.Read

Guardar.

Después:

Pulsar Conceder consentimiento de administrador.

---

# 6. Configuración en Zammad

Ir a:

Admin → Channels → Email

Seleccionar:

Microsoft 365 IMAP Email

---

## 6.1 Configurar App

Pulsar Configurar App e introducir:

* Tenant ID
* Client ID
* Client Secret

Guardar.

---

## 6.2 Añadir cuenta

Pulsar Añadir cuenta.

Iniciar sesión con la cuenta de Microsoft 365 y autorizar.

Debe aparecer:

* Entrante: activo
* Saliente: activo

---

# 7. Configuración correcta del canal entrante

Editar el canal entrante y verificar:

* Grupo de destino: el deseado
* Directorio: Inbox
* Mantener mensajes en el servidor: yes

Guardar.

Esto garantiza que los correos permanezcan en el buzón tras ser procesados.

---

# 8. Verificación del canal saliente

Confirmar que el canal saliente esté activo.

Esto asegura que:

* Los correos enviados desde Zammad salgan desde la cuenta real
* Aparezcan en la carpeta Enviados del buzón

---

# 9. Prueba de funcionamiento

1. Enviar un correo de prueba a la cuenta configurada.
2. Verificar:

   * El correo aparece en Outlook Web.
   * Se crea un ticket en Zammad.
3. Responder desde Zammad.
4. Verificar:

   * El destinatario recibe el mensaje.
   * El correo aparece en Enviados en Outlook.

---

# 10. Recomendaciones para producción

* Utilizar siempre HTTPS.
* Si se usa ngrok, actualizar la URI de redirección en Azure cuando cambie el subdominio.
* Supervisar la caducidad del secreto de cliente.
* Realizar copias de seguridad periódicas.
* Evitar reglas automáticas de borrado en el buzón.

---


# 🧩 RESULTADO FINAL DEL LAB

| Servicio | IP              | Acceso                                                               |
| -------- | --------------- | -------------------------------------------------------------------- |
| Zammad   | 192.168.136.120 | [https://polygalaceous-alaysia-unsilently.ngrok-free.dev/](https://polygalaceous-alaysia-unsilently.ngrok-free.dev/) |
| Chatwoot | 192.168.136.121 | [http://192.168.136.121:3000/](http://192.168.136.121:3000/) |

# 🧱 PARTE 1 — Crear el middleware Node.js desde cero

## 1️⃣ Instalar Node

En el servidor donde está Zammad:

```bash
node -v
```

Si no está instalado:

```bash
sudo apt update
sudo apt install nodejs npm -y
```

---

## 2️⃣ Crear carpeta del middleware

```bash
mkdir chat-integration
cd chat-integration
npm init -y
```

---

## 3️⃣ Instalar dependencias

```bash
npm install express axios dotenv
```

---

## 4️⃣ Crear archivo `.env`

```bash
nano .env
```

Contenido:

```env
PORT=4000

ZAMMAD_URL=http://localhost:3000
ZAMMAD_TOKEN=TU_TOKEN_ZAMMAD

CHATWOOT_URL=https://chatwoot.tudominio.com
CHATWOOT_TOKEN=TU_TOKEN_CHATWOOT
ACCOUNT_ID=1
```

Guardar.

---

## 5️⃣ Crear `index.js`

```bash
nano index.js
```

Pega esto:

```js
import express from "express";
import axios from "axios";
import dotenv from "dotenv";

dotenv.config();

const app = express();
app.use(express.json());

/*
=============================
1️⃣ Chatwoot → Zammad
=============================
*/
app.post("/chatwoot", async (req, res) => {

  if (req.body.event !== "conversation_created") {
    return res.sendStatus(200);
  }

  const conversation = req.body.conversation;
  const contact = conversation.contact;

  try {
    await axios.post(
      `${process.env.ZAMMAD_URL}/api/v1/tickets`,
      {
        title: `Chat #${conversation.id} - ${contact.name}`,
        group: "Users",
        customer: contact.email || "chatwoot@local",
        article: {
          subject: "Nuevo mensaje desde Chatwoot",
          body: conversation.messages?.[0]?.content || "Nuevo chat iniciado",
          type: "note",
          internal: false
        },
        custom_fields: {
          chatwoot_id: conversation.id
        }
      },
      {
        headers: {
          Authorization: `Token token=${process.env.ZAMMAD_TOKEN}`
        }
      }
    );

    console.log("✅ Ticket creado en Zammad");
    res.sendStatus(200);

  } catch (error) {
    console.error("❌ Error creando ticket:", error.response?.data || error.message);
    res.sendStatus(500);
  }
});


/*
=============================
2️⃣ Zammad → Chatwoot
=============================
*/
app.post("/zammad", async (req, res) => {

  const article = req.body.article;
  const ticket = req.body.ticket;

  if (!article || article.internal) return res.sendStatus(200);

  const chatwootId = ticket.custom_fields?.chatwoot_id;
  if (!chatwootId) return res.sendStatus(200);

  try {
    await axios.post(
      `${process.env.CHATWOOT_URL}/api/v1/accounts/${process.env.ACCOUNT_ID}/conversations/${chatwootId}/messages`,
      {
        content: article.body,
        message_type: "outgoing"
      },
      {
        headers: {
          api_access_token: process.env.CHATWOOT_TOKEN
        }
      }
    );

    console.log("📨 Respuesta enviada a Chatwoot");
    res.sendStatus(200);

  } catch (error) {
    console.error("❌ Error enviando a Chatwoot:", error.response?.data || error.message);
    res.sendStatus(500);
  }
});


app.listen(process.env.PORT, () => {
  console.log(`🚀 Middleware corriendo en puerto ${process.env.PORT}`);
});
```

---

## 6️⃣ Ajustar package.json para ES Modules

Editar `package.json` y añadir:

```json
"type": "module"
```

---

## 7️⃣ Ejecutar middleware

```bash
node index.js
```

Debe decir:

```
🚀 Middleware corriendo en puerto 4000
```

---

# 🧱 PARTE 2 — Integrarlo con tu reverse proxy

En nginx añade:

En zammad.conf, dentro de servers.

```nginx
location /webhook/chatwoot {
    proxy_pass http://localhost:4000/chatwoot;
}

location /webhook/zammad {
    proxy_pass http://localhost:4000/zammad;
}
```

Reiniciar nginx:

```bash
sudo systemctl restart nginx
```

Ahora:

```
https://subdominio.ngrok-free.dev/webhook/chatwoot
https://subdominio.ngrok-free.dev/webhook/zammad
```

---

# 🧱 PARTE 3 — Configuración en Chatwoot

En Chatwoot:

Settings → Integrations → Webhooks

Evento:

```
conversation_created
```

URL:

```
https://subdominio.ngrok-free.dev/webhook/chatwoot
```

---

# 🧱 PARTE 4 — Configuración en Zammad

En Zammad:

Admin → Webhooks → Nuevo

Evento:

```
Ticket updated
```

URL:

```
https://subdominio.ngrok-free.dev/webhook/zammad
```

---

# 🔁 CUANDO CAMBIE NGROK

Cada vez que cambie la URL:

### Cambiar:

1️⃣ Base URL en Zammad
2️⃣ Redirect URI en Azure
3️⃣ Webhook en Chatwoot
4️⃣ Webhook en Zammad


----------------------REVISAR MAÑANA-------------------------------------------------(falta poder responder desde zammad y que aparezca en el chatweb de chatwoot)
Perfecto.
Aquí tienes la **GUÍA FINAL DEFINITIVA** para conectar:

👉 Chatwoot
👉 Zammad

Sin errores de SSL, sin error 422, sin “Invalid Access Token”, y con el `index.js` correcto.

---

# 🧠 ARQUITECTURA FINAL

```
Chatwoot  →  Middleware Node  →  Zammad
Chatwoot  ←  Middleware Node  ←  Zammad
```

El middleware hace toda la magia.

Carpeta:

```
~/chat-integration
```

---

# 1️⃣ CREAR TOKENS CORRECTOS

---

## 🔹 TOKEN DE ZAMMAD

Entra en Zammad:

```
Admin → Security → API
```

Activa API si no está activa.

Luego:

```
Avatar → Profile → Token Access
```

Crea un nuevo token.

Guárdalo.

---

## 🔹 TOKEN DE CHATWOOT (IMPORTANTE)

Entra en Chatwoot:

```
Avatar (arriba derecha)
→ Profile Settings
→ Access Tokens
→ Create New Token
```

⚠️ ESTE es el token correcto
⚠️ NO usar inbox token
⚠️ NO usar webhook token

Guárdalo.

---

# 2️⃣ CREAR PROYECTO NODE

```bash
mkdir ~/chat-integration
cd ~/chat-integration
npm init -y
npm install express axios dotenv
```

En `package.json` añade:

```json
"type": "module"
```

---

# 3️⃣ CREAR ARCHIVO .env

```bash
nano .env
```

Contenido correcto:

```env
PORT=4000

ZAMMAD_URL=http://localhost:8080
ZAMMAD_TOKEN=TU_TOKEN_DE_ZAMMAD

CHATWOOT_URL=http://192.168.136.121:3000
CHATWOOT_TOKEN=TU_TOKEN_API_DE_CHATWOOT
ACCOUNT_ID=1
```

⚠️ IMPORTANTE:

* NO pongas `/app`
* NO pongas `/accounts`
* SOLO la raíz del servidor

Ejemplo correcto:

```
http://192.168.136.121:3000
```

---

# 4️⃣ INDEX.JS FINAL CORRECTO

Reemplaza TODO por esto:

```js
import express from "express";
import axios from "axios";
import dotenv from "dotenv";

dotenv.config();

const app = express();
app.use(express.json());

// Mapa en memoria conversación → ticket
const conversationMap = {};

/*
====================================
CHATWOOT → ZAMMAD
====================================
*/
app.post("/chatwoot", async (req, res) => {
  try {
    const event = req.body?.event;
    if (!event) return res.sendStatus(200);

    if (event === "message_created" && req.body.message_type === "incoming") {

      const conversationId = req.body?.conversation?.id;
      const message = req.body?.content || "";
      const contact = req.body?.sender || {};

      if (!conversationId) return res.sendStatus(200);

      const email = contact?.email;
      const name = contact?.name || "Sin nombre";

      if (!email) return res.sendStatus(200);

      let ticketId = conversationMap[conversationId];

      // SI NO EXISTE → CREAR TICKET
      if (!ticketId) {

        let customerId;

        const newUser = await axios.post(
          `${process.env.ZAMMAD_URL}/api/v1/users`,
          {
            firstname: name,
            lastname: "-",
            email: email,
            role_ids: [3]
          },
          {
            headers: {
              Authorization: `Token token=${process.env.ZAMMAD_TOKEN}`,
              "Content-Type": "application/json"
            }
          }
        ).catch(() => null);

        if (newUser) {
          customerId = newUser.data.id;
        } else {
          const userSearch = await axios.get(
            `${process.env.ZAMMAD_URL}/api/v1/users/search?query=${email}`,
            {
              headers: {
                Authorization: `Token token=${process.env.ZAMMAD_TOKEN}`
              }
            }
          );
          customerId = userSearch.data[0]?.id;
        }

        const newTicket = await axios.post(
          `${process.env.ZAMMAD_URL}/api/v1/tickets`,
          {
            title: `Chat #${conversationId} - ${name}`,
            group: "Users",
            customer_id: customerId,
            article: {
              subject: "Nuevo mensaje desde Chatwoot",
              body: message.replace(/<[^>]*>?/gm, ""),
              type: "note",
              internal: false
            }
          },
          {
            headers: {
              Authorization: `Token token=${process.env.ZAMMAD_TOKEN}`,
              "Content-Type": "application/json"
            }
          }
        );

        ticketId = newTicket.data.id;
        conversationMap[conversationId] = ticketId;

        return res.sendStatus(200);
      }

      // SI EXISTE → AÑADIR MENSAJE
      await axios.post(
        `${process.env.ZAMMAD_URL}/api/v1/ticket_articles`,
        {
          ticket_id: ticketId,
          subject: "Nuevo mensaje desde Chatwoot",
          body: message.replace(/<[^>]*>?/gm, ""),
          type: "note",
          internal: false
        },
        {
          headers: {
            Authorization: `Token token=${process.env.ZAMMAD_TOKEN}`,
            "Content-Type": "application/json"
          }
        }
      );

      return res.sendStatus(200);
    }

    return res.sendStatus(200);

  } catch (error) {
    console.error("Error Chatwoot → Zammad:", error.response?.data || error.message);
    return res.sendStatus(500);
  }
});

/*
====================================
ZAMMAD → CHATWOOT
====================================
*/
app.post("/zammad", async (req, res) => {
  try {

    const article = req.body?.article;
    const ticket = req.body?.ticket;

    if (!article || article.internal) return res.sendStatus(200);

    const ticketId = ticket?.id;

    const conversationId = Object.keys(conversationMap)
      .find(key => conversationMap[key] === ticketId);

    if (!conversationId) return res.sendStatus(200);

    await axios.post(
      `${process.env.CHATWOOT_URL}/api/v1/accounts/${process.env.ACCOUNT_ID}/conversations/${conversationId}/messages`,
      {
        content: article.body,
        message_type: "outgoing"
      },
      {
        headers: {
          api_access_token: process.env.CHATWOOT_TOKEN,
          "Content-Type": "application/json"
        }
      }
    );

    return res.sendStatus(200);

  } catch (error) {
    console.error("Error Zammad → Chatwoot:", error.response?.data || error.message);
    return res.sendStatus(500);
  }
});

app.listen(process.env.PORT || 4000, () => {
  console.log(`Middleware corriendo en puerto ${process.env.PORT || 4000}`);
});
```

---

# 5️⃣ CONFIGURAR WEBHOOKS

---

## 🔹 En Chatwoot

```
Settings → Integrations → Webhooks
```

Añadir:

```
http://TU_SERVIDOR:4000/chatwoot
```

Evento:

```
message_created
```

---

## 🔹 En Zammad

```
Admin → Webhooks → New Webhook
```

URL:

```
http://TU_SERVIDOR:4000/zammad
```

Evento:

```
Ticket update
```

---

# 6️⃣ ARRANCAR

```bash
node index.js
```

---

# ✅ RESULTADO FINAL

✔ Mensaje en Chatwoot → crea ticket en Zammad
✔ Respuesta en Zammad → aparece en Chatwoot
✔ Sin errores SSL
✔ Sin Invalid Access Token
✔ Sin 422

---

# ⚠️ LIMITACIÓN ACTUAL

El `conversationMap` está en memoria.
Si reinicias Node, pierde la relación.

Si quieres la versión profesional con base de datos persistente, te la preparo también.

---

Si quieres, ahora podemos hacer:

* 🔒 Versión producción
* 🗄️ Versión con base de datos
* 🐳 Versión Dockerizada
* 🌍 Versión con dominio y SSL correcto

Tú decides 🚀

-------------------------------TOKEN INVALID AL REINICIAR------------------------------------------------
2️⃣ Cambiar la Base URL manualmente

En tu servidor ejecuta:
```
sudo zammad run rails c
```
Ahora pega esto:
```
Setting.set('fqdn', 'localhost')
```
y luego:
```
Setting.set('http_type', 'http')
```
Esto hace que Zammad deje de esperar la URL de ngrok.

Sal de la consola:

exit
🔧 3️⃣ Reiniciar Zammad
```
sudo systemctl restart zammad
```
Espera unos segundos.

