# Technical Design Specification (TDS) — Azam CRM

## 1. Overview
Azam CRM is a Spring Boot–based service that exposes REST APIs for:
- Agent onboarding, approval, update, and retry flows
- Customer registration, approval, update, and retry flows
- Common data such as Center Master, Currency Master, and onboarding dropdown lists
- Plant information retrieval via an external service

The service orchestrates business workflows, persists onboarding records (via MyBatis mappers), calls external services through OpenFeign, and issues consistent API responses. JWT-based authentication is expected on every request, with correlation/tracing headers propagated for logging.

## 2. Technology Stack
- Language/Runtime: Java 21
- Framework: Spring Boot 3.4.7
- REST: spring-boot-starter-web
- Persistence: MyBatis (mybatis-spring-boot-starter 3.0.4) + MyBatis XML mappers (src/main/resources/mappers)
- Mapping: MapStruct 1.5.5.Final (+ lombok-mapstruct-binding)
- Cloud: Spring Cloud 2024.0.1
  - spring-cloud-starter-config (Config Server client)
  - spring-cloud-starter-netflix-eureka-client (Service Discovery)
  - spring-cloud-starter-loadbalancer
  - spring-cloud-starter-openfeign (HTTP clients)
- HTTP client: Feign with Apache HttpClient (feign-httpclient, httpclient 4.5.13)
- Security/crypto: spring-security-crypto; JWT: io.jsonwebtoken (0.12.6)
- DB: MySQL (mysql-connector-j)
- Validation: jakarta.validation, hibernate-validator
- Logging: SLF4J + MDC
- Build: Maven; Plugins include spring-boot-maven-plugin and maven-compiler-plugin

## 3. Architecture
Layered package structure under com.vai.crm:
- config: Web and Feign configuration, tracing filter, properties binding
- constant: Cross-cutting constants for statuses, keys, and predicates
- controller: REST controllers for agents, customers, common data, and plants
- exception: Custom exceptions and global handler; custom Feign error decoder
- mapper: MapStruct mappers for domain-DTO transformations
- model: DTOs for request/response/wrapper/data/enums
- repository: MyBatis repository interfaces
- service (+ impl): Service interfaces and implementations encapsulating business logic
- transformer: Response transformers (DTO composition)
- util: Utility classes (validation, JWT, exception helpers)
- resources/mappers: MyBatis XML files (SQL statements)

Component responsibilities:
- Controllers: Validate JWT context via injected UserContext and delegate to services.
- Services: Implement business workflows, orchestrate repository operations and Feign calls, raise ApplicationException on business errors.
- Repositories: Abstractions over MyBatis mapper XML (CRUD operations).
- Mappers (MapStruct): Convert between internal beans, request/response DTOs, and client payloads.
- Config: 
  - TracingFilter populates MDC with trace/span IDs; validates JWT; populates request attributes used by controllers/resolvers.
  - UserContextResolver injects UserContext parameters into controller methods from request attributes.
  - FeignClientSSLConfig configures permissive SSL, custom Decoder supporting application/octet-stream, and ErrorDecoder (CustomFeignErrorDecoder).

### 3.1 Architecture Diagram
```mermaid
flowchart LR
  client[Client Apps\n(Web/Mobile/Backend)] -->|HTTPS REST + JWT| tf[TracingFilter]

  subgraph app[Azam CRM (Spring Boot)]
    direction TB
    tf --> ucr[UserContextResolver]
    ucr --> ctrl[Controllers\n(Agent, Customer, Common, Plant)]
    ctrl --> svc[Services\n(AgentService, CustomerService, CommonService, PlantService)]
    svc --> repo[Repositories (MyBatis)]
    svc --> mappers[MapStruct Mappers]
    svc --> feign[Feign Clients\n(OnboardingClient, PlantInfoClient)]
    svc --> transformer[Response Transformers]
    transformer --> ctrl

    subgraph cfg[Config]
      feignCfg[FeignClientSSLConfig]
      errDec[CustomFeignErrorDecoder]
    end
    cfg --> feign
  end

  repo --> db[(MySQL)]
  feign --> cm[(External CM Onboarding Service)]
  feign --> plant[(External Plant Info Service)]

  note1[[JWT Validation + MDC]]
  tf -. populates traceId/spanId, validates JWT, sets request attrs .-> note1
```

## 4. Code Flow (Main Features)
### 4.1 Agent Onboarding Flow
1) Controller: AgentController.agentOnboarding
- Accepts AgentOnboardingRequest + UserContext
- Delegates to AgentService.agentOnboardingService
2) Service (AgentServiceImpl.agentOnboardingService)
- Validates request (ValidatorUtil + agentOnboardingRequestValidator)
- Builds AgentDetailsBean (AgentMapper.toAgentDetailsBeanForOnboarding)
- Persists onboarding record via AgentRepository
- Inserts AgentActivity record
- Returns AgentDetailsBean for next steps (approval triggers CM calls)
3) Approval: AgentController.agentOnboardingApproval -> AgentServiceImpl.agentOnboardingApproval
- Loads agent by id and valid stage; updates approval status
- If approved and not a parent type (i.e., needs CM), constructs CM request (agentOnboardingClientRequestBuilder), invokes OnboardingClient.createOnboarding
- Handles success/failure via mapper methods; updates DB; records activity; creates user if applicable
4) Update: AgentController.agentDetailsUpdate -> AgentServiceImpl.agentDetailsUpdate
- For non-parent types, calls CM update (OnboardingClient.updateOnboarding) then updates DB and activity log with change list
5) Retry: AgentController.retryAgentOnboarding -> AgentServiceImpl.retryAgentOnboarding
- Re-fetches record on RETRY stage and re-invokes CM onboarding

### 4.2 Customer Flows
- Registration: CustomerController.customerRegistration -> CustomerServiceImpl.customerRegistration: validate, insert onboarding record, return wrapper
- Approval & CM onboarding: CustomerController.customerOnboardingApproval -> CustomerServiceImpl.customerOnboardingApproval -> customerOnboardingProcessInCM; on success, update DB and create user
- Retry: retryCustomerOnboarding mirrors agent retry for RETRY stage
- Registration update: customerRegistrationUpdate updates onboarding data

### 4.3 Common Data
- Center Master & Currency Master via CommonController -> CommonServiceImpl -> CommonRepository
- Onboarding dropdowns: CommonController.getOnboardingDropDowns calls CommonServiceImpl.getOnboardingDropDowns which composes role-filtered dropdowns from MultiLevelDataBean list
- MZ Mapping (TreeMap) derived via CommonServiceImpl.getOnboardingMzMapping and used by mapping/builders

### 4.4 Plant Info
- PlantController delegates to PlantServiceImpl, which calls PlantInfoClient to fetch plant info; maps response list through PlantResponseTransformer

## 5. Data Flow
- Request DTOs (model.request.*) arrive at controllers. UserContext is injected via UserContextResolver (from request attributes set by TracingFilter JWT validation).
- Service layer maps requests to domain beans (model.data.*) using MapStruct mappers, persists via repository interfaces backed by MyBatis XML mappers (in resources/mappers).
- External API calls use Feign clients (OnboardingClient, PlantInfoClient). Responses are wrapped in OnboardingResponseWrapper/PlantInfoResponseWrapper subtypes that indicate status.
- Based on responses, services map to updated domain beans and persist state transitions (stages, messages, sapBpId) and write activity logs.
- Responses are transformed to API response DTOs (model.response.*) and returned via ResponseBuilder.success/failure in a consistent envelope (GenericResponse).

## 6. APIs / Integrations
- External Services:
  - OnboardingClient (url: ${crm-service.onboardingClientUrl})
    - POST /onboarding/new — create onboarding in CM
    - POST /onboarding/update — update onboarding in CM
    - GET /onboarding/ — list onboardings
    - GET /onboarding/customerInfo?BP={bp} — fetch details by BP ID
  - PlantInfoClient (url: ${crm-service.plantInfoClientUrl})
    - GET /plantInfo — all plants
    - GET /plantInfo?plan_number={num} — by plan number
- Feign configuration:
  - SSL: Trust-all with NoopHostnameVerifier (intended for non-prod; hardening recommended)
  - Decoder: Supports application/json and application/octet-stream
  - ErrorDecoder: CustomFeignErrorDecoder routes errors to domain exceptions (OnboardingClientException, PlantInfoClientException) based on configured context substrings

## 7. Error Handling & Logging
- ApplicationException carries an HTTP status code; GlobalExceptionHandler maps to GenericResponse failure.
- Global handler also unwraps enum deserialization failures (AgentType) into ApplicationException messages.
- TracingFilter sets MDC traceId/spanId (from header X-Trace-Id or generated), validates JWT token via JwtUtil and UserService, and populates request attributes (userId, userName, userRole, userAllAccessRole). Missing/invalid tokens raise ApplicationException.
- Controllers and services log key events; activities are recorded as domain records for auditing.

## 8. Setup & Deployment
- Requirements: Java 21, Maven, MySQL, access to Config Server/Eureka (if used), and external CM endpoints.
- Configuration (application.yml or via Config Server):
  - crm-service.secret, crm-service.salt — JWT/crypto
  - crm-service.permitPaths — permitted endpoint patterns
  - crm-service.onboardingClientUrl — CM onboarding base URL
  - crm-service.plantInfoClientUrl — Plant info base URL
  - crm-service.onboardingClientContext, crm-service.plantInfoClientContext — substrings used by CustomFeignErrorDecoder to route error body parsing
  - DB settings for MyBatis/MySQL
  - Eureka client, Config server endpoints (if applicable)
- Build and Run:
  - mvn clean package
  - mvn spring-boot:run or run the generated jar
- Packaging: spring-boot-maven-plugin is configured; war packaging not currently active (SpringBootServletInitializer import present but not used).

## 9. Guidelines for New Development
- New Endpoints:
  - Create a controller method under com.vai.crm.controller
  - Define request/response DTOs under model.request/response
  - Add service interface method and implementation under service/impl
  - If persistence required, add repository interface methods and corresponding MyBatis XML SQL
  - Use MapStruct mappers for DTO<->domain conversions; configure additional @Mappings as needed
  - Validate inputs via ValidatorUtil and jakarta.validation annotations
  - For external calls, define a Feign client in service.feignClient and (optionally) add error handling to CustomFeignErrorDecoder
  - Return responses via ResponseBuilder.success/failure within GenericResponse envelope
- Error Handling:
  - Throw ApplicationException with appropriate HTTP code for business errors
  - Let GlobalExceptionHandler translate to API responses
- Logging & Tracing:
  - Use SLF4J with MDC (traceId/spanId auto-populated by TracingFilter)
  - Avoid logging secrets or PII
- Security:
  - Expect Authorization: Bearer <token> header; TracingFilter validates and sets attributes for UserContextResolver
- MapStruct & Lombok:
  - Keep mapper logic declarative; use @AfterMapping for complex compositions
  - Ensure maven-compiler-plugin has annotationProcessorPaths (already configured)
- Testing:
  - Add unit tests for mappers and services
  - Consider integration tests with @SpringBootTest and @AutoConfigureMockMvc for controllers

## 10. Notable Conventions & Constants
- Stages and statuses live in ApplicationConstant (e.g., CAPTURED, RELEASE_TO_KYC, COMPLETED, RETRY)
- Error codes and retryability logic in ErrorConstant
- Keys for request attributes and MDC in ApplicationConstant (USER_ID, USER_ROLE, TRACE_ID, SPAN_ID, etc.)

## 11. File/Code Pointers
- Main: src/main/java/com/vai/crm/AzamCrmApplication.java
- Controllers: src/main/java/com/vai/crm/controller
- Services: src/main/java/com/vai/crm/service/impl
- Repositories: src/main/java/com/vai/crm/repository (with SQL XML under src/main/resources/mappers)
- Mappers (MapStruct): src/main/java/com/vai/crm/mapper
- Config: src/main/java/com/vai/crm/config
- Exceptions: src/main/java/com/vai/crm/exception
- DTOs: src/main/java/com/vai/crm/model

## 12. MapStruct Mappers — Detailed Flow and Usage
This project uses MapStruct to generate type-safe, compile-time mappers between:
- Request DTOs (model.request.*)
- Domain/data beans persisted in DB (model.data.*)
- External client payloads for CM (AgentOnboardingClientRequest)
- Response DTOs/Wrappers (model.response.*, model.wrapper.*)

Configuration
- All mappers are annotated with @Mapper(componentModel = "spring") so Spring injects generated implementations as beans.
- The maven-compiler-plugin configures annotationProcessorPaths for MapStruct, Lombok, and the lombok-mapstruct-binding.
- Generated sources are available in target\generated-sources\annotations\com\vai\crm\mapper.

Key Patterns Used
- @Mapping to map fields directly, or via constant/expression/source.
- default helper methods inside mapper interfaces for reusable logic.
- @AfterMapping to compose nested/collection fields that are easier to build programmatically (e.g., contactMedium, relatedParty).
- @BeanMapping(ignoreByDefault = true) to perform partial updates into domain beans during status transitions, without touching unrelated fields.

12.1 AgentMapper
Primary responsibilities:
- Build AgentDetailsBean for DB from AgentOnboardingRequest:
  - toAgentDetailsBeanForOnboarding sets agentStage using setValidStatus(request) which leverages ApplicationConstant.isAnyKycPresent; status is set to SUCCESS.
- Build CM request from AgentDetailsBean:
  - toAgentOnboardingClientRequest maps basic fields and defers complex fields.
  - @AfterMapping fills:
    - contactMedium list (mobile/phone/fax/email) using StringUtils checks and constants MOBILE_TYPE, PHONE_TYPE, FAX, EMAIL_TYPE.
    - AddressContact (billing) when country is present.
    - relatedParty via getRelatedParty, which pulls MZ mapping values from a TreeMap (provided by CommonService.getOnboardingMzMapping).
- Handle CM responses (success/error/connectivity):
  - agentOnboardingClientSuccessMapper uses:
    - getSapBpId: extracts sapBpId from first RelatedParty item in the CM success response.
    - getAgentStageSuccess: returns RELEASE_TO_CM when status = P (in-progress) else COMPLETED for S.
  - agentOnboardingClientErrorMapper uses ErrorConstant.isRetryable to derive:
    - agentStage: RETRY for retryable codes (F + CMONB1), else FAILED.
    - statusMsg: CM_RETRY_MSG or CM_FAILED_MSG.
    - Also copies CM error attributes (cmStatus, cmStatusCode, cmErrorReason, cmStatusMsg) into domain bean.
    - @BeanMapping(ignoreByDefault = true) ensures only relevant fields are updated.
  - agentOnboardingConnectivityErrorMapper sets FAILED status with RETRY stage and a portal message when exceptions are due to connectivity (timeouts, etc.).
- Approval and updates:
  - updateAgentApprovalUpdateMapper maps approval stage changes with updateId and type.
  - insertAgentActivityMapper creates an AgentActivityBean logging the action, request transaction id, and user metadata.
  - updateAgentDetailsMapper helps build a partial update bean when updating details (uses sapBpId for non-parent types).
  - getAgentByIdMapper converts a CM info response to AgentDetailsResponse for read endpoints.
  - getAgentActivitiesMapper supports activity list composition; used by AgentDetailsChangeBuilder in service.
  - toAgentDetailsBeanForOnbUpdate processes registration detail updates.

Where AgentMapper is used in flow:
- AgentServiceImpl.agentOnboardingService -> toAgentDetailsBeanForOnboarding -> DB insert -> insertAgentActivityMapper.
- AgentServiceImpl.agentOnboardingApproval -> agentOnboardingClientRequestBuilder -> OnboardingClient.createOnboarding:
  - Success -> agentOnboardingClientSuccessMapper -> repository.update -> insertAgentActivityMapper -> userService.userCreation.
  - Error -> agentOnboardingClientErrorMapper -> repository.update -> insertAgentActivityMapper.
  - Connectivity -> agentOnboardingConnectivityErrorMapper -> repository.update -> insertAgentActivityMapper.
- AgentServiceImpl.agentDetailsUpdate -> updateAgentDetailsMapper then builds client request for CM update; upon success, repository.update + activities.

12.2 CustomerMapper
Mirrors AgentMapper concepts for customer workflows:
- toCustomerDetailsBeanForOnboarding sets initial customerStage using setValidStatus (based on isAnyKycPresentForCustomer) and derives connectionType (POSTPAID or PREPAID) using isPostPaidConnection predicate.
- toOnboardingClientRequest + @AfterMapping:
  - Builds contactMedium, billing and installation addresses.
  - Builds relatedParty with division and salesOrg fetched from MZ mapping (TreeMap) keys CUSTOMER_TYPE/DIVISION_TYPE/SALES_ORG.
- Response mapping:
  - customerOnboardingClientSuccessMapper sets sapBpId and decides stage via getCustomerStageSuccess (RELEASE_TO_CM vs COMPLETED) using P vs S status.
  - customerOnboardingClientErrorMapper and customerOnboardingConnectivityErrorMapper mirror the agent mappers for error paths.
- Approval and updates:
  - customerApprovalUpdateMapper updates approval stage.
  - toCustomerDetailsBeanForOnbUpdate recalculates connectionType for updates using generateConnectionTypeForUpdate.

12.3 UserMapper
- Maps AgentDetailsBean/CustomerDetailsBean to UserDetailsBean for local user creation after successful onboarding.
- Uses @Context Map<String, String> to pass generated credentials (userName, password) into the mapping expressions, sets userType accordingly (agent type from bean or CUSTOMER_ROLE), and sets isPwdReset = "N" consistently.

12.4 MZ Mapping (TreeMap) and MapStruct cooperation
- CommonServiceImpl.getOnboardingMzMapping builds a nested TreeMap (ENTITYMAP -> MZ_MAPPING_LIST -> ONBOARD) from MultiLevelDataBean rows.
- Agent/Customer mappers receive this TreeMap to translate UI/DB values (e.g., division, salesOrg, type) into CM codes when building AgentOnboardingClientRequest.
- This keeps CM-specific code tables out of service logic and centralized in mapper composition.

12.5 Edge cases and conventions in mappers
- Null/blank checks: Contact mediums and addresses are only added when source values are non-empty (StringUtils checks).
- @BeanMapping(ignoreByDefault = true) is used for partial state updates from CM responses; this prevents unintended overwrites of other fields.
- Predicates in ApplicationConstant and ErrorConstant encapsulate business rules used inside default mapper methods, keeping mapping configurations tidy and testable.
- The success mapper extracts sapBpId from the first RelatedParty; ensure that CM always returns at least one item, or guard accordingly if API changes.

12.6 Testing recommendations for mappers
- Unit-test default methods (e.g., setValidStatus, getAgentStageSuccess, generateConnectionType) directly on the generated mapper.
- Verify @AfterMapping builds the correct contactMedium and relatedParty elements given various input combinations.
- Validate error mappers set agent/customer stage and messages correctly based on retryable vs non-retryable codes.

## 13. End-to-End Flow Examples (Expanded)
Example: Agent onboarding (non-parent type)
- Controller receives AgentOnboardingRequest and UserContext.
- Service validates and calls AgentMapper.toAgentDetailsBeanForOnboarding -> DB insert.
- On approval, Service prepares MZ mapping via CommonService.getOnboardingMzMapping, calls AgentMapper.toAgentOnboardingClientRequest, enriches EngagedParty, then OnboardingClient.createOnboarding.
- On success (S or P): AgentMapper.agentOnboardingClientSuccessMapper produces a partial update bean; repository updates DB, activity inserted; userService.userCreation is invoked with updated sapBpId.
- On failure: AgentMapper.agentOnboardingClientErrorMapper maps CM error metadata and sets stage/status message for retry logic.

Example: Customer onboarding
- Same pattern with CustomerMapper; connectionType set to POSTPAID for specific account classes (isAllowForPostpaid) otherwise PREPAID.

Notes for developers
- If you add new fields to requests or beans, prefer adding @Mapping lines rather than writing imperative mapping code in services.
- For complex derived fields, add small default helper methods and call them via expression or from @AfterMapping.

---
This TDS is intended as a concise quick-start guide for new developers joining the Azam CRM project. For questions or updates, add notes here or link to further docs.