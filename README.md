

---

# 🧪 Fasttify: Sistema de Tiendas con Plantillas Dinámicas

Renderiza tiendas personalizadas como `tienda1.fasttify.com`, usando LiquidJS y plantillas heredadas desde una base común almacenada en S3, renderizadas dinámicamente en tiempo real con Next.js 15 y AWS Amplify.

---

## 🛠 Tecnologías usadas

* **Next.js 15 (App Router, SSR)**
* **LiquidJS** para plantillas heredadas (`base.liquid`)
* **AWS S3** para almacenamiento de plantillas
* **AWS DynamoDB** para datos por tienda
* **Amplify Hosting + SSR**
* **AWS SDK v3** (`@aws-sdk/client-s3`, `@aws-sdk/client-dynamodb`)

---

## 📁 Estructura del proyecto

```
📁 /app
   └── page.tsx            ← Entrada principal
📁 /lib
   └── render.ts           ← Lógica SSR que obtiene y renderiza la tienda
📁 /liquid                 ← Local dev (mock de plantillas)

🪣 AWS S3
   - fasttify-base-plantillas/base.liquid
   - fasttify-tiendas/tienda1/home.liquid

🗃️ DynamoDB
   Tabla: FasttifyTiendas
```

---

## 🔁 Flujo de funcionamiento

1. Usuario entra a `tienda1.fasttify.com`
2. Next.js 15 (SSR con App Router) detecta el dominio
3. Se hace SSR en el servidor:

   * Busca en DynamoDB los datos y la plantilla
   * Descarga la plantilla de S3
   * Usa `base.liquid` como layout base
   * Renderiza con LiquidJS
4. Devuelve el HTML

---

## 🧩 Código

### `/app/page.tsx`

```tsx
import { renderTienda } from '@/lib/render';

export default async function Page() {
  const html = await renderTienda();

  return (
    <div dangerouslySetInnerHTML={{ __html: html }} />
  );
}
```

---

### `/lib/render.ts`

```ts
import { Liquid } from 'liquidjs';
import { S3Client, GetObjectCommand } from '@aws-sdk/client-s3';
import { DynamoDBClient, GetItemCommand } from '@aws-sdk/client-dynamodb';
import { headers } from 'next/headers';

const s3 = new S3Client({ region: 'us-east-1' });
const ddb = new DynamoDBClient({ region: 'us-east-1' });

const engine = new Liquid({
  extname: '.liquid',
  fs: {
    async readFile(filepath) {
      const bucket = filepath === 'base.liquid'
        ? 'fasttify-base-plantillas'
        : 'fasttify-tiendas';

      const res = await s3.send(new GetObjectCommand({
        Bucket: bucket,
        Key: filepath
      }));

      return await res.Body!.transformToString();
    },
    async exists() {
      return true;
    }
  }
});

export async function renderTienda(): Promise<string> {
  const host = headers().get('host')!;
  const dominio = host.replace(/^www\./, '');

  const { Item } = await ddb.send(new GetItemCommand({
    TableName: 'FasttifyTiendas',
    Key: { dominio: { S: dominio } }
  }));

  if (!Item) return `<h1>Tienda no encontrada</h1>`;

  const templateKey = Item.templateKey.S!;
  const datos = JSON.parse(Item.datos.S!);

  return await engine.renderFile(templateKey, datos);
}
```

---

## 🧪 Plantillas en S3

### Bucket: `fasttify-base-plantillas`

Archivo `base.liquid`:

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

### Bucket: `fasttify-tiendas`

Archivo `tienda1/home.liquid`:

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

## 🧠 DynamoDB

Tabla: `FasttifyTiendas`

Ejemplo de ítem:

```json
{
  "dominio": { "S": "tienda1.fasttify.com" },
  "templateKey": { "S": "tienda1/home.liquid" },
  "datos": {
    "S": "{\"tienda\": \"La tienda de Juan\", \"productos\": [{\"nombre\": \"Gorra\", \"precio\": 25000}]}"
  }
}
```

---

## 🌐 Subdominios personalizados

* Configura un registro DNS `*.fasttify.com` que apunte al dominio principal de Amplify
* Asegúrate de que Amplify tenga habilitado **SSR**

---

## 🧪 Test rápido

Visita:

```
https://tienda1.fasttify.com
```

Y verás la tienda renderizada con su plantilla y datos únicos.

---

## 🚀 Mejores prácticas y mejoras futuras

* Separar bloques de diseño en componentes Liquid
* Agregar variables globales (colores, fuentes) por tienda
* Editor visual para modificar plantillas
* Multilenguaje en los datos

---

