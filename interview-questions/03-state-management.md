# 상태 관리 질문

## React Query

### Q1. Redux → React Query 마이그레이션으로 보일러플레이트 60%를 제거한 과정을 설명해주세요.
> **출제 의도**: 상태 관리 기술 선택 근거와 마이그레이션 전략
>
> **경력 연관**: 마이리얼트립 렌터카/숙소 고도화

<details>
<summary><b>✅ 답변</b></summary>

**Redux 문제점**: 서버 데이터를 다루기 위해 Action Type 정의 → Action Creator → Reducer → Saga/Thunk → Selector까지 하나의 API 호출에 5개 파일을 수정해야 했습니다. 로딩/에러/성공 상태를 수동 관리하고, 캐시 무효화 로직도 직접 구현해야 했습니다.

**마이그레이션 전략 (점진적 접근)**:
1. **1단계**: 신규 기능부터 React Query 적용. 기존 Redux 코드는 유지
2. **2단계**: 페이지 단위로 Redux 의존성을 React Query로 전환. 하나의 페이지가 양쪽 모두 사용하는 과도기 허용
3. **3단계**: Redux에서 서버 상태를 완전히 제거하고, 순수 클라이언트 상태(UI 토글, 모달 열림 등)만 남김

**구체적 변환 예시**:
- Redux: `fetchProductsAction` → `FETCH_PRODUCTS_REQUEST/SUCCESS/FAILURE` → reducer → selector → 컴포넌트
- React Query: `useQuery({ queryKey: ['products', filters], queryFn: fetchProducts })` 한 줄로 대체

**결과**: git diff stat 기준 관련 코드 60% 감소. 캐싱, 리패칭, 에러/로딩 처리가 자동화되어 개발 속도도 체감적으로 크게 향상되었습니다.

</details>

**꼬리 질문**:
- Redux가 적합한 경우와 React Query가 적합한 경우의 기준은?
- 서버 상태와 클라이언트 상태를 어떻게 구분하나요?
- 점진적 마이그레이션에서 두 라이브러리가 공존하는 시기의 관리는?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **적합성 기준**: 서버에서 가져오는 데이터(API 응답)는 React Query, 순수 클라이언트 상태(모달 열림, 사이드바 토글, 폼 입력값, 위자드 현재 단계 등)는 Redux나 Context API가 적합합니다. 복잡한 클라이언트 상태 전환이 있는 경우(워크플로우, 상태 머신 등)에는 Redux/Zustand가 여전히 유효합니다.

- **구분 기준**: "이 데이터의 Source of Truth가 서버인가, 클라이언트인가?"로 구분합니다. 상품 목록, 사용자 정보 등은 서버 상태(다른 클라이언트도 같은 데이터를 공유), 테마 설정, UI 상태 등은 클라이언트 상태(해당 클라이언트에서만 의미)입니다.

- **공존 시기 관리**: 패키지 의존성이 늘어나지만, 기능 단위로 격리되어 있으면 충돌이 적었습니다. 규칙으로 "신규 코드에서 Redux로 서버 상태를 추가하지 않는다"를 정했고, PR 리뷰에서 이를 체크했습니다. Redux DevTools와 React Query DevTools를 동시에 활용하여 디버깅했습니다.

</details>

---

### Q2. React Query의 `staleTime`/`cacheTime` 최적화로 API 호출 40%를 절감한 전략을 설명해주세요.
> **출제 의도**: React Query 캐싱 전략 실무 이해
>
> **경력 연관**: 마이리얼트립 통합숙소

<details>
<summary><b>✅ 답변</b></summary>

**데이터 성격별 차등 캐싱 전략**:

| 데이터 유형 | staleTime | gcTime(cacheTime) | 이유 |
|------------|-----------|-------------------|------|
| 상품 목록 | 30초 | 5분 | 가격/재고가 자주 변동 |
| 상품 상세 | 2분 | 10분 | 뒤로 가기 시 즉시 표시 필요 |
| 카테고리/필터 옵션 | 10분 | 30분 | 변경 빈도 매우 낮음 |
| 사용자 정보 | 5분 | 30분 | 세션 동안 변경 드뭄 |

**핵심 원칙**:
1. `staleTime` 동안은 refetch하지 않으므로, 데이터 변경 빈도에 맞게 설정
2. `gcTime`은 컴포넌트 언마운트 후에도 캐시를 유지하는 시간 → 뒤로 가기 시 API 호출 없이 즉시 표시
3. `refetchOnWindowFocus: false`를 기본값으로 설정(관리자 도구가 아닌 일반 서비스이므로 포커스마다 refetch 불필요)

**측정**: Datadog APM에서 API 호출 횟수를 before/after로 비교하여 40% 감소를 확인했습니다.

</details>

**꼬리 질문**:
- Query Key 설계 전략은?
- `invalidateQueries`와 `setQueryData`의 사용 기준은?
- Optimistic Update 적용 경험은?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **Query Key 설계**: 계층 구조로 설계했습니다. `['products']` → `['products', 'list', { category, page, sort }]` → `['products', 'detail', productId]`. 이렇게 하면 `queryClient.invalidateQueries({ queryKey: ['products'] })`로 상품 관련 모든 캐시를 한번에 무효화할 수 있고, 상세 페이지만 개별 무효화도 가능합니다. Query Key Factory 함수를 만들어 팀 내 일관성을 유지했습니다.

- **invalidate vs setQueryData**: `invalidateQueries`는 캐시를 stale로 만들어 다음 접근 시 refetch. `setQueryData`는 캐시 데이터를 직접 수정. 일반적으로 mutation 후 관련 목록을 갱신할 때는 `invalidateQueries`(서버 데이터가 정확), 즉각적인 UI 반영이 필요한 좋아요/북마크 같은 경우 `setQueryData`로 로컬 캐시를 직접 업데이트했습니다.

- **Optimistic Update**: 숙소 찜하기(즐겨찾기) 기능에 적용했습니다. 버튼 클릭 즉시 UI를 업데이트하고 API를 호출. 실패 시 `onError` 콜백에서 `context.previousData`로 롤백했습니다. 사용자 경험이 즉각적으로 느껴지면서 서버 에러 시 자연스럽게 원복되는 패턴입니다.

</details>

---

### Q3. 쿼리스트링 ↔ UI 상태 양방향 싱크(deep-link 지원)는 어떻게 구현했나요?
> **출제 의도**: URL 상태 관리와 사용자 경험 설계
>
> **경력 연관**: 마이리얼트립 통합숙소

<details>
<summary><b>✅ 답변</b></summary>

**구현 구조**: 커스텀 훅 `useQueryParams`를 만들어 URL 쿼리스트링을 Single Source of Truth로 사용했습니다.

```
URL 쿼리스트링 ← useQueryParams 훅 → UI 컴포넌트
     ↕ (양방향)              ↕
router.push/replace       React Query (queryKey에 params 포함)
```

**핵심 구현**:
1. **URL → UI**: `useRouter`의 `query`에서 파라미터를 읽어 Zod 스키마로 파싱/검증. 유효하지 않은 값은 기본값으로 대체
2. **UI → URL**: 필터 변경 시 `router.replace`로 URL을 업데이트 (브라우저 히스토리 남기지 않음, 단 페이지네이션은 `router.push`)
3. **URL → API**: React Query의 queryKey에 파싱된 params를 포함시켜 URL 변경 시 자동으로 refetch
4. **직렬화/역직렬화**: 배열 값은 `?category=hotel&category=pension` 형태, 범위 값은 `?price=100000-500000` 형태로 인코딩

**결과**: 검색 결과 페이지의 URL을 공유하면 동일한 필터 상태가 복원되는 deep-link 지원. 뒤로 가기 시 이전 필터 상태로 자연스럽게 복원.

</details>

**꼬리 질문**:
- 쿼리스트링 변경 시 불필요한 리렌더링 방지는?
- 복잡한 필터 상태의 직렬화 전략은?
- 뒤로 가기 시 이전 상태 복원은?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **리렌더링 방지**: `useMemo`로 파싱된 params 객체를 메모이제이션하여, URL은 변경되었지만 실제 파라미터 값이 동일한 경우(순서만 다른 등) 리렌더링을 방지했습니다. 또한 `router.replace`는 shallow 옵션을 사용하여 `getServerSideProps` 재실행 없이 클라이언트에서만 URL을 업데이트했습니다.

- **직렬화 전략**: 단순 문자열/숫자는 그대로, 배열은 같은 키 반복(`category=a&category=b`), 날짜는 ISO 형식(`checkIn=2024-01-01`), 중첩 객체는 flat하게 변환(`priceMin=100000&priceMax=500000`). URL 길이 제한(약 2000자)을 고려하여 불필요한 기본값은 URL에서 제거했습니다.

- **뒤로 가기 복원**: `router.push`로 히스토리 스택에 쌓인 URL이 뒤로 가기 시 `popstate` 이벤트로 트리거됩니다. `useQueryParams` 훅이 새 URL의 쿼리스트링을 읽어 UI를 업데이트하고, React Query가 해당 파라미터로 캐시를 확인합니다. 캐시가 있으면 즉시 표시, 없으면 refetch합니다.

</details>

---

## 폼 상태 관리

### Q4. React Hook Form + Zod를 사용한 다단계 폼 Wizard 구현에 대해 설명해주세요.
> **출제 의도**: 복잡한 폼 상태 관리 설계 역량
>
> **경력 연관**: 이그나이트 딜러포탈

<details>
<summary><b>✅ 답변</b></summary>

**아키텍처**:

1. **Zod 스키마 분리**: 전체 폼 스키마를 단계별 서브 스키마로 분리하고, `z.union`으로 결합. 각 단계에서는 해당 단계의 스키마만 검증
2. **React Hook Form의 `useForm`을 최상위에서 한 번만 호출**: 모든 단계가 동일한 `form` 인스턴스를 공유하여 단계 전환 시 데이터 유실 방지
3. **단계별 컴포넌트**: 각 단계를 별도 컴포넌트로 분리하고, `useFormContext`로 form 인스턴스에 접근
4. **비제어 컴포넌트(Uncontrolled)**: React Hook Form의 `register`를 사용하여 DOM에 직접 값을 저장. 입력마다 state 업데이트가 발생하지 않아 렌더링 성능 크게 개선

**단계 전환 로직**:
```
이전 단계 ← [현재 단계의 Zod 검증 통과] → 다음 단계
                    ↓ (실패)
              에러 메시지 표시
```

**React Profiler 측정 결과**: 비제어 컴포넌트 전환으로 폼 필드 30개 기준 입력 시 렌더링 횟수가 약 85% 감소했습니다.

</details>

**꼬리 질문**:
- 단계 간 데이터 의존성 처리는?
- Zod 스키마를 단계별로 분리한 이유는?
- 서버 사이드 밸리데이션과 클라이언트 밸리데이션의 역할 분담은?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **데이터 의존성**: `useWatch`로 특정 필드 값을 구독하여 다음 단계의 옵션을 동적으로 변경했습니다. 예를 들어 1단계에서 선택한 차량 브랜드에 따라 2단계의 모델 목록이 달라지는 경우, `useWatch({ name: 'brand' })`로 브랜드 값을 읽어 모델 API를 호출합니다. `useWatch`는 해당 필드만 구독하므로 다른 필드 변경 시 리렌더링되지 않습니다.

- **스키마 분리 이유**: 전체 스키마로 한 번에 검증하면 미래 단계의 필수 필드가 현재 단계에서 에러를 발생시킵니다. 단계별로 `resolver`를 교체하면 복잡해지므로, 서브 스키마로 분리하여 `trigger(['field1', 'field2'])` 형태로 현재 단계의 필드만 검증했습니다.

- **밸리데이션 역할 분담**: 클라이언트에서는 즉각적인 UX를 위해 형식 검증(이메일 형식, 필수 입력, 글자 수 등)을 수행하고, 서버에서는 비즈니스 규칙 검증(중복 확인, 재고 확인, 권한 등)을 수행합니다. 서버 에러는 `setError`로 React Hook Form에 전달하여 해당 필드에 에러 메시지를 표시했습니다.

</details>

---

### Q5. 20단계 이상의 체이닝 폼 플로우를 상태 머신으로 관리한 방법은?
> **출제 의도**: 복잡한 UI 플로우의 상태 관리 패턴
>
> **경력 연관**: 마이리얼트립 Staynet

<details>
<summary><b>✅ 답변</b></summary>

**설계 방식**: 각 단계를 상태(state)로, 단계 전환을 이벤트(event)로 모델링했습니다.

```
상태: idle → roomType → dateRange → guestCount → amenities → ... → review → submit
이벤트: NEXT, PREV, SKIP, SUBMIT
전환 조건: 각 상태에서 유효성 검사 통과 여부
```

**구현 방식**:
1. `useReducer`로 현재 단계, 폼 데이터, 유효성 상태를 통합 관리
2. 각 단계의 전환 규칙을 설정 객체로 정의 (어떤 이벤트에 어떤 조건에서 어떤 상태로 전환)
3. 조건부 단계 건너뛰기: 특정 선택에 따라 중간 단계를 skip하는 로직을 전환 규칙에 포함

**30개+ 필드 성능 문제 해결**:
- ContextAPI + Ref 기반 uncontrolled 패턴: 폼 데이터를 `useRef`로 저장하고, 제출 시에만 ref에서 값을 수집
- 입력 중에는 React 상태가 변경되지 않으므로 리렌더링 발생 없음
- Profiler commit 횟수 비교: 기존(controlled) 대비 90% 감소

</details>

**꼬리 질문**:
- 상태 머신을 선택한 이유는? XState 사용 여부?
- 유효성 검사 타이밍은?
- 폼 데이터 임시 저장(Draft) 기능은?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **상태 머신 선택 이유**: 20단계+의 복잡한 플로우에서 `if/else`로 관리하면 조건이 기하급수적으로 증가합니다. 상태 머신은 "현재 상태에서 가능한 전환"만 정의하므로 불가능한 상태 전환을 원천 차단합니다. XState를 검토했지만 프로젝트 규모 대비 러닝 커브가 있어, `useReducer` 기반으로 가볍게 구현했습니다.

- **유효성 검사 타이밍**: `onBlur`를 기본으로, 에러가 한 번 발생한 필드는 `onChange`로 전환하여 즉각적인 피드백을 제공했습니다. "다음" 버튼 클릭 시에는 현재 단계의 모든 필드를 한 번에 검증(`onSubmit`)합니다.

- **Draft 기능**: `sessionStorage`에 현재 단계와 폼 데이터를 자동 저장했습니다. 페이지 이탈 시 `beforeunload` 이벤트로 경고를 표시하고, 재방문 시 저장된 데이터를 복원할지 묻는 UI를 제공했습니다. 민감한 데이터(카드 번호 등)는 저장에서 제외했습니다.

</details>

---

## 로컬 상태

### Q6. 로컬스토리지 기반 크로스 탭 동기화 훅은 어떻게 구현했나요?
> **출제 의도**: 브라우저 API 활용과 상태 동기화 패턴
>
> **경력 연관**: 마이리얼트립 통합숙소

<details>
<summary><b>✅ 답변</b></summary>

**문제 상황**: 결제 플로우에서 사용자가 탭 A에서 상품을 장바구니에 추가하고, 탭 B에서 결제를 진행할 때 장바구니 상태가 동기화되지 않아 결제 이탈 발생.

**구현**:

1. **StorageEvent**: `window.addEventListener('storage', callback)`으로 다른 탭에서 localStorage 변경 감지. 단, 같은 탭에서의 변경은 이벤트가 발생하지 않음.
2. **Custom Event**: 같은 탭 내 동기화를 위해 `window.dispatchEvent(new CustomEvent('local-storage', { detail: { key, value } }))`를 localStorage 업데이트 시마다 발생시킴.
3. **통합 훅**: 두 이벤트를 하나의 `useLocalStorage` 훅으로 통합하여, 어떤 탭/컴포넌트에서 값이 변경되든 구독하는 모든 곳에 반영.

```
탭 A: localStorage.setItem() → CustomEvent → 같은 탭 컴포넌트 업데이트
                               → StorageEvent → 다른 탭 컴포넌트 업데이트
```

**결과**: 결제 플로우 이탈률 감소, 멀티 탭 사용자의 장바구니 일관성 확보.

</details>

**꼬리 질문**:
- `StorageEvent`만으로 부족한 이유는?
- 보안 민감 데이터의 localStorage 관리 주의점은?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **StorageEvent 한계**: `StorageEvent`는 같은 origin의 다른 window/tab에서만 발생합니다. 같은 탭에서 `localStorage.setItem()`을 호출해도 해당 탭에서는 이벤트가 트리거되지 않습니다. 따라서 같은 탭 내에서 서로 다른 컴포넌트가 같은 키를 구독할 때는 CustomEvent를 사용해야 합니다.

- **보안 주의점**: localStorage는 JavaScript로 자유롭게 접근 가능하므로 토큰, 비밀번호 등 민감 정보를 저장하면 XSS 공격에 취약합니다. 장바구니 같은 비민감 데이터만 저장하고, 인증 토큰은 httpOnly 쿠키를 사용했습니다. 또한 저장 데이터를 JSON.parse할 때 항상 try-catch로 감싸 악의적으로 변조된 데이터에 의한 크래시를 방지했습니다.

</details>

---

### Q7. 버전 관리 가능한 LocalStorage 어댑터를 어떻게 설계했나요?
> **출제 의도**: 하위 호환성과 데이터 마이그레이션 설계
>
> **경력 연관**: 마이리얼트립 렌터카/숙소 고도화

<details>
<summary><b>✅ 답변</b></summary>

**문제**: LocalStorage에 저장된 데이터 스키마가 변경되면(필드 추가/삭제/타입 변경), 기존 사용자의 브라우저에 남아있는 이전 형식 데이터로 인해 `undefined is not a function` 등의 런타임 에러가 발생.

**어댑터 설계**:

1. **버전 번호**: 저장 시 `{ _version: 2, data: { ... } }` 형태로 버전을 함께 저장
2. **마이그레이션 함수 체인**: 각 버전 업그레이드에 대한 마이그레이션 함수를 정의
   - `v1 → v2`: 필드명 변경 (`userName` → `name`)
   - `v2 → v3`: 새 필드 추가 + 기본값 설정
3. **읽기 시 자동 마이그레이션**: 저장된 버전이 현재 코드 버전보다 낮으면 마이그레이션 체인을 순차 실행하여 최신 형식으로 변환 후 재저장
4. **버전 불일치 폴백**: 마이그레이션 실패 시 기본값으로 초기화 (데이터 손실보다 에러 방지 우선)

이 패턴으로 배포 후 기존 사용자의 로컬 데이터 충돌 에러가 완전히 해소되었습니다.

</details>

**꼬리 질문**:
- 마이그레이션 체인의 구체적 구현은?
- 버전 충돌 감지와 폴백 전략은?
- IndexedDB 대신 LocalStorage를 선택한 이유는?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **마이그레이션 체인**: `const migrations = { 1: (data) => ({ ...data, name: data.userName }), 2: (data) => ({ ...data, preferences: defaultPrefs }) }` 형태로 정의합니다. 읽기 시 `for (let v = storedVersion; v < currentVersion; v++) { data = migrations[v](data) }`로 순차 적용합니다.

- **폴백 전략**: try-catch로 마이그레이션 전체를 감싸서, JSON 파싱 실패나 마이그레이션 함수 에러 시 해당 키를 삭제하고 기본값을 반환합니다. Sentry에 이벤트를 전송하여 어떤 버전에서 문제가 발생했는지 추적합니다.

- **LocalStorage 선택 이유**: 저장 데이터가 단순한 JSON 객체(수 KB)이고, 동기 접근이 필요했습니다(렌더링 초기에 즉시 읽어야 함). IndexedDB는 비동기 API라 초기 렌더링 전에 데이터를 읽으려면 별도의 로딩 상태가 필요합니다. 데이터 크기와 복잡도가 커지면 IndexedDB로 전환을 고려했을 것입니다.

</details>
