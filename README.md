
# Fasttify: Tienda Headless con LiquidJS, Next.js y AWS

## Visión general

Fasttify es un sistema **headless** de comercio electrónico multi-tenant que combina Next.js, LiquidJS y servicios de AWS. Cada tienda se accede mediante un subdominio personalizado, y la apariencia del sitio se define mediante plantillas al estilo de Shopify Online Store 2.0 (con layouts, secciones, bloques y esquemas JSON). Los datos de productos y colecciones se almacenan en **DynamoDB**, mientras que las plantillas Liquid (`.liquid`) se guardan en **AWS S3**.

## Arquitectura inspirada en Shopify Online Store 2.0

La arquitectura de Fasttify se inspira en los temas de Shopify Online Store 2.0. Cada tienda dispone de un **layout base** (p. ej. `theme.liquid`) que incluye el HTML común (cabecera, pie de página, etc.) y usa los objetos Liquid `{{ content_for_header }}` y `{{ content_for_layout }}` para insertar los contenidos específicos de cada página. Este layout base actúa como envoltorio en el que se renderizan las secciones dinámicas.

El diseño de la página se describe mediante un archivo **JSON de template** (similar a los JSON templates de Shopify). Este JSON especifica el nombre del layout base y las secciones a renderizar en orden. Por ejemplo, el JSON raíz contiene atributos como `layout` (nombre del layout base o `false`), `sections` (definición de cada sección) y `order` (array con el orden de IDs). El campo `layout` indica el archivo de layout a usar (por ejemplo `"theme"` para `layout/theme.liquid`), mientras que `sections` es un objeto cuyos valores incluyen el tipo de sección y sus configuraciones, y `order` lista las secciones en el orden en que deben mostrarse.

Las **secciones** son fragmentos `.liquid` independientes que se renderizan dinámicamente. Cada sección puede incluir dentro de su archivo un bloque `{% schema %}` con un esquema JSON que define sus opciones (settings) y bloques. Esto permite personalizar los contenidos desde un editor visual. Por ejemplo, el archivo `sections/hero.liquid` podría ser:

```liquid
<div class="hero">
  <h1>{{ section.settings.heading }}</h1>
  <p>{{ section.settings.subheading }}</p>
</div>

{% schema %}
{
  "name": "Hero Section",
  "settings": [
    { "type": "text", "id": "heading", "label": "Encabezado",   "default": "Bienvenido" },
    { "type": "text", "id": "subheading", "label": "Subtítulo",    "default": "Descripción" }
  ]
}
{% endschema %}
```

En este ejemplo la sección **Hero** define un encabezado y un párrafo que usan `section.settings.heading` y `section.settings.subheading`. El bloque `{% schema %}` describe dos campos editables de tipo texto. Fasttify leerá estos valores (por defecto o personalizados) del JSON del layout y los pasará al motor Liquid para generar el HTML dinámico.

## Detección de la tienda por dominio

Al llegar una petición a Fasttify, se determina la tienda activa a partir del dominio o subdominio de la URL. En Next.js esto se suele hacer en `getServerSideProps`, leyendo la cabecera `Host`. Por ejemplo:

```ts
  import { headers } from 'next/headers';
  const domain = headers().get('host');  // p.ej. 'tienda1.fasttify.com'

  // Buscar la tienda cuyo customDomain coincide
  const stores = await cookiesClient.models.UserStore.listUserStoreByCustomDomain(domain);
  const store = stores?.[0];
  if (!store) {
    // Redirigir o mostrar 404 si no se encuentra la tienda
    return { notFound: true };
  }
  // Aquí ya tenemos store.id para usar en las siguientes consultas...
  return { props: { storeId: store.id } };
}
```

Este código usa headers  (SSR) para manejar cada petición en el servidor. Con `cookiesClient.models.UserStore.listUserStoreByCustomDomain(domain)` se obtiene la entidad **UserStore** cuyo campo `customDomain` coincide con el dominio recibido. Si existe, `store.id` se usa luego para cargar el layout y los datos asociados a esa tienda.

## Modelos de datos (TypeScript)

Los principales modelos de datos en Fasttify se tipan en TypeScript como sigue:

```ts
interface UserStore {
  id: string;           // ID único de la tienda
  name: string;         // Nombre de la tienda
  customDomain: string; // Dominio personalizado (e.g. tienda.ejemplo.com)
  // ...otros campos de configuración de la tienda
}

interface Product {
  id: string;           // ID del producto
  storeId: string;      // Referencia a UserStore.id
  title: string;        // Título del producto
  description?: string; // Descripción
  price: number;        // Precio
  imageUrl?: string;    // URL de imagen
  // ...otros campos adicionales
}

interface Collection {
  id: string;           // ID de la colección
  storeId: string;      // Referencia a UserStore.id
  title: string;        // Nombre de la colección
  productIds: string[]; // IDs de productos incluidos
  // ...otros campos adicionales
}

interface SectionData {
  type: string;             // Nombre de la plantilla de sección (archivo .liquid)
  settings: Record<string, any>;  // Configuraciones de la sección según JSON
  blocks?: any[];           // Bloques opcionales definidos en schema
}

interface LayoutJSON {
  layout?: string;             // Nombre del layout base (p.ej. "theme"), o false
  sections: Record<string, SectionData>; // Secciones por ID
  order: string[];             // IDs de secciones en el orden a renderizar
}
```

Aquí `LayoutJSON` modela el JSON que viene de la entidad **StoreTemplate** (campo `templateData`). Cada sección referenciada en `LayoutJSON.sections` corresponde a un archivo `.liquid` en S3.

## Flujo de renderizado

### Obtención del layout y datos

Una vez identificada la tienda activa, se carga su configuración de tema desde la base de datos. Por ejemplo:

```ts
const [template] = await cookiesClient.models.StoreTemplate.listStoreTemplateByStoreId(storeId);
const layoutJSON = JSON.parse(template.templateData) as LayoutJSON;
```

El objeto `layoutJSON` contiene el nombre del layout base (`layoutJSON.layout`) y los detalles de cada sección (`layoutJSON.sections` y `layoutJSON.order`).

A continuación, se obtienen los datos de negocio necesarios: los productos y colecciones de la tienda, almacenados en DynamoDB. Por ejemplo:

```ts
const products = await cookiesClient.models.Product.listProductsByStoreId(storeId);
const collections = await cookiesClient.models.Collection.listCollectionsByStoreId(storeId);
```

DynamoDB es un servicio de base de datos NoSQL administrado por AWS, altamente escalable, ideal para almacenar colecciones de productos por tienda.

### Renderizado de secciones con LiquidJS

Fasttify usa **LiquidJS** para procesar plantillas Liquid en Node.js. Se crea un motor Liquid y se carga cada archivo de sección almacenado en un bucket S3 (por ejemplo `fasttify-templates`). AWS S3 es un servicio de almacenamiento de objetos perfecto para archivos estáticos. Un ejemplo de función para cargar la plantilla de S3 podría ser:

```ts
import { S3Client, GetObjectCommand } from "@aws-sdk/client-s3";
import { Liquid } from "liquidjs";

const s3 = new S3Client({ region: "us-east-1" });
const liquid = new Liquid();

// Función auxiliar: obtener texto de plantilla desde S3
async function loadTemplateFromS3(bucket: string, key: string): Promise<string> {
  const response = await s3.send(new GetObjectCommand({ Bucket: bucket, Key: key }));
  return await response.Body.transformToString();
}

const sectionsHtml: string[] = [];
for (const sectionId of layoutJSON.order) {
  const sectionDef = layoutJSON.sections[sectionId];
  const sectionKey = `sections/${sectionDef.type}.liquid`;
  const sectionTpl = await loadTemplateFromS3("fasttify-templates", sectionKey);
  // Renderizar con LiquidJS pasando la configuración y datos
  const html = await liquid.parseAndRender(sectionTpl, {
    section: sectionDef.settings,
    blocks: sectionDef.blocks,
    products,
    collections
  });
  sectionsHtml.push(html);
}
```

Este código recorre las secciones según `layoutJSON.order`. Por cada sección obtiene su plantilla Liquid (`*.liquid`) desde S3 y la renderiza usando `liquid.parseAndRender`. Se pasan como contexto los ajustes de la sección (`section.settings`), los bloques (`section.blocks`), y los datos de `products` y `collections`. De esta forma se genera el HTML parcial de cada sección.  (El sitio oficial de LiquidJS muestra un ejemplo similar: al crear un motor Liquid, llamar a `engine.renderFile("hello", {name: 'alice'})` lee y renderiza `hello.liquid`.)

### Renderizado del layout base

Tras generar las secciones individuales, se carga el layout base desde S3 (por ejemplo `layout/theme.liquid`) y se renderiza, inyectando el contenido de las secciones. Por ejemplo:

```ts
const layoutKey = `layout/${layoutJSON.layout || "theme"}.liquid`;
const layoutTpl = await loadTemplateFromS3("fasttify-templates", layoutKey);
const finalHtml = await liquid.parseAndRender(layoutTpl, {
  content_for_layout: sectionsHtml.join(""), // inserta las secciones juntas
  store: storeData,
  products,
  collections
});
```

En el layout base, el marcador `{{ content_for_layout }}` se reemplaza por la concatenación de todas las secciones renderizadas. El resultado `finalHtml` es el HTML completo que se envía al cliente. De esta forma, el layout base actúa como esqueleto general (cabecera, pie, etc.) y las secciones aportan el contenido específico de la página.

### Datos de productos y colecciones

Los modelos **Product** y **Collection** se consultan filtrando por `storeId`. Por ejemplo, usamos `cookiesClient.models.Product.listProductsByStoreId(storeId)` para obtener todos los productos de la tienda desde DynamoDB, y de manera similar para las colecciones. Estos datos se pasan al contexto de Liquid para que las secciones puedan mostrarlos (por ejemplo, una sección de productos destacados recorrerá el arreglo `products` o `collections`).

## Ejemplo de sección en un archivo `.liquid`

A modo de ilustración, supongamos una sección `sections/featured-products.liquid` que muestra los productos de una colección. El código podría ser:

```liquid
<div class="featured-products">
  {% for prodId in collections['nuevos'].productIds %}
    <div class="product">
      <h2>{{ products[prodId].title }}</h2>
      <p>{{ products[prodId].description }}</p>
    </div>
  {% endfor %}
</div>

{% schema %}
{
  "name": "Productos Destacados",
  "settings": [
    { "type": "text",       "id": "section_title", "label": "Título de sección", "default": "Nuevos Productos" },
    { "type": "collection", "id": "collection",    "label": "Colección",        "default": "nuevos" }
  ]
}
{% endschema %}
```

En este ejemplo, la sección recorre los IDs de productos de la colección `"nuevos"`. El bloque `{% schema %}` define las configuraciones de la sección: aquí un título (`section_title`) y la colección a usar (`collection`). Fasttify leerá estos valores y los pasará a `section.settings` para generar el contenido final.

## Conclusión

Fasttify implementa un flujo headless completo donde cada petición determina la tienda por subdominio, obtiene su configuración de tema (layout y secciones) desde la base de datos, renderiza dinámicamente las secciones Liquid almacenadas en S3, e inyecta los datos de productos/colecciones desde DynamoDB. Esto permite crear tiendas altamente personalizables y escalables aprovechando tecnologías modernas (Next.js SSR, LiquidJS, AWS S3 y DynamoDB) y el modelo de temas modulares de Shopify 2.0.
