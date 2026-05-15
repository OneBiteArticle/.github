# 내일 작업용 프롬프트 (Handoff)

> 이 파일을 열어서 아래 "복붙용 프롬프트" 섹션을 그대로 Claude에 붙여넣으면 어제 멈춘 지점부터 이어서 작업합니다.

---

## 어제까지 진행 상황

### 목표
- `OneBiteArticle` Org의 `.github` 레포에 들어갈 **프로필 README** 작성
- 본인이 채울 수 있는 부분 먼저 만들고, 팀원들이 나머지 채우는 구조
- 추가로 `profile/` 폴더 안에 **코드 분석 문서들** 작성

### 레퍼런스 구조
- 예시: https://github.com/BarmiSpeechLab/.github/tree/main/profile
- 차이점: 예시는 `profile/README.md` 한 파일에 다 들어있는데, 우리는
  - `README.md` (최상위, 메인)
  - `profile/` 폴더 — 다른 상세 문서들 보관

### 현재 폴더 상태 (`C:/Users/SSAFY/Desktop/.github/`)
```
.github/
├── .git/
├── README.md      ← 제목만 있는 상태 ("# OneBiteArticle")
├── PROMPT.md      ← 이 파일
└── profile/       ← 빈 폴더 (아직 git에 잡히지 않음)
```

### 프로젝트 개요 (조사 완료 분)
- **이름**: 한입 기사 (OneBiteArticle)
- **소스 레포**: `C:/Users/SSAFY/Desktop/oba_deploy` (브랜치: `develop`)
- **구조**: 모놀리포 배포용 (5개 모듈)
  - `oba_frontend/` — React + Vite + Expo (크로스플랫폼 가능성), TypeScript
  - `oba_backend/` — Spring Boot (Java), MongoDB 사용
  - `oba_AI/` — FastAPI (Python), AI 분석/요약/퀴즈 생성
  - `oba_data/` — Airflow 데이터 파이프라인 (IT World 크롤링)
  - `infra/` — Jenkins CI/CD
- **공통 인프라**: MongoDB 7, Docker Compose, `oba_shared_network`

### ✅ 이미 조사 완료된 부분 (Data + Infra)

#### Data Pipeline (`oba_data/`)
- **DAG 1: `news_crawling_to_db_dag.py`** — 매일 07:00 (KST)
  - `get_article_links`: ITWorld(itworld.co.kr) 20개 카테고리 크롤링 (AI 22.5%, GenAI 19.6%, Cloud 16.9% 가중치)
  - `upsert_articles`: MySQL `Articles`/`Categories`/`Article_Categories` 업서트
- **DAG 2: `select_top5_and_save_dag.py`** — 매일 09:00
  - `select_top5_articles`: 랭킹 알고리즘으로 Top 5 선정
  - `crawl_and_save_contents`: 본문 스크래핑 → MongoDB 저장
  - `generate_gpt_results`: FastAPI(`oba-ai:8000`) 호출해서 퀴즈/요약 생성 (300s 타임아웃)
- **크롤링**: BeautifulSoup4 + requests, 1-3초 랜덤 딜레이
- **DB 스키마**:
  - MySQL `oba_article`: Articles (URL unique, ordering score, is_used, serving_date), Categories, Article_Categories
  - MySQL `oba_backend`: users (OAuth), Incorrect_Articles, Incorrect_Quiz
  - MongoDB `OneBitArticle`: 선정된 기사 스냅샷
- **DB 초기화 파일**: `db_initialize.sql`, `docker_db_initialize.sql`, `first_day_insertion.sql`

#### Infrastructure (`infra/`)
- **Jenkins** (포트 9090:8080, 50000:50000) — Docker-in-Docker
- **CI/CD 파이프라인** (`infra/jenkins/Jenkinsfile`)
  - Stage 1: Jenkins credentials → `.env` 파일 주입 (DB_PASSWORD, MONGODB_URI, JWT_SECRET, OAuth 3종(G/K/N), OPENAI_API_KEY)
  - Stage 2: `docker-compose up --build -d`
  - Post: 성공/실패 시 Mattermost webhook 알림, 실패 시 `analyze_log.py`로 로그 100줄 분석
- **포트 매핑**:
  - Jenkins 9090, Airflow 8080, FastAPI 8000, Spring 8081, Frontend 8082, MongoDB 27017
  - Nginx/리버스 프록시 **없음** — 직접 포트 노출
- **로깅**: json-file 로테이션, Airflow는 S3(`obadata-airflow-logs`) 원격 로깅

#### docker-compose.yml (루트)
- Airflow standalone (MySQL backend, 커넥션 풀 3개 제한)
- Spring backend (prod 프로필, Hikari pool 8 max / 2 min)
- FastAPI uvicorn (`app.main:app`)
- Frontend (Vite 빌드)
- MongoDB 7 (AI + Spring 공유)

### ❌ 아직 조사 안 된 부분 (어제 권한 거절됨)
- **oba_backend**: 도메인 구조, 컨트롤러/엔드포인트, JWT/OAuth 구현, AI 연동 방식
- **oba_AI**: FastAPI 라우트, LLM(OpenAI?) 호출 구조, 프롬프트 템플릿
- **oba_frontend**: 화면/라우트, 상태관리, API 연동, Expo vs Web

---

## 복붙용 프롬프트 (내일 출근해서 그대로 붙여넣기)

```
어제 OneBiteArticle org의 .github 레포 작업을 하다가 퇴근했어.
C:/Users/SSAFY/Desktop/.github/PROMPT.md 파일을 읽어서 진행 상황을 파악하고,
아래 순서대로 작업해줘:

1. 먼저 oba_backend, oba_AI, oba_frontend 세 모듈을 병렬로 깊게 분석해줘.
   (Explore 서브에이전트 3개 동시 실행. 어제 권한 거절돼서 못한 부분이야.)

2. 분석이 끝나면 다음 파일들을 작성해줘:

   A) C:/Users/SSAFY/Desktop/.github/README.md (최상위 메인 README)
      - 프로젝트 소개 (한입 기사 = 매일 IT 뉴스 Top5 + AI 퀴즈)
      - 서비스 컨셉/기획 배경 자리 (팀원이 채울 수 있게 placeholder)
      - 통합 아키텍처 다이어그램 자리 (이미지 placeholder)
      - 저장소 구성 가이드 (5개 모듈 표)
      - 핵심 가치 체인 (데이터 파이프라인 + AI 퀴즈 + 백엔드 API)
      - 실행 방법 (docker-compose up)
      - 팀원 자리 (placeholder)
      - 참고: https://github.com/BarmiSpeechLab/.github/tree/main/profile 스타일

   B) C:/Users/SSAFY/Desktop/.github/profile/ 폴더 안에 모듈별 상세 문서
      - frontend.md
      - backend.md
      - ai.md
      - data-pipeline.md   ← PROMPT.md에 이미 조사 완료된 내용 정리해서 작성
      - infrastructure.md  ← PROMPT.md에 이미 조사 완료된 내용 정리해서 작성

3. 작성 원칙:
   - 내가 직접 채우기 어려운 부분(기획 배경, 데모 GIF, 팀원 정보 등)은
     `<!-- TODO: 팀원이 채울 부분 -->` 주석으로 명확히 표시
   - 코드에서 확인 가능한 사실(스택, 엔드포인트, DAG 동작 등)은 구체적으로
   - 마크다운 한글로 작성, 이모지는 섹션 헤더에만 가볍게 사용
   - 추측 금지 — 코드/문서에서 확인된 것만 기재

작업 전에 한 번 더 확인할 거 있으면 물어봐.
```

---

## 참고 메모

- `.github` 레포는 GitHub Org 프로필 페이지에 자동 노출되는 메타 레포
- 일반적으로는 `profile/README.md` 하나만 두면 되는데, 본인 팀은 `README.md` + `profile/` 폴더 구조로 갈 예정
- 이게 GitHub 프로필 페이지에 어떻게 표시될지 한 번 확인 필요할 수도 (둘 다 두면 어느 게 우선?)
- `profile/` 안에 파일이 있어야 git이 폴더 추적함
