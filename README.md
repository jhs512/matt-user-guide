# Matt Pocock 스킬 튜토리얼 — 공지사항 게시판을 배포까지

**설치 → `/setup-matt-pocock-skills` → `/grill-with-docs` → `/to-spec` → `/to-tickets` → `/implement`로 전체 구현**을 진행합니다. 하나의 공지사항 게시판을 설계하고, Kotlin/Spring 백엔드와 React 프론트엔드를 만들어 GitHub Actions로 배포하는 실습입니다.

각 강에는 **복사용 입력 → 예상 결과물 → 사용 전후 차이 → 완료 확인**이 있습니다.

> 이 문서는 실습 시나리오입니다. 파일 내용·대화·화면은 예상 예시이며, 실제 앱 구현·테스트·배포 로그가 아닙니다. 확인 기준은 2026-09-06의 공식 문서와 현재 설치된 스킬입니다. `/...`는 에이전트 대화창의 스킬 호출 표기이며 터미널 명령이 아닙니다. 사용 도구에서 해당 스킬을 선택해 호출하세요.

## 만들 제품: 작은 공지사항 게시판

방문자는 로그인 없이 공지 목록과 상세를 읽습니다. 관리자 한 명은 로그인해 공지를 등록·수정·삭제합니다. 작성 즉시 공개되며, 초안·첨부파일·댓글·회원가입은 이번 범위에서 제외합니다.

이 기능 범위와 아래 세부 동작은 **튜토리얼의 제안값**입니다. 3강의 인터뷰에서 확인하고 확정하는 흐름으로 사용합니다.

```text
방문자 화면                         관리자 화면
┌──────────────────────────┐        ┌──────────────────────────┐
│ 공지사항       [관리자 로그인]│     │ 공지사항 [새 공지] [로그아웃]│
│ 서비스 점검 안내  09.06   │        │ 서비스 점검 안내 [수정][삭제]│
│ 이용 안내         09.05   │        │ 이용 안내        [수정][삭제]│
│         [이전] 1 [다음]   │        │         [이전] 1 [다음]   │
└──────────────────────────┘        └──────────────────────────┘
```

### 기술 구성

| 영역 | 실습 기준 |
| --- | --- |
| 백엔드 | Kotlin + Spring Boot, Gradle Kotlin DSL, Spring MVC |
| 영속성 | Spring Data JPA, Hibernate, Flyway |
| 인증·인가 | Spring Security, 관리자 로그인, 쓰기 API 권한 검사 |
| 로컬 DB | H2 파일 DB: 서버 재시작 후에도 개발 데이터 유지 |
| 테스트 DB | H2 인메모리: 테스트마다 데이터 격리 |
| 운영 DB | Railway PostgreSQL |
| 프론트 | Vite + React + TypeScript + shadcn/ui + Tailwind CSS |
| 백엔드 배포 | Railway API 서비스 |
| 프론트 배포 | Cloudflare Pages, GitHub Actions에서 빌드 후 업로드 |
| CI/CD | GitHub Actions: PR 검증, main 반영 후 검증·배포 |
| 코드 구성 | 한 저장소의 `backend/`, `frontend/`; 하나의 공지 도메인 |

현재 공식 문서와 Spring Initializr에서 확인한 최신 안정 Spring Boot는 **4.1.1**입니다. 실습 JDK는 **21**로 통일합니다. Kotlin·Gradle은 해당 Boot 버전을 선택한 Initializr 생성 결과를 기준으로 고정하고, JPA·Security는 Boot의 의존성 관리를 따릅니다. [Spring 요구사항](https://docs.spring.io/spring-boot/system-requirements.html), [Kotlin 지원](https://docs.spring.io/spring-boot/reference/features/kotlin.html)

‘최신’은 최초 생성 시점에 확인합니다. 이후 CI에서는 고정된 플러그인 버전, Gradle Wrapper와 npm lockfile을 사용합니다. 프론트도 처음 설치할 때 호환되는 안정 버전을 선택하고 확정된 버전을 기록합니다.

### 목표 배포 구조

```text
브라우저
   │ 정적 파일                     │ HTTPS API 요청
   ▼                               ▼
Cloudflare Pages              Railway: Kotlin/Spring
React + shadcn/ui                   │ JPA
                                   ▼
                              PostgreSQL

GitHub PR ──► 백엔드 H2 테스트 + 프론트 검사 + 빌드
GitHub main push ──► 같은 검증 ──► Railway 배포·준비 확인
                               ──► Pages 배포 ──► 공개 URL 확인
```

Pages는 **Direct Upload 프로젝트**를 만들고 Actions에서 Wrangler로 자동 배포합니다. 서비스의 Git 연동 자동 배포를 동시에 켜지 않아, 검증 전 배포되거나 같은 커밋이 중복 배포되는 일을 피합니다. [Pages Direct Upload](https://developers.cloudflare.com/pages/get-started/direct-upload/)

## 수업 지도

| 강 | 호출 / 작업 | 강이 끝나면 보이는 결과 |
| --- | --- | --- |
| 1강 | 설치 | 필요한 스킬 목록 |
| 2강 | `/setup-matt-pocock-skills` | 프로젝트 지침·설정 문서 |
| 3강 | `/grill-with-docs` | 합의된 동작, 용어집, 필요한 ADR |
| 4강 | `/to-spec` | 기능·권한·DB·배포·검증 기준을 담은 명세 |
| 5강 | `/to-tickets` | 시연 가능한 기능별 티켓과 의존성 |
| 6강 | `/implement` | 앱 코드·테스트·워크플로우·배포 확인 기록 |

## 1강. 설치

### 준비

Git, JDK 21, Node.js LTS와 npm, 코딩 에이전트를 준비합니다. 프론트 생성 시 선택된 Vite 버전의 Node 요구사항을 확인하고, 로컬과 CI에서 같은 Node 메이저 버전을 사용합니다.

터미널에서 확인합니다.

```powershell
java -version
node --version
npm --version
git --version
```

새 실습 폴더가 필요한 경우에만 다음을 실행합니다.

```powershell
mkdir notice-board
cd notice-board
git init
```

### 설치: 터미널 입력

```powershell
npx skills@latest add mattpocock/skills
```

설치기에서 사용할 에이전트와 프로젝트 설치 범위를 선택합니다. 이 과정에서 사용할 스킬은 다음과 같습니다. [공식 설치 안내](https://github.com/mattpocock/skills)

| 스킬 | 역할 |
| --- | --- |
| `setup-matt-pocock-skills` | 저장소별 작업 규칙 |
| `grill-with-docs` | 질문과 문서화를 연결 |
| `grilling`, `domain-modeling` | 인터뷰, 용어집과 설계 결정 기록 |
| `to-spec`, `to-tickets` | 명세 작성, 구현 단위 분해 |
| `implement`, `tdd`, `code-review` | 구현, 검증, 리뷰 |
| `triage` | 이번 실습의 기본 분류 라벨 |

`grill-with-docs`는 설치된 원문에서 `grilling`과 `domain-modeling`을 함께 호출합니다. 세 스킬이 모두 설치되어 있는지 확인합니다.

### 사용 전 → 후

```text
전: /grill-with-docs, /to-spec을 찾을 수 없음
후: 설치 목록에서 위 스킬들과 각 SKILL.md를 확인할 수 있음
```

**완료 확인:** 설치기가 출력한 경로와 에이전트의 스킬 목록을 확인합니다. 인식되지 않으면 설치 대상·범위를 확인하고 새 대화에서 다시 선택합니다. 아직 앱 파일은 생기지 않습니다.

## 2강. /setup-matt-pocock-skills — 저장 규칙 정하기

튜토리얼의 명세와 티켓은 로컬 Markdown으로 기록합니다. **코드는 GitHub에서 관리하고 CI/CD를 실행하면서, 작업 티켓은 저장소 안의 파일로 관리할 수 있습니다.** GitHub Issues가 필수인 것은 아닙니다.

### 입력

```text
/setup-matt-pocock-skills

공지사항 게시판 실습 저장소를 설정해줘.
이슈와 명세는 로컬 Markdown, triage는 기본 라벨을 사용해.
backend/와 frontend/를 둘 예정이지만 도메인은 공지사항 하나이므로
단일 CONTEXT.md와 docs/adr/를 사용하는 구성을 원해.
AGENTS.md와 CLAUDE.md가 모두 없다면 AGENTS.md를 생성해줘.
초안을 보여줘.
```

초안을 확인한 뒤 답합니다.

```text
초안대로 작성해줘.
```

### 예상 결과물

```text
notice-board/
├── AGENTS.md
└── docs/agents/
    ├── issue-tracker.md
    ├── triage-labels.md
    └── domain.md
```

기존 `CLAUDE.md`가 있으면 그것을, 없으면 기존 `AGENTS.md`를 편집하는 것이 스킬의 선택 규칙입니다. 위 트리는 두 파일이 없었던 실습 폴더 기준입니다.

| 파일 | 확정되는 내용 |
| --- | --- |
| 프로젝트 지침 | 어떤 상황에 설정 문서를 읽어야 하는지 |
| issue-tracker.md | `.scratch/<기능>/spec.md`, `issues/<번호>-<이름>.md` |
| triage-labels.md | `needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix` |
| domain.md | 용어집·ADR의 위치와 읽는 규칙 |

### 사용 전 → 후

```diff
- 명세를 어디에 저장할지 매번 설명해야 함
+ .scratch/notice-board/spec.md로 연결되는 저장 규칙이 있음
- 도메인 문서 위치가 정해지지 않음
+ CONTEXT.md와 docs/adr/를 읽는 규칙이 있음
```

**완료 확인:** 설정 문서 세 개와 프로젝트 지침의 참조가 일치하는지 확인합니다. `CONTEXT.md` 자체는 의미 있는 용어가 정리되는 다음 강에서 생성합니다.

## 3강. /grill-with-docs — 모호한 요구를 질문으로 정제하기

이번에 추가한 핵심 단계입니다. ‘시큐리티를 쓴다’는 말만으로는 누가 무엇을 할 수 있는지, 로그인은 얼마나 유지되는지 결정되지 않습니다. 구현 전에 실제 사용 상황을 질문하고 답을 문서로 남깁니다.

### 입력

```text
/grill-with-docs

간단한 공지사항 게시판을 만들고 싶어.
방문자는 목록과 상세를 보고, 관리자 한 명만 등록·수정·삭제해.
회원가입, 댓글, 첨부파일, 검색, 초안, 상단 고정은 이번 범위에서 빼자.

백엔드는 최신 안정 Spring Boot + Kotlin, JPA, Spring Security야.
로컬 DB는 H2 파일, 테스트 DB는 H2 인메모리, 운영 DB는 PostgreSQL.
백엔드는 Railway, 프론트는 Vite React TypeScript shadcn/ui야.
프론트는 Cloudflare Pages로 배포하고 GitHub Actions가 CI/CD를 담당해.
backend/와 frontend/를 같은 저장소에 둬.

아직 구현하지 말고 권한, 인증 유지, 입력 제한, 오류,
페이지네이션, DB 전환, 자동배포 완료 기준을 질문해서 정제해줘.
확정된 도메인 용어는 CONTEXT.md에 기록하고,
대안과 비용이 있는 중요한 결정만 ADR로 제안해줘.
버전·플랫폼 제약은 공식 자료를 직접 확인해줘.
```

### 예상 인터뷰: 한 번에 모든 답을 추측하지 않기

| 라운드 | 에이전트가 확인할 질문 예시 | 튜토리얼에서 선택할 답 |
| --- | --- | --- |
| 1 | 공지는 저장 즉시 공개되나요? 삭제는 복구되나요? | 즉시 공개, 확인 후 영구 삭제 |
| 1 | 제목·본문 제한과 정렬 기준은 무엇인가요? | 제목 1~100자, 본문 1~10,000자; 앞뒤 공백 제거; 최신순 |
| 1 | 한 페이지 크기와 빈 화면은 어떻게 하나요? | 10개 고정, 없으면 안내; 마지막 항목 삭제 시 이전 페이지로 |
| 2 | 관리자 로그인을 새로고침 후에도 유지해야 하나요? | 실습에서는 다시 로그인해도 됨 |
| 2 | 프론트·API 주소가 다를 때 인증은 어떻게 할까요? | 짧은 Bearer 토큰, 쿠키 인증 없이 메모리에 보관 |
| 3 | 토큰 만료·로그아웃은 어떤 의미인가요? | 30분 만료, refresh 없음; 로그아웃은 브라우저 토큰 제거 |
| 3 | CI와 실제 서비스 확인 중 어디까지가 완료인가요? | 검증 통과, 배포된 커밋 확인, 운영 화면·DB 저장 확인 |

실제 질문은 앞선 답에 따라 달라집니다. 예를 들어 토큰 방식을 선택하기 전에는 토큰 만료 시간을 확정하지 않습니다.

### 정제된 실습 조건

| 주제 | 결정 예시 |
| --- | --- |
| 공지 | 제목·일반 텍스트 본문·생성일·수정일; HTML을 실행하지 않음 |
| 목록 | 생성일 내림차순, 동률은 ID 내림차순, 10개씩 페이지 이동 |
| 작성 | 관리자만 가능; 저장 성공 후 상세 화면으로 이동 |
| 수정 | 생성일과 목록 순서는 유지; 수정일 갱신 |
| 삭제 | 확인창에서 승인 후 삭제, 목록 복귀; 없는 공지는 404 |
| 오류 | 입력 오류 400, 인증 오류 401, 권한 부족 403, 대상 없음 404 |
| 관리자 | 운영 환경변수의 사용자명·비밀번호 해시 사용; 회원가입 없음 |
| 인증 | 앱이 로그인 성공 시 서명한 access JWT 발급; Spring Security로 검증 |
| 토큰 | 30분, 프론트 메모리에만 저장; 새로고침·만료 시 재로그인 |
| 로그아웃 | 현재 화면에서 토큰 제거; 이미 발급된 토큰은 만료 전까지 유효 |
| 운영 연결 | 허용된 프론트 Origin만 CORS 허용; Authorization 헤더 허용 |

이 인증 구성은 단순 실습을 위한 **설계 선택**입니다. Spring Security의 JWT 검증 기능이 로그인·토큰 발급 API까지 자동으로 만들어주지는 않습니다. 발급과 검증, 관리자 권한 매핑, 서명·만료·issuer·audience 확인을 명세에 포함합니다. [JWT 검증 공식 문서](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html)

쿠키·세션·HTTP Basic 인증을 사용하지 않는 stateless API로 구성합니다. 이 전제에서 CSRF 비활성화를 검토하고 이유를 기록합니다. 향후 쿠키 인증을 도입하면 CSRF 정책도 다시 설계합니다. CORS는 인증이나 권한 검사를 대신하지 않습니다.

### 예상 결과물: 용어집과 결정 기록

```text
새로 생기는 문서
├── CONTEXT.md
└── docs/adr/
    ├── 0001-admin-authentication.md
    └── 0002-ci-controlled-deployment.md
```

`CONTEXT.md` 내용 예시:

```markdown
# Domain glossary

- 공지(Notice): 관리자가 게시하며 방문자가 읽을 수 있는 안내문.
- 방문자(Visitor): 로그인 없이 공지를 읽는 사람.
- 관리자(Administrator): 공지를 등록·수정·삭제할 권한이 있는 사람.
```

용어집에는 Kotlin 버전·DB URL·API 구현 방법을 섞지 않습니다. ADR은 모든 기술 선택마다 만들지 않고, 대안·이유·후속 비용을 기록할 가치가 있는 결정만 남깁니다.

인증 ADR 발췌 예시:

```markdown
# 관리자 인증 방식

상태: 수락
맥락: 서로 다른 서비스 도메인에서 운영하며 관리자 한 명만 사용한다.
결정: 짧은 Bearer 토큰을 프론트 메모리에 보관한다.
대안: 쿠키 세션, 외부 인증 제공자.
결과: 새로고침 시 재로그인한다. 즉시 토큰 폐기는 이번 범위에 없다.
```

### 사용 전 → 후

```diff
- 시큐리티 넣어줘
+ 방문자는 조회만, 관리자는 쓰기 가능; 서버에서 권한 검사
- 로그인 기능
+ 30분 토큰, 메모리 보관, 새로고침 시 재로그인
- 자동배포
+ main 검증 통과 → API 배포 확인 → Pages 배포 → 운영 동작 확인
```

**완료 확인:** 에이전트의 최종 합의 요약을 읽고 “이 조건으로 합의했어. 문서에 반영해줘”라고 답합니다. 아직 앱 구현은 시작하지 않습니다.

## 4강. /to-spec — 합의 내용을 명세로 고정하기

`/grill-with-docs`에서 결정한 내용을 `/to-spec`으로 합성합니다. 이 단계는 새 요구사항 인터뷰보다 합의된 내용을 빠짐없이 옮기는 데 집중합니다. 설치된 스킬은 테스트 경계를 사용자에게 확인하는 단계도 포함합니다.

### 입력

```text
/to-spec

앞서 합의한 공지사항 게시판을 명세로 작성해줘.
CONTEXT.md와 관련 ADR을 읽고 .scratch/notice-board/spec.md에 저장해.
기능, API 계약, 관리자 인증, 환경별 DB, CI/CD 완료 기준을 포함해.
구현 파일 목록이나 코드를 명세 대신 나열하지 마.

테스트 경계는 먼저 제안해줘.
백엔드는 Security 필터를 포함한 HTTP 통합 테스트와 H2 인메모리,
프론트는 사용자 화면 테스트,
최종 연결은 실제 백엔드+H2로 브라우저 E2E를 원해.
운영 PostgreSQL의 마이그레이션·저장 확인은 별도 배포 검증으로 구분해.
```

예상 확인에 답합니다.

```text
그 테스트 경계로 진행해. 운영 DB 확인을 H2 테스트 통과로 대체하지 말고
별도 완료 항목으로 남겨줘.
```

### 결과물 A: 명세의 구조

```markdown
# 공지사항 게시판
Status: ready-for-agent

## Problem Statement
방문자는 최신 안내를 읽고 관리자는 개발자 도움 없이 안내를 관리해야 한다.

## Solution
공개 읽기 화면과 관리자 쓰기 기능을 가진 게시판을 제공한다.

## User Stories
1. 방문자는 최신순 목록에서 공지를 선택할 수 있다.
2. 방문자는 제목과 본문, 작성·수정 시점을 확인할 수 있다.
3. 관리자는 로그인해 공지를 등록할 수 있다.
4. 관리자는 내용을 수정하고 삭제할 수 있다.
5. 관리자는 입력 오류와 인증 만료의 원인을 확인할 수 있다.
6. 운영자는 검증된 main 변경을 자동 배포할 수 있다.

## Implementation Decisions
합의한 권한·API·저장·환경·배포 계약을 기록한다.

## Testing Decisions
HTTP/H2, 화면, H2 E2E, 운영 PG 검증의 범위를 구분한다.

## Out of Scope
회원가입, 댓글, 첨부, 검색, 초안, 상단 고정, refresh token.

## Further Notes
버전과 외부 서비스의 실제 설정값은 구현·배포 문서에 기록한다.
```

발췌 예시입니다. 실제 명세는 페이지 끝 삭제, 만료, 중복 제출, 없는 공지, 실패 중 입력 보존 등 합의한 사례까지 사용자 스토리와 검증 조건으로 포함합니다.

### 결과물 B: API 계약 예시

| 메서드 | 경로 | 권한 | 성공 결과 |
| --- | --- | --- | --- |
| POST | `/api/auth/login` | 공개 | 200, accessToken·expiresIn |
| GET | `/api/notices?page=0` | 공개 | 200, items·page·size·totalElements·totalPages |
| GET | `/api/notices/{id}` | 공개 | 200, 공지 상세 |
| POST | `/api/notices` | ADMIN | 201, 생성된 공지와 Location |
| PUT | `/api/notices/{id}` | ADMIN | 200, 수정된 공지 |
| DELETE | `/api/notices/{id}` | ADMIN | 204, 본문 없음 |

목록의 API 페이지 번호는 0부터, 화면 표시는 1부터 시작합니다. `page < 0`은 400, 범위를 초과한 페이지는 빈 items와 전체 개수를 반환합니다. 날짜는 UTC ISO 8601로 전달하고 화면에서 사용자 시간대로 표시합니다.

### 결과물 C: 환경별 DB 계약

| 환경 | 프로필 | DB 예시 | 스키마 정책 |
| --- | --- | --- | --- |
| 로컬 | local | `jdbc:h2:file:./data/notice-board` | Flyway 적용 + JPA validate |
| 테스트 | test | `jdbc:h2:mem:<test-id>` | 테스트 시작 시 Flyway 적용 + JPA validate, 데이터 격리 |
| 운영 | prod | `jdbc:postgresql://<host>:<port>/<db>` | Flyway 적용 + JPA validate |

H2와 PG에서 동작하는 스키마 변경을 우선 사용합니다. 차이가 필요한 SQL은 DB별 마이그레이션을 명시적으로 구분합니다. 운영에서는 `create`·`create-drop`·자동 `update`로 스키마를 관리하지 않습니다. H2의 PG 호환 모드를 사용하더라도 PG와 동일하다는 보장은 없습니다.

### 사용 전 → 후

```text
전: 대화 + 용어집 + 설계 결정
후: 위 자료에 연결된 spec.md
    ├── 누가 무엇을 할 수 있는가
    ├── 요청과 응답은 무엇인가
    ├── 실패하면 어떻게 보이는가
    └── 어느 검증까지 끝나야 완료인가
```

**완료 확인:** 대화의 결정과 명세를 대조합니다. H2 파일/메모리/PG의 구분, 쓰기 권한, 배포 완료 조건이 빠지지 않아야 합니다.

## 5강. /to-tickets — 시연 가능한 기능 단위로 나누기

### 입력

```text
/to-tickets

.scratch/notice-board/spec.md를 티켓으로 분해해줘.
UI → API → DB → 테스트가 연결되는 작은 기능 단위로 나눠줘.
각 티켓에 선행 티켓, 수용 기준, 완료 후 시연 방법을 넣어.
CI와 운영 배포도 명세 범위에 포함해.
파일을 만들기 전에 제안한 분할과 의존성을 보여줘.
```

### 예상 분할

| 번호 | 티켓 | 선행 | 완료 후 눈으로 볼 결과 |
| --- | --- | --- | --- |
| 01 | 공개 목록·상세 읽기 | 없음 | local 전용 예제 공지를 UI에서 조회, H2 API 테스트 통과 |
| 02 | 관리자 로그인·공지 등록 | 01 | 로그인 후 작성, 새 공지가 공개 목록에 표시 |
| 03 | 공지 수정·삭제 | 02 | 본문 변경 반영, 삭제 후 목록에서 사라짐 |
| 04 | PR 품질 검증 자동화 | 03 | GitHub PR에서 백엔드·프론트·E2E 검사 결과 표시 |
| 05 | 운영 DB·자동배포 | 04 | Railway API + PG, Pages UI, main 자동배포 결과 |

01에 프로젝트 초기 구성과 shadcn/ui 설정도 포함합니다. 읽기 기능부터 연결하며, 예제 데이터는 local 프로필 전용으로 운영에 들어가지 않습니다. 04·05는 인프라 티켓이지만 완료 조건이 ‘파일 생성’이 아니라 관찰 가능한 검증·배포 결과입니다. 작은 실습이라 선형 순서를 택한 것이며 항상 순차로 분해해야 하는 것은 아닙니다.

```text
이 분할과 의존성으로 진행해.
.scratch/notice-board/issues/ 아래 티켓당 한 파일로 작성해줘.
```

### 예상 파일 트리

```text
.scratch/notice-board/
├── spec.md
└── issues/
    ├── 01-read-notices.md
    ├── 02-login-and-create.md
    ├── 03-update-and-delete.md
    ├── 04-ci.md
    └── 05-production-deploy.md
```

### 티켓 발췌: 등록 기능

```markdown
# 02: 관리자 로그인과 공지 등록

**What to build:** 관리자가 로그인해 공지를 작성하면 방문자가 읽을 수 있다.
**Blocked by:** 01: 공개 목록·상세 읽기
**Status:** ready-for-agent

- [ ] 잘못된 로그인은 401이며 입력 화면에 오류가 표시된다.
- [ ] 관리자는 제목과 본문을 입력하고 저장할 수 있다.
- [ ] 제목·본문 제한을 API와 화면에서 검증한다.
- [ ] 저장 성공 시 상세로 이동하고 공개 목록에 표시된다.
- [ ] 미인증 쓰기는 401, 인증된 비관리자의 쓰기는 403이다.
- [ ] 만료된 토큰은 재로그인 안내로 이어진다.
- [ ] 저장 실패 시 입력을 보존하고 재시도할 수 있다.
- [ ] H2 통합 테스트와 화면 테스트로 위 동작을 확인한다.
```

공개 계정 생성 기능은 없지만, 인가 테스트에서는 관리자 권한이 없는 인증 사용자를 구성해 403을 확인할 수 있습니다.

### 사용 전 → 후

```text
전: spec.md 하나, 어디부터 구현할지 불분명
후: 01 읽기 → 02 로그인·등록 → 03 수정·삭제 → 04 CI → 05 배포
    각 단계에 파일·수용 기준·시연 방법이 있음
```

**완료 확인:** 모든 명세 조건이 티켓에 연결되어 있는지 확인합니다. 선행 관계에 순환이 없어야 하고, 배포 티켓에는 필요한 계정·변수·검증 방법까지 있어야 합니다.

## 6강. /implement — 모든 티켓 구현하고 배포 결과 확인하기

`/implement-all`이라는 별도 스킬 대신 `/implement`에 **모든 티켓**을 지정합니다. `/implement-spec`은 별도 worktree·하위 에이전트·PR 중심의 흐름이므로 이 실습의 기본 호출과 구분합니다.

### 입력

```text
/implement

.scratch/notice-board/spec.md와 issues/의 모든 티켓을 구현해줘.
01 → 02 → 03 → 04 → 05 순서로 진행하고,
각 티켓의 수용 기준을 검증한 뒤 다음으로 넘어가.

backend/는 Spring Initializr의 최신 안정 Boot + Kotlin 조합을 확인해
JDK 21, Gradle Kotlin DSL, MVC, JPA, Security로 생성해.
Spring Boot, Kotlin, Gradle 버전을 고정하고 docs/versions.md에 기록해.
Spring용 Kotlin 플러그인, JPA 엔티티용 no-arg/open 설정도 검증해.
local=H2 파일, test=H2 메모리, prod=PG를 명시적으로 분리해.
Flyway 적용과 JPA validate를 사용해.

frontend/는 Vite React TypeScript + shadcn/ui로 구성해.
등록/로그인/수정/삭제/페이지 이동을 실제 API에 연결해.
API 주소는 VITE_API_BASE_URL로 빌드할 때 주입해.

합의한 경계에서 /tdd를 적용해.
백엔드 Security 포함 HTTP 통합 테스트는 H2 메모리를 사용하고,
프론트 화면 테스트와 실제 백엔드+H2의 브라우저 E2E도 작성해.
frontend에 lint, typecheck, test, test:e2e, build, dev 스크립트를 제공해.
test는 한 번 실행 후 종료하고 E2E용 서버 시작·종료를 자동화해.

GitHub Actions에서 PR과 main을 검증해.
main 검증이 통과한 동일 커밋만 Railway와 Cloudflare Pages에 배포해.
Railway 준비 완료와 배포 커밋을 확인한 다음 Pages 배포를 진행해.
외부 서비스 초기 설정은 docs/deployment.md에 단계별로 작성해.
설정값이 없으면 가능한 코드와 워크플로우부터 완성하고,
실제 배포에 필요한 누락값을 정확히 보고해.

/code-review로 변경을 검토하고 문제를 수정해.
실제 검증을 통과한 티켓만 체크하고 Comments에 근거를 기록해.
이번 실습의 완료 티켓 Status는 done으로 사용해.
커밋은 현재 브랜치에 남기고, 푸시·최초 운영 배포는 설정 확인 후 진행해.
실행하지 않은 테스트나 배포를 완료로 보고하지 마.
```

생성 순서의 기준은 [Spring Initializr](https://start.spring.io/)와 [shadcn/ui Vite 안내](https://ui.shadcn.com/docs/installation/vite)입니다. 프론트는 비어 있는 `frontend/`에서 `npx shadcn@latest init -t vite`를 사용하는 방식을 선택할 수 있습니다. Vite를 먼저 생성했다면 기존 프로젝트 설치 절차를 따릅니다.

### 결과물 A: 프로젝트 트리 예시

```text
notice-board/
├── AGENTS.md
├── CONTEXT.md
├── .scratch/notice-board/...
├── docs/
│   ├── agents/...
│   ├── adr/...
│   ├── versions.md
│   └── deployment.md
├── backend/
│   ├── build.gradle.kts
│   ├── gradlew / gradlew.bat
│   ├── gradle/wrapper/...
│   ├── Dockerfile
│   └── src/
│       ├── main/kotlin/...          # API, 도메인, 저장, 인증
│       ├── main/resources/         # local/prod 설정, 마이그레이션
│       └── test/                   # H2 test 설정과 통합 테스트
├── frontend/
│   ├── package.json
│   ├── package-lock.json
│   ├── vite.config.ts
│   ├── playwright.config.ts
│   └── src/                       # 화면, API 연결, shadcn 컴포넌트
└── .github/workflows/
    └── ci-cd.yml
```

### 결과물 B: 화면 변화

```text
01 이후: 공지 목록 → 클릭하면 상세, 페이지 이동
02 이후: 관리자 로그인 → 작성 → 새 공지가 목록에 추가
03 이후: 수정 → 상세 반영 / 삭제 확인 → 목록에서 제거
04 이후: PR 화면에 테스트·타입 검사·빌드 결과
05 이후: Pages 공개 URL에서 같은 흐름 + Railway PG에 실제 저장
```

### 로컬에서 직접 확인

백엔드 폴더의 PowerShell 터미널:

```powershell
.\gradlew.bat bootRun --args='--spring.profiles.active=local'
```

프론트 폴더의 다른 터미널:

```powershell
npm ci
npm run dev
```

개발용 관리자 자격 증명과 API 주소는 구현 시 생성한 `.env.example`·배포 문서에 따라 설정합니다. 실제 비밀번호를 저장소에 커밋하지 않습니다.

| 직접 조작 | 확인할 결과 |
| --- | --- |
| 로그아웃 상태에서 목록·상세 열기 | 공개 조회 가능, 작성 버튼 없음 |
| 인증 없이 쓰기 API 호출 | 401; 버튼을 숨기는 것과 별개로 서버에서 차단 |
| 관리자 로그인 후 작성 | 상세로 이동, 목록에도 새 공지 표시 |
| 잘못된 제목 제출 | 필드 오류 표시, DB에 등록되지 않음 |
| 수정 후 새로고침 | 수정 결과 유지, 로그인은 해제 |
| 다시 로그인하여 삭제 | 확인 후 삭제, 상세 재조회 404 |
| 로컬 백엔드 재시작 | H2 파일의 공지 유지 |

## 6강의 배포 실습 — GitHub Actions가 전체 흐름을 제어하기

### 최초 한 번 준비할 것

| 위치 | 설정 |
| --- | --- |
| GitHub | 저장소, main 브랜치, Actions 사용, production 환경 |
| Railway | 프로젝트, API 서비스, PostgreSQL 서비스, API 공개 HTTPS 주소 |
| Railway API | backend 루트/Dockerfile 경로, prod 환경변수, 헬스체크 경로 |
| Cloudflare | Pages Direct Upload 프로젝트, production branch=main |
| Actions | 아래 Secrets·Variables, 고정된 CLI/Action 버전 |

Railway API는 플랫폼의 `PORT`를 사용하도록 `server.port=${PORT:8080}`을 설정합니다. Railway의 PostgreSQL 연결 변수를 **JDBC URL·사용자·비밀번호**에 맞게 매핑합니다. 일반 `postgresql://...` URL을 JDBC URL 자리에 그대로 넣지 않습니다.

### 환경변수의 역할 구분

| 보관 위치 | 이름 예시 | 용도 |
| --- | --- | --- |
| GitHub Secret | RAILWAY_TOKEN | 프로젝트·환경 범위의 배포 토큰 |
| GitHub Secret | CLOUDFLARE_API_TOKEN | 해당 계정 Pages 편집 권한 |
| GitHub Variable | RAILWAY_SERVICE_ID | 배포할 API 서비스 |
| GitHub Variable | CLOUDFLARE_ACCOUNT_ID, CLOUDFLARE_PAGES_PROJECT | Pages 배포 대상 |
| GitHub Variable | VITE_API_BASE_URL | Railway의 공개 API URL, 프론트 빌드에 주입 |
| Railway Variable | SPRING_PROFILES_ACTIVE=prod | 운영 프로필 |
| Railway Variable | SPRING_DATASOURCE_URL / USERNAME / PASSWORD | PG 연결, Railway 변수 참조 사용 |
| Railway Variable | ADMIN_USERNAME, ADMIN_PASSWORD_HASH | 관리자 검증 정보 |
| Railway Variable | JWT_SECRET | 운영 서명 키, 코드 기본값 없이 설정 |
| Railway Variable | CORS_ALLOWED_ORIGINS | 실제 Pages production Origin |

`VITE_` 값은 브라우저 번들에 포함되므로 공개 API 주소만 넣습니다. DB 비밀번호와 JWT 키를 넣으면 안 됩니다. Pages에 런타임 변수만 바꿔서는 이미 만들어진 JS의 API 주소가 바뀌지 않으므로 **Actions 빌드 단계**에 주입합니다. [Vite 환경변수](https://vite.dev/guide/env-and-mode)

### 생성할 워크플로우의 계약

아래는 구현 시 생성할 `ci-cd.yml`의 **흐름 명세**입니다. 완성된 실행용 YAML로 오해하지 않도록 잡 관계로 표시합니다.

```text
pull_request / push(main)
        │
        ├─ backend-check
        │    JDK 21 + Gradle Wrapper
        │    test 프로필 H2 메모리 테스트 + bootJar
        │
        └─ frontend-check
             고정 Node + npm ci
             lint + typecheck + 화면 테스트 + build
        │
        ▼
      e2e-check
        실제 API(test/H2 메모리) + 프론트 + Playwright
        CI 전용 관리자·서명 키 사용; 운영 자격 증명 접근 없음
        │
        ▼ push(main)에서만, production 환경
      deploy-backend
        같은 SHA의 backend를 Railway에 배포
        배포 ID/SHA와 해당 배포의 준비 완료 확인
        │
        ▼ 성공한 경우만
      deploy-frontend
        같은 SHA의 frontend 빌드, VITE_API_BASE_URL 주입
        Wrangler로 dist를 Pages production에 업로드
        │
        ▼
      smoke-check
        공개 UI/API, 로그인·공지 저장·수정·삭제 확인
        각 자동/수동 검증의 증거와 미실행 항목 기록
```

backend와 frontend 검증이 끝나기 전에는 배포하지 않습니다. PR에는 운영 Secret을 제공하지 않습니다. 배포는 하나의 production concurrency 그룹으로 직렬화하고 진행 중인 배포를 중간 취소하지 않도록 구성합니다. GitHub의 대기 중 실행 교체 정책에 따라 중간 커밋 배포가 생략될 수 있으며, 실제 배포된 SHA를 기록합니다.

Railway 배포 명령과 인증은 [Railway CLI](https://docs.railway.com/cli), [railway up](https://docs.railway.com/cli/up)을 기준으로 구현합니다. `RAILWAY_TOKEN`으로 인증하고 서비스 대상을 명시합니다. CLI 종료만으로 앱 정상 기동을 판단하지 않습니다.

Railway는 DB 연결까지 확인하는 health endpoint를 배포 준비 조건에 연결하고 세부 정보는 노출하지 않습니다. 기존 버전의 health 200을 새 버전 성공으로 착각하지 않도록 **이번 배포 ID·커밋**과 함께 확인합니다. [Railway healthchecks](https://docs.railway.com/deployments/healthchecks)

Pages 업로드의 핵심 명령은 다음 형태입니다. `frontend/`에서 실행하며, 프로젝트 값과 인증 환경변수는 Actions가 제공합니다. CLI 버전은 구현 시 고정합니다. [Pages CI 안내](https://developers.cloudflare.com/pages/how-to/use-direct-upload-with-continuous-integration/)

```bash
npx wrangler pages deploy dist --project-name="$CLOUDFLARE_PAGES_PROJECT" --branch=main
```

외부 서비스 자체의 Git 자동배포는 끄고 위 Actions 경로로 통일합니다. 두 서비스는 하나의 원자적 배포가 아니므로, 백엔드 성공 후 프론트 실패 시 프론트를 재배포할 수 있게 API 하위 호환성을 유지합니다. DB 변경은 이전 앱과도 호환되는 형태부터 적용하며, 파괴적인 마이그레이션을 자동 롤백한다고 가정하지 않습니다.

### H2 테스트와 PG 운영 확인은 서로 다른 결과물

| 확인 | DB | 의미 |
| --- | --- | --- |
| HTTP 통합 테스트 | H2 메모리 | 기능·검증·권한 규칙이 기대대로 동작 |
| 브라우저 E2E | H2 메모리 | 프론트와 백엔드 연결 동작 |
| 로컬 재시작 확인 | H2 파일 | 개발 데이터가 재시작 후 유지 |
| 운영 마이그레이션·CRUD 확인 | PG | 실제 운영 스키마·연결·저장 동작 |

운영 검증에서는 `[배포확인 <커밋>]`처럼 식별 가능한 임시 공지를 생성·수정·삭제합니다. 관리자 인증이 필요한 검증은 배포 담당자가 직접 하거나 별도로 승인한 자동화 자격 증명으로 수행합니다. 수동으로 했다면 워크플로우가 자동 통과한 것으로 쓰지 않고 실행자·시간·결과를 별도로 기록합니다.

### 사용 전 → 후: GitHub 화면과 운영 화면

```text
전
  소스만 있음 / 배포 상태 알 수 없음

후 — 실제 실행 후 채워야 하는 기록
  Commit: <실제 SHA>
  backend-check: <결과 + 실행 링크>
  frontend-check: <결과 + 실행 링크>
  e2e-check: <결과 + 실행 링크>
  Railway deployment: <배포 ID + 준비 상태>
  Pages deployment: <배포 URL + SHA>
  PG CRUD: <자동/수동 구분 + 확인 결과>
```

### 티켓 완료 표기 예시

```diff
- **Status:** ready-for-agent
+ **Status:** done
- - [ ] 공지 등록 후 공개 목록에서 확인된다.
+ - [x] 공지 등록 후 공개 목록에서 확인된다.
+
+ ## Comments
+ 실제 테스트 명령·결과와 해당 변경 커밋을 기록한다.
```

`done`은 이 튜토리얼에서 추가한 완료 표기입니다. 기본 triage 라벨에 자동으로 포함되는 값이 아닙니다. 외부 설정이 없어 배포하지 못했다면 05를 완료로 바꾸지 않습니다.

## 한눈에 보는 스킬 사용 결과의 차이

| 단계 | 입력 | 결과물 | 아직 없는 것 |
| --- | --- | --- | --- |
| 설치 | 설치 명령 | 사용할 수 있는 스킬 | 프로젝트 규칙 |
| `/setup-matt-pocock-skills` | 저장소와 선호 | 작업 저장 규칙 | 제품 요구사항 합의 |
| `/grill-with-docs` | 게시판 아이디어·기술 제약 | 용어·정제된 요구·결정 기록 | 통합된 명세 |
| `/to-spec` | 합의된 대화와 문서 | 검증 가능한 spec.md | 구현 단위 |
| `/to-tickets` | 명세 | 의존성·수용 기준이 있는 티켓 | 실행되는 기능 |
| `/implement` | 전체 티켓 | 코드·테스트·CI/CD·실행 근거 | 미실행 배포는 별도 미완료 |

## 최종 완료 체크

- [ ] 모든 스킬 호출 예시는 `/스킬이름` 형식으로 실행했다.
- [ ] 인터뷰에서 인증·권한·입력·삭제·페이지·배포 기준을 합의했다.
- [ ] 명세와 티켓에 같은 조건이 일관되게 반영됐다.
- [ ] Boot/Kotlin/Gradle/Node/프론트 패키지 버전을 실제 생성 결과로 기록했다.
- [ ] local H2 파일, test H2 메모리, prod PG가 분리됐다.
- [ ] UI와 API에서 읽기·쓰기 권한 및 실패 상황을 확인했다.
- [ ] PR에서 검사하고 main의 검증된 커밋만 배포한다.
- [ ] Railway와 Pages가 같은 커밋 기준이며 공개 주소에서 동작한다.
- [ ] H2 통과와 별개로 PG 마이그레이션·CRUD 결과를 확인했다.
- [ ] 테스트·배포·화면 확인을 실제 수행한 경우에만 완료로 기록했다.

## 참고 기준

스킬 흐름은 현재 설치된 `setup-matt-pocock-skills`, `grill-with-docs`, `grilling`, `domain-modeling`, `to-spec`, `to-tickets`, `implement`, `implement-spec`의 원문을 확인했습니다. 업데이트하면 질문과 동작이 달라질 수 있습니다. 공식 기술 문서는 각 관련 절에 링크했습니다. 본 문서는 실행된 게시판의 결과 보고서가 아니라, 따라 실행하고 결과를 비교하기 위한 튜토리얼입니다.
