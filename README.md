# 🚀 DevOps Roadmap: Java Spring Boot + Angular/TypeScript
## Scalable, Version-Managed Full-Stack Applications

---

## 🏗️ PROJECT ARCHITECTURE OVERVIEW

```
monorepo/
├── backend/                     # Java Spring Boot
│   ├── pom.xml                  # Maven dependency management
│   ├── src/
│   │   └── main/java/
│   │       ├── config/
│   │       ├── controller/
│   │       ├── service/
│   │       └── repository/
│   └── Dockerfile
├── frontend/                    # Angular + TypeScript
│   ├── package.json             # npm dependency management
│   ├── angular.json
│   ├── src/app/
│   └── Dockerfile
├── infrastructure/              # IaC (Terraform / Helm)
├── .github/workflows/           # CI/CD pipelines
├── docker-compose.yml
└── k8s/                         # Kubernetes manifests
```

---

## 📦 DEPENDENCY MANAGEMENT (Core Foundation)

### Backend – Maven (pom.xml)

```xml
<!-- Use Spring Boot BOM to control ALL dependency versions from one place -->
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>3.2.5</version>   <!-- Pin this version explicitly -->
</parent>

<!-- Version properties block — upgrade/downgrade by changing ONE line -->
<properties>
  <java.version>21</java.version>
  <mapstruct.version>1.5.5.Final</mapstruct.version>
  <springdoc.version>2.5.0</springdoc.version>
  <testcontainers.version>1.19.8</testcontainers.version>
  <lombok.version>1.18.32</lombok.version>
</properties>

<!-- Dependency Management block for third-party libs not in BOM -->
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.testcontainers</groupId>
      <artifactId>testcontainers-bom</artifactId>
      <version>${testcontainers.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

**Key commands:**
```bash
mvn versions:display-dependency-updates   # See what can be upgraded
mvn versions:use-latest-releases          # Auto-upgrade (use carefully)
mvn dependency:tree                       # Show dependency graph
mvn dependency:analyze                    # Find unused/missing deps
```

### Frontend – npm + package.json

```json
{
  "name": "my-app-frontend",
  "version": "1.0.0",
  "engines": {
    "node": ">=20.0.0",
    "npm": ">=10.0.0"
  },
  "dependencies": {
    "@angular/core": "17.3.0",
    "@angular/router": "17.3.0",
    "rxjs": "7.8.1",
    "zone.js": "0.14.6"
  },
  "devDependencies": {
    "@angular/cli": "17.3.0",
    "typescript": "5.4.5",
    "jest": "29.7.0",
    "@angular-eslint/eslint-plugin": "17.3.0"
  }
}
```

**Key commands:**
```bash
npm outdated                    # See stale packages
npm update <package>@<version>  # Upgrade specific package
npx ng update @angular/core     # Angular-safe upgrade wizard
npm ci                          # Reproducible install from lock file
npm shrinkwrap                  # Lock deps for production
```

---

## 🟢 BEGINNER PROJECTS (Foundation)

---

### Project 1 — Local Full-Stack Dev Environment with Docker Compose

**Goal:** Run Spring Boot + Angular + PostgreSQL + Redis locally with one command.

**Stack:** Docker Compose, Maven, npm, Nginx

**docker-compose.yml:**
```yaml
version: "3.9"

services:
  postgres:
    image: postgres:16.2-alpine        # Always pin image versions!
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7.2.4-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}

  backend:
    build:
      context: ./backend
      target: development            # Multi-stage: dev vs prod
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/appdb
      SPRING_REDIS_HOST: redis
    ports:
      - "8080:8080"
    depends_on:
      postgres:
        condition: service_healthy
    volumes:
      - ./backend:/app               # Hot reload in dev

  frontend:
    build:
      context: ./frontend
      target: development
    ports:
      - "4200:4200"
    volumes:
      - ./frontend/src:/app/src      # Angular live reload
    environment:
      API_URL: http://localhost:8080

volumes:
  postgres_data:
```

**Backend Dockerfile (multi-stage):**
```dockerfile
# Stage 1: Build
FROM maven:3.9.6-eclipse-temurin-21-alpine AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -B      # Cache dependencies layer
COPY src ./src
RUN mvn package -DskipTests -B

# Stage 2: Development (with debugger)
FROM eclipse-temurin:21-jre-alpine AS development
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080 5005
ENTRYPOINT ["java", "-agentlib:jdwp=transport=dt_socket,server=y,address=*:5005", "-jar", "app.jar"]

# Stage 3: Production (minimal image)
FROM eclipse-temurin:21-jre-alpine AS production
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75", "-jar", "app.jar"]
```

**Frontend Dockerfile (multi-stage):**
```dockerfile
# Stage 1: Build
FROM node:20.12.2-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --prefer-offline          # Reproducible install
COPY . .
RUN npm run build -- --configuration=production

# Stage 2: Development (live reload)
FROM node:20.12.2-alpine AS development
WORKDIR /app
COPY --from=build /app/node_modules ./node_modules
COPY . .
EXPOSE 4200
CMD ["npx", "ng", "serve", "--host", "0.0.0.0", "--poll=2000"]

# Stage 3: Production (Nginx static)
FROM nginx:1.25.4-alpine AS production
COPY --from=build /app/dist/my-app /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

**Skills learned:** Docker multi-stage builds, environment variables, health checks, volume mounts for hot reload.

---

### Project 2 — Version-Controlled Configuration with Spring Profiles + Angular Environments

**Goal:** Manage dev/staging/prod configs without hardcoding.

**Spring Backend — application.yml:**
```yaml
# Shared defaults
spring:
  application:
    name: my-app-backend
  datasource:
    hikari:
      maximum-pool-size: 10

---
# Dev profile
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:h2:mem:testdb        # H2 in-memory for local dev
  jpa:
    show-sql: true
logging.level.root: DEBUG

---
# Production profile
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: ${DATABASE_URL}           # Injected at runtime from secret
  jpa:
    show-sql: false
logging.level.root: WARN
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
```

**Angular Environments:**
```
frontend/src/environments/
├── environment.ts          # dev (default)
├── environment.staging.ts
└── environment.prod.ts
```

```typescript
// environment.ts
export const environment = {
  production: false,
  apiUrl: 'http://localhost:8080/api/v1',
  featureFlags: {
    newDashboard: true,
    betaCheckout: false,
  }
};

// environment.prod.ts
export const environment = {
  production: true,
  apiUrl: 'https://api.myapp.com/api/v1',
  featureFlags: {
    newDashboard: true,
    betaCheckout: false,
  }
};
```

**Build with specific env:**
```bash
ng build --configuration=staging
ng build --configuration=production
```

---

### Project 3 — GitHub Actions CI Pipeline (Build + Test + Lint)

**Goal:** Automate build, test, and code quality checks on every push.

**.github/workflows/ci.yml:**
```yaml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  JAVA_VERSION: '21'
  NODE_VERSION: '20'

jobs:
  backend-ci:
    name: Backend Build & Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK ${{ env.JAVA_VERSION }}
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: temurin
          cache: maven                    # Cache .m2 between runs

      - name: Run tests with coverage
        working-directory: ./backend
        run: mvn verify -B --no-transfer-progress

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          files: backend/target/site/jacoco/jacoco.xml

  frontend-ci:
    name: Frontend Build & Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Node ${{ env.NODE_VERSION }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: npm
          cache-dependency-path: frontend/package-lock.json

      - name: Install dependencies
        working-directory: ./frontend
        run: npm ci

      - name: Lint
        working-directory: ./frontend
        run: npm run lint

      - name: Unit tests
        working-directory: ./frontend
        run: npm run test -- --watch=false --browsers=ChromeHeadless

      - name: Build production
        working-directory: ./frontend
        run: npm run build -- --configuration=production
```

---

## 🟡 INTERMEDIATE PROJECTS (Scalability & Automation)

---

### Project 4 — Semantic Versioning + Automated Releases

**Goal:** Auto-increment versions on merge, generate changelogs, tag releases.

**Strategy:** Conventional Commits + semantic-release

**Commit convention:**
```
feat: add user authentication endpoint     → minor bump (1.0.0 → 1.1.0)
fix: correct JWT expiry calculation        → patch bump (1.1.0 → 1.1.1)
feat!: redesign REST API structure         → major bump (1.1.1 → 2.0.0)
chore: update dependencies                 → no release
```

**.github/workflows/release.yml:**
```yaml
name: Release

on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          persist-credentials: false

      - name: Semantic Release
        uses: cycjimmy/semantic-release-action@v4
        with:
          semantic_version: 23
          branches: '[{"name": "main"}]'
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      # Update pom.xml version
      - name: Update Maven version
        if: steps.semantic.outputs.new_release_published == 'true'
        run: |
          mvn versions:set -DnewVersion=${{ steps.semantic.outputs.new_release_version }} -f backend/pom.xml

      # Update package.json version
      - name: Update npm version
        if: steps.semantic.outputs.new_release_published == 'true'
        run: |
          npm version ${{ steps.semantic.outputs.new_release_version }} --no-git-tag-version
        working-directory: ./frontend
```

**Maven version management commands:**
```bash
# Check current version
mvn help:evaluate -Dexpression=project.version -q -DforceStdout

# Upgrade Spring Boot version (then re-test!)
mvn versions:set -DnewVersion=3.3.0 -f pom.xml
mvn verify   # Test that nothing broke

# Downgrade a specific dependency
mvn versions:set-property -Dproperty=mapstruct.version -DnewVersion=1.5.3.Final
```

---

### Project 5 — Blue/Green Deployment with Zero Downtime

**Goal:** Deploy new version, test it, then switch traffic — with instant rollback.

**Concept:**
```
Load Balancer
     │
 ┌───┴────┐
 │        │
Blue     Green
(v1.2)  (v1.3 NEW)
```
Traffic starts on Blue. Deploy Green with new version. Switch load balancer. If anything fails, switch back in < 30 seconds.

**Kubernetes deployment strategy:**
```yaml
# backend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-green                  # Toggle blue/green here
  labels:
    app: backend
    slot: green
    version: "1.3.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
      slot: green
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0               # Zero downtime guarantee
  template:
    spec:
      containers:
        - name: backend
          image: myregistry/backend:1.3.0   # Pinned version tag
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10

---
# service.yaml — switch traffic by changing selector
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
    slot: green             # ← Change to 'blue' to rollback instantly
  ports:
    - port: 80
      targetPort: 8080
```

**Rollback script:**
```bash
#!/bin/bash
# rollback.sh
CURRENT=$(kubectl get service backend-service -o jsonpath='{.spec.selector.slot}')
TARGET=$([[ "$CURRENT" == "green" ]] && echo "blue" || echo "green")

echo "Rolling back from $CURRENT → $TARGET"
kubectl patch service backend-service -p "{\"spec\":{\"selector\":{\"slot\":\"$TARGET\"}}}"
echo "✅ Rollback complete in $(date)"
```

---

### Project 6 — Canary Releases with Feature Flags

**Goal:** Release new features to 5% of users, monitor, then gradually roll out.

**Backend — Flagsmith/LaunchDarkly integration:**
```java
@RestController
@RequestMapping("/api/v1/dashboard")
public class DashboardController {

    private final FeatureFlagService flags;

    @GetMapping
    public ResponseEntity<DashboardDto> getDashboard(@AuthenticationPrincipal User user) {
        if (flags.isEnabled("new-dashboard-v2", user.getId())) {
            return ResponseEntity.ok(dashboardServiceV2.build(user));  // New 5% path
        }
        return ResponseEntity.ok(dashboardServiceV1.build(user));      // Stable 95% path
    }
}
```

**Angular — Feature flag guard:**
```typescript
// feature-flag.guard.ts
@Injectable({ providedIn: 'root' })
export class FeatureFlagGuard implements CanActivate {
  constructor(private flags: FeatureFlagService) {}

  canActivate(route: ActivatedRouteSnapshot): Observable<boolean> {
    const flagName = route.data['flag'] as string;
    return this.flags.isEnabled(flagName).pipe(
      tap(enabled => {
        if (!enabled) this.router.navigate(['/not-found']);
      })
    );
  }
}

// app-routing.module.ts
{
  path: 'dashboard-v2',
  component: DashboardV2Component,
  canActivate: [FeatureFlagGuard],
  data: { flag: 'new-dashboard-v2' }
}
```

---

### Project 7 — Centralized Logging (ELK Stack)

**Goal:** Aggregate logs from backend + frontend into searchable Elasticsearch.

**Spring Boot — Structured JSON logging:**
```xml
<!-- pom.xml -->
<dependency>
  <groupId>net.logstash.logback</groupId>
  <artifactId>logstash-logback-encoder</artifactId>
  <version>7.4</version>
</dependency>
```

```xml
<!-- logback-spring.xml -->
<configuration>
  <springProfile name="prod">
    <appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
      <destination>logstash:5044</destination>
      <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        <customFields>{"app":"backend","version":"${APP_VERSION}"}</customFields>
      </encoder>
    </appender>
    <root level="INFO">
      <appender-ref ref="LOGSTASH"/>
    </root>
  </springProfile>
</configuration>
```

**Angular — Frontend error tracking:**
```typescript
// error-handler.service.ts
@Injectable({ providedIn: 'root' })
export class GlobalErrorHandler implements ErrorHandler {
  constructor(private logger: LoggingService) {}

  handleError(error: Error): void {
    this.logger.sendToLogstash({
      level: 'ERROR',
      message: error.message,
      stack: error.stack,
      url: window.location.href,
      userAgent: navigator.userAgent,
      timestamp: new Date().toISOString(),
      appVersion: environment.version
    });
    console.error(error);
  }
}

// app.module.ts
providers: [{ provide: ErrorHandler, useClass: GlobalErrorHandler }]
```

---

### Project 8 — API Versioning Strategy (Backend + Frontend)

**Goal:** Upgrade API without breaking existing clients.

**Spring Boot — URL versioning:**
```java
// Old stable endpoint
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 { ... }

// New endpoint with breaking changes
@RestController
@RequestMapping("/api/v2/users")
public class UserControllerV2 { ... }
```

**Spring Boot — Header versioning (more flexible):**
```java
@GetMapping(value = "/users", headers = "API-Version=1")
public List<UserDtoV1> getUsersV1() { ... }

@GetMapping(value = "/users", headers = "API-Version=2")
public Page<UserDtoV2> getUsersV2(@RequestParam int page) { ... }
```

**Angular — API version interceptor:**
```typescript
// api-version.interceptor.ts
@Injectable()
export class ApiVersionInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<unknown>, next: HttpHandler): Observable<HttpEvent<unknown>> {
    const versioned = req.clone({
      setHeaders: {
        'API-Version': environment.apiVersion,   // '2' in environment.prod.ts
        'App-Version': environment.appVersion,
      }
    });
    return next.handle(versioned);
  }
}
```

**API deprecation lifecycle:**
```
v1 Released → v2 Released → v1 Deprecated (warn header) → v1 Sunset (6 months later)
```

```java
// Add deprecation warning header automatically
@ControllerAdvice
public class ApiDeprecationAdvice implements ResponseBodyAdvice<Object> {
  @Override
  public Object beforeBodyWrite(...) {
    if (request.getRequestURI().contains("/api/v1/")) {
      response.addHeader("Deprecation", "version=\"v1\"");
      response.addHeader("Sunset", "Sat, 31 Dec 2025 23:59:59 GMT");
      response.addHeader("Link", "/api/v2/; rel=\"successor-version\"");
    }
    return body;
  }
}
```

---

## 🔴 ADVANCED PROJECTS (Production-Grade)

---

### Project 9 — Kubernetes with Helm Charts + Easy Upgrades

**Goal:** Package backend and frontend as Helm charts for one-command upgrades/rollbacks.

**Helm chart structure:**
```
helm/
├── backend/
│   ├── Chart.yaml           # Chart version + app version
│   ├── values.yaml          # Default config
│   ├── values-staging.yaml  # Staging overrides
│   ├── values-prod.yaml     # Prod overrides
│   └── templates/
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── ingress.yaml
│       ├── hpa.yaml         # Auto-scaling
│       └── configmap.yaml
└── frontend/
    └── ... (same structure)
```

**Chart.yaml:**
```yaml
apiVersion: v2
name: backend
description: Spring Boot Backend
type: application
version: 1.0.0          # Chart version (infrastructure)
appVersion: "2.1.3"     # App version (matches Docker image tag)
```

**values.yaml:**
```yaml
image:
  repository: myregistry/backend
  tag: ""                    # Overridden at deploy time
  pullPolicy: IfNotPresent

replicaCount: 2

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"

livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
```

**Deploy commands:**
```bash
# First deploy
helm install backend ./helm/backend \
  --namespace production \
  -f helm/backend/values-prod.yaml \
  --set image.tag=2.1.3

# Upgrade to new version
helm upgrade backend ./helm/backend \
  --namespace production \
  -f helm/backend/values-prod.yaml \
  --set image.tag=2.2.0

# Rollback to previous revision
helm rollback backend 1 --namespace production

# See revision history
helm history backend --namespace production
```

---

### Project 10 — Microservices with API Gateway + Service Discovery

**Goal:** Break monolith into services; route via Spring Cloud Gateway.

**Architecture:**
```
Internet
    │
[ Angular Frontend ]
    │
[ API Gateway :8080 ] ← Spring Cloud Gateway
    ├── /api/users/**    → User Service :8081
    ├── /api/products/** → Product Service :8082
    ├── /api/orders/**   → Order Service :8083
    └── /api/notify/**   → Notification Service :8084
         │
    [ Eureka Service Registry ]
```

**Gateway routes config:**
```java
@Bean
public RouteLocator routes(RouteLocatorBuilder builder) {
  return builder.routes()
    .route("user-service", r -> r
      .path("/api/users/**")
      .filters(f -> f
        .circuitBreaker(c -> c.setName("usersCB").setFallbackUri("forward:/fallback"))
        .retry(config -> config.setRetries(3).setStatuses(HttpStatus.SERVICE_UNAVAILABLE))
        .requestRateLimiter(rl -> rl.setRateLimiter(redisRateLimiter()))
      )
      .uri("lb://USER-SERVICE")              // lb = load-balanced via Eureka
    )
    .build();
}
```

**pom.xml versions for microservices:**
```xml
<properties>
  <spring-cloud.version>2023.0.1</spring-cloud.version>
</properties>

<dependencyManagement>
  <dependencies>
    <!-- Spring Cloud BOM — pins Eureka, Gateway, Config, etc. -->
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

---

### Project 11 — GitOps with ArgoCD

**Goal:** Git as the single source of truth for Kubernetes state; auto-sync on merge.

**ArgoCD Application manifest:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: backend-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/my-app
    targetRevision: main
    path: helm/backend
    helm:
      valueFiles:
        - values-prod.yaml
      parameters:
        - name: image.tag
          value: "2.2.0"         # Updated by CI pipeline automatically
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true                # Remove resources deleted from Git
      selfHeal: true             # Fix manual changes in cluster
    syncOptions:
      - CreateNamespace=true
```

**CI pipeline updates ArgoCD via Git:**
```yaml
# In GitHub Actions release job:
- name: Update image tag in GitOps repo
  run: |
    git clone https://github.com/myorg/gitops-config.git
    cd gitops-config
    yq e '.image.tag = "${{ env.NEW_VERSION }}"' -i helm/backend/values-prod.yaml
    git commit -am "ci: deploy backend ${{ env.NEW_VERSION }} to production"
    git push
  # ArgoCD detects the change and syncs automatically
```

---

### Project 12 — Observability Stack (Metrics + Tracing + Alerting)

**Goal:** Full visibility into production with Prometheus, Grafana, and Jaeger.

**Spring Boot — Actuator + Micrometer:**
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
```

**Custom business metrics:**
```java
@Service
public class OrderService {
  private final Counter ordersCreated;
  private final Timer orderProcessingTime;

  public OrderService(MeterRegistry registry) {
    ordersCreated = Counter.builder("orders.created.total")
      .description("Total orders created")
      .tag("env", "production")
      .register(registry);

    orderProcessingTime = Timer.builder("orders.processing.seconds")
      .description("Order processing duration")
      .register(registry);
  }

  public Order createOrder(OrderRequest request) {
    return orderProcessingTime.record(() -> {
      Order order = processOrder(request);
      ordersCreated.increment();
      return order;
    });
  }
}
```

**Grafana Alert Rule (YAML):**
```yaml
- alert: HighErrorRate
  expr: rate(http_server_requests_seconds_count{status=~"5.."}[5m]) > 0.05
  for: 2m
  annotations:
    summary: "High 5xx error rate on {{ $labels.instance }}"
    runbook_url: "https://wiki.myapp.com/runbooks/high-error-rate"
  labels:
    severity: critical
```

---

### Project 13 — Database Migration Versioning with Flyway

**Goal:** Track DB schema changes in version control. Upgrade AND rollback safely.

```xml
<!-- pom.xml -->
<dependency>
  <groupId>org.flywaydb</groupId>
  <artifactId>flyway-core</artifactId>
  <!-- version managed by Spring Boot BOM -->
</dependency>
```

**Migration files:**
```
backend/src/main/resources/db/migration/
├── V1__create_users_table.sql
├── V2__add_user_email_index.sql
├── V3__create_orders_table.sql
├── V4__add_order_status_column.sql
└── U4__undo_order_status_column.sql   # Undo script for v4 (downgrade)
```

```sql
-- V4__add_order_status_column.sql
ALTER TABLE orders ADD COLUMN status VARCHAR(20) DEFAULT 'PENDING' NOT NULL;
CREATE INDEX idx_orders_status ON orders(status);

-- U4__undo_order_status_column.sql (rollback)
DROP INDEX IF EXISTS idx_orders_status;
ALTER TABLE orders DROP COLUMN IF EXISTS status;
```

**Flyway commands:**
```bash
mvn flyway:info        # Show migration status
mvn flyway:migrate     # Run pending migrations
mvn flyway:undo        # Rollback last migration (requires Flyway Teams)
mvn flyway:repair      # Fix failed migration checksum
mvn flyway:validate    # Verify migrations match DB state
```

---

### Project 14 — Dependency Vulnerability Scanning Pipeline

**Goal:** Fail CI automatically when critical CVEs are found.

**.github/workflows/security.yml:**
```yaml
name: Security Scan

on:
  schedule:
    - cron: '0 6 * * 1'       # Every Monday 6am
  push:
    branches: [main]

jobs:
  backend-security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: OWASP Dependency Check (Maven)
        run: |
          mvn org.owasp:dependency-check-maven:check \
            -DfailBuildOnCVSS=7 \            # Fail on HIGH severity
            -Dformat=HTML,JSON \
            -DsuppressionFile=.owasp-suppression.xml
        working-directory: ./backend

      - name: Upload report
        uses: actions/upload-artifact@v4
        with:
          name: owasp-report
          path: backend/target/dependency-check-report.html

  frontend-security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
        working-directory: ./frontend
      - name: npm audit
        run: npm audit --audit-level=high
        working-directory: ./frontend
      - name: Snyk scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high
```

---

## 📋 UPGRADE/DOWNGRADE QUICK REFERENCE

| Scenario | Command |
|---|---|
| Upgrade Spring Boot | `mvn versions:set -DnewVersion=3.3.0` in pom.xml parent |
| Downgrade Spring Boot | Same command with older version |
| Upgrade Angular | `npx ng update @angular/core @angular/cli` |
| Downgrade Angular | `npm install @angular/core@16.2.0 @angular/cli@16.2.0` |
| Upgrade Java in Docker | Change `FROM eclipse-temurin:21...` to `FROM eclipse-temurin:22...` |
| Upgrade Node in Docker | Change `FROM node:20...` to `FROM node:22...` |
| Rollback Kubernetes deployment | `kubectl rollout undo deployment/backend` |
| Rollback Helm release | `helm rollback backend <revision>` |
| Rollback database migration | `mvn flyway:undo` |
| Pin npm dependency | `npm install package@1.2.3 --save-exact` |
| Pin Maven dependency | Add `<version>1.2.3</version>` explicitly in pom.xml |

---

## 🗺️ LEARNING PATH PROGRESSION

```
Week 1-2:    Projects 1-2   → Docker, multi-stage builds, profiles/environments
Week 3-4:    Project 3      → GitHub Actions CI pipeline
Week 5-6:    Projects 4-5   → Semantic versioning, blue/green deployment
Week 7-8:    Projects 6-7   → Feature flags, ELK logging
Week 9-10:   Project 8      → API versioning strategies
Week 11-12:  Project 9      → Helm charts + Kubernetes
Week 13-16:  Projects 10-11 → Microservices + ArgoCD GitOps
Week 17-20:  Projects 12-14 → Full observability + security scanning
```

---

*Stack versions pinned as of 2025: Java 21 LTS, Spring Boot 3.2.x, Angular 17.x, Node 20 LTS, TypeScript 5.4.x*
