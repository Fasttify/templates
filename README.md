
---

# 🧪 Fasttify: Sistema de Plantillas Dinámicas

Este proyecto permite renderizar tiendas personalizadas (tipo `tienda1.fasttify.com`) usando plantillas heredadas desde una base `base.liquid` central en S3, y renderizadas dinámicamente desde Next.js + Amplify con LiquidJS.

---

## 🛠 Tecnologías utilizadas

* **Next.js con SSR** (hospedado en AWS Amplify)
* **AWS S3** (para almacenar la plantilla base y plantillas personalizadas)
* **AWS DynamoDB** (para almacenar la configuración de cada tienda)
* **LiquidJS** (para renderizar las plantillas)
* **AWS SDK v3** (para interactuar con S3 y DynamoDB desde el backend)

---

## 📁 Estructura del proyecto

```
📁 /pages
   └── api
       └── render.ts ← renderiza las tiendas dinámicamente

📁 S3 Buckets
   - fasttify-base-plantillas/base.liquid
   - fasttify-tiendas/tienda1/home.liquid

🗃️ DynamoDB
   Tabla: FasttifyTiendas
   Item:
   {
     dominio: tienda1.fasttify.com,
     templateKey: tienda1/home.liquid,
     datos: {
       tienda: "La tienda de Juan",
       productos: [...]
     }
   }
```

---

## 🧩 ¿Cómo funciona?

1. Usuario entra a `tienda1.fasttify.com`.
2. Amplify SSR redirige a `/api/render`.
3. Se detecta el subdominio (`tienda1`).
4. Se buscan los datos y plantilla correspondiente en DynamoDB.
5. Se renderiza el HTML usando LiquidJS con base en `base.liquid` + plantilla hija.
6. Se responde el HTML al cliente.

---

## 🪣 Configuración de S3

### 1. Bucket: `fasttify-base-plantillas`

Archivo: `base.liquid`

```liquid
<!DOCTYPE html>
<html>
  <head><title>{{ tienda }}</title></head>
  <body>
    {% block contenido %}{% endblock %}
  </body>
</html>
```

---

### 2. Bucket: `fasttify-tiendas`

Archivo: `tienda1/home.liquid`

```liquid
{% extends 'base.liquid' %}

{% block contenido %}
  <h1>{{ tienda }}</h1>
  <ul>
    {% for producto in productos %}
      <li>{{ producto.nombre }} - ${{ producto.precio }}</li>
    {% endfor %}
  </ul>
{% endblock %}
```

---

## 🧠 Configuración de DynamoDB

Tabla: `FasttifyTiendas`

Ejemplo de ítem:

```json
{
  "dominio": { "S": "tienda1.fasttify.com" },
  "templateKey": { "S": "tienda1/home.liquid" },
  "datos": {
    "S": "{\"tienda\": \"Tienda de Juan\", \"productos\": [{\"nombre\": \"Gorra\", \"precio\": 25000}]}"
  }
}
```

---

## 🧾 Código del renderizador

Archivo: `/pages/api/render.ts`

```ts
import { Liquid } from 'liquidjs';
import { S3Client, GetObjectCommand } from '@aws-sdk/client-s3';
import { DynamoDBClient, GetItemCommand } from '@aws-sdk/client-dynamodb';
import type { NextApiRequest, NextApiResponse } from 'next';

const s3 = new S3Client({ region: 'us-east-1' });
const ddb = new DynamoDBClient({ region: 'us-east-1' });

const engine = new Liquid({
  extname: '.liquid',
  fs: {
    async readFile(filepath) {
      const bucket = filepath === 'base.liquid' ? 'fasttify-base-plantillas' : 'fasttify-tiendas';
      const command = new GetObjectCommand({ Bucket: bucket, Key: filepath });
      const res = await s3.send(command);
      return await res.Body.transformToString();
    },
    async exists() {
      return true;
    }
  }
});

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  const dominio = req.headers.host?.split(':')[0];

  const { Item } = await ddb.send(new GetItemCommand({
    TableName: 'FasttifyTiendas',
    Key: { dominio: { S: dominio } }
  }));

  if (!Item) return res.status(404).send('Tienda no encontrada');

  const templateKey = Item.templateKey.S!;
  const datos = JSON.parse(Item.datos.S!);

  const html = await engine.renderFile(templateKey, datos);

  res.setHeader('Content-Type', 'text/html');
  res.send(html);
}
```

---

## 🔁 Configuración de rutas (Next.js)

En `next.config.js`:

```js
module.exports = {
  async rewrites() {
    return [
      {
        source: '/',
        destination: '/api/render',
      }
    ];
  },
};
```

---

## 🌐 Subdominios personalizados

Asegúrate de:

* Configurar un DNS comodín (`*.fasttify.com`) apuntando a tu dominio de Amplify.
* Habilitar SSR en Amplify para que `/api/render` pueda ejecutarse correctamente.

---

## 🧪 Pruebas

Visita:

```
https://tienda1.fasttify.com/
```

Y deberías ver los datos y diseño personalizados para esa tienda.

---

## 🚀 Futuras mejoras

* Agregar caché con CloudFront.
* Agregar un panel para que usuarios editen sus plantillas en tiempo real.
* Soporte para múltiples páginas (`/producto`, `/contacto`, etc.).

---

