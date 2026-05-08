# Hyundai FO 인증 아키텍처 문서

> 작성일: 2026-03-24
> 관련 앱: `apps/hyundai-fo`
> 핵심 파일: `InternalApi.ts`, `hmgPartnersAuth.tsx`, `auths.tsx`, `GuardedLayout.tsx`

---

## 1. 전체 인증 아키텍처

```
┌─────────────┐      ┌──────────────────────┐      ┌──────────────────────┐
│   사용자      │ ──→  │   HMG Admin Partner   │ ──→  │    Internal Auth API  │
│  (Browser)  │      │   (External SSO)      │      │   /api/v1/auth/fo    │
└─────────────┘      └──────────────────────┘      └──────────────────────┘
       │                        │                              │
       │             HMG Partner Token               Internal JWT Tokens
       │           (X-ADMIN-FO-Authorization)       (accessToken + refreshToken)
       │                        │                              │
       └────────────────────────┴──────────────────────────────┘
                                │
                   모든 API 요청에 두 토큰 동시 포함
```

### 이중 토큰 구조

| 토큰 | 헤더 | 발급처 | 관리 위치 |
|------|------|--------|-----------|
| HMG Partner Token | `X-ADMIN-FO-Authorization` | HMG Admin Partner SSO | `hmgPartnersAuth` 유틸 |
| Internal JWT (Bearer) | `Authorization: Bearer {token}` | `/api/v1/auth/fo/token` | `localStorage` (AUTH_STORAGE_KEYS) |

모든 API 호출은 `InternalApi.request()`를 통해 이루어지며, 매 요청마다 두 토큰이 함께 전송된다.

```ts
// InternalApi.ts
public async request<T>(config: AxiosRequestConfig) {
  const token = await hmgAdminPartnerAuth.signIn()  // HMG Partner 토큰 갱신

  return super.request<T>({
    ...config,
    headers: {
      ...config.headers,
      'X-ADMIN-FO-Authorization': token,           // HMG 토큰 삽입
    },
  })
}
```

---

## 2. 인증 플로우

### 전체 순서도

```
[파트너 로그인]
     │
     ▼
HMG Admin Partner 인증
→ X-ADMIN-FO-Authorization 쿠키/토큰 발급
     │
     ▼
[앱 초기화] HmgPartnersAuthProvider
→ hmgAdminPartnerAuth.create() + signIn()
→ isHmgAuthenticated = true
     │
     ▼
[딜러 선택 확인] DealerSelectionGuard
→ useHasMultipleActiveDealers()
  ├── 복수 딜러 + 미선택 → /authentication/multi-dealer 리다이렉트
  └── 단일 딜러 → 자동 선택 (POST /fo/dealers/selected)
     │
     ▼
[브랜드 확인] useBrandRedirectGuard
→ useSelectedDealer()
  ├── 딜러 미선택 → 멀티딜러 페이지
  └── 다른 브랜드 → 해당 브랜드 도메인으로 window.location.replace()
     │
     ▼
[Internal 토큰 발급] useAuthorization
→ POST /api/v1/auth/fo/token (dealerNo 포함)
→ accessToken + refreshToken 발급
→ setAuthorization() → 6개 API 인스턴스에 Bearer 토큰 세팅
→ localStorage 저장
     │
     ▼
[사용자 정보 조회]
→ useDealerEmployeeMe()    → GET /api/v1/basic/fo/dealer-employees/me
→ useCheckCustomPagePermission()  → 권한 체크
     │
     ▼
[동의 확인] ConsentGuard
→ useConsent() → HMG Notice 약관 동의 여부
  ├── 미동의 → /authentication/terms 리다이렉트
  └── 동의 완료 → children 렌더링
     │
     ▼
[홈 화면 접근]
```

---

## 3. Guard 체인 구조

```tsx
// GuardedLayout.tsx
<MaintenanceGuard.Redirect>         // 1. 점검 중이면 점검 페이지로
  <DealerSelectionGuard>            // 2. 딜러 선택 여부 확인
    <AuthGuard redirectPath="/">    // 3. Internal 토큰 유효성 확인
      <HomePrefetcher />            // 4. Home API 병렬 prefetch (신규)
      <ConsentGuardComponent>       // 5. 약관 동의 확인
        {isChangingDealer
          ? <GlobalLoadingActivator />  // 딜러 변경 중 로딩
          : children}
      </ConsentGuardComponent>
    </AuthGuard>
  </DealerSelectionGuard>
</MaintenanceGuard.Redirect>
```

### Guard별 역할

| Guard | 통과 조건 | 미통과 시 처리 |
|-------|-----------|--------------|
| `MaintenanceGuard` | 서비스 점검 아님 | 점검 페이지 리다이렉트 |
| `DealerSelectionGuard` | 딜러 선택 완료 | 단일 딜러면 자동선택, 복수면 선택 페이지 |
| `AuthGuard` | `isAuthenticated === true` | 홈(로그인 페이지)으로 리다이렉트 |
| `ConsentGuard` | 약관 동의 완료 | 약관 동의 페이지로 리다이렉트 |

---

## 4. 토큰 갱신 (Refresh) 플로우

```
API 응답 401 수신
     │
     ▼
HttpClient 인터셉터 → refreshTokenHandler 실행
     │
     ▼
POST /api/v1/auth/fo/refresh
(Authorization: Bearer {refreshToken})
     │
     ├── 성공 → setAuthorization(newAccessToken) → 원래 요청 재시도
     │          localStorage 업데이트
     │          hmgAdminPartnerAuth.instance?.clearTimer()
     │
     └── 실패 → Logout → 로그인 페이지
```

`setRefreshTokenHandler()`에서 6개 API 인스턴스에 동일 핸들러 등록:

```ts
// InternalApi.ts
const apis = [authApi, contentsApi, basicApi, authzApi, searchApi, tenantApi]
apis.forEach((api) => {
  api.setRefreshTokenHandler(async (token) => {
    const { data } = await authRequest.createJwtTokenUsingRefreshToken(...)
    setAuthorization(data.accessToken)
    return { accessToken: data.accessToken }
  })
})
```

---

## 5. 딜러 변경 플로우

```
사용자가 딜러 변경 선택
     │
     ▼
useSaveSelectedDealer.mutateAsync()
→ PUT /api/v1/auth/fo/dealers/selected
→ isPending = true → GlobalLoadingActivator 표시 (isChangingDealer)
     │
     ├── onMutate: 낙관적 업데이트 (캐시 선반영)
     │
     ├── onSuccess: queryClient.invalidateQueries() → 전체 쿼리 무효화 → 리페치
     │
     └── onError: 캐시 롤백 (previousData 복원)
```

---

## 6. 브랜드 리다이렉트 (멀티 브랜드 지원)

딜러 소속 브랜드(현대/기아 등)가 현재 접속 도메인과 다를 경우 자동 리다이렉트:

```ts
// useBrandRedirectGuard.ts
if (isDifferentBrand(currentBrand, selectedDealerData.tenant)) {
  const targetUrl = getBrandDomainUrl(selectedDealerData.tenant, stage)
  window.location.replace(`${targetUrl}${pathname}`)
}
```

---

## 7. 주요 기여 사항

### 7-1. Guard 체인 중 API 병렬 Prefetch (GIDPDVO-1183)

**문제**: Guard 체인이 순차적으로 실행되므로 ConsentGuard 통과 후에야 Home 데이터 fetch가 시작되어 초기 로딩이 지연됨.

**해결**: `HomePrefetcher` 컴포넌트를 `ConsentGuard`의 형제 노드로 배치. `AuthGuard` 통과 직후 ConsentGuard 검사와 **병렬**로 3개 API를 동시에 prefetch.

```tsx
// GuardedLayout.tsx
<AuthGuard redirectPath={routePath.home()}>
  <HomePrefetcher />        {/* ConsentGuard와 동시에 렌더링 */}
  <ConsentGuardComponent>
    {children}
  </ConsentGuardComponent>
</AuthGuard>
```

```ts
// HomePrefetcher 내부
// await 없이 연속 호출 → 3개 API 동시 시작
queryClient.prefetchQuery({ queryKey: contentKeys.notice(), ... })
queryClient.prefetchQuery({ queryKey: contentKeys.noticeBoard(), ... })
queryClient.prefetchQuery({ queryKey: contentKeys.highlight(), ... })
```

**효과**: ConsentGuard 검사 시간(네트워크 왕복) 동안 Home 데이터가 캐시에 적재되어, 홈 화면 진입 시 즉시 렌더링 가능.

---

### 7-2. 딜러 변경 중 로딩 UX 처리 (GIDPDVO-1183)

**문제**: 딜러 변경 API 호출 중 화면이 깜빡이거나 이전 딜러의 데이터가 잠시 노출됨.

**해결**: `saveSelectedDealerMutation.isPending`을 `isChangingDealer`로 컨텍스트에 노출하고, 변경 중에는 `GlobalLoadingActivator`로 대체.

```tsx
// DealerSelectionContext
isChangingDealer: saveSelectedDealerMutation.isPending,

// GuardedLayout.tsx
{isChangingDealer ? <GlobalLoadingActivator /> : children}
```

**효과**: 딜러 변경 중 이전 딜러 데이터 노출 없이 로딩 화면 표시. API 완료 후 `invalidateQueries()`로 전체 데이터 일관성 보장.

---

## 8. API 엔드포인트 목록

| 역할 | Method | Path |
|------|--------|------|
| Internal 토큰 발급 | POST | `/api/v1/auth/fo/token` |
| 토큰 갱신 | POST | `/api/v1/auth/fo/refresh` |
| 비밀번호 변경 | PUT | `/api/v1/auth/fo/password` |
| 딜러 목록 조회 | GET | `/api/v1/auth/fo/dealers` |
| 복수 딜러 활성화 여부 | GET | `/api/v1/auth/fo/dealers/status` |
| 선택된 딜러 조회 | GET | `/api/v1/auth/fo/dealers/selected` |
| 딜러 선택 저장 | PUT | `/api/v1/auth/fo/dealers/selected` |
| 딜러 선택 초기화 | DELETE | `/api/v1/auth/fo/dealers/selected` |
| 약관 동의 | POST | `/api/v1/auth/fo/terms-agreement` |
| 파트너 어드민 연결 | POST | `/api/v1/auth/fo/partner/connect` |
