# Arquitectura Completa de Soporte + CRM
Laboratorio técnico preparado para producción

---

# 1. OBJETIVO

Construir una arquitectura profesional, simple y mantenible:

- Zammad → Centro único de tickets
- Chatwoot → Conector de redes sociales + chat web
- Odoo → CRM y seguimiento comercial
- Web corporativa → Formulario + Chatbot
- Email pruebas → ndrepruebas@gmail.com

---

# 2. ARQUITECTURA GENERAL

Instagram / Facebook
        ↓
     Meta Webhook
        ↓
     Chatwoot
        ↓ (Email forward)
      Zammad
        ↓ (Webhook)
      Odoo CRM

Web Formulario
        ↓
      Email o API
        ↓
      Zammad
        ↓
      Odoo

Web Chat
        ↓
     Chatwoot
        ↓
     Zammad
        ↓
     Odoo

---

# 3. PREPARACIÓN DEL ENTORNO

## 3.1 Requisitos mínimos

- Servidor Linux (Ubuntu recomendado)
- Docker
- Docker Compose
- Dominio (recomendado)
- Cuenta Gmail
- Perfil personal Facebook
- Página de Facebook
- Instagram Business

---

# 4. CONFIGURAR ZAMMAD COMO CENTRO DE SOPORTE

## 4.1 Configuración de Email

Ir a:

Admin → Channels → Email

### IMAP

Host: imap.gmail.com  
Puerto: 993  
SSL: Activado  

### SMTP

Host: smtp.gmail.com  
Puerto: 587  
STARTTLS  

---

## 4.2 Configuración en Google

1. Ir a Google Account
2. Activar verificación en dos pasos
3. Ir a Contraseñas de aplicación
4. Crear contraseña para "Correo"
5. Usar esa contraseña en Zammad

Resultado:
Todo correo enviado a ndrepruebas@gmail.com genera ticket automático.

---

# 5. CONFIGURAR META (INSTAGRAM + FACEBOOK)

## 5.1 Crear Página de Facebook

Ir a:

https://facebook.com/pages/create

Debe ser Página de empresa.

---

## 5.2 Configurar Instagram Business

En Instagram:

Configuración → Cuenta → Cambiar a cuenta profesional → Empresa

Después:

Configuración → Centro de cuentas → Vincular con la Página de Facebook

Debe quedar asociada correctamente.

---

## 5.3 Crear Meta Business Manager

Ir a:

https://business.facebook.com

Pasos:

1. Crear empresa
2. Añadir Página de Facebook
3. Añadir cuenta Instagram
4. Verificar conexión entre ambas

---

# 6. CREAR APP EN META DEVELOPERS

Ir a:

https://developers.facebook.com

Crear nueva aplicación:

Tipo: Business

---

## 6.1 Añadir productos

Agregar:

- Messenger
- Instagram Graph API

---

## 6.2 Configuración básica

App Settings → Basic

Añadir:

- Dominio de Chatwoot (ej: chat.midominio.com)
- Email contacto
- URL política privacidad

Guardar cambios.

---

# 7. GENERAR TOKEN

Ir a:

Tools → Graph API Explorer

Seleccionar permisos:

- pages_manage_metadata
- pages_messaging
- instagram_basic
- instagram_manage_messages
- pages_read_engagement

Generar token.

Convertir a long-lived token.

---

# 8. CONFIGURAR WEBHOOK EN META

Messenger → Settings → Webhooks

Añadir:

Callback URL → URL proporcionada por Chatwoot  
Verify Token → El definido en Chatwoot  

Suscribirse a:

Facebook:
- messages
- messaging_postbacks
- messaging_optins

Instagram:
- messages
- messaging_seen

---

# 9. CONFIGURAR CHATWOOT

Settings → Inboxes → Add Inbox

## Facebook

Se solicitará:

- App ID
- App Secret
- Page ID
- Page Access Token

---

## Instagram

Se solicitará:

- Cuenta Business vinculada
- Token generado
- App conectada

Verificar funcionamiento enviando mensajes de prueba.

---

# 10. ENVIAR CHATWOOT A ZAMMAD

Método recomendado:

Settings → Automation → Email Forward

Reenviar todos los mensajes entrantes a:

ndrepruebas@gmail.com

Zammad los convierte en tickets automáticamente.

Ventajas:
- Estable
- Bajo mantenimiento
- Sin dependencias API complejas

---

# 11. SINCRONIZAR ZAMMAD CON ODOO

Método: Webhook personalizado

---

# 11.1 Crear módulo Odoo

Ruta:

/mnt/extra-addons/zammad_webhook/

## __manifest__.py

```python
{
   'name': 'Zammad Webhook',
   'version': '1.0',
   'category': 'Tools',
   'summary': 'Receive Zammad tickets into CRM',
   'depends': ['crm'],
   'data': [],
   'installable': True,
   'application': False,
}
````

---

## **init**.py

```python
from . import controllers
```

---

## controllers/**init**.py

```python
from . import main
```

---

## controllers/main.py

```python
from odoo import http
from odoo.http import request
import json

class ZammadWebhook(http.Controller):

   @http.route('/api/zammad_ticket', type='http', auth='public', methods=['POST'], csrf=False)
   def create_ticket(self, **kwargs):

       data = json.loads(request.httprequest.data)

       ticket = data.get('ticket', {})
       article = data.get('article', {})

       title = ticket.get('title')
       customer_email = ticket.get('customer', {}).get('email')
       body = article.get('body')

       lead = request.env['crm.lead'].sudo().create({
           'name': title or 'Sin título',
           'contact_name': customer_email,
           'email_from': customer_email,
           'description': body,
           'type': 'lead',
       })

       return http.Response(
           json.dumps({'status': 'ok', 'lead_id': lead.id}),
           content_type='application/json'
       )
```

---

## Reiniciar Odoo

```
docker compose restart odoo
```

---

## Instalar módulo

Apps → Activar modo desarrollador → Actualizar lista → Buscar "Zammad Webhook" → Instalar

---

# 11.2 Configurar Webhook en Zammad

Admin → Webhook

Crear nuevo:

Nombre: Mandar a Odoo
Método: POST
URL: http://IP_ODDO:8069/api/zammad_ticket
Content-Type: application/json
Activo: Sí

Guardar.

---

# 11.3 Crear Trigger

Admin → Disparadores

Nuevo:

Nombre: Enviar ticket a Odoo

Condición:
Acción → es → creado

Ejecutar cambios:
Webhook → Mandar a Odoo

Guardar.

Resultado:
Cada ticket crea Lead en Odoo automáticamente.

---

# 12. FORMULARIO WEB QUE CREA TICKET

## 12.1 Estructura HTML

```html
<form method="POST" action="/contacto">
  <input type="text" name="nombre" placeholder="Nombre" required>
  <input type="email" name="email" placeholder="Email" required>
  <input type="text" name="asunto" placeholder="Asunto" required>
  <textarea name="mensaje" placeholder="Mensaje" required></textarea>
  <button type="submit">Enviar</button>
</form>
```

---

## 12.2 Backend Node.js (Opción Email)

Instalar dependencias:

```
npm init -y
npm install express nodemailer body-parser
```

Crear archivo server.js:

```javascript
const express = require("express");
const nodemailer = require("nodemailer");
const bodyParser = require("body-parser");

const app = express();
app.use(bodyParser.urlencoded({ extended: false }));

app.post("/contacto", async (req, res) => {

  const transporter = nodemailer.createTransport({
    service: "gmail",
    auth: {
      user: "ndrepruebas@gmail.com",
      pass: "CONTRASEÑA_APP"
    }
  });

  await transporter.sendMail({
    from: req.body.email,
    to: "ndrepruebas@gmail.com",
    subject: req.body.asunto,
    text: `
Nombre: ${req.body.nombre}
Email: ${req.body.email}

Mensaje:
${req.body.mensaje}
`
  });

  res.send("Mensaje enviado correctamente");
});

app.listen(3000, () => {
  console.log("Servidor iniciado en puerto 3000");
});
```

Resultado:
El email llega a Gmail → Zammad lo convierte en ticket.

---

## 12.3 Crear Ticket vía API Zammad

Endpoint:

POST /api/v1/tickets

Headers:

Authorization: Token token=API_TOKEN
Content-Type: application/json

Ejemplo Body:

```json
{
  "title": "Asunto del formulario",
  "group": "Users",
  "customer": "cliente@email.com",
  "article": {
    "subject": "Asunto",
    "body": "Mensaje del cliente",
    "type": "note",
    "internal": false
  }
}
```

Generar API Token en:

Admin → Security → API

---

# 13. CHATBOT WEB CON CHATWOOT

## 13.1 Crear Inbox Website

Chatwoot:

Settings → Inboxes → Add Inbox → Website

Copiar Website Token.

---

## 13.2 Insertar Script en la Web

Antes de </body>:

```html
<script>
(function(d,t) {
  var BASE_URL="https://chat.midominio.com";
  var g=d.createElement(t),s=d.getElementsByTagName(t)[0];
  g.src=BASE_URL+"/packs/js/sdk.js";
  s.parentNode.insertBefore(g,s);
  g.onload=function(){
    window.chatwootSDK.run({
      websiteToken: 'TOKEN_AQUI',
      baseUrl: BASE_URL
    })
  }
})(document,"script");
</script>
```

---

## 13.3 Automatización básica chatbot

En Chatwoot:

Automation → Rules → Add Rule

Ejemplo:

Condición:
Conversation created

Acción:
Send message

Mensaje:
"Gracias por contactarnos. En breve un agente le responderá."

---

# 14. ERRORES COMUNES

## Instagram no conecta

* No es Business
* No vinculada a Página
* App en modo Development
* Token expirado

## Webhook no funciona

* Trigger mal creado
* URL incorrecta
* Odoo no reiniciado
* JSON mal formado

## Gmail no conecta

* No usar contraseña normal
* No activar 2FA
* Puerto incorrecto

---

# 15. RECOMENDACIONES PRODUCCIÓN

* HTTPS obligatorio
* Token secreto en endpoint Odoo
* Validar firma webhook
* App Meta en Live Mode
* No exponer endpoint sin validación
* Backups diarios

---

# 16. PRINCIPIO ARQUITECTÓNICO

Zammad → Soporte
Chatwoot → Entrada conversaciones
Odoo → CRM
Web → Captación

Separación estricta de responsabilidades.

---


