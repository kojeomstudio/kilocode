# Kilo CLI - 빌드, 실행 및 배포 가이드

## 사전 요구사항

| 항목 | 버전 |
|---|---|
| Bun | 1.3.13+ |
| Git | 최신 버전 |
| Node.js | (VS Code 확장 개발 시) 24+ |

Bun 설치:

```bash
curl -fsSL https://bun.sh/install | bash
```

## 1. 로컬 개발 환경 설정

### 의존성 설치

```bash
git clone <repository-url> kilocode
cd kilocode
bun install
```

### 개발 서버 실행

```bash
# TUI 모드 (기본) - packages/opencode 디렉터리에서 실행
bun dev

# 특정 디렉터리에서 실행
bun dev /path/to/project

# 리포지터리 루트에서 실행
bun dev .
```

### kilodev로 어디서든 실행

`bin/kilodev`는 현재 체크아웃을 임의 디렉터리에서 실행하는 자체 위치 확인 런처입니다.

```bash
# 한 번 설치 (쉘 설정에 alias 추가)
./bin/kilodev dev-setup

# 이후 어디서든
cd ~/some/project
kilodev                        # TUI 실행
kilodev run "task description" # 비대화형 실행
kilodev serve                  # HTTP 서버
```

CI/컨테이너 환경에서는 `--yes` 플래그로 확인을 건너뜁니다:

```bash
./bin/kilodev dev-setup --yes
```

### 로컬 백엔드로 테스트

로컬 Kilo API 서버를 사용하려면:

```bash
KILO_API_URL=http://localhost:3000 bun dev
```

## 2. CLI 명령어

### 주요 명령어

| 명령어 | 설명 |
|---|---|
| `kilo` | 대화형 터미널 UI (TUI) |
| `kilo run "<task>"` | 비대화형 작업 실행 (CI/CD용) |
| `kilo serve` | 헤드리스 HTTP API 서버 |
| `kilo web` | 서버 + 브라우저 UI |
| `kilo models` | 사용 가능한 모델 목록 |
| `kilo config` | 설정 관리 |
| `kilo session` | 세션 관리 (목록, 삭제) |
| `kilo export` | 세션 내보내기 |
| `kilo import` | 세션 가져오기 |
| `kilo auth` | 인증 관리 |
| `kilo providers` | 프로바이더 관리 |
| `kilo agent` | 에이전트 관리 |
| `kilo mcp` | MCP 서버 관리 |
| `kilo debug` | 디버그 진단 |

### `kilo run` (Auto 모드) - CI/CD 파이프라인용

```bash
# 기본 사용
kilo run "Fix the typo in README.md"

# 특정 에이전트 지정
kilo run --agent code "Refactor the auth module"

# 특정 모델 지정
kilo run --model anthropic/claude-sonnet-4-20250514 "Add unit tests"

# 특정 디렉터리에서 실행
kilo run --dir /path/to/project "Implement feature X"

# 파일 첨부
kilo run --file screenshot.png "Fix the layout issue shown in this image"

# JSON 출력 (파이프라인용)
kilo run --format json "List all TODO comments"

# 권한 모두 자동 승인 (완전 자율 모드)
kilo run --auto "Refactor the codebase"

# 기존 세션 계속
kilo run --continue "Continue the refactoring"

# 커스텀 설정
kilo run --dangerously-skip-permissions "Deploy changes"
```

`kilo run`은 터미널 환경이나 CI/CD 파이프라인에서 사용하기 적합합니다. `--auto` 플래그로 권한을 자동 승인하고, `--format json`으로 구조화된 출력을 얻을 수 있습니다.

### `kilo serve` - 헤드리스 HTTP API 서버

```bash
# 기본 실행
kilo serve

# 비밀번호 설정
KILO_SERVER_PASSWORD=secret kilo serve

# 특정 포트
kilo serve --port 8080
```

SDK를 통해 REST API로 통신합니다. VS Code 확장과 같은 클라이언트가 `kilo serve` 프로세스를 자식으로 실행하여 HTTP + SSE로 통신합니다.

### 환경 변수

| 변수 | 기본값 | 설명 |
|---|---|---|
| `KILO_API_URL` | `https://api.kilo.ai` | Kilo API (gateway, auth, models) |
| `KILO_SESSION_INGEST_URL` | `https://ingest.kilosessions.ai` | 세션 내보내기 / 클라우드 동기화 |
| `KILO_MODELS_URL` | `https://models.dev` | 모델 메타데이터 |
| `KILO_SERVER_PASSWORD` | (없음) | `kilo serve` Basic 인증 |
| `KILO_DB` | (파일시스템) | `:memory:` 설정 시 휘발성 DB |
| `KILO_TELEMETRY_LEVEL` | (활성) | `off`로 원격 분석 비활성화 |
| `KILO_CONFIG_CONTENT` | (없음) | 인라인 JSON 설정 |

## 3. 프로덕션 바이너리 빌드

### 단일 플랫폼 빌드

```bash
cd packages/opencode
bun run script/build.ts --single
```

빌드 결과물:

```
dist/@kilocode/cli-<platform>/bin/kilo
```

지원 플랫폼: `darwin-arm64`, `darwin-x64`, `linux-arm64`, `linux-x64`, `win32-arm64`, `win32-x64`

### 전체 플랫폼 빌드 (릴리스용)

```bash
cd packages/opencode
bun run script/build.ts
```

12개 타겟 빌드 (linux/darwin/win32 × arm64/x64 + musl/baseline 변형).

### 빌드 플래그

| 플래그 | 설명 |
|---|---|
| `--single` | 현재 플랫폼만 빌드 |
| `--baseline` | 베이스라인 (비 AVX2) 바이너리 |
| `--skip-install` | 의존성 설치 생략 |
| `--skip-embed-web-ui` | 웹 UI 임베드 생략 |

## 4. Docker

### Docker 이미지 빌드

```bash
# CLI 바이너리를 먼저 빌드
cd packages/opencode
bun run script/build.ts

# Docker 이미지 빌드 (linux/amd64 + linux/arm64)
docker build -t kilocode .

# 실행
docker run --rm -it kilocode run "Fix the bug"
```

### GitHub Container Registry

공식 이미지: `ghcr.io/kilo-org/kilocode`

```bash
docker pull ghcr.io/kilo-org/kilocode:latest
docker run --rm -it ghcr.io/kilo-org/kilocode:latest run "task"
```

## 5. VS Code 확장 개발

### 개발 모드로 빌드 및 실행

```bash
# 루트에서
bun run extension
```

옵션:
- `--no-build` — 빌드 스킵 (이미 빌드된 경우)
- `--workspace PATH` — 특정 폴더 열기
- `--insiders` — VS Code Insiders 사용
- `--clean` — 캐시된 상태 초기화

### 확장 패키징 (.vsix)

```bash
cd packages/kilo-vscode
bun run package
```

### 확장 테스트

```bash
cd packages/kilo-vscode
bun run test:unit
```

## 6. 테스트

```bash
# CLI 테스트 (packages/opencode에서)
cd packages/opencode
bun test                           # 전체 테스트
bun test ./test/session/prompt.test.ts  # 특정 파일

# 타입체크 (루트에서)
bun run typecheck

# 린트
bun run lint
```

> 루트에서 `bun test`를 실행하지 마세요. 루트는 테스트를 차단합니다.

## 7. 프로덕션 설치

### npm

```bash
npm install -g @kilocode/cli
```

### Homebrew

```bash
brew install Kilo-Org/tap/kilo
```

### GitHub Releases

https://github.com/Kilo-Org/kilocode/releases 에서 플랫폼별 바이너리 다운로드.

### 설치 스크립트

```bash
curl -fsSL https://kilo.ai/cli/install | bash
```

### AUR (Arch Linux)

```bash
pacman -S kilo-bin
```

## 8. CI/CD 파이프라인 예시

### GitHub Actions

```yaml
name: AI Code Review
on:
  pull_request:
    branches: [main]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm install -g @kilocode/cli
      - run: kilo run --format json --agent code "Review this PR for bugs and suggest fixes"
```

### GitLab CI

```yaml
review:
  stage: review
  image: ghcr.io/kilo-org/kilocode:latest
  script:
    - kilo run --format json "Review the latest commit for issues"
```

### Docker Compose

```yaml
services:
  kilo:
    image: ghcr.io/kilo-org/kilocode:latest
    volumes:
      - .:/workspace
    working_dir: /workspace
    command: run --auto "Run linting and fix any issues"
```
