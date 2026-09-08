# 스웨거 API 컴포넌트 구현 가이드

## 📖 개요

이 문서는 Swagger API를 기반으로 Vue.js 컴포넌트를 구현하는 표준 워크플로우와 아키텍처 패턴을 정리한 것입니다. 실제 조직(Organization) API 구현 사례를 통해 개발자들이 일관된 방식으로 API 컴포넌트를 개발할 수 있도록 가이드를 제공합니다.

## 🏗️ 아키텍처 패턴

### FSD (Feature-Sliced Design) 구조

```
packages/cmp/src/
├── features/
│   └── {feature}-api/
│       ├── api/
│       │   ├── index.ts                 # API 함수 정의
│       │   ├── queryKeys.ts             # Query Key 상수
│       │   ├── use{Feature}ListQuery.ts # 목록 조회 훅
│       │   └── use{Feature}DetailQuery.ts # 상세 조회 훅
│       ├── ui/
│       │   ├── {Feature}Table.vue       # 테이블 컴포넌트
│       │   └── {Feature}DetailPanel.vue # 상세 패널 컴포넌트
│       ├── types.ts                     # 타입 정의
│       └── index.ts                     # Public exports
└── pages/
    └── {feature}/
        ├── ui/
        │   └── {Feature}ListPage.vue    # 페이지 컴포넌트
        └── index.ts                     # 라우트 정의
```

## 📋 4단계 표준 워크플로우

### 1단계: 스웨거 접근성 확인

**목적**: API 서버 연결 상태와 인증 방식 확인

```bash
# 기본 연결 테스트
curl -I https://dev.admin.okestro.cloud/v1/maestro/admin

# 스웨거 UI 접근 테스트
curl -I https://dev.admin.okestro.cloud/v1/maestro/admin/webjars/swagger-ui/index.html
```

**성공 기준**:
- HTTP 200 또는 302 (인증 리다이렉트) 응답
- SSL 인증서 유효성 확인
- 인증 방식 식별

### 2단계: 토큰 유효성 검증

**목적**: OAuth2 토큰 존재 및 유효성 확인

```bash
# 환경 파일에서 토큰 확인
grep "VITE_OAUTH2_COOKIE=" apps/okestro-cmp-app-admin/.env.local

# 토큰 유효성 테스트
COOKIE=$(grep "VITE_OAUTH2_COOKIE=" apps/okestro-cmp-app-admin/.env.local | cut -d '=' -f2- | tr -d '"')
curl -k -H "Cookie: _oauth2_proxy=$COOKIE" https://dev.admin.okestro.cloud/v1/maestro/admin/api-docs
```

**성공 기준**:
- API 문서에 HTTP 200으로 접근 가능
- JSON 형태의 OpenAPI 스펙 응답 수신
- 토큰이 `.env.local`에 올바르게 저장됨

### 3단계: API 스펙 추출 및 검증

**목적**: OpenAPI 문서 분석 및 실제 API 동작 확인

```bash
# API 문서 다운로드 및 분석
COOKIE=$(grep "VITE_OAUTH2_COOKIE=" apps/okestro-cmp-app-admin/.env.local | cut -d '=' -f2- | tr -d '"')
curl -k -H "Cookie: _oauth2_proxy=$COOKIE" "https://dev.admin.okestro.cloud/v1/maestro/admin/api-docs" > swagger-spec.json

# 특정 API 엔드포인트 확인
cat swagger-spec.json | jq '.paths | keys[]' | grep -i {target-api}

# 실제 API 테스트
curl -k -H "Cookie: _oauth2_proxy=$COOKIE" "https://dev.admin.okestro.cloud/v1/maestro/admin/{api-endpoint}"
```

**성공 기준**:
- OpenAPI 스펙 다운로드 성공
- 대상 API 엔드포인트 존재 확인
- 실제 API 호출로 데이터 응답 수신
- 응답 스키마 구조 파악

### 4단계: 구현

**목적**: FSD 아키텍처 패턴에 따른 컴포넌트 구현

## 💻 코드 구현 패턴

### 1. 타입 정의 (types.ts)

```typescript
// 실제 API 응답을 기반으로 한 타입 정의
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
  key: string;
  orgId: string;
  depth: number;
  // API 응답에서 확인된 추가 필드들
}

export interface GetOrgListReqType {
  page?: number;
  size?: number;
  sort?: string[];
  status?: number;
  workspaceName?: string;
}

export interface ApiResponse<T> {
  data: T;
  status?: number;
  message?: string;
}
```

### 2. API 함수 (api/index.ts)

```typescript
import { adminApiInstance } from '@shared-api';

export const ORG_API_ENDPOINTS = {
  list: '/org/org-only',
  simple: '/org/simple',
  detail: (orgId: string) => `/org/${orgId}`,
} as const;

export const getOrgList = (params?: GetOrgListReqType) => {
  return adminApiInstance.get<ApiResponse<OrgType[]>>(
    ORG_API_ENDPOINTS.list,
    { params }
  );
};
```

### 3. Query Keys (api/queryKeys.ts)

```typescript
export const orgQueryKeys = {
  all: ['org'] as const,
  lists: () => [...orgQueryKeys.all, 'list'] as const,
  list: (params?: GetOrgListReqType) => [...orgQueryKeys.lists(), params] as const,
  details: () => [...orgQueryKeys.all, 'detail'] as const,
  detail: (orgId: string) => [...orgQueryKeys.details(), orgId] as const,
};
```

### 4. Query Hook (api/useOrgListQuery.ts)

```typescript
import { useQuery } from '@tanstack/vue-query';

export const useOrgListQuery = (params?: GetOrgListReqType) => {
  return useQuery<AxiosResponse<ApiResponse<OrgType[]>>, Error>({
    queryKey: orgQueryKeys.list(params),
    queryFn: () => getOrgList(params),
  });
};
```

### 5. 테이블 컴포넌트 (ui/OrgTable.vue)

```vue
<template>
  <CmpListLayout>
    <template #leftPanel>
      <CmpTableFilter
        v-model="filteredList"
        :columns="props.columns"
        :data="list"
      />
    </template>
    <CmpTable
      :columns="props.columns"
      :data="filteredList"
      :loading="isLoading"
      :filter-buttons="filterButtons"
      :show-date-filter="{
        columnKey: 'createTime',
      }"
    >
      <template #bodyCell="{ column, record }">
        <!-- 커스텀 셀 렌더링 -->
      </template>
    </CmpTable>
  </CmpListLayout>
</template>

<script setup lang="ts">
interface Props {
  columns: CmpTableColumnType[];
  searchParams?: GetOrgListReqType;
}

const props = defineProps<Props>();
const emit = defineEmits<{
  selectItem: [item: OrgType];
}>();

// API 데이터 조회
const { data: response, isLoading } = useOrgListQuery(props.searchParams);

// 데이터 변환
const list = computed(() => {
  if (!response.value) return [];
  const data = response.value.data?.data;
  return Array.isArray(data) ? data : [];
});
</script>
```

### 6. 페이지 컴포넌트 (pages/org/ui/OrgListPage.vue)

```vue
<template>
  <TheTabLayout v-model:is-panel-open="isPanelOpen" :tab-list="tabItems">
    <template #default="{ activeKey }">
      <OrgTable
        v-if="activeKey === 'list'"
        :columns="tableColumns"
        @select-item="handleSelectItem"
      />
    </template>
    <template #rightPanel="{ activeKey }">
      <CmpPushSidePanel v-model:is-open="isPanelOpen">
        <template v-if="isPanelOpen && activeKey === 'list' && selectedItemId">
          <OrgDetailPanel :item-id="selectedItemId" />
        </template>
      </CmpPushSidePanel>
    </template>
  </TheTabLayout>
</template>

<script setup lang="ts">
// 테이블 컬럼 정의 (페이지에서 관리)
const tableColumns: CmpTableColumnType[] = [
  {
    title: 'Name',
    dataIndex: 'name',
    key: 'name',
    type: 'link',
    searchFilter: true,
    typeOption: {
      onClick: (record: OrgType) => {
        handleSelectItem(record);
      },
    },
  },
  // 다른 컬럼들...
];
</script>
```

## 🔧 라우트 및 메뉴 등록

### 라우트 등록 (pages/org/index.ts)

```typescript
const orgRoutes: RouteRecordRaw[] = [
  {
    path: 'org',
    name: 'CmpOrgListPage',
    component: () => import('./ui/OrgListPage.vue'),
    meta: {
      title: 'Organization Management',
      requiresAuth: true,
      roles: ['ROOT', 'ADMIN'],
    },
  },
];
```

### 메뉴 등록 (menuItemList.json)

```json
{
  "key": "menu-org",
  "link": "cmp/org",
  "name": "menu-org",
  "meta": { "title": "조직 관리" }
}
```

## 📝 구현 체크리스트

### ✅ API 계층
- [ ] API 함수 정의 (`api/index.ts`)
- [ ] Query Keys 정의 (`api/queryKeys.ts`) 
- [ ] Query Hooks 구현 (`api/use*Query.ts`)
- [ ] 타입 정의 (`types.ts`)

### ✅ UI 계층
- [ ] 테이블 컴포넌트 (`ui/*Table.vue`)
- [ ] 상세 패널 컴포넌트 (`ui/*DetailPanel.vue`)
- [ ] 페이지 컴포넌트 (`pages/*/ui/*Page.vue`)

### ✅ 연결 계층
- [ ] 라우트 등록 (`pages/*/index.ts`)
- [ ] 메인 라우트 파일 업데이트 (`pages/index.ts`)
- [ ] 메뉴 등록 (`menuItemList.json`)

### ✅ 품질 확인
- [ ] Lint 오류 수정
- [ ] TypeScript 오류 해결
- [ ] 실제 API 테스트
- [ ] 브라우저 동작 확인

## 🎯 핵심 원칙

1. **실제 API 우선**: Swagger 문서보다 실제 API 응답을 기준으로 구현
2. **일관성 유지**: 모든 기능에 동일한 패턴 적용
3. **타입 안전성**: TypeScript를 활용한 컴파일 타임 오류 방지
4. **재사용성**: 컴포넌트 간 Props/Emits를 통한 데이터 흐름
5. **성능 최적화**: React Query를 활용한 캐싱 및 상태 관리

## 🚀 자동화 스크립트

추후 개발 효율성을 위해 다음과 같은 자동화 스크립트 개발 예정:

```bash
# 전체 워크플로우 실행
bash tools/swagger-workflow.sh {api-name}

# 단계별 실행
bash tools/check-swagger-access.sh
bash tools/verify-token.sh
bash tools/extract-api-spec.sh {api-name}
bash tools/generate-feature.sh {api-name}
```

## 📚 참고 자료

- [FSD 아키텍처 가이드](https://feature-sliced.design/)
- [Vue 3 Composition API](https://vuejs.org/guide/extras/composition-api-faq.html)
- [TanStack Query](https://tanstack.com/query/latest)
- [Ant Design Vue](https://antdv.com/)

---

**작성일**: 2025년 1월 9일  
**마지막 업데이트**: 조직 API 구현 완료 후 표준화  
**작성자**: 조기현