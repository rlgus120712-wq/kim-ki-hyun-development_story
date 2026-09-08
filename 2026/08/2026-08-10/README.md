# 2026-08-10 개발 기록

프론트엔드(Vue 3 + TypeScript + Ant Design Vue) 작업과 개발 환경 세팅에서 배운 점을 정리한다.

---

## 1. Ant Design Vue Tree 검색 기능 직접 구현

### 배운 점
- **`a-tree`(Tree)는 내장 검색·빈 상태(empty)가 없다.** `a-table`, `a-list`, `a-select`는 데이터가 없으면 자동으로 `<a-empty>`를 렌더하고 `locale.emptyText`로 커스텀도 되지만, **Tree만 예외**다.
- 그래서 트리 검색은 프레임워크가 안 해주고 **직접 구현**해야 한다:
  1. 키워드로 노드 필터링 (매칭 노드 + 조상 유지)
  2. 매칭 결과 자동 펼침(`expandedKeys` 세팅)
  3. 빈 결과 처리

### 실시간 검색 vs 태그(엔터) 검색
| 방식 | 트리거 | 특징 |
|------|--------|------|
| 실시간 | 입력 즉시 | UX 즉각적이나 데이터가 매 글자 교체됨 |
| 태그 | 엔터로 태그 확정 시 | 검색 실행 시점 명확, 데이터 churn 최소 |

### 함정: checkable 트리 + 실시간 검색
- **체크박스가 있는 트리**에서 실시간 검색을 붙이면, 매 키 입력마다 `:data`가 새 배열로 바뀌고 → 제어된 `checked-keys`(객체)가 새 노드 집합 기준으로 **재계산되며 체크가 튀는 부작용**이 났다.
- 단일 선택(`selected-keys`) 트리는 체크박스가 없어 이 문제가 없다.
- **해결:** 검색을 태그(엔터 확정) 방식으로 바꿔 데이터 변경 시점을 한정 → 부작용 제거.

### 빈 결과(No Data) 처리
- 컴포넌트 교체(`v-if`로 트리→NoData)로 하면, **트리에만 걸린 고정 높이가 사라져 모달 크기가 줄어드는** 문제가 생긴다.
- CSS를 못 쓰는 제약이 있으면, **트리에 선택·체크 불가한 안내 노드 한 개를 주입**해 "검색 결과 없음"을 표시하면 높이가 유지된다. (antd 표준은 아니고 우회책)

---

## 2. 공용 컴포넌트 수정의 영향 범위

- 조직 선택 모달 하나가 **생성/수정 등 여러 화면에서 공용**으로 쓰였다.
- 공용 컴포넌트/훅을 고치면 **연결된 모든 화면에 동시 반영**된다 → 수정 전 사용처를 먼저 파악하고 회귀 범위를 인지해야 한다.
- 교훈: "한 곳 고치면 끝"이 아니라, **import/사용처 grep으로 영향 범위부터 확인.**

---

## 3. Vue 3.4 → 3.5 주요 변경점

3.4 → 3.5는 마이너 업데이트라 breaking change 없이 호환된다.

- **반응성 시스템 재작성** — 메모리 최대 ~56% 절감, 깊은 반응형 데이터 성능 대폭 향상
- **신규 Composition API**
  - `useTemplateRef()` — 문자열 이름 기반 템플릿 ref
  - `useId()` — SSR 안전 고유 ID
  - `onWatcherCleanup()` — watcher 정리 로직 전역 등록
- **Reactive Props Destructure 정식화** — 구조분해 + 기본값 지원
  ```ts
  const { count = 0, msg = 'hi' } = defineProps<{ count?: number; msg?: string }>()
  ```
- **`watch` 개선** — `deep`에 숫자(깊이) 지정 가능 → 성능 튜닝

---

## 4. 개발 환경: Claude Code + GitHub MCP 연동

### 연동 방식 3가지
| 방식 | Docker | 상시 실행 | 인증 |
|------|:--:|:--:|------|
| 원격 HTTP | ❌ | ❌ (GitHub 호스팅) | OAuth |
| 네이티브 바이너리 | ❌ | ❌ (필요 시 stdio) | PAT |
| Docker | ✅ | ✅ (데몬 필요) | PAT |

### 함정과 교훈
- **`@modelcontextprotocol/server-github`(npx)는 deprecated.** 공식 `github/github-mcp-server`(Go)로 이관됨. Homebrew로 설치 가능(`brew install github-mcp-server`).
- **`claude mcp list`의 `✔ Connected`는 토큰 유효를 의미하지 않는다.** 프로세스 기동/초기화만 확인된 것. 실제 GitHub API(`/user`) 호출로 **401/200**을 봐야 토큰 유효성이 확정된다.
- PAT는 classic 토큰 + `repo` scope면 기본 작업에 충분. 발급 후 페이지 벗어나면 다시 못 보니 즉시 복사.
- MCP 서버를 세션 도중 추가하면 도구가 로드되지 않는다 → **Claude Code 재시작** 필요.

---

## 오늘의 한 줄
> "프레임워크가 당연히 해줄 것 같은 기능(트리 검색·빈 상태)도 컴포넌트마다 다르다. 문서보다 실제 API를 확인하자."
