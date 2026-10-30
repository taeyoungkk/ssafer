# SSAFER Backend

SSAFER 백엔드 애플리케이션입니다. 게스트 인증, 프로젝트 관리, 스캔 요청과 결과 조회, Local Agent 연동, 업로드 스캔 처리, Finding 상태 관리를 담당합니다.

## 기술 스택

- Java 21
- Spring Boot 4.0.5
- Spring Web MVC
- Spring Security
- Spring Data JPA
- PostgreSQL
- Redis
- RabbitMQ
- Flyway
- SpringDoc OpenAPI
- AWS SDK for S3
- JWT

빌드와 라이브러리 버전은 [pom.xml](./pom.xml)을 참고합니다. 인증에는 Spring Security·JWT·OAuth2를 사용하고, API 문서는 SpringDoc OpenAPI로 제공합니다.

## 분석 작업 추적 및 Agent 연동

| 영역 | 구현 |
| --- | --- |
| 분석 작업 추적 | `worker_jobs`에 작업 상태를 저장하고 Worker 콜백 처리에 활용합니다. |
| 재발행 | `@Scheduled` 기반 `WorkerJobRepublishScheduler`가 오래 대기한 작업을 찾아 재발행합니다. |
| 웹 상태 통지 | DONE / FAILED 상태 전이의 트랜잭션 커밋 이후 SSE 이벤트를 발행합니다. 프론트엔드는 폴링으로도 상태를 조회합니다. |
| Agent 연결 | Raw WebSocket에서 CONNECT / PING / PONG 및 `TASK_AVAILABLE` 메시지를 처리합니다. |
| 작업 조회 / 결과 보고 | WebSocket 알림과 별개로 internal Agent API를 통해 pending task를 조회하고 `patchResults`를 보고합니다. |
| 패치 추적 | 승인 시 `PATCH_APPLY` 작업을 생성하고 적용 결과, 백업 정보, 적용 시각을 기록합니다. |

## 로컬 실행 준비

### 1. `.env` 생성

`.env.sample`을 참고해서 `.env` 파일을 생성합니다. 실제 비밀값은 Git에 커밋하지 않습니다.

최소 필수 값:

```properties
JWT_SECRET=replace-with-at-least-32-byte-secret-key
WORKER_API_SECRET=replace-with-worker-internal-api-secret
AWS_ACCESS_KEY_ID=replace-with-access-key-id
AWS_SECRET_ACCESS_KEY=replace-with-secret-access-key
```

현재 `S3Client` 설정은 AWS 키가 비어 있으면 애플리케이션 컨텍스트 생성 단계에서 실패합니다. S3 API를 직접 호출하지 않는 로컬 실행이나 테스트만 확인할 때는 더미 값을 사용할 수 있지만, 실제 presigned URL 발급이나 S3 조회 기능은 권한이 있는 실제 값이 필요합니다.

OAuth, 이메일, 실제 S3 버킷을 사용하는 경우 아래 값도 설정합니다.

```properties
GOOGLE_OAUTH_CLIENT_ID=
GOOGLE_OAUTH_CLIENT_SECRET=
GITHUB_OAUTH_CLIENT_ID=
GITHUB_OAUTH_CLIENT_SECRET=
RESEND_API_KEY=
EMAIL_FROM_ADDRESS=
APP_SCAN_RAW_S3_BUCKET=
APP_SCAN_RAW_S3_REGION=ap-northeast-2
```

### 2. 로컬 인프라 실행

PostgreSQL, Redis, RabbitMQ를 Docker Compose로 실행합니다.

```powershell
docker compose up -d
```

RabbitMQ Management UI:

```text
http://localhost:15672
```

기본 계정:

```text
guest / guest
```

### 3. 애플리케이션 실행

`local` 프로필로 실행합니다.

```powershell
.\mvnw.cmd spring-boot:run "-Dspring-boot.run.profiles=local"
```

PowerShell에서는 환경 변수로 프로필을 지정할 수도 있습니다.

```powershell
$env:SPRING_PROFILES_ACTIVE="local"
.\mvnw.cmd spring-boot:run
```

기본 애플리케이션 포트는 `8080`입니다.

## 주요 접속 경로

Health check:

```text
http://localhost:8080/actuator/health
```

Swagger UI:

```text
http://localhost:8080/swagger-ui/index.html
```

OpenAPI JSON:

```text
http://localhost:8080/v3/api-docs
```

## 테스트

`.env`에 필수 값이 없다면 테스트 실행 전에 더미 환경 변수를 지정합니다.

```powershell
$env:JWT_SECRET="replace-with-at-least-32-byte-secret-key"
$env:WORKER_API_SECRET="replace-with-worker-internal-api-secret"
$env:AWS_ACCESS_KEY_ID="test-access-key"
$env:AWS_SECRET_ACCESS_KEY="test-secret-key"
```

전체 테스트 실행:

```powershell
.\mvnw.cmd test
```

애플리케이션 빌드:

```powershell
.\mvnw.cmd package
```

테스트를 제외하고 빌드:

```powershell
.\mvnw.cmd package -DskipTests
```

## Docker 이미지

개발용 Dockerfile은 `Dockerfile.dev`입니다.

```powershell
docker build -f Dockerfile.dev -t ssafer-backend:dev .
```

`Dockerfile.dev`는 애플리케이션 JAR를 빌드하고, 런타임 이미지에 Trivy 바이너리를 포함합니다.

## 주요 설정

### 데이터베이스

`local` 프로필은 아래 PostgreSQL 접속 정보를 사용합니다.

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/ssafer
spring.datasource.username=ssafer
spring.datasource.password=ssafer
```

Flyway 마이그레이션은 아래 경로에서 실행됩니다.

```text
src/main/resources/db/migration
```

### Redis

```properties
REDIS_HOST=localhost
REDIS_PORT=6379
```

이메일 인증 코드, 비밀번호 재설정 코드, refresh token 저장에 사용됩니다.

### RabbitMQ

```properties
RABBITMQ_HOST=localhost
RABBITMQ_PORT=5672
RABBITMQ_USERNAME=guest
RABBITMQ_PASSWORD=guest
```

Local Agent 작업 발행과 worker job 재발행 흐름에서 사용됩니다.

### S3

```properties
APP_SCAN_RAW_S3_BUCKET=
APP_SCAN_RAW_S3_REGION=ap-northeast-2
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
```

raw scan 결과 업로드 URL 발급, raw result 검증, analysis result 다운로드 URL 발급에 사용됩니다.

### 업로드 스캔 엔진

HTTP 스캔 엔진 연동은 기본적으로 활성화되어 있습니다.

```properties
APP_SCAN_ENGINE_ENABLED=true
APP_SCAN_ENGINE_URL=http://engine:8100
APP_SCAN_UPLOAD_SCAN_TIMEOUT_SECONDS=120
INTERNAL_TOKEN=
```

로컬 프로세스 기반 스캔을 사용하려면 아래처럼 설정합니다.

```properties
APP_SCAN_ENGINE_ENABLED=false
```

## 인증 개요

- 공개 인증 API, 게스트 진입 API, Swagger, OpenAPI 문서, health check는 JWT 인증 없이 접근할 수 있습니다.
- 일반 `/api/**` 엔드포인트는 `SecurityConfig`에서 명시적으로 허용한 경로가 아니면 JWT 인증이 필요합니다.
- `/api/v1/internal/agents/**` 경로는 Agent token 인증을 사용합니다.
- `/api/v1/internal/scans/*/analysis-results` 경로는 worker secret 인증을 사용합니다.

## 자주 쓰는 명령

```powershell
# 인프라 실행
docker compose up -d

# 인프라 종료
docker compose down

# local 프로필로 서버 실행
.\mvnw.cmd spring-boot:run "-Dspring-boot.run.profiles=local"

# 테스트 실행
.\mvnw.cmd test
```
