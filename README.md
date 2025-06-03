
# 🧪 Fasttify: Sistema de Tiendas con Plantillas Dinámicas

Renderiza tiendas personalizadas como `tienda1.fasttify.com`, usando LiquidJS y plantillas heredadas desde una base común almacenada en S3, renderizadas dinámicamente en tiempo real con Next.js 15 y AWS Amplify.

---

## 🛠 Tecnologías usadas

* **Next.js 15 (App Router, SSR)**
* **LiquidJS** para plantillas heredadas (`base.liquid`) y bloques dinámicos
* **AWS S3** para almacenamiento de plantillas y secciones
* **AWS DynamoDB** para datos por tienda (estructura por secciones)
* **Amplify Hosting + SSR**
* **AWS SDK v3** (`@aws-sdk/client-s3`, `@aws-sdk/client-dynamodb`)

---

## 📁 Estructura del proyecto

```
📁 /app
   └── page.tsx            ← Entrada principal
📁 /lib
   └── render.ts           ← Lógica SSR que obtiene y renderiza la tienda
📁 /liquid/secciones       ← Plantillas por sección: hero.liquid, productos.liquid, etc.

🪣 AWS S3
   - fasttify-base-plantillas/base.liquid
   - fasttify-secciones/hero.liquid
   - fasttify-secciones/productos.liquid

🗃️ DynamoDB
   Tabla: FasttifyTiendas
```

---

## 🔁 Flujo de funcionamiento

1. Usuario entra a `tienda1.fasttify.com`
2. Next.js 15 (SSR) detecta el dominio
3. Se consulta DynamoDB:

   * Lista ordenada de secciones (`tipo`, `settings`)
4. Se renderiza cada sección desde S3 usando LiquidJS
5. El HTML resultante se inserta dentro de `base.liquid`

---

## 🧠 DynamoDB: Datos por tienda

```json
{
  "dominio": { "S": "tienda1.fasttify.com" },
  "secciones": {
    "S": JSON.stringify([
      {
        "tipo": "hero",
        "settings": {
          "titulo": "Bienvenido a mi tienda",
          "descripcion": "Los mejores productos"
        }
      },
      {
        "tipo": "productos",
        "settings": {
          "titulo": "Nuestros productos"
        }
      }
    ])
  }
}
```

---

## 🧩 Ejemplo de sección: `hero.liquid`

```liquid
<section class="hero">
  <h1>{{ section.settings.titulo }}</h1>
  <p>{{ section.settings.descripcion }}</p>
</section>

{% schema %}
{
  "name": "Hero",
  "settings": [
    { "type": "text", "id": "titulo", "label": "Título", "default": "Título por defecto" },
    { "type": "text", "id": "descripcion", "label": "Descripción", "default": "Descripción por defecto" }
  ]
}
{% endschema %}
```

---

## `/lib/render.ts` actualizado (por secciones)

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
      const [bucket, key] = filepath.startsWith('base.liquid')
        ? ['fasttify-base-plantillas', 'base.liquid']
        : ['fasttify-secciones', filepath];

      const res = await s3.send(new GetObjectCommand({ Bucket: bucket, Key: key }));
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

  const secciones = JSON.parse(Item.secciones.S!);
  const bloquesHtml: string[] = [];

  for (const bloque of secciones) {
    const html = await engine.renderFile(`${bloque.tipo}.liquid`, {
      section: { settings: bloque.settings }
    });
    bloquesHtml.push(html);
  }

  const layoutHtml = await engine.renderFile('base.liquid', {
    contenido: bloquesHtml.join('\n')
  });

  return layoutHtml;
}
```

---

## 🧪 base.liquid en S3

```liquid
<!DOCTYPE html>
<html>
  <head>
    <title>Fasttify</title>
  </head>
  <body>
    {{ contenido }}
  </body>
</html>
```

---

## 🚀 Próximas mejoras

✅ Separación de secciones editable tipo Shopify
✅ Uso de `{% schema %}` para el editor visual
🛠 Editor visual tipo arrastrar y soltar (drag & drop)
🧩 Variables de diseño global (fuentes, colores, etc.)
🌎 Multilenguaje por tienda
🔐 Autenticación para el editor


