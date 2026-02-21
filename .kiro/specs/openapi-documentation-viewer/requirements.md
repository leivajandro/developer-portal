# Requirements Document

## Introduction

Este documento describe los requisitos para un Developer Portal que se genera automáticamente desde pipelines de CI/CD. El sistema agregará y visualizará la documentación OpenAPI de múltiples microservicios, proporcionando un portal centralizado para desarrolladores.

## Glossary

- **OpenAPI Specification**: Un formato estándar para describir APIs REST, anteriormente conocido como Swagger
- **Developer Portal**: El sitio web estático generado que muestra la documentación de todos los microservicios
- **Microservice**: Un servicio individual que expone una API documentada con OpenAPI
- **Pipeline**: El proceso de CI/CD que genera y publica la documentación
- **Service Registry**: El índice de todos los microservicios disponibles en el portal
- **Endpoint**: Una ruta específica de la API con su método HTTP asociado
- **Schema**: La estructura de datos definida en la especificación OpenAPI
- **Developer**: Cualquier persona que utiliza el Developer Portal para consultar documentación de APIs

## Requirements

### Requirement 1

**User Story:** Como ingeniero de DevOps, quiero que mi pipeline de CI/CD publique automáticamente la especificación OpenAPI de mi microservicio, para mantener la documentación actualizada sin intervención manual.

#### Acceptance Criteria

1. WHEN a Pipeline executes, THE Developer Portal SHALL accept OpenAPI specifications via a CLI tool or API endpoint
2. WHEN a Microservice publishes its OpenAPI specification, THE Developer Portal SHALL validate the specification format
3. IF an OpenAPI specification is invalid or malformed, THEN THE Developer Portal SHALL reject the publication and return validation errors to the Pipeline
4. WHEN a valid OpenAPI specification is published, THE Developer Portal SHALL store it with metadata including service name, version, and timestamp
5. THE Developer Portal SHALL support OpenAPI versions 3.0 and 3.1

### Requirement 2

**User Story:** Como desarrollador, quiero ver un catálogo de todos los microservicios disponibles, para descubrir y explorar las APIs de mi organización.

#### Acceptance Criteria

1. WHEN a Developer accesses the Developer Portal, THE Developer Portal SHALL display a Service Registry with all published Microservices
2. WHEN displaying Microservices, THE Developer Portal SHALL show the service name, version, description, and last update timestamp
3. WHEN a Developer clicks on a Microservice, THE Developer Portal SHALL navigate to the detailed documentation of that service
4. THE Developer Portal SHALL provide search functionality that filters Microservices by name, description, or tags
5. WHEN multiple versions of a Microservice exist, THE Developer Portal SHALL allow Developers to select and view different versions

### Requirement 3

**User Story:** Como ingeniero de DevOps, quiero que mi pipeline publique la especificación OpenAPI en un repositorio centralizado, para que esté disponible en el Developer Portal.

#### Acceptance Criteria

1. WHEN a Pipeline completes successfully, THE Developer Portal SHALL accept the OpenAPI specification through a publish endpoint or file storage
2. WHEN publishing a specification, THE Pipeline SHALL include metadata such as service name, version, environment, and git commit hash
3. WHEN a specification is published, THE Developer Portal SHALL store it in a structured format accessible for web rendering
4. THE Developer Portal SHALL support publishing specifications to different environments (development, staging, production)
5. WHEN a new version is published, THE Developer Portal SHALL maintain previous versions for historical reference

### Requirement 4

**User Story:** Como desarrollador, quiero ver los detalles completos de un endpoint específico de un microservicio, para entender cómo utilizarlo correctamente.

#### Acceptance Criteria

1. WHEN a Developer views an Endpoint detail, THE Developer Portal SHALL display the complete path, HTTP method, description, and parameters
2. WHEN an Endpoint has request body requirements, THE Developer Portal SHALL display the Schema with examples
3. WHEN an Endpoint has response definitions, THE Developer Portal SHALL display all possible response codes with their Schemas
4. WHEN a Schema contains nested objects, THE Developer Portal SHALL display the structure in an expandable/collapsible format
5. THE Developer Portal SHALL display authentication requirements for each Endpoint when specified

### Requirement 5

**User Story:** Como desarrollador, quiero generar un sitio web estático con toda la documentación, para poder desplegarlo fácilmente en cualquier servidor web.

#### Acceptance Criteria

1. THE Developer Portal SHALL provide a build command that generates a static website from all published OpenAPI specifications
2. WHEN the build process executes, THE Developer Portal SHALL create HTML, CSS, and JavaScript files that can be served without a backend
3. WHEN generating the static site, THE Developer Portal SHALL include all Microservices, Endpoints, and Schemas in the output
4. THE Developer Portal SHALL generate an index page with the Service Registry as the entry point
5. WHEN the static site is deployed, THE Developer Portal SHALL function without requiring server-side processing

### Requirement 6

**User Story:** Como desarrollador, quiero ver ejemplos de requests y responses, para poder probar las APIs rápidamente.

#### Acceptance Criteria

1. WHEN viewing an Endpoint, THE Developer Portal SHALL generate example request payloads based on the Schema
2. WHEN viewing an Endpoint, THE Developer Portal SHALL display example responses for each response code
3. WHEN examples are defined in the OpenAPI specification, THE Developer Portal SHALL prioritize showing those examples
4. THE Developer Portal SHALL display examples in JSON format with syntax highlighting
5. WHEN a Developer clicks on an example, THE Developer Portal SHALL allow copying the example to clipboard

### Requirement 7

**User Story:** Como desarrollador, quiero explorar los schemas y modelos de datos, para entender las estructuras de datos que utiliza cada API.

#### Acceptance Criteria

1. WHEN an OpenAPI specification contains schema definitions, THE Developer Portal SHALL display a dedicated schemas section
2. WHEN viewing a Schema, THE Developer Portal SHALL display all properties with their types, descriptions, and constraints
3. WHEN a Schema references another Schema, THE Developer Portal SHALL provide navigation links between related Schemas
4. WHEN a property is required, THE Developer Portal SHALL visually indicate the requirement
5. THE Developer Portal SHALL display validation rules such as minimum, maximum, pattern, and enum values

### Requirement 8

**User Story:** Como desarrollador, quiero que la interfaz sea responsive y fácil de usar, para poder consultar la documentación desde cualquier dispositivo.

#### Acceptance Criteria

1. THE Developer Portal SHALL render correctly on desktop, tablet, and mobile screen sizes
2. WHEN the viewport width is below 768 pixels, THE Developer Portal SHALL adapt the layout for mobile viewing
3. THE Developer Portal SHALL provide a clean and intuitive navigation structure
4. WHEN displaying long content, THE Developer Portal SHALL implement proper scrolling without breaking the layout
5. THE Developer Portal SHALL use consistent typography and spacing throughout the interface
