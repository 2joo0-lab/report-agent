# Report Agent

PostgreSQL/pgvector에 저장한 6종의 보고서를 관리하고, 다국어 E5 임베딩 기반의 하이브리드 검색과 LLM 요약을 제공하는 FastAPI 애플리케이션입니다.

보고서 생성·수정 시 개요(`overview`)를 실제 임베딩으로 변환해 저장합니다. 검색은 제목·개요·본문의 키워드 결과를 우선하고, 결과가 없을 때 코사인 유사도 기반 의미 검색을 사용합니다.

## 주요 기능

- 6개 카테고리 보고서 CRUD: 실험, 일일 업무, 연구 계획, 데이터 처리, 프롬프트, 회의
- 카테고리별 상세 필드 검증 및 Prompt 본문 일관성 검증
- PostgreSQL + pgvector(384차원) 기반 임베딩 저장·검색
- 키워드 및 의미 검색 API
- OpenAI Responses API를 이용한 간단/구조화 요약
- 보고서 조회·검색·요약을 조합하는 Agent API
- 단계별 기능 제어: `basic`, `intermediate`, `advanced`

## 구성

```text
app/                 FastAPI 라우터, 서비스, ORM 모델, 외부 연동
common/
  sql/               DB 생성·스키마·시드·검증 SQL
  data/              시드 카탈로그와 검색 검증 케이스
  postman/           Postman 컬렉션과 환경 파일
  requirements.txt   Python 의존성 목록
scripts/
  index_overviews.py 시드 보고서 개요의 임베딩 인덱싱
tests/               계약 테스트와 선택적 라이브 통합 테스트
```

## 사전 요구 사항

- Python 3.11 이상
- PostgreSQL 및 `pgvector` 확장
- 최초 모델 다운로드가 가능한 네트워크 환경
- LLM 요약을 사용할 경우 OpenAI API 키와 모델명

## 빠른 시작

PowerShell 기준입니다.

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r common\requirements.txt
Copy-Item common\.env.example .env
```

`.env`에서 최소한 아래 값을 실제 환경에 맞게 변경합니다. `.env`에는 비밀값이 들어가므로 커밋하지 않습니다.

```dotenv
DATABASE_URL=postgresql+psycopg://postgres:<password>@localhost:5432/report_agent_lab
EMBEDDING_MODEL=intfloat/multilingual-e5-small
SEARCH_MIN_SIMILARITY=0.75
PROJECT_LEVEL=basic

# 요약 API를 사용할 때만 설정
OPENAI_API_KEY=
LLM_MODEL=
```

### 데이터베이스 초기화

1. PostgreSQL의 기본 `postgres` 데이터베이스에서 `common/sql/00_create_database.sql`을 한 번 실행합니다.
2. 생성된 `report_agent_lab` 데이터베이스에서 순서대로 `common/sql/schema.sql`, `common/sql/seed.sql`을 실행합니다.
3. 시드 데이터의 개요 벡터를 생성합니다. E5 모델은 첫 실행 시 다운로드될 수 있습니다.

```powershell
python scripts\index_overviews.py
```

초기 시드 상태에서는 보고서 180건과 임베딩 180건이 준비됩니다. 다음 SQL로 상태를 점검할 수 있습니다.

```powershell
psql -d report_agent_lab -f common\sql\verify.sql
```

### 서버 실행

```powershell
python -m app.main
```

서버는 `http://127.0.0.1:8000`에서 실행됩니다. 대화형 API 문서는 `http://127.0.0.1:8000/docs`에서 확인할 수 있습니다.

## API 요약

| Method | Endpoint | 설명 |
| --- | --- | --- |
| `GET` | `/health` | DB 및 pgvector 준비 상태 확인 |
| `GET` | `/api/v1/categories` | 카테고리 목록 |
| `POST` | `/api/v1/reports` | 보고서 생성 및 개요 임베딩 저장 |
| `GET` | `/api/v1/reports` | 보고서 목록 조회 |
| `GET` | `/api/v1/reports/{report_id}` | 보고서 상세 조회 |
| `PUT` | `/api/v1/reports/{report_id}` | 보고서 수정 |
| `DELETE` | `/api/v1/reports/{report_id}` | 보고서 삭제 |
| `POST` | `/api/v1/search/similar` | 키워드/의미 유사 검색 |
| `POST` | `/api/v1/reports/{report_id}/summarize` | OpenAI 기반 요약 |
| `POST` | `/api/v1/agent/run` | 조회·검색·요약 Agent 실행 |

예: 유사 보고서 검색

```powershell
Invoke-RestMethod -Method Post `
  -Uri http://127.0.0.1:8000/api/v1/search/similar `
  -ContentType 'application/json' `
  -Body '{"query":"데이터 품질 개선", "keyword_limit":20, "semantic_limit":20}'
```

검색 결과가 임계값을 넘지 못하면 HTTP 200과 함께 `status: "not_found"`, `best_match: null`을 반환합니다.

## 프로젝트 단계

`PROJECT_LEVEL`로 공개 기능을 제어합니다.

| 값 | 포함 기능 |
| --- | --- |
| `basic` | CRUD, 기본 검색, 간단 요약, 명시적 Agent action |
| `intermediate` | 구조화 요약, 고급 목록 필터, 가중치 검색, 검색·요약 조합 Agent action |
| `advanced` | 수정 요청의 revision 충돌 감지, LLM 재시도·사용량 정보, Agent 실행 이력 |

현재 설정 확인:

```powershell
python -c "from app.core.config import settings; print(settings.project_level)"
```

## 테스트

DB 없이 실행 가능한 계약 테스트:

```powershell
python -m pytest -m "not live"
```

실제 PostgreSQL/pgvector와 시드·인덱싱까지 준비된 환경에서만 라이브 테스트를 실행합니다.

```powershell
$env:RUN_LIVE_TESTS = '1'
python -m pytest -m live
```

## Postman과 유료 LLM 호출

`common/postman/Report_Agent.postman_collection.json` 및 환경 파일을 Import하면 API 흐름을 점검할 수 있습니다. 상세한 실행 순서는 [Postman 가이드](common/postman/POSTMAN_GUIDE.md)를 참고하세요.

LLM 요청은 API 비용이 발생할 수 있습니다. Postman 환경의 `allow_paid_llm`은 기본값 `false`로 유지하고, 필요한 단일 요청을 확인할 때만 `true`로 바꾸세요. 요약 API는 `OPENAI_API_KEY`와 `LLM_MODEL`이 없으면 `503 LLM_UNAVAILABLE`을 반환합니다.

## 참고 자료

- [요구사항 명세서](common/docs/요구사항명세서_v1_4.html)
- [ERD](common/docs/report_agent_full_erd_v1_3.svg)
- [검색 검증 케이스](common/data/search_cases.json)
