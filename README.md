# SSAfer

> **당신의 코드가 세상에 안전하게 닿도록.**

SSAfer는 Docker·환경 변수·서버 설정의 보안 문제를 점검하고, AI로 원인 설명과 수정안을 제공하는 **Web + CLI 보안 점검 도구**입니다. 사용자가 수정안을 승인하면 Local Agent가 파일 변경 여부를 확인하고 백업한 뒤 패치를 적용합니다.

| 항목 | 내용 |
| --- | --- |
| 개발 기간 | 2026.04.06 ~ 2026.05.22 (7주) |
| 개발 인원 | 6명 |
| 플랫폼 | Web + CLI |
| 기관 | 삼성 청년 SW·AI 아카데미 14기 |

## 목차

- [주요 기능](#주요-기능)
- [기술 스택](#기술-스택)
- [시스템 아키텍처](#시스템-아키텍처)
- [핵심 구현](#핵심-구현)
- [팀 구성 및 역할](#팀-구성-및-역할)
- [시작하기](#시작하기)
- [관련 문서](#관련-문서)
- [라이선스](#라이선스)

## 주요 기능

| 기능 | 설명 |
| --- | --- |
| 웹 파일 점검 | Docker Compose·환경 변수 파일 등을 업로드해 점검 결과와 AI 설명을 확인합니다. |
| CLI 프로젝트 점검 | 로컬 프로젝트의 지원 파일을 스캔하고 결과를 웹 대시보드로 업로드합니다. |
| 수정안 검토 및 적용 | AI가 제안한 수정안을 검토하고, 웹 승인과 Local Agent 또는 CLI를 통해 파일에 적용합니다. |
| 서버 런타임 점검 | 서버의 열린 포트, SSH 설정, 방화벽, OS 패키지 취약점 등을 점검합니다. |
| 점검 이력 비교 | 이전 결과와 비교해 신규·해결·유지 항목과 심각도 변화를 확인합니다. |
| Guest 체험 | 회원가입 없이 Guest 토큰으로 웹과 CLI를 연결해 기능을 체험합니다. |

## 기술 스택

| 분야 | 주요 기술 |
| --- | --- |
| Frontend | TypeScript, React, Vite, Tailwind CSS, Zustand |
| Backend | Java 21, Spring Boot, Spring Security, Spring Data JPA |
| AI | Python, FastAPI, LangChain, LangGraph |
| CLI / Engine | Python, Typer, Rich, Trivy, Custom Rule Engine |
| 데이터 / 메시징 | PostgreSQL, Redis, RabbitMQ, AWS S3 |
| 인프라 | AWS EC2, Docker Compose, Nginx, Jenkins |

세부 라이브러리와 설정은 아래 [관련 문서](#관련-문서)에서 확인할 수 있습니다.

## 시스템 아키텍처

서비스 서버와 AI 분석 서버를 분리하고, RabbitMQ로 분석 작업을 전달합니다.

```text
[점검 입력]
웹 → Nginx → Spring Boot → Engine (Trivy + Custom Rule) → S3
CLI → Trivy + Custom Rule → 결과 마스킹 → S3 → Spring Boot

[AI 분석]
Spring Boot → RabbitMQ → AI Worker → FastAPI
  → 근거 수집 Agent (설정 및 탐지 항목 조건 충족 시)
  → 설명·수정안 생성 → 검증 (설정 시) → S3
  → Spring Boot 결과 콜백 → SSE / 폴링 → 웹 결과 조회

[웹에서 패치 적용]
사용자 승인 → Spring Boot → WebSocket 작업 알림 → Local Agent
  → 작업 조회 → 파일 해시 검증 → 백업·파일 수정 → 적용 결과 보고
```

운영 서버 구성과 네트워크·포트 정책은 [인프라 문서](./Infra/README.md)에 정리되어 있습니다.

## 핵심 구현

### 룰 기반 탐지와 AI 설명 분리

Trivy와 자체 룰 엔진이 점검 항목을 생성하고, AI는 해당 결과의 설명과 수정안 생성을 담당합니다. 웹 Engine도 CLI의 룰 엔진을 재사용해 두 경로에서 점검 로직을 공유합니다.

CLI는 업로드할 결과에 마스킹 규칙을 적용합니다. 이는 규칙이 인식하는 민감정보의 노출을 줄이기 위한 처리이며, 웹 파일 업로드와는 입력 경로가 다릅니다.

### 비동기 분석과 작업 상태 추적

분석 작업을 RabbitMQ와 AI Worker로 넘겨 웹 요청 처리와 분리합니다. 백엔드는 `worker_jobs`에 작업 상태를 저장하고, 재발행 스케줄러로 오래 대기한 작업을 다시 전달합니다. Worker는 오류 유형에 따라 재시도 여부를 구분합니다.

웹에는 DB 트랜잭션 커밋 이후 SSE로 상태 변화를 알리고, 주기적인 폴링으로도 진행 상태를 확인합니다.

### 조건부 근거 수집과 AI 응답 검증

`AGENT_ENABLED`가 켜져 있고 CVE가 있거나 심각도가 HIGH·CRITICAL인 항목에 대해 Agent가 CVE·웹·코드 정보를 수집합니다. 수집한 정보를 설명·수정안 생성의 입력에 추가하고, `VERIFY_ENABLED`가 켜져 있으면 수정안 검증 단계도 수행합니다. 두 옵션은 코드 기본값이 비활성입니다.

Agent 대상이 아닌 항목의 배치 처리, LLM 호출 재시도, 출력 스키마 검증을 적용합니다. LLM 제공자는 `LLM_PROVIDER` 설정으로 Ollama·GMS·Anthropic 중 선택합니다.

### 승인과 파일 검증을 거치는 패치 적용

CLI가 원본 텍스트와 파일 해시를 전달하고, AI는 교체할 텍스트를 제안합니다. Local Agent는 사용자 승인과 파일 해시를 확인하고 백업을 생성한 뒤 적용합니다.

WebSocket은 작업 도착 알림을 담당하며, 실제 작업 조회와 적용 결과 보고는 별도 API로 처리합니다. 백엔드는 성공·실패, 백업 정보, 적용 시각을 기록합니다.

## 팀 구성 및 역할

| 이름 | 담당 | 주요 기여 |
| --- | --- | --- |
| 유동훈 (팀장) | Infrastructure | 프로젝트 일정 관리, 서버 구성·접근 제어, 배포·운영 자동화 |
| 김건남 | Backend | 스캔·결과 API, 분석 작업 발행·상태 통지, 인증·인가 |
| 김태영 | Backend | Local Agent 연동·패치 승인 API, 히스토리 비교, JPA 설계 |
| 양혜지 | Full Stack | 화면·상태 관리 및 공통 UI, 실시간 진행 화면, ERD·인증 설계 참여 |
| 김승수 | CLI | CLI 명령어·배포, 룰 엔진·마스킹, Local Agent 구현 |
| 김은수 | AI | AI 분석 파이프라인, Agent·도구 호출, 비동기 분석 Worker |

## 시작하기

### CLI 사용

Python 3.10 이상이 필요합니다. 계정이 없다면 `ssafer signup`으로 가입하거나 [CLI 문서](./CLI/README.md)의 Guest 사용 방법을 확인합니다.

```bash
pip install ssafer
ssafer tools
ssafer login
ssafer run --upload --path ./my-project
```

수정안을 확인한 뒤 적용하려면 점검한 프로젝트 디렉터리에서 `ssafer apply`를 실행합니다. 웹에서 작업을 전달받으려면 해당 디렉터리에서 `ssafer agent`를 실행합니다.

### 로컬 개발

Java 21, Node.js, Python, Docker 및 Docker Compose를 준비합니다. AI의 운영 컨테이너는 Python 3.11을 사용하며, CLI와 Engine은 Python 3.10 이상을 지원합니다. 프론트엔드 런타임은 설치된 Vite 버전의 요구사항을 따릅니다.

먼저 [백엔드 환경 설정](./Backend/ssafer/README.md)과 [AI 환경 설정](./AI/README.md)에 따라 환경 변수를 준비하고 PostgreSQL·Redis·RabbitMQ 및 필요한 외부 서비스 연결을 설정합니다. 아래 명령은 각 터미널에서 **저장소 루트를 기준으로** 실행합니다.

**Backend**

```bash
cd Backend/ssafer
./mvnw spring-boot:run -Dspring-boot.run.profiles=local
```

Windows PowerShell에서는 `./mvnw` 대신 `.\mvnw.cmd`를 사용합니다.

**AI Server / Worker** — 같은 Python 환경에서 별도 프로세스로 실행합니다.

```bash
cd AI
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000
```

```bash
cd AI
python -m app.worker.async_consumer
```

**Engine** — 웹 업로드 점검에 사용하며 CLI 패키지도 함께 설치합니다. 백엔드 연결 설정은 [Backend 문서](./Backend/ssafer/README.md), 컨테이너 구성은 [Infra 문서](./Infra/README.md)를 참고합니다.

```bash
pip install -e ./CLI
pip install -e ./Engine
cd Engine
uvicorn engine.app:app --port 8100
```

**Frontend**

```bash
cd Frontend
npm install
npm run dev
```

## 관련 문서

| 문서 | 내용 |
| --- | --- |
| [Frontend](./Frontend/README.md) | 개발 명령어, Storybook |
| [Backend](./Backend/ssafer/README.md) | 환경 변수, 로컬 인프라, 인증, 작업 추적 및 Agent 연동 |
| [AI](./AI/README.md) | 분석 파이프라인, LLM 설정, API, 테스트 |
| [CLI](./CLI/README.md) | 설치, 점검·업로드, Guest, Agent, 패치 적용 |
| [Infra](./Infra/README.md) | 서버 구성, 네트워크, CI/CD, 배포 |

## 라이선스

이 프로젝트는 [MIT License](./LICENSE)를 따릅니다.
