# 조직(Organization) API 구현 사례

## 📋 프로젝트 개요

**프로젝트명**: Okestro CMP Admin - 조직 관리 기능  
**구현 기간**: 2025년 1월 9일  
**API 서버**: https://dev.admin.okestro.cloud/v1/maestro/admin  
**스웨거 UI**: https://dev.admin.okestro.cloud/v1/maestro/admin/webjars/swagger-ui/index.html  

## 🎯 구현 목표

1. **조직 목록 조회** - 모든 조직의 계층 구조 표시
2. **조직 상세 정보** - 선택된 조직의 세부 정보 패널
3. **하위 구조 탐색** - 폴더 및 워크스페이스 목록
4. **실시간 데이터** - React Query를 활용한 캐싱 및 동기화

## 🔍 API 분석 결과

### 주요 엔드포인트

| 메서드 | 엔드포인트 | 설명 | 응답 형태 |
|--------|------------|------|----------|
| GET | `/org/org-only` | 모든 조직 조회 | `ApiResponse<OrgType[]>` |
| GET | `/org/simple` | 단순 조직 목록 (ID, 이름만) | `ApiResponse<SimpleOrgType[]>` |
| GET | `/org/{orgId}` | 조직 상세 정보 | `ApiResponse<OrgDetailType>` |
| GET | `/org/find` | 조직 이름 중복 체크 | `ApiResponse<boolean>` |

### 실제 API 응답 구조

```json
{
  "data": [
    {
      "id": "40b08afc-8219-11f0-9552-8e258ed8fffe",
      "name": "OKESTRO",
      "createUserId": "system",
      "createTime": "2024-12-19T02:36:52.000+00:00",
      "updateUserId": null,
      "updateTime": null,
      "description": "OKESTRO 조직",
      "status": 1,
      "type": "org",
      "externalLink": null,
      "key": "OKESTRO",
      "orgId": "40b08afc-8219-11f0-9552-8e258ed8fffe",
      "depth": 0,
      "parentId": null,
      "childList": {
        "folders": [],
        "workspaces": []
      },
      "parent": null,
      "org": null,
      "path": "/OKESTRO",
      "root": true
    }
  ],
  "status": 200
}
```

## 🏗️ 구현 아키텍처

```
packages/cmp/src/
├── features/
│   └── org-api/
│       ├── api/
│       │   ├── index.ts                 # API 함수들
│       │   ├── queryKeys.ts             # Query Key 상수
│       │   ├── useOrgListQuery.ts       # 목록 조회 훅
│       │   ├── useSimpleOrgListQuery.ts # 단순 목록 훅
│       │   └── useOrgDetailQuery.ts     # 상세 조회 훅
│       ├── ui/
│       │   ├── OrgTable.vue             # 조직 테이블
│       │   └── OrgDetailPanel.vue       # 상세 패널
│       ├── types.ts                     # 타입 정의
│       └── index.ts                     # Public exports
└── pages/
    └── org/
        ├── ui/
        │   └── OrgListPage.vue          # 메인 페이지
        └── index.ts                     # 라우트 정의
```

## 💾 구현된 타입 정의

```typescript
// 기본 조직 정보 타입
export interface OrgType {
  id: string;
  name: string;
  createUserId: string;
  createTime: string;
  updateUserId?: string;
  updateTime?: string;
  description?: string;
  status: number;
  type: string;
  externalLink?: number;
  key: string;
  orgId: string;
  depth: number;
  parentId?: string;
  childList?: {
    folders?: FolderType[];
    workspaces?: WorkspaceType[];
  };
  parent?: any;
  org?: any;
  path?: string;
  root?: boolean;
}

// 폴더 타입
export interface FolderType {
  id: string;
  name: string;
  description?: string;
  status: number;
  depth: number;
  createTime: string;
  updateTime?: string;
}

// 워크스페이스 타입
export interface WorkspaceType {
  id: string;
  name: string;
  description?: string;
  status: number;
  userCount?: number;
  createTime: string;
  updateTime?: string;
}
```

## 🎨 UI 컴포넌트 구현

### 1. 조직 테이블 (OrgTable.vue)

**주요 기능**:
- 좌측 필터 패널을 통한 데이터 필터링
- 상태별 색상 구분 (활성/비활성/삭제됨)
- 계층 구조 표시 (depth 기반)
- 클릭 시 상세 패널 오픈

**컬럼 구성**:
- 조직명 (링크 타입, 클릭 가능)
- 설명
- 상태 (뱃지 표시)
- 타입 (org/folder 구분)
- 깊이 (계층 레벨)
- 생성일시
- 수정일시

### 2. 상세 패널 (OrgDetailPanel.vue)

**섹션 구성**:
1. **기본 정보** - ID, 이름, 설명, 상태, 타입 등
2. **메타데이터** - 생성자, 생성일시, 수정일시 등
3. **하위 구조** - 폴더 및 워크스페이스 목록
4. **액션 버튼** - 수정, 삭제 버튼

**특징**:
- 카드 기반 레이아웃
- 로딩 상태 표시
- 에러 상태 처리
- 빈 데이터 상태 UI

### 3. 메인 페이지 (OrgListPage.vue)

**레이아웃**:
- `TheTabLayout`을 사용한 탭 구조
- 우측 사이드 패널을 통한 상세 정보 표시
- Props를 통한 컴포넌트 간 데이터 전달

## 🔧 기술적 구현 세부사항

### API 요청 최적화

```typescript
// Query Key 관리로 캐싱 최적화
export const orgQueryKeys = {
  all: ['org'] as const,
  lists: () => [...orgQueryKeys.all, 'list'] as const,
  list: (params?: GetOrgListReqType) => [...orgQueryKeys.lists(), params] as const,
  details: () => [...orgQueryKeys.all, 'detail'] as const,
  detail: (orgId: string) => [...orgQueryKeys.details(), orgId] as const,
};
```

### 에러 처리 및 로딩 상태

```typescript
// Query Hook에서 자동 에러 처리
const { data: response, isLoading, error } = useOrgListQuery(props.searchParams);

// 데이터 변환 시 예외 처리
const list = computed(() => {
  if (!response.value) return [];
  
  try {
    const data = response.value.data?.data;
    return Array.isArray(data) ? data.map(transformItem) : [];
  } catch (error) {
    console.error('Error processing data:', error);
    return [];
  }
});
```

### 타입 안전성 보장

```typescript
// Props 타입 정의
interface Props {
  columns: CmpTableColumnType[];
  searchParams?: GetOrgListReqType;
}

// Emit 이벤트 타입 정의
const emit = defineEmits<{
  selectItem: [item: OrgType];
  createOrg: [];
  editOrg: [org: OrgType];
  deleteOrg: [org: OrgType];
}>();
```

## 📊 성능 최적화

### 1. React Query 캐싱
- API 응답을 자동으로 캐싱하여 불필요한 요청 방지
- staleTime과 cacheTime 설정으로 캐시 정책 관리

### 2. 컴포넌트 최적화
- `computed`를 사용한 리액티브 데이터 변환
- Props를 통한 단방향 데이터 흐름
- 이벤트 기반 컴포넌트 간 통신

### 3. 번들 최적화
- 지연 로딩을 통한 초기 번들 크기 감소
- Tree shaking으로 사용하지 않는 코드 제거

## 🧪 테스트 전략

### 실제 API 테스트

```bash
# 조직 목록 조회 테스트
COOKIE=$(grep "VITE_OAUTH2_COOKIE=" apps/okestro-cmp-app-admin/.env.local | cut -d '=' -f2- | tr -d '"')
curl -k -H "Cookie: _oauth2_proxy=$COOKIE" \
  "https://dev.admin.okestro.cloud/v1/maestro/admin/org/org-only?size=5" \
  | jq '.data[0] | keys'
```

**응답 확인**:
```json
[
  "childList",
  "createTime",
  "createUserId",
  "depth",
  "description",
  "externalLink",
  "id",
  "key",
  "name",
  "org",
  "orgId",
  "parent",
  "parentId",
  "path",
  "root",
  "status",
  "type",
  "updateTime",
  "updateUserId"
]
```

### 브라우저 동작 테스트
1. 페이지 로딩 및 데이터 표시 확인
2. 필터링 기능 동작 확인
3. 상세 패널 오픈/클로즈 확인
4. 에러 상태 및 로딩 상태 확인

## 🐛 해결된 이슈들

### 1. Vue Props 타입 경고

**문제**: `AFlex` 컴포넌트에 `vertical="true"` 문자열 전달
```
[Vue warn]: Invalid prop: type check failed for prop "vertical". Expected Boolean, got String with value "true".
```

**해결**: `:vertical="true"`로 Boolean 값 전달
```vue
<!-- 수정 전 -->
<a-flex vertical="true" class="item-ant-form">

<!-- 수정 후 -->
<a-flex :vertical="true" class="item-ant-form">
```

### 2. ESLint Import 순서 오류

**문제**: Import 구문의 순서가 ESLint 규칙과 맞지 않음

**해결**: 다음 순서로 import 정리
1. Vue 관련 imports
2. 내부 API imports
3. 타입 imports
4. UI 컴포넌트 imports

### 3. 사용하지 않는 변수 제거

**문제**: TypeScript에서 사용하지 않는 변수로 인한 경고

**해결**: 필요없는 import 및 변수들 제거

## 📈 향후 개선 계획

### 1. 기능 확장
- 조직 생성/수정/삭제 기능 추가
- 조직 계층 구조 드래그 앤 드롭
- 권한 관리 기능 연동

### 2. 성능 최적화
- 가상화 스크롤링으로 대용량 데이터 처리
- 무한 스크롤 페이징 구현
- 검색 디바운싱 적용

### 3. 사용자 경험 개선
- 로딩 스켈레톤 UI 적용
- 토스트 알림 시스템 연동
- 키보드 네비게이션 지원

## 📚 학습 포인트

1. **실제 API 우선 접근법**의 중요성
2. **FSD 아키텍처**의 확장성과 유지보수성
3. **TypeScript**를 활용한 타입 안전성 확보
4. **React Query**를 통한 서버 상태 관리
5. **Vue 3 Composition API**의 효과적 활용

---

**구현 완료일**: 2025년 1월 9일  
**구현 시간**: 약 4시간  
**구현자**: 조기현  
**관련 문서**: [Confluence 가이드](https://okestro.atlassian.net/wiki/spaces/PCD/pages/2154955523)