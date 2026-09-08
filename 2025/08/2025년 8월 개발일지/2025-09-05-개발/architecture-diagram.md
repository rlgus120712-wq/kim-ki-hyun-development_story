# 스웨거 API 컴포넌트 아키텍처 다이어그램

## 🏗️ 전체 시스템 아키텍처

```mermaid
graph TB
    subgraph "브라우저 (Client)"
        subgraph "Vue 애플리케이션"
            Router["Vue Router"]
            Store["Pinia Store"]
            Query["TanStack Query"]
        end
        
        subgraph "페이지 레이어"
            OrgPage["OrgListPage.vue"]
            UserPage["UserListPage.vue"]
            WorkspacePage["WorkspaceListPage.vue"]
        end
        
        subgraph "Feature 레이어"
            OrgFeature["org-api"]
            UserFeature["user-api"]
            WorkspaceFeature["workspace-api"]
        end
        
        subgraph "Shared 레이어"
            SharedUI["@shared-ui"]
            SharedAPI["@shared-api"]
            SharedCommon["@shared-common"]
        end
    end
    
    subgraph "네트워크"
        OAuth["OAuth2 Proxy"]
        LoadBalancer["Load Balancer"]
    end
    
    subgraph "백엔드 서버"
        AdminAPI["Admin API Server"]
        SwaggerUI["Swagger UI"]
        Database[("Database")]
    end
    
    %% 연결 관계
    Router --> OrgPage
    Router --> UserPage
    Router --> WorkspacePage
    
    OrgPage --> OrgFeature
    UserPage --> UserFeature
    WorkspacePage --> WorkspaceFeature
    
    OrgFeature --> SharedAPI
    UserFeature --> SharedAPI
    WorkspaceFeature --> SharedAPI
    
    OrgFeature --> SharedUI
    UserFeature --> SharedUI
    WorkspaceFeature --> SharedUI
    
    SharedAPI --> OAuth
    OAuth --> LoadBalancer
    LoadBalancer --> AdminAPI
    AdminAPI --> Database
    AdminAPI --> SwaggerUI
```

## 📦 FSD (Feature-Sliced Design) 구조

```mermaid
graph LR
    subgraph "FSD 레이어 구조"
        subgraph "app"
            AppLayer["App Layer<br/>- 앱 진입점<br/>- 전역 설정<br/>- 라우터 설정"]
        end
        
        subgraph "pages"
            PagesLayer["Pages Layer<br/>- 라우트별 페이지<br/>- 레이아웃 구성<br/>- Feature 조합"]
        end
        
        subgraph "features"
            FeaturesLayer["Features Layer<br/>- 비즈니스 로직<br/>- API 호출<br/>- 상태 관리"]
        end
        
        subgraph "entities"
            EntitiesLayer["Entities Layer<br/>- 도메인 모델<br/>- 비즈니스 엔티티<br/>- 데이터 스키마"]
        end
        
        subgraph "shared"
            SharedLayer["Shared Layer<br/>- 공통 컴포넌트<br/>- 유틸리티<br/>- API 클라이언트"]
        end
    end
    
    AppLayer --> PagesLayer
    PagesLayer --> FeaturesLayer
    FeaturesLayer --> EntitiesLayer
    FeaturesLayer --> SharedLayer
    EntitiesLayer --> SharedLayer
```

## 🔄 데이터 플로우 다이어그램

```mermaid
sequenceDiagram
    participant User as 사용자
    participant Page as OrgListPage
    participant Table as OrgTable
    participant Hook as useOrgListQuery
    participant API as adminApiInstance
    participant Server as Admin Server
    
    User->>Page: 페이지 접근
    Page->>Table: 컴포넌트 렌더링
    Table->>Hook: 데이터 요청
    Hook->>API: HTTP GET 요청
    API->>Server: /org/org-only
    
    Server-->>API: JSON 응답
    API-->>Hook: AxiosResponse
    Hook-->>Table: 캐시된 데이터
    Table-->>Page: 테이블 렌더링
    Page-->>User: UI 표시
    
    User->>Table: 행 클릭
    Table->>Page: selectItem 이벤트
    Page->>Page: 상세 패널 오픈
```

## 🧩 컴포넌트 구조 다이어그램

```mermaid
graph TD
    subgraph "조직 관리 기능"
        OrgPage["📄 OrgListPage.vue<br/>- 메인 페이지<br/>- 탭 레이아웃<br/>- 상태 관리"]
        
        subgraph "UI 컴포넌트"
            OrgTable["📊 OrgTable.vue<br/>- 데이터 테이블<br/>- 필터링<br/>- 정렬"]
            
            OrgDetail["📋 OrgDetailPanel.vue<br/>- 상세 정보<br/>- 하위 구조<br/>- 액션 버튼"]
            
            Filter["🔍 CmpTableFilter<br/>- 좌측 필터 패널<br/>- 검색 기능"]
        end
        
        subgraph "데이터 레이어"
            ListQuery["🔄 useOrgListQuery<br/>- 목록 조회<br/>- 캐싱"]
            
            DetailQuery["🔄 useOrgDetailQuery<br/>- 상세 조회<br/>- 실시간 업데이트"]
            
            APIFunctions["🌐 API Functions<br/>- HTTP 요청<br/>- 에러 처리"]
        end
    end
    
    OrgPage --> OrgTable
    OrgPage --> OrgDetail
    OrgTable --> Filter
    
    OrgTable --> ListQuery
    OrgDetail --> DetailQuery
    
    ListQuery --> APIFunctions
    DetailQuery --> APIFunctions
```

## 🔐 인증 플로우 다이어그램

```mermaid
sequenceDiagram
    participant Dev as 개발자
    participant Browser as 브라우저
    participant Proxy as OAuth2 Proxy
    participant Server as Admin Server
    
    Dev->>Browser: pnpm local 실행
    Browser->>Proxy: 인증 요청
    Proxy->>Proxy: OAuth2 인증 처리
    Proxy-->>Browser: 쿠키 설정
    Browser-->>Dev: 로그인 완료
    
    Note over Browser: _oauth2_proxy 쿠키 저장
    
    Browser->>Server: API 요청 + 쿠키
    Server->>Proxy: 쿠키 검증
    Proxy-->>Server: 인증 확인
    Server-->>Browser: API 응답
```

## 📊 상태 관리 다이어그램

```mermaid
stateDiagram-v2
    [*] --> Loading: API 요청
    Loading --> Success: 데이터 수신
    Loading --> Error: 요청 실패
    
    Success --> Loading: 새로고침
    Success --> Cache: 캐시된 데이터
    
    Error --> Loading: 재시도
    
    Cache --> Success: 캐시 만료
    Cache --> Loading: 강제 새로고침
    
    state Loading {
        [*] --> Fetching
        Fetching --> Processing
        Processing --> [*]
    }
    
    state Success {
        [*] --> Rendering
        Rendering --> Interactive
        Interactive --> [*]
    }
    
    state Error {
        [*] --> ErrorDisplay
        ErrorDisplay --> RetryButton
        RetryButton --> [*]
    }
```

## 🛠️ 빌드 및 배포 파이프라인

```mermaid
graph LR
    subgraph "개발 환경"
        Dev["💻 로컬 개발"]
        Lint["🔍 ESLint"]
        Type["📝 TypeScript"]
        Test["🧪 테스트"]
    end
    
    subgraph "빌드 과정"
        Bundle["📦 Vite 빌드"]
        Optimize["⚡ 최적화"]
        Assets["🎨 에셋 처리"]
    end
    
    subgraph "배포"
        Staging["🔄 스테이징"]
        Production["🚀 프로덕션"]
    end
    
    Dev --> Lint
    Lint --> Type
    Type --> Test
    Test --> Bundle
    Bundle --> Optimize
    Optimize --> Assets
    Assets --> Staging
    Staging --> Production
```

## 📁 파일 구조 맵

```
packages/cmp/src/features/org-api/
├── 📂 api/
│   ├── 📄 index.ts                    # API 엔드포인트 및 함수 정의
│   ├── 📄 queryKeys.ts                # React Query 키 관리
│   ├── 📄 useOrgListQuery.ts          # 목록 조회 훅
│   ├── 📄 useSimpleOrgListQuery.ts    # 단순 목록 훅
│   └── 📄 useOrgDetailQuery.ts        # 상세 조회 훅
├── 📂 ui/
│   ├── 📄 OrgTable.vue                # 조직 목록 테이블
│   └── 📄 OrgDetailPanel.vue          # 조직 상세 패널
├── 📄 types.ts                        # TypeScript 타입 정의
└── 📄 index.ts                        # Public API exports

packages/cmp/src/pages/org/
├── 📂 ui/
│   └── 📄 OrgListPage.vue             # 메인 페이지 컴포넌트
└── 📄 index.ts                        # Vue Router 정의
```

## 🎯 성능 최적화 포인트

```mermaid
mindmap
  root((성능 최적화))
    번들 최적화
      Tree Shaking
      Code Splitting
      Lazy Loading
    
    렌더링 최적화
      Virtual Scrolling
      Computed Properties
      Memo Components
    
    네트워크 최적화
      HTTP Caching
      Request Debouncing
      Parallel Requests
    
    메모리 최적화
      Query Cache
      Component Cleanup
      Event Listener 해제
```

---

**다이어그램 작성일**: 2025년 1월 9일  
**도구**: Mermaid.js  
**작성자**: 조기현