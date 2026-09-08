# 2025년 8월 개발일지 · MCP × Swagger 연동

> Okestro · August Dev Log · MCP × Swagger Integration
>
> Notion 원문: https://www.notion.so/2025-08-13-Swagger-MCP-24e3c6e76b8e819a8b8cc0299e69d2b5 · 작성일: 2025-08-13

---

## 목차
- [TL;DR](#tldr)
- [아키텍처 개요](#아키텍처-개요)
- [MCP 툴 요약](#mcp-툴-요약)
- [환경/설정](#환경설정)
- [퀵스타트](#퀵스타트)
- [연동 방식(Integration Flow) 상세](#연동-방식integration-flow-상세)
- [레시피 예시](#레시피-예시)
- [Troubleshooting](#troubleshooting)
- [운영 팁](#운영-팁)
- [TODO / Next Steps](#todo--next-steps)

---

## TL;DR
MCP 서버(okestro-swagger)와 Swagger/OpenAPI 스펙을 연결해 엔드포인트 탐색 → 요청 실행 → 응답 스키마 검증까지 자동화합니다. 환경변수(API_BASE_URL 또는 SWAGGER_URL)만 맞추면 빠르게 활용할 수 있습니다.

## 아키텍처 개요
```
Client (Agent)
   └─▶ MCP Server (okestro-swagger)
           └─▶ Swagger/OpenAPI Spec
                   └─▶ Target API
```
데이터 흐름: (1) 스펙 로드 → (2) 엔드포인트 탐색 → (3) 상세 스키마 확인 → (4) 요청 실행 → (5) 응답 스키마 검증

## MCP 툴 요약
- mcp3_fetch_swagger_info: Swagger 문서를 불러옵니다. (API_BASE_URL 기반 자동 탐색 or SWAGGER_URL 명시)
- mcp3_list_endpoints: 사용 가능한 모든 엔드포인트를 나열합니다.
- mcp3_get_endpoint_details: 특정 경로/메서드의 파라미터, 요청/응답 스키마 확인.
- mcp3_execute_api_request: 설정된 베이스 URL과 스키마 기반으로 실제 API 요청 실행.
- mcp3_validate_api_response: 응답을 Swagger 스키마에 맞춰 검증.

## 환경/설정
- API_BASE_URL (권장): 예) https://api.example.com
- SWAGGER_URL (선택): 예) https://api.example.com/swagger.json 또는 /swagger.yaml
- AUTH 헤더(선택): Authorization: Bearer <token> 또는 x-api-key 등
- TLS_SKIP_VERIFY/TIMEOUT_MS (선택): 내부망/테스트 환경용 편의 설정

## 퀵스타트
1) mcp3_fetch_swagger_info — API_BASE_URL 또는 SWAGGER_URL로 스펙 로드
2) mcp3_list_endpoints — 경로/메서드 전체 목록 확인
3) mcp3_get_endpoint_details — 파라미터/요청바디/응답 스키마 확인
4) mcp3_execute_api_request — 헤더/쿼리/바디 구성 후 실제 호출
5) mcp3_validate_api_response — 응답을 스키마로 검증해 회귀 탐지

## 연동 방식(Integration Flow) 상세
- Path 변수: `/users/{id}` → `/users/123` (details에서 변수명 확인 후 치환)
- Query: `?limit=10&offset=0` 형태, required/enum/default는 details 참고
- Body: JSON 또는 multipart/form-data, requestBody 스키마 준수(Content-Type 유의)
- 파일 업로드: multipart/form-data로 파일 파트 포함, 필드명/타입을 스키마로 검증
- 인증 전략: Bearer/API Key/Basic Auth/쿠키 세션 등. 민감정보는 MCP 서버 환경변수나 보안 비밀 저장 사용 권장

## 레시피 예시
### 1) 헬스체크 (예: GET /health)
- list_endpoints → get_endpoint_details → execute_api_request → validate_api_response 순으로 점검

### 2) 리소스 상세 (예: GET /users/{id})
- Path 변수와 쿼리 파라미터를 details에서 확인 후 요청 실행

### 3) 생성 요청 (POST /items)
- requestBody 스키마 참고해 최소 필드로 샘플 페이로드 구성 → 실행 → validate_api_response

## Troubleshooting
- 에러: “No API_BASE_URL configured …”
  - okestro-swagger 서버 환경변수(API_BASE_URL)를 설정하거나, fetch 시 SWAGGER_URL을 명시하세요.
- 404/401 발생
  - 스펙과 배포 버전 차이 또는 인증 필요 여부 확인
- 415/422
  - Content-Type 또는 요청 필드 유효성 문제. requestBody 스키마 재확인
- CORS/게이트웨이 제한
  - 내부망/프록시 경유가 필요한 경우 MCP 서버 측 네트워크 설정 확인

## 운영 팁
- DEV/STAGE/PROD 환경 분리: 프로필별 API_BASE_URL(or SWAGGER_URL) 분리
- 스키마 회귀 방지: mcp3_validate_api_response로 배포 전후 응답 형식 이탈 탐지
- 즐겨찾기 섹션: 인증/카탈로그 조회 등 자주 쓰는 엔드포인트를 별도 문서화

## TODO / Next Steps
- [ ] 실제 사내 Swagger URL 또는 API_BASE_URL로 엔드포인트 예시와 호출 결과 추가 반영
- [ ] 인증(토큰/키) 주입 전략 표준화 및 보안 가이드 문서화
- [ ] 빈번히 쓰는 쿼리/페이로드 스니펫 수집(샘플 컬렉션)

---
본 문서는 GitHub 가독성에 맞춰 구성되었으며, Notion 페이지와 동기 의도를 유지합니다. 스펙 URL/API_BASE_URL 공유 시 실제 호출 예시를 추가 업데이트하겠습니다.
