# 설치 매뉴얼 — Qwen Code CLI + qwen-extension(eGovFramework 4.3)

- 작성일: 2026-07-21
- 대상 독자: 사내 개발자(PC에 직접 설치)
- 근거 자료: [QwenLM/qwen-code](https://github.com/QwenLM/qwen-code) npm 패키지 정보, `qwen-extension-egovframework-4.3` 저장소 `README.md` 설치 절 원문

---

## 1. 사전 요구사항

| 항목 | 요구 버전 | 비고 |
|---|---|---|
| Node.js | **22.0.0 이상** | `@qwen-code/qwen-code` npm 패키지 `engines.node` 기준(`>=22.0.0`). 확장의 훅 스크립트는 Node 20 이상이면 동작하지만, **qwen-code CLI 자체가 Node 22+를 요구**하므로 최종 기준은 22 이상 |
| OS | macOS / Linux / Windows(cmd, PowerShell) | 확장 훅은 Node.js ESM 기반으로 크로스플랫폼 동작 |
| Git | 최신 버전 | 확장을 git URL 방식으로 설치할 경우 필요 |
| 사내 GitLab 접근 권한 | `git.bluewindsky.com` 계정/인증 | qwen-extension 저장소가 내부 GitLab에 있어 접근 권한 필요 |
| Java/Maven 환경 | JDK 17, Maven | 확장 사용 시(코드 생성·빌드 검증 훅) 대상 프로젝트 빌드용 — 확장 자체 설치에는 불필요하나 실사용 전 준비 필요 |

---

## 2. Qwen Code CLI 설치

### 2.1 Node.js 버전 확인
```bash
node -v
# v22.0.0 이상이어야 함. 아니라면 nvm 등으로 Node 22+ 설치 후 진행
```

### 2.2 Qwen Code CLI 설치 (npm)
```bash
npm install -g @qwen-code/qwen-code
```

### 2.3 설치 확인
```bash
qwen --version
```

> 참고: 2026-07-21 기준 npm `latest` 태그 버전은 **0.20.0** 입니다. 사내 표준 버전을 고정하려면 `npm install -g @qwen-code/qwen-code@0.20.0` 형태로 버전을 명시하십시오.

---

## 3. qwen-extension(eGovFramework 4.3) 설치

두 가지 방식 중 상황에 맞게 선택합니다.

### 방법 1 — Git 저장소에서 직접 설치 (권장, 일반 개발자용)

저장소를 미리 클론하지 않고 git URL로 바로 설치합니다. 이후 `qwen extensions update`로 최신화할 수 있습니다.

```bash
# 기본 브랜치(main, 안정 버전)로 설치
qwen extensions install https://git.bluewindsky.com/internal/ssis/qwen/extension/qwen-extension-egovframework-4.3.git

# 특정 브랜치/태그/커밋 지정이 필요한 경우 (--ref)
qwen extensions install https://git.bluewindsky.com/internal/ssis/qwen/extension/qwen-extension-egovframework-4.3.git --ref=develop

# 설치 확인
qwen extensions list
```

⚠️ **주의**
- 이 저장소는 사내 GitLab(`git.bluewindsky.com`)에 있으므로 **접근 권한(인증)이 사전에 필요**합니다.
- `develop` 브랜치는 다음 릴리스 후보를 통합하는 **테스트 브랜치**입니다. 일반 개발자는 **기본 브랜치(`main`)**를 사용하십시오.

**업데이트 / 제거**
```bash
qwen extensions update qwen-extension     # 최신화 (이미 최신이면 아무 동작 없음)
qwen extensions uninstall qwen-extension  # 제거
```
> `update`는 `~/.qwen/extensions/`에 설치된 사본만 갱신합니다. **변경 사항을 반영하려면 실행 중인 qwen 세션을 재시작**해야 합니다.

### 방법 2 — 로컬 경로에서 설치/link (확장 개발자·오프라인 환경용)

저장소를 직접 클론한 경우 사용합니다.

```bash
# 저장소 클론
git clone https://git.bluewindsky.com/internal/ssis/qwen/extension/qwen-extension-egovframework-4.3.git

# 방식 A: 로컬 경로 설치 (디렉터리 복사본 생성)
qwen extensions install /path/to/qwen-extension-egovframework-4.3
qwen extensions list

# 방식 B: 심볼릭 링크 설치 (개발 모드 — 로컬 수정이 즉시 반영됨)
qwen extensions link /path/to/qwen-extension-egovframework-4.3
```

- 재설치 시에는 먼저 `qwen extensions uninstall qwen-extension`으로 기존 설치본을 제거해야 합니다.
- `install`(복사본)과 `link`(심볼릭 링크)의 차이: `link`로 설치하면 로컬 디렉터리가 곧 설치본이므로 **항상 최신 상태**이며 별도 update가 불필요합니다(세션 재시작만 필요). `install`은 매번 `update` 명령으로 갱신해야 합니다.
- 설치 방식은 `qwen extensions list`의 `Type` 컬럼(`git`/`local`/`link`)으로 확인할 수 있습니다.
- 설치/업데이트 후에는 **활성 qwen CLI 세션을 재시작**해야 변경 사항이 반영됩니다.

---

## 4. 설치 검증

```bash
qwen extensions list
# qwen-extension 이 목록에 표시되고 Type이 git/local/link 중 하나로 확인되어야 함

qwen
# qwen 세션 진입 후
/guide
# 확장의 커맨드/스킬/훅 카탈로그가 출력되면 정상 설치
```

---

## 5. 선택 사항 — Langfuse 관찰성 훅 연동

세션 종료 시 trace를 Langfuse로 전송하는 훅(`langfuse-hook.mjs`)이 포함되어 있으며, 환경변수가 없으면 자동으로 건너뜁니다(설치 필수 아님). 사용하려면:

```bash
# 프로젝트 루트 또는 ~/.env.qwen 에 추가 후 qwen 실행 전 source
LANGFUSE_PUBLIC_KEY=pk-lf-...
LANGFUSE_SECRET_KEY=sk-lf-...
LANGFUSE_BASE_URL=https://langfuse.nemoai.kr   # 자체 호스팅 시 변경
```

