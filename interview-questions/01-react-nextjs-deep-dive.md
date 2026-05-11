# React / Next.js 심화 질문

## React 핵심

### Q1. React 18에서 도입된 주요 변경점과 실제 프로젝트에서 활용한 경험을 설명해주세요.
> **출제 의도**: React 18의 Concurrent Features, Automatic Batching, Suspense 개선 등에 대한 이해도 확인
>
> **경력 연관**: 이그나이트(React 18), 마이리얼트립(React 17→18 마이그레이션)

<details>
<summary><b>✅ 답변</b></summary>

React 18의 핵심 변경점은 크게 세 가지입니다.

**1) Automatic Batching**: 이전에는 이벤트 핸들러 내에서만 state 업데이트가 배칭되었지만, React 18부터는 `setTimeout`, `Promise`, 네이티브 이벤트 핸들러 등에서도 자동으로 배칭됩니다. 이그나이트 딜러포탈에서 토큰 갱신 후 여러 상태를 업데이트하는 로직에서 불필요한 리렌더링이 자연스럽게 줄어드는 효과를 봤습니다.

**2) Concurrent Features**: `useTransition`을 활용해 딜러포탈의 48개국 필터링 UI에서 무거운 리스트 렌더링을 낮은 우선순위로 처리하고, 사용자 입력은 즉시 반영되도록 구현했습니다. `startTransition`으로 감싸면 해당 업데이트가 중단 가능(interruptible)해져 UI 응답성이 크게 개선됩니다.

**3) Suspense 개선**: 서버 사이드에서도 Suspense를 지원하게 되면서, 마이리얼트립에서 React 17→18 마이그레이션 시 Suspense + ErrorBoundary 조합으로 선언적 로딩/에러 처리 구조를 도입했습니다. 데이터 페칭 라이브러리(React Query)와 결합하여 컴포넌트 레벨에서 로딩 상태를 관리하는 구조를 확립했습니다.

</details>

**꼬리 질문**:
- Automatic Batching이 이전 버전과 어떻게 다른가요?
- `useTransition`과 `useDeferredValue`의 차이는?
- Concurrent Mode에서 tearing 문제를 어떻게 해결하나요?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **Automatic Batching 차이**: React 17에서는 React 이벤트 핸들러(onClick 등)에서만 배칭이 동작했고, `setTimeout`이나 `fetch().then()` 안에서 setState를 여러 번 호출하면 각각 리렌더링이 발생했습니다. React 18의 `createRoot`를 사용하면 모든 컨텍스트에서 자동 배칭됩니다. 배칭을 원하지 않는 경우 `flushSync`로 강제 동기 업데이트가 가능합니다.

- **useTransition vs useDeferredValue**: `useTransition`은 state 업데이트 자체를 낮은 우선순위로 만들고, `useDeferredValue`는 이미 업데이트된 값의 반영을 지연시킵니다. 전자는 "업데이트를 트리거하는 측"에서, 후자는 "값을 소비하는 측"에서 사용합니다. 검색어 입력 → 결과 리스트에서, 입력 컴포넌트가 검색을 직접 트리거하면 `useTransition`, 부모로부터 받은 검색어를 소비하는 리스트에서는 `useDeferredValue`가 적합합니다.

- **Tearing 문제**: 외부 스토어(Redux, Zustand 등)를 사용할 때, Concurrent Rendering 중 스토어 값이 변경되면 같은 렌더 트리 내에서 서로 다른 값을 읽는 tearing이 발생할 수 있습니다. React 18에서 제공하는 `useSyncExternalStore` 훅으로 외부 스토어를 구독하면, 동기적으로 일관된 스냅샷을 보장하여 해결됩니다.

</details>

---

### Q2. Suspense + ErrorBoundary 기반 에러 핸들링 구조를 도입했다고 하셨는데, 구체적으로 어떻게 설계하셨나요?
> **출제 의도**: 선언적 에러 핸들링 패턴에 대한 설계 역량
>
> **경력 연관**: 마이리얼트립 렌터카/숙소 고도화

<details>
<summary><b>✅ 답변</b></summary>

**계층적 에러 바운더리 구조**를 설계했습니다.

```
App ErrorBoundary (전역 - 500 에러 페이지)
  └─ Layout ErrorBoundary (레이아웃 단위 - 부분 에러 UI)
       └─ Feature ErrorBoundary (기능 단위 - 재시도 버튼)
            └─ Suspense (로딩 UI)
                 └─ Component
```

**핵심 설계 원칙**:
1. **Granularity**: 에러 바운더리를 페이지 단위가 아닌 기능 단위로 분리하여, 검색 필터에서 에러가 나도 상품 리스트는 정상 노출
2. **재시도 메커니즘**: ErrorBoundary에 `resetErrorBoundary` 콜백을 제공하여 사용자가 "다시 시도" 클릭 시 React Query의 `retry`와 연동
3. **에러 분류**: HTTP 상태 코드에 따라 다른 UI 제공 (401→로그인 유도, 404→빈 상태, 500→에러+재시도)
4. **React Query 연동**: `useQuery`의 `suspense: true` 옵션으로 Suspense와 자연스럽게 통합, `useErrorBoundary` 옵션으로 ErrorBoundary에 에러 전파

기존에는 각 컴포넌트에서 `isLoading`, `isError`를 개별 체크하는 코드가 반복되었는데, 이 구조 도입 후 컴포넌트는 성공 케이스만 다루면 되어 코드 가독성이 크게 개선되었습니다.

</details>

**꼬리 질문**:
- ErrorBoundary로 잡을 수 없는 에러 유형은?
- Suspense fallback이 너무 자주 보이는 문제를 어떻게 해결했나요?
- 에러 발생 시 사용자에게 어떤 수준의 정보를 노출해야 할까요?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **ErrorBoundary 한계**: 이벤트 핸들러, `setTimeout`/`Promise` 내부, SSR에서 발생하는 에러는 잡지 못합니다. 이벤트 핸들러 에러는 try-catch로 처리하고, 비동기 에러는 React Query가 렌더링 사이클로 에러를 전파시켜 ErrorBoundary에서 포착하도록 했습니다. `window.addEventListener('unhandledrejection', ...)`으로 나머지 에러를 Sentry로 보고했습니다.

- **Suspense fallback 빈번 노출 문제**: `keepPreviousData: true`(v5에서는 `placeholderData`)를 활용해 새 데이터를 가져오는 동안 이전 데이터를 유지했습니다. 또한 `useTransition`과 결합하여 loading 상태가 짧은 경우(300ms 이하) fallback을 보여주지 않도록 했습니다.

- **에러 정보 노출 수준**: 프로덕션에서는 사용자 친화적 메시지만 표시하고, Sentry event ID를 함께 노출하여 CS 문의 시 추적 가능하게 했습니다. 기술적 상세(stack trace, HTTP status)는 Sentry/Datadog으로만 전송했습니다.

</details>

---

### Q3. `useEffect` 남용 문제를 해결했다고 하셨는데, useEffect가 적절한 경우와 부적절한 경우를 구분해주세요.
> **출제 의도**: React의 데이터 흐름과 사이드 이펙트 관리에 대한 깊은 이해
>
> **경력 연관**: 마이리얼트립 렌터카/숙소 - useEffect 컴포넌트당 3.4개 → 0.7개

<details>
<summary><b>✅ 답변</b></summary>

**useEffect가 적절한 경우**:
- 외부 시스템과의 동기화: WebSocket, DOM API 직접 조작, 브라우저 이벤트 리스너 등록
- 컴포넌트 마운트 시 일회성 초기화: 서드파티 라이브러리, Analytics
- 클린업이 필요한 구독: 타이머, IntersectionObserver, ResizeObserver

**useEffect가 부적절한 경우 (실제 리팩토링한 패턴들)**:
1. **파생 상태 계산**: `useEffect`로 A 변경 시 B를 업데이트 → `useMemo` 또는 렌더링 중 직접 계산으로 대체
2. **이벤트에 대한 반응**: 버튼 클릭 → state 변경 → useEffect로 API 호출 → 이벤트 핸들러에서 직접 API 호출로 변경
3. **props 변경에 따른 state 초기화**: `useEffect`로 리셋 → `key` prop 활용한 컴포넌트 재마운트로 해결
4. **데이터 페칭 체이닝**: useEffect A → B → C → React Query의 `enabled`/dependent queries로 대체

마이리얼트립 렌터카에서 컴포넌트당 평균 3~4개의 useEffect를 hooks 기반 단방향 데이터 플로우로 재구성하여 0.7개로 줄였고, useEffect 관련 버그가 약 80% 감소했습니다(Jira 레이블 추적 기준).

</details>

**꼬리 질문**:
- `useEffect` 대신 어떤 패턴을 사용했나요?
- 이벤트 핸들러에서 처리할 수 있는 로직을 useEffect로 작성하면 어떤 문제가 생기나요?
- React 공식 문서의 "You Might Not Need an Effect" 가이드를 어떻게 팀에 전파했나요?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **대체 패턴**: (1) 이벤트 핸들러에서 직접 처리 (2) `useMemo`/렌더링 중 계산으로 파생 상태 처리 (3) React Query의 `onSuccess`/`onSettled` 콜백 (4) `key` prop으로 컴포넌트 리셋 (5) 커스텀 훅으로 캡슐화

- **useEffect 남용의 문제**: 불필요한 렌더 사이클이 추가됩니다. 이벤트 → state 변경 → 렌더 → useEffect → state 변경 → 렌더, 최소 2번 렌더가 발생합니다. 또한 의존성 배열 관리 실수로 무한 루프가 발생하기 쉽고, 데이터 흐름이 비직관적이 되어 디버깅이 어려워집니다.

- **팀 전파**: PR 리뷰에서 before/after 코드를 보여주며 React 공식 문서 링크와 함께 코멘트했습니다. 팀 위키에 "useEffect 대체 패턴 가이드"를 정리하고, 실제 프로젝트 리팩토링 예시를 공유했습니다.

</details>

---

## Next.js

### Q4. Next.js 13의 App Router와 Pages Router의 차이, 그리고 실제 프로젝트에서의 선택 기준은?
> **출제 의도**: Next.js 아키텍처 이해와 기술 선택 근거
>
> **경력 연관**: 이그나이트 Admin CMS(Next.js 13), 통합숙소(Next.js 13)

<details>
<summary><b>✅ 답변</b></summary>

| 항목 | Pages Router | App Router |
|------|-------------|------------|
| 렌더링 기본값 | Client Component | Server Component |
| 데이터 페칭 | `getServerSideProps`/`getStaticProps` | `fetch` + `async` 컴포넌트 |
| 레이아웃 | `_app.tsx` 단일 | 중첩 레이아웃 (`layout.tsx`) |
| 라우팅 | 파일 기반 | 폴더 기반 (route groups, parallel routes) |
| 로딩/에러 | 수동 구현 | `loading.tsx`, `error.tsx` 컨벤션 |

**프로젝트별 선택 기준**:
- **이그나이트 Admin CMS**: App Router 채택. 관리자 콘솔이라 SEO 필요성이 낮았지만, 중첩 레이아웃(사이드바+탑바+콘텐츠)이 복잡해서 App Router의 `layout.tsx` 패턴이 적합했습니다. Server Component로 RBAC 권한 체크를 서버에서 처리하여 보안도 강화했습니다.
- **마이리얼트립 통합숙소**: Pages Router 사용. 도입 시점이 App Router 초기 안정화 단계였고, 팀 경험이 Pages Router에 집중되어 있어 리스크를 최소화했습니다.

핵심은 "새로운 것이 항상 좋다"가 아니라, 팀 역량·프로젝트 요구사항·기술 성숙도를 종합적으로 고려하는 것입니다.

</details>

**꼬리 질문**:
- Server Component와 Client Component의 경계를 어떻게 설정하나요?
- `use client` 디렉티브의 영향 범위는?
- Streaming SSR을 활용한 경험이 있나요?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **경계 설정 기준**: 인터랙션(onClick, onChange)이 있거나, 브라우저 API를 사용하거나, state/effect 훅이 필요한 컴포넌트만 Client Component로 지정합니다. 데이터 페칭, 정적 UI, 인증 체크 등은 Server Component에 유지합니다. "Client 경계를 가능한 말단(leaf)으로 내린다"는 원칙을 따랐습니다.

- **`use client` 영향 범위**: 해당 파일과 그 파일이 import하는 모든 모듈이 Client Component가 됩니다. 단, Server Component를 children으로 전달하는 것은 가능합니다(Composition 패턴). Client Component인 InteractiveWrapper 안에 Server Component인 DataDisplay를 children으로 넘겨 Server Component의 이점을 최대한 유지했습니다.

- **Streaming SSR**: Admin CMS 대시보드에서 각 섹션을 별도의 Suspense 경계로 감싸 먼저 준비된 섹션부터 스트리밍 전송했습니다. 사용자는 전체 데이터가 준비될 때까지 기다리지 않고 점진적으로 콘텐츠를 볼 수 있었습니다.

</details>

---

### Q5. SSR + 클라이언트 하이드레이션 최적화로 초기 로딩을 2.1s → 1.2s로 개선했는데, 구체적인 방법은?
> **출제 의도**: SSR/하이드레이션 최적화 실무 역량
>
> **경력 연관**: 이그나이트 Admin CMS

<details>
<summary><b>✅ 답변</b></summary>

**1) 서버 측 최적화**:
- Server Component 도입으로 클라이언트 번들에서 제외되는 컴포넌트를 분리하여 JS 번들 사이즈 약 30% 감소
- 서버에서 인증 토큰 검증과 초기 데이터 페칭을 동시 수행하여 클라이언트 워터폴 제거

**2) 하이드레이션 최적화**:
- `React.lazy` + `Suspense`로 초기 하이드레이션에 필요 없는 컴포넌트(모달, 드로어 등) 지연 로딩
- `dynamic(() => import(...), { ssr: false })` 로 클라이언트 전용 컴포넌트(차트, 에디터)를 SSR에서 제외

**3) 리소스 최적화**:
- Critical CSS 인라인 삽입으로 FOUC 제거
- 폰트 `preload`와 `font-display: swap` 적용
- 서드파티 스크립트를 `afterInteractive` 전략으로 지연 로딩

**측정**: Lighthouse CI에서 LCP 2.1s → 1.2s, TTI 3.5s → 2.0s 개선을 확인했습니다.

</details>

**꼬리 질문**:
- 하이드레이션 미스매치 에러를 어떻게 디버깅하나요?
- Selective Hydration을 활용한 경험이 있나요?
- SSR에서 인증 상태를 어떻게 처리했나요?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **하이드레이션 미스매치 디버깅**: React 18 콘솔의 상세 diff를 기반으로 원인 파악합니다. 주요 원인은 `Date.now()` 등 서버/클라이언트 결과가 다른 코드, 브라우저 전용 API 사용, 서드파티 확장이 DOM을 변경하는 경우입니다. 의도적인 차이는 `suppressHydrationWarning`으로 처리하되 남용하지 않았습니다.

- **Selective Hydration**: Suspense 경계로 감싼 영역이 독립적으로 하이드레이션됩니다. Admin CMS에서 사이드바, 헤더, 메인 콘텐츠를 각각 별도 Suspense로 감싸 사용자가 클릭한 영역을 우선 하이드레이션하는 동작이 자동으로 이루어졌습니다.

- **SSR 인증 처리**: Next.js Middleware에서 쿠키 기반 토큰을 검증하고, 유효하지 않으면 로그인 페이지로 리다이렉트했습니다. Server Component에서는 `cookies()` API로 토큰을 읽어 서버에서 사용자 정보를 가져와 클라이언트에서 별도 인증 API 호출 없이 즉시 인증된 UI를 렌더링할 수 있었습니다.

</details>

---

### Q6. SSR + JSON-LD 기반 SEO 최적화를 통해 검색 노출 순위를 개선했는데, 어떤 전략을 사용했나요?
> **출제 의도**: SEO 기술적 이해와 실무 적용 역량
>
> **경력 연관**: 기아 인증 중고차

<details>
<summary><b>✅ 답변</b></summary>

**1) JSON-LD 구조화 데이터**: 차량 상세 페이지에 `Vehicle` + `Product` 스키마를 적용하여 Google 검색 결과에 가격, 주행거리, 연식 등이 Rich Snippet으로 노출되도록 했습니다. SSR 시점에 서버에서 데이터를 기반으로 동적 생성했습니다.

**2) 동적 메타 태그**: Next.js의 `generateMetadata`로 페이지별 title, description, OG 태그를 서버에서 생성. 차량명, 가격, 이미지를 포함한 OG 태그로 소셜 미디어 공유 시 풍부한 미리보기를 제공했습니다.

**3) 기술적 SEO**: `sitemap.xml` 동적 생성(차량 목록 API 기반), `robots.txt`로 의미 없는 필터 URL 크롤링 차단, Canonical URL 설정으로 중복 콘텐츠 방지, SSR로 크롤러가 완전한 HTML을 받도록 보장했습니다.

**4) Core Web Vitals 개선**: LCP, CLS 최적화가 Google 검색 순위에 직접 반영되므로, 이미지 최적화와 레이아웃 시프트 방지에 집중했습니다. Google Search Console에서 노출수와 클릭수 변화를 추적하여 효과를 측정했습니다.

</details>

**꼬리 질문**:
- JSON-LD 스키마를 어떤 기준으로 선택했나요?
- 동적 메타 태그 생성은 어떻게 처리했나요?
- Core Web Vitals가 SEO에 미치는 영향을 어떻게 측정했나요?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **스키마 선택 기준**: Google 구조화 데이터 가이드라인과 Schema.org 문서를 참고했습니다. 인증 중고차이므로 `Vehicle`, 구매 가능 상품이므로 `Product`+`Offer`를 결합했습니다. Google Rich Results Test로 스니펫을 미리 확인하며 개선했습니다.

- **동적 메타 태그**: 차량 상세 API 응답(차량명, 가격, 이미지 URL)을 기반으로 SSR 시점에 동적 생성했습니다. 이미지 누락 시 기본 OG 이미지를 폴백으로 설정하고, description은 차량 스펙을 조합하여 자연스러운 문장으로 생성했습니다.

- **Core Web Vitals와 SEO**: Google Search Console의 "Core Web Vitals" 보고서에서 "양호" 판정 URL 비율 변화를 추적하고, 검색 성과(노출수, 평균 게재순위)와 상관관계를 분석했습니다. LCP/CLS 개선 후 약 2~3주 후부터 노출수 증가가 관찰되었습니다.

</details>

---

### Q7. React의 렌더링 최적화 전략에 대해 설명해주세요. `React.memo`, `useMemo`, `useCallback`을 각각 언제 사용하나요?
> **출제 의도**: 렌더링 최적화에 대한 실무 판단력

<details>
<summary><b>✅ 답변</b></summary>

**React.memo**: 부모 리렌더링 시 props가 변경되지 않은 자식의 리렌더링을 방지합니다. 렌더링 비용이 높은 컴포넌트(큰 리스트, 차트), 동일한 props로 자주 리렌더링되는 컴포넌트에 적용합니다.

**useMemo**: 비용이 큰 계산 결과를 메모이제이션합니다. 배열 필터링/정렬 등 O(n) 이상 연산, 참조 동일성이 중요한 객체/배열에 적용합니다.

**useCallback**: 함수의 참조 동일성을 유지합니다. `React.memo`된 자식에게 전달하는 콜백, 의존성 배열에 포함되는 함수에 적용합니다.

**실무 원칙**: 측정 없이 최적화하지 않습니다. React Profiler로 실제 병목을 확인한 후 적용합니다. 딜러포탈에서 48개국 × 3개 브랜드 필터링 리스트에 `React.memo`를, 필터 결과 계산에 `useMemo`를 적용하여 체감 성능을 개선했습니다.

</details>

**꼬리 질문**:
- 모든 컴포넌트에 `React.memo`를 적용하면 안 되는 이유는?
- `useMemo`의 비용이 재계산보다 클 수 있는 경우는?
- React Compiler(React Forget)가 수동 최적화를 대체할 수 있을까요?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **React.memo 남용 문제**: 이전 props를 저장하는 메모리 비용, 매 렌더마다 shallow comparison하는 CPU 비용이 발생합니다. props가 거의 항상 변경되는 컴포넌트에 적용하면 비교 비용만 추가됩니다. 객체/배열 props를 인라인으로 전달하면 매번 새 참조가 생성되어 메모이제이션이 무용지물이 됩니다.

- **useMemo 비용이 더 큰 경우**: 단순 원시값 계산, 의존성이 자주 변경되어 거의 매번 재계산되는 경우, 결과값이 원시 타입이라 참조 동일성이 의미 없는 경우에는 메모리 점유와 비교 비용이 오히려 손해입니다.

- **React Compiler**: 수동 `memo`/`useMemo`/`useCallback` 대부분을 대체할 수 있지만, 현재 모든 패턴을 완벽히 최적화하지는 못하고, 사이드 이펙트가 있는 코드에서 예상치 못한 동작이 있을 수 있어 점진적 도입을 권장합니다.

</details>
