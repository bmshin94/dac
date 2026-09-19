# DAC 전수조사 & 활용 전략 정리 (한국어)

> 작성일: 2026-09-19
> 대상 저장소(포크): https://github.com/bmshin94/dac
> 원본(업스트림): https://github.com/bruin-data/dac
> 공식 문서: https://getbruin.com/docs/dac/
> 제품 소개: https://getbruin.com/dashboards-as-code/

---

## 목차

1. [DAC란 무엇인가](#1-dac란-무엇인가)
2. [저장소 전수조사 결과](#2-저장소-전수조사-결과)
3. [동작 원리](#3-동작-원리)
4. [핵심 기능 3가지](#4-핵심-기능-3가지)
5. [AI 에이전트 친화 설계](#5-ai-에이전트-친화-설계)
6. [설치 및 사용법](#6-설치-및-사용법)
7. [플러그인 / 스킬 / MCP 구분](#7-플러그인--스킬--mcp-구분)
8. [API 토큰 필요 여부](#8-api-토큰-필요-여부)
9. [GitHub에서 주목받는 이유](#9-github에서-주목받는-이유)
10. [로컬 에이전트 구축 활용법](#10-로컬-에이전트-구축-활용법)
11. [React / PHP 재구현 가능성](#11-react--php-재구현-가능성)
12. [수익화 아이디어](#12-수익화-아이디어)
13. [실행 로드맵](#13-실행-로드맵)
14. [주의사항 요약](#14-주의사항-요약)

---

## 1. DAC란 무엇인가

**DAC = Dashboard-as-Code.** 대시보드를 GUI에서 클릭으로 만드는 대신, YAML 또는 TSX 파일로
정의해서 Git으로 버전 관리하는 도구다.

- 제작: Bruin Data (`bruin-data/dac`)
- 언어: Go 1.25 (백엔드/CLI) + React 19 (프론트엔드, 바이너리에 embed)
- 라이선스: **AGPL-3.0-only**
- 현재 버전대: v0.18.x (stable) / edge 채널 병행
- GitHub 스타: 약 711~739 (2026-09 기준)

### GUI 기반 BI 툴의 5가지 문제와 DAC의 해결

| 문제 | DAC의 해결 |
|---|---|
| 누가 언제 바꿨는지 추적 불가 | 파일이므로 Git 히스토리에 전부 기록 |
| 롤백 불가 | `git revert` 한 줄 |
| 복제/재사용 어려움 | 파일 복사 |
| 사전 리뷰 불가 | Pull Request 리뷰 |
| AI가 만들 수 없음 | AI는 텍스트 생성에 강함 → 자동 생성 가능 |

---

## 2. 저장소 전수조사 결과

### 최상위 구조

```
.
├── main.go / embed.go      진입점, 프론트엔드 embed
├── cmd/                    CLI 명령어 13개
├── pkg/                    핵심 로직 (11개 패키지)
├── frontend/               React 19 + Vite + Tailwind v4
├── docs/                   VitePress 문서 (27개 md)
├── examples/               예제 프로젝트 5개
├── schemas/                JSON Schema (에디터 자동완성)
├── testdata/               테스트 픽스처
├── install.sh              설치 스크립트
├── Makefile                빌드 진입점
└── .claude/skills/         AI 에이전트용 스킬
```

### pkg/ 패키지별 역할

| 패키지 | 역할 | 주요 파일 |
|---|---|---|
| `pkg/dashboard/` | 핵심 두뇌. YAML/TSX 파싱, 검증, 시맨틱 SQL 생성 | `loader.go`, `jsloader.go`, `validator.go`, `semantic_query.go`, `model.go` |
| `pkg/query/` | 쿼리 실행. `bruin query` CLI를 shell out | `bruin.go`, `cache.go`, `backend.go` |
| `pkg/server/` | HTTP 서버 + REST API + SSE 핫리로드 | `server.go`, `api.go`, `watcher.go`, `admin.go` |
| `pkg/config/` | `.bruin.yml` 파싱, 상위 디렉토리 탐색 | `config.go` |
| `pkg/render/` | 정적 빌드. 쿼리 결과를 HTML에 베이크 | `render.go` |
| `pkg/slides/` | Google Slides 내보내기 (OAuth) | `auth.go`, `charts.go`, `export.go` |
| `pkg/importer/metabase/` | Metabase 대시보드 → DAC YAML 변환 | `convert.go`, `semantic.go` |
| `pkg/theme/` | 테마 레지스트리 (`bruin`, `bruin-dark`, 커스텀 YAML) | `registry.go`, `theme.go` |
| `pkg/template/` | Jinja 템플릿 처리 (gonja) | `template.go` |
| `pkg/telemetry/` | 익명 사용 통계 (RudderStack) | - |
| `pkg/update/` | 업데이트 체크 & 넛지 | - |

### 주요 의존성

| 라이브러리 | 용도 |
|---|---|
| `urfave/cli/v3` | CLI 프레임워크 |
| `dop251/goja` | Go 내장 JS 엔진 (TSX 실행) |
| `evanw/esbuild` | TSX → JS 트랜스파일 |
| `nikolalohinski/gonja/v2` | Jinja2 템플릿 엔진 |
| `bruin-data/bruin/semantic-engine` | 시맨틱 레이어 SQL 생성 |
| `santhosh-tekuri/jsonschema/v6` | JSON Schema 검증 |
| `fsnotify/fsnotify` | 파일 감시 (핫리로드) |
| `google.golang.org/api` | Google Slides/Drive API |
| `rudderlabs/analytics-go/v4` | 텔레메트리 |

### 프론트엔드 스택

React 19.2 / Vite 8 / Tailwind CSS v4 / TanStack Query 5 /
Recharts 3 / Vega + Vega-Lite 6 / react-router 7 / react-select /
react-day-picker / react-markdown / shiki / jspdf / html-to-image

### REST API 엔드포인트

```
GET  /api/v1/dashboards
GET  /api/v1/dashboards/{name}
GET  /api/v1/dashboards/{name}/raw
POST /api/v1/dashboards/{name}/data
POST /api/v1/dashboards/{name}/stream
POST /api/v1/dashboards/{name}/widgets/{widgetId}/query
POST /api/v1/query
GET  /api/v1/themes
GET  /api/v1/themes/{name}
GET  /api/v1/config
GET  /api/v1/events                      (SSE — 핫리로드)

# --password 지정 시에만 활성화
POST   /api/v1/admin/login
GET    /api/v1/admin/connections
POST   /api/v1/admin/connections
PUT    /api/v1/admin/connections/{type}/{name}
DELETE /api/v1/admin/connections/{type}/{name}
POST   /api/v1/admin/connections/{type}/{name}/test
```

---

## 3. 동작 원리

```
  sales.yml  ──┐
               ├─► [DAC Go 바이너리] ──► [bruin query CLI] ──► DB
  sales.tsx  ──┘         │                                     │
                         │  ◄────────── JSON 결과 ────────────┘
                         ▼
              [임베디드 React SPA] ──► localhost:8321
```

1. `dashboards/` 폴더의 `.yml` / `.dashboard.tsx` 파일을 로드
2. TSX는 esbuild로 트랜스파일 → goja에서 실행 → JSON 위젯 트리 추출
3. SQL은 gonja로 Jinja 템플릿 처리 (필터 값 주입)
4. `bruin query --output json`을 exec으로 호출해 DB 결과 수신
5. embed된 React SPA가 Recharts / Vega-Lite로 렌더링
6. fsnotify가 파일 변경 감지 → SSE로 브라우저에 알림 → 자동 리로드

**중요**: DAC 자체에는 DB 드라이버가 없다. `bruin` CLI가 필수 의존성이며,
Postgres / MySQL / Snowflake / BigQuery / Redshift / Databricks / DuckDB 등을 지원한다.

### 위젯 타입

`metric`, `chart`, `table`, `pivot_table`, `text`, `divider`, `image`

차트 종류(17종+): bar, line, area, pie, funnel, sankey, heatmap, treemap,
calendar, gauge, histogram, forest, combo, scatter 등 + **Vega-Lite spec 직접 삽입 가능**

---

## 4. 핵심 기능 3가지

### 4.1 Jinja 템플릿 필터

```yaml
filters:
  - name: region
    type: select
    options: { values: ["All", "North America", "Europe", "APAC"] }
  - name: date_range
    type: date-range
    default: all_time
```

```sql
SELECT SUM(amount) as value FROM sales
WHERE created_at >= '{{ filters.date_range.start }}'
  AND created_at <= '{{ filters.date_range.end }}'
{% if filters.region != 'All' %}
  AND region = '{{ filters.region }}'
{% endif %}
```

필터 타입: `select`, `multiselect`, `date-range`, `number`, `search`

### 4.2 시맨틱 레이어 (지표 용어사전)

`semantic/sales.yml`:

```yaml
name: sales
source:
  table: sales
dimensions:
  - name: created_at
    type: time
    granularities:
      day: date_trunc('day', created_at)
      month: date_trunc('month', created_at)
  - name: region
    type: string
metrics:
  - name: revenue
    expression: sum(amount)
  - name: sales_count
    expression: count(*)
  - name: avg_sale_value
    expression: "{revenue} / {sales_count}"     # 지표 간 조합
  - name: online_revenue
    expression: sum(amount)
    filter: "channel = 'online'"
segments:
  - name: online
    filter: "channel = 'online'"
```

위젯에서는 SQL 없이 참조만:

```yaml
- name: Revenue by Region
  type: chart
  model: sales
  metrics: [revenue]
  dimensions: [region]
```

→ 지표 정의가 조직 전체에서 단일화되고, 정의 변경 시 한 곳만 수정하면 된다.

### 4.3 TSX 동적 대시보드

```tsx
// 로드 타임에 DB를 조회해 대시보드 구조 자체를 생성
const regions = query("local_duckdb", "SELECT DISTINCT region FROM sales ORDER BY 1")

function KPI({ name, sql, format = ",.0f", ...rest }) {
  return <Metric name={name} sql={sql} value={{ field: "value", type: "number", format }} {...rest} />
}

export default (
  <Dashboard name="Sales (TSX)" connection="local_duckdb">
    <Filter name="date_range" type="date-range" default="last_90_days" />
    <Filter name="region" type="select" default="All"
            options={{ values: ["All", ...regions.rows.map(r => r[0])] }} />
    <Row>
      {regions.rows.map(([region]) => (
        <KPI name={region} format="$,.0f"
             col={Math.floor(12 / regions.rows.length)}
             sql={`SELECT SUM(amount) as value FROM sales WHERE region = '${region}'`} />
      ))}
    </Row>
  </Dashboard>
)
```

DB에 새 지역이 추가되면 대시보드에 카드가 자동으로 생긴다. YAML로는 불가능한 기능.

**제약**: 진짜 React가 아니라 goja(JS 엔진) 위에서 빌드타임에 1회 실행되는 제한된 JS다.
npm 패키지 사용 불가, hooks 사용 불가. "JSX 문법을 빌린 설정 파일"에 가깝다.

---

## 5. AI 에이전트 친화 설계

공식 문서 인용:

> "It is built for **AI agents** to build dashboards in a reliable and reviewable way."
>
> "DAC is meant to be a tool to be used heavily by AI agents, such as
> **Claude Code, Codex, or OpenCode**."

`dac init` 또는 `dac skills install` 실행 시 자동 설치:

```
.claude/skills/create-dashboard/SKILL.md   # Claude Code용
.codex/skills/create-dashboard             # Codex용
```

### 왜 AI에게 유리한가

| 항목 | 기존 GUI BI 툴 | DAC |
|---|---|---|
| 입력 방식 | 마우스 클릭 | 텍스트 파일 |
| AI 생성 가능성 | 불가능 | 가능 |
| 셀프 검증 | 없음 | `dac validate`, `dac check` |
| 사용법 전달 | 수동 프롬프트 | `SKILL.md` 자동 주입 |
| 결과물 리뷰 | 불가 | PR diff |

**핵심**: AI가 자기 작업물을 스스로 채점하고 반복 수정할 수 있는 피드백 루프가 내장돼 있다.

---

## 6. 설치 및 사용법

### 설치

```bash
# 안정 버전 (bruin CLI도 자동 설치)
curl -LsSf https://getbruin.com/install/dac | sh

# 최신 edge 빌드
curl -LsSf https://getbruin.com/install/dac | sh -s -- --channel edge

# 설치 위치 지정
curl -LsSf https://getbruin.com/install/dac | sh -s -- -b /usr/local/bin
```

- 기본 설치 경로: `~/.local/bin`
- `DAC_SKIP_BRUIN_INSTALL=1` → bruin 설치 생략
- 릴리즈 바이너리: macOS/Linux/Windows × amd64/arm64

### 소스 빌드 (이 저장소 기준)

```bash
make deps      # Go 모듈 + npm 패키지
make build     # 프론트 빌드 → embed → Go 바이너리
make test      # 전체 테스트
make dev       # 프론트 + 백엔드 라이브 리로드
```

> 프로젝트 규칙: `go build` / `npm run build`를 직접 호출하지 말고 **반드시 `make`** 를 사용할 것.
> (프론트엔드 embedding 순서와 CGO 플래그를 Makefile이 관리한다.)

### 기본 워크플로우

```bash
dac init my-dashboards            # 스캐폴딩 + 예제 + AI 스킬 설치
cd my-dashboards
dac validate --dir .              # 구조 검증 (DB 불필요, 빠름)
dac validate --dir . --with-database   # SQL dry-run까지
dac check --dir .                 # 실제 쿼리 실행 검증
dac serve --dir . --open          # localhost:8321, 저장 시 자동 리로드
```

### CLI 명령어 전체

| 명령어 | 설명 |
|---|---|
| `dac init [name]` | 프로젝트 스캐폴딩 + AI 스킬 설치 |
| `dac serve` | 개발 서버 (`-p` 포트, `-t` 테마, `--open`, `--password`) |
| `dac build` | 정적 HTML 생성 (쿼리 결과 베이크) |
| `dac validate` | 구조 검증 (`--with-database` 옵션) |
| `dac check` | 실제 쿼리 실행 검증 |
| `dac query` | 특정 위젯의 SQL/결과 확인 (디버깅) |
| `dac ls` | 대시보드/위젯 목록 |
| `dac connections` | 커넥션 목록/테스트 |
| `dac skills install` | 기존 프로젝트에 AI 스킬 설치 |
| `dac import metabase` | Metabase → DAC YAML 변환 |
| `dac export slides` | Google Slides 내보내기 |
| `dac upgrade` | 자체 업데이트 |
| `dac version` | 버전 확인 |

### 프로젝트 레이아웃

```
my-dashboards/
├── .bruin.yml                  # DB 커넥션 정의
├── dashboards/
│   ├── sales.yml               # YAML 대시보드
│   ├── sales.dashboard.tsx     # TSX 대시보드 (.dashboard.tsx 확장자 필수)
│   └── queries/*.sql
├── semantic/sales.yml          # 시맨틱 모델
├── themes/brand.yml            # 커스텀 테마
└── .claude/skills/create-dashboard/SKILL.md
```

### DB 커넥션 예시

```yaml
# .bruin.yml
default_environment: default
environments:
  default:
    connections:
      duckdb:
        - name: local_duckdb
          path: ./data/test.db
          read_only: true
      postgres:
        - name: warehouse
          host: db.example.com
          username: analyst
          password: ${POSTGRES_PASSWORD}
```

`dac serve -e prod` 로 환경 전환.

### 예제 프로젝트

| 예제 | 내용 |
|---|---|
| `examples/basic-yaml` | 필터 + SQL + Vega-Lite 레이어 차트 |
| `examples/basic-tsx` | 로드타임 쿼리로 레이아웃 생성 |
| `examples/semantic-yaml` | 시맨틱 모델 기반 YAML |
| `examples/semantic-tsx` | 시맨틱 모델 기반 TSX |
| `examples/semantic-joins` | 모델 간 조인 |

---

## 7. 플러그인 / 스킬 / MCP 구분

**결론: DAC는 독립 실행형 CLI 도구다. 플러그인도, 스킬도, MCP 서버도 아니다.**

| 구분 | 해당 여부 | 설명 |
|---|---|---|
| 독립 CLI 도구 | **O (정답)** | Go 단일 바이너리. `git`, `docker` 같은 터미널 명령어 |
| 스킬(Skill) | 포함 관계 | DAC가 스킬을 **배포**한다. DAC 자체가 스킬은 아님 |
| 플러그인 | X | 호스트 앱 확장이 아님. 반대로 `bruin`을 호출하는 쪽 |
| MCP 서버 | **X** | 코드베이스에 MCP 관련 구현이 전혀 없음 (go.mod에도 없음) |

### 관계도

```
사람 또는 AI 에이전트 (Claude Code)
        │  ← SKILL.md 를 읽고
        ▼
   dac (CLI, Go 단일 바이너리)
        │  ← exec (shell out)
        ▼
   bruin (CLI, 별도 설치)
        ▼
   Postgres / BigQuery / Snowflake / DuckDB ...
```

### MCP 서버로 감싸는 것은 가능

```
mcp__dac__list_dashboards      → dac ls
mcp__dac__validate             → dac validate
mcp__dac__run_widget_query     → dac query
mcp__dac__build_static         → dac build
```

→ 아직 아무도 안 만든 영역. 선점 기회가 있다.

---

## 8. API 토큰 필요 여부

**결론: 핵심 기능은 토큰이 전혀 필요 없다.**

| 기능 | 토큰 | 상세 |
|---|---|---|
| `init` / `serve` / `build` / `validate` / `check` / `query` / `ls` | 불필요 | 메인 기능 전부 무인증 |
| DB 접속 | DB 자격증명 | API 토큰 아님. `.bruin.yml` (`${ENV_VAR}` 지원) |
| `import metabase` (라이브) | 필요 | `METABASE_API_KEY` 또는 `METABASE_SESSION_TOKEN`. JSON 파일 입력(`-i`) 시 불필요 |
| `export slides` | 필요 | Google OAuth (gcloud ADC 또는 `~/.dac/credentials.json`) |
| `serve --password` | 선택 | 미지정 시 admin 엔드포인트 자체가 비활성화 |
| `upgrade` | 불필요 | GitHub 공개 릴리즈 API |
| 텔레메트리 | 불필요 | 빌드 시 주입. `make build`는 키 없이 빌드 → 전송 없음 |

### 보안 주의사항

1. `.bruin.yml`에 평문 비밀번호 금지 → `${DB_PASSWORD}` 환경변수 사용, `.gitignore` 등록
2. `--password`는 평문 문자열 비교(`pkg/server/admin.go`) — 해싱 없음.
   내부망 전용 또는 리버스 프록시 뒤에 배치할 것
3. 텔레메트리 비활성화: `export DO_NOT_TRACK=1` 또는 `TELEMETRY_OPTOUT=1`

---

## 9. GitHub에서 주목받는 이유

현재 약 **711~739 stars** (2026-09 기준). 메가히트는 아니지만 빠르게 성장 중.

1. **타이밍** — "AI 에이전트가 쓸 도구"가 가장 핫한 주제. DAC는 태생이 AI-native
2. **"dbt for dashboards"** — dbt가 데이터 변환을 코드화해 성공했듯, 대시보드 계층의 빈자리를 채움
3. **단일 바이너리** — Metabase(Java+DB+Docker), Superset(Python+Redis+Celery) 대비 압도적으로 쉬운 설치
4. **AGPL 오픈소스 + Metabase 탈출 서사** — `dac import metabase`가 곧 마케팅 메시지
5. **최신 기술 스택** — Go 1.25 / React 19 / Tailwind v4 / Vite 8 / goja / esbuild
6. **TSX 동적 대시보드** — "DB 쿼리로 레이아웃을 생성"은 경쟁 제품에 없는 차별점
7. **Bruin이라는 모회사 신뢰도** — 개인 토이 프로젝트가 아니라는 안정감

### 한계

- `bruin` CLI 의존 → 생태계 락인
- AGPL → 기업 도입 시 법무 검토 필요
- 비개발자는 사용 불가 (YAML 진입장벽)
- v0.x — API 파괴적 변경 가능성

---

## 10. 로컬 에이전트 구축 활용법

### A. 설계 교과서로서 — AI 친화 도구 5원칙

| 원칙 | DAC 구현 | 적용 방법 |
|---|---|---|
| 1. 선언적 텍스트 포맷 | YAML / TSX | 설정을 GUI가 아닌 파일로 노출 |
| 2. **셀프 검증 루프** | `dac validate` / `check` | **가장 중요.** AI가 결과물을 스스로 채점할 명령어 제공 |
| 3. 기계 판독 스펙 | `schemas/` JSON Schema | AI가 필드를 추측하지 않고 확인 |
| 4. 사용설명서 번들 | `SKILL.md` 자동 설치 | 도구가 자기 사용법을 AI에 주입 |
| 5. 구조화된 에러 | 위치 + 원인 + 해결책 | 모호한 에러는 AI를 헤매게 함 |

> 핵심 인사이트: **좋은 AI 도구 = AI가 실패를 스스로 감지하고 고칠 수 있는 도구**

### B. 실제 부품으로서

1. **MCP 서버로 감싸기** — Bash 경유 없이 구조화된 툴 호출
2. **데이터 분석 에이전트의 출력 계층** — 분석 결과가 일회성 텍스트가 아니라
   재사용 가능한 대시보드로 남는다
3. **완전 로컬 스택** — Ollama + DuckDB + DAC = 에어갭 환경에서도 동작

### C. 훔쳐올 만한 기술 패턴

| 파일 | 패턴 |
|---|---|
| `embed.go` | React 빌드를 Go 바이너리에 embed → 단일 파일 배포 |
| `pkg/server/watcher.go` + SSE | 파일 감시 → 브라우저 자동 리로드 |
| `pkg/dashboard/jsloader.go` | goja + esbuild로 Go 안에서 JS 샌드박스 실행 (에이전트 생성 코드 안전 실행에 응용) |
| `pkg/query/cache.go` | 쿼리 결과 캐싱 (LLM 호출 캐싱에 동일 적용) |
| `cmd/skills.go` | 스킬 파일 버전 관리 및 배포/업데이트 |

---

## 11. React / PHP 재구현 가능성

DAC의 본질은 5개 부품이다:

```
① 파일 파서   (YAML/TSX → 구조체)
② 검증기      (스키마 체크)
③ 템플릿 엔진 (Jinja → SQL)
④ 쿼리 실행기 (DB 접속)
⑤ 렌더러      (구조체 → 차트)
```

### React / Next.js — 난이도 낮음

| 부품 | 구현 | 난이도 |
|---|---|---|
| YAML 파싱 | `js-yaml` | 매우 쉬움 |
| 검증 | `zod` 또는 `ajv` | 매우 쉬움 |
| Jinja 템플릿 | `nunjucks` | 쉬움 |
| DB 접속 | `pg`, `mysql2`, `@duckdb/node-api`, `@google-cloud/bigquery` | 보통 |
| 차트 | Recharts / Vega-Lite 그대로 재사용 | 매우 쉬움 |
| **TSX 대시보드** | **진짜 React → 그냥 실행. Go보다 훨씬 쉽고 강력** | 매우 쉬움 |

추천 스택: `Next.js 15 + js-yaml + zod + nunjucks + Kysely/Drizzle + Recharts + Vega-Lite`

단점: 단일 바이너리 이점 상실 → `bun build --compile` 또는 Docker/Vercel로 보완

### PHP / Laravel — 난이도 중간

| 부품 | 구현 | 난이도 |
|---|---|---|
| YAML 파싱 | `symfony/yaml` | 매우 쉬움 |
| 검증 | `justinrainbow/json-schema` | 쉬움 |
| Jinja 템플릿 | **`Twig`** (Jinja2에서 파생 — 호환성 최상) | 매우 쉬움 |
| DB 접속 | PDO | 쉬움 |
| 차트 | Blade/Twig + Chart.js / ECharts / Vega-Lite | 보통 |
| **TSX 대시보드** | **사실상 불가능** | - |

전략: TSX는 포기하고 YAML 전용으로 스코프 축소. 대신 공유 호스팅 지원,
기존 Laravel 프로젝트에 패키지로 삽입, 국내 SI/중소기업 시장 공략.

### 비교표

| 항목 | Go (원본) | React/Next.js | PHP/Laravel |
|---|---|---|---|
| 개발 난이도 | 높음 | **낮음** | 중간 |
| YAML 대시보드 | O | O | O |
| TSX 동적 대시보드 | goja로 제한적 | **네이티브, 더 강력** | X |
| Jinja 템플릿 | gonja | nunjucks | **Twig (최고 호환)** |
| 배포 | **단일 바이너리** | node_modules | 공유 호스팅 가능 |
| 차트 생태계 | 프론트 의존 | **최고** | JS 라이브러리 차용 |
| 국내 시장 적합도 | 보통 | 좋음 | **SI/중기 최강** |
| MVP 소요 | 2~4주 | **3~7일** | 1~2주 |

**권장: React / Next.js.** TSX 기능이 원본보다 강력해지고, 차트 생태계와
AI SDK(Vercel AI SDK, MCP TypeScript SDK)가 전부 JS 진영에 있다.

---

## 12. 수익화 아이디어

### 12.0 먼저: AGPL-3.0 라이선스 제약

AGPL은 GPL과 달리 **네트워크 서비스 제공만으로도 소스 공개 의무**가 발생한다.

| 시나리오 | 안전 여부 |
|---|---|
| 사내 내부 사용 | 안전 |
| DAC 기반 컨설팅 | 안전 (서비스 판매) |
| 교육/강의 판매 | 안전 |
| YAML 템플릿 판매 | 안전 (데이터, 파생저작물 아님) |
| MCP 서버를 별도 프로세스로 구현 | 대체로 안전 |
| DAC fork해서 SaaS 운영 | **소스 공개 의무 발생** |
| DAC를 라이브러리로 링크 | **전체 AGPL 전염** |
| 처음부터 직접 재구현 | **완전 자유** |

안전한 3가지 경로: (1) 서비스로 수익화, (2) 주변부 도구 제작, (3) 클린룸 재구현

---

### 12.1 DAC-MCP 서버 (오픈소스 → 명성 → 수익)

DAC CLI를 MCP 툴로 감싸는 브릿지.

```
dac_list_dashboards / dac_validate / dac_run_widget_query
dac_create_dashboard / dac_build_static / dac_preview_screenshot
```

| 단계 | 방법 | 예상 수익 |
|---|---|---|
| 1 | npm 무료 배포, GitHub 스타 확보 | $0 (투자) |
| 2 | GitHub Sponsors | 월 $50~500 |
| 3 | "MCP 전문가" 포지셔닝 → 기업 커스텀 개발 | 건당 500~2,000만원 |
| 4 | Pro 버전 (팀 협업, 스크린샷 diff, 권한 관리) | 월 $20/팀 |

- AGPL 안전 (별도 프로세스)
- MCP TS SDK로 **2~3일** MVP
- MCP 생태계 선점 효과
- 직접 수익은 작음 → "명함" 전략

---

### 12.2 한국형 대시보드 템플릿 마켓

| 팩 | 내용 | 가격 |
|---|---|---|
| 이커머스 | 쿠팡/네이버커머스/카페24 매출·정산·광고ROAS | 15만원 |
| 마케팅 | 카카오모먼트/네이버광고/GA4 퍼널 | 15만원 |
| SaaS | MRR/Churn/LTV/Cohort | 20만원 |
| 핀테크 | 결제 성공률/이탈/한도 | 25만원 |
| 올인원 | 전체 + 커스터마이징 1회 | 60만원 |

- YAML은 AGPL 영향 없음 → 판매 자유
- 실제 가치는 "지표 정의(MRR 계산식, 코호트 SQL)"에 있음
- AI가 고객 스키마에 맞게 자동 변환

예상: 템플릿 150만원 + 커스터마이징 200만원 + 구독 150만원 = **월 약 500만원**

---

### 12.3 "자연어 → 대시보드" SaaS

```
자연어 요청
  → AI가 DB 스키마 탐색
  → 대시보드 YAML/TSX 생성
  → validate로 셀프 검증
  → 실패 시 자동 수정 반복
  → 완성 + GitHub PR 자동 생성
```

차별점: 결과가 블랙박스가 아닌 **읽고 수정 가능한 코드**, PR로 거버넌스 확보, 자가 치유

| 플랜 | 가격 | 내용 |
|---|---|---|
| Free | $0 | 대시보드 3개, 월 20회 |
| Pro | $49/월 | 무제한, GitHub 연동, 커스텀 테마 |
| Team | $199/월 | SSO, 권한관리, 감사로그 |
| Enterprise | 문의 | 온프렘, 전용 모델, SLA |

**AGPL 충돌 주의.** 해결책:
- (A) React/Next.js로 직접 재구현 — 권장
- (B) Bruin에 상용 라이선스 문의
- (C) SaaS는 "PR 생성기"만 담당하고 DAC 실행은 고객 인프라에서

난이도 매우 높음 / 수익 잠재력 매우 높음

---

### 12.4 BI 마이그레이션 컨설팅 (현금화 최속)

`dac import metabase`로 변환이 반자동이라는 점을 활용.

| 패키지 | 내용 | 가격 |
|---|---|---|
| 진단 | 현황 분석 + 마이그레이션 계획서 | 200만원 |
| 실행 | 대시보드 50개 변환 + 검증 | 800만원 |
| 정착 | 팀 교육 + CI 구축 + 3개월 지원 | 500만원 |
| 운영 | 월 유지보수 + 신규 대시보드 | 월 150만원 |

- AGPL 완전 무관
- 개발 기간 0일 — 즉시 영업 가능
- 고객 ROI가 명확 ("BI 라이선스 연 3,000만원 → 0원")
- 단점: 시간 판매 → 확장성 낮음, 영업력 필요

---

### 12.5 교육 콘텐츠

| 상품 | 플랫폼 | 가격 | 예상 |
|---|---|---|---|
| "대시보드도 코드다: DAC 완전정복" | 인프런 | 8.8만원 | 300명 = 2,640만원 |
| "AI 에이전트를 위한 도구 설계" | Udemy | $89 | 글로벌 |
| 데이터 엔지니어링 채널 | YouTube | 광고+제휴 | 월 50~300만원 |
| 기업 출강 | 직접 영업 | 일 200만원 | 월 2~4회 |
| 유료 뉴스레터 | Stibee/Substack | 월 1만원 | 500명 = 월 500만원 |

AGPL 무관, 리스크 최소, 자산화 효과 높음

---

### 12.6 한국형 클론 직접 개발 (Open Core)

DAC의 **아이디어만** 차용해 React/Next.js로 재구현 → 라이선스 완전 자유

| 기능 | DAC | 자체 제품 |
|---|---|---|
| 한글 폰트/레이아웃 | 기본 | Pretendard 최적화 |
| 국내 데이터소스 | X | 네이버커머스/쿠팡/카카오 커넥터 |
| 카카오 알림톡 | X | 목표 달성 알림 |
| 엑셀 내보내기 | 부분 | 한국식 서식 완벽 지원 |
| 전자결재 연동 | X | 그룹웨어 연동 |
| 자연어 생성 | X | AI 내장 |

수익 모델:
```
Community  : MIT/Apache 오픈소스 → 커뮤니티 확보
Pro        : $29/월 — AI 생성, 커넥터팩, 알림톡, 권한관리
Enterprise : 온프렘, SSO, 감사로그 → 연 3,000만원+
Cloud      : 호스팅 SaaS $49~199/월
```

로드맵:
```
1~2주   MVP (YAML 파싱 + Recharts + Postgres/DuckDB)
3~4주   TSX 동적 대시보드 (진짜 React라 수월)
5~6주   필터 + nunjucks + 시맨틱 레이어
7~8주   AI 생성 (Claude API 연동)
9~12주  국내 커넥터 + 알림톡 + 베타 런칭
```

---

## 13. 실행 로드맵

```
[0~1개월] 인지도 확보
  - DAC-MCP 서버 오픈소스 공개
  - 블로그/유튜브 콘텐츠 제작
  → 수익 거의 0, 투자 낮음

[1~3개월] 현금 흐름 확보
  - 템플릿 팩 판매 (이커머스/마케팅/SaaS)
  - 마이그레이션 컨설팅 영업
  → 월 300~800만원

[3~12개월] 스케일업
  - React 기반 한국형 클론 개발 (라이선스 자유)
  - AI 대시보드 생성 SaaS 런칭
  → 월 2,000만원+
```

**첫 번째 액션 아이템: DAC-MCP 서버.**
2~3일 투자로 MCP 생태계 선점 + 포트폴리오 확보가 가능하고,
여기서 얻은 도메인 지식이 그대로 클론 개발로 연결된다.

---

## 14. 주의사항 요약

| 항목 | 내용 |
|---|---|
| **라이선스** | AGPL-3.0-only. SaaS 운영 시 소스 공개 의무 |
| **의존성** | `bruin` CLI 필수. DAC 단독으로 DB 접속 불가 |
| **TSX 제약** | 진짜 React 아님. goja 기반 제한 JS. npm 패키지/hooks 사용 불가 |
| **텔레메트리** | 기본 활성화. `DO_NOT_TRACK=1` 권장 |
| **admin 비밀번호** | 평문 비교. 내부망 또는 리버스 프록시 뒤에서만 사용 |
| **DB 자격증명** | `.bruin.yml`에 평문 금지. `${ENV_VAR}` + `.gitignore` |
| **버전 안정성** | v0.x — 파괴적 변경 가능성 |
| **빌드 규칙** | `go build`/`npm run build` 직접 호출 금지. `make` 사용 |
| **포크 상태** | 업스트림 `bruin-data/dac` 대비 커밋이 뒤처져 있을 수 있음 |

---

## 참고 링크

- 포크 저장소: https://github.com/bmshin94/dac
- 업스트림: https://github.com/bruin-data/dac
- 공식 문서: https://getbruin.com/docs/dac/
- 제품 페이지: https://getbruin.com/dashboards-as-code/
- 활용 사례: https://getbruin.com/use-cases/dashboards-as-code/
- Bruin CLI: https://github.com/bruin-data/bruin
- 릴리즈: https://github.com/bruin-data/dac/releases
