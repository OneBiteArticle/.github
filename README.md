# 🍕 한입 기사 (OneBiteArticle)

> *"하루 5분, IT 뉴스 한 입씩"*
> *세상은 빠르게 바뀌지만, 우리가 따라가야 할 IT 뉴스는 매일 너무 많이 쏟아져 옵니다. 어디서부터 읽어야 할지, 무엇을 기억해야 할지 막막한 사람들이 있습니다.*

<br/>

**한입 기사(OneBiteArticle)** 는 매일 자동 큐레이션된 **IT 뉴스 Top 5**를 **AI 요약·키워드·퀴즈**와 함께 학습하는 모바일·웹 학습 플랫폼입니다. 한입 기사는 매일 새벽 ITWorld에서 기사를 자동 크롤링·랭킹한 뒤, GPT가 IT 취업준비생 관점에서 분석한 5문항 퀴즈와 핵심 키워드 해설을 통해 사용자가 짧은 시간 안에 학습을 마치고 자신의 학습 흔적을 남길 수 있도록 합니다.

<br/>

## 기획 배경
### 기획 의도 및 기대효과
+ **배경**
  + IT 분야 취업을 준비하는 사람에게 매일의 기술 뉴스 습득은 사실상 학습이지만, 발행량이 많고 우선순위가 흐려 챙겨 읽는 것 자체가 부담스럽다.
  + 매일 신문·뉴스 사이트를 직접 챙기는 습관을 유지하기 어렵고, 한 번 읽고 지나가는 방식만으로는 학습 흔적이 남지 않는다.
  + 그날의 핵심만 짧게 추려 보여주고, 능동적으로 한 번 더 풀어보게 하는 학습 장치가 필요하다.
+ **목표**
  + 한입 기사는 매일 가장 중요하고 많이 읽힌 **IT 기사 Top 5** 만 자동으로 골라, AI가 만든 요약·키워드·퀴즈와 함께 전달합니다. 매일 5분 분량의 부담 없는 능동 학습으로 IT 취업준비생이 그날의 기술 흐름을 자기 것으로 만들 수 있도록 돕는 것이 목표입니다.

<br/>

### 사전 인터뷰
<!-- TODO: 사전 인터뷰 결과 이미지 -->
![prev_interview](사전인터뷰.png)


<br/>
<br/>


## 🔎 한입 기사 기능 살펴보기

### 홈화면 진입 | 로그인 | 튜토리얼
![](./인트로.gif)

<br/>

### 오늘의 Top 5 · 기사 상세
![](./기사학습.gif)

<br/>

### 퀴즈 풀이 · 키워드 학습
![](./퀴즈.gif)

<br/>

### 오답 다시 풀기 · 학습 리포트
![](./리포트.gif)

<br/>

## 서비스 사용 후기
<!-- TODO: 사용자 후기 이미지/캡처 -->
![서비스 사용 후기](#)

<br/>
<br/>
<br/>


## 🏗 통합 서비스 아키텍처

한입 기사는 **데이터 수집 → AI 분석 → 서비스 제공**의 3단 비동기 파이프라인 구조를 따릅니다. Airflow가 새벽 시간대에 크롤링·랭킹·AI 생성을 차례로 트리거하고, Spring 백엔드는 미리 만들어진 결과만 서빙해 사용자 요청 시점의 부하를 분리했습니다.
<!-- TODO: 통합 아키텍처 다이어그램 이미지 -->
<div style="text-align: center;">
  <img src="#" alt="한입 기사 아키텍처">
</div>

<br/>
<br/>
<br/>

## 📄 ERD
<!-- TODO: ERD 이미지 -->
<img width="1653" height="1279" alt="ERD" src="#" />

<br/>
<br/>
<br/>

## 📂 저장소 구성 가이드

한입 기사는 단일 배포 레포(모놀리포) 구조이며, 5개 모듈이 독립된 기술 스택을 가지고 역할에 따라 최적화되어 있습니다.

| 모듈명 | 주요 기술 스택 | 핵심 역할 |
| :--- | :--- | :--- |
| **[Frontend](https://github.com/OneBiteArticle/oba_deploy/tree/develop/oba_frontend)** | React 19, React Native, Expo Router 6, TypeScript 5.9, Axios | **크로스플랫폼 클라이언트**: iOS / Android / Web 단일 코드베이스, Context API 인증, Axios 인터셉터 기반 토큰 자동 갱신 |
| **[Backend](https://github.com/OneBiteArticle/oba_deploy/tree/develop/oba_backend)** | Spring Boot 3.5.4, Java 17, Spring Security + JWT, JPA(MySQL) + Spring Data MongoDB, Quartz, Caffeine | **서비스 코어**: 인증·인가, 일일 기사 서빙, 퀴즈 결과·오답·통계 관리, AI 서버 트리거 |
| **[AI](https://github.com/OneBiteArticle/oba_deploy/tree/develop/oba_AI)** | Python, FastAPI 0.119, OpenAI SDK 1.44 (`gpt-4o-mini`), PyMongo | **AI 분석 서버**: 기사 본문 → 요약 · 키워드 10개 · 4지선다 퀴즈 5문항 단일 응답 생성 |
| **[Data](https://github.com/OneBiteArticle/oba_deploy/tree/develop/oba_data)** | Apache Airflow (standalone), BeautifulSoup4, MySQL, MongoDB | **데이터 파이프라인**: ITWorld 20개 카테고리 일일 크롤링 → 랭킹 → Top 5 선정 → AI 분석 트리거 |
| **[Infra](https://github.com/OneBiteArticle/oba_deploy/tree/develop/infra)** | Jenkins (Docker-in-Docker), Docker Compose, Mattermost Webhook | **CI/CD 자동화**: 시크릿 주입 · 빌드/배포, 실패 시 로그 자동 요약 알림 |

<br/>
<br/>
<br/>

## 🚀 서비스 핵심 가치 체인

### 1. 자동 큐레이션 파이프라인
1. **Crawling**: 매일 07:00 KST, Airflow가 ITWorld 20개 카테고리(AI 22.5% · GenAI 19.6% · Cloud 16.9% 등 가중치)에서 기사 링크를 수집해 MySQL에 업서트합니다.
2. **Top 5 선정**: 매일 09:00 KST, 랭킹 알고리즘으로 그날의 Top 5를 선정하고 본문을 스크래핑해 MongoDB에 스냅샷을 저장합니다.
3. **AI Trigger**: 같은 DAG가 FastAPI(`oba-ai:8000`)를 호출(300초 타임아웃)해 분석 결과 생성을 위임합니다.
4. **Serving**: Spring 백엔드는 이미 완성된 결과를 즉시 응답해, 사용자 요청 시점에 LLM 호출이 일어나지 않습니다.

<br/>

### 2. AI 기반 학습 콘텐츠 생성
- **단일 응답 다중 산출**: 기사 본문(첫 7,000자) → 요약 · 키워드 10개(설명 포함) · 4지선다 퀴즈 5문항(정답·해설)을 한 번의 GPT-4o-mini 호출로 생성합니다.
- **편향 방지**: 정답 선택지가 한쪽에 몰리지 않도록 옵션 순서를 자동 셔플합니다.
- **취준생 관점 프롬프트**: 기술 동향·시장 변화·기업 전략 중심의 요약을 유도해, 단순 정보 전달을 넘어선 학습 가치를 만듭니다.

<br/>

### 3. 학습 흔적의 정량화
 - **이중 DB 책임 분리**: 트랜잭션이 필요한 사용자·통계 데이터는 MySQL(JPA, Hikari pool 8)로, 매일 갈아 끼우는 콘텐츠 스냅샷은 MongoDB로 운영해 서로의 부하를 분리했습니다.
 - **JWT + 자동 재발급**: access/refresh 이중 토큰, 401 발생 시 프론트 Axios 인터셉터가 `/auth/reissue`로 자동 재시도해 사용자가 로그인 끊김을 체감하지 않도록 설계했습니다.
 - **학습 데이터 모델**: 연속 학습일(streak), 5/5 정답 "완벽한 날", 카테고리별 진행률을 `UserStats` · `UserCategoryStats`로 누적해 리포트 화면을 구성합니다.

<br/>
<br/>

## 🛠 실행 및 설치 (Getting Started)

전체 시스템은 Docker Compose를 통해 한 번에 실행할 수 있습니다.

```bash
# 공용 네트워크 생성 (최초 1회)
docker network create oba_shared_network

# 전체 서비스 구동
docker-compose up -d --build
```

**주요 접속 정보:**
- **Frontend**: `http://localhost:8082`
- **Backend API**: `http://localhost:8081`
- **AI Server**: `http://localhost:8000`
- **Airflow**: `http://localhost:8080`
- **Jenkins**: `http://localhost:9090`

<br/>

## 🎓 프로젝트의 기술적 지향점
- **시간 분리 아키텍처**: 사용자 요청 시점이 아닌 새벽 배치에서 LLM 호출을 끝내, 비용·지연을 사용자 동선 밖으로 밀어냈습니다.
- **DB 책임 분리**: 트랜잭션 데이터(MySQL)와 콘텐츠 스냅샷(MongoDB)을 한 백엔드에서 명확히 책임 분리해 운영했습니다.
- **One Codebase, Three Platforms**: Expo Router로 iOS · Android · Web을 단일 코드베이스로 유지해 학습 프로젝트 규모에 맞는 합리적 선택을 했습니다.
- **운영 가시성**: Jenkins 빌드 실패 시 `analyze_log.py`가 로그 100줄을 요약해 Mattermost로 자동 알림합니다.

<br/>

## 📑 개발 문서 목록
> 한입 기사의 기획·설계·운영 문서는 팀 노션에서 관리하고 있습니다. 아래 표에서 각 문서로 바로 이동할 수 있습니다.

| 문서 | 링크 |
| :--- | :--- |
| 기술개발 문서 | [Notion](#) <!-- TODO: 링크 --> |
| 요구사항 정의서 | [Notion](#) <!-- TODO: 링크 --> |
| Git 컨벤션 | [Notion](#) <!-- TODO: 링크 --> |
| Jira 컨벤션 | [Notion](#) <!-- TODO: 링크 --> |
<!-- TODO: 필요한 문서 행 추가 -->

## 💁‍♂️ 프로젝트 팀원
<!-- TODO: 권혁준 외 4명의 GitHub 핸들과 역할 입력 -->
| **[권혁준](https://github.com/yuwolx)** | **변지훈** | **박사랑** | **임선우** | **김휘민** |
|:---:|:---:|:---:|:---:|:---:|
| ![](https://github.com/yuwolx.png?width=120&height=120) | ![](#) | ![](#) | ![](#) | ![](#) |
| Frontend · PM | <!-- TODO: 역할 --> | <!-- TODO: 역할 --> | <!-- TODO: 역할 --> | <!-- TODO: 역할 --> |
