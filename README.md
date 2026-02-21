# Developer Portal API

Portal de documentación centralizado que genera automáticamente sitios web estáticos a partir de especificaciones OpenAPI de múltiples microservicios mediante pipelines de CI/CD.

## Descripción

Developer Portal API es un sistema event-driven basado en arquitectura serverless que agrega y visualiza documentación OpenAPI de múltiples microservicios en un portal centralizado. El sistema procesa especificaciones OpenAPI publicadas por pipelines de CI/CD, las valida, genera páginas de documentación HTML usando Redoc, y mantiene un catálogo de servicios actualizado automáticamente.

## Características Principales

- **Publicación Automática**: Los pipelines de CI/CD publican especificaciones OpenAPI directamente a S3
- **Procesamiento Event-Driven**: Procesamiento asíncrono mediante S3 Event Notifications y SQS
- **Validación de OpenAPI**: Validación automática de especificaciones OpenAPI 3.0 y 3.1
- **Generación con Redoc**: Páginas de documentación profesionales generadas con @redocly/openapi-core
- **Catálogo de Servicios**: Índice dinámico con búsqueda y filtrado de servicios
- **Versionamiento**: Soporte para múltiples versiones de cada microservicio
- **Multi-Ambiente**: Publicación en diferentes ambientes (development, staging, production)
- **Sitio Estático**: Portal completamente estático desplegable en S3 o cualquier servidor web
- **Responsive**: Interfaz adaptable a dispositivos móviles, tablets y desktop

## Arquitectura

### Componentes Principales

```
Pipeline CI/CD → S3 Specs Bucket → S3 Event → SQS Queue → Lambda Processor → S3 Portal Bucket → Usuarios
```

1. **S3 Specs Bucket**: Almacena especificaciones OpenAPI publicadas por pipelines
2. **SQS Queue**: Cola de mensajes para procesamiento asíncrono de eventos S3
3. **Lambda Processor (Node.js)**: Función que valida specs, genera HTML con Redoc, y actualiza el catálogo
4. **S3 Portal Bucket**: Aloja el sitio web estático con páginas de documentación y catálogo
5. **CLI Tool (Opcional)**: Herramienta de línea de comandos para publicación manual

### Flujo de Datos

1. Pipeline de CI/CD publica `openapi.json` a S3 (`{service-name}/{version}/openapi.json`)
2. S3 emite evento de creación/actualización de objeto
3. Evento se envía a cola SQS
4. Lambda se activa automáticamente al recibir mensaje
5. Lambda descarga y valida la especificación OpenAPI
6. Lambda genera HTML standalone usando @redocly/openapi-core
7. Lambda sube HTML al portal bucket (`services/{service-name}-{version}.html`)
8. Lambda actualiza `services.json` con metadatos del servicio
9. Usuarios acceden al portal vía S3 Static Website Hosting o CloudFront

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
- **Node.js 18+**: Runtime para Lambda y CLI
- **TypeScript**: Lenguaje principal
- **AWS Lambda**: Procesamiento serverless
- **Amazon S3**: Almacenamiento y hosting estático
- **Amazon SQS**: Cola de mensajes
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
- **AWS CDK** o **Terraform**: Infraestructura como código

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
│   ├── cli/
│   │   ├── index.ts                # CLI principal
│   │   ├── commands/
│   │   │   ├── publish.ts          # Comando publish
│   │   │   ├── validate.ts         # Comando validate
│   │   │   └── list.ts             # Comando list
│   │   └── config.ts               # Configuración CLI
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
│   ├── cdk/                        # AWS CDK stacks
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
- Cuenta de AWS con permisos para S3, Lambda, SQS
- AWS CLI configurado

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

### Desplegar Infraestructura

#### Opción 1: AWS CDK

```bash
cd infrastructure/cdk

# Instalar dependencias
npm install

# Sintetizar CloudFormation template
cdk synth

# Desplegar stack
cdk deploy

# Destruir stack
cdk destroy
```

#### Opción 2: Terraform

```bash
cd infrastructure/terraform

# Inicializar Terraform
terraform init

# Planificar cambios
terraform plan

# Aplicar cambios
terraform apply

# Destruir infraestructura
terraform destroy
```

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

#### Opción 1: AWS CLI

```bash
# Publicar especificación a S3
aws s3 cp openapi.json s3://my-openapi-specs/user-service/v1.0.0/openapi.json

# El sistema procesará automáticamente la especificación
```

#### Opción 2: CLI Tool

```bash
# Instalar CLI globalmente
npm install -g openapi-portal-cli

# Publicar especificación
openapi-portal publish openapi.json \
  --service-name user-service \
  --version v1.0.0 \
  --environment production \
  --commit abc123 \
  --bucket my-openapi-specs

# Validar especificación sin publicar
openapi-portal validate openapi.json

# Listar especificaciones publicadas
openapi-portal list --bucket my-openapi-specs --service user-service
```

### Integración en Pipeline CI/CD

#### GitHub Actions

```yaml
name: Publish API Documentation

on:
  push:
    branches: [main]

jobs:
  publish-docs:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Publish OpenAPI spec
        run: |
          aws s3 cp openapi.json \
            s3://my-openapi-specs/${{ github.event.repository.name }}/${{ github.ref_name }}/openapi.json
```

#### GitLab CI

```yaml
publish-docs:
  stage: deploy
  image: amazon/aws-cli
  script:
    - aws s3 cp openapi.json s3://my-openapi-specs/${CI_PROJECT_NAME}/${CI_COMMIT_REF_NAME}/openapi.json
  only:
    - main
```

### Acceder al Portal

Una vez desplegado, el portal estará disponible en:

- **S3 Static Website**: `http://my-developer-portal.s3-website-us-east-1.amazonaws.com`
- **CloudFront** (si está configurado): `https://portal.example.com`

## Configuración

### Archivo de Configuración CLI

```yaml
# openapi-portal.config.yml
aws:
  region: us-east-1
  specsBucket: my-openapi-specs
  portalBucket: my-developer-portal

validation:
  strict: true
  allowedVersions: ["3.0", "3.1"]

environments:
  - development
  - staging
  - production

redoc:
  theme:
    colors:
      primary:
        main: "#32329f"
  hideDownloadButton: false
  disableSearch: false
  expandResponses: "200,201"
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
- DLQ message count > 0
- Lambda duration > 4 minutos
- Lambda memory usage > 90%

## Solución de Problemas

### La especificación no se procesa

1. Verificar que el archivo se subió correctamente a S3
2. Revisar logs de Lambda en CloudWatch
3. Verificar mensajes en Dead Letter Queue
4. Validar la especificación localmente: `openapi-portal validate openapi.json`

### Error de validación

```bash
# Validar especificación localmente
openapi-portal validate openapi.json

# Ver errores detallados en logs
aws logs tail /aws/lambda/openapi-portal-processor --follow
```

### El catálogo no se actualiza

1. Verificar que `services.json` existe en el portal bucket
2. Revisar permisos de escritura de Lambda en S3
3. Verificar logs de actualización de catálogo

### HTML generado está vacío

1. Verificar que la especificación OpenAPI es válida
2. Revisar logs de generación con Redoc
3. Verificar límites de memoria de Lambda

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
- ✅ Procesamiento event-driven con S3/SQS/Lambda
- ✅ Validación de OpenAPI 3.0 y 3.1
- ✅ Generación con Redoc
- ✅ Catálogo de servicios con búsqueda
- ✅ Soporte multi-versión

### Versión 1.1 (Próxima)
- 🔄 Autenticación para portal privado
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
