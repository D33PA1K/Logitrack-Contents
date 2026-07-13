
# Expert Refactoring Plan: Add, Modify, and Remove

Your current implementation is functional as a learning architecture, but it mixes several responsibilities:

* Gateway authentication and service authentication;
* local configuration and centralized configuration;
* explicit Gateway routes and automatic discovery routes;
* user-token propagation and service-to-service trust;
* business failures and infrastructure fallbacks.

The following plan preserves your existing architecture while making responsibilities explicit and production-oriented.

***

# 1. Target responsibility model

## API Gateway

The API Gateway should:

* expose the public port `8089`;
* route external requests;
* authenticate incoming JWTs;
* sanitize identity headers;
* handle browser CORS;
* optionally apply rate limiting and circuit breakers;
* forward the original bearer token downstream.

The Gateway should not:

* contain business authorization for every domain;
* connect to a database;
* use Servlet APIs;
* contain Feign clients for normal routing;
* become the route for every internal service-to-service call.

Spring Cloud Gateway is reactive and based on Spring WebFlux/Netty. Gateway routes consist of a destination URI, predicates, and filters, and the filter chain can modify requests before forwarding them.

## Config Server

Config Server should:

* centralize non-secret configuration;
* provide shared defaults;
* provide service-specific configuration;
* separate local, test, and production profiles.

It should not:

* contain hardcoded production secrets;
* contain business code;
* duplicate configuration inside its own `application.yml`;
* depend on Eureka unless there is a deliberate requirement.

## Service Registry

Eureka should:

* maintain service-instance locations;
* allow Gateway and Feign clients to locate services dynamically.

It should not:

* register with itself in standalone mode;
* fetch its own registry;
* contain Gateway, Feign, JWT, database, or LoadBalancer dependencies.

## Client microservices

Each business service should:

* own its database and business logic;
* own authorization rules for its endpoints;
* validate JWTs if direct/internal access must remain protected;
* register with Eureka;
* consume centralized configuration;
* declare only the Feign clients it genuinely needs.

***

# 2. ADD

# A. Add to the parent Maven POM

## Add one compatible Spring Cloud BOM

Your Spring Boot version is `3.2.3`. Manage Spring Cloud dependencies from the parent POM so every module receives compatible versions from one source.

```xml
<properties>
    <java.version>21</java.version>
    <spring-cloud.version>2023.0.1</spring-cloud.version>
    <jjwt.version>0.11.5</jjwt.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### Why

Without a BOM, modules can silently use incompatible versions of:

* Spring Cloud Gateway;
* Eureka Client/Server;
* OpenFeign;
* LoadBalancer;
* Config Client;
* Circuit Breaker.

Spring Cloud `2023.0.x` is the release-train generation associated with Spring Boot `3.2.x`.

Do not add individual Spring Cloud versions in child POMs after introducing the BOM.

***

# B. Add Config Client support to Gateway and business services

Add this dependency to:

* `api-gateway`;
* `identity-access-service`;
* all business microservices.

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

Each client should retain a minimal local `application.yml`:

```yaml
spring:
  application:
    name: logistics-analytics-service

  config:
    import: configserver:${CONFIG_SERVER_URL:http://localhost:8888}
```

For optional local fallback:

```yaml
spring:
  application:
    name: logistics-analytics-service

  config:
    import: optional:configserver:${CONFIG_SERVER_URL:http://localhost:8888}
```

### Expert recommendation

Use non-optional import for controlled environments:

```yaml
import: configserver:${CONFIG_SERVER_URL:http://localhost:8888}
```

This causes startup to fail immediately when required configuration cannot be loaded.

Use `optional:` only when local files contain a complete fallback configuration. Otherwise, a client might start with missing or unintended defaults.

Spring Config Data uses `spring.config.import`; `bootstrap.yml` is not required for this approach.

***

# C. Add a properly structured Config Repository

Create this directory outside any module's `target` folder:

```text
microservices_migration/
├── config-repo/
├── config-server/
├── service-registry/
├── api-gateway/
└── business-services/
```

Add:

```text
config-repo/
├── application.yml
├── api-gateway.yml
├── identity-access-service.yml
├── supplier-po-service.yml
├── warehouse-inventory-service.yml
├── route-carrier-service.yml
├── shipment-freight-service.yml
├── compliance-doc-service.yml
├── notification-alert-service.yml
└── logistics-analytics-service.yml
```

## Shared `application.yml`

```yaml
eureka:
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${server.port}:${random.value}

  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: ${EUREKA_URL:http://localhost:8761/eureka/}

management:
  endpoints:
    web:
      exposure:
        include: health,info

jwt:
  secret: ${JWT_SECRET}
  expiration: ${JWT_EXPIRATION:86400000}
```

### Why

`application.yml` in Config Server acts as shared configuration. Service-specific files override shared properties when necessary.

Avoid putting database URLs in the shared file because each service owns a different database.

***

# D. Add explicit Gateway CORS configuration

Place CORS configuration at the public boundary—the Gateway:

```yaml
spring:
  cloud:
    gateway:
      globalcors:
        cors-configurations:
          '[/**]':
            allowed-origins:
              - http://localhost:3000
            allowed-methods:
              - GET
              - POST
              - PUT
              - PATCH
              - DELETE
              - OPTIONS
            allowed-headers:
              - Authorization
              - Content-Type
              - Accept
            exposed-headers:
              - Location
            allow-credentials: true

      default-filters:
        - DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin
```

### Why

Browser traffic is supposed to terminate at the Gateway. CORS should therefore be enforced at that boundary rather than duplicated in every service.

CORS does not secure backend ports. CORS only controls browser behavior; network isolation and application authentication protect backend services.

***

# E. Add Gateway diagnostic endpoints

Add Actuator to the Gateway:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

For local development:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,gateway
```

Use:

```text
GET http://localhost:8089/actuator/health
GET http://localhost:8089/actuator/gateway/routes
```

### Why

The route endpoint allows you to verify that the Gateway actually loaded:

```text
lb://identity-access-service
```

This is particularly useful when configuration comes from Config Server.

Do not expose the Gateway route endpoint publicly in production.

***

# F. Add unique Feign `contextId` values

Modify each Feign declaration to include a consumer-specific context ID:

```java
@FeignClient(
    name = "shipment-freight-service",
    contextId = "analyticsShipmentClient",
    path = "/api/shipments",
    fallbackFactory = ShipmentClientFallbackFactory.class
)
public interface ShipmentClient {

    @GetMapping("/{id}")
    ShipmentDTO getShipmentById(
            @PathVariable("id") Integer id
    );

    @GetMapping
    List<ShipmentDTO> getAllShipments();
}
```

Another service consuming the same destination should use a different context:

```java
@FeignClient(
    name = "shipment-freight-service",
    contextId = "notificationShipmentClient",
    path = "/api/shipments"
)
```

### Why

Multiple clients may share the same Eureka service name but represent different consuming use cases. A unique `contextId` avoids Spring bean/configuration collisions without globally enabling bean replacement.

Feign uses the client name to build a load-balanced client; Eureka resolves physical instances when no fixed URL is configured.

***

# G. Add Feign circuit-breaker activation

In services that have Feign fallbacks, add:

```yaml
spring:
  cloud:
    openfeign:
      circuitbreaker:
        enabled: true
```

Add dependencies only to Feign-consuming services:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
```

Add reusable default resilience settings:

```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:
        registerHealthIndicator: true
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 3

  timelimiter:
    configs:
      default:
        timeoutDuration: 5s
        cancelRunningFuture: true
```

### Why

A declared `FallbackFactory` does not automatically mean every Feign operation is wrapped by a circuit breaker. Feign circuit-breaker integration must be enabled.

Resilience4j distinguishes reusable `configs` from specifically named `instances`.

***

# H. Add infrastructure-specific exception handling

Add:

```java
public class DownstreamServiceUnavailableException
        extends RuntimeException {

    public DownstreamServiceUnavailableException(
            String message,
            Throwable cause) {

        super(message, cause);
    }
}
```

Handle it with:

```java
@ExceptionHandler(DownstreamServiceUnavailableException.class)
public ResponseEntity<Map<String, Object>>
handleDownstreamUnavailable(
        DownstreamServiceUnavailableException exception) {

    Map<String, Object> body = Map.of(
            "status", 503,
            "error", "Service Unavailable",
            "message", exception.getMessage()
    );

    return ResponseEntity
            .status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(body);
}
```

### Why

An infrastructure outage must not be represented as valid empty business data.

These results mean different things:

```text
200 + []     = target service responded and found no records
404          = requested business resource does not exist
503          = target service could not be reached
```

***

# I. Add role normalization

Add one normalization function before constructing `SimpleGrantedAuthority`:

```java
private String normalizeRole(String role) {
    if (role == null || role.isBlank()) {
        return null;
    }

    return role.startsWith("ROLE_")
            ? role.substring("ROLE_".length())
            : role;
}
```

Then:

```java
String normalizedRole =
        normalizeRole(jwtUtil.extractRole(token));

SimpleGrantedAuthority authority =
        new SimpleGrantedAuthority(
                "ROLE_" + normalizedRole
        );
```

### Why

This prevents:

```text
ROLE_ROLE_ADMIN
```

when a JWT already stores the prefixed role.

Choose one token convention:

```json
{
  "role": "ADMIN"
}
```

This is preferable when your Java filter always adds `ROLE_`.

***

# J. Add correlation/error IDs

Update generic exception processing:

```java
@ExceptionHandler(Exception.class)
public ResponseEntity<Map<String, Object>> handleGeneric(
        Exception exception,
        HttpServletRequest request) {

    String errorId = UUID.randomUUID().toString();

    log.error(
            "Unhandled error. errorId={}, path={}",
            errorId,
            request.getRequestURI(),
            exception
    );

    Map<String, Object> body = Map.of(
            "status", 500,
            "error", "Internal Server Error",
            "message", "The request could not be completed",
            "errorId", errorId
    );

    return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(body);
}
```

### Why

The client receives a stable reference:

```text
errorId
```

The application log retains the full exception and stack trace.

***

# 3. MODIFY

# A. Modify the Service Registry configuration

Use:

```yaml
server:
  port: 8761

spring:
  application:
    name: service-registry

eureka:
  client:
    register-with-eureka: false
    fetch-registry: false

  server:
    enable-self-preservation: true

management:
  endpoints:
    web:
      exposure:
        include: health,info
```

### Architectural effect

The Registry becomes a standalone registry rather than simultaneously acting as its own client.

It should no longer:

* register `SERVICE-REGISTRY`;
* fetch its own application registry;
* treat `localhost:8761` as a peer node;
* generate the startup connection-refused stack trace.

***

# B. Modify Config Server path handling

Current:

```yaml
search-locations: file:./config-repo/
```

This path depends on the Java process working directory. If STS and the command line launch the application from different directories, Config Server may search different locations.

Use an environment-based absolute path:

```yaml
server:
  port: 8888

spring:
  application:
    name: config-server

  profiles:
    active: native

  cloud:
    config:
      server:
        native:
          search-locations: ${CONFIG_REPO_LOCATION:file:///C:/Users/2507291/Documents/FINAL%20VR/microservices_migration/config-repo}

management:
  endpoints:
    web:
      exposure:
        include: health,info
```

Spring Cloud Config's native profile reads configuration from the filesystem or classpath using `spring.cloud.config.server.native.search-locations`.

Test:

```text
http://localhost:8888/api-gateway/default
```

The result should contain non-empty `propertySources`.

***

# C. Modify the Gateway route configuration

Use explicit routes and disable automatic route creation:

```yaml
server:
  port: 8089

spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: false

      routes:
        - id: identity-access-service
          uri: lb://identity-access-service
          predicates:
            - Path=/api/auth/**,/api/users/**

        - id: supplier-po-service
          uri: lb://supplier-po-service
          predicates:
            - Path=/api/suppliers/**,/api/purchase-orders/**

        - id: warehouse-inventory-service
          uri: lb://warehouse-inventory-service
          predicates:
            - Path=/api/inventory/**,/api/inbound-receipts/**,/api/pick-lists/**

        - id: route-carrier-service
          uri: lb://route-carrier-service
          predicates:
            - Path=/api/routes/**,/api/carriers/**,/api/rate-cards/**

        - id: shipment-freight-service
          uri: lb://shipment-freight-service
          predicates:
            - Path=/api/shipments/**,/api/freight-orders/**,/api/delivery-events/**

        - id: compliance-doc-service
          uri: lb://compliance-doc-service
          predicates:
            - Path=/api/shipment-documents/**,/api/compliance-flags/**

        - id: notification-alert-service
          uri: lb://notification-alert-service
          predicates:
            - Path=/api/notifications/**

        - id: logistics-analytics-service
          uri: lb://logistics-analytics-service
          predicates:
            - Path=/api/logistics-reports/**
```

### Why disable Discovery Locator?

With the locator enabled, Gateway automatically creates service routes such as:

```text
/identity-access-service/**
/shipment-freight-service/**
```

You already have business-friendly public paths. Automatically exposing service-name paths adds another public route surface and makes access rules harder to reason about.

Explicit `lb://...` routes still use Eureka and LoadBalancer when the Discovery Locator is disabled. Discovery Locator controls automatic route generation, not service discovery itself.

***

# D. Modify the Gateway JWT filter

## Existing problems

Your current filter:

* uses `startsWith("/api/auth")`, making every authentication path public at Gateway;
* returns empty `401` responses;
* uses platform-default character encoding;
* injects headers without first removing client-supplied versions;
* validates authentication but not authorization;
* injects headers that business services currently ignore.

## Required behavior

The Gateway should:

1. allow only exact public method/path combinations;
2. remove untrusted identity headers;
3. validate bearer-token signature and expiration;
4. require subject and role claims;
5. install trusted identity headers;
6. preserve the original `Authorization` header for downstream validation;
7. return structured authentication errors.

Core public-path logic:

```java
private boolean isPublicRequest(
        HttpMethod method,
        String path) {

    return method == HttpMethod.POST
            && "/api/auth/login".equals(path);
}
```

Header sanitization:

```java
ServerHttpRequest modifiedRequest = request.mutate()
        .headers(headers -> {
            headers.remove("X-User-Id");
            headers.remove("X-User-Role");
            headers.set("X-User-Id", subject);
            headers.set("X-User-Role", role);
        })
        .build();
```

Signing key:

```java
secretKey.getBytes(StandardCharsets.UTF_8)
```

### Important design decision

Gateway authentication does not replace business-service authorization.

The Gateway proves:

```text
The token is valid and identifies this caller.
```

The business service decides:

```text
This caller may create, update, dispatch, cancel, or read this domain object.
```

***

# E. Modify business-service SecurityConfig classes

Do not use one system-wide authorization file in every service.

## Identity Service

Only identity-owned routes:

```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers(
        HttpMethod.POST,
        "/api/auth/login"
    ).permitAll()

    .requestMatchers(
        HttpMethod.POST,
        "/api/auth/register"
    ).hasRole("ADMIN")

    .requestMatchers(
        "/api/users/**"
    ).hasRole("ADMIN")

    .requestMatchers(
        HttpMethod.GET,
        "/api/audit-logs/**"
    ).hasAnyRole("ANALYST", "ADMIN")

    .anyRequest().authenticated()
)
```

## Analytics Service

Only analytics-owned routes:

```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers(
        HttpMethod.GET,
        "/api/logistics-reports/**"
    ).hasAnyRole("ANALYST", "ADMIN")

    .anyRequest().authenticated()
)
```

## Route and Carrier Service

```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers(
        HttpMethod.GET,
        "/api/routes/**",
        "/api/carriers/**",
        "/api/rate-cards/**"
    ).hasAnyRole("COORDINATOR", "ADMIN")

    .requestMatchers(
        "/api/routes/**",
        "/api/carriers/**",
        "/api/rate-cards/**"
    ).hasRole("ADMIN")

    .anyRequest().authenticated()
)
```

### Why

Spring Security rules apply only inside the process receiving the request.

A rule for:

```text
/api/shipments/**
```

inside `identity-access-service` does not protect `shipment-freight-service`.

This separation also prevents authorization drift and simplifies testing.

***

# F. Modify the Feign token interceptor

Use a configuration class rather than an unconstrained component:

```java
@Configuration
public class FeignSecurityConfiguration {

    @Bean
    public RequestInterceptor authorizationForwardingInterceptor() {
        return template -> {
            if (!(RequestContextHolder.getRequestAttributes()
                    instanceof ServletRequestAttributes attributes)) {
                return;
            }

            HttpServletRequest request = attributes.getRequest();

            String authorization =
                    request.getHeader(HttpHeaders.AUTHORIZATION);

            if (authorization != null
                    && authorization.startsWith("Bearer ")) {

                template.header(
                        HttpHeaders.AUTHORIZATION,
                        authorization
                );
            }
        };
    }
}
```

Attach it to relevant clients if you do not want it applied globally:

```java
@FeignClient(
    name = "shipment-freight-service",
    contextId = "analyticsShipmentClient",
    configuration = FeignSecurityConfiguration.class
)
```

### Why

A globally scanned `RequestInterceptor` applies to all Feign clients, including clients that might require a different authentication mechanism.

Explicit client configuration is clearer.

### Limitation

This interceptor works only for servlet request-bound synchronous operations.

It does not provide authentication for:

* scheduled tasks;
* message consumers;
* asynchronous threads;
* startup tasks.

Those require service credentials, not blind propagation of a missing user token.

***

# G. Modify Feign fallbacks

Replace:

```java
return null;
```

and:

```java
return Collections.emptyList();
```

with explicit infrastructure failure:

```java
@Override
public ShipmentDTO getShipmentById(Integer id) {
    throw new DownstreamServiceUnavailableException(
            "Shipment service is temporarily unavailable",
            cause
    );
}

@Override
public List<ShipmentDTO> getAllShipments() {
    throw new DownstreamServiceUnavailableException(
            "Shipment service is temporarily unavailable",
            cause
    );
}
```

### Why

Returning empty data creates incorrect analytics and can lead to incorrect business decisions.

A fallback should either:

* return truly meaningful cached/degraded data;
* or return an explicit `503`.

A fallback is not automatically equivalent to a successful empty response.

***

# H. Modify secret management

Replace every hardcoded value:

```yaml
jwt:
  secret: 5367566B59703373367639792F423F4528482B4D6251655468576D5A71347437
```

with:

```yaml
jwt:
  secret: ${JWT_SECRET}
  expiration: ${JWT_EXPIRATION:86400000}
```

Replace database credentials:

```yaml
username: root
password: root
```

with:

```yaml
username: ${DB_USERNAME:root}
password: ${DB_PASSWORD}
```

### Expert security note

Your current HS256 design uses the same key for:

* token creation;
* token verification.

Every service that can validate an HS256 token can also create a valid token.

For the immediate refactoring, environment-based secret sharing is acceptable for local development.

For production, migrate to asymmetric signing:

```text
Identity service: private RSA/EC key
Gateway and business services: public verification key
```

This preserves distributed validation without distributing token-signing capability.

***

# I. Modify service registration for local development

Use the shared Config Server setting:

```yaml
eureka:
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${server.port}:${random.value}
```

### Why

Your environment previously registered:

```text
LTIN709824.cts.com
```

A corporate machine hostname can be resolvable in one context but not another. Registering an IP for local development reduces `UnknownHostException` risk.

A unique instance ID also prevents multiple instances from overwriting one another.

***

# J. Modify `JwtFilter`

Your current filter catches invalid tokens and continues, which is acceptable if Spring Security later rejects protected endpoints. Improve clarity and role normalization:

```java
String authorization =
        request.getHeader(HttpHeaders.AUTHORIZATION);

if (authorization == null
        || !authorization.startsWith("Bearer ")) {

    filterChain.doFilter(request, response);
    return;
}

String token = authorization.substring(7);

try {
    if (jwtUtil.validateToken(token)
            && SecurityContextHolder.getContext()
                    .getAuthentication() == null) {

        String email = jwtUtil.extractEmail(token);
        String role = normalizeRole(
                jwtUtil.extractRole(token)
        );

        if (email != null && role != null) {
            SimpleGrantedAuthority authority =
                    new SimpleGrantedAuthority(
                            "ROLE_" + role
                    );

            UsernamePasswordAuthenticationToken authentication =
                    new UsernamePasswordAuthenticationToken(
                            email,
                            null,
                            List.of(authority)
                    );

            authentication.setDetails(
                    new WebAuthenticationDetailsSource()
                            .buildDetails(request)
            );

            SecurityContextHolder.getContext()
                    .setAuthentication(authentication);
        }
    }
} catch (Exception exception) {
    SecurityContextHolder.clearContext();
}

filterChain.doFilter(request, response);
```

### Behavior

* Public endpoints continue without a token.
* Valid token creates an authenticated principal.
* Invalid token does not create authentication.
* SecurityConfig determines whether the endpoint permits anonymous access.
* Protected endpoints return `401` or `403`.

***

# K. Modify API error behavior

Do not return:

```java
body.put(
    "detail",
    ex.getClass().getSimpleName()
        + ": "
        + ex.getMessage()
);
```

Return:

```json
{
  "status": 500,
  "error": "Internal Server Error",
  "message": "The request could not be completed",
  "errorId": "..."
}
```

Keep class names, SQL errors, remote URLs, and stack traces in application logs.

### Why

Exception messages can expose:

* table names;
* SQL fragments;
* hostnames;
* internal classes;
* configuration values;
* downstream system details.

***

# 4. REMOVE

# A. Remove `spring.cloud.config.enabled: false`

Remove from every Config Client:

```yaml
spring:
  cloud:
    config:
      enabled: false
```

### Why

This disables the Config Client while `spring.config.import` requests Config Server configuration.

Keep only:

```yaml
spring:
  config:
    import: configserver:${CONFIG_SERVER_URL:http://localhost:8888}
```

***

# B. Remove duplicate full configuration from client modules

After Config Server is verified, remove these from local client YAML files:

* `server.port`;
* datasource configuration;
* Eureka URL;
* JWT secret;
* Feign circuit-breaker configuration;
* Gateway route definitions.

Keep only bootstrap essentials:

```yaml
spring:
  application:
    name: api-gateway

  config:
    import: configserver:${CONFIG_SERVER_URL:http://localhost:8888}
```

### Why

Keeping the same property in Config Server and the module creates two sources of truth. Configuration precedence can make a service use a value other than the one being inspected.

***

# C. Remove automatic Gateway Discovery Locator

Remove or disable:

```yaml
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
```

Use:

```yaml
enabled: false
```

### Why

Your public API contract is explicitly defined through routes. Automatic routes can create alternate paths based directly on Eureka service IDs. Spring documents that Discovery Locator automatically creates load-balanced routes for discovered services.

***

# D. Remove CORS from business services

Remove copied `CorsConfig` classes from:

* Analytics;
* Shipment;
* Supplier;
* Warehouse;
* Route;
* Compliance;
* Notification;
* Identity, unless direct browser traffic is a documented requirement.

### Why

With Gateway-only browser access, business-service CORS is redundant.

The Gateway should be the only browser-facing CORS boundary.

***

# E. Remove system-wide authorization rules from every service

Remove unrelated matchers.

For example, remove this from Analytics Service:

```java
.requestMatchers("/api/users/**").hasRole("ADMIN")
.requestMatchers("/api/shipments/**").hasAnyRole(...)
.requestMatchers("/api/routes/**").hasAnyRole(...)
```

Keep only Analytics routes.

### Why

Unrelated matchers provide no cross-service protection. They increase confusion and maintenance cost.

***

# F. Remove `permitAll()` from internal audit writes

Remove:

```java
.requestMatchers(
    HttpMethod.POST,
    "/api/audit-logs"
).permitAll()
```

Replace it with authenticated service/user access:

```java
.requestMatchers(
    HttpMethod.POST,
    "/api/audit-logs"
).hasAnyRole("SYSTEM", "ADMIN")
```

### Why

`permitAll()` means unauthenticated callers can access the endpoint. It does not mean “only internal microservices.”

***

# G. Remove `allow-bean-definition-overriding`

Remove:

```yaml
spring:
  main:
    allow-bean-definition-overriding: true
```

Do this after adding unique Feign `contextId` values and resolving any remaining duplicate beans.

### Why

Bean overriding converts configuration conflicts into order-dependent behavior. One bean silently replaces another, making startup appear successful while the wrong configuration may be active.

***

# H. Remove unnecessary Feign from services

Remove OpenFeign dependencies and `@EnableFeignClients` from any service that does not call another service.

For example, a pure CRUD service with no outgoing synchronous HTTP calls does not need:

```xml
spring-cloud-starter-openfeign
spring-cloud-starter-loadbalancer
spring-cloud-starter-circuitbreaker-resilience4j
```

### Why

Feign belongs only in a service acting as an HTTP consumer.

Do not create a rule that every service must contain every client.

***

# I. Remove unnecessary dependencies from Eureka Server

Remove from `service-registry`:

```text
spring-cloud-starter-netflix-eureka-client
spring-cloud-starter-loadbalancer
spring-cloud-starter-openfeign
spring-cloud-starter-gateway
spring-boot-starter-security
jjwt-api
jjwt-impl
jjwt-jackson
database drivers
JPA
```

Keep:

```xml
spring-cloud-starter-netflix-eureka-server
spring-boot-starter-actuator
```

### Why

The Registry is infrastructure, not an application client or business service.

***

# J. Remove self-registration URL from standalone Registry

Remove:

```yaml
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

from the standalone Registry.

Keep that URL in Eureka clients through shared Config Server configuration.

***

# K. Remove `null` and empty-list outage fallbacks

Remove:

```java
return null;
return Collections.emptyList();
```

when the reason is a remote-service outage.

### Why

Infrastructure failure should not be reported as legitimate business state.

***

# L. Remove plaintext JWT secrets and DB passwords from source control

Remove actual values from:

* local `application.yml`;
* Config Repository;
* test scripts;
* documentation;
* Postman environment exports committed to Git.

Because the JWT secret was already posted and may already be committed, rotate it before using the application outside local development.

***

# M. Remove `spring-boot-starter-web` from API Gateway

The Gateway should use:

```xml
spring-cloud-starter-gateway
```

Do not add traditional MVC/Tomcat dependencies to the Gateway unless there is a deliberate and supported architecture for that combination.

Spring Cloud Gateway is built on WebFlux and Reactor Netty, not a traditional Servlet container.

***

# 5. Service-by-service resulting structure

## Service Registry

### Add

* Actuator.

### Modify

* standalone Eureka settings;
* local-only configuration.

### Remove

* Eureka client behavior;
* LoadBalancer;
* Feign;
* Gateway;
* JWT;
* database dependencies;
* `defaultZone` self-reference.

***

## Config Server

### Add

* Config repository;
* health endpoint;
* absolute/environment-based repository path.

### Modify

* native repository configuration;
* application/profile file organization.

### Remove

* business configuration from Config Server's own local YAML;
* plaintext secrets;
* ambiguous relative repository paths for controlled environments.

***

## API Gateway

### Add

* Config Client;
* Actuator;
* centralized CORS;
* precise JWT public-path handling;
* structured authentication errors;
* route diagnostics;
* trusted-header sanitization.

### Modify

* explicit route definitions;
* JWT parsing with UTF-8;
* application configuration loaded from Config Server;
* Eureka registration with a reachable address.

### Remove

* automatic Discovery Locator;
* MVC/Servlet dependencies;
* Feign for normal routing;
* business authorization rules;
* database dependencies;
* hardcoded secrets;
* `allow-bean-definition-overriding`.

***

## Identity Service

### Add

* Config Client;
* identity-specific SecurityConfig;
* centralized secret reference;
* optional token issuer/audience claims.

### Modify

* own only `/api/auth/**`, `/api/users/**`, and actual identity-owned audit endpoints;
* continue signing tokens;
* normalize role format.

### Remove

* authorization rules for Shipment, Warehouse, Route, Supplier, Analytics, and Notification;
* unrelated Feign clients;
* duplicated CORS when browser traffic goes through Gateway.

***

## Business microservices

### Add

* Config Client;
* service-specific authorization;
* Feign only for actual dependencies;
* unique Feign `contextId`;
* explicit `503` fallback;
* Actuator;
* environment-based database credentials.

### Modify

* validate JWT consistently;
* propagate bearer token only for request-bound Feign calls;
* separate infrastructure exceptions from business results.

### Remove

* all-domain SecurityConfig;
* duplicated browser CORS;
* hardcoded secrets;
* unused Feign clients;
* `null`/empty outage fallbacks;
* bean overriding after resolving collisions.

***

# 6. Recommended implementation order

Do not change everything simultaneously.

## Phase 1 — Dependency consistency

1. Add the Spring Cloud BOM.
2. Run:

```bat
mvn clean install -DskipTests
```

3. Resolve dependency compatibility errors before proceeding.

## Phase 2 — Stabilize Eureka

1. Configure standalone Registry.
2. Remove self-registration.
3. Verify:

```text
http://localhost:8761
```

## Phase 3 — Stabilize Gateway without Config Server

1. Keep Gateway routes locally.
2. Disable Discovery Locator.
3. Fix Gateway JWT filter.
4. Verify:

```text
POST http://localhost:8089/api/auth/login
```

Do not introduce Config Server until Gateway routing works reliably.

## Phase 4 — Enable Config Server

1. Add Config Client dependencies.
2. Create `config-repo`.
3. Move shared and service-specific configuration.
4. Verify Config endpoints.
5. Reduce client-local YAML files to bootstrap configuration.

## Phase 5 — Refactor security

1. Split SecurityConfig per service.
2. Remove unrelated matchers.
3. Remove business-service CORS.
4. Normalize roles.
5. Rotate JWT secret.

## Phase 6 — Refactor Feign

1. Remove unused Feign clients.
2. Add unique context IDs.
3. Enable circuit-breaker integration.
4. Replace false-data fallbacks with `503`.
5. Remove bean overriding.

***

# 7. Final expected behavior

## Login

```text
POST localhost:8089/api/auth/login
    → Gateway exact public-path bypass
    → Eureka resolves Identity Service
    → Identity Service authenticates credentials
    → Identity Service generates JWT
```

## Protected external request

```text
GET localhost:8089/api/logistics-reports
Authorization: Bearer <token>
    → Gateway validates JWT
    → Gateway sanitizes identity headers
    → Eureka resolves Analytics Service
    → Analytics validates JWT again
    → Analytics enforces ANALYST/ADMIN authorization
```

## Internal Feign request

```text
Analytics Service
    → Feign interceptor forwards user token
    → Eureka resolves Shipment Service
    → LoadBalancer selects an instance
    → Shipment Service validates JWT
    → Shipment Service applies shipment authorization
```

## Downstream outage

```text
Shipment Service unavailable
    → Feign circuit breaker records failure
    → fallback factory executes
    → Analytics returns 503
```

Not:

```text
200 OK
[]
```

This final structure gives each component one clear responsibility while preserving defence-in-depth authentication and Eureka-based internal load balancing.
