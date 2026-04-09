# Tractus-X Semantic Hub — Deep Analysis Report

> **Generated:** 3 April 2026  
> **Repository:** `public-sldt-semantic-hub` (fork of `eclipse-tractusx/sldt-semantic-hub`)  
> **Current Version:** `DEV-SNAPSHOT` (latest tagged: `0.6.1`)

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture](#2-architecture)
3. [Technology Stack & Dependencies](#3-technology-stack--dependencies)
4. [Source Code Deep Dive](#4-source-code-deep-dive)
   - [Application Entry Point](#41-application-entry-point)
   - [Security Layer](#42-security-layer)
   - [Core Service Layer](#43-core-service-layer)
   - [Persistence Layer](#44-persistence-layer)
   - [SDK Integration Layer](#45-sdk-integration-layer)
   - [Domain Model](#46-domain-model)
   - [Error Handling](#47-error-handling)
   - [Health Monitoring](#48-health-monitoring)
   - [Configuration Properties](#49-configuration-properties)
5. [API Endpoints](#5-api-endpoints)
6. [Model Lifecycle & State Machine](#6-model-lifecycle--state-machine)
7. [SPARQL Query Engine](#7-sparql-query-engine)
8. [Code Generation (OpenAPI)](#8-code-generation-openapi)
9. [Testing Strategy](#9-testing-strategy)
10. [CI/CD Pipelines](#10-cicd-pipelines)
11. [Deployment Architecture](#11-deployment-architecture)
12. [Running Locally](#12-running-locally)
13. [Observations & Findings](#13-observations--findings)

---

## 1. Project Overview

The **Semantic Hub** is a core component of the [Eclipse Tractus-X](https://eclipse-tractusx.github.io/) ecosystem (part of the Catena-X automotive data space). It serves as a **centralized registry for Semantic Aspect Models** — formal RDF/Turtle definitions that describe data interfaces shared across the Catena-X network.

### What It Does

- **Stores** aspect models written in **SAMM** (Semantic Aspect Meta Model) or the legacy **BAMM** format as RDF triples in an Apache Jena Fuseki triple store
- **Validates** models against the ESMF (Eclipse Semantic Modeling Framework) specification
- **Generates artifacts** on-the-fly from stored models:
  - SVG diagrams
  - JSON Schemas
  - HTML documentation
  - OpenAPI specifications
  - Example JSON payloads
  - AAS (Asset Administration Shell) submodel templates
- **Manages lifecycle** of models through a state machine: DRAFT → RELEASED → STANDARDIZED → DEPRECATED
- **Enforces RBAC** via OAuth2/JWT with Keycloak integration

### Who Uses It

- **Data providers** publish aspect models describing the data they share
- **Data consumers** discover and retrieve models to understand data formats
- **Standardization bodies** manage model lifecycle from draft to standardized
- **Developer tools** generate code artifacts (schemas, API specs) from models

---

## 2. Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Clients / Portal / Tools                   │
└──────────────────────────┬──────────────────────────────────┘
                           │  REST API (JWT Bearer Token)
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                 Semantic Hub (Spring Boot 3.4.5)             │
│  ┌──────────────┐   ┌────────────────┐   ┌──────────────┐  │
│  │  Spring       │   │  ModelsApi     │   │  Swagger UI  │  │
│  │  Security 6   │──▶│  Controller    │   │  (SpringDoc) │  │
│  │  (OAuth2/JWT) │   │  (Generated)   │   └──────────────┘  │
│  └──────────────┘   └───────┬────────┘                      │
│                              │ delegates to                  │
│                   ┌──────────▼──────────┐                    │
│                   │  AspectModelService  │                    │
│                   │  (ModelsApiDelegate) │                    │
│                   └──────┬─────┬────────┘                    │
│                          │     │                             │
│              ┌───────────┘     └───────────┐                 │
│              ▼                             ▼                 │
│  ┌────────────────────────┐  ┌──────────────────────────┐   │
│  │  PersistenceLayer       │  │  SDKAccessHelper          │   │
│  │  (TripleStorePersistence│  │  (SDKAccessHelperSAMM)    │   │
│  │   + SdsSdk + SAMMSdk)   │  │                           │   │
│  └──────────┬─────────────┘  └──────────┬───────────────┘   │
│             │ SPARQL/HTTP                │ In-Process         │
│  Port 4242 (Jetty)                       │                   │
└─────────────┼────────────────────────────┼───────────────────┘
              ▼                            ▼
┌──────────────────────┐      ┌────────────────────────────┐
│  Apache Jena Fuseki  │      │  Eclipse ESMF SDK 2.9.8    │
│  5.0.0               │      │  (Validation + Artifact    │
│  (RDF Triple Store)  │      │   Generation)              │
│  Port 3030           │      └────────────────────────────┘
└──────────────────────┘
```

### Internal Dependency Graph

```
SemanticHubApplication (@SpringBootApplication)
  ├── @Bean → AuthorizationEvaluator(GeneralProperties.idm.publicClientId)
  ├── @Bean → WebMvcConfigurer (CORS + OpenID redirect)
  ├── @Bean → StrictHttpFirewall (allow encoded slashes for URN paths)
  ├── @Bean → SpringDoc manual bootstrapping
  └── @ComponentScan → OAuthSecurityConfig → uses AuthorizationEvaluator via SpEL

TripleStoreConfiguration (@Configuration)
  ├── @Bean → RDFConnectionRemoteBuilder (embedded or remote Fuseki)
  ├── @Bean (conditional) → FusekiServer (embedded, destroyMethod="stop")
  ├── @Bean → PersistenceLayer (TripleStorePersistence + SdsSdk)
  └── @Bean → AspectModelService(PersistenceLayer, SDKAccessHelper)

SDKAccessHelper (@Component)
  └── SDKAccessHelperSAMM (new'd, not Spring-managed)
        └── Uses ESMF SDK generators

TripleStorePersistence
  ├── RDFConnectionRemoteBuilder → Fuseki
  ├── SdsSdk → SAMMSdk
  │     ├── AspectModelValidator (ESMF)
  │     └── TripleStoreResolutionStrategy (custom dependency resolution)
  └── SparqlQueries (static SPARQL builders)

ApiExceptionHandler (@ControllerAdvice)
  └── Catches all domain exceptions → ErrorResponse envelope

TriplestoreLivenessProbe (@Component, HealthIndicator)
  └── PersistenceLayer.echo() → Health UP/DOWN
```

### Design Patterns Used

| Pattern | Where Used |
|---------|-----------|
| **Delegate** | `AspectModelService` implements generated `ModelsApiDelegate` — separates codegen from business logic |
| **Facade** | `SDKAccessHelper` wraps `SDKAccessHelperSAMM` |
| **Strategy** | `TripleStoreConfiguration.rdfConnectionBuilder` switches between embedded/remote connection strategies |
| **Strategy** | `TripleStoreResolutionStrategy` implements `ResolutionStrategy` for ESMF model dependency resolution |
| **Fail-Closed Security** | `AuthorizationEvaluator.containsRole()` returns `false` on any structural JWT deviation |
| **Error Envelope** | `ApiExceptionHandler` wraps all errors in uniform `ErrorResponse` |
| **Health Indicator** | `TriplestoreLivenessProbe` implements Spring's `HealthIndicator` |
| **Configuration Properties** | `GeneralProperties` and `TripleStoreProperties` use `@ConfigurationProperties` with validation |
| **Profile-Based Activation** | `OAuthSecurityConfig` is `@Profile("!local")` |
| **Contract-First API** | OpenAPI spec generates controller stubs; service implements the delegate |

---

## 3. Technology Stack & Dependencies

### Core Framework

| Technology | Version | Role |
|-----------|---------|------|
| **Java** | 17 | Language runtime |
| **Spring Boot** | 3.4.5 | Application framework (web, DI, actuator, OAuth2) |
| **Spring Security** | 6.3.8 | Authentication & authorization (OAuth2 JWT Resource Server) |
| **Spring Framework** | 6.2.11 | Core framework (managed by Spring Boot parent) |
| **Jetty** | (managed) | Embedded servlet container (replaces default Tomcat) |
| **Jakarta EE** | 6.0 (servlet), 3.0.2 (validation), 4.0.0 (XML bind) | Enterprise interfaces |

### Data & Storage

| Technology | Version | Role |
|-----------|---------|------|
| **Apache Jena** | 5.0.0 | RDF/SPARQL triple store client library |
| **Apache Jena Fuseki** | 5.0.0 | Triple store server (separate Docker container or embedded) |
| **TopBraid SHACL** | 1.3.1 | SHACL validation engine (used by Jena) |

### Semantic Modeling

| Technology | Version | Role |
|-----------|---------|------|
| **Eclipse ESMF SDK** (`esmf-aspect-model-starter`) | 2.9.8 | SAMM/BAMM model loading, validation, and artifact generation |

### Serialization

| Technology | Version | Role |
|-----------|---------|------|
| **Jackson Core** | 2.18.6 | JSON serialization/deserialization |
| **Jackson Databind** | 2.18.6 | JSON data binding |
| **Jackson Dataformat XML** | 2.18.6 | XML serialization (for AAS templates) |
| **Woodstox** | 6.4.0 | StAX XML processor |
| **SnakeYAML** | 2.0 | YAML processing |

### API Documentation

| Technology | Version | Role |
|-----------|---------|------|
| **SpringDoc OpenAPI** | 2.0.2 | Swagger UI and OpenAPI docs serving |
| **Swagger Annotations v3** | 2.0.0 | OpenAPI 3 annotations |
| **Swagger Annotations v2** | 1.5.20 | Legacy swagger annotations |

### Utilities

| Technology | Version | Role |
|-----------|---------|------|
| **Vavr** | 0.10.3 | Functional programming (`Try` monad for error handling) |
| **Guava** | 32.1.1-jre | Collection utilities (`Lists.transform`) |
| **Lombok** | 1.18.34 | Boilerplate reduction (`@Data`, `@Setter`, `@RequiredArgsConstructor`) |
| **MapStruct** | 1.5.3 | Object mapping (configured, used minimally) |
| **Commons IO** | 2.17.0 | I/O utilities |
| **Commons FileUpload** | 1.6.0 | Multipart file handling |
| **Apache HttpClient** | 4.5.12 | HTTP client for Fuseki basic auth |

### Observability

| Technology | Version | Role |
|-----------|---------|------|
| **Spring Boot Actuator** | 3.4.5 | Health checks, metrics, info endpoints |
| **Micrometer Prometheus SimpleClient** | (managed) | Bridge dependency for Jena Fuseki 5.0.0 compatibility with Micrometer 1.14.x |

### Testing

| Technology | Version | Role |
|-----------|---------|------|
| **JUnit Jupiter** | 5.9.3 | Unit/integration test framework |
| **Testcontainers** | 1.20.4 | Docker container management for integration tests |
| **Spring Boot Test** | 3.4.5 | `@SpringBootTest`, `MockMvc`, mock beans |
| **Spring Security Test** | 6.3.8 | JWT mock tokens for security tests |
| **AssertJ** | 3.24.2 | Fluent assertion library |

### Build & Code Generation

| Technology | Version | Role |
|-----------|---------|------|
| **Maven** | 3.x | Build tool |
| **openapi-generator-maven-plugin** | 6.2.1 | Generates Spring REST stubs from OpenAPI spec |
| **maven-compiler-plugin** | 3.8.1 | Java compilation with annotation processors |
| **git-commit-id-maven-plugin** | 5.0.0 | Generates `git.properties` with version info |
| **spring-boot-maven-plugin** | 3.4.5 | Boot JAR packaging |
| **maven-surefire-plugin** | 3.0.0-M5 | Test execution |

---

## 4. Source Code Deep Dive

### 4.1 Application Entry Point

**File:** `backend/src/main/java/org/eclipse/tractusx/semantics/SemanticHubApplication.java`

| Annotation | Purpose |
|-----------|---------|
| `@SpringBootApplication` | Enables auto-configuration, component scanning, configuration |
| `@ComponentScan(basePackages = {"org.eclipse.tractusx.semantics", "org.openapitools.configuration"})` | Adds the OpenAPI-generated configuration package |
| `@EnableConfigurationProperties(GeneralProperties.class)` | Binds `hub.general.*` properties |

**Beans defined:**

| Bean | Purpose |
|------|---------|
| `WebMvcConfigurer` | CORS (allow all origins/methods) + redirect `/.well-known/openid-configuration` to issuer's OIDC discovery endpoint |
| `SpringDocConfiguration` | Manual SpringDoc bootstrapping (needed due to custom component scan) |
| `SpringDocConfigProperties` | SpringDoc configuration properties |
| `ObjectMapperProvider` | ObjectMapper for SpringDoc |
| `StrictHttpFirewall` | Allows URL-encoded slashes (required for URN paths like `urn:samm:...`) |
| `AuthorizationEvaluator` | RBAC evaluator — created with IDM client ID from configuration |

**Key behavior:** The OIDC redirect is notable — it maps `/.well-known/openid-configuration` to `<issuer-uri>/.well-known/openid-configuration`, enabling Swagger UI's OAuth flow to discover the IdP endpoints.

---

### 4.2 Security Layer

#### OAuthSecurityConfig.java (Production — `@Profile("!local")`)

**File:** `backend/src/main/java/org/eclipse/tractusx/semantics/OAuthSecurityConfig.java`

Uses **Spring Security 6 lambda DSL** with `SecurityFilterChain` bean:

```
CSRF: disabled (stateless JWT API — no cookies/sessions)
Session: STATELESS (SessionCreationPolicy.STATELESS)
OAuth2: Resource Server with JWT validation
```

**Authorization Matrix:**

| HTTP Method | Path Pattern | Authorization Rule |
|-------------|-------------|-------------------|
| `OPTIONS` | `*` | `permitAll()` |
| `*` | `/`, `/swagger-ui/**`, `/swagger-resources/**`, `/v3/api-docs/**`, `/semantic-hub-openapi.yaml` | `permitAll()` |
| `*` | `/actuator/**` | `permitAll()` |
| `*` | `/error` | `permitAll()` |
| `GET` | `/**/models/**` | SpEL: `@authorizationEvaluator.hasRoleViewSemanticModel()` |
| `POST` | `/**/models/**` | SpEL: `@authorizationEvaluator.hasRoleAddSemanticModel()` |
| `PUT` | `/**/models/**` | SpEL: `@authorizationEvaluator.hasRoleUpdateSemanticModel()` |
| `DELETE` | `/**/models/**` | SpEL: `@authorizationEvaluator.hasRoleDeleteSemanticModel()` |

Authorization is implemented via `WebExpressionAuthorizationManager` with a `DefaultHttpSecurityExpressionHandler` that has the `ApplicationContext` set (required for SpEL bean references).

#### LocalOauthSecurityConfig.java (Local Dev — `@Profile("local")`)

Permits all requests. Disables CSRF. No OAuth2 resource server configured.

#### AuthorizationEvaluator.java

**File:** `backend/src/main/java/org/eclipse/tractusx/semantics/AuthorizationEvaluator.java`

Extracts roles from Keycloak JWT tokens. Expected JWT claim structure:

```json
{
  "resource_access": {
    "<clientId>": {
      "roles": ["view_semantic_model", "add_semantic_model", ...]
    }
  }
}
```

**4 Roles:**

| Method | Role Constant | Grants |
|--------|-------------|--------|
| `hasRoleViewSemanticModel()` | `view_semantic_model` | Read operations (GET) |
| `hasRoleAddSemanticModel()` | `add_semantic_model` | Create operations (POST) |
| `hasRoleUpdateSemanticModel()` | `update_semantic_model` | Update operations (PUT) |
| `hasRoleDeleteSemanticModel()` | `delete_semantic_model` | Delete operations (DELETE) |

**Fail-closed logic:** The `containsRole()` method uses defensive `instanceof` checks at every level of the nested JWT claim structure. Any structural deviation returns `false` (deny access).

**Configuration:** Client ID sourced from `hub.general.idm.public-client-id` (default: `catenax-portal`).

---

### 4.3 Core Service Layer

#### AspectModelService.java

**File:** `backend/src/main/java/org/eclipse/tractusx/semantics/hub/AspectModelService.java`

Implements `ModelsApiDelegate` (generated from OpenAPI). Acts as the orchestrator between `PersistenceLayer` (data access) and `SDKAccessHelper` (artifact generation).

**Methods (all override `ModelsApiDelegate`):**

| Method | Purpose | Key Logic |
|--------|---------|-----------|
| `getModelList(pageSize, page, namespaceFilter, status)` | Paginated listing | URL-decodes namespace filter, delegates to `persistenceLayer.getModels()` |
| `getModelByUrn(urn)` | Single model lookup | Delegates to `persistenceLayer.getModel()` |
| `createModelWithUrn(type, newModel, status)` | Create model | Delegates to `persistenceLayer.save()`, defaults status to `DRAFT` if null |
| `modifyModel(type, newModel, status)` | Update model definition | Delegates to `persistenceLayer.save()` |
| `updateModel(urn, status)` | Change model status | Delegates to `persistenceLayer.updateModel()` |
| `deleteModel(modelId)` | Delete model package | Extracts package URN, delegates to `persistenceLayer.deleteModelsPackage()` |
| `getModelFile(modelId)` | Get raw Turtle | Delegates to `persistenceLayer.getModelDefinition()` |
| `getModelDiagram(urn)` | Generate SVG | Uses `sdkHelper.generateSvg()` with `Try` error handling |
| `getModelJsonSchema(modelId)` | Generate JSON Schema | Uses `sdkHelper.getJsonSchema()` |
| `getModelDocu(modelId)` | Generate HTML docs | Uses `sdkHelper.getHtmlDocu()` with `Try` |
| `getModelOpenApi(modelId, baseUrl)` | Generate OpenAPI JSON | Uses `sdkHelper.getOpenApiDefinitionJson()` |
| `getModelExamplePayloadJson(modelId)` | Generate example JSON | Uses `sdkHelper.getExamplePayloadJson()` with `Try` |
| `getAasSubmodelTemplate(urn, aasFormat)` | Generate AAS template | Supports FILE (AASX binary), XML, JSON formats |
| `getModelListByUrns(pageSize, page, urns)` | Bulk URN lookup | Converts to `AspectModelUrn` list via Guava `Lists.transform` |

**Notable:** Uses Vavr's `Try<T>` for failable operations — unwraps with `.get()` and checks `.isFailure()` to throw `RuntimeException` on SDK errors.

---

### 4.4 Persistence Layer

#### PersistenceLayer.java (Interface)

**File:** `backend/src/main/java/org/eclipse/tractusx/semantics/hub/persistence/PersistenceLayer.java`

```java
public interface PersistenceLayer {
    SemanticModelList getModels(String namespaceFilter, ModelPackageStatus status, Integer page, Integer pageSize);
    SemanticModel getModel(AspectModelUrn urn);
    SemanticModel save(SemanticModelType type, String newModel, SemanticModelStatus status);
    String getModelDefinition(AspectModelUrn urn);
    void deleteModelsPackage(ModelPackageUrn urn);
    boolean echo();
    SemanticModelList findModelListByUrns(List<AspectModelUrn> urns, int page, int pageSize);
    SemanticModel updateModel(String urn, SemanticModelStatus status);
}
```

#### TripleStorePersistence.java (Implementation)

**File:** `backend/src/main/java/org/eclipse/tractusx/semantics/hub/persistence/triplestore/TripleStorePersistence.java`

The main persistence implementation. Communicates with Apache Jena Fuseki via `RDFConnectionRemoteBuilder`.

**Key method flows:**

**`save()` flow:**
1. Validate status parameter is non-null
2. Load RDF model from Turtle string via `sdsSdk.load()`
3. Extract aspect URN from the model
4. If package already exists → validate state transition
5. Validate model via `sdsSdk.validate()` (passes `findContainingModelByUrn` as the triple store requester function for dependency resolution)
6. Reload original RDF model, add status property, insert into triple store via `UpdateBuilder.addInsert`

**`updateModel()` flow:**
1. Validate status, find existing model by URN
2. Retrieve the full Jena `Model` graph (CONSTRUCT query)
3. Extract aspect URN
4. Validate state transition via `validateStatus()`
5. Delete old triples, insert updated model with new status

**`deleteModelsPackage()` flow:**
1. Find package by URN — throw `ModelPackageNotFoundException` if not found
2. Block deletion if status is `RELEASED` or `STANDARDIZED` → throw `IllegalArgumentException`
3. Delete all triples for the package URN prefix

**`echo()` — health check:**
- Executes `SparqlQueries.echoQuery()` (SPARQL `ASK {}`) against Fuseki
- Returns `true` if Fuseki responds, `false` otherwise

**RDF connection pattern:** All connections use try-with-resources (`try (RDFConnection conn = ...)`) for automatic cleanup.

**Result mapping:** `aspectModelFrom(QuerySolution)` extracts `SemanticModel` fields from SPARQL query results:
- `urn` → aspect URI string
- `version` → extracted from URN (namespace pattern `prefix:version#Name`)
- `name` → extracted from URN (fragment after `#`)
- `type` → determined by `contains("bamm")` heuristic
- `status` → from auxiliary `aux:status` property

---

### 4.5 SDK Integration Layer

#### SDKAccessHelper.java (Facade)

**File:** `backend/src/main/java/org/eclipse/tractusx/semantics/hub/SDKAccessHelper.java`

Thin `@Component` facade delegating to `SDKAccessHelperSAMM` (which is `new`'d directly, not Spring-managed).

#### SDKAccessHelperSAMM.java (Implementation)

**File:** `backend/src/main/java/org/eclipse/tractusx/semantics/hub/samm/SDKAccessHelperSAMM.java`

Uses `@Setter` and `@RequiredArgsConstructor` (Lombok). Each artifact generation method:
1. Retrieves model Turtle from `persistenceLayer.getModelDefinition()`
2. Loads as `AspectModel` via `AspectModelLoader`
3. Creates the appropriate ESMF SDK generator
4. Generates and returns the artifact

**Generator mapping:**

| Method | ESMF Generator | Output |
|--------|---------------|--------|
| `generateDiagram()` | `AspectModelDiagramGenerator` | SVG/PNG `byte[]` |
| `getJsonSchema()` | `AspectModelJsonSchemaGenerator` | `JsonNode` |
| `getHtmlDocument()` | `AspectModelDocumentationGenerator` | HTML `byte[]` (uses `/catena-template.css`) |
| `getOpenApiDefinitionJson()` | `AspectModelOpenApiGenerator` | JSON `String` |
| `getExamplePayloadJson()` | `AspectModelJsonPayloadGenerator` | JSON `String` |
| `getAasSubmodelTemplate()` | `AspectModelAasGenerator` | AAS `byte[]` (XML/JSON/AASX) |

#### SdsSdk.java (Model Loading)

**File:** `backend/src/main/java/org/eclipse/tractusx/semantics/hub/persistence/triplestore/SdsSdk.java`

Facade for model I/O:
- `load(byte[])` → parses Turtle bytes using `TurtleLoader.loadTurtle()` from ESMF SDK
- `load(String resourceName)` → loads from classpath resource
- `validate()` / `getAspectUrn()` → delegates to `SAMMSdk`

#### SAMMSdk.java (Validation)

**File:** `backend/src/main/java/org/eclipse/tractusx/semantics/hub/persistence/triplestore/SAMMSdk.java`

**Validation flow:**
1. Parse model string to `InputStream`
2. Create `TripleStoreResolutionStrategy` (custom `ResolutionStrategy` implementation)
3. Load model via `AspectModelLoader(resolutionStrategy).load(inputStream)`
4. Run `AspectModelValidator.validateModel(aspectModel)`
5. If violations exist → group by `errorCode`, join messages, throw `InvalidAspectModelException`

**`TripleStoreResolutionStrategy`** (inner class):
- Implements ESMF's `ResolutionStrategy` interface
- When the ESMF SDK needs to resolve an external model dependency during validation, this strategy queries the Fuseki triple store instead of the local filesystem
- Handles BAMM ↔ SAMM prefix translation for backward compatibility
- On resolution failure → throws `ResourceDefinitionNotFoundException`

**URN extraction (`getAspectUrn`):**
- Iterates all `rdf:type` statements in the RDF model
- Filters for URI resources matching `SAMM_ASPECT_URN_REGEX` (both SAMM and BAMM patterns)
- Maps subject to `AspectModelUrn` → returns first match or throws

---

### 4.6 Domain Model

| Class | Fields | Purpose |
|-------|--------|---------|
| `ModelPackageStatus` | Enum: `DRAFT`, `RELEASED`, `STANDARDIZED`, `DEPRECATED` | Model lifecycle states |
| `ModelPackageUrn` | `String urn` | Wraps a URN representing a model package (namespace prefix, no fragment) |
| `ModelPackage` | `ModelPackageStatus status` | Minimal domain object holding package status |

**URN Structure:**
```
urn:samm:org.eclipse.tractusx.example:1.0.0#AspectName
└──────────── package URN ──────────────────┘└─ fragment ┘
```

`ModelPackageUrn.fromUrn(AspectModelUrn)` strips the fragment to get the package-level URN.

---

### 4.7 Error Handling

**File:** `backend/src/main/java/org/eclipse/tractusx/semantics/ApiExceptionHandler.java`

`@ControllerAdvice` extending `ResponseEntityExceptionHandler`. All handlers return a consistent `ErrorResponse` → `Error` envelope:

```json
{
  "error": {
    "message": "...",
    "path": "...",
    "details": { ... }
  }
}
```

| Exception | HTTP Status | Notes |
|-----------|------------|-------|
| `MethodArgumentNotValidException` | 400 | Streams field errors into details map |
| `InvalidAspectModelException` | 400 | Includes validation violation details |
| `IllegalArgumentException` | 400 | General bad input |
| `MethodArgumentConversionNotSupportedException` | 400 | URL-decodes query string in message |
| `InvalidStateTransitionException` | 400 | Invalid lifecycle transition |
| `UrnSyntaxException` | 400 | Malformed URN |
| `HttpException` (Jena) | 400 | Triple store communication errors |
| `AspectModelNotFoundException` | 404 | Model URN not found |
| `ModelPackageNotFoundException` | 404 | Package URN not found |
| `EntityNotFoundException` | 404 | Generic not-found |

**Custom exception classes:**
- `InvalidAspectModelException` — carries a `Map<String, Object>` of validation details
- `InvalidStateTransitionException` — illegal state machine transition
- `AspectModelNotFoundException` — model URN not in triple store
- `ModelPackageNotFoundException` — package URN not found
- `EntityNotFoundException` — generic not-found
- `ResolutionException` — model dependency resolution failure
- `ResourceDefinitionNotFoundException` — external reference can't be resolved from triple store

---

### 4.8 Health Monitoring

**File:** `backend/src/main/java/org/eclipse/tractusx/semantics/TriplestoreLivenessProbe.java`

`@Component` implementing `HealthIndicator`:
- Calls `persistenceLayer.echo()` → executes SPARQL `ASK {}` against Fuseki
- Returns `Health.up()` if Fuseki responds, `Health.down()` on any failure
- Available at `/actuator/health` (permitted without authentication)

---

### 4.9 Configuration Properties

#### `hub.general.*` → `GeneralProperties.java`

| Property Path | Type | Validation | Default |
|--------------|------|-----------|---------|
| `hub.general.idm.public-client-id` | `String` | `@NotEmpty` | `catenax-portal` |

#### `hub.triple-store.*` → `TripleStoreProperties.java`

| Property Path | Type | Default |
|--------------|------|---------|
| `hub.triple-store.base-url` | `URL` | — |
| `hub.triple-store.query-endpoint` | `String` | — |
| `hub.triple-store.update-endpoint` | `String` | — |
| `hub.triple-store.username` | `String` | — |
| `hub.triple-store.password` | `String` | — |
| `hub.triple-store.embedded.enabled` | `boolean` | `false` |
| `hub.triple-store.embedded.port` | `int` | `3335` |
| `hub.triple-store.embedded.default-dataset` | `String` | `/data` |
| `hub.triple-store.embedded.context-path` | `String` | `/fuseki` |

#### `application.yml` (Main)

| Key | Value | Purpose |
|-----|-------|---------|
| `server.port` | `4242` | HTTP port |
| `server.forward-headers-strategy` | `framework` | Trust forwarded headers (for proxies) |
| `spring.application.name` | `semantics-services` | App name |
| `spring.mvc.pathmatch.matching-strategy` | `ant-path-matcher` | URL matching strategy |
| `spring.servlet.multipart.max-file-size` | `200MB` | Max upload size |
| `springdoc.swagger-ui.path` | `/` | Swagger UI at root |
| `springdoc.swagger-ui.urls[0]` | `/semantic-hub-openapi.yaml` | Static OpenAPI spec |

#### `application-local.yml` (Local Profile)

| Key | Value |
|-----|-------|
| `spring.security.oauth2.resourceserver.jwt.issuer-uri` | (empty) |
| `hub.triple-store.baseUrl` | `http://localhost:3030/ds` |
| `hub.triple-store.username` | (empty) |
| `hub.triple-store.password` | (empty) |
| `hub.triple-store.queryEndpoint` | `query` |
| `hub.triple-store.updateEndpoint` | `update` |

---

## 5. API Endpoints

All endpoints are served under `/api/v1` (defined in OpenAPI spec `servers` block).

### Model CRUD

| # | Method | Path | Operation ID | Auth Role | Description |
|---|--------|------|-------------|-----------|-------------|
| 1 | `GET` | `/models` | `getModelList` | `view_semantic_model` | List models with pagination. Query params: `namespaceFilter` (substring), `status` (enum), `pageSize` (10/50/100), `page` (0-indexed) |
| 2 | `POST` | `/models` | `createModelWithUrn` | `add_semantic_model` | Create a new model. Body: raw Turtle (text/plain). Query params: `type` (SAMM/BAMM, required), `status` (default DRAFT) |
| 3 | `PUT` | `/models` | `modifyModel` | `update_semantic_model` | Update model definition. Body: raw Turtle. Query params: `type`, `status` |
| 4 | `POST` | `/models/lookup` | `getModelListByUrns` | `view_semantic_model` | Bulk lookup. Body: JSON array of URN strings (max 10,000). Paginated response. |
| 5 | `GET` | `/models/{urn}` | `getModelByUrn` | `view_semantic_model` | Get model metadata by URN |
| 6 | `PUT` | `/models/{urn}` | `updateModel` | `update_semantic_model` | Change model status (state transition only) |
| 7 | `DELETE` | `/models/{urn}` | `deleteModel` | `delete_semantic_model` | Delete model package (only DRAFT or DEPRECATED) |

### Artifact Generation

| # | Method | Path | Operation ID | Auth Role | Output Content-Type |
|---|--------|------|-------------|-----------|-------------------|
| 8 | `GET` | `/models/{urn}/file` | `getModelFile` | `view_semantic_model` | `text/turtle` |
| 9 | `GET` | `/models/{urn}/diagram` | `getModelDiagram` | `view_semantic_model` | `image/svg` |
| 10 | `GET` | `/models/{urn}/documentation` | `getModelDocu` | `view_semantic_model` | `text/html` |
| 11 | `GET` | `/models/{urn}/json-schema` | `getModelJsonSchema` | `view_semantic_model` | `application/schema+json` |
| 12 | `GET` | `/models/{urn}/openapi` | `getModelOpenApi` | `view_semantic_model` | `application/json` (requires `baseUrl` query param) |
| 13 | `GET` | `/models/{urn}/example-payload` | `getModelExamplePayloadJson` | `view_semantic_model` | `application/json` |
| 14 | `GET` | `/models/{urn}/aas` | `getAasSubmodelTemplate` | `view_semantic_model` | `application/xml` (default), `application/json`, or `application/octet-stream` (AASX). Query param: `aasFormat` (FILE/XML/JSON) |

### Public Endpoints (No Auth Required)

| Path | Purpose |
|------|---------|
| `/` | Swagger UI (redirects) |
| `/swagger-ui/**` | Swagger UI static resources |
| `/v3/api-docs/**` | OpenAPI spec (generated by SpringDoc) |
| `/semantic-hub-openapi.yaml` | Static OpenAPI spec file |
| `/actuator/**` | Health checks, metrics, info |
| `/error` | Spring Boot error page |

### Response Models

| Model | Fields |
|-------|--------|
| `SemanticModel` | `urn`, `version`, `name`, `type` (SAMM/BAMM), `status` (DRAFT/RELEASED/STANDARDIZED/DEPRECATED) |
| `SemanticModelList` | `items[]`, `totalItems`, `currentPage`, `totalPages`, `itemCount` |
| `ErrorResponse` | `error` → `{ message, path, details }` |

---

## 6. Model Lifecycle & State Machine

```
                     ┌──────────────┐
                     │    DRAFT     │
                     └──────┬───────┘
                            │
              ┌─────────────┼─────────────┐
              │ (if no DRAFT │             │
              │  dependencies)│             │
              ▼              │             ▼
     ┌──────────────┐       │    ┌──────────────┐
     │   RELEASED   │       │    │  DEPRECATED  │
     └──────┬───────┘       │    │  (terminal)  │
            │               │    └──────────────┘
            │               │             ▲
     ┌──────▼───────┐       │             │
     │ STANDARDIZED │───────┘─────────────┘
     └──────────────┘
```

### Transition Rules (enforced in `TripleStorePersistence.validateStatus()`)

| From State | Allowed Transitions | Blocked | Notes |
|-----------|-------------------|---------|-------|
| `DRAFT` | → `RELEASED`, → `DEPRECATED` | → `STANDARDIZED` | Release blocked if any referenced package is in DRAFT |
| `RELEASED` | → `STANDARDIZED`, → `DEPRECATED` | → `DRAFT` | — |
| `STANDARDIZED` | → `DEPRECATED` | → `DRAFT`, → `RELEASED` | — |
| `DEPRECATED` | (none — terminal) | All transitions | Modifications also blocked |

### Deletion Rules

- `DRAFT` → deletable
- `DEPRECATED` → deletable
- `RELEASED` → **not** deletable (throws `IllegalArgumentException`)
- `STANDARDIZED` → **not** deletable (throws `IllegalArgumentException`)

### Status Storage

Status is stored as an RDF property using a custom auxiliary namespace:
```
Property: urn:bamm:io.openmanufacturing:aspect-model:aux#status
Value: "DRAFT" | "RELEASED" | "STANDARDIZED" | "DEPRECATED"
```
This is a custom extension, not part of the SAMM/BAMM specification.

---

## 7. SPARQL Query Engine

**File:** `backend/src/main/java/org/eclipse/tractusx/semantics/hub/persistence/triplestore/SparqlQueries.java`

All queries use `ParameterizedSparqlString` for injection-safe parameter binding.

### Query Catalog

| Query | Type | Purpose |
|-------|------|---------|
| `echoQuery()` | ASK | Liveness check: `ASK {}` |
| `buildFindByUrnQuery(urn)` | SELECT | Find single model by exact URN + package URN match |
| `buildFindAllQuery(namespace, status, page, pageSize)` | SELECT | Paginated listing with optional namespace/status filters |
| `buildCountAspectModelsQuery(namespace, status)` | SELECT | Count matching models (for pagination metadata) |
| `buildFindListByUrns(urns, page, pageSize)` | SELECT | Bulk lookup with VALUES clause, ORDER BY, OFFSET, LIMIT |
| `buildCountSelectiveAspectModelsQuery(...)` | SELECT | Count for bulk lookup |
| `buildFindByPackageQuery(modelsPackage)` | SELECT | Find status for a package URN |
| `buildDeleteByUrnRequest(modelsPackage)` | UPDATE | Delete all triples where subject starts with URN prefix |
| `buildFindByUrnConstructQuery(urn)` | CONSTRUCT | Full model closure (property path `(<>\|!<>)*`) |
| `buildFindModelElementClosureQuery(urn)` | CONSTRUCT | Element-level closure |

### Pattern: Dual SAMM/BAMM URN Matching

All queries use a regex filter that matches both URN schemes:
```sparql
FILTER regex(str(?type), "(urn:samm:...#Aspect)|(urn:bamm:...#Aspect)")
```

This ensures backward compatibility with models published under the older BAMM (OpenManufacturing) namespace.

### Status Filtering

The custom `aux:status` property is used in SPARQL queries:
```sparql
OPTIONAL { ?package <urn:bamm:io.openmanufacturing:aspect-model:aux#status> ?statusResult }
```

### Property Path Closure

The CONSTRUCT query uses SPARQL 1.1 property paths for transitive closure:
```sparql
CONSTRUCT { ?s ?p ?o } WHERE { <aspect-urn> (<>|!<>)* ?s . ?s ?p ?o }
```
This retrieves the entire sub-graph reachable from the aspect node.

---

## 8. Code Generation (OpenAPI)

### Source Spec

`backend/src/main/resources/static/semantic-hub-openapi.yaml` — the contract-first OpenAPI 3.0.3 specification.

### Plugin Configuration

The `openapi-generator-maven-plugin` (v6.2.1) generates Spring server stubs during build:

```xml
<configuration>
  <inputSpec>${project.basedir}/src/main/resources/static/semantic-hub-openapi.yaml</inputSpec>
  <generatorName>spring</generatorName>
  <configOptions>
    <delegatePattern>true</delegatePattern>
    <useSpringBoot3>true</useSpringBoot3>
    <apiPackage>org.eclipse.tractusx.semantics.hub.api</apiPackage>
    <modelPackage>org.eclipse.tractusx.semantics.hub.model</modelPackage>
  </configOptions>
</configuration>
```

### Generated Output (`target/generated-sources/openapi/`)

**API package (`api/`):**

| File | Role |
|------|------|
| `ModelsApi.java` | `@Controller` interface with Spring MVC annotations (`@GetMapping`, `@PostMapping`, etc.) |
| `ModelsApiController.java` | Default implementation delegating every call to `ModelsApiDelegate` |
| `ModelsApiDelegate.java` | Interface with `default` methods returning `NOT_IMPLEMENTED` |
| `ApiUtil.java` | Test utility for setting example responses |

**Model package (`model/`):**

| File | Role |
|------|------|
| `SemanticModel.java` | DTO: `urn`, `name`, `version`, `type`, `status` |
| `SemanticModelList.java` | Paginated list: `items[]`, `totalItems`, `totalPages`, `currentPage`, `itemCount` |
| `SemanticModelStatus.java` | Enum: `DRAFT`, `RELEASED`, `STANDARDIZED`, `DEPRECATED` |
| `SemanticModelType.java` | Enum: `SAMM`, `BAMM` |
| `AasFormat.java` | Enum: `FILE`, `XML`, `JSON` |
| `NewSemanticModel.java` | Creation request DTO |
| `Error.java` | Error detail: `message`, `details`, `path` |
| `ErrorResponse.java` | Wrapper containing an `Error` |

**Key:** The `delegatePattern: true` option cleanly separates generated controller code from hand-written business logic in `AspectModelService`.

---

## 9. Testing Strategy

### Test Infrastructure

| Class | Purpose |
|-------|---------|
| `FusekiTestContainer.java` | Base class — starts a Testcontainers-managed Fuseki (`jena-fuseki-docker:5.0.0`, port 3030) and injects dynamic properties |
| `TestOAuthSecurityConfig.java` | Test security config overriding production OAuth |
| `JwtTokenFactory.java` | Creates mock JWT tokens with configurable roles |
| `TestUtils.java` | Utilities (loading `.ttl` files from test resources) |
| `AbstractModelsApiTest.java` | Abstract base for integration tests — provides `MockMvc` helpers for POST/PUT/DELETE with JWT tokens |

### Test Classes

| Class | Scope | What's Tested |
|-------|-------|-------------|
| `ApplicationTest` | Smoke | Spring Boot context loads successfully |
| `SwaggerUITest` | Integration | Swagger UI endpoint availability |
| `HealthCheckTest` | Integration | Actuator health returns UP with Fuseki running |
| `ModelsApiTest` | Integration | Full CRUD lifecycle for SAMM and BAMM models; all artifact generation endpoints (diagram, JSON schema, documentation, OpenAPI, AASX, example payload); state transitions via inner classes `StateTransitionTestsForBAMM` and `StateTransitionTestsForSAMM` |
| `ModelsApiFilterTest` | Integration | Namespace filtering and status filtering on list endpoint |
| `ModelsApiPaginationTest` | Integration | Pagination (page, pageSize, totalItems, totalPages) |
| `ModelsApiSecurityTest` | Integration | RBAC enforcement — unauthorized access rejected, each role grants correct access |
| `SdsSdkTest` | Unit | Turtle loading and model validation at SDK level |
| `BammHelperTest` | Unit | BAMM-specific compatibility/helper functions |

### Test Data

Test Turtle files are located in `backend/src/test/resources/org/eclipse/tractusx/semantics/hub/` — various SAMM/BAMM aspect model definitions used by the integration tests.

### E2E Tests

**Framework:** Tavern (Python YAML-based HTTP API testing)  
**Location:** `e2e-tests/semantic-hub/`

| Test Suite | Tests |
|-----------|-------|
| Authentication | GET without token → 401; GET with token → 200 |
| CRUD BAMM | Create (DRAFT) → Read → Update → Delete |
| CRUD SAMM | Create (DRAFT) → Read → Update → Delete |
| Artifacts BAMM | Create → GET file/diagram/documentation/json-schema/openapi/example-payload/aas → Delete |
| Artifacts SAMM | Same as BAMM but for SAMM type |

E2E tests require: a running hub instance URL + valid bearer token (provided via `workflow_dispatch` inputs).

---

## 10. CI/CD Pipelines

### Build & Test

| Workflow | Trigger | What It Does |
|----------|---------|-------------|
| `build-snapshot.yml` | Push to main, PRs | JDK 17 setup → build Fuseki Docker image → `mvn clean install` |
| `codeql.yml` | Push to main, daily cron | CodeQL security analysis (`security-extended` + `security-and-quality` queries) |
| `helm-test.yml` | PRs | KinD cluster → chart-testing lint + install + upgrade |

### Security Scanning

| Workflow | Tool | Scope |
|----------|------|-------|
| `codeql.yml` | CodeQL | Java SAST |
| `trivy.yml` | Trivy | Repo config scan + Docker image scan (CRITICAL+HIGH) |
| `kics.yml` | Checkmarx KICS | Infrastructure-as-code scan (Helm, Dockerfile) |
| `gitleaks.yml` | Gitleaks | Secret detection in git history |
| `trufflehog.yml` | TruffleHog v3 | Secret detection (entropy filter=4, verified results) |

### Release

| Workflow | Trigger | What It Does |
|----------|---------|-------------|
| `release.yml` | Manual (`version` input) | Checkout `release` branch → merge main → build → set Maven version → git tag → push → trigger image publish |
| `publish-image-semantic-hub.yml` | Push to main / workflow_call | Build Docker image → push to DockerHub as `tractusx/sldt-semantic-hub` |
| `helm-release.yml` | Changes to `charts/**` | Helm chart-releaser → publishes to GitHub Pages |

### E2E Testing

| Workflow | Trigger | What It Does |
|----------|---------|-------------|
| `semantic-hub-e2e-test.yml` | Manual (requires `bearerToken`, `semantichubUrl`) | Python 3.8 + Tavern → runs E2E API tests against a live instance |

### Upstream Sync

| Workflow | Trigger | What It Does |
|----------|---------|-------------|
| `sync-upstream.yml` | Daily cron | Fast-forward merges `eclipse-tractusx/sldt-semantic-hub` main into `main-upstream` branch |

---

## 11. Deployment Architecture

### Docker Image

**File:** `backend/Dockerfile`

| Stage | Base Image | Purpose |
|-------|-----------|---------|
| Builder | `maven:3-eclipse-temurin-17-alpine` | `mvn package -DskipTests` |
| Runtime | `eclipse-temurin:17-jre-alpine` | Runs the JAR |

**Security hardening:**
- `apk upgrade --no-cache` for Alpine security patches
- Non-root user: `spring` (UID 100, GID 101)
- JVM: `-Xms512m -Xmx2048m`
- Exposes port **4242**
- Legal files at `/legal/`

### Helm Chart

**Location:** `charts/semantic-hub/`

**Components deployed:**

```
┌── Namespace: semantics ──────────────────────────────────────┐
│                                                              │
│  ┌─────────────────────┐      ┌─────────────────────────┐   │
│  │  hub Deployment      │      │  graphdb Deployment      │   │
│  │  (semantic-hub)      │──────│  (Fuseki 5.0.0)          │   │
│  │  Port 4242           │      │  Port 3030               │   │
│  │  Resources:          │      │  Resources:              │   │
│  │    CPU: 250m-750m    │      │    Memory: 512Mi-1Gi     │   │
│  │    Memory: 1Gi       │      │  PVC: 50Gi (TDB2)       │   │
│  └─────────┬────────────┘      └──────────────────────────┘   │
│            │                                                  │
│  ┌─────────▼────────────┐      ┌─────────────────────────┐   │
│  │  hub Service          │      │  graphdb Service          │   │
│  │  ClusterIP:8080       │      │  ClusterIP:3030           │   │
│  └─────────┬────────────┘      └──────────────────────────┘   │
│            │                                                  │
│  ┌─────────▼────────────┐      ┌─────────────────────────┐   │
│  │  hub Ingress          │      │  Keycloak (optional)     │   │
│  │  /semantics/hub       │      │  hub-keycloak            │   │
│  └──────────────────────┘      └──────────────────────────┘   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**Key Helm values:**

| Category | Key | Default | Description |
|----------|-----|---------|-------------|
| Hub | `hub.authentication` | `false` | Enable OAuth2 (when false, uses `local` profile) |
| Hub | `hub.embeddedTripleStore` | `false` | Use in-memory triple store (no Fuseki needed) |
| Hub | `hub.containerPort` | `4242` | Application port |
| Hub | `hub.service.port` | `8080` | Service port |
| Hub | `hub.ingress.urlPrefix` | `/semantics/hub` | Ingress path |
| GraphDB | `graphdb.enabled` | `false` | Deploy Fuseki alongside |
| GraphDB | `graphdb.image` | `jena-fuseki-docker:5.0.0` | Fuseki Docker image |
| GraphDB | `graphdb.storageSize` | `50Gi` | PVC size |
| Keycloak | `enableKeycloak` | `false` | Deploy Keycloak alongside |

**Pod security:**
- `runAsUser: 100` (Hub), `runAsUser: 1000` (GraphDB)
- `allowPrivilegeEscalation: false`
- `readOnlyRootFilesystem: true` (Hub)
- `emptyDir` at `/tmp` for temp file writes

**Health probes:**
- Liveness: `GET /actuator/health/liveness` (initial delay: 100s)
- Readiness: `GET /actuator/health/readiness` (initial delay: 100s)

---

## 12. Running Locally

### Prerequisites

1. **JDK 17** (newer versions like 25 are incompatible with `maven-compiler-plugin 3.8.1`)
2. **Docker Desktop** (for Fuseki container)
3. **Maven** (or use IntelliJ bundled Maven)

### Step-by-Step

```bash
# 1. Build the Fuseki Docker image (one-time)
# Download: https://repo1.maven.org/maven2/org/apache/jena/jena-fuseki-docker/5.0.0/jena-fuseki-docker-5.0.0.zip
# Unzip, cd into the directory, then:
docker build --platform linux/amd64 --build-arg JENA_VERSION=5.0.0 -t jena-fuseki-docker:5.0.0 .

# 2. Start Fuseki triple store
docker run -d --name fuseki -p 3030:3030 jena-fuseki-docker:5.0.0 --tdb2 --update --mem /ds

# 3. Build the project (skip tests for faster build)
mvn clean install -DskipTests

# 4. Run with local profile (no auth required)
mvn spring-boot:run -pl backend -Dspring-boot.run.profiles=local -DskipTests

# 5. Run tests (Fuseki auto-started by Testcontainers)
mvn test -pl backend
```

### Accessing the Application

| URL | Description |
|-----|-------------|
| `http://localhost:4242/` | Swagger UI (interactive API explorer) |
| `http://localhost:4242/actuator/health` | Health check (shows Fuseki connectivity) |
| `http://localhost:4242/api/v1/models?page=0&pageSize=10` | List all models |

### Quick API Test

```bash
# Create a SAMM model
curl -X POST "http://localhost:4242/api/v1/models?type=SAMM&status=DRAFT" \
  -H "Content-Type: text/plain" \
  -d '@prefix samm: <urn:samm:org.eclipse.esmf.samm:meta-model:2.1.0#> .
@prefix : <urn:samm:org.example:1.0.0#> .
@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .

:TestAspect a samm:Aspect ;
    samm:preferredName "Test"@en ;
    samm:properties () ;
    samm:operations () .'

# List models
curl "http://localhost:4242/api/v1/models?page=0&pageSize=10"

# Get model artifacts (replace URN with actual)
curl "http://localhost:4242/api/v1/models/urn%3Asamm%3Aorg.example%3A1.0.0%23TestAspect/json-schema"
curl "http://localhost:4242/api/v1/models/urn%3Asamm%3Aorg.example%3A1.0.0%23TestAspect/documentation"
curl "http://localhost:4242/api/v1/models/urn%3Asamm%3Aorg.example%3A1.0.0%23TestAspect/file"
```

### Docker Compose (Manual)

```bash
# Hub + Fuseki
docker run -d --name fuseki -p 3030:3030 jena-fuseki-docker:5.0.0 --tdb2 --update --mem /ds
docker build -t semantic-hub -f backend/Dockerfile .
docker run -d --name hub -p 4242:4242 \
  -e HUB_TRIPLE_STORE_BASE_URL=http://host.docker.internal:3030/ds \
  -e HUB_TRIPLE_STORE_QUERY_ENDPOINT=query \
  -e HUB_TRIPLE_STORE_UPDATE_ENDPOINT=update \
  -e SPRING_PROFILES_ACTIVE=local \
  semantic-hub
```

### Helm (Kubernetes)

```bash
kubectl create namespace semantics
helm install hub -n semantics ./charts/semantic-hub \
  --set graphdb.enabled=true \
  --set hub.authentication=false
```

---

## 13. Observations & Findings

### Architectural Strengths

1. **Contract-First API Design**: The OpenAPI spec is the single source of truth. Code generation with `delegatePattern: true` cleanly separates generated controller stubs from business logic.

2. **Comprehensive Security Scanning**: Five separate security tools (CodeQL, Trivy, KICS, Gitleaks, TruffleHog) in CI/CD provide defense-in-depth for SAST, container scanning, IaC scanning, and secret detection.

3. **Testcontainers Integration**: Real Fuseki instance in tests (not mocked) ensures integration tests closely match production behavior.

4. **BAMM Backward Compatibility**: Dual URN scheme support (`urn:samm:` and `urn:bamm:`) with prefix translation ensures smooth migration from legacy models.

5. **Fail-Closed Authentication**: `AuthorizationEvaluator` returns `false` on any JWT structural deviation — secure by default.

6. **State Machine Enforcement**: Lifecycle transitions are validated server-side with dependency checking (can't release if referenced models are still in DRAFT).

7. **Custom Resolution Strategy**: The `TripleStoreResolutionStrategy` enables ESMF SDK validation to resolve model dependencies from the triple store rather than the filesystem — essential for a centralized registry.

### Issues & Concerns

1. **Thread Safety in `SDKAccessHelperSAMM`**: The `aspectModel` field is mutable instance state updated during each request. Since `SDKAccessHelper` is a Spring `@Component` (singleton by default), concurrent requests would overwrite each other's model reference. This is a **race condition**.

2. **Pagination Offset Bug in `SparqlQueries.getOffset()`**: The offset calculation has an off-by-one error:
   - `page=0` → offset `0` ✓
   - `page=1` → offset `1 * pageSize` ✓
   - `page=2` → offset `(2-1) * pageSize = pageSize` ✗ (same as page 1)
   - `page=3` → offset `(3-1) * pageSize = 2 * pageSize` ✓
   
   This means page 1 and page 2 return the **same data**.

3. **Stale Javadoc**: `PersistenceLayer.getModels()` Javadoc mentions `nameFilter` and `nameType` parameters that don't exist in the method signature. The `buildExtendedSearchQuery()` method in `SparqlQueries` is dead code that was never wired up.

4. **Duplicate Constants**: `SAMM_ASPECT_URN_REGEX` and `BAMM_ASPECT_URN_REGEX` contain identical values — confusing naming.

5. **Manual SpringDoc Bootstrapping**: The `SemanticHubApplication` manually creates `SpringDocConfiguration`, `SpringDocConfigProperties`, and `ObjectMapperProvider` beans. This is fragile and may break on SpringDoc version upgrades. Root cause is likely the custom `@ComponentScan` overriding auto-configuration.

6. **`SDKAccessHelperSAMM` Not Spring-Managed**: It's instantiated with `new` inside `SDKAccessHelper`, and `PersistenceLayer` is injected via a setter called from `AspectModelService`'s constructor. This two-phase initialization is fragile and avoids Spring's lifecycle management.

7. **Broad Exception Catching**: `SAMMSdk.TripleStoreResolutionStrategy.apply()` uses `catch (Exception e)` for all resolution errors — makes debugging difficult as specific exceptions are swallowed.

8. **`determineModelType` Heuristic**: Uses `urn.contains("bamm")` to determine model type. This could misclassify a SAMM model that happens to have "bamm" in its namespace.

9. **Hardcoded Auxiliary Namespace**: `urn:bamm:io.openmanufacturing:aspect-model:aux#` uses the legacy BAMM namespace for status metadata, even for SAMM models. This is technical debt from the BAMM→SAMM migration.

10. **CI Workflow Versions**: Some workflows use older action versions (`checkout@v3` in some, `v4` in others; inconsistent `codeql-action` versions). This was partially fixed in our `VM-6` branch.

### Potential Improvements

| Area | Suggestion |
|------|-----------|
| Thread Safety | Make `SDKAccessHelperSAMM` methods create local `AspectModel` variables instead of using instance field |
| Pagination | Fix `getOffset()` to use `page * pageSize` consistently for 0-indexed pages |
| Dead Code | Remove `buildExtendedSearchQuery()` or wire it up for name/description search |
| Spring Management | Make `SDKAccessHelperSAMM` a Spring-managed bean with proper injection |
| Error Handling | Replace broad `catch (Exception)` with specific exception types in resolution strategy |
| Auxiliary Namespace | Migrate to a SAMM-based auxiliary namespace for consistency |
| CI Consistency | Standardize all workflows to latest stable action versions |
| Test Coverage | Add unit tests for `SparqlQueries` query generation |
| Documentation | Update stale Javadoc in `PersistenceLayer` |

---

*This analysis covers the complete repository structure, architecture, source code, APIs, testing strategy, CI/CD pipelines, deployment architecture, and identified issues as of the codebase state on 3 April 2026.*
