# 사용자 매뉴얼 — qwen-extension (eGovFramework 4.3)

- 작성일: 2026-07-21
- 대상 독자: 확장을 사용해 eGovFramework 4.3(Spring Boot) 코드를 생성·리뷰·검증하는 개발자
- 근거 자료: `qwen-extension-egovframework-4.3` 저장소 `README.md`, `commands/guide.md` 원문
- 설치가 필요한 경우 [설치_매뉴얼_qwen-code-extension.md](설치_매뉴얼_qwen-code-extension.md) 참고

---

## 1. 개요

qwen-extension은 [Qwen Code CLI](https://github.com/QwenLM/qwen-code) 확장으로 동작하며, 전자정부 표준프레임워크(eGovFramework) **4.3** 컨벤션에 맞춰 프로젝트 부트스트랩, 백엔드 코드 생성·설계·리뷰·빌드, **JUnit 테스트 자동화**를 지원합니다.

**타깃 스택**: Java 17(컴파일·구동) · Spring Boot 2.7.18 · eGov RTE 4.3.0 · MyBatis 3.5.16 · Java `@Configuration`(`EgovConfig*`) · JSP 2.x + HTML 4.01/HTML5(JSTL) · Oracle(RDBMS) · Lombok · jar 패키징 · JUnit 4.13(테스트)

> ⚠️ **초기 개발 단계(v0.1.0)** — 명령·프롬프트·생성 결과가 예고 없이 바뀔 수 있습니다. 생성된 코드는 실서비스 적용 전 반드시 검증하십시오.

---

## 2. 권장 작업 흐름

```
/egov-init  ──▶  /architect  ──▶  [egov-render(미구현)]  ──▶  JUnit 자동화  ──▶  /code-review · /security-review
(프로젝트 셋업)    (설계→승인 시 코드)                          (테스트 생성·검증)      (품질·보안 검증)
                                                                                └▶ /build-fix (빌드 오류 시)
```

- **슬래시 커맨드**(`/이름`)는 명시적으로 호출해야 동작합니다.
- 각 커맨드에는 **자연어 트리거로 자동 발동하는 짝 스킬**이 있습니다(예: "설계해줘"라고 말하면 `/architect`와 동일한 스킬이 자동 발동).
- 레거시 소스를 이관하는 경우 `/egov-init` 대신 `/migrate`로 현행화부터 시작할 수 있습니다.

---

## 3. 커맨드별 사용법

### 3.1 `/egov-init` — 프로젝트 부트스트랩
새 eGovFrame 4.3(Spring Boot) 프로젝트를 백지 상태에서 "요구사항 정의 직전 단계"까지 셋업합니다.

```
/egov-init [프로젝트명] [clone|generate]
```
- 예: `/egov-init my-portal clone`
- 둘째 인자 기본값은 `clone`(공식 `egovframe-boot-sample-java-config` v4.3.0 클론, 권장). `generate`는 표준 pom + `EgovConfig*` + 진입점을 직접 생성.
- 동작: 환경 점검 → 공식 boot 샘플 클론/골격 생성 → JDK 17 고정 → DB 연결·스키마 → `mvn package` 빌드 검증
- 자동 발동 키워드: "프로젝트 생성", "초기 셋업", "처음부터", "스캐폴딩", "환경 구축" 등

### 3.2 `/architect` — 설계 → 승인 시 코드 생성
기능 하나를 입력하면 요구사항정의서·화면목업·설계서·테스트케이스·DB테이블·ERD·API명세를 산출하고, **사용자 승인 시에만** 실제 코드(VO·Mapper·XML·Service·Controller·Test)를 생성합니다.

```
/architect <기능 설명>
```
- 예: `/architect 회원 포인트 적립/사용 기능`
- 흐름: 요구사항 게이트(부족하면 질문 후 정지) → 요구사항정의서·유스케이스명세서·화면목업 생성 → 설계 → 설계 문서 저장 → 구현 여부 확인 → 승인 시 코드 생성
- 설계 문서만 원한다면 구현 승인을 보류할 수 있습니다.
- 자동 발동 키워드: "설계해줘", "구조 설계", "코드 생성", "컴포넌트 생성" 등

### 3.3 `/migrate` — 레거시 소스 현행화
레거시 소스를 eGov 4.3(Spring Boot) 스펙으로 현행화합니다.

```
/migrate <파일|디렉터리|프로젝트 경로>
```
- 스코프를 자동 감지해 분석·갭 리포트·플랜을 먼저 제시하고, **승인 시에만 수정**합니다.
- 자동 발동 키워드: "현행화", "마이그레이션", "레거시 코드 수정", "기존 소스 변환" 등

### 3.4 `/code-review` — 코드 품질 리뷰
```
/code-review <대상 경로|변경분>
```
- 예: `/code-review egovframework/point/`
- eGov 4.3 컨벤션 기준 규약 위반·로직 결함·가독성·중복·예외/트랜잭션 처리를 **Critical/High/Medium/Low** 등급으로 분류해 보고합니다.
- 자동 발동 키워드: "코드 리뷰", "코드 검토", "품질 점검", "컨벤션 검사" 등

### 3.5 `/security-review` — 보안·시크릿 점검
```
/security-review <대상 경로|변경분>
```
- SQL Injection/XSS/CSRF/경로조작/인증우회/민감정보 로깅 취약점과 하드코딩 시크릿(비밀번호/API키/DB접속정보/토큰)을 점검하고 위험도·대안을 제시합니다.
- 자동 발동 키워드: "보안 리뷰", "보안 점검", "취약점", "시크릿", "SQL injection" 등

### 3.6 `/build-fix` — 빌드 오류 진단·해결
```
/build-fix <빌드 로그 붙여넣기>
```
- 근본 원인 → 수정 위치 → 패치 → 재발 방지 순으로 제시합니다.
- 자동 발동 키워드: "빌드 오류", "빌드 실패", "컴파일 에러", "Cannot find symbol" 등

### 3.7 `/guide` — 사용법 카탈로그
```
/guide [전체|init|architect|review|security|build|render|junit|hooks]
```
- 인자 없이 실행하면 확장의 전체 사용법을 출력합니다. 인자를 주면 해당 영역만 간결히 출력합니다.
- 사용 중 헷갈리는 부분이 있을 때 가장 먼저 실행해볼 명령입니다.

### 3.8 `egov-render` — 결정형 CRUD 렌더 (⚠️ 현재 미구현)
동일 입력 → 동일 출력(SHA256 동일)을 목표로 하는 결정형 CRUD 생성 기능이나, **실행 인프라(`render.py`·템플릿)가 아직 저장소에 없어 현재 사용할 수 없습니다.** 반복·대량 CRUD 생성이 필요하면 현재는 `/architect`의 코드 생성 단계를 사용하십시오.

---

## 4. JUnit 테스트 자동화

슬래시 명령이 아니라 **트리거 기반 스킬**로 동작하며, 두 가지 진입 방식이 있습니다.

### 4.1 자연어 트리거 (권장 — 가장 간단)
대상 Java 클래스를 두고 "이 클래스 JUnit 테스트 생성/리뷰해줘"라고 요청하면 `junit-gen`이 발동해 파이프라인(분석 → 참조수집 → 케이스 도출 → 렌더 → 검증)을 진행합니다. 케이스 목록을 사용자가 승인한 뒤 렌더링하며, 파일 저장 후 자동으로 컴파일·테스트가 검증됩니다.

### 4.2 명시 호출 (정밀 제어가 필요할 때)
| 명령 | 인자 | 역할 |
|---|---|---|
| `/junit-api` | `<endpoint>` | 엔드포인트 1개로 Controller·Service·Repository 3레이어 테스트를 일괄 생성 (권장 진입점). 예: `/junit-api /record` |
| `/auto-skill-junit-gen` | `<source_file> <layer> [entity_file] [repository_file] [swagger_file]` | 단일 파일 오케스트레이터. `layer` = `controller`\|`service`\|`repository` |
| `/auto-skill-junit-review` | `<test_file>` | 테스트 코드 리뷰(가독성·Mock·픽스처·커버리지 4영역 채점 → PASS/FAIL) |

> `/junit-api`, `/auto-skill-*`는 자동 발동이 꺼져 있어 **반드시 명시 호출**해야 합니다.

---

## 5. 사용 중 자주 겪는 상황

| 상황 | 원인/설명 |
|---|---|
| `/architect` 없이 코드를 쓰려는데 차단됨 | 설계 게이트(A-10) — `/architect`로 먼저 슬래시 호출한 상태에서 설계서 없이 `src/` 쓰기를 시도하면 차단됩니다. 자연어로 "설계해줘"라고 한 경우는 차단되지 않습니다. |
| `/migrate` 중 JPA/`jakarta.*` 코드를 썼는데 차단됨 | 금지 스택 가드(A-12) — eGov 4.3은 `javax.*`·MyBatis·JUnit4 스택이 표준이라, JPA/`jakarta`/JUnit5/`@Transactional`(메서드)/XML 스프링설정 도입을 차단합니다. |
| 방금 요청이 의도한 명령으로 인식 안 됨 | "생성해줘"·"수정해줘" 같은 범용 동사 단독으로는 인식되지 않습니다(대상 명사와 결합해야 함). 명확히 하려면 슬래시 명령을 직접 사용하십시오. |
| 확장을 업데이트했는데 반영이 안 됨 | `qwen extensions update` 이후 **실행 중인 qwen 세션 재시작이 필요**합니다. |

---

## 6. 더 알아보기
- 전체 개요·구조: 저장소 `README.md`
- 스택·레이어·네이밍 컨벤션(SSOT): `QWEN.md`
- 훅 동작 원리·유지보수: [관리자_매뉴얼_qwen-extension.md](관리자_매뉴얼_qwen-extension.md)
