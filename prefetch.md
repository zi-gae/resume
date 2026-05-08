# Guard 체인 중 API 병렬 Prefetch 전략

> 관련 커밋: `5b65026` — GIDPDVO-1183: 홈 페이지 데이터 prefetch 및 딜러 변경 시 로딩 처리 추가
> 관련 파일: `apps/hyundai-fo/src/components/layouts/GuardedLayout.tsx`

---

## 배경

Guard 체인(`MaintenanceGuard → DealerSelectionGuard → AuthGuard → ConsentGuard`)은 각 Guard가 통과된 후 다음 Guard를 렌더링하는 **순차적 구조**다.

문제는 Home 페이지 진입 시 ConsentGuard 검사가 끝난 뒤에야 Home 데이터를 fetch하기 시작한다는 점이다. Guard 검사에 걸리는 시간만큼 데이터 로딩이 지연된다.

---

## 해결 전략: Sibling 컴포넌트로 병렬 시작

`HomePrefetcher`를 `ConsentGuard`의 **형제 노드(sibling)** 로 배치해, React가 두 컴포넌트를 동시에 렌더링하도록 만든다.

### 구조

```tsx
<AuthGuard redirectPath={routePath.home()}>
  <HomePrefetcher />          {/* AuthGuard 통과 직후 즉시 렌더링 */}
  <ConsentGuardComponent>     {/* HomePrefetcher와 동시에 실행 */}
    {children}
  </ConsentGuardComponent>
</AuthGuard>
```

AuthGuard를 통과하는 순간 두 컴포넌트가 함께 렌더링된다.
`HomePrefetcher`는 ConsentGuard가 검사를 마치기 전에 이미 API 호출을 시작한다.

---

## HomePrefetcher 구현

```tsx
function HomePrefetcher() {
  const location = useLocation()

  useEffect(() => {
    if (location.pathname !== '/') return  // 홈 페이지일 때만 실행

    // await 없이 순차 호출 → 세 API가 동시에 날아감
    queryClient.prefetchQuery({
      queryKey: contentKeys.notice(),
      queryFn: () => contentsRequest.getNotice(),
    })
    queryClient.prefetchQuery({
      queryKey: contentKeys.noticeBoard(),
      queryFn: () => contentsRequest.getNoticeBoard(),
    })
    queryClient.prefetchQuery({
      queryKey: contentKeys.highlight(),
      queryFn: () => contentsRequest.getHomeHighlight(),
    })
  }, [location.pathname])

  return null  // UI 없음, 순수 사이드이펙트 컴포넌트
}
```

### 포인트

- `prefetchQuery()`를 `await` 없이 연속 호출 → 각 Promise가 독립적으로 실행되어 세 API가 **동시**에 호출된다.
- `return null`로 UI를 렌더링하지 않는다.
- Bearer 토큰은 AuthGuard 통과 시점에 이미 설정되어 있으므로 인증 문제 없다.

---

## 타이밍 다이어그램

```
AuthGuard 통과
    │
    ├─── HomePrefetcher 렌더링 ──→ API 호출 시작 (notice, noticeBoard, highlight)
    │                                     │
    └─── ConsentGuard 검사 중 ────────────┤
              │                           │
         ConsentGuard 통과                │ (캐시에 데이터 적재 중)
              │                           │
         Home 페이지 렌더링 ◀─────────────┘
         (캐시 히트 or 로딩 단축)
```

---

## 딜러 변경 시 로딩 처리

딜러 변경 API(`saveSelectedDealerMutation`) 호출 중에는 children 대신 글로벌 로딩 스피너를 보여준다.

```tsx
// GuardedLayout.tsx
const { isChangingDealer } = useDealerSelection()

<ConsentGuardComponent>
  {isChangingDealer ? <GlobalLoadingActivator /> : children}
</ConsentGuardComponent>
```

```tsx
// dealerSelection.tsx
isChangingDealer: saveSelectedDealerMutation.isPending,
```

```tsx
// auth/query.ts — useSaveSelectedDealer
onSuccess() {
  // 딜러 변경 완료 후 전체 쿼리 무효화 → 모든 데이터 리페치 트리거
  queryClient.invalidateQueries()
},
```

### 흐름

1. 딜러 변경 API 호출 시작 → `isPending = true` → 로딩 화면 표시
2. API 성공 → `invalidateQueries()` → 전체 쿼리 무효화 → 리페치
3. `isPending = false` → 기존 children 복원

---

## 요약

| 기법 | 설명 |
|------|------|
| Sibling 렌더링 | `ConsentGuard`와 형제로 배치해 Guard 검사와 API 호출을 병렬화 |
| `prefetchQuery` 연속 호출 | `await` 없이 호출해 세 API를 동시에 시작 |
| `isPending` 기반 로딩 | 딜러 변경 중 UI를 로딩 스피너로 교체 |
| `invalidateQueries` | 딜러 변경 성공 후 전체 캐시를 무효화해 일관성 유지 |
