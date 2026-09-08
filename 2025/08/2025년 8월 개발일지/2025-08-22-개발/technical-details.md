# 🔧 기술적 구현 세부사항

## 📁 파일 구조 및 역할

### Service Template 모듈
```
packages/cmp/src/features/service-template/
├── hooks/
│   ├── useUpdateModal.ts          # 모달 상태 관리
│   └── useCmpStepForm.ts         # 다단계 폼 관리
└── api/
    └── useUpdateTemplateMutation.ts # API 상태 관리
```

### Push Side Panel Sample 모듈
```
packages/cmp/src/features/push-side-panel-sample/
├── hooks/
│   └── useUpdateModal.ts          # 사이드 패널 모달 관리
└── api/
    └── useUpdateTemplateMutation.ts # 패널 업데이트 API
```

### A-Sample 모듈
```
packages/cmp/src/features/a-sample/
└── hooks/
    └── useCmpStepForm.ts         # 샘플 폼 관리
```

---

## 🎯 핵심 Hook 구현 패턴

### 1. useUpdateModal Hook 패턴

```typescript
// 공통 모달 관리 패턴
export const useUpdateModal = () => {
  const isVisible = ref(false)
  const loading = ref(false)
  const formData = ref({})

  const openModal = (data?: any) => {
    if (data) {
      formData.value = { ...data }
    }
    isVisible.value = true
  }

  const closeModal = () => {
    isVisible.value = false
    formData.value = {}
  }

  const handleSubmit = async () => {
    loading.value = true
    try {
      // API 호출 로직
      await updateMutation.mutateAsync(formData.value)
      closeModal()
    } catch (error) {
      console.error('Update failed:', error)
    } finally {
      loading.value = false
    }
  }

  return {
    isVisible,
    loading,
    formData,
    openModal,
    closeModal,
    handleSubmit
  }
}
```

### 2. useUpdateTemplateMutation Hook 패턴

```typescript
// Vue Query 기반 API 상태 관리
export const useUpdateTemplateMutation = () => {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: async (data: UpdateTemplateRequest) => {
      return await templateApi.update(data)
    },
    onSuccess: (data) => {
      // 캐시 무효화
      queryClient.invalidateQueries(['templates'])
      
      // 낙관적 업데이트
      queryClient.setQueryData(['template', data.id], data)
      
      // 성공 알림
      notification.success('템플릿이 성공적으로 업데이트되었습니다.')
    },
    onError: (error) => {
      console.error('Template update failed:', error)
      notification.error('템플릿 업데이트에 실패했습니다.')
    }
  })
}
```

### 3. useCmpStepForm Hook 패턴

```typescript
// 다단계 폼 관리 시스템
export const useCmpStepForm = <T extends Record<string, any>>(
  initialData: T,
  validationSchema?: any
) => {
  const currentStep = ref(0)
  const formData = reactive<T>({ ...initialData })
  const errors = ref<Record<string, string>>({})

  const validateStep = (step: number) => {
    if (!validationSchema) return true
    
    try {
      validationSchema.validateSyncAt(`step${step}`, formData)
      delete errors.value[`step${step}`]
      return true
    } catch (error) {
      errors.value[`step${step}`] = error.message
      return false
    }
  }

  const nextStep = () => {
    if (validateStep(currentStep.value)) {
      currentStep.value++
    }
  }

  const prevStep = () => {
    if (currentStep.value > 0) {
      currentStep.value--
    }
  }

  const submitForm = async () => {
    // 전체 폼 검증
    for (let i = 0; i <= currentStep.value; i++) {
      if (!validateStep(i)) {
        currentStep.value = i
        return false
      }
    }
    return true
  }

  return {
    currentStep: readonly(currentStep),
    formData,
    errors: readonly(errors),
    nextStep,
    prevStep,
    validateStep,
    submitForm
  }
}
```

---

## 🏗 아키텍처 설계 원칙

### 1. Composition API 활용
- **재사용성**: 로직을 독립적인 함수로 분리
- **타입 안전성**: TypeScript Generic 활용
- **반응성**: Vue 3의 반응형 시스템 최대 활용

### 2. Feature-Sliced Design (FSD)
```
features/
├── service-template/     # 서비스 템플릿 기능
├── push-side-panel/     # 사이드 패널 기능  
└── a-sample/            # 샘플 기능

각 feature는 독립적이며:
├── api/                 # API 레이어
├── hooks/               # 비즈니스 로직
├── components/          # UI 컴포넌트
└── types/               # 타입 정의
```

### 3. 상태 관리 전략
- **로컬 상태**: `ref`, `reactive` 활용
- **서버 상태**: Vue Query로 캐싱 및 동기화
- **글로벌 상태**: Pinia 스토어 활용

---

## 🔄 데이터 플로우

### 1. 사용자 인터랙션 플로우
```
사용자 액션 → Hook 호출 → API 요청 → 상태 업데이트 → UI 리렌더링
```

### 2. 에러 핸들링 플로우
```
API 에러 → Mutation onError → 에러 상태 설정 → 사용자 알림
```

### 3. 캐시 관리 플로우
```
API 성공 → 캐시 무효화 → 자동 리페치 → UI 업데이트
```

---

## 🧪 테스트 전략

### 단위 테스트 (예정)
```typescript
// Hook 테스트 예시
describe('useUpdateModal', () => {
  it('should open modal with data', () => {
    const { openModal, isVisible, formData } = useUpdateModal()
    const testData = { id: 1, name: 'test' }
    
    openModal(testData)
    
    expect(isVisible.value).toBe(true)
    expect(formData.value).toEqual(testData)
  })
})
```

### 통합 테스트 (예정)
- API 호출 및 응답 테스트
- 컴포넌트 간 상호작용 테스트
- 에러 시나리오 테스트

---

## 📊 성능 최적화

### 1. 메모이제이션
```typescript
// computed를 활용한 계산 캐싱
const processedData = computed(() => {
  return expensiveCalculation(formData.value)
})
```

### 2. 지연 로딩
```typescript
// 동적 import를 활용한 코드 스플리팅
const HeavyComponent = defineAsyncComponent(() => 
  import('./HeavyComponent.vue')
)
```

### 3. 가상화
- 대용량 리스트 렌더링 최적화
- 무한 스크롤 구현

---

## 🔧 개발 도구 설정

### TypeScript 설정
```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true
  }
}
```

### ESLint 규칙
- Vue 3 Composition API 규칙
- TypeScript 엄격 모드
- 접근성 규칙

### Vite 최적화
- 번들 분석 및 최적화
- 개발 서버 성능 튜닝
- 빌드 시간 단축

---

## 🚀 배포 및 CI/CD

### 빌드 프로세스
1. **린트 검사**: ESLint + Prettier
2. **타입 체크**: TypeScript 컴파일
3. **테스트 실행**: Vitest 단위 테스트
4. **번들 빌드**: Vite 프로덕션 빌드
5. **배포**: Docker 컨테이너화

### 모니터링
- 성능 메트릭 수집
- 에러 추적 및 알림
- 사용자 행동 분석

---

**작성일**: 2025년 8월 22일  
**업데이트**: 기술적 구현 세부사항 추가