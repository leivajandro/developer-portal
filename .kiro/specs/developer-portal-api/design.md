b# Documento de Diseño: Developer Portal API

## Resumen General

El Developer Portal API es un sistema event-driven que genera automáticamente un Developer Portal centralizado mediante la agregación de especificaciones OpenAPI de múltiples microservicios. El sistema utiliza una arquitectura serverless basada en AWS Lambda y S3 para procesar y publicar documentación de forma incremental, con CloudFront y WAF obligatorios para distribución segura.

### Arquitectura Final

El sistema consta de los siguientes componentes principales:

1. **Pipeline de Publicación (GitLab CI)**: Los pipelines de GitLab CI publican especificaciones OpenAPI a un bucket S3 privado
2. **Event-Driven Processing**: S3 Event Notifications activan directamente una función Lambda cuando se publican nuevas especificaciones
3. **Lambda Processor (Node.js)**: Función Lambda que procesa eventos S3, valida especificaciones, genera HTML usando Redoc, y actualiza el portal
4. **Portal Estático en S3**: Bucket S3 privado que aloja el sitio web estático con páginas de documentación generadas y un índice de servicios
5. **CloudFront Distribution**: Distribución CloudFront con Origin Access Identity (OAI) para acceso seguro a buckets S3 privados
6. **AWS WAF**: Web Application Firewall con reglas de protección, rate limiting y geo-blocking
7. **Dominio Personalizado**: Route 53 y ACM para certificado SSL/TLS en dominio personalizado (opcional)

### Decisiones Clave de Diseño

- **Lambda + Node.js**: Elegido porque Redoc (@redocly/openapi-core) es nativo de Node.js, proporcionando la mejor integración
- **Redoc para renderizado**: Utiliza la biblioteca @redocly/openapi-core para generar HTML standalone de alta calidad, eliminando la necesidad de templates personalizados
- **Event-driven con S3 triggers directos**: Lambda se activa directamente desde eventos S3 (sin SQS), simplificando la arquitectura y reduciendo latencia
- **Regeneración incremental**: Solo regenera las especificaciones que cambian, no reconstruye todo el sitio
- **S3 privado con CloudFront**: Buckets S3 privados con acceso controlado mediante Origin Access Identity (OAI)
- **CloudFront + WAF obligatorio**: Distribución segura con protección contra ataques DDoS, rate limiting, AWS Managed Rules y geo-blocking
- **Index page estático con datos dinámicos**: HTML estático que carga services.json dinámicamente para el catálogo
- **Terraform HCL para infraestructura**: Infraestructura como código usando Terraform HCL nativo
- **GitLab CI para publicación**: Pipeline de GitLab CI para publicar especificaciones y desplegar infraestructura
- **Enfoque en OpenAPI 3.x**: Soporte para especificaciones OpenAPI 3.0 y 3.1 usando bibliotecas de validación estándar


## Arquitectura

### Diagrama de Componentes

```mermaid
graph TB
    subgraph "GitLab CI Pipeline"
        A[Build de Microservicio]
        B[Especificación OpenAPI]
    end
    
    subgraph "S3 Specs Bucket privado"
        C[user-service/v1.0.0/openapi.json]
        D[order-service/v2.0.0/openapi.json]
        E[S3 Event Notification]
    end
    
    subgraph "Lambda Function Node.js"
        F[S3 Event Handler]
        G[OpenAPI Validator]
        H[Redoc Generator]
        I[Catalog Updater]
    end
    
    subgraph "S3 Portal Bucket privado"
        J[index.html]
        K[services.json]
        L[services/user-service-v1.0.0.html]
        M[services/order-service-v2.0.0.html]
        N[assets/]
    end
    
    subgraph "CloudFront + WAF"
        O[AWS WAF Web ACL]
        P[CloudFront Distribution]
        Q[Origin Access Identity]
        R[ACM Certificate]
        S[Route 53 opcional]
    end
    
    subgraph "Usuarios"
        T[Desarrolladores]
    end
    
    A --> B
    B --> C
    B --> D
    C --> E
    D --> E
    E --> F
    F --> G
    G --> H
    H --> L
    H --> M
    I --> K
    J --> Q
    K --> Q
    L --> Q
    M --> Q
    N --> Q
    Q --> P
    O --> P
    R --> P
    S --> P
    P --> T
```

### Flujo de Datos

1. **Publicación**: Los pipelines de GitLab CI publican especificaciones OpenAPI a S3 specs bucket privado (path: `{service-name}/{version}/openapi.json`)
2. **Event Notification**: S3 emite un evento cuando se crea/actualiza un objeto
3. **Lambda Trigger**: Lambda se activa automáticamente al recibir el evento S3 directamente (sin SQS)
4. **Descarga**: Lambda descarga la especificación OpenAPI desde S3
5. **Validación**: Lambda valida el formato y conformidad con OpenAPI 3.0/3.1
6. **Generación con Redoc**: Lambda usa @redocly/openapi-core para generar HTML standalone
7. **Upload HTML**: Lambda sube el HTML generado al portal bucket privado (path: `services/{service-name}-{version}.html`)
8. **Actualización de Catálogo**: Lambda actualiza el archivo `services.json` con los metadatos del nuevo servicio
9. **Acceso Seguro**: Los usuarios acceden al portal vía CloudFront (https://portal.example.com) con protección WAF
10. **CloudFront**: CloudFront usa OAI para acceder a los buckets S3 privados y sirve el contenido con certificado SSL/TLS

### Estructura de S3

```
specs-bucket/
├── user-service/
│   ├── v1.0.0/
│   │   └── openapi.json
│   └── v1.1.0/
│       └── openapi.json
└── order-service/
    └── v2.0.0/
        └── openapi.json

portal-bucket/
├── index.html                          # Página de catálogo estática
├── services.json                       # Metadatos dinámicos de servicios
├── assets/
│   ├── search.js                       # Búsqueda del lado del cliente
│   └── styles.css                      # Estilos con Redoc CSS
└── services/
    ├── user-service-v1.0.0.html       # Página Redoc generada
    ├── user-service-v1.1.0.html
    └── order-service-v2.0.0.html
```


## Componentes e Interfaces

### 1. Validador de OpenAPI

El validador asegura que las especificaciones cumplan con el estándar OpenAPI antes de procesarlas.

#### Interfaz del Validador

```typescript
interface OpenAPIValidator {
  validate(spec: object): ValidationResult;
  getSupportedVersions(): string[];
}

interface ValidationResult {
  valid: boolean;
  errors: ValidationError[];
  warnings: ValidationWarning[];
}

interface ValidationError {
  path: string;
  message: string;
  code: string;
}

interface ValidationWarning {
  path: string;
  message: string;
  suggestion?: string;
}
```

#### Reglas de Validación

- Verificar que la versión de OpenAPI sea 3.0.x o 3.1.x
- Validar que exista el campo `info` con `title` y `version`
- Verificar que `paths` esté definido y contenga al menos un endpoint
- Validar que todos los `$ref` apunten a definiciones existentes
- Verificar que los schemas tengan tipos válidos
- Validar que los códigos de respuesta sean válidos (100-599)

#### Implementación

```typescript
import SwaggerParser from '@apidevtools/swagger-parser';

class OpenAPIValidatorImpl implements OpenAPIValidator {
  async validate(spec: object): Promise<ValidationResult> {
    try {
      await SwaggerParser.validate(spec);
      return { valid: true, errors: [], warnings: [] };
    } catch (error) {
      return {
        valid: false,
        errors: this.parseValidationErrors(error),
        warnings: []
      };
    }
  }
  
  getSupportedVersions(): string[] {
    return ['3.0.0', '3.0.1', '3.0.2', '3.0.3', '3.1.0'];
  }
}
```


### 2. Buckets S3

#### Specs Bucket (Privado)

Almacena las especificaciones OpenAPI publicadas por los pipelines de GitLab CI.

**Estructura de Paths:**
- Pattern: `{service-name}/{version}/openapi.json`
- Ejemplo: `user-service/v1.0.0/openapi.json`

**Configuración:**
```json
{
  "versioning": "Enabled",
  "publicAccess": "BlockAll",
  "eventNotifications": [
    {
      "events": ["s3:ObjectCreated:*"],
      "destination": "lambda-function-arn"
    }
  ]
}
```

#### Portal Bucket (Privado)

Almacena el sitio web estático generado con las páginas de documentación.

**Configuración:**
```json
{
  "publicAccess": "BlockAll",
  "staticWebsiteHosting": {
    "enabled": true,
    "indexDocument": "index.html",
    "errorDocument": "error.html"
  },
  "corsConfiguration": {
    "allowedOrigins": ["https://portal.example.com"],
    "allowedMethods": ["GET", "HEAD"],
    "allowedHeaders": ["*"]
  }
}
```

**Acceso:**
- Solo accesible mediante CloudFront con Origin Access Identity (OAI)
- No hay acceso público directo al bucket S3

### 3. Función Lambda (Node.js)

La función Lambda es el componente central que procesa especificaciones OpenAPI y genera el portal.

#### Configuración de Lambda

```json
{
  "runtime": "nodejs18.x",
  "handler": "index.handler",
  "timeout": 300,
  "memorySize": 1024,
  "environment": {
    "SPECS_BUCKET": "my-openapi-specs",
    "PORTAL_BUCKET": "my-developer-portal",
    "REDOC_OPTIONS": "{\"theme\":{\"colors\":{\"primary\":{\"main\":\"#32329f\"}}}}"
  },
  "triggers": [
    {
      "type": "S3",
      "bucketArn": "arn:aws:s3:::my-openapi-specs",
      "events": ["s3:ObjectCreated:*"]
    }
  ]
}
```

#### Interfaz del Handler

```typescript
import { S3Event, S3EventRecord, Context } from 'aws-lambda';
import { S3Client, GetObjectCommand, PutObjectCommand } from '@aws-sdk/client-s3';
import { bundle } from '@redocly/openapi-core';

interface LambdaHandler {
  handler(event: S3Event, context: Context): Promise<void>;
}

interface ProcessingResult {
  serviceName: string;
  version: string;
  success: boolean;
  error?: string;
  outputPath?: string;
}
```

#### Implementación del Handler

```typescript
export async function handler(event: S3Event, context: Context): Promise<void> {
  const results: ProcessingResult[] = [];
  
  for (const record of event.Records) {
    try {
      const result = await processRecord(record);
      results.push(result);
    } catch (error) {
      console.error('Error processing record:', error);
      results.push({
        serviceName: 'unknown',
        version: 'unknown',
        success: false,
        error: error.message
      });
      // Re-throw para que Lambda reintente
      throw error;
    }
  }
  
  // Actualizar services.json con todos los resultados exitosos
  await updateServicesCatalog(results.filter(r => r.success));
  
  console.log('Processing complete:', JSON.stringify(results));
}

async function processRecord(record: S3EventRecord): Promise<ProcessingResult> {
  // 1. Extraer información del evento S3
  const bucket = record.s3.bucket.name;
  const key = record.s3.object.key;
  
  // 2. Extraer service name y version del path
  const [serviceName, version] = key.split('/');
  
  // 3. Descargar la especificación desde S3
  const spec = await downloadSpec(bucket, key);
  
  // 4. Validar la especificación
  const validation = await validator.validate(spec);
  if (!validation.valid) {
    throw new Error(`Invalid spec: ${JSON.stringify(validation.errors)}`);
  }
  
  // 5. Generar HTML con Redoc
  const html = await generateRedocHTML(spec, serviceName, version);
  
  // 6. Subir HTML al portal bucket
  const outputPath = `services/${serviceName}-${version}.html`;
  await uploadToPortal(outputPath, html);
  
  return {
    serviceName,
    version,
    success: true,
    outputPath
  };
}
```


### 4. Integración con Redoc

Utiliza la biblioteca @redocly/openapi-core para generar páginas de documentación HTML standalone.

#### Interfaz de Redoc Integration

```typescript
interface RedocIntegration {
  generateHTML(spec: OpenAPISpec, options: RedocOptions): Promise<string>;
  validateRedocCompatibility(spec: OpenAPISpec): ValidationResult;
}

interface RedocOptions {
  theme?: {
    colors?: {
      primary?: { main: string };
    };
    typography?: {
      fontSize?: string;
      fontFamily?: string;
    };
  };
  hideDownloadButton?: boolean;
  disableSearch?: boolean;
  expandResponses?: string;
  jsonSampleExpandLevel?: number;
  hideHostname?: boolean;
  pathInMiddlePanel?: boolean;
}
```

#### Implementación con @redocly/openapi-core

```typescript
import { bundle } from '@redocly/openapi-core';
import { createHTML } from './redoc-template';

async function generateRedocHTML(
  spec: OpenAPISpec,
  serviceName: string,
  version: string
): Promise<string> {
  // Bundle the spec (resolve all $refs)
  const { bundle: bundledSpec } = await bundle({
    ref: spec,
    config: {}
  });
  
  // Opciones de Redoc desde variables de entorno
  const redocOptions: RedocOptions = JSON.parse(
    process.env.REDOC_OPTIONS || '{}'
  );
  
  // Generar HTML standalone con Redoc embebido
  const html = createHTML({
    title: `${spec.info.title} - ${version}`,
    spec: bundledSpec,
    options: redocOptions
  });
  
  return html;
}

function createHTML(params: {
  title: string;
  spec: object;
  options: RedocOptions;
}): string {
  return `
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8"/>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>${params.title}</title>
  <style>
    body {
      margin: 0;
      padding: 0;
    }
  </style>
</head>
<body>
  <div id="redoc-container"></div>
  <script src="https://cdn.redoc.ly/redoc/latest/bundles/redoc.standalone.js"></script>
  <script>
    Redoc.init(
      ${JSON.stringify(params.spec)},
      ${JSON.stringify(params.options)},
      document.getElementById('redoc-container')
    );
  </script>
</body>
</html>
  `.trim();
}
```

#### Opciones de Redoc Recomendadas

```json
{
  "theme": {
    "colors": {
      "primary": {
        "main": "#32329f"
      }
    },
    "typography": {
      "fontSize": "14px",
      "fontFamily": "Inter, -apple-system, BlinkMacSystemFont, sans-serif"
    }
  },
  "hideDownloadButton": false,
  "disableSearch": false,
  "expandResponses": "200,201",
  "jsonSampleExpandLevel": 2,
  "hideHostname": false,
  "pathInMiddlePanel": false
}
```

#### Ventajas de Usar Redoc

- **Renderizado profesional**: Interfaz pulida y bien diseñada sin desarrollo personalizado
- **Características integradas**: Búsqueda, navegación, ejemplos de código, syntax highlighting
- **Mantenimiento reducido**: Redoc maneja la complejidad de renderizar especificaciones OpenAPI
- **Responsive por defecto**: Funciona bien en todos los tamaños de pantalla
- **HTML standalone**: Genera un único archivo HTML autocontenido con JavaScript embebido
- **Soporte completo de OpenAPI 3.x**: Maneja todas las características de OpenAPI 3.0 y 3.1


### 5. Generador de Catálogo

Actualiza el archivo `services.json` y mantiene la página `index.html` del catálogo de servicios.

#### Interfaz del Generador de Catálogo

```typescript
interface CatalogGenerator {
  updateServicesCatalog(newService: ServiceEntry): Promise<void>;
  generateSearchIndex(services: ServiceEntry[]): SearchIndex;
}

interface ServiceEntry {
  serviceName: string;
  displayName: string;
  description: string;
  versions: VersionEntry[];
  latestVersion: string;
  tags: string[];
  lastUpdated: string;
  url: string;
}

interface VersionEntry {
  version: string;
  publishedAt: string;
  url: string;
}

interface SearchIndex {
  services: SearchableService[];
}

interface SearchableService {
  id: string;
  name: string;
  displayName: string;
  description: string;
  tags: string[];
  versions: string[];
  latestVersion: string;
  url: string;
}
```

#### Implementación

```typescript
async function updateServicesCatalog(
  results: ProcessingResult[]
): Promise<void> {
  // 1. Descargar services.json actual desde S3
  let catalog: ServiceEntry[] = [];
  try {
    const existing = await downloadFromPortal('services.json');
    catalog = JSON.parse(existing);
  } catch (error) {
    console.log('No existing catalog, creating new one');
  }
  
  // 2. Actualizar con nuevos servicios
  for (const result of results) {
    const existingService = catalog.find(
      s => s.serviceName === result.serviceName
    );
    
    if (existingService) {
      // Agregar nueva versión
      existingService.versions.push({
        version: result.version,
        publishedAt: new Date().toISOString(),
        url: result.outputPath
      });
      existingService.latestVersion = result.version;
      existingService.lastUpdated = new Date().toISOString();
    } else {
      // Crear nuevo servicio
      catalog.push({
        serviceName: result.serviceName,
        displayName: result.serviceName,
        description: '', // Se puede extraer del spec
        versions: [{
          version: result.version,
          publishedAt: new Date().toISOString(),
          url: result.outputPath
        }],
        latestVersion: result.version,
        tags: [],
        lastUpdated: new Date().toISOString(),
        url: result.outputPath
      });
    }
  }
  
  // 3. Subir services.json actualizado
  await uploadToPortal('services.json', JSON.stringify(catalog, null, 2));
}
```

#### Estructura de services.json

```json
[
  {
    "serviceName": "user-service",
    "displayName": "User Service",
    "description": "Gestión de usuarios y autenticación",
    "versions": [
      {
        "version": "v1.0.0",
        "publishedAt": "2024-01-15T10:30:00Z",
        "url": "services/user-service-v1.0.0.html"
      },
      {
        "version": "v1.1.0",
        "publishedAt": "2024-01-20T14:45:00Z",
        "url": "services/user-service-v1.1.0.html"
      }
    ],
    "latestVersion": "v1.1.0",
    "tags": ["authentication", "users"],
    "lastUpdated": "2024-01-20T14:45:00Z",
    "url": "services/user-service-v1.1.0.html"
  },
  {
    "serviceName": "order-service",
    "displayName": "Order Service",
    "description": "Procesamiento de pedidos y pagos",
    "versions": [
      {
        "version": "v2.0.0",
        "publishedAt": "2024-01-18T09:15:00Z",
        "url": "services/order-service-v2.0.0.html"
      }
    ],
    "latestVersion": "v2.0.0",
    "tags": ["orders", "payments"],
    "lastUpdated": "2024-01-18T09:15:00Z",
    "url": "services/order-service-v2.0.0.html"
  }
]
```


#### Página Index.html

La página de índice es un archivo HTML estático que carga `services.json` dinámicamente y renderiza el catálogo.

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Developer Portal</title>
  <link rel="stylesheet" href="https://cdn.redoc.ly/redoc/latest/bundles/redoc.standalone.css">
  <link rel="stylesheet" href="assets/styles.css">
</head>
<body>
  <header class="portal-header">
    <div class="container">
      <h1>Developer Portal</h1>
      <p>Documentación de APIs de Microservicios</p>
    </div>
  </header>
  
  <main class="container">
    <div class="search-bar">
      <input 
        type="search" 
        id="search" 
        placeholder="Buscar servicios por nombre, descripción o tags..."
        aria-label="Buscar servicios"
      >
    </div>
    
    <div class="filters">
      <label>
        <input type="checkbox" id="filter-latest" checked>
        Mostrar solo última versión
      </label>
    </div>
    
    <div id="services-grid" class="services-grid">
      <!-- Tarjetas de servicios generadas dinámicamente -->
    </div>
    
    <div id="no-results" class="no-results" style="display: none;">
      <p>No se encontraron servicios que coincidan con tu búsqueda.</p>
    </div>
  </main>
  
  <footer class="portal-footer">
    <div class="container">
      <p>Última actualización: <span id="last-update"></span></p>
    </div>
  </footer>
  
  <script src="assets/search.js"></script>
  <script>
    // Cargar y renderizar servicios
    fetch('services.json')
      .then(response => response.json())
      .then(services => {
        initCatalog(services);
      })
      .catch(error => {
        console.error('Error loading services:', error);
        document.getElementById('services-grid').innerHTML = 
          '<p class="error">Error al cargar el catálogo de servicios.</p>';
      });
  </script>
</body>
</html>
```

#### JavaScript de Búsqueda (assets/search.js)

```javascript
let allServices = [];
let filteredServices = [];

function initCatalog(services) {
  allServices = services;
  filteredServices = services;
  
  renderServices(services);
  updateLastUpdate(services);
  
  // Event listeners
  document.getElementById('search').addEventListener('input', handleSearch);
  document.getElementById('filter-latest').addEventListener('change', handleFilterChange);
}

function handleSearch(event) {
  const query = event.target.value.toLowerCase();
  
  filteredServices = allServices.filter(service => 
    service.serviceName.toLowerCase().includes(query) ||
    service.displayName.toLowerCase().includes(query) ||
    service.description.toLowerCase().includes(query) ||
    service.tags.some(tag => tag.toLowerCase().includes(query))
  );
  
  renderServices(filteredServices);
}

function handleFilterChange(event) {
  const showLatestOnly = event.target.checked;
  renderServices(filteredServices, showLatestOnly);
}

function renderServices(services, showLatestOnly = true) {
  const grid = document.getElementById('services-grid');
  const noResults = document.getElementById('no-results');
  
  if (services.length === 0) {
    grid.style.display = 'none';
    noResults.style.display = 'block';
    return;
  }
  
  grid.style.display = 'grid';
  noResults.style.display = 'none';
  
  grid.innerHTML = services.map(service => {
    const versions = showLatestOnly 
      ? [service.versions[service.versions.length - 1]]
      : service.versions;
    
    return `
      <div class="service-card">
        <h3 class="service-name">${service.displayName}</h3>
        <p class="service-description">${service.description || 'Sin descripción'}</p>
        
        <div class="service-tags">
          ${service.tags.map(tag => `<span class="tag">${tag}</span>`).join('')}
        </div>
        
        <div class="service-versions">
          ${versions.map(v => `
            <a href="${v.url}" class="version-link">
              ${v.version}
            </a>
          `).join('')}
        </div>
        
        <div class="service-meta">
          <span class="service-version">Última versión: ${service.latestVersion}</span>
          <span class="service-updated">Actualizado: ${formatDate(service.lastUpdated)}</span>
        </div>
        
        <a href="${service.url}" class="btn-primary">Ver documentación →</a>
      </div>
    `;
  }).join('');
}

function formatDate(isoString) {
  const date = new Date(isoString);
  return date.toLocaleDateString('es-ES', {
    year: 'numeric',
    month: 'short',
    day: 'numeric'
  });
}

function updateLastUpdate(services) {
  const latest = services.reduce((max, service) => {
    const serviceDate = new Date(service.lastUpdated);
    return serviceDate > max ? serviceDate : max;
  }, new Date(0));
  
  document.getElementById('last-update').textContent = formatDate(latest.toISOString());
}
```


### 6. CloudFront Distribution

CloudFront proporciona distribución global del portal con baja latencia y acceso seguro a los buckets S3 privados.

#### Configuración de CloudFront

```typescript
interface CloudFrontConfig {
  originAccessIdentity: string; // OAI para acceder a S3 privado
  origins: Origin[];
  defaultCacheBehavior: CacheBehavior;
  customDomain?: string;
  certificateArn?: string; // ACM certificate
  webAclArn: string; // AWS WAF Web ACL
}

interface Origin {
  id: string;
  domainName: string; // S3 bucket domain
  s3OriginConfig: {
    originAccessIdentity: string;
  };
}

interface CacheBehavior {
  targetOriginId: string;
  viewerProtocolPolicy: 'redirect-to-https' | 'https-only';
  allowedMethods: string[];
  cachedMethods: string[];
  compress: boolean;
  defaultTTL: number;
  maxTTL: number;
  minTTL: number;
}
```

#### Origin Access Identity (OAI)

```json
{
  "comment": "OAI for Developer Portal",
  "callerReference": "developer-portal-oai"
}
```

**Política de Bucket S3:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontOAI",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::cloudfront:user/CloudFront Origin Access Identity XXXXX"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-developer-portal/*"
    }
  ]
}
```

#### Configuración de Caché

```json
{
  "defaultCacheBehavior": {
    "viewerProtocolPolicy": "redirect-to-https",
    "allowedMethods": ["GET", "HEAD", "OPTIONS"],
    "cachedMethods": ["GET", "HEAD"],
    "compress": true,
    "defaultTTL": 86400,
    "maxTTL": 31536000,
    "minTTL": 0,
    "forwardedValues": {
      "queryString": false,
      "cookies": {
        "forward": "none"
      }
    }
  }
}
```

#### Dominio Personalizado (Opcional)

**Route 53:**
```json
{
  "name": "portal.example.com",
  "type": "A",
  "aliasTarget": {
    "hostedZoneId": "Z2FDTNDATAQYW2",
    "dnsName": "d123456789.cloudfront.net",
    "evaluateTargetHealth": false
  }
}
```

**ACM Certificate:**
- Debe estar en región us-east-1 para CloudFront
- Validación mediante DNS (Route 53)
- Incluir dominio principal y wildcard (*.example.com)


### 7. AWS WAF (Web Application Firewall)

AWS WAF protege el portal contra ataques comunes y proporciona control de acceso granular.

#### Configuración de WAF Web ACL

```typescript
interface WAFWebACL {
  name: string;
  scope: 'CLOUDFRONT' | 'REGIONAL';
  defaultAction: 'ALLOW' | 'BLOCK';
  rules: WAFRule[];
  visibilityConfig: VisibilityConfig;
}

interface WAFRule {
  name: string;
  priority: number;
  statement: RuleStatement;
  action: 'ALLOW' | 'BLOCK' | 'COUNT';
  visibilityConfig: VisibilityConfig;
}

interface VisibilityConfig {
  sampledRequestsEnabled: boolean;
  cloudWatchMetricsEnabled: boolean;
  metricName: string;
}
```

#### Reglas de WAF Recomendadas

**1. Rate Limiting:**
```json
{
  "name": "RateLimitRule",
  "priority": 1,
  "statement": {
    "rateBasedStatement": {
      "limit": 2000,
      "aggregateKeyType": "IP"
    }
  },
  "action": {
    "block": {}
  }
}
```

**2. AWS Managed Rules:**
```json
{
  "name": "AWSManagedRulesCommonRuleSet",
  "priority": 2,
  "statement": {
    "managedRuleGroupStatement": {
      "vendorName": "AWS",
      "name": "AWSManagedRulesCommonRuleSet"
    }
  },
  "overrideAction": {
    "none": {}
  }
}
```

**3. AWS Managed Rules - Known Bad Inputs:**
```json
{
  "name": "AWSManagedRulesKnownBadInputsRuleSet",
  "priority": 3,
  "statement": {
    "managedRuleGroupStatement": {
      "vendorName": "AWS",
      "name": "AWSManagedRulesKnownBadInputsRuleSet"
    }
  },
  "overrideAction": {
    "none": {}
  }
}
```

**4. Geo-Blocking (Opcional):**
```json
{
  "name": "GeoBlockingRule",
  "priority": 4,
  "statement": {
    "geoMatchStatement": {
      "countryCodes": ["CN", "RU", "KP"]
    }
  },
  "action": {
    "block": {}
  }
}
```

**5. IP Reputation List:**
```json
{
  "name": "AWSManagedRulesAmazonIpReputationList",
  "priority": 5,
  "statement": {
    "managedRuleGroupStatement": {
      "vendorName": "AWS",
      "name": "AWSManagedRulesAmazonIpReputationList"
    }
  },
  "overrideAction": {
    "none": {}
  }
}
```

#### Logging y Monitoreo

```json
{
  "loggingConfiguration": {
    "resourceArn": "arn:aws:wafv2:us-east-1:123456789012:global/webacl/developer-portal/xxxxx",
    "logDestinationConfigs": [
      "arn:aws:s3:::aws-waf-logs-developer-portal"
    ],
    "redactedFields": []
  }
}
```

#### Métricas de CloudWatch

- `AllowedRequests` - Solicitudes permitidas
- `BlockedRequests` - Solicitudes bloqueadas
- `CountedRequests` - Solicitudes contadas (modo COUNT)
- `RateLimitedRequests` - Solicitudes limitadas por rate limiting

#### Alarmas Recomendadas

```json
{
  "alarmName": "HighBlockedRequests",
  "metricName": "BlockedRequests",
  "threshold": 1000,
  "evaluationPeriods": 2,
  "period": 300,
  "statistic": "Sum",
  "comparisonOperator": "GreaterThanThreshold"
}
```


## Modelos de Datos

### Especificación OpenAPI (Entrada)

```typescript
// Basado en OpenAPI 3.0/3.1
interface OpenAPISpec {
  openapi: string; // "3.0.0" | "3.1.0"
  info: Info;
  servers?: Server[];
  paths: Paths;
  components?: Components;
  security?: SecurityRequirement[];
  tags?: Tag[];
  externalDocs?: ExternalDocumentation;
}

interface Info {
  title: string;
  version: string;
  description?: string;
  termsOfService?: string;
  contact?: Contact;
  license?: License;
}

interface Contact {
  name?: string;
  url?: string;
  email?: string;
}

interface License {
  name: string;
  url?: string;
}

interface Server {
  url: string;
  description?: string;
  variables?: { [name: string]: ServerVariable };
}

interface ServerVariable {
  enum?: string[];
  default: string;
  description?: string;
}

interface Paths {
  [path: string]: PathItem;
}

interface PathItem {
  get?: Operation;
  post?: Operation;
  put?: Operation;
  delete?: Operation;
  patch?: Operation;
  options?: Operation;
  head?: Operation;
  trace?: Operation;
  parameters?: Parameter[];
}

interface Operation {
  summary?: string;
  description?: string;
  operationId?: string;
  parameters?: Parameter[];
  requestBody?: RequestBody;
  responses: Responses;
  security?: SecurityRequirement[];
  tags?: string[];
  deprecated?: boolean;
}

interface Parameter {
  name: string;
  in: 'query' | 'header' | 'path' | 'cookie';
  description?: string;
  required?: boolean;
  schema: Schema;
  example?: any;
  examples?: { [name: string]: Example };
}

interface RequestBody {
  description?: string;
  content: { [mediaType: string]: MediaType };
  required?: boolean;
}

interface Responses {
  [statusCode: string]: Response;
}

interface Response {
  description: string;
  content?: { [mediaType: string]: MediaType };
  headers?: { [name: string]: Header };
}

interface Header {
  description?: string;
  schema: Schema;
  required?: boolean;
}

interface MediaType {
  schema?: Schema;
  example?: any;
  examples?: { [name: string]: Example };
}

interface Example {
  summary?: string;
  description?: string;
  value?: any;
  externalValue?: string;
}

interface Schema {
  type?: string;
  format?: string;
  properties?: { [name: string]: Schema };
  required?: string[];
  items?: Schema;
  enum?: any[];
  minimum?: number;
  maximum?: number;
  minLength?: number;
  maxLength?: number;
  pattern?: string;
  description?: string;
  default?: any;
  nullable?: boolean;
  readOnly?: boolean;
  writeOnly?: boolean;
  $ref?: string;
  allOf?: Schema[];
  oneOf?: Schema[];
  anyOf?: Schema[];
}

interface Components {
  schemas?: { [name: string]: Schema };
  responses?: { [name: string]: Response };
  parameters?: { [name: string]: Parameter };
  examples?: { [name: string]: Example };
  requestBodies?: { [name: string]: RequestBody };
  headers?: { [name: string]: Header };
  securitySchemes?: { [name: string]: SecurityScheme };
}

interface SecurityScheme {
  type: 'apiKey' | 'http' | 'oauth2' | 'openIdConnect';
  description?: string;
  name?: string;
  in?: 'query' | 'header' | 'cookie';
  scheme?: string;
  bearerFormat?: string;
  flows?: OAuthFlows;
  openIdConnectUrl?: string;
}

interface OAuthFlows {
  implicit?: OAuthFlow;
  password?: OAuthFlow;
  clientCredentials?: OAuthFlow;
  authorizationCode?: OAuthFlow;
}

interface OAuthFlow {
  authorizationUrl?: string;
  tokenUrl?: string;
  refreshUrl?: string;
  scopes: { [scope: string]: string };
}

interface SecurityRequirement {
  [name: string]: string[];
}

interface Tag {
  name: string;
  description?: string;
  externalDocs?: ExternalDocumentation;
}

interface ExternalDocumentation {
  description?: string;
  url: string;
}
```

### Evento S3 (Entrada Lambda)

```typescript
interface S3Event {
  Records: S3EventRecord[];
}

interface S3EventRecord {
  eventVersion: string;
  eventSource: string;
  awsRegion: string;
  eventTime: string;
  eventName: string;
  s3: {
    s3SchemaVersion: string;
    configurationId: string;
    bucket: {
      name: string;
      arn: string;
    };
    object: {
      key: string;
      size: number;
      eTag: string;
      sequencer: string;
    };
  };
}
```

### Catálogo de Servicios (Salida)

```typescript
interface ServiceCatalog {
  services: ServiceEntry[];
  lastUpdated: string;
  totalServices: number;
}

interface ServiceEntry {
  serviceName: string;
  displayName: string;
  description: string;
  versions: VersionEntry[];
  latestVersion: string;
  tags: string[];
  lastUpdated: string;
  url: string;
}

interface VersionEntry {
  version: string;
  publishedAt: string;
  url: string;
}
```


## Stack Tecnológico

### Runtime y Lenguaje

- **Node.js 18+**: Runtime para Lambda
- **TypeScript**: Lenguaje principal para type safety y mejor experiencia de desarrollo

### AWS Services

- **AWS Lambda**: Procesamiento serverless de especificaciones OpenAPI
- **Amazon S3**: Almacenamiento de especificaciones (privado) y hosting del sitio estático (privado)
- **Amazon CloudFront**: CDN para distribución global del portal con HTTPS
- **AWS WAF**: Web Application Firewall para protección contra ataques
- **AWS Certificate Manager (ACM)**: Certificados SSL/TLS para dominio personalizado (opcional)
- **Amazon Route 53**: DNS para dominio personalizado (opcional)
- **AWS SDK v3**: Cliente para interactuar con servicios AWS

### Bibliotecas Core

#### Validación OpenAPI
- `@apidevtools/swagger-parser` - Parser y validador de OpenAPI con soporte para $ref resolution
- `openapi-types` - Tipos TypeScript para OpenAPI 3.x

#### Generación de Documentación
- `@redocly/openapi-core` - Biblioteca core de Redoc para bundling y procesamiento de specs
- Redoc CDN - Para renderizado en el navegador (incluido en HTML generado)

#### AWS SDK
- `@aws-sdk/client-s3` - Cliente S3 para upload/download
- `@aws-sdk/client-lambda` - Cliente Lambda para invocaciones

#### Utilidades
- `js-yaml` - Parsing de YAML para configuración

### Frontend (Sitio Estático)

#### CSS
- Redoc CSS (incluido vía CDN) - Estilos base de Redoc
- CSS personalizado con variables CSS para el catálogo
- Diseño responsive mobile-first

#### JavaScript
- Vanilla JavaScript para el catálogo (sin frameworks)
- Fetch API para cargar services.json
- History API para navegación (opcional)

#### Características del Cliente
- Búsqueda con filtrado simple (string matching)
- Responsive design con media queries
- Copy-to-clipboard (proporcionado por Redoc)
- Syntax highlighting (proporcionado por Redoc)

### Testing

- **Unit Tests**: Vitest o Jest
- **Integration Tests**: Vitest con AWS SDK mocking
- **Property-Based Tests**: fast-check
- **E2E Tests**: Playwright para validar el sitio generado

### Infraestructura como Código

- **Terraform HCL** para provisionar:
  - Buckets S3 privados con configuración de eventos
  - Función Lambda con permisos IAM y trigger S3
  - CloudFront distribution con OAI
  - AWS WAF Web ACL con reglas de protección
  - ACM certificate (opcional)
  - Route 53 records (opcional)

### CI/CD

- **GitLab CI** para:
  - Ejecutar tests
  - Build de la función Lambda
  - Deploy de infraestructura con Terraform
  - Publicar especificaciones OpenAPI a S3
  - Deploy inicial de index.html y assets


## Infraestructura con Terraform

La infraestructura se define usando Terraform HCL nativo para provisionar todos los recursos AWS necesarios.

### Estructura de Archivos Terraform

```
terraform/
├── main.tf              # Recursos principales
├── variables.tf         # Variables de entrada
├── outputs.tf           # Outputs de la infraestructura
├── providers.tf         # Configuración de providers
├── s3.tf               # Buckets S3
├── lambda.tf           # Función Lambda
├── cloudfront.tf       # CloudFront distribution
├── waf.tf              # AWS WAF Web ACL
├── iam.tf              # Roles y políticas IAM
├── acm.tf              # Certificado ACM (opcional)
├── route53.tf          # DNS Route 53 (opcional)
└── terraform.tfvars    # Valores de variables
```

### Ejemplo de Configuración Terraform

#### providers.tf

```hcl
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "developer-portal/terraform.tfstate"
    region = "us-east-1"
  }
}

provider "aws" {
  region = var.aws_region
}

provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"  # Para ACM certificate (CloudFront requiere us-east-1)
}
```

#### variables.tf

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "project_name" {
  description = "Project name"
  type        = string
  default     = "developer-portal"
}

variable "specs_bucket_name" {
  description = "S3 bucket name for OpenAPI specs"
  type        = string
}

variable "portal_bucket_name" {
  description = "S3 bucket name for portal static site"
  type        = string
}

variable "custom_domain" {
  description = "Custom domain for CloudFront (optional)"
  type        = string
  default     = ""
}
```

#### s3.tf (Buckets Privados)

```hcl
# Bucket para especificaciones OpenAPI (privado)
resource "aws_s3_bucket" "specs" {
  bucket = var.specs_bucket_name
}

resource "aws_s3_bucket_versioning" "specs" {
  bucket = aws_s3_bucket.specs.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_public_access_block" "specs" {
  bucket                  = aws_s3_bucket.specs.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Notificación S3 -> Lambda (directo, sin SQS)
resource "aws_s3_bucket_notification" "specs" {
  bucket = aws_s3_bucket.specs.id
  
  lambda_function {
    lambda_function_arn = aws_lambda_function.processor.arn
    events              = ["s3:ObjectCreated:*"]
  }
  
  depends_on = [aws_lambda_permission.allow_s3]
}

# Bucket para portal estático (privado)
resource "aws_s3_bucket" "portal" {
  bucket = var.portal_bucket_name
}

resource "aws_s3_bucket_public_access_block" "portal" {
  bucket                  = aws_s3_bucket.portal.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Política de bucket para CloudFront OAI
resource "aws_s3_bucket_policy" "portal" {
  bucket = aws_s3_bucket.portal.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowCloudFrontOAI"
        Effect = "Allow"
        Principal = {
          AWS = aws_cloudfront_origin_access_identity.portal.iam_arn
        }
        Action   = "s3:GetObject"
        Resource = "${aws_s3_bucket.portal.arn}/*"
      }
    ]
  })
}
```

#### lambda.tf (Trigger S3 Directo)

```hcl
# Función Lambda
resource "aws_lambda_function" "processor" {
  filename         = "lambda.zip"
  function_name    = "${var.project_name}-processor"
  role            = aws_iam_role.lambda.arn
  handler         = "index.handler"
  source_code_hash = filebase64sha256("lambda.zip")
  runtime         = "nodejs18.x"
  timeout         = 300
  memory_size     = 1024
  
  environment {
    variables = {
      SPECS_BUCKET  = aws_s3_bucket.specs.id
      PORTAL_BUCKET = aws_s3_bucket.portal.id
    }
  }
}

# Permiso para que S3 invoque Lambda directamente
resource "aws_lambda_permission" "allow_s3" {
  statement_id  = "AllowExecutionFromS3"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.processor.function_name
  principal     = "s3.amazonaws.com"
  source_arn    = aws_s3_bucket.specs.arn
}

# Rol IAM para Lambda
resource "aws_iam_role" "lambda" {
  name = "${var.project_name}-lambda-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "lambda.amazonaws.com"
      }
    }]
  })
}

# Política para acceso a S3
resource "aws_iam_role_policy" "lambda_s3" {
  name = "${var.project_name}-lambda-s3-policy"
  role = aws_iam_role.lambda.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["s3:GetObject", "s3:GetObjectVersion"]
        Resource = "${aws_s3_bucket.specs.arn}/*"
      },
      {
        Effect   = "Allow"
        Action   = ["s3:PutObject", "s3:GetObject"]
        Resource = "${aws_s3_bucket.portal.arn}/*"
      }
    ]
  })
}

# Política para CloudWatch Logs
resource "aws_iam_role_policy_attachment" "lambda_logs" {
  role       = aws_iam_role.lambda.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}
```

#### cloudfront.tf (Con OAI)

```hcl
# Origin Access Identity para acceso a S3 privado
resource "aws_cloudfront_origin_access_identity" "portal" {
  comment = "OAI for ${var.project_name}"
}

# CloudFront Distribution
resource "aws_cloudfront_distribution" "portal" {
  enabled             = true
  is_ipv6_enabled     = true
  default_root_object = "index.html"
  web_acl_id          = aws_wafv2_web_acl.portal.arn
  
  aliases = var.custom_domain != "" ? [var.custom_domain] : []
  
  origin {
    domain_name = aws_s3_bucket.portal.bucket_regional_domain_name
    origin_id   = "S3-${aws_s3_bucket.portal.id}"
    
    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.portal.cloudfront_access_identity_path
    }
  }
  
  default_cache_behavior {
    allowed_methods        = ["GET", "HEAD", "OPTIONS"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "S3-${aws_s3_bucket.portal.id}"
    viewer_protocol_policy = "redirect-to-https"
    compress               = true
    
    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }
    
    min_ttl     = 0
    default_ttl = 86400
    max_ttl     = 31536000
  }
  
  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }
  
  viewer_certificate {
    cloudfront_default_certificate = var.custom_domain == ""
    acm_certificate_arn           = var.custom_domain != "" ? aws_acm_certificate.portal[0].arn : null
    ssl_support_method            = var.custom_domain != "" ? "sni-only" : null
    minimum_protocol_version      = "TLSv1.2_2021"
  }
}
```

#### waf.tf (Reglas de Protección)

```hcl
# WAF Web ACL
resource "aws_wafv2_web_acl" "portal" {
  name  = "${var.project_name}-web-acl"
  scope = "CLOUDFRONT"
  
  default_action {
    allow {}
  }
  
  # Rate Limiting
  rule {
    name     = "RateLimitRule"
    priority = 1
    
    action {
      block {}
    }
    
    statement {
      rate_based_statement {
        limit              = 2000
        aggregate_key_type = "IP"
      }
    }
    
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name               = "RateLimitRule"
      sampled_requests_enabled  = true
    }
  }
  
  # AWS Managed Rules - Common Rule Set
  rule {
    name     = "AWSManagedRulesCommonRuleSet"
    priority = 2
    
    override_action {
      none {}
    }
    
    statement {
      managed_rule_group_statement {
        vendor_name = "AWS"
        name        = "AWSManagedRulesCommonRuleSet"
      }
    }
    
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name               = "AWSManagedRulesCommonRuleSet"
      sampled_requests_enabled  = true
    }
  }
  
  # AWS Managed Rules - Known Bad Inputs
  rule {
    name     = "AWSManagedRulesKnownBadInputsRuleSet"
    priority = 3
    
    override_action {
      none {}
    }
    
    statement {
      managed_rule_group_statement {
        vendor_name = "AWS"
        name        = "AWSManagedRulesKnownBadInputsRuleSet"
      }
    }
    
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name               = "AWSManagedRulesKnownBadInputsRuleSet"
      sampled_requests_enabled  = true
    }
  }
  
  # IP Reputation List
  rule {
    name     = "AWSManagedRulesAmazonIpReputationList"
    priority = 4
    
    override_action {
      none {}
    }
    
    statement {
      managed_rule_group_statement {
        vendor_name = "AWS"
        name        = "AWSManagedRulesAmazonIpReputationList"
      }
    }
    
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name               = "AWSManagedRulesAmazonIpReputationList"
      sampled_requests_enabled  = true
    }
  }
  
  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name               = "${var.project_name}-web-acl"
    sampled_requests_enabled  = true
  }
}
```

#### acm.tf (Certificado SSL/TLS - Opcional)

```hcl
# Certificado ACM para dominio personalizado
resource "aws_acm_certificate" "portal" {
  count = var.custom_domain != "" ? 1 : 0
  
  provider          = aws.us_east_1  # CloudFront requiere us-east-1
  domain_name       = var.custom_domain
  validation_method = "DNS"
  
  lifecycle {
    create_before_destroy = true
  }
}
```

#### route53.tf (DNS - Opcional)

```hcl
# Registro A para dominio personalizado
resource "aws_route53_record" "portal" {
  count = var.custom_domain != "" ? 1 : 0
  
  zone_id = var.route53_zone_id
  name    = var.custom_domain
  type    = "A"
  
  alias {
    name                   = aws_cloudfront_distribution.portal.domain_name
    zone_id                = aws_cloudfront_distribution.portal.hosted_zone_id
    evaluate_target_health = false
  }
}
```

### GitLab CI Pipeline para Terraform

```yaml
# .gitlab-ci.yml
stages:
  - validate
  - plan
  - apply

variables:
  TF_ROOT: terraform
  AWS_DEFAULT_REGION: us-east-1

.terraform_base:
  image: hashicorp/terraform:1.6
  before_script:
    - cd $TF_ROOT
    - terraform init

terraform:validate:
  extends: .terraform_base
  stage: validate
  script:
    - terraform validate
    - terraform fmt -check

terraform:plan:
  extends: .terraform_base
  stage: plan
  script:
    - terraform plan -out=tfplan
  artifacts:
    paths:
      - $TF_ROOT/tfplan
    expire_in: 1 week
  only:
    - merge_requests
    - main

terraform:apply:
  extends: .terraform_base
  stage: apply
  script:
    - terraform apply -auto-approve tfplan
  dependencies:
    - terraform:plan
  only:
    - main
  when: manual
  environment:
    name: production
```

### GitLab CI Pipeline para Publicar Especificaciones

```yaml
# .gitlab-ci.yml (en repositorio de microservicio)
stages:
  - build
  - publish

variables:
  AWS_DEFAULT_REGION: us-east-1
  SPECS_BUCKET: my-openapi-specs
  SERVICE_NAME: user-service
  SERVICE_VERSION: v1.0.0

publish:openapi:
  stage: publish
  image: amazon/aws-cli:latest
  script:
    - |
      aws s3 cp openapi.json \
        s3://${SPECS_BUCKET}/${SERVICE_NAME}/${SERVICE_VERSION}/openapi.json \
        --metadata "commit=${CI_COMMIT_SHA},pipeline=${CI_PIPELINE_ID},timestamp=$(date -u +%Y-%m-%dT%H:%M:%SZ)"
  only:
    - main
    - tags
  variables:
    AWS_ACCESS_KEY_ID: $AWS_ACCESS_KEY_ID
    AWS_SECRET_ACCESS_KEY: $AWS_SECRET_ACCESS_KEY
```


## Propiedades de Correctitud

*Una propiedad es una característica o comportamiento que debe ser verdadero en todas las ejecuciones válidas de un sistema - esencialmente, una declaración formal sobre lo que el sistema debe hacer. Las propiedades sirven como puente entre las especificaciones legibles por humanos y las garantías de correctitud verificables por máquinas.*

### Propiedad 1: Validación de formato OpenAPI

*Para cualquier* especificación OpenAPI en versión 3.0 o 3.1, el validador debe clasificarla correctamente como válida o inválida según el estándar OpenAPI, y para especificaciones inválidas debe retornar errores descriptivos.

**Valida: Requisitos 1.2, 1.3**

### Propiedad 2: Round-trip de publicación con metadatos completos

*Para cualquier* especificación OpenAPI válida publicada con metadatos (nombre de servicio, versión, ambiente, timestamp, commit hash), al recuperarla del almacenamiento debe retornar la especificación idéntica con todos los metadatos preservados.

**Valida: Requisitos 1.4, 3.2, 3.3**

### Propiedad 3: Soporte de versiones OpenAPI

*Para cualquier* especificación OpenAPI válida en versión 3.0.x o 3.1.x, el sistema debe aceptarla y procesarla exitosamente sin errores de versión.

**Valida: Requisitos 1.5**

### Propiedad 4: Preservación de versiones históricas

*Para cualquier* servicio con versiones existentes, cuando se publica una nueva versión, todas las versiones anteriores deben permanecer accesibles en el almacenamiento sin modificación.

**Valida: Requisitos 3.5**

### Propiedad 5: Soporte multi-ambiente

*Para cualquier* especificación OpenAPI válida, debe poder publicarse en cualquier ambiente configurado (development, staging, production) y recuperarse correctamente filtrada por ambiente.

**Valida: Requisitos 3.4**

### Propiedad 6: Completitud del registro de servicios

*Para cualquier* conjunto de especificaciones OpenAPI publicadas, el sitio estático generado debe incluir todos los servicios, endpoints y schemas en el HTML de salida, sin omisiones.

**Valida: Requisitos 2.1, 5.3**

### Propiedad 7: Información completa de servicios en el registro

*Para cualquier* servicio publicado, el HTML del registro de servicios debe contener el nombre del servicio, versión, descripción y timestamp de última actualización.

**Valida: Requisitos 2.2**

### Propiedad 8: Enlaces de navegación válidos

*Para cualquier* servicio en el registro, debe existir un enlace navegable a su página de documentación detallada, y ese archivo HTML debe existir en el output generado.

**Valida: Requisitos 2.3**

### Propiedad 9: Funcionalidad de búsqueda

*Para cualquier* término de búsqueda y conjunto de servicios, los resultados filtrados deben contener únicamente servicios cuyo nombre, descripción o tags coincidan con el término de búsqueda.

**Valida: Requisitos 2.4**

### Propiedad 10: Selector de versiones

*Para cualquier* servicio con múltiples versiones publicadas, el HTML generado debe incluir un mecanismo de selección de versiones y páginas separadas para cada versión.

**Valida: Requisitos 2.5**

### Propiedad 11: Completitud de información de endpoints

*Para cualquier* endpoint en una especificación OpenAPI, la página HTML generada debe contener el path completo, método HTTP, descripción, parámetros, request body (si existe), todos los códigos de respuesta definidos, y requisitos de autenticación (si están especificados).

**Valida: Requisitos 4.1, 4.2, 4.3, 4.5**

### Propiedad 12: Estructura expandible para schemas anidados

*Para cualquier* schema que contenga objetos anidados, el HTML generado debe incluir elementos con atributos de expansión/colapso (data attributes o clases CSS apropiadas).

**Valida: Requisitos 4.4**

### Propiedad 13: Generación de archivos estáticos

*Para cualquier* conjunto de especificaciones OpenAPI, el comando build debe generar únicamente archivos estáticos (HTML, CSS, JavaScript, imágenes) sin código que requiera procesamiento del lado del servidor.

**Valida: Requisitos 5.1, 5.2**

### Propiedad 14: Generación de ejemplos para requests y responses

*Para cualquier* endpoint con schema de request o responses definidos, el HTML generado debe incluir ejemplos de código para el request (si aplica) y para cada código de respuesta definido.

**Valida: Requisitos 6.1, 6.2**

### Propiedad 15: Priorización de ejemplos explícitos

*Para cualquier* endpoint que tenga ejemplos explícitos definidos en la especificación OpenAPI, los ejemplos mostrados en el HTML deben coincidir con los ejemplos de la especificación en lugar de ejemplos generados automáticamente.

**Valida: Requisitos 6.3**

### Propiedad 16: Formato JSON con syntax highlighting

*Para cualquier* ejemplo de código generado, debe estar formateado como JSON válido y el HTML debe contener elementos con clases CSS para syntax highlighting.

**Valida: Requisitos 6.4**

### Propiedad 17: Funcionalidad de copiar al portapapeles

*Para cualquier* ejemplo de código mostrado, el HTML debe incluir un botón o elemento interactivo con atributos apropiados para copiar el contenido al portapapeles.

**Valida: Requisitos 6.5**

### Propiedad 18: Sección dedicada de schemas

*Para cualquier* especificación OpenAPI que contenga definiciones de schemas en components.schemas, el HTML generado debe incluir una sección o página dedicada listando todos los schemas.

**Valida: Requisitos 7.1**

### Propiedad 19: Información completa de propiedades de schemas

*Para cualquier* schema con propiedades definidas, el HTML generado debe mostrar para cada propiedad: nombre, tipo, descripción (si existe), indicador visual de required, y todas las reglas de validación (minimum, maximum, pattern, enum, etc).

**Valida: Requisitos 7.2, 7.4, 7.5**

### Propiedad 20: Enlaces entre schemas relacionados

*Para cualquier* schema que contenga referencias ($ref) a otros schemas, el HTML generado debe incluir enlaces navegables (<a> tags con href válidos) hacia los schemas referenciados.

**Valida: Requisitos 7.3**

### Propiedad 21: Media queries para diseño responsive

*Para cualquier* sitio generado, el CSS debe contener media queries con breakpoint en 768px o similar para adaptar el layout en dispositivos móviles.

**Valida: Requisitos 8.2**

### Propiedad 22: Variables CSS consistentes

*Para cualquier* sitio generado, el CSS debe utilizar variables CSS (custom properties) para tipografía y espaciado, asegurando consistencia visual en toda la interfaz.

**Valida: Requisitos 8.5**


## Manejo de Errores

El sistema debe manejar errores de manera robusta en múltiples capas de la arquitectura serverless.

### Errores de Validación de OpenAPI

**Escenarios de Error:**
- Especificación con versión no soportada (< 3.0 o formato inválido)
- Campos requeridos faltantes (info, paths, etc)
- Referencias $ref rotas o circulares
- Tipos de datos inválidos en schemas
- Códigos de respuesta HTTP inválidos

**Manejo:**
```typescript
interface ValidationError {
  code: string;
  message: string;
  path: string; // JSON path al elemento con error
  severity: 'error' | 'warning';
}

// Ejemplo de respuesta de error
{
  success: false,
  errors: [
    {
      code: 'MISSING_REQUIRED_FIELD',
      message: 'El campo "info.title" es requerido',
      path: '$.info.title',
      severity: 'error'
    }
  ]
}
```

**Comportamiento:**
- Lambda registra el error en CloudWatch Logs
- Lambda reintenta automáticamente (hasta 2 veces por defecto)
- Después de reintentos fallidos, el evento va a Dead Letter Queue (DLQ)
- Se envía notificación (SNS/email) sobre especificaciones rechazadas
- No se modifica el estado del portal (no se genera HTML)
- El pipeline de CI/CD puede consultar el estado vía API o logs

### Errores de S3

**Escenarios de Error:**
- Especificación no encontrada en el bucket
- Permisos insuficientes para leer/escribir
- Bucket no existe
- Objeto demasiado grande (> límite de Lambda)
- Timeout al descargar/subir archivos

**Manejo:**
```typescript
class S3Error extends Error {
  constructor(
    message: string,
    public code: S3ErrorCode,
    public bucket: string,
    public key: string,
    public recoverable: boolean
  ) {
    super(message);
  }
}

enum S3ErrorCode {
  NOT_FOUND = 'NOT_FOUND',
  ACCESS_DENIED = 'ACCESS_DENIED',
  BUCKET_NOT_FOUND = 'BUCKET_NOT_FOUND',
  OBJECT_TOO_LARGE = 'OBJECT_TOO_LARGE',
  TIMEOUT = 'TIMEOUT',
}
```

**Comportamiento:**
- Para errores recuperables (timeout): reintentar con backoff exponencial
- Para errores de permisos: registrar error crítico y alertar al equipo de ops
- Para objetos muy grandes: considerar procesamiento por streaming o rechazar
- Implementar circuit breaker para evitar reintentos excesivos

### Errores de Procesamiento Lambda

**Escenarios de Error:**
- Timeout de Lambda (> 5 minutos)
- Memoria insuficiente (OOM)
- Error al generar HTML con Redoc
- Error al actualizar services.json (conflicto de escritura)
- Excepción no capturada en el código

**Manejo:**
```typescript
interface ProcessingError {
  phase: 'download' | 'validation' | 'generation' | 'upload' | 'catalog-update';
  serviceName: string;
  version: string;
  error: Error;
  timestamp: string;
}

async function handleLambdaError(error: ProcessingError): Promise<void> {
  // 1. Registrar error detallado en CloudWatch
  console.error('Processing error:', JSON.stringify(error));
  
  // 2. Publicar métrica de error a CloudWatch Metrics
  await publishErrorMetric(error.phase);
  
  // 3. Si es error crítico, enviar alerta
  if (isCriticalError(error)) {
    await sendAlert(error);
  }
  
  // 4. Lanzar excepción para que Lambda reintente
  throw error;
}
```

**Comportamiento:**
- Lambda retorna error para activar reintentos automáticos
- Después de reintentos fallidos, el evento va a DLQ
- CloudWatch Alarms monitorean tasa de errores y activan notificaciones
- Logs estructurados en JSON para facilitar debugging
- Métricas personalizadas para monitorear salud del sistema

### Errores de Procesamiento Lambda con Reintentos

**Escenarios de Error:**
- Evento S3 malformado
- Especificación duplicada (mismo spec publicado múltiples veces)
- Timeout al procesar mensaje

**Manejo:**
```typescript
export async function handler(event: S3Event, context: Context): Promise<void> {
  for (const record of event.Records) {
    try {
      // Validar formato del evento
      if (!record.s3 || !record.s3.object || !record.s3.object.key) {
        throw new Error('Invalid S3 event format');
      }
      
      const result = await processRecord(record);
      console.log('Processing successful:', result);
    } catch (error) {
      console.error('Error processing record:', error);
      // Re-throw para que Lambda reintente automáticamente
      throw error;
    }
  }
}
```

**Comportamiento:**
- Lambda reintenta automáticamente en caso de error (hasta 2 veces por defecto)
- Para especificaciones duplicadas: verificar si el HTML ya existe y es idéntico (skip)
- Implementar idempotencia: procesar el mismo evento múltiples veces debe ser seguro
- Configurar Dead Letter Queue (DLQ) en Lambda para eventos fallidos después de reintentos
- CloudWatch Alarms monitorean tasa de errores

**Configuración de Reintentos en Lambda:**
```hcl
resource "aws_lambda_function_event_invoke_config" "processor" {
  function_name = aws_lambda_function.processor.function_name
  
  maximum_retry_attempts = 2
  maximum_event_age_in_seconds = 3600
  
  destination_config {
    on_failure {
      destination = aws_sqs_queue.lambda_dlq.arn
    }
  }
}

resource "aws_sqs_queue" "lambda_dlq" {
  name = "${var.project_name}-lambda-dlq"
  message_retention_seconds = 1209600  # 14 días
}
```

### Errores de Generación con Redoc

**Escenarios de Error:**
- Especificación demasiado compleja para Redoc
- Error en bundling de $refs
- Memoria insuficiente durante generación
- HTML generado corrupto o vacío

**Manejo:**
```typescript
async function generateRedocHTML(
  spec: OpenAPISpec,
  serviceName: string,
  version: string
): Promise<string> {
  try {
    // Validar tamaño de spec
    const specSize = JSON.stringify(spec).length;
    if (specSize > MAX_SPEC_SIZE) {
      throw new Error(`Spec too large: ${specSize} bytes`);
    }
    
    // Generar HTML con timeout
    const html = await Promise.race([
      generateHTML(spec),
      timeout(30000, 'Redoc generation timeout')
    ]);
    
    // Validar HTML generado
    if (!html || html.length < 100) {
      throw new Error('Generated HTML is empty or too small');
    }
    
    return html;
  } catch (error) {
    console.error('Redoc generation failed:', error);
    throw new RedocGenerationError(serviceName, version, error);
  }
}
```

**Comportamiento:**
- Establecer límites de tamaño para especificaciones
- Timeout para generación de HTML (30 segundos)
- Validar HTML generado antes de subirlo a S3
- Fallback: generar página de error informativa si Redoc falla
- Registrar especificaciones problemáticas para análisis

### Errores de Actualización de Catálogo

**Escenarios de Error:**
- Conflicto de escritura (dos Lambdas actualizan services.json simultáneamente)
- services.json corrupto o con formato inválido
- Error al parsear JSON existente

**Manejo:**
```typescript
async function updateServicesCatalogSafe(
  newService: ServiceEntry
): Promise<void> {
  const maxRetries = 3;
  let attempt = 0;
  
  while (attempt < maxRetries) {
    try {
      // 1. Descargar con ETag para detección de conflictos
      const { data, etag } = await downloadWithETag('services.json');
      
      // 2. Parsear y actualizar
      const catalog = JSON.parse(data);
      const updated = mergeServiceEntry(catalog, newService);
      
      // 3. Subir con conditional write (if-match ETag)
      await uploadWithETag('services.json', JSON.stringify(updated), etag);
      
      return; // Éxito
    } catch (error) {
      if (error.code === 'PreconditionFailed') {
        // Conflicto detectado, reintentar
        attempt++;
        await sleep(Math.pow(2, attempt) * 100); // Backoff exponencial
      } else {
        throw error;
      }
    }
  }
  
  throw new Error('Failed to update catalog after retries');
}
```

**Comportamiento:**
- Usar ETags de S3 para detección de conflictos
- Reintentar con backoff exponencial en caso de conflicto
- Si services.json está corrupto, intentar reconstruir desde archivos HTML existentes
- Mantener backup de services.json antes de cada actualización

### Logging y Observabilidad

**Niveles de Log:**
```typescript
enum LogLevel {
  ERROR = 'error',   // Errores que impiden operación
  WARN = 'warn',     // Problemas que no impiden operación
  INFO = 'info',     // Información de progreso
  DEBUG = 'debug',   // Información detallada para debugging
}
```

**Configuración de CloudWatch:**
```typescript
// Logs estructurados en JSON
const logger = {
  error: (message: string, context: object) => {
    console.error(JSON.stringify({
      level: 'ERROR',
      message,
      ...context,
      timestamp: new Date().toISOString()
    }));
  },
  
  info: (message: string, context: object) => {
    console.log(JSON.stringify({
      level: 'INFO',
      message,
      ...context,
      timestamp: new Date().toISOString()
    }));
  }
};

// Uso
logger.info('Processing spec', {
  serviceName: 'user-service',
  version: 'v1.0.0',
  specSize: 12345
});
```

**Métricas de CloudWatch:**
- `SpecsProcessed` - Contador de especificaciones procesadas exitosamente
- `ProcessingErrors` - Contador de errores por fase
- `ProcessingDuration` - Duración del procesamiento por servicio
- `ValidationFailures` - Contador de validaciones fallidas
- `RedocGenerationTime` - Tiempo de generación de HTML con Redoc

**Alarmas:**
- Error rate > 10% en 5 minutos
- DLQ message count > 0
- Lambda duration > 4 minutos (cerca del timeout)
- Lambda memory usage > 90%


## Estrategia de Testing

### Enfoque Dual de Testing

El proyecto implementará dos tipos complementarios de pruebas:

1. **Pruebas Unitarias**: Verifican ejemplos específicos, casos edge y condiciones de error
2. **Pruebas Basadas en Propiedades**: Verifican propiedades universales a través de múltiples entradas generadas aleatoriamente

Ambos tipos son necesarios para cobertura completa: las pruebas unitarias detectan bugs concretos, mientras que las pruebas de propiedades verifican correctitud general.

### Balance de Pruebas Unitarias

- Las pruebas unitarias son útiles para ejemplos específicos y casos edge
- Evitar escribir demasiadas pruebas unitarias - las pruebas de propiedades manejan la cobertura de múltiples inputs
- Las pruebas unitarias deben enfocarse en:
  - Ejemplos específicos que demuestran comportamiento correcto
  - Puntos de integración entre componentes
  - Casos edge y condiciones de error
- Las pruebas de propiedades deben enfocarse en:
  - Propiedades universales que se mantienen para todos los inputs
  - Cobertura comprehensiva de inputs mediante randomización

### Configuración de Property-Based Testing

**Biblioteca Recomendada:** `fast-check` para JavaScript/TypeScript

**Configuración:**
```typescript
import fc from 'fast-check';

// Configuración global
const propertyTestConfig = {
  numRuns: 100, // Mínimo 100 iteraciones por propiedad
  verbose: true,
  seed: Date.now(), // Para reproducibilidad
};
```

**Formato de Tags:**
Cada prueba de propiedad debe referenciar su propiedad del documento de diseño:

```typescript
describe('Feature: developer-portal-api, Property 1: Validación de formato OpenAPI', () => {
  it('should correctly classify OpenAPI specs as valid or invalid', () => {
    fc.assert(
      fc.property(
        openAPISpecArbitrary(),
        (spec) => {
          const result = validator.validate(spec);
          // Verificar que la clasificación sea correcta
          expect(result.valid).toBe(isValidOpenAPISpec(spec));
        }
      ),
      propertyTestConfig
    );
  });
});
```

### Generadores de Datos (Arbitraries)

Para property-based testing, necesitamos generadores de datos aleatorios:

```typescript
// Generador de especificaciones OpenAPI válidas
function openAPISpecArbitrary(): fc.Arbitrary<OpenAPISpec> {
  return fc.record({
    openapi: fc.constantFrom('3.0.0', '3.0.1', '3.0.2', '3.0.3', '3.1.0'),
    info: fc.record({
      title: fc.string({ minLength: 1, maxLength: 100 }),
      version: fc.string({ minLength: 1, maxLength: 20 }),
      description: fc.option(fc.string({ maxLength: 500 })),
    }),
    paths: fc.dictionary(
      fc.string({ minLength: 1 }).map(s => `/${s}`),
      pathItemArbitrary(),
      { minKeys: 1, maxKeys: 20 }
    ),
    components: fc.option(componentsArbitrary()),
  });
}

// Generador de especificaciones inválidas
function invalidOpenAPISpecArbitrary(): fc.Arbitrary<any> {
  return fc.oneof(
    fc.record({ openapi: fc.constant('2.0') }), // Versión no soportada
    fc.record({ openapi: fc.constant('3.0.0') }), // Sin info
    fc.record({ openapi: fc.constant('3.0.0'), info: {} }), // Info sin title
    fc.anything(), // Datos completamente aleatorios
  );
}

// Generador de eventos S3
function s3EventArbitrary(): fc.Arbitrary<S3Event> {
  return fc.record({
    Records: fc.array(
      fc.record({
        eventName: fc.constant('ObjectCreated:Put'),
        s3: fc.record({
          bucket: fc.record({
            name: fc.string({ minLength: 3, maxLength: 63 })
          }),
          object: fc.record({
            key: fc.string({ minLength: 1 }).map(s => `${s}/v1.0.0/openapi.json`),
            size: fc.integer({ min: 100, max: 1000000 }),
            eTag: fc.hexaString({ minLength: 32, maxLength: 32 })
          })
        })
      }),
      { minLength: 1, maxLength: 10 }
    )
  });
}

// Generador de service entries
function serviceEntryArbitrary(): fc.Arbitrary<ServiceEntry> {
  return fc.record({
    serviceName: fc.string({ minLength: 1, maxLength: 50 }),
    displayName: fc.string({ minLength: 1, maxLength: 100 }),
    description: fc.string({ maxLength: 500 }),
    versions: fc.array(versionEntryArbitrary(), { minLength: 1, maxLength: 10 }),
    latestVersion: fc.string({ minLength: 1, maxLength: 20 }),
    tags: fc.array(fc.string({ minLength: 1, maxLength: 20 }), { maxLength: 5 }),
    lastUpdated: fc.date().map(d => d.toISOString()),
    url: fc.string({ minLength: 1 })
  });
}
```


### Pruebas Unitarias

**Estructura de Pruebas:**

```
tests/
├── unit/
│   ├── validator.test.ts
│   ├── redoc-generator.test.ts
│   ├── catalog-updater.test.ts
│   ├── s3-client.test.ts
│   └── lambda-handler.test.ts
├── integration/
│   ├── end-to-end-flow.test.ts
│   ├── s3-lambda.test.ts
│   └── catalog-update-concurrency.test.ts
├── property/
│   ├── validation.property.test.ts
│   ├── catalog.property.test.ts
│   ├── generation.property.test.ts
│   └── search.property.test.ts
└── e2e/
    ├── portal-navigation.test.ts
    └── redoc-rendering.test.ts
```

**Ejemplos de Pruebas Unitarias:**

```typescript
// Ejemplo específico de validación
describe('OpenAPI Validator', () => {
  it('should reject spec without info.title', async () => {
    const spec = {
      openapi: '3.0.0',
      info: { version: '1.0.0' },
      paths: {},
    };
    
    const result = await validator.validate(spec);
    
    expect(result.valid).toBe(false);
    expect(result.errors).toContainEqual(
      expect.objectContaining({
        code: 'MISSING_REQUIRED_FIELD',
        path: '$.info.title',
      })
    );
  });
  
  // Caso edge: referencias circulares
  it('should detect circular $ref in schemas', async () => {
    const spec = createSpecWithCircularRef();
    const result = await validator.validate(spec);
    
    expect(result.valid).toBe(false);
    expect(result.errors).toContainEqual(
      expect.objectContaining({
        code: 'CIRCULAR_REFERENCE',
      })
    );
  });
});

// Ejemplo de Lambda handler
describe('Lambda Handler', () => {
  it('should process S3 event and generate HTML', async () => {
    // Mock S3 client
    const mockS3 = {
      getObject: jest.fn().mockResolvedValue({
        Body: Buffer.from(JSON.stringify(validSpec))
      }),
      putObject: jest.fn().mockResolvedValue({})
    };
    
    const event = createS3Event('user-service/v1.0.0/openapi.json');
    
    await handler(event, {} as Context);
    
    // Verificar que se generó y subió el HTML
    expect(mockS3.putObject).toHaveBeenCalledWith(
      expect.objectContaining({
        Key: 'services/user-service-v1.0.0.html'
      })
    );
  });
  
  it('should move message to DLQ after 3 failed attempts', async () => {
    const mockS3 = {
      getObject: jest.fn().mockRejectedValue(new Error('Not found'))
    };
    
    const event = createS3Event('invalid/path/spec.json');
    
    // Simular 3 intentos
    for (let i = 0; i < 3; i++) {
      await expect(handler(event, {} as Context)).rejects.toThrow();
    }
    
    // Verificar que el evento fue a DLQ (esto se maneja por Lambda)
  });
});

// Ejemplo de generación con Redoc
describe('Redoc Generator', () => {
  it('should generate valid HTML with embedded spec', async () => {
    const spec = createValidSpec();
    const html = await generateRedocHTML(spec, 'test-service', 'v1.0.0');
    
    expect(html).toContain('<!DOCTYPE html>');
    expect(html).toContain('Redoc.init');
    expect(html).toContain(spec.info.title);
  });
  
  it('should apply custom Redoc options', async () => {
    const spec = createValidSpec();
    const options = {
      theme: { colors: { primary: { main: '#ff0000' } } }
    };
    
    const html = await generateRedocHTML(spec, 'test-service', 'v1.0.0', options);
    
    expect(html).toContain('#ff0000');
  });
});
```

### Pruebas de Integración

```typescript
describe('S3 -> Lambda Integration', () => {
  it('should process spec from S3 event to portal generation', async () => {
    // 1. Subir spec a S3 (esto activa Lambda automáticamente)
    await s3Client.putObject({
      Bucket: SPECS_BUCKET,
      Key: 'user-service/v1.0.0/openapi.json',
      Body: JSON.stringify(validSpec)
    });
    
    // 2. Esperar a que Lambda procese (trigger directo desde S3)
    await waitForProcessing(5000);
    
    // 3. Verificar que el HTML fue generado
    const html = await s3Client.getObject({
      Bucket: PORTAL_BUCKET,
      Key: 'services/user-service-v1.0.0.html'
    });
    
    expect(html.Body).toBeDefined();
    
    // 4. Verificar que services.json fue actualizado
    const catalog = await s3Client.getObject({
      Bucket: PORTAL_BUCKET,
      Key: 'services.json'
    });
    
    const services = JSON.parse(catalog.Body.toString());
    expect(services).toContainEqual(
      expect.objectContaining({
        serviceName: 'user-service',
        latestVersion: 'v1.0.0'
      })
    );
  });
});

describe('Concurrent Catalog Updates', () => {
  it('should handle multiple simultaneous updates without data loss', async () => {
    // Simular múltiples Lambdas actualizando el catálogo simultáneamente
    const updates = [
      { serviceName: 'service-1', version: 'v1.0.0' },
      { serviceName: 'service-2', version: 'v1.0.0' },
      { serviceName: 'service-3', version: 'v1.0.0' },
    ];
    
    await Promise.all(
      updates.map(update => updateServicesCatalog(update))
    );
    
    // Verificar que todos los servicios están en el catálogo
    const catalog = await loadCatalog();
    expect(catalog.length).toBe(3);
  });
});
```


### Pruebas End-to-End

**Herramienta:** Playwright para validar el sitio generado

```typescript
import { test, expect } from '@playwright/test';

test.describe('Generated Portal', () => {
  test.beforeAll(async () => {
    // Desplegar portal de prueba en S3
    await deployTestPortal();
  });
  
  test('should display service catalog on index page', async ({ page }) => {
    await page.goto('https://test-portal.s3.amazonaws.com/index.html');
    
    // Verificar que aparezcan servicios
    const services = await page.locator('.service-card').all();
    expect(services.length).toBeGreaterThan(0);
    
    // Verificar información de servicio
    const firstService = services[0];
    await expect(firstService.locator('.service-name')).toBeVisible();
    await expect(firstService.locator('.service-version')).toBeVisible();
  });
  
  test('should navigate to Redoc documentation page', async ({ page }) => {
    await page.goto('https://test-portal.s3.amazonaws.com/index.html');
    await page.click('.service-card:first-child .btn-primary');
    
    // Verificar que estamos en página Redoc
    await expect(page.locator('.redoc-wrap')).toBeVisible();
    await expect(page.locator('.api-info')).toBeVisible();
  });
  
  test('should filter services by search term', async ({ page }) => {
    await page.goto('https://test-portal.s3.amazonaws.com/index.html');
    
    // Esperar a que carguen los servicios
    await page.waitForSelector('.service-card');
    const initialCount = await page.locator('.service-card').count();
    
    // Buscar
    await page.fill('input[type="search"]', 'user');
    await page.waitForTimeout(300); // Debounce
    
    // Verificar que solo aparezcan servicios que coincidan
    const filteredCount = await page.locator('.service-card:visible').count();
    expect(filteredCount).toBeLessThanOrEqual(initialCount);
    
    const visibleServices = await page.locator('.service-card:visible').all();
    for (const service of visibleServices) {
      const text = await service.textContent();
      expect(text?.toLowerCase()).toContain('user');
    }
  });
  
  test('should display multiple versions of a service', async ({ page }) => {
    await page.goto('https://test-portal.s3.amazonaws.com/index.html');
    
    // Desmarcar "mostrar solo última versión"
    await page.uncheck('#filter-latest');
    
    // Verificar que aparezcan múltiples versiones
    const serviceCard = page.locator('.service-card').first();
    const versionLinks = await serviceCard.locator('.version-link').all();
    
    if (versionLinks.length > 1) {
      expect(versionLinks.length).toBeGreaterThan(1);
    }
  });
  
  test('should render Redoc with all endpoint information', async ({ page }) => {
    await page.goto('https://test-portal.s3.amazonaws.com/services/user-service-v1.0.0.html');
    
    // Verificar elementos de Redoc
    await expect(page.locator('.api-info')).toBeVisible();
    await expect(page.locator('.api-content')).toBeVisible();
    
    // Verificar que hay endpoints
    const operations = await page.locator('[data-section-id]').all();
    expect(operations.length).toBeGreaterThan(0);
    
    // Verificar que hay ejemplos de código
    await expect(page.locator('.redoc-json')).toBeVisible();
  });
  
  test('should allow copying code examples', async ({ page, context }) => {
    await context.grantPermissions(['clipboard-read', 'clipboard-write']);
    await page.goto('https://test-portal.s3.amazonaws.com/services/user-service-v1.0.0.html');
    
    // Buscar botón de copiar (proporcionado por Redoc)
    const copyButton = page.locator('button[title*="Copy"]').first();
    await copyButton.click();
    
    // Verificar que se copió al portapapeles
    const clipboardText = await page.evaluate(() => navigator.clipboard.readText());
    expect(clipboardText).toBeTruthy();
    expect(clipboardText).toContain('{'); // Debe ser JSON
  });
  
  test('should be responsive on mobile', async ({ page }) => {
    await page.setViewportSize({ width: 375, height: 667 }); // iPhone SE
    await page.goto('https://test-portal.s3.amazonaws.com/index.html');
    
    // Verificar que el layout se adapta
    const grid = page.locator('.services-grid');
    const gridStyle = await grid.evaluate(el => window.getComputedStyle(el).gridTemplateColumns);
    
    // En móvil debe ser una columna
    expect(gridStyle).toContain('1fr');
  });
});
```

### Cobertura de Código

**Objetivo:** Mínimo 80% de cobertura de código

**Herramienta:** c8 (cobertura para Node.js)

```json
{
  "scripts": {
    "test": "vitest run",
    "test:unit": "vitest run tests/unit",
    "test:integration": "vitest run tests/integration",
    "test:property": "vitest run tests/property",
    "test:coverage": "vitest run --coverage",
    "test:e2e": "playwright test"
  }
}
```

**Métricas a Monitorear:**
- Cobertura de líneas
- Cobertura de branches
- Cobertura de funciones
- Cobertura de statements

### Integración Continua

**Pipeline de CI:**

```yaml
# .gitlab-ci.yml
stages:
  - test
  - build
  - deploy

variables:
  NODE_VERSION: "18"
  AWS_DEFAULT_REGION: us-east-1

.node_base:
  image: node:${NODE_VERSION}
  before_script:
    - npm ci
  cache:
    paths:
      - node_modules/

test:unit:
  extends: .node_base
  stage: test
  script:
    - npm run test:unit
  coverage: '/All files[^|]*\|[^|]*\s+([\d\.]+)/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

test:property:
  extends: .node_base
  stage: test
  script:
    - npm run test:property

test:integration:
  extends: .node_base
  stage: test
  script:
    - npm run test:integration
  variables:
    AWS_ACCESS_KEY_ID: $AWS_ACCESS_KEY_ID
    AWS_SECRET_ACCESS_KEY: $AWS_SECRET_ACCESS_KEY

test:coverage:
  extends: .node_base
  stage: test
  script:
    - npm run test:coverage
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

build:lambda:
  extends: .node_base
  stage: build
  script:
    - npm run build
    - zip -r lambda.zip dist/ node_modules/
  artifacts:
    paths:
      - lambda.zip
    expire_in: 1 week
  only:
    - main
    - tags

deploy:test:
  image: amazon/aws-cli:latest
  stage: deploy
  script:
    - aws s3 sync test-portal/ s3://${TEST_PORTAL_BUCKET}/
  variables:
    AWS_ACCESS_KEY_ID: $AWS_ACCESS_KEY_ID
    AWS_SECRET_ACCESS_KEY: $AWS_SECRET_ACCESS_KEY
  only:
    - main
  environment:
    name: test

test:e2e:
  image: mcr.microsoft.com/playwright:v1.40.0-focal
  stage: deploy
  script:
    - npm ci
    - npx playwright install
    - npm run test:e2e
  dependencies:
    - deploy:test
  only:
    - main
```

### Estrategia de Testing por Componente

| Componente | Pruebas Unitarias | Pruebas de Propiedades | Pruebas E2E |
|------------|-------------------|------------------------|-------------|
| Validator | ✓ Casos específicos de error | ✓ Specs válidas/inválidas aleatorias | - |
| Lambda Handler | ✓ Procesamiento de eventos S3 | ✓ Manejo de errores | - |
| Redoc Generator | ✓ Generación de HTML | ✓ Specs aleatorias | ✓ Renderizado en navegador |
| Catalog Updater | ✓ Actualización de servicios | ✓ Concurrencia | - |
| S3 Client | ✓ Upload/download | - | - |
| Portal UI | ✓ Búsqueda y filtrado | ✓ Filtrado correcto | ✓ Navegación completa |

### Pruebas de Regresión

Mantener un conjunto de especificaciones OpenAPI reales como fixtures:

```
tests/fixtures/
├── real-world-specs/
│   ├── stripe-api.json
│   ├── github-api.json
│   ├── petstore.json
│   └── complex-nested-schemas.json
└── edge-cases/
    ├── circular-refs.json
    ├── deeply-nested.json
    ├── minimal-spec.json
    └── large-spec.json
```

Estas especificaciones deben procesarse exitosamente en cada ejecución de pruebas para prevenir regresiones.

### Pruebas de Carga

**Objetivo:** Validar que el sistema maneja volumen esperado

```typescript
describe('Load Tests', () => {
  it('should process 100 specs concurrently', async () => {
    const specs = Array.from({ length: 100 }, (_, i) => 
      createValidSpec(`service-${i}`, `v1.0.0`)
    );
    
    const startTime = Date.now();
    
    await Promise.all(
      specs.map(spec => publishSpec(spec))
    );
    
    const duration = Date.now() - startTime;
    
    // Debe completar en menos de 5 minutos
    expect(duration).toBeLessThan(5 * 60 * 1000);
  });
  
  it('should handle 1000 concurrent S3 events', async () => {
    // Simular 1000 eventos S3 (Lambda se invoca para cada uno)
    const events = Array.from({ length: 1000 }, (_, i) => 
      createS3Event(`service-${i}/v1.0.0/openapi.json`)
    );
    
    // Lambda debe procesar todos sin errores
    const results = await Promise.all(
      events.map(event => handler(event, {} as Context))
    );
    
    const successCount = results.filter(r => r.success).length;
    expect(successCount).toBe(1000);
  });
});
```

