# 한입 기사 — 코드 분석 마스터 문서

> 이 문서는 **새로 합류한 팀원이 한입 기사 레포를 처음부터 끝까지 이해**할 수 있도록, 5개 모듈의 기술 스택·구조·핵심 로직을 한 곳에 모아 둔 참고용 마스터 문서입니다. 모듈을 깊이 파기 전에 먼저 이 문서를 한 번 훑어보면, 코드 어디를 봐야 할지 감을 잡을 수 있습니다.
>
> 본문은 `oba_deploy` 레포의 `develop` 브랜치를 기준으로 작성되었습니다.

<br/>

## 목차
1. [전체 그림](#1-전체-그림)
2. [데이터 흐름 — 하루의 라이프사이클](#2-데이터-흐름--하루의-라이프사이클)
3. [모듈별 심층 분석](#3-모듈별-심층-분석)
   - [3.1 Frontend (`oba_frontend`)](#31-frontend-oba_frontend)
   - [3.2 Backend (`oba_backend`)](#32-backend-oba_backend)
   - [3.3 AI Server (`oba_AI`)](#33-ai-server-oba_ai)
   - [3.4 Data Pipeline (`oba_data`)](#34-data-pipeline-oba_data)
   - [3.5 Infrastructure (`infra`)](#35-infrastructure-infra)
4. [학습 가이드 — 어디부터 봐야 할까?](#4-학습-가이드--어디부터-봐야-할까)

<br/>

---

## 1. 전체 그림

한입 기사는 **단일 배포 레포(모놀리포)** 안에 5개의 독립 모듈을 담고 있습니다. 모든 컨테이너는 `oba_shared_network` (Docker external network)를 통해 서로 통신합니다.

```
oba_deploy/
├── docker-compose.yml      ← 5개 컨테이너 오케스트레이션
├── oba_frontend/           ← Expo / React Native / TypeScript
├── oba_backend/            ← Spring Boot 3.5.4 / Java 17
├── oba_AI/                 ← FastAPI / OpenAI SDK
├── oba_data/               ← Apache Airflow / BeautifulSoup
└── infra/                  ← Jenkins / Mattermost
```

### 핵심 설계 원칙

| 원칙 | 어떻게 구현했나 |
| :--- | :--- |
| **시간 분리** | LLM 호출을 사용자 요청 시점이 아닌 **새벽 배치(Airflow)** 에서 끝낸다. 사용자는 이미 만들어진 결과만 본다. |
| **DB 책임 분리** | 트랜잭션이 필요한 사용자 데이터는 **MySQL(JPA)**, 매일 통째로 갈아 끼우는 콘텐츠 스냅샷은 **MongoDB** 로 분리. |
| **단일 코드, 다중 플랫폼** | **Expo Router 6** 로 iOS · Android · Web 을 하나의 코드베이스에서 동시 빌드. |
| **운영 가시성** | Jenkins 빌드 실패 시 `analyze_log.py` 가 로그 100줄을 요약해 **Mattermost** 로 자동 알림. |

<br/>

---

## 2. 데이터 흐름 — 하루의 라이프사이클

### 새벽 (07:00 ~ 09:00 KST) — 콘텐츠 준비 단계
```
[07:00] Airflow DAG #1
  └─ get_article_links: ITWorld 20개 카테고리 크롤링
       (AI 22.5% / GenAI 19.6% / Cloud 16.9% 등 가중치)
  └─ upsert_articles: MySQL `Articles` / `Categories` / `Article_Categories`

[09:00] Airflow DAG #2
  └─ select_top5_articles: 랭킹 알고리즘으로 Top 5 선정
  └─ crawl_and_save_contents: 본문 스크래핑 → MongoDB 스냅샷 저장
  └─ generate_gpt_results: FastAPI(oba-ai:8000) 호출 (300s 타임아웃)
        └─ GPT-4o-mini: 요약 + 키워드 10개 + 4지선다 5문항 생성
        └─ MongoDB `Selected_Articles.gpt_result` 필드에 저장
```

### 낮 (사용자 접속 시점) — 콘텐츠 서빙 단계
```
[사용자] Expo 앱에서 홈 진입
  └─ Axios → Spring `/api/articles/latest`
       └─ MongoDB에서 오늘 Top 5 스냅샷 즉시 응답 (LLM 호출 없음)

[사용자] 기사 상세 > 퀴즈 탭
  └─ 같은 스냅샷의 gpt_result.quizzes 표시

[사용자] 퀴즈 풀고 제출
  └─ Axios (Bearer 토큰) → Spring `/api/quiz/result`
       └─ 틀린 문제는 MySQL `IncorrectQuiz` 에 누적
       └─ UserStats / UserCategoryStats 업데이트
       └─ 401 발생 시 프론트 인터셉터가 `/auth/reissue` 자동 호출
```

이 흐름에서 가장 중요한 포인트는 **사용자 요청 경로에 LLM 호출이 없다**는 점입니다. LLM 호출의 비용·지연은 새벽 배치에서 모두 끝나 있기 때문에, 사용자는 항상 빠르게 응답을 받습니다.

<br/>

---

## 3. 모듈별 심층 분석

### 3.1 Frontend (`oba_frontend`)

**기술 스택**
- React 19.1.0 / React Native 0.81.5 / **Expo Router 6.0.23** (파일 기반 라우팅)
- TypeScript 5.9.2
- Axios 1.13.2 (`baseURL: https://onebitearticle.com`)
- React Context API (상태관리, Redux/Zustand 미사용)
- AsyncStorage (토큰 영속화)
- 빌드: Native Expo (Vite 없음)

**디렉토리 구조**
```
app/
├── (auth)/login.tsx
├── (tabs)/
│   ├── index.tsx              ← 홈 (오늘의 Top 5)
│   ├── my/index.tsx           ← 마이페이지
│   ├── report/index.tsx       ← 학습 리포트
│   ├── wrongArticles/index.tsx ← 오답 다시 풀기
│   └── history/index.tsx      ← 학습 히스토리
├── article/[id].tsx           ← 기사 상세 (기사·요약·퀴즈·키워드 4탭)
└── components/                ← HomeHeader, PizzaMenu, ReportStats, DailyChart 등
src/
├── auth/AuthContext.tsx       ← 토큰 저장 + 리프레시 인터셉터
├── api/apiClient.ts           ← Axios 인스턴스
└── utils/learningStats.ts
```

**핵심 로직 포인트**
- **인증 플로우**: 로그인 → access/refresh 토큰을 `AsyncStorage` 에 저장 → Axios 인스턴스가 `Authorization: Bearer ${access}` 헤더 자동 부착 → 401 응답 시 인터셉터가 `/auth/reissue` 호출해 새 토큰 받고 원 요청 재시도.
- **크로스플랫폼**: 동일 코드를 `npx expo start` 로 iOS · Android · Web 어디서든 실행. 플랫폼 분기는 거의 없음.
- **PizzaMenu**: 홈 화면의 시그니처 인터랙션. `usePizzaAnimation` 커스텀 훅으로 RN Animated API 활용.

**학습 포인트**
- Expo Router의 파일 기반 라우팅은 Next.js App Router와 유사하니, Next.js 경험자는 빠르게 적응 가능.
- AuthContext 한 곳에서 토큰 관리 + 인터셉터 등록하는 패턴 익혀두면 다른 RN 프로젝트에도 그대로 적용 가능.

<br/>

### 3.2 Backend (`oba_backend`)

**기술 스택**
- **Spring Boot 3.5.4** (Java 17)
- Spring Security + **JWT** (jjwt 0.11.5) — `JwtProvider` / `JwtAuthenticationFilter`
- JPA(MySQL) + Spring Data MongoDB **이중 DB**
- Quartz (스케줄링) / Caffeine 3.1.8 (캐싱)
- RestTemplate (AI 서버 호출)
- BCrypt (비밀번호 해시)

**도메인 구조**
```
domain/
├── article  (MongoDB)  → SelectedArticle, GptResult, QuizItem
├── quiz     (MySQL)    → IncorrectQuiz (사용자가 틀린 문제 기록)
├── user     (MySQL)    → User (Role, AuthProvider enum)
├── stats    (MySQL)    → UserStats, UserCategoryStats (스트릭, "완벽한 날" 추적)
├── report                → 리포트 조립
├── feedback              → 사용자 피드백 (1000자 제한)
├── log      (MySQL)    → ArticleLog (열람 로그)
└── ai                    → FastAPI 호출 어댑터
```

**주요 API 엔드포인트**

| 경로 | 메서드 | 보호 | 기능 |
| :--- | :--- | :--- | :--- |
| `/api/articles/latest` | GET | 공개 | 최신 Top 5 조회 |
| `/api/articles/{id}` | GET | 공개* | 기사 상세 (선택적 인증) |
| `/api/quiz/result` | POST | 인증 | 퀴즈 결과 저장 |
| `/api/quiz/wrong` | GET | 인증 | 틀린 기사 목록 |
| `/api/report/stats` · `/progress` · `/category-progress` · `/daily-stats` | GET | 인증 | 학습 통계 |
| `/api/users/me` | GET | 인증 | 사용자 정보 + 주간 학습 로그 |
| `/api/users/nickname` | PUT | 인증 | 닉네임 변경 |
| `/auth/signup` · `/login` · `/reissue` | POST | 공개 | 가입 / 로그인 / 토큰 재발급 |
| `/auth/check-email` · `/reset-password` | GET/POST | 공개 | 이메일 확인 / 비밀번호 재설정 |
| `/auth/password` | PUT | 인증 | 비밀번호 변경 |
| `/ai/generate/daily` | POST | 내부키 | Daily AI 작업 트리거 (Airflow 용) |

**핵심 로직 포인트**
- **이중 DB**: `JpaRepository` 와 `MongoRepository` 가 다른 DataSource를 본다. MongoDB는 매일 새 스냅샷이 들어오므로 사실상 읽기 전용으로 운영.
- **JWT 세션 모델**: STATELESS. 모든 요청은 `Authorization` 헤더의 Bearer 토큰을 `JwtAuthenticationFilter` 가 검증.
- **AI 연동**: `RestTemplate` 으로 `${AI_SERVER_URL}/generate/daily_gpt_results` 호출. 선택적 `X-Internal-Key` 헤더로 내부 호출 식별.
- **통계 누적**: 사용자가 퀴즈 제출 시 `UserStats` 의 연속 학습일(streak) / "완벽한 날" / 카테고리별 진행률이 동시에 업데이트됨.

**학습 포인트**
- Spring Boot에서 **JPA + MongoDB를 한 앱에서 같이 쓰는 설정** — `@EnableJpaRepositories` / `@EnableMongoRepositories` 패키지 분리 패턴 확인.
- JWT 필터 체인이 SecurityFilterChain에 어떻게 끼워지는지, JwtAuthenticationFilter가 OncePerRequestFilter를 어떻게 상속하는지가 백엔드 학습 핵심.

<br/>

### 3.3 AI Server (`oba_AI`)

**기술 스택**
- FastAPI 0.119.0 + Uvicorn 0.37.0
- **OpenAI SDK 1.44.0** (LangChain 미사용, SDK 직접 호출)
- PyMongo 4.7.2
- 기타: python-dotenv, BeautifulSoup4, httpx

**디렉토리 구조**
```
app/
├── main.py                ← FastAPI 진입점 (포트 8000, CORS 허용)
├── api/ai_controller.py   ← 라우터 (/generate 프리픽스)
├── services/ai_service.py ← AI 핵심 로직
├── schemas/gpt_schema.py  ← Pydantic 스키마
├── db/mongo.py            ← MongoDB 연결
└── core/config.py         ← 환경변수
```

**API**
- `POST /generate/gpt_result` — 특정 Article ID 1건에 대해 GPT 분석 수행
- `POST /generate/daily_gpt_results` — 오늘 기사 5개 일괄 분석 (Airflow 용)
- `GET /` — Health check

**LLM 호출**
- 모델: **`gpt-4o-mini`** / Temperature: 0.7
- 입력 제한: 기사 본문 첫 **7,000자** 만 전송
- 호출 방식: OpenAI SDK 직접 호출 (단순/투명)

**프롬프트 설계 (정교한 부분)**
- **Summary**: IT 취업준비생 관점 — 기술 동향 · 시장 변화 · 기업 전략 중심
- **Keywords**: 최대 10개, 각 키워드에 신뢰할 수 있는 기술 설명을 함께 생성
- **Quizzes**: 기사 기반 4지선다 5문항, 각 항목에 **정답 인덱스 + 상세 해설** 포함
- **편향 방지**: 정답 위치가 한쪽에 몰리지 않도록 옵션 순서를 자동 셔플

**출력 스키마**
```json
{
  "summary": "string",
  "keywords": [{ "keyword": "string", "description": "string" }],
  "quizzes": [
    {
      "question": "string",
      "options": ["A", "B", "C", "D"],
      "answer": 0,
      "explanation": "string"
    }
  ]
}
```

**DB 연동**
- DB명: `OneBitArticle` (주의: 'e' 없음 — 프로젝트 전체에서 이 이름 그대로 사용)
- 컬렉션: `Selected_Articles`
- 읽기: `content_col` (본문 블록 배열), `serving_date`
- 쓰기: `gpt_result` 필드만 업데이트

**학습 포인트**
- LangChain 없이 **OpenAI SDK + Pydantic** 만으로 LLM 출력을 검증·소비하는 깔끔한 패턴.
- 정답 셔플 같은 사소해 보이는 후처리가 실제 학습 품질에 미치는 영향 — 코드에서 한 번 직접 따라가보면 좋음.

<br/>

### 3.4 Data Pipeline (`oba_data`)

**기술 스택**
- Apache Airflow (standalone 모드)
- BeautifulSoup4 + requests (크롤링)
- MySQL (메타데이터) + MongoDB (스냅샷)

**DAG 구성**

**DAG #1 — `news_crawling_to_db_dag.py`** (매일 07:00 KST)
1. `get_article_links`: ITWorld(itworld.co.kr) **20개 카테고리** 크롤링
   - 카테고리별 가중치: AI 22.5% · GenAI 19.6% · Cloud 16.9% 등
   - 1~3초 랜덤 딜레이로 매너 있는 크롤링
2. `upsert_articles`: MySQL `Articles` / `Categories` / `Article_Categories` 업서트
   - `Articles.URL` 은 unique constraint

**DAG #2 — `select_top5_and_save_dag.py`** (매일 09:00 KST)
1. `select_top5_articles`: 랭킹 알고리즘으로 그날의 Top 5 선정
2. `crawl_and_save_contents`: Top 5 본문 스크래핑 → MongoDB `Selected_Articles` 저장
3. `generate_gpt_results`: FastAPI(`oba-ai:8000/generate/daily_gpt_results`) 호출 (300초 타임아웃)

**DB 스키마 (요약)**

| DB | 테이블/컬렉션 | 역할 |
| :--- | :--- | :--- |
| MySQL `oba_article` | `Articles` (URL unique, ordering score, is_used, serving_date) | 크롤링한 기사 메타 |
|  | `Categories`, `Article_Categories` | 카테고리 매핑 |
| MySQL `oba_backend` | `users` (OAuth provider), `Incorrect_Articles`, `Incorrect_Quiz` | 사용자 데이터 |
| MongoDB `OneBitArticle` | `Selected_Articles` (본문 + `gpt_result`) | 매일 선정된 기사 스냅샷 |

**초기화 파일**: `db_initialize.sql`, `docker_db_initialize.sql`, `first_day_insertion.sql`

**리소스 제어**
- Airflow DB 풀: `pool_size=3`, `max_overflow=0`, `pool_recycle=300`
- 실행 동시성: `PARALLELISM=2`, `MAX_ACTIVE_RUNS_PER_DAG=1`, `DAG_CONCURRENCY=1`
- 원격 로깅: S3 `obadata-airflow-logs/airflow/logs`

**학습 포인트**
- **모놀리식 Airflow standalone**은 작은 팀에 적합한 선택. CeleryExecutor를 굳이 안 쓴 이유는 트래픽이 일 2회 배치라 단순성을 우선했기 때문.
- 크롤링 시 **랜덤 딜레이 + User-Agent 매너**가 실제로 차단을 막아주는 핵심.

<br/>

### 3.5 Infrastructure (`infra`)

**기술 스택**
- Jenkins (포트 9090:8080) — **Docker-in-Docker** 구성
- Docker Compose (루트의 `docker-compose.yml` 이 5개 컨테이너 오케스트레이션)
- Mattermost Webhook (CI 알림)

**CI/CD 파이프라인** (`infra/jenkins/Jenkinsfile`)
```
Stage 1: Inject Secrets
  └─ Jenkins credentials → .env 파일로 주입
       (DB_PASSWORD, MONGODB_URI, JWT_SECRET,
        OAuth 3종(Google/Kakao/Naver), OPENAI_API_KEY)

Stage 2: Build & Deploy
  └─ docker-compose up --build -d

Post: 알림
  └─ 성공 → Mattermost webhook 알림
  └─ 실패 → analyze_log.py 가 로그 100줄 요약 → Mattermost 통보
```

**포트 매핑 (전체)**

| 서비스 | 호스트 포트 | 컨테이너 내부 |
| :--- | :--- | :--- |
| Jenkins | 9090 | 8080 |
| Airflow | 8080 | 8080 |
| FastAPI (AI) | 8000 | 8000 |
| Spring Backend | 8081 | 8080 |
| Frontend | 8082 | 8081 |
| MongoDB | 27017 | 27017 |

> **Nginx / 리버스 프록시 없음** — 직접 포트 노출 구조. 운영 환경에서는 별도 도메인의 SSL 처리가 외부에서 이루어진다고 가정.

**로깅 정책**
- 모든 컨테이너: `json-file` 드라이버, 회전 정책 (max-size 5~10m, max-file 3)
- Airflow: 추가로 S3 원격 로깅 활성화, 로컬 로그는 자동 삭제

**학습 포인트**
- Jenkins **DinD** 패턴은 보안·성능 트레이드오프가 있음. 왜 굳이 DinD를 선택했는지 Jenkinsfile에서 docker 명령이 어떻게 실행되는지를 따라가보면 좋음.
- `analyze_log.py` 같은 작은 운영 도구 — 큰 모니터링 솔루션 없이도 팀이 빨리 반응할 수 있도록 만든 합리적 선택.

<br/>

---

## 4. 학습 가이드 — 어디부터 봐야 할까?

처음 합류한 팀원이라면 **자기 담당 모듈을 먼저 깊이 보고**, 다음 순서로 인접 모듈을 훑으면 가장 효율적입니다.

### 📱 Frontend 담당
1. `oba_frontend/app/(tabs)/index.tsx` — 홈 화면이 어떻게 Top 5를 받아오는지
2. `oba_frontend/src/auth/AuthContext.tsx` — JWT 자동 갱신 인터셉터
3. `oba_backend` 의 `ArticleController` — 자기가 호출하는 API의 응답 스키마 확인
4. 데이터 흐름 섹션 한 번 더 읽기

### ☕ Backend 담당
1. `oba_backend/src/main/java/.../SecurityConfig.java` — 보안 필터 체인
2. `oba_backend/.../domain/article` 패키지 — MongoDB 도메인이 어떻게 구성되는지
3. `oba_backend/.../domain/ai` 패키지 — FastAPI 호출 어댑터
4. `oba_AI/app/services/ai_service.py` — 자기가 호출하는 AI 서버의 응답 스키마

### 🤖 AI 담당
1. `oba_AI/app/services/ai_service.py` — 프롬프트 + 셔플 로직
2. `oba_AI/app/schemas/gpt_schema.py` — Pydantic 출력 스키마
3. `oba_data/airflow/dags/select_top5_and_save_dag.py` — 자기를 호출하는 쪽
4. `oba_backend/.../domain/ai` — 백엔드가 직접 호출하는 경로도 확인

### 📊 Data 담당
1. `oba_data/airflow/dags/news_crawling_to_db_dag.py` — 크롤링 DAG
2. `oba_data/airflow/dags/select_top5_and_save_dag.py` — 랭킹 + AI 트리거 DAG
3. `oba_data/sql/*.sql` — DB 스키마
4. `oba_AI` 와 `oba_backend` 에서 자기 데이터를 어떻게 소비하는지

### ⚙️ Infra / PM
1. 루트 `docker-compose.yml` — 전체 토폴로지
2. `infra/jenkins/Jenkinsfile` — 시크릿 주입 흐름
3. `infra/jenkins/analyze_log.py` — 운영 자동화
4. 5개 모듈의 README를 차례로 훑기

<br/>

---

> 이 문서에 없는 더 세세한 부분은 각 모듈 폴더의 README와 코드 주석을 참조해주세요. 잘못된 내용이나 빠진 부분을 발견하면 PR로 바로 수정해주시면 감사하겠습니다.
