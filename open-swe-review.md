# Open-SWE 기술 검토 보고서

**작성일**: 2026-03-30
**작성자**: me3602
**대상 프로젝트**: [me3602.github.io](https://github.com/me3602/me3602.github.io) (Jekyll 기반 개인 블로그)

---

## 1. 개요

본 보고서는 LangChain에서 개발한 오픈소스 코딩 에이전트 프레임워크 **Open-SWE**를 실제 GitHub 리포지토리(`me3602.github.io`)에 적용하여 테스트한 결과를 정리한 기술 검토 보고서이다.

Open-SWE는 LangGraph와 Deep Agents 기반으로 구축된 내부 코딩 에이전트 프레임워크로, GitHub Issue/PR 코멘트, Slack, Linear 등을 통해 에이전트를 호출하고, 샌드박스 환경에서 코드를 작성하여 자동으로 PR을 생성하는 기능을 제공한다.

---

## 2. 테스트 환경

| 항목 | 내용 |
|------|------|
| **Open-SWE 실행 환경** | macOS (Darwin 25.3.0), Python 3.13 |
| **샌드박스 타입** | `local` (로컬 개발용, 격리 없음) |
| **대상 리포지토리** | `me3602/me3602.github.io` |
| **대상 리포지토리 기술 스택** | Jekyll, GitHub Pages |
| **호출 방식** | GitHub Issue에서 `@openswe` 멘션 |

---

## 3. 환경 구성 시 발생한 이슈

Open-SWE를 로컬에서 구동하기까지 다음과 같은 설정 이슈가 순차적으로 발생했다.

### 3.1 GitHub 유저 매핑 누락

- **에러**: `No email mapping for GitHub user 'me3602', skipping`
- **원인**: `agent/utils/github_user_email_map.py`에 GitHub 유저명을 `m3602`로 오타 입력
- **해결**: `m3602` → `me3602`로 수정

### 3.2 암호화 키 미설정

- **에러**: `EncryptionKeyMissingError`
- **원인**: `.env`에 `TOKEN_ENCRYPTION_KEY` 환경변수 미설정
- **해결**: `python3 -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"` 로 Fernet 키를 생성하여 `.env`에 추가

### 3.3 Fernet 키 형식 오류

- **에러**: `Fernet key must be 32 url-safe base64-encoded bytes`
- **원인**: 올바르지 않은 형식의 키 값 설정
- **해결**: Fernet 모듈로 생성한 유효한 키로 교체

### 3.4 LangSmith Sandbox 미활성화

- **에러**: `Sandbox feature is not enabled for this organization`
- **원인**: 기본 `SANDBOX_TYPE`이 `langsmith`이나, 사용 중인 LangSmith 조직에서 Sandbox 기능이 비활성화 상태
- **해결**: `.env`에 `SANDBOX_TYPE=local` 설정

### 3.5 LangSmith API 인증 실패

- **에러**: `401 Unauthorized for url: https://api.smith.langchain.com/runs/multipart`
- **원인**: `LANGSMITH_API_KEY`가 유효하지 않음
- **해결**: `LANGCHAIN_TRACING_V2=false`로 tracing 비활성화 또는 유효한 API 키로 교체

### 3.6 기타 Warning

- **LangChain Deprecation Warning**: `.text()` 메서드 대신 `.text` 프로퍼티 사용 권고 — `langchain` 내부 코드의 경고로, 동작에 영향 없음
- **Pydantic Serialization Warning**: `context` 필드 직렬화 시 타입 불일치 경고 — 동작에 영향 없음

---

## 4. Open-SWE 수행 작업 결과

Open-SWE가 GitHub Issue를 기반으로 총 **3건의 작업**을 자동 수행하였다.

### 4.1 작업 1: Jekyll 웹사이트 초기화 (Issue #1 → PR #2)

| 항목 | 내용 |
|------|------|
| **브랜치** | `open-swe/e7046ca8-b5ee-2224-e828-bfea878c9b9f` |
| **커밋** | `124c851` — `feat: initialize Jekyll website with layouts, styles, and sample content` |
| **변경 사항** | 15개 파일, +246줄 |
| **주요 내용** | Jekyll 설정(`_config.yml`), 레이아웃(`_layouts/`), 스타일(`_sass/main.scss`), 헤더/푸터(`_includes/`), 샘플 포스트(`_posts/2025-01-01-welcome.md`), About 페이지, Gemfile 등 웹사이트 전체 구조 생성 |
| **상태** | PR #2 머지 완료 |

### 4.2 작업 2: GraphRAG 기술 리서치 글 작성 (Issue #3 → PR #4)

| 항목 | 내용 |
|------|------|
| **브랜치** | `open-swe/ad84a1c7-513a-30e7-80e9-e6249a77e287` |
| **커밋** | `ab2c1a1` — `feat: add GraphRAG technical research article` |
| **변경 사항** | 1개 파일, +239줄 |
| **주요 내용** | `_posts/2025-01-02-graphrag-technical-research.md` — GraphRAG의 정의, 구성 요소, 인덱싱 파이프라인, 쿼리 엔진, 장단점, 관련 라이브러리, 활용 예시, 연관 기술, 참고문헌까지 포함한 포괄적 기술 리서치 문서 |
| **상태** | PR #4 머지 완료 |

### 4.3 작업 3: Clean Blog 테마 적용 (진행 중)

| 항목 | 내용 |
|------|------|
| **브랜치** | `open-swe/88ceaad0-5809-edfa-5549-31565dd7436d` |
| **커밋** | `e5d94d0` — `feat: apply StartBootstrap Clean Blog Jekyll theme via remote_theme` |
| **변경 사항** | 21개 파일, +108줄 / -199줄 |
| **주요 내용** | 커스텀 레이아웃/스타일 제거 후 `jekyll-remote-theme` 플러그인을 통한 Clean Blog 테마 적용, 배경 이미지 추가(`img/`), Contact 페이지 폼 구현, Posts 페이지 페이지네이션 구현 |
| **상태** | 브랜치 생성 완료 (PR 대기 또는 진행 중) |

---

## 5. 평가

### 5.1 잘 된 점

| 항목 | 상세 |
|------|------|
| **자동 PR 생성** | GitHub Issue에 `@openswe` 멘션만으로 자동 브랜치 생성 → 코드 작성 → PR 생성까지 전 과정이 자동화됨 |
| **코드 품질** | Jekyll 프로젝트 구조를 정확히 이해하고, 표준적인 디렉토리 구조(`_layouts/`, `_includes/`, `_posts/`, `_sass/`)로 초기화 |
| **콘텐츠 생성 능력** | GraphRAG 리서치 글은 239줄 분량의 체계적인 기술 문서로, 정의/구조/장단점/라이브러리 비교/활용 예시/참고문헌까지 포괄 |
| **커밋 메시지** | Conventional Commits 형식(`feat:`)으로 명확한 커밋 메시지 작성, `Co-authored-by` 태그 포함 |
| **반복적 개선** | 1차 커스텀 테마 → 3차에서 Clean Blog 테마로 리팩터링하는 등, 단계적으로 프로젝트를 개선 |

### 5.2 개선이 필요한 점

| 항목 | 상세 |
|------|------|
| **초기 설정 복잡도** | 6건의 설정 이슈를 순차적으로 해결해야 구동 가능. `INSTALLATION.md`가 있지만 로컬 개발 시 필수 환경변수(`TOKEN_ENCRYPTION_KEY`, `SANDBOX_TYPE=local`)에 대한 안내가 부족 |
| **에러 메시지 명확성** | `EncryptionKeyMissingError`는 키 생성 방법을 안내하지 않음. 에러 발생 시 해결 가이드가 로그에 포함되면 개발자 경험이 개선될 것 |
| **LangSmith 의존도** | 기본값이 `langsmith` 샌드박스이므로, LangSmith 미사용 환경에서는 반드시 `SANDBOX_TYPE` 변경이 필요하나 이를 알기 어려움 |

---

## 6. 결론

Open-SWE는 GitHub Issue 기반의 코딩 에이전트로서 기본적인 기능이 잘 동작함을 확인하였다. Jekyll 프로젝트 초기화, 기술 문서 작성, 테마 변경 등 3건의 작업을 성공적으로 자동 수행하였으며, 코드 품질과 커밋 관리 수준도 양호하다.

다만 초기 환경 구성 단계에서의 진입 장벽이 높은 편이므로, 로컬 개발 환경을 위한 빠른 시작 가이드나 `docker-compose` 기반의 올인원 설정이 보완되면 도입이 더 용이해질 것으로 판단된다.

---

*본 보고서는 Open-SWE를 `me3602.github.io` 리포지토리에 적용한 테스트 결과를 기반으로 작성되었습니다.*
