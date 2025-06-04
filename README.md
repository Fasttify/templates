
# Fasttify: Sistema de tiendas headless con LiquidJS y AWS

Fasttify es un sistema headless para crear tiendas dinámicas basado en LiquidJS y servicios AWS (DynamoDB, S3, etc.), inspirado en la arquitectura de Shopify Online Store 2.0. Sigue el modelo de *JSON templates* de Shopify – cada página se define mediante un JSON de secciones ordenadas – y usa plantillas Liquid para el marcado. Por ejemplo, en Shopify cada tema tiene layouts, templates, secciones y bloques que interactúan; Fasttify adapta esto usando Next.js y AWS.

## Flujo de renderizado y detección de tienda

Cuando un cliente navega a un subdominio personalizado (por ejemplo `tienda1.fasttify.com`), una función del servidor Next.js obtiene el nombre de host de la petición HTTP usando la API de cabeceras. Por ejemplo:

```js
import { headers } from 'next/headers';
const host = headers().get('host');  // p.ej. 'tienda1.fasttify.com'
```

Este fragmento de Next.js (App Router) extrae el dominio actual. A continuación, Fasttify consulta la tabla **UserStore** en DynamoDB usando un índice secundario global (GSI) sobre el campo `customDomain`. Un ejemplo de query en TypeScript podría ser:

```ts
import { cookiesClient } from '@/utils/AmplifyUtils'

async function checkDomain(customDomain: string) {
  try {
    const { data: stores } = await cookiesClient.models.UserStore.listUserStoreByCustomDomain({
      customDomain: customDomain,
    })
```

En este ejemplo se especifica `IndexName: "customDomain-index"` para indicar el GSI apropiado. El resultado devuelve el registro de la tienda asociada a ese dominio, que incluye el `storeId` y la configuración de onboarding. (Usar un **Global Secondary Index** es una práctica común en DynamoDB para consultas rápidas por un atributo no clave, como `customDomain`.)

## Configuración de layout (JSON de secciones)

Dentro de los datos de la tienda (por ejemplo en `onboardingData.layout` de **UserStore**) se almacena la configuración del *layout* en formato JSON compatible con Shopify 2.0. Este JSON contiene un objeto `sections` con las secciones de la página y sus `settings`, y un arreglo `order` con los IDs de sección en el orden de renderizado. Por ejemplo:

```json
{
  "layout": "full-width",
  "sections": {
    "hero": {
      "type": "hero",
      "settings": { "title": "Bienvenido", "image": "hero.jpg" }
    },
    "featured": {
      "type": "featured-products",
      "settings": { "count": 4 }
    }
  },
  "order": ["hero", "featured"]
}
```

Según la documentación de Shopify, los JSON templates almacenan “una lista de secciones para renderizar y sus settings” y el array `order` indica el orden de renderizado. Fasttify usa este JSON para saber qué secciones cargar del tema: el objeto `sections` mapea cada sección a sus datos y el array `order` determina la secuencia en que se renderizan.

## Renderizado dinámico con LiquidJS

Para generar el HTML final, Fasttify emplea **LiquidJS**, un motor de plantillas compatible con Liquid de Shopify. Se define un layout base (`base.liquid`, similar a `theme.liquid` de Shopify) que incluye el contenido común (HTML, encabezado, pie, etc.). En este archivo base se insertan marcadores de posición (p.ej. `{{ content_for_layout }}` o tags `{% include %}`) donde se inyectarán las secciones dinámicas.

Luego, el código JavaScript inicializa LiquidJS y carga las plantillas. Por ejemplo:

```js
import { Liquid } from 'liquidjs';
const engine = new Liquid({ root: '/path/to/sections/', extname: '.liquid' });
// Renderizar la sección "hero"
const htmlHero = await engine.renderFile('hero', { settings: { title: 'Hola Mundo' } });
```

Este fragmento carga `hero.liquid` desde el directorio de secciones y la procesa con los datos (`settings`) dados. De forma similar, se renderiza `base.liquid` pasando un contexto que contenga los contenidos renderizados de cada sección. LiquidJS soporta etiquetas como `{% include %}`, bloques y filtros al estilo Shopify, permitiendo construir la página completa a partir de las secciones dinámicas.

## Integración con Productos y Colecciones

Muchas secciones presentan datos del catálogo. Fasttify conecta con tablas DynamoDB separadas, p.ej. **Product** y **Collection**, filtrando por el `storeId` obtenido del **UserStore**. Por ejemplo, la tabla *Product* puede tener un atributo `storeId` en su clave de partición, de modo que una query como `KeyConditionExpression: 'storeId = :id'` devuelve todos los productos de esa tienda. Este patrón multitenant (aislar datos por tienda) es común en DynamoDB. Al renderizar una sección como `featured-products`, Fasttify obtiene los productos reales (`title`, `price`, `image`, etc.) de DynamoDB y los pasa al contexto Liquid. De forma análoga, la tabla *Collection* puede listar los IDs de producto asociados a una colección. En resumen, cada sección dinámica carga los datos reales necesarios desde la base de datos usando `storeId` como filtro, e inyecta esos datos en la plantilla Liquid para mostrarlos.

## Estructura de archivos y temas en S3

Las plantillas (.liquid) del tema se almacenan en AWS S3, organizadas por tienda. Por convención (siguiendo la estructura de Shopify) existirían carpetas como `layout/` y `sections/`. Por ejemplo, podríamos tener en S3:

```
fasttify-themes/
└── <storeId>/
    ├── layout/
    │   └── base.liquid
    └── sections/
        ├── hero.liquid
        ├── featured-products.liquid
        └── ... 
```

En tiempo de ejecución, el backend descarga estos archivos usando AWS SDK. Por ejemplo, en JavaScript:

```js
import { S3Client, GetObjectCommand } from "@aws-sdk/client-s3";
const client = new S3Client({});
const bucketName = "fasttify-themes";
const key = `${storeId}/sections/hero.liquid`;
const { Body } = await client.send(new GetObjectCommand({ Bucket: bucketName, Key: key }));
const liquidTemplate = await new Response(Body).text();
```

Este código usa `GetObjectCommand` para obtener `hero.liquid` del bucket. Luego `liquidTemplate` (que contiene el texto Liquid) se pasa a `engine.parseAndRender` o `render` de LiquidJS junto con los datos para producir el HTML. De este modo, Fasttify puede almacenar cada tema de usuario en S3 y cargar dinámicamente solo los archivos necesarios.

## Tipos en TypeScript

Se definen interfaces para tipar los datos principales. Por ejemplo:

```ts
interface SectionData {
  type: string;
  settings: Record<string, any>;
}
interface LayoutJSON {
  layout: string | false;
  sections: Record<string, SectionData>;
  order: string[];
}
interface UserStore {
  storeId: string;
  customDomain: string;
  onboardingData: { layout: LayoutJSON };
  // ...otros campos...
}
interface Product {
  id: string;
  storeId: string;
  title: string;
  price: number;
  // ...otros campos...
}
interface Collection {
  id: string;
  storeId: string;
  title: string;
  productIds: string[];
  // ...otros campos...
}
```

Estas interfaces ilustran cómo tipar los registros de DynamoDB: `UserStore` para la tienda y su configuración, `LayoutJSON` para el JSON de layout, y `Product`/`Collection` para elementos del catálogo. El soporte de TypeScript asegura coherencia al manejar estos objetos en el código.

## Editor visual y esquema de secciones

Fasttify admite un editor visual de temas basado en los esquemas de sección (como en Shopify 2.0). Cada archivo de sección `.liquid` incluye un bloque `{% schema %}` con metadatos JSON (nombre de la sección, tipos de ajustes, bloques, presets, etc.). Por ejemplo, `hero.liquid` podría declarar:

```liquid
{% schema %}
{
  "name": "Hero",
  "settings": [
    { "type": "text",  "id": "title", "label": "Título" },
    { "type": "color", "id": "bg_color", "label": "Color de fondo" }
  ]
}
{% endschema %}
```

Este JSON dentro de `{% schema %}` indica al editor qué opciones mostrar. Fasttify puede leer este esquema para generar formularios de configuración visual donde el administrador ajusta los `settings` de cada sección sin escribir código. En conjunto, esto permite emular la experiencia de personalización de Shopify: el usuario reordena secciones y edita sus opciones mediante la interfaz gráfica, mientras que Fasttify almacena esos cambios en el JSON de layout y vuelve a renderizar la tienda dinámicamente.

**Resumen:** Fasttify combina la detección de dominio (Next.js), almacenamiento en DynamoDB de la configuración de tienda, archivos Liquid en S3 y renderizado con LiquidJS para crear páginas de tienda dinámicas. Usa `storeId` como identificador de tienda (multitenancy), tipado TypeScript y esquemas de sección para ofrecer una experiencia similar a Shopify Online Store 2.0 en un stack moderno con AWS.


