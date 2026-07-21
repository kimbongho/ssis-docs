# 관리자 매뉴얼 — qwen-extension (eGovFramework 4.3)

- 작성일: 2026-07-21
- 대상 독자: 확장을 유지보수·배포·트러블슈팅하는 관리자/개발 리드
- 근거 자료: `qwen-extension-egovframework-4.3` 저장소 `README.md`, `commands/guide.md`, `qwen-extension.json` 원문

---

## 1. 확장 구성 개요

| 항목 | 내용 |
|---|---|
| 매니페스트 | `qwen-extension.json` (name=`qwen-extension`, version=`0.1.0`, contextFileName=`QWEN.md`) |
| 소스 저장소 | `git.bluewindsky.com/internal/ssis/qwen/extension/qwen-extension-egovframework-4.3.git` (사내 GitLab) |
| 배포 브랜치 정책 | `main` = 안정 버전(개발자 배포용), `develop` = 다음 릴리스 후보 통합 브랜치(JUnit 자동화·훅 등 실험 기능 우선 반영) |
| 동작 원리 | 셸 스크립트/템플릿 엔진이 아니라 **qwen LLM이 프롬프트(`commands/*.md`) 기반으로 작업을 수행**하고, `hooks/`(Node.js)가 결과를 자동 검증하는 구조 |
| 런타임 요구사항 | Node.js 20 LTS 이상(훅), 단 qwen-code CLI 자체는 Node 22+ 요구 → 실질 기준 **Node 22+** |

디렉터리 구조 및 각 구성요소 역할은 [소프트웨어_납품/소프트웨어_납품명세서.md](../소프트웨어_납품/소프트웨어_납품명세서.md) 4.2절 표를 참고하십시오.

---

## 2. 세션 훅 시스템 (`hooks/hooks.json`)

qwen 세션의 수명주기 각 시점에 자동 검증이 끼워져 있습니다. 등록 순서:

| 시점 | 훅(실행 순서) | 동작 |
|---|---|---|
| UserPromptSubmit | `user-prompt-architect-trigger.mjs`, `cmd-context-stamp.mjs` | 아키텍트 트리거 감지 + 명령 컨텍스트 각인 |
| SessionStart | `session-start-env.mjs` | 세션 시작 시 환경 점검(JDK/Maven 등) |
| PreToolUse(WriteFile\|Edit) | `pre-write-check.mjs`, `pre-write-prod-guard.mjs`, `pre-write-rule-check.mjs`, `pre-write-architect-gate.mjs`, `pre-write-egov-stack-guard.mjs` | 쓰기 전 규칙·운영가드 + 설계 게이트(A-10) + 금지 스택 차단(A-12) |
| PreToolUse(Bash) | `pre-bash-guard.mjs` | 위험 bash 명령 실행 전 차단 |
| PostToolUse(WriteFile\|Edit) | `post-write-rule-check.mjs`, `post-write-compile.mjs`, `post-write-mvn-test.mjs` | 쓰기 후 컨벤션 점검 → 컴파일 → `mvn test` |
| Stop | `langfuse-hook.mjs`, `stop-gate.mjs`, `retry-counter-reconcile.mjs`, `prod-diff-guard.mjs`, `stop-architect-doc-check.mjs`, `stop-buildfix-verify.mjs` | trace 전송(선택) + 종료 게이트·정리 + 설계서 검증(A-11) + 빌드 검증(A-13) |

### 2.1 명령별 차단(블로킹) 게이트 상세

`/architect`·`/migrate`·`/build-fix` 실행 시 `cmd-context-stamp.mjs`가 어떤 명령 흐름인지 마커(`.egov-cmd-context.json`)로 각인하고, Pre/Stop 훅이 이를 읽어 규칙을 **강제(차단)** 합니다.

**컨텍스트 각인 조건** — 슬래시 명령(1순위) 없으면 자연어 키워드(2순위)로 감지:

| 명령 | 슬래시 | 자연어 키워드 예 |
|---|---|---|
| `architect` | `/architect` | "…기능 설계해줘", "구조 설계", "코드 생성", "컴포넌트 생성" |
| `migrate` | `/migrate` | "현행화", "마이그레이션", "레거시 코드 수정", "기존 소스 변환" |
| `build-fix` | `/build-fix` | "빌드 오류", "빌드 실패", "빌드가 안 됨" |

> "생성해줘"·"수정해줘" 같은 범용 동사 단독은 각인되지 않습니다(대상 명사와 결합할 때만 각인).

**차단 훅 목록**:

| 코드 | 훅 | 발동 조건 | 차단 사유 | 통과 조건 |
|---|---|---|---|---|
| A-10 | `pre-write-architect-gate` | architect 컨텍스트(슬래시 각인만) + `src/**` 쓰기 | 설계서(`egov-docs-architect/*-설계서.md`)가 하나도 없음 | 설계서 존재 시 / 자연어 각인 시 |
| A-12 | `pre-write-egov-stack-guard` | migrate·build-fix 컨텍스트 + `src/main/**`·`pom.xml` 쓰기 | 금지 스택 도입(JPA·`jakarta.*`·JUnit5·메서드 `@Transactional`·XML 스프링설정) | eGov 4.3 정상 스택(`javax.*`·MyBatis·JUnit4·`org.hibernate.validator`) |
| A-11 | `stop-architect-doc-check` | architect 컨텍스트 종료 + 설계서 존재 | 설계서에 6대 섹션 누락(도메인·컴포넌트·의존성·구현플랜·위험·추적성) | 6섹션 완비(미완결 시 최대 3회 후 통과) |
| A-13 | `stop-buildfix-verify` | build-fix 컨텍스트 종료 + `pom.xml` + `src/main`·`pom.xml`에 커밋 안 된 변경 존재 | `mvn compile` 실패 | 컴파일 성공 / 변경 없음(분석만) / mvn·의존성 문제(fail-open) |

**오탐 방지 정책**: 컨텍스트는 TTL 2시간, 세션 검증, 다음 명령 덮어쓰기로 만료됩니다. build-fix는 검증 성공 시 즉시 정리됩니다. 판단 불가(파일 없음·mvn/git 부재·의존성 실패) 상황에서는 **차단하지 않고 통과(fail-open)** 하도록 설계되어 있습니다 — 이는 오탐으로 인한 업무 중단을 막기 위한 의도적 설계이므로, "왜 차단이 안 됐지?"라는 문의가 오면 이 fail-open 정책을 우선 안내하십시오.

### 2.2 Langfuse 관찰성 훅 (선택)
`langfuse-hook.mjs`는 세션 종료 시 trace + tool span을 Langfuse로 전송합니다. 환경변수(`LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`, `LANGFUSE_BASE_URL`)가 미설정이면 자동으로 건너뛰므로 필수 설치가 아닙니다. Node 내장 `fetch`로 직접 호출하므로 별도 SDK 설치가 필요 없습니다.

---

## 3. JUnit 자동화 파이프라인 (관리 관점)

`junit-gen`을 진입점으로 하는 파이프라인 구성:

```
junit-analyze (소스 파싱→메타 JSON)
  → junit-collect-refs (VO/Mapper/Swagger 참조 수집)
  → junit-plan-cases (Stage1: TC 목록 도출)
  → junit-render (Stage2: 골든 템플릿 렌더)
  → junit-verify (컴파일/테스트/게이트 집계 → verdict)
```
보통 이 서브 스킬들은 직접 호출하지 않고 `junit-gen`(자연어 트리거) 또는 `/junit-api`, `/auto-skill-junit-gen`(명시 호출)이 순서대로 부릅니다.

이 서브시스템은 저장소 내 별도 변경이력(`CHANGELOG-junit-test.md`)과 태그(`junit-test_v0.1.0` ~ `junit-test_v0.6.0`, 최신 태그 2026-07-08)로 독립 버전 관리되고 있습니다 — 전체 확장 버전(`0.1.0`)과 별개로 진행 상황을 추적할 때는 이 태그를 참고하십시오.

---

## 4. 배포/업데이트 운영 절차

### 4.1 신규 개발자 배포
1. Node.js 22+ 설치 확인 안내
2. `npm install -g @qwen-code/qwen-code` 안내
3. 사내 GitLab 접근 권한 발급 확인
4. `qwen extensions install <git URL>` (기본 `main` 브랜치) 안내
5. `/guide` 실행으로 정상 설치 확인 안내

상세 명령어는 [설치_매뉴얼_qwen-code-extension.md](설치_매뉴얼_qwen-code-extension.md) 참고.

### 4.2 버전 업데이트 배포
- `main` 브랜치에 변경사항 머지 → 개발자는 각자 `qwen extensions update qwen-extension` 실행 → **qwen 세션 재시작**
- `link` 방식(심볼릭 링크)으로 설치한 개발자는 로컬 저장소를 `git pull`만 하면 되고 세션 재시작만 필요(별도 update 불필요)
- 설치 방식은 각 개발자가 `qwen extensions list`의 `Type` 컬럼으로 자가 확인 가능

### 4.3 확장 기능 추가/수정 (개발자 온보딩용)
| 작업 | 위치 |
|---|---|
| 새 슬래시 명령 추가 | `commands/*.md` (Markdown + frontmatter `description`/`argument-hint`, 본문에 `{{args}}`/`@{path}` 사용) |
| 새 스킬 추가 | `skills/*/SKILL.md` (frontmatter `name`/`description`/`triggers`/`license`) |
| 새 훅 추가 | `hooks/*.mjs` 작성 후 `hooks/hooks.json`에 시점·matcher 등록 |
| 컨벤션 수정 | `QWEN.md` (SSOT — 스택·레이어·네이밍 규칙) |
| 모범 예시/패턴 수정 | `references/` |
| 설계 변경 기록 | `doc/architecture.md` |

---

## 5. 트러블슈팅

| 증상 | 점검 사항 |
|---|---|
| 확장 설치는 됐는데 명령이 안 보임 | qwen 세션을 재시작했는지 확인. `qwen extensions list`로 설치 상태(Type) 확인 |
| 훅이 원하는 대로 차단을 안 함 | fail-open 정책(§2.1) 때문일 수 있음 — 대상 파일 부재, git/mvn 미설치, 의존성 문제 등으로 판단 불가 시 의도적으로 통과시키도록 설계됨 |
| 훅이 과도하게 차단함(오탐) | 컨텍스트 TTL(2시간) 만료를 기다리거나, 슬래시 명령 대신 자연어로 작업해 A-10 게이트를 우회(자연어 각인은 A-10 미적용) |
| `mvn compile`/`mvn test` 훅이 느림/실패 | 대상 프로젝트의 JDK 17·Maven 환경이 올바른지, `pom.xml` 의존성 다운로드가 정상인지 로컬에서 직접 `mvn compile` 실행해 재현 확인 |
| Langfuse 연동이 안 됨 | 환경변수(`LANGFUSE_PUBLIC_KEY` 등) 설정 여부 및 `.env.qwen` source 여부 확인. 미설정 시 정상적으로 스킵되는 것이 기본 동작 |

---

## 6. 알려진 미구현/제한 사항 (관리자가 사전 인지해야 할 사항)
- **`egov-render` 스킬은 계약만 정의되어 있고 실행 인프라(`render.py`, `templates/*.j2`, `spec.schema.json`)가 저장소에 없어 현재 실행 불가**합니다. 사용자에게 "결정형 CRUD 렌더" 관련 문의가 오면 이 제한을 안내하고, 대안으로 `/architect`의 코드 생성 단계를 안내하십시오.
- 확장 전체가 **v0.1.0 초기 개발 단계**로, 프롬프트/명령 동작이 변경될 수 있습니다. 배포 전 릴리스 노트를 반드시 개발자에게 공지하십시오.
- `doc/architecture.md`의 2~7절은 과거(Hermes→Qwen 전환 초기) 설계 기록으로 **현재 구조와 다릅니다**(예: 옛 문서의 `/egov:crud` 네임스페이스는 현재 `/architect` 등 평면 네임스페이스로 대체됨). 최신 구조는 README.md와 9절을 기준으로 안내하십시오.

