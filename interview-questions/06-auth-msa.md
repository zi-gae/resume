# 인증 & MSA 질문

### Q1. 이중 액세스 토큰(A→B) 구조를 설계한 배경과 구현 방식을 설명해주세요.
> **출제 의도**: MSA 환경의 인증 아키텍처 설계 역량
>
> **경력 연관**: 이그나이트 딜러포탈

<details>
<summary><b>✅ 답변</b></summary>

**배경**: MSA 환경에서 도메인별로 인증 방식과 요청 헤더가 달랐습니다. 딜러포탈(A 도메인)의 인증과 차량 관리 서비스(B 도메인)의 인증이 별도로 운영되어, 클라이언트가 두 도메인 모두에 요청하려면 각각의 토큰이 필요했습니다.

**구조**:
```
로그인 → A 액세스 토큰 발급
       → A 토큰으로 B 도메인에 요청 → B 액세스 토큰 발급 (Child 인증)
       → A 토큰: 딜러포탈 API에 사용
       → B 토큰: 차량 관리 API에 사용
```

**핵심 과제 - A 토큰 만료 시 연쇄 처리**:
1. A 토큰 만료 감지 (401 응답)
2. A 토큰 갱신 (Refresh Token 사용)
3. 새 A 토큰으로 B 토큰 재발급
4. 대기 중이던 A/B 도메인 요청 모두 재시도

**구현 핵심**:
- **요청 큐잉**: 토큰 갱신 중 들어오는 요청을 Promise 큐에 저장. 갱신 완료 후 큐의 모든 요청을 새 토큰으로 재시도
- **Mutex 패턴**: 동시에 여러 요청이 401을 받아도 토큰 갱신은 1회만 실행. `isRefreshing` 플래그로 중복 방지
- **순서 보장**: A 갱신 → B 재발급 → 대기 요청 재시도 순서를 Promise 체인으로 보장

**성과**: Datadog 측정 기준 인증 성공률 99.7%, 동시 요청 최대 50건 처리.

</details>

**꼬리 질문**:
- Refresh Token 자체가 만료된 경우의 처리는?
- 토큰 갱신 중 사용자 경험(UX)은?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **Refresh Token 만료**: Refresh Token이 만료되면 복구 불가능하므로, 로그인 페이지로 리다이렉트합니다. 이때 현재 URL을 `returnUrl` 쿼리 파라미터에 저장하여 재로그인 후 원래 페이지로 복귀합니다. Refresh Token 만료가 가까우면(예: 7일 이내) 자동으로 Refresh Token도 갱신(Rotation)하여 장기 사용자의 세션 만료를 방지했습니다.

- **UX**: 토큰 갱신은 백그라운드에서 투명하게 처리되므로, 사용자는 인식하지 못합니다. 요청 큐잉으로 갱신 중에도 UI가 블로킹되지 않고, 갱신 완료 후 자연스럽게 데이터가 표시됩니다. 갱신 실패 시에만 에러 바운더리에서 "세션이 만료되었습니다" 토스트를 표시하고 로그인 페이지로 안내합니다.

</details>

---

### Q2. 도메인별로 다른 요청 헤더를 관리하는 통합 HTTP 클라이언트를 어떻게 구현했나요?
> **출제 의도**: HTTP 클라이언트 추상화 설계
>
> **경력 연관**: 이그나이트 딜러포탈

<details>
<summary><b>✅ 답변</b></summary>

**문제**: A 도메인은 `Authorization: Bearer {A_TOKEN}` + `X-Brand: hyundai`, B 도메인은 `Authorization: Bearer {B_TOKEN}` + `X-Service-Id: dealer-portal` 등 도메인마다 헤더 구성이 다름.

**설계**:

```
httpClient.create({
  baseURL: 'https://api-a.example.com',
  headers: (config) => ({
    Authorization: `Bearer ${getTokenA()}`,
    'X-Brand': getCurrentBrand(),
  }),
  interceptors: {
    onError: handleTokenRefresh,
  },
})
```

**핵심 구현**:
1. **도메인별 인스턴스**: 각 도메인별로 독립된 HTTP 클라이언트 인스턴스 생성. 헤더 설정, 에러 핸들링, 타임아웃 등을 독립적으로 관리
2. **동적 헤더**: 헤더를 함수로 정의하여 요청 시점에 최신 토큰/브랜드 정보를 반영
3. **공통 인터셉터**: 에러 핸들링(401 → 토큰 갱신), 요청/응답 로깅(Datadog), 타임아웃 등 공통 로직은 베이스 인터셉터로 추상화
4. **타입 안전성**: 각 도메인의 API 응답 타입을 제네릭으로 정의하여 호출부에서 타입 추론

**도메인 인스턴스 선택**: React Query의 `queryFn`에서 도메인에 따라 적절한 클라이언트 인스턴스를 자동 선택하도록 wrapper를 구성했습니다.

</details>

---

### Q3. 역할 기반 접근 제어(RBAC) UI를 3개 브랜드 × 48개국에 적용한 경험을 설명해주세요.
> **출제 의도**: 복잡한 권한 시스템의 프론트엔드 구현
>
> **경력 연관**: 이그나이트 Admin CMS

<details>
<summary><b>✅ 답변</b></summary>

**복잡도**: 3개 브랜드(현대/기아/제네시스) × 48개국 × N개 역할(슈퍼어드민, 국가관리자, 딜러매니저, 딜러 등) = 수백 개의 권한 조합.

**프론트엔드 구현**:

1. **권한 데이터 모델**:
```typescript
interface Permission {
  brand: 'hyundai' | 'kia' | 'genesis';
  country: string;  // ISO 국가 코드
  role: Role;
  features: string[];  // 접근 가능한 기능 목록
}
```

2. **권한 체크 컴포넌트**:
```tsx
<Authorize feature="vehicle.register" brand="hyundai">
  <VehicleRegisterButton />
</Authorize>
```
- 권한이 없으면 컴포넌트를 렌더링하지 않음
- fallback prop으로 권한 없을 때 대체 UI 표시 가능

3. **라우트 가드**: 페이지 레벨에서 권한 체크. 접근 불가 시 403 페이지로 리다이렉트

4. **권한 매트릭스 관리 UI**: 슈퍼어드민이 브랜드/국가/역할별 권한을 시각적으로 관리할 수 있는 매트릭스 테이블 구현. 체크박스 토글로 직관적 조작.

**주의점**: 프론트엔드의 권한 체크는 UX를 위한 것이고, 실제 보안은 반드시 서버에서 검증합니다. API에서도 동일한 권한 체크를 수행하여, 프론트엔드를 우회한 요청도 차단됩니다.

</details>

---

### Q4. 국가별 데이터 보호 규정(GDPR/CCPA)을 프론트엔드 미들웨어로 어떻게 대응했나요?
> **출제 의도**: 글로벌 서비스의 규정 준수 구현 역량
>
> **경력 연관**: 이그나이트 딜러포탈

<details>
<summary><b>✅ 답변</b></summary>

**구현 방식**: Next.js 미들웨어에서 사용자의 국가를 감지하고, 해당 국가의 데이터 보호 정책에 맞는 처리를 적용했습니다.

**국가 감지**: 
1. CloudFront의 `CloudFront-Viewer-Country` 헤더로 1차 감지
2. 사용자 프로필의 국가 설정으로 보정

**정책별 처리**:

| 규정 | 적용 국가 | 프론트엔드 처리 |
|------|----------|---------------|
| GDPR | EU 27개국 | 동의 배너 필수 표시, 동의 전 트래킹 스크립트 차단 |
| CCPA | 캘리포니아 | "Do Not Sell" 옵트아웃 링크 표시 |
| 기타 | 그 외 | 기본 개인정보 정책 적용 |

**ConsentGuard 구현**:
- 사용자의 동의 상태를 확인하고, 동의하지 않은 경우 마케팅 트래킹(GA, Datadog RUM 등)을 차단
- 동의 상태는 쿠키에 저장하여 재방문 시 배너 재표시 방지
- Guard chain 구조에서 ConsentGuard가 완료된 후에야 API 호출이 시작되는 문제를 HomePrefetcher로 개선 (별도 성능 최적화 항목에서 설명)

</details>

---

### Q5. Suspense + ErrorBoundary 기반 에러 핸들링 구조를 어떻게 도입했나요?
> **출제 의도**: 선언적 에러 핸들링 아키텍처
>
> **경력 연관**: 마이리얼트립 렌터카/숙소 고도화

<details>
<summary><b>✅ 답변</b></summary>

**기존 문제**: 각 컴포넌트에서 `isLoading`, `isError`를 개별 처리하여 코드 중복이 심하고, 에러 상태 누락으로 빈 화면이 표시되는 케이스 발생.

**도입 구조**:
```tsx
<ErrorBoundary fallback={<ErrorFallback />}>
  <Suspense fallback={<Skeleton />}>
    <ProductList />
  </Suspense>
</ErrorBoundary>
```

**계층별 에러 핸들링**:
1. **페이지 레벨**: 전체 페이지 에러 → "문제가 발생했습니다" + 재시도 버튼
2. **섹션 레벨**: 특정 영역 에러 → 해당 영역만 에러 표시, 나머지는 정상 동작
3. **컴포넌트 레벨**: 개별 위젯 에러 → 해당 위젯만 "로드 실패" 표시

**React Query 연동**:
- `useQuery`의 `suspense: true` 옵션으로 로딩 상태를 Suspense에 위임
- `throwOnError: true`로 에러를 ErrorBoundary에 위임
- 컴포넌트에서는 데이터만 사용하면 되므로 코드가 간결해짐

**재시도 로직**: ErrorBoundary의 `resetErrorBoundary`와 React Query의 `queryClient.resetQueries`를 연결하여 에러 상태 초기화 + 데이터 재요청을 한 번의 버튼 클릭으로 처리.

</details>

**꼬리 질문**:
- ErrorBoundary가 잡지 못하는 에러 유형은?
- 에러 리포팅(Sentry) 연동은?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **잡지 못하는 에러**: ErrorBoundary는 렌더링 중 발생하는 에러만 잡습니다. 이벤트 핸들러, 비동기 코드(`setTimeout`, `Promise`), 서버 사이드 렌더링의 에러는 잡지 못합니다. 이벤트 핸들러의 에러는 try-catch로 직접 처리하고, 비동기 에러는 React Query의 `onError` 콜백이나 전역 `window.onerror`로 처리했습니다.

- **Sentry 연동**: ErrorBoundary의 `componentDidCatch`에서 `Sentry.captureException(error, { extra: { componentStack } })`으로 에러를 전송했습니다. 컴포넌트 스택 정보를 포함하여 어떤 컴포넌트에서 에러가 발생했는지 추적 가능합니다. 환경별(dev/staging/production) 필터링으로 개발 환경 에러가 Sentry에 전송되지 않도록 했습니다.

</details>
