# Confluence 가이드

## TL;DR
- MCP 도구: confluence_create_page, confluence_update_page, confluence_add_label, confluence_add_comment, confluence_search, confluence_get_page
- 빠른 퍼블리시: scripts/publish-confluence-fast.sh (버전 자동 증가, --dry-run)

## Quick Start
- .env.mcp 예시
```
ATLASSIAN_BASE_URL=https://okestro.atlassian.net
ATLASSIAN_EMAIL=you@example.com
ATLASSIAN_API_TOKEN=your_api_token
ATLASSIAN_CLOUD=true
CONFLUENCE_SPACES_FILTER=PCD
```
- 서버 실행: ./scripts/start-mcp-atlassian.sh

## Common Actions
- 페이지 생성/업데이트/라벨/코멘트/검색/조회 예시 포함 (Confluence 페이지 참조)

## 참고
- https://okestro.atlassian.net/wiki/spaces/PCD/pages/2111799452
- scripts/publish-confluence-fast.sh
