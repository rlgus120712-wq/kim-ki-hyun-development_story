# 🎨 Figma × MCP 연동 가이드 — 2025-08-13

> 오케스트로 클라우드 프론트엔드 팀을 위한 Figma 연동 표준 가이드. Windsurf/Codeium MCP 서버를 통해 디자인-개발 워크플로우를 자동화하고, 팀 기준을 통일합니다.

---

## ⚡ TL;DR
- Figma 개인 액세스 토큰을 생성하고, 개발 도구(MCP)에서 figma 서버를 등록하세요.
- 파일 키, 노드 ID를 알면 썸네일/코멘트/프리뷰를 바로 불러올 수 있습니다.
- 보안: 토큰은 .env.local 등에 저장하고 권한 최소화 원칙을 지킵니다.

---

## 🚀 Quick Start
1) 준비물
- GitHub Codespaces 또는 로컬 IDE(Windsurf/Codeium 등)
- Figma 계정 및 Personal Access Token

2) Figma 토큰 발급
- https://www.figma.com/developers/api#access-tokens
- 발급된 토큰은 안전하게 보관(.env.local)

3) MCP 서버 등록 예시
- 도구 설정에서 figma MCP 서버를 활성화하고 아래 환경 변수를 설정합니다.
```
FIGMA_TOKEN=<your_figma_token>
```

4) 연결 테스트 시나리오
- 파일 추가: 파일 URL을 등록하여 컨텍스트에 추가
- 노드 썸네일/코멘트: 특정 노드 미리보기/코멘트 읽기

---

## 🧭 Common Actions (예시)
아래 예시는 IDE에서 MCP figma 서버를 사용하는 워크플로우를 가정합니다.

- 파일 컨텍스트에 추가
  - 입력: Figma 파일 URL
  - 결과: 에이전트가 파일 메타/페이지 구조에 접근 가능

- 노드 썸네일 미리보기
  - 입력: file_key, node_id
  - 결과: 컴포넌트 썸네일/레이어 구조 파악에 용이

- 코멘트 읽기/작성/답글
  - 입력: file_key, (옵션) node_id, comment_id
  - 결과: 디자인 리뷰를 IDE 안에서 이어서 진행

Tip: 파일 키(file_key)는 Figma URL의 /file/<file_key>/ 구간, 노드 ID는 좌측 레이어에서 우클릭 → Copy/Paste → Copy Link 로 확인할 수 있습니다.

---

## 👥 Team Workflow
- PR 전 최종 점검: 디자인 스펙(간격/색상/폰트) → 코드 변환 체크리스트 완료 후 리뷰 요청
- 리뷰어 가이드: PR에서 해당 컴포넌트의 Figma 노드 링크를 확인하고 IDE에서 썸네일/코멘트 병행 확인
- 변경 추적: 디자인 업데이트 시 관련 노드 링크를 이슈/PR에 참조로 남겨 변경 맥락 유지

---

## ✅ Checklist
- [ ] FIGMA_TOKEN 발급 및 안전 저장(.env.local, Secret)
- [ ] figma MCP 서버 등록 및 연결 테스트 완료
- [ ] 파일 키/노드 ID 정리(README 또는 이슈에 첨부)
- [ ] PR 설명에 Figma 링크 포함(해당 화면/컴포넌트)
- [ ] 접근 권한 최소화(팀 정책 준수)

---

## 🔒 Security Best Practices
- 토큰은 절대 커밋하지 않기(.gitignore, 환경 변수 사용)
- 필요한 범위로만 권한 부여(원칙: 최소 권한)
- 팀/조직 보안 정책 준수(토큰 로테이션, 비밀 관리)

---

## 🧩 Troubleshooting
- 401 Unauthorized: 토큰 만료/오타 → 재발급 또는 환경 변수 재설정
- Not Found(file_key/node_id): URL/노드 링크 재확인
- Rate Limit: 요청 간격 조절 또는 토큰 권한 확인

---

## 📚 References
- Figma REST API: https://www.figma.com/developers/api
- Figma Webhooks: https://www.figma.com/developers/api#webhooks_v2
- Personal Access Token: https://www.figma.com/developers/api#access-tokens

---

## 🧪 향후 확장 아이디어
- 노드 → Vue 컴포넌트 스캐폴딩(토큰/컬러/스페이싱 자동 매핑)
- 디자인 토큰 동기화 파이프라인(Figma Variables ↔ Design System)
- CI에서 디자인-코드 스냅샷 비교로 시각 회귀 테스트
