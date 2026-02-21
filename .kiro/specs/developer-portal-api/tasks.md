# Plan de Implementación: Developer Portal API

## Resumen

Este plan implementa un sistema serverless event-driven que genera automáticamente un Developer Portal centralizado a partir de especificaciones OpenAPI publicadas por pipelines de GitLab CI. El sistema utiliza AWS Lambda (Node.js/TypeScript), S3 privado con CloudFront + WAF, y Redoc para procesar y publicar documentación de forma incremental y segura.

## Tareas

- [ ] 1. Configurar estructura del proyecto y dependencias
  - Crear estructura de directorios para Lambda, frontend y tests
  - Configurar package.json con dependencias: @aws-sdk/client-s3, @redocly/openapi-core, @apidevtools/swagger-parser, typescript, vitest, fast-check
  - Configurar tsconfig.json para Node.js 18+ con strict mode
  - Configurar scripts de build y test en package.json
  - _Requisitos: 1.1, 1.2, 1.3, 1.4, 1.5_

- [ ] 2. Implementar validador de OpenAPI
  - [ ] 2.1 Crear interfaces TypeScript para ValidationResult, ValidationError y ValidationWarning
    - Definir tipos según el diseño (OpenAPIValidator, ValidationResult, etc.)
    - _Requisitos: 1.2, 1.3_
  
  - [ ]* 2.2 Escribir test de propiedad para validación de formato OpenAPI
    - **Propiedad 1: Validación de formato OpenAPI**
    - **Valida: Requisitos 1.2, 1.3**
  
  - [ ] 2.3 Implementar OpenAPIValidatorImpl usando @apidevtools/swagger-parser
    - Implementar método validate() que retorna ValidationResult
    - Implementar getSupportedVersions() para OpenAPI 3.0.x y 3.1.x
    - Parsear errores de validación en formato estructurado
    - _Requisitos: 1.2, 1.3, 1.5_
  
  - [ ]* 2.4 Escribir tests unitarios para casos específicos de validación
    - Test para spec sin info.title
    - Test para referencias circulares
    - Test para versiones no soportadas
    - _Requisitos: 1.2, 1.3_

- [ ] 3. Implementar integración con Redoc
  - [ ] 3.1 Crear interfaces para RedocIntegration y RedocOptions
    - Definir tipos según el diseño
    - _Requisitos: 4.1, 4.2, 4.3, 4.4, 4.5_
  
  - [ ] 3.2 Implementar función generateRedocHTML
    - Usar @redocly/openapi-core para bundling de specs
    - Generar HTML standalone con Redoc embebido
    - Aplicar opciones de tema desde variables de entorno
    - _Requisitos: 4.1, 4.2, 4.3, 6.1, 6.2, 6.4_
  
  - [ ] 3.3 Implementar función createHTML para template HTML
    - Generar HTML con Redoc CDN y spec embebido
    - Incluir configuración de opciones de Redoc
    - _Requisitos: 5.1, 5.2, 6.4, 6.5_
  
  - [ ]* 3.4 Escribir tests unitarios para generación de HTML
    - Test para HTML válido con spec embebido
    - Test para aplicación de opciones personalizadas
    - _Requisitos: 4.1, 6.1, 6.2_

- [ ] 4. Implementar cliente S3
  - [ ] 4.1 Crear funciones para download y upload de archivos S3
    - Implementar downloadSpec() usando GetObjectCommand
    - Implementar uploadToPortal() usando PutObjectCommand
    - Implementar downloadFromPortal() para leer archivos existentes
    - Manejar errores de S3 (NOT_FOUND, ACCESS_DENIED, TIMEOUT)
    - _Requisitos: 1.4, 3.3_
  
  - [ ] 4.2 Implementar funciones con ETag para actualizaciones concurrentes
    - Implementar downloadWithETag() para detección de conflictos
    - Implementar uploadWithETag() con conditional write
    - _Requisitos: 3.3_
  
  - [ ]* 4.3 Escribir tests unitarios para operaciones S3
    - Test para upload/download exitoso
    - Test para manejo de errores
    - _Requisitos: 1.4, 3.3_

- [ ] 5. Implementar generador de catálogo
  - [ ] 5.1 Crear interfaces para ServiceEntry, VersionEntry y SearchIndex
    - Definir tipos según el diseño
    - _Requisitos: 2.1, 2.2, 2.5_
  
  - [ ]* 5.2 Escribir test de propiedad para completitud del registro de servicios
    - **Propiedad 6: Completitud del registro de servicios**
    - **Valida: Requisitos 2.1, 5.3**
  
  - [ ] 5.3 Implementar función updateServicesCatalog
    - Descargar services.json existente desde S3
    - Agregar o actualizar entrada de servicio
    - Manejar múltiples versiones del mismo servicio
    - Subir services.json actualizado
    - _Requisitos: 2.1, 2.2, 2.5, 3.5_
  
  - [ ] 5.4 Implementar updateServicesCatalogSafe con manejo de concurrencia
    - Usar ETags para detección de conflictos
    - Implementar reintentos con backoff exponencial
    - _Requisitos: 2.1, 3.3_
  
  - [ ]* 5.5 Escribir tests unitarios para actualización de catálogo
    - Test para agregar nuevo servicio
    - Test para actualizar servicio existente
    - Test para múltiples versiones
    - _Requisitos: 2.1, 2.2, 2.5_

- [ ] 6. Checkpoint - Validar componentes core
  - Ejecutar todos los tests unitarios y de propiedades
  - Verificar que no hay errores de compilación TypeScript
  - Preguntar al usuario si hay dudas o ajustes necesarios

- [ ] 7. Implementar función Lambda handler
  - [ ] 7.1 Crear interfaces para LambdaHandler y ProcessingResult
    - Definir tipos según el diseño
    - _Requisitos: 1.1, 1.4, 3.1, 3.2_
  
  - [ ]* 7.2 Escribir test de propiedad para round-trip de publicación
    - **Propiedad 2: Round-trip de publicación con metadatos completos**
    - **Valida: Requisitos 1.4, 3.2, 3.3**
  
  - [ ] 7.3 Implementar función handler principal
    - Procesar eventos S3 directamente (sin SQS)
    - Iterar sobre cada record y llamar a processRecord
    - Recolectar resultados de procesamiento
    - Actualizar services.json con resultados exitosos
    - Implementar logging estructurado en JSON
    - _Requisitos: 1.1, 1.4, 3.1, 3.2_
  
  - [ ] 7.4 Implementar función processRecord
    - Parsear evento S3 directamente
    - Extraer serviceName y version del path S3
    - Descargar especificación desde S3
    - Validar especificación con OpenAPIValidator
    - Generar HTML con Redoc
    - Subir HTML al portal bucket
    - Retornar ProcessingResult
    - _Requisitos: 1.1, 1.2, 1.3, 1.4, 3.1, 3.2, 3.3_
  
  - [ ] 7.5 Implementar manejo de errores en Lambda
    - Manejar errores de validación (rechazar y loggear)
    - Manejar errores de S3 (reintentar si es recuperable)
    - Manejar timeouts y errores de Redoc
    - Lanzar excepciones para que Lambda reintente automáticamente
    - Configurar Dead Letter Queue para eventos fallidos
    - _Requisitos: 1.3_
  
  - [ ]* 7.6 Escribir tests unitarios para Lambda handler
    - Test para procesamiento exitoso de evento S3
    - Test para manejo de especificación inválida
    - Test para manejo de errores de S3
    - _Requisitos: 1.1, 1.2, 1.3, 1.4_

- [ ] 8. Implementar frontend del portal
  - [ ] 8.1 Crear index.html con estructura del catálogo
    - Implementar header con título y descripción
    - Crear barra de búsqueda con input type="search"
    - Crear filtros (checkbox para "mostrar solo última versión")
    - Crear contenedor para grid de servicios
    - Crear footer con timestamp de última actualización
    - Incluir enlaces a assets/search.js y assets/styles.css
    - _Requisitos: 2.1, 2.2, 2.3, 2.4, 5.1, 5.2_
  
  - [ ] 8.2 Implementar assets/search.js para búsqueda y filtrado
    - Implementar función initCatalog para cargar services.json
    - Implementar función handleSearch para filtrar por nombre, descripción y tags
    - Implementar función handleFilterChange para mostrar/ocultar versiones
    - Implementar función renderServices para generar tarjetas de servicios
    - Implementar función formatDate para formatear timestamps
    - _Requisitos: 2.1, 2.2, 2.3, 2.4, 2.5_
  
  - [ ]* 8.3 Escribir test de propiedad para funcionalidad de búsqueda
    - **Propiedad 9: Funcionalidad de búsqueda**
    - **Valida: Requisitos 2.4**
  
  - [ ] 8.4 Crear assets/styles.css con diseño responsive
    - Implementar estilos para header, search bar, service cards
    - Implementar CSS Grid para layout de servicios
    - Implementar media queries para mobile (breakpoint 768px)
    - Usar variables CSS para colores, tipografía y espaciado
    - Asegurar diseño mobile-first
    - _Requisitos: 8.1, 8.2, 8.3, 8.4, 8.5_
  
  - [ ]* 8.5 Escribir tests unitarios para funciones de búsqueda
    - Test para filtrado por nombre
    - Test para filtrado por tags
    - Test para mostrar/ocultar versiones
    - _Requisitos: 2.4, 2.5_

- [ ] 9. Checkpoint - Validar integración de componentes
  - Ejecutar todos los tests (unitarios, propiedades)
  - Verificar que el frontend carga correctamente
  - Preguntar al usuario si hay ajustes necesarios

- [ ] 10. Implementar infraestructura AWS con Terraform HCL
  - [ ] 10.1 Crear archivos Terraform para infraestructura
    - Crear providers.tf con configuración de AWS provider y backend S3
    - Crear variables.tf con variables de entrada (region, project_name, bucket names, custom_domain)
    - Crear s3.tf para buckets privados (specs y portal) con versionado y block public access
    - Crear lambda.tf para función Lambda con trigger S3 directo y Dead Letter Queue
    - Crear iam.tf para roles y políticas IAM con permisos mínimos
    - Crear cloudfront.tf para CloudFront distribution con Origin Access Identity (OAI)
    - Crear waf.tf para AWS WAF Web ACL con reglas de protección
    - Crear acm.tf para certificado ACM en us-east-1 (opcional)
    - Crear route53.tf para registro DNS A (opcional)
    - Crear outputs.tf con outputs de CloudFront URL, bucket names, Lambda ARN
    - _Requisitos: 1.1, 3.1, 3.3, 3.4_
  
  - [ ] 10.2 Configurar S3 Event Notification para trigger directo a Lambda
    - Configurar S3 Event Notification en specs bucket para eventos s3:ObjectCreated:*
    - Configurar Lambda permission para permitir invocación desde S3
    - Eliminar configuración de SQS (no se usa)
    - _Requisitos: 1.1, 3.1_
  
  - [ ] 10.3 Configurar buckets S3 privados con CloudFront OAI
    - Configurar block public access en ambos buckets
    - Crear Origin Access Identity para CloudFront
    - Configurar bucket policy en portal bucket para permitir acceso solo desde OAI
    - _Requisitos: 3.3, 3.4_
  
  - [ ] 10.4 Configurar CloudFront distribution
    - Configurar origin apuntando a portal bucket con OAI
    - Configurar default cache behavior con redirect-to-https
    - Configurar compresión y TTL para caché
    - Asociar WAF Web ACL a la distribución
    - Configurar certificado ACM y dominio personalizado (opcional)
    - _Requisitos: 3.3, 3.4_
  
  - [ ] 10.5 Configurar AWS WAF Web ACL
    - Crear Web ACL con scope CLOUDFRONT
    - Agregar regla de rate limiting (2000 requests por IP)
    - Agregar AWS Managed Rules: CommonRuleSet, KnownBadInputsRuleSet, AmazonIpReputationList
    - Configurar logging a S3 bucket
    - Configurar métricas de CloudWatch
    - _Requisitos: 3.4_
  
  - [ ] 10.6 Configurar variables de entorno para Lambda
    - SPECS_BUCKET: nombre del bucket de especificaciones
    - PORTAL_BUCKET: nombre del bucket del portal
    - REDOC_OPTIONS: opciones de tema en JSON
    - _Requisitos: 1.1, 4.1_
  
  - [ ] 10.7 Configurar CloudWatch Logs y Metrics
    - Crear log group para Lambda
    - Definir métricas personalizadas (SpecsProcessed, ProcessingErrors, etc.)
    - Crear alarmas para error rate y DLQ messages
    - Crear alarmas para WAF (HighBlockedRequests)
    - _Requisitos: 1.3_

- [ ] 11. Implementar tests de integración
  - [ ]* 11.1 Escribir test de integración S3 -> Lambda
    - Subir spec a S3, esperar procesamiento, verificar HTML generado
    - Verificar actualización de services.json
    - _Requisitos: 1.1, 1.4, 2.1, 3.1, 3.2, 3.3_
  
  - [ ]* 11.2 Escribir test de concurrencia para actualización de catálogo
    - Simular múltiples Lambdas actualizando simultáneamente
    - Verificar que no hay pérdida de datos
    - _Requisitos: 2.1, 3.3_
  
  - [ ]* 11.3 Escribir tests de carga
    - Test para procesar 100 specs concurrentemente
    - Test para manejar múltiples eventos S3 simultáneos
    - _Requisitos: 1.1, 3.1_

- [ ] 12. Implementar tests E2E con Playwright
  - [ ]* 12.1 Escribir test E2E para navegación del portal
    - Test para visualizar catálogo de servicios
    - Test para navegar a página de documentación Redoc
    - Test para búsqueda y filtrado de servicios
    - Test para selector de versiones
    - **Valida: Propiedades 7, 8, 9, 10**
    - _Requisitos: 2.1, 2.2, 2.3, 2.4, 2.5_
  
  - [ ]* 12.2 Escribir test E2E para renderizado de Redoc
    - Test para visualizar información completa de endpoints
    - Test para copiar ejemplos al portapapeles
    - Test para expandir/colapsar schemas anidados
    - **Valida: Propiedades 11, 12, 14, 17**
    - _Requisitos: 4.1, 4.2, 4.3, 4.4, 4.5, 6.1, 6.2, 6.4, 6.5_
  
  - [ ]* 12.3 Escribir test E2E para diseño responsive
    - Test para layout en viewport móvil (375px)
    - Test para adaptación de grid en diferentes tamaños
    - **Valida: Propiedad 21**
    - _Requisitos: 8.1, 8.2_

- [ ] 13. Configurar pipeline de GitLab CI
  - [ ] 13.1 Crear workflow de GitLab CI para el proyecto
    - Configurar stages: validate, plan, apply
    - Configurar job para ejecutar tests unitarios
    - Configurar job para ejecutar tests de propiedades
    - Configurar job para ejecutar tests de integración
    - Configurar job para generar reporte de cobertura
    - Configurar job para build de función Lambda
    - Configurar job para Terraform validate, plan y apply
    - Configurar secrets para credenciales AWS
    - _Requisitos: 1.1, 1.2, 1.3, 1.4_
  
  - [ ] 13.2 Crear pipeline de GitLab CI para microservicios
    - Crear ejemplo de .gitlab-ci.yml para publicar especificaciones
    - Configurar job para subir openapi.json a S3 specs bucket
    - Incluir metadatos (commit SHA, pipeline ID, timestamp)
    - Configurar para ejecutar en branches main y tags
    - _Requisitos: 1.1, 3.2_
  
  - [ ] 13.3 Configurar deploy automático del portal
    - Deploy de index.html y assets a S3 portal bucket
    - Deploy de función Lambda con código actualizado
    - Invalidar caché de CloudFront después de deploy
    - _Requisitos: 5.1, 5.2_

- [ ] 14. Checkpoint final - Validación completa del sistema
  - Ejecutar todos los tests (unitarios, propiedades, integración, E2E)
  - Verificar cobertura de código (objetivo: 80%+)
  - Desplegar infraestructura en ambiente de prueba con Terraform
  - Publicar especificación de prueba vía GitLab CI y verificar generación del portal
  - Verificar acceso al portal vía CloudFront con HTTPS
  - Verificar que WAF está bloqueando solicitudes maliciosas
  - Validar que todas las propiedades de correctitud se cumplen
  - Preguntar al usuario si el sistema cumple con las expectativas

## Notas

- Las tareas marcadas con `*` son opcionales y pueden omitirse para un MVP más rápido
- Cada tarea referencia requisitos específicos para trazabilidad
- Los checkpoints aseguran validación incremental
- Los tests de propiedades validan propiedades universales de correctitud
- Los tests unitarios validan ejemplos específicos y casos edge
- La implementación usa TypeScript para type safety
- El sistema es completamente serverless y event-driven
- Lambda se activa directamente desde eventos S3 (sin SQS)
- Buckets S3 son privados con acceso controlado mediante CloudFront OAI
- CloudFront + WAF proporcionan distribución segura con protección contra ataques
- Terraform HCL se usa para infraestructura como código
- GitLab CI se usa para pipelines de publicación y deployment
- Redoc maneja el renderizado de documentación, eliminando la necesidad de templates personalizados
