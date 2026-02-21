# Developer Portal API

Portal de documentación centralizado que genera automáticamente sitios web estáticos a partir de especificaciones OpenAPI de múltiples microservicios mediante pipelines de CI/CD.

## Descripción

Developer Portal API es un sistema event-driven basado en arquitectura serverless que agrega y visualiza documentación OpenAPI de múltiples microservicios en un portal centralizado. El sistema procesa especificaciones OpenAPI publicadas por pipelines de CI/CD, las valida, genera páginas de documentación HTML usando Redoc, y mantiene un catálogo de servicios actualizado automáticamente.

## Características Principales

- **Publicación Automática**: Los pipelines de CI/CD publican especificaciones OpenAPI directamente a S3
- **Procesamiento Event-Driven**: Procesamiento directo mediante S3 Event Notifications a Lambda
- **Validación de OpenAPI**: Validación automática de especificaciones OpenAPI 3.0 y 3.1
- **Generación con Redoc**: Páginas de documentación profesionales generadas con @redocly/openapi-core
- **Catálogo de Servicios**: Índice dinámico con búsqueda y filtrado de servicios
- **Versionamiento**: Soporte para múltiples versiones de cada microservicio
- **Multi-Ambiente**: Publicación en diferentes ambientes (development, staging, production)
- **Sitio Estático**: Portal completamente estático desplegable en S3 con CloudFront
- **Seguridad**: CloudFront + AWS WAF para protección contra amenazas web
- **Responsive**: Interfaz adaptable a dispositivos móviles, tablets y desktop

## Arquitectura

### Componentes Principales

```
Pipeline CI/CD → S3 Specs Bucket → S3 Event → Lambda Processor → S3 Portal Bucket (privado) → CloudFront + WAF → Usuarios
```

1. **S3 Specs Bucket (privado)**: Almacena especificaciones OpenAPI publicadas por pipelines
2. **Lambda Processor (Node.js)**: Función que valida specs, genera HTML con Redoc, y actualiza el catálogo
3. **S3 Portal Bucket (privado)**: Aloja el sitio web estático con páginas de documentación y catálogo
4. **CloudFront Distribution**: CDN que distribuye el contenido del portal con baja latencia
5. **AWS WAF**: Web Application Firewall que protege contra amenazas web comunes
6. **ACM Certificate**: Certificado SSL/TLS para HTTPS
7. **Route 53**: DNS para dominio personalizado (opcional)

### Flujo de Datos

1. Pipeline de CI/CD publica `openapi.json` a S3 (`{service-name}/{version}/openapi.json`)
2. S3 emite evento de creación/actualización de objeto directamente a Lambda
3. Lambda se activa automáticamente al recibir el evento S3
4. Lambda descarga y valida la especificación OpenAPI
5. Lambda genera HTML standalone usando @redocly/openapi-core
6. Lambda sube HTML al portal bucket (`services/{service-name}-{version}.html`)
7. Lambda actualiza `services.json` con metadatos del servicio
8. Usuarios acceden al portal vía CloudFront (HTTPS) protegido por WAF
9. CloudFront obtiene contenido de S3 usando Origin Access Identity (OAI)

### Seguridad

- **Buckets S3 Privados**: Ambos buckets (specs y portal) son privados, sin acceso público directo
- **Origin Access Identity (OAI)**: 
  - CloudFront accede a S3 mediante OAI, no mediante URLs públicas
  - El bucket policy de S3 solo permite acceso desde CloudFront
  - Los usuarios no pueden acceder directamente a S3, deben pasar por CloudFront
  - OAI se crea automáticamente con Terraform y se asocia a la distribución
- **AWS WAF**: Protección contra:
  - SQL injection
  - Cross-site scripting (XSS)
  - Rate limiting (prevención de DDoS)
  - Bloqueo de IPs maliciosas
  - Filtrado geográfico (opcional)
- **HTTPS Obligatorio**: Todo el tráfico usa TLS 1.2+
- **ACM Certificate**: Certificado SSL/TLS gestionado automáticamente por AWS
- **Security Headers**: CloudFront añade headers de seguridad (HSTS, X-Content-Type-Options, etc.)

### Estructura de Directorios S3

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
├── index.html                          # Página de catálogo
├── services.json                       # Metadatos de servicios
├── assets/
│   ├── search.js                       # Búsqueda del lado del cliente
│   └── styles.css                      # Estilos personalizados
└── services/
    ├── user-service-v1.0.0.html       # Documentación Redoc
    ├── user-service-v1.1.0.html
    └── order-service-v2.0.0.html
```

## Stack Tecnológico

### Backend
- **Node.js 18+**: Runtime para Lambda
- **TypeScript**: Lenguaje principal
- **AWS Lambda**: Procesamiento serverless
- **Amazon S3**: Almacenamiento de especificaciones y contenido estático
- **Amazon CloudFront**: CDN para distribución de contenido
- **AWS WAF**: Web Application Firewall
- **AWS Certificate Manager (ACM)**: Certificados SSL/TLS
- **Amazon Route 53**: DNS (opcional)
- **AWS SDK v3**: Cliente para servicios AWS

### Bibliotecas Core
- **@redocly/openapi-core**: Generación de documentación HTML
- **@apidevtools/swagger-parser**: Validación de especificaciones OpenAPI
- **openapi-types**: Tipos TypeScript para OpenAPI 3.x

### Frontend
- **Redoc**: Renderizado de documentación OpenAPI
- **Vanilla JavaScript**: Búsqueda y filtrado en el catálogo
- **CSS3**: Diseño responsive con variables CSS

### Testing
- **Vitest**: Framework de testing unitario
- **fast-check**: Property-based testing
- **Playwright**: Testing end-to-end

### Infraestructura
- **Terraform HCL**: Infraestructura como código

## Estructura del Proyecto

```
developer-portal-api/
├── src/
│   ├── lambda/
│   │   ├── handler.ts              # Handler principal de Lambda
│   │   ├── validator.ts            # Validador de OpenAPI
│   │   ├── redoc-generator.ts      # Generador de HTML con Redoc
│   │   ├── catalog-updater.ts      # Actualizador de catálogo
│   │   └── s3-client.ts            # Cliente S3
│   ├── portal/
│   │   ├── index.html              # Página de catálogo
│   │   ├── assets/
│   │   │   ├── search.js           # Búsqueda y filtrado
│   │   │   └── styles.css          # Estilos personalizados
│   │   └── templates/
│   │       └── redoc-template.ts   # Template HTML para Redoc
│   └── types/
│       ├── openapi.ts              # Tipos OpenAPI
│       ├── catalog.ts              # Tipos de catálogo
│       └── events.ts               # Tipos de eventos AWS
├── tests/
│   ├── unit/                       # Pruebas unitarias
│   ├── integration/                # Pruebas de integración
│   ├── property/                   # Property-based tests
│   ├── e2e/                        # Pruebas end-to-end
│   └── fixtures/                   # Especificaciones de prueba
├── infrastructure/
│   └── terraform/                  # Terraform modules
├── .kiro/
│   └── specs/
│       └── developer-portal-api/
│           ├── requirements.md     # Documento de requisitos
│           ├── design.md           # Documento de diseño
│           └── tasks.md            # Lista de tareas
├── package.json
├── tsconfig.json
└── README.md
```

## Documentación

La documentación completa del proyecto se encuentra en el directorio `.kiro/specs/developer-portal-api/`:

- **[requirements.md](.kiro/specs/developer-portal-api/requirements.md)**: Requisitos funcionales detallados con user stories y criterios de aceptación
- **[design.md](.kiro/specs/developer-portal-api/design.md)**: Diseño técnico completo incluyendo:
  - Arquitectura del sistema
  - Componentes e interfaces
  - Modelos de datos
  - Stack tecnológico
  - Propiedades de correctitud
  - Manejo de errores
  - Estrategia de testing
- **[tasks.md](.kiro/specs/developer-portal-api/tasks.md)**: Plan de implementación con tareas organizadas

## Configuración del Entorno de Desarrollo

### Prerrequisitos

- Node.js 18 o superior
- npm o yarn
- Cuenta de AWS con permisos para S3, Lambda, CloudFront, WAF, ACM
- AWS CLI configurado
- Terraform 1.0 o superior

### Instalación

```bash
# Clonar el repositorio
git clone <repository-url>
cd developer-portal-api

# Instalar dependencias
npm install

# Configurar variables de entorno
cp .env.example .env
# Editar .env con tus credenciales AWS y nombres de buckets
```

### Variables de Entorno

```bash
# AWS Configuration
AWS_REGION=us-east-1
SPECS_BUCKET=my-openapi-specs
PORTAL_BUCKET=my-developer-portal
CLOUDFRONT_DISTRIBUTION_ID=E1234567890ABC

# Redoc Configuration
REDOC_OPTIONS='{"theme":{"colors":{"primary":{"main":"#32329f"}}}}'

# Lambda Configuration
LAMBDA_TIMEOUT=300
LAMBDA_MEMORY=1024
```

### Desarrollo Local

```bash
# Compilar TypeScript
npm run build

# Ejecutar en modo desarrollo con watch
npm run dev

# Ejecutar linter
npm run lint

# Formatear código
npm run format
```

## Testing

### Ejecutar Pruebas

```bash
# Todas las pruebas
npm test

# Pruebas unitarias
npm run test:unit

# Property-based tests
npm run test:property

# Pruebas de integración
npm run test:integration

# Pruebas end-to-end
npm run test:e2e

# Cobertura de código
npm run test:coverage
```

### Estrategia de Testing

El proyecto implementa tres tipos de pruebas:

1. **Pruebas Unitarias**: Verifican ejemplos específicos y casos edge
2. **Property-Based Tests**: Verifican propiedades universales con datos aleatorios (mínimo 100 iteraciones)
3. **Pruebas E2E**: Validan el sitio generado en navegador real con Playwright

Objetivo de cobertura: **80% mínimo**

## Despliegue

### Desplegar Infraestructura con Terraform

```bash
cd infrastructure/terraform

# Inicializar Terraform
terraform init

# Crear archivo de variables
cat > terraform.tfvars <<EOF
aws_region = "us-east-1"
project_name = "developer-portal"
domain_name = "portal.example.com"  # Opcional
enable_waf = true
EOF

# Planificar cambios
terraform plan

# Aplicar cambios
terraform apply

# Ver outputs (CloudFront URL, etc.)
terraform output

# Destruir infraestructura
terraform destroy
```

### Configuración de WAF

El WAF se configura automáticamente con las siguientes reglas:

```hcl
# Reglas incluidas en el módulo Terraform:
- AWS Managed Rules - Core Rule Set (CRS)
- AWS Managed Rules - Known Bad Inputs
- Rate limiting: 2000 requests por 5 minutos por IP
- Bloqueo de SQL injection
- Bloqueo de XSS
- Filtrado geográfico (opcional)
```

Para personalizar las reglas de WAF, editar `infrastructure/terraform/waf.tf`.

### Desplegar Función Lambda

```bash
# Build de la función Lambda
npm run build:lambda

# Empaquetar función
npm run package:lambda

# Desplegar a AWS
npm run deploy:lambda
```

### Desplegar Portal Inicial

```bash
# Subir index.html y assets al portal bucket
npm run deploy:portal

# Esto sube:
# - index.html
# - assets/search.js
# - assets/styles.css
```

## Uso

### Publicar Especificación OpenAPI desde Pipeline

```bash
# Publicar especificación a S3 usando AWS CLI
aws s3 cp openapi.json s3://my-openapi-specs/user-service/v1.0.0/openapi.json

# El sistema procesará automáticamente la especificación
# Lambda se activará directamente por el evento S3
```

### Integración en Pipeline CI/CD

#### GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - publish-docs

variables:
  AWS_DEFAULT_REGION: us-east-1
  SPECS_BUCKET: my-openapi-specs

publish-api-docs:
  stage: publish-docs
  image: amazon/aws-cli:latest
  script:
    - |
      aws s3 cp openapi.json \
        s3://${SPECS_BUCKET}/${CI_PROJECT_NAME}/${CI_COMMIT_TAG:-${CI_COMMIT_REF_NAME}}/openapi.json \
        --metadata "commit=${CI_COMMIT_SHA},pipeline=${CI_PIPELINE_ID},branch=${CI_COMMIT_REF_NAME}"
  only:
    - main
    - tags
  environment:
    name: production
```

#### GitLab CI con Validación

```yaml
# .gitlab-ci.yml con validación previa
stages:
  - validate
  - publish-docs

validate-openapi:
  stage: validate
  image: node:18
  script:
    - npm install -g @apidevtools/swagger-cli
    - swagger-cli validate openapi.json
  only:
    - main
    - tags

publish-api-docs:
  stage: publish-docs
  image: amazon/aws-cli:latest
  needs:
    - validate-openapi
  script:
    - |
      VERSION=${CI_COMMIT_TAG:-v${CI_COMMIT_SHORT_SHA}}
      aws s3 cp openapi.json \
        s3://${SPECS_BUCKET}/${CI_PROJECT_NAME}/${VERSION}/openapi.json \
        --metadata "commit=${CI_COMMIT_SHA},version=${VERSION},environment=production"
    - echo "API documentation published for ${CI_PROJECT_NAME} ${VERSION}"
  only:
    - main
    - tags
  environment:
    name: production
```

### Acceder al Portal

Una vez desplegado, el portal estará disponible en:

- **CloudFront Distribution**: `https://d1234567890.cloudfront.net` (URL generada automáticamente)
- **Dominio Personalizado** (si está configurado): `https://portal.example.com`

El acceso directo a S3 no está disponible por seguridad. Todo el tráfico debe pasar por CloudFront + WAF.

## Configuración

### Variables de Terraform

```hcl
# infrastructure/terraform/terraform.tfvars
aws_region = "us-east-1"
project_name = "developer-portal"

# Dominio personalizado (opcional)
domain_name = "portal.example.com"
route53_zone_id = "Z1234567890ABC"

# Configuración de WAF
enable_waf = true
waf_rate_limit = 2000

# Configuración de Lambda
lambda_timeout = 300
lambda_memory = 1024

# Ambientes
environments = ["development", "staging", "production"]

# Redoc
redoc_theme_primary_color = "#32329f"
```

### Configuración de WAF

El Web Application Firewall (WAF) es obligatorio y protege el portal contra amenazas comunes:

#### Reglas Gestionadas por AWS

```hcl
# Incluidas automáticamente en el módulo Terraform:

1. AWSManagedRulesCommonRuleSet
   - Protección contra vulnerabilidades OWASP Top 10
   - SQL injection, XSS, path traversal, etc.

2. AWSManagedRulesKnownBadInputsRuleSet
   - Bloqueo de patrones de ataque conocidos
   - Protección contra exploits comunes

3. AWSManagedRulesAmazonIpReputationList
   - Bloqueo de IPs con mala reputación
   - Actualización automática de listas
```

#### Reglas Personalizadas

```hcl
# Rate Limiting
- Límite: 2000 requests por 5 minutos por IP
- Acción: Block
- Scope: CloudFront distribution

# Filtrado Geográfico (opcional)
- Países permitidos: configurable en terraform.tfvars
- Acción: Block para países no permitidos
```

#### Monitoreo de WAF

```bash
# Ver requests bloqueadas
aws wafv2 get-sampled-requests \
  --web-acl-arn <web-acl-arn> \
  --rule-metric-name <rule-name> \
  --scope CLOUDFRONT \
  --time-window StartTime=<timestamp>,EndTime=<timestamp>

# Ver métricas en CloudWatch
aws cloudwatch get-metric-statistics \
  --namespace AWS/WAFV2 \
  --metric-name BlockedRequests \
  --dimensions Name=Rule,Value=ALL \
  --start-time <timestamp> \
  --end-time <timestamp> \
  --period 300 \
  --statistics Sum
```

### Opciones de Redoc

Las opciones de Redoc se configuran mediante la variable de entorno `REDOC_OPTIONS`:

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

## Monitoreo y Observabilidad

### CloudWatch Logs

Los logs de Lambda están disponibles en CloudWatch Logs:

```bash
# Ver logs recientes
aws logs tail /aws/lambda/openapi-portal-processor --follow
```

### CloudWatch Metrics

Métricas personalizadas disponibles:

- `SpecsProcessed`: Especificaciones procesadas exitosamente
- `ProcessingErrors`: Errores por fase de procesamiento
- `ProcessingDuration`: Duración del procesamiento
- `ValidationFailures`: Validaciones fallidas
- `RedocGenerationTime`: Tiempo de generación HTML

### CloudWatch Alarms

Alarmas configuradas:

- Error rate > 10% en 5 minutos
- Lambda duration > 4 minutos
- Lambda memory usage > 90%
- CloudFront 5xx errors > 5% en 5 minutos

### WAF Metrics

Métricas de WAF disponibles en CloudWatch:

- `BlockedRequests`: Requests bloqueadas por WAF
- `AllowedRequests`: Requests permitidas
- `CountedRequests`: Requests contadas (modo count)
- Métricas por regla individual

## Solución de Problemas

### La especificación no se procesa

1. Verificar que el archivo se subió correctamente a S3
2. Revisar logs de Lambda en CloudWatch
3. Verificar que el evento S3 está configurado correctamente
4. Validar la especificación localmente con herramientas como swagger-cli

### Error de validación

```bash
# Validar especificación localmente
npx @apidevtools/swagger-cli validate openapi.json

# Ver errores detallados en logs
aws logs tail /aws/lambda/openapi-portal-processor --follow
```

### El catálogo no se actualiza

1. Verificar que `services.json` existe en el portal bucket
2. Revisar permisos de escritura de Lambda en S3
3. Verificar logs de actualización de catálogo
4. Invalidar caché de CloudFront si es necesario

### HTML generado está vacío

1. Verificar que la especificación OpenAPI es válida
2. Revisar logs de generación con Redoc
3. Verificar límites de memoria de Lambda

### CloudFront devuelve 403 Forbidden

1. Verificar que Origin Access Identity está configurado correctamente
2. Revisar bucket policy de S3
3. Verificar que el objeto existe en S3
4. Revisar reglas de WAF que puedan estar bloqueando

### Invalidar caché de CloudFront

```bash
# Invalidar todo el contenido
aws cloudfront create-invalidation \
  --distribution-id E1234567890ABC \
  --paths "/*"

# Invalidar archivos específicos
aws cloudfront create-invalidation \
  --distribution-id E1234567890ABC \
  --paths "/index.html" "/services.json"
```

## Contribución

### Guías de Contribución

1. Fork el repositorio
2. Crear una rama para tu feature: `git checkout -b feature/nueva-funcionalidad`
3. Hacer commit de tus cambios: `git commit -am 'Agregar nueva funcionalidad'`
4. Push a la rama: `git push origin feature/nueva-funcionalidad`
5. Crear un Pull Request

### Estándares de Código

- Seguir las guías de estilo de TypeScript
- Escribir pruebas para nuevas funcionalidades
- Mantener cobertura de código > 80%
- Documentar funciones públicas con JSDoc
- Ejecutar linter antes de commit: `npm run lint`

### Proceso de Review

- Todos los PRs requieren al menos una aprobación
- Las pruebas de CI deben pasar
- La cobertura de código no debe disminuir

## Licencia

[Especificar licencia aquí - MIT, Apache 2.0, etc.]

## Contacto y Soporte

- **Documentación**: Ver archivos en `.kiro/specs/developer-portal-api/`
- **Issues**: [Enlace al issue tracker]
- **Discusiones**: [Enlace a foro o canal de comunicación]

## Roadmap

### Versión 1.0 (Actual)
- ✅ Procesamiento event-driven con S3 → Lambda
- ✅ Validación de OpenAPI 3.0 y 3.1
- ✅ Generación con Redoc
- ✅ Catálogo de servicios con búsqueda
- ✅ Soporte multi-versión
- ✅ CloudFront + WAF obligatorio
- ✅ Infraestructura con Terraform HCL

### Versión 1.1 (Próxima)
- 🔄 Autenticación para portal privado (Cognito)
- 🔄 API REST para consultar catálogo
- 🔄 Webhooks para notificaciones
- 🔄 Métricas de uso de documentación

### Versión 2.0 (Futuro)
- 📋 Playground interactivo para probar APIs
- 📋 Generación de SDKs automática
- 📋 Comparación de versiones
- 📋 Changelog automático

---

**Desarrollado con ❤️ para simplificar la documentación de microservicios**
