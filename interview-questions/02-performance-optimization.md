# 성능 최적화 질문

## Core Web Vitals

### Q1. LCP 2.4s → 1.1s, CLS 0.25 → 0.05로 개선한 구체적인 방법을 설명해주세요.
> **출제 의도**: Core Web Vitals 개선 실무 경험과 측정 기반 접근
>
> **경력 연관**: 기아 인증 중고차

<details>
<summary><b>✅ 답변</b></summary>

**LCP 2.4s → 1.1s 개선**:
1. **이미지 최적화**: `next/image`의 `priority` 속성을 LCP 대상 이미지(차량 메인 사진)에 적용하여 preload 처리. `sizes` 속성으로 뷰포트별 적절한 이미지 사이즈를 요청하여 불필요한 대역폭 절약
2. **지연 로딩**: 뷰포트 밖의 차량 리스트 이미지는 `loading="lazy"`로 설정하고, 스크롤 시점에 로딩
3. **Critical Rendering Path 최적화**: above-the-fold 영역에 필요한 CSS만 인라인으로 삽입하고, 나머지는 `rel="preload"`로 비동기 로딩
4. **서버 응답 시간 개선**: 차량 상세 API 응답을 ISR(Incremental Static Regeneration)로 캐싱하여 TTFB 단축

**CLS 0.25 → 0.05 개선**:
1. **이미지에 명시적 width/height 지정**: `next/image`가 자동으로 aspect-ratio를 설정하여 레이아웃 시프트 방지
2. **폰트 로딩 최적화**: `font-display: swap`과 함께 폰트 사이즈 조정 디스크립터(`size-adjust`)로 FOUT에 의한 CLS 최소화
3. **동적 콘텐츠 공간 확보**: 광고, 배너 영역에 `min-height`를 사전 지정
4. **스켈레톤 UI**: 데이터 로딩 중 최종 레이아웃과 동일한 크기의 스켈레톤을 표시

WebPageTest로 3G/LTE/WiFi 환경별로 측정하여 모든 환경에서 "Good" 범위 달성을 확인했습니다.

</details>

**꼬리 질문**:
- LCP에 영향을 주는 요소는 무엇이고, 가장 효과적인 개선 방법은?
- CLS 0.25 → 0.05 개선에서 가장 큰 원인은?
- `next/image`의 `priority`, `sizes`, `placeholder` 속성 활용법은?
- WebPageTest와 Lighthouse 결과가 다를 때 어떤 기준으로 판단하나요?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **LCP 영향 요소**: (1) 서버 응답 시간(TTFB) (2) 리소스 로딩 시간(이미지/폰트) (3) 렌더링 차단 리소스(CSS/JS) (4) 클라이언트 사이드 렌더링 지연. 가장 효과적인 것은 LCP 대상 리소스를 `preload`하고 렌더 차단 리소스를 최소화하는 것입니다.

- **CLS 최대 원인**: 이미지에 width/height가 없어 로딩 후 레이아웃이 밀리는 것이 가장 큰 원인이었습니다. `next/image`로 전환하면서 자동 aspect-ratio가 적용되어 대부분 해결되었습니다.

- **next/image 속성 활용**: `priority`는 LCP 대상 이미지(히어로, 메인 차량 사진)에만 적용합니다. `sizes`는 `(max-width: 768px) 100vw, 50vw`처럼 뷰포트별 필요 사이즈를 명시하여 모바일에서 불필요하게 큰 이미지를 다운로드하지 않게 합니다. `placeholder="blur"`는 blurDataURL과 함께 사용하여 로딩 중 저해상도 프리뷰를 표시합니다.

- **WebPageTest vs Lighthouse**: Lighthouse는 시뮬레이션(throttling) 기반이고, WebPageTest는 실제 디바이스/네트워크에서 측정합니다. 실제 사용자 경험에 가까운 WebPageTest를 기본으로 하되, CI에서는 Lighthouse를 회귀 방지 용도로 활용했습니다. 차이가 큰 경우 CrUX(Chrome User Experience Report) 데이터를 최종 기준으로 삼았습니다.

</details>

---

### Q2. CloudFront 캐시 전략 수립으로 정적 자원 응답 시간을 210ms → 85ms로 개선한 방법은?
> **출제 의도**: CDN 캐시 전략 설계 역량
>
> **경력 연관**: 이그나이트 딜러포탈

<details>
<summary><b>✅ 답변</b></summary>

**캐시 전략을 리소스 유형별로 차등 적용**했습니다.

1. **정적 자원 (JS/CSS/이미지)**: 파일명에 content hash를 포함시켜(`main.a1b2c3.js`) `Cache-Control: public, max-age=31536000, immutable`로 1년 캐시. 파일 내용이 바뀌면 hash가 바뀌므로 캐시 무효화가 자연스럽게 처리됩니다.

2. **HTML 파일**: `Cache-Control: no-cache` (매 요청마다 재검증) + `ETag`로 변경 여부 확인. HTML이 최신 해시가 포함된 JS/CSS를 참조하므로 HTML만 최신화되면 전체가 갱신됩니다.

3. **API 응답 (CloudFront → API Gateway)**: `s-maxage=300, stale-while-revalidate=60`으로 CDN에서 5분 캐시하면서, 만료 직후에도 기존 캐시를 제공하며 백그라운드 갱신.

**CloudFront 설정**:
- Cache Policy: 쿼리 스트링, 헤더 등 캐시 키를 최소화하여 캐시 적중률 극대화
- Origin Request Policy: 오리진에 필요한 헤더만 전달
- 결과적으로 캐시 적중률(Hit Rate) 92%를 달성했고, CloudFront Metrics에서 응답 시간 210ms → 85ms 개선을 확인했습니다.

</details>

**꼬리 질문**:
- `max-age`, `s-maxage`, `stale-while-revalidate` 차이는?
- 해시 기반 파일명과 캐시 무효화 전략은?
- CloudFront의 Cache Policy와 Origin Request Policy 설정은?
- 캐시 적중률(Hit Rate)과 모니터링은?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **헤더 차이**: `max-age`는 브라우저+CDN 모두에 적용되는 캐시 수명, `s-maxage`는 CDN(shared cache)에만 적용됩니다. `stale-while-revalidate`는 캐시 만료 후에도 지정된 시간 동안 기존 캐시를 제공하면서 백그라운드에서 새 응답을 가져옵니다. 사용자는 항상 즉각적인 응답을 받으면서 데이터 신선도도 유지됩니다.

- **해시 + 캐시 무효화**: Webpack/Turbopack의 `[contenthash]`로 빌드 시 파일 내용 기반 해시를 생성합니다. 내용이 변경된 파일만 새 URL이 되므로, 기존 캐시를 수동으로 무효화(invalidation)할 필요가 없습니다. CloudFront invalidation은 HTML 파일 배포 시에만 사용했습니다.

- **Cache Policy**: 기본 정책에서 쿼리 스트링을 캐시 키에서 제외(마케팅 UTM 파라미터 등이 캐시 미스를 유발하므로), 필요한 경우만 화이트리스트로 포함했습니다. Origin Request Policy는 Authorization 헤더 등 오리진에 필요한 것만 전달합니다.

- **모니터링**: CloudFront의 캐시 적중률 메트릭을 Datadog에 연동하여 대시보드로 모니터링했습니다. 적중률이 갑자기 떨어지면 캐시 키 설정 변경이나 배포 문제를 의심하여 알림을 설정했습니다.

</details>

---

### Q3. Prefetch 및 클라이언트 캐싱으로 LCP 1.8s → 0.9s를 달성한 구체적인 구현 방식은?
> **출제 의도**: Prefetch 전략과 캐싱 아키텍처 설계
>
> **경력 연관**: 이그나이트 딜러포탈

<details>
<summary><b>✅ 답변</b></summary>

**1) React Query `prefetchQuery` 활용**:
- 사용자가 다음 페이지로 이동할 가능성이 높은 시점(네비게이션 hover, 현재 페이지 데이터 로딩 완료 시점)에 `queryClient.prefetchQuery`로 다음 페이지 데이터를 미리 가져옴
- prefetch된 데이터는 React Query 캐시에 저장되어, 실제 페이지 전환 시 네트워크 요청 없이 즉시 렌더링

**2) HomePrefetcher 최적화**:
- Guard chain(AuthGuard → ConsentGuard)이 순차 실행되는 동안 API 호출이 대기하는 구조가 2.5~3s의 지연을 유발
- HomePrefetcher를 ConsentGuard의 sibling으로 배치하여, AuthGuard 통과 직후 Guard 검사와 병렬로 `prefetchQuery` 시작
- 3G: ConsentGuard→API 간격 1873ms → 1207ms (36% 개선)
- 일반: 374ms → 140ms (63% 개선)

**3) 클라이언트 캐싱 전략**:
- `staleTime: 5 * 60 * 1000` (5분): 자주 변경되지 않는 차량 정보/딜러 정보 등에 적용하여 불필요한 refetch 방지
- 페이지 이동 후 뒤로 가기 시 캐시된 데이터로 즉시 렌더링

</details>

**꼬리 질문**:
- `prefetchQuery`와 `useQuery`의 캐시 공유 메커니즘은?
- Prefetch를 어떤 기준으로 선택적으로 적용했나요?
- 네트워크 대역폭이 제한된 환경에서 과도한 prefetch를 어떻게 제어했나요?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **캐시 공유**: `prefetchQuery`와 `useQuery`는 동일한 `queryClient` 인스턴스의 캐시를 공유합니다. 같은 Query Key를 사용하면 prefetch로 가져온 데이터를 `useQuery`가 캐시에서 즉시 읽습니다. `staleTime` 내에 있으면 추가 네트워크 요청 없이 캐시된 데이터를 사용합니다.

- **선택적 Prefetch 기준**: (1) 사용자 행동 분석 기반으로 이동 확률이 높은 경로(Home → 차량 목록, 목록 → 상세) (2) 네비게이션 링크에 `onMouseEnter`/`onFocus` 이벤트로 hover 시점에 prefetch (3) Intersection Observer로 뷰포트에 들어온 링크의 목적지를 prefetch. 모든 페이지를 prefetch하지 않고 주요 동선에만 집중했습니다.

- **대역폭 제한 환경 제어**: `navigator.connection` API로 네트워크 상태를 확인하여 `effectiveType`이 `2g`/`slow-2g`인 경우 prefetch를 비활성화했습니다. 또한 `navigator.connection.saveData`가 true인 경우도 prefetch를 건너뛰었습니다. 동시 prefetch 요청 수도 2개로 제한하여 메인 콘텐츠 로딩을 방해하지 않도록 했습니다.

</details>

---

## 렌더링 성능

### Q4. 캘린더 컴포넌트 렌더링 비용을 85% 절감한 구체적인 방법을 설명해주세요.
> **출제 의도**: 렌더링 최적화 실전 역량, DOM 구조와 성능의 관계
>
> **경력 연관**: 마이리얼트립 Staynet

<details>
<summary><b>✅ 답변</b></summary>

**문제 상황**: table 기반 캘린더에서 셀 안에 input을 삽입할 때 하나의 input 값 변경 시 전체 캘린더(약 42셀)가 리렌더링되어 Performance 탭 기준 800ms+ 소요.

**원인 분석**:
- HTML `<table>`은 브라우저가 전체 테이블의 레이아웃을 한 번에 계산하므로, 하나의 셀 변경이 전체 테이블 reflow를 유발
- table의 자식 요소 관계(`<tr>` → `<td>`)에서 React가 행 단위 최적화를 적용하기 어려움

**해결 방법**:
1. **div 기반 그리드 구조로 전환**: CSS Grid(`display: grid; grid-template-columns: repeat(7, 1fr)`)로 테이블과 동일한 레이아웃을 구현. div 기반이므로 각 셀이 독립적인 레이아웃 컨텍스트를 가짐
2. **React.memo + row 단위 격리**: 각 행(week)을 별도 컴포넌트로 분리하고 `React.memo`를 적용. 특정 행의 input 변경이 해당 행만 리렌더링하도록 격리
3. **커스텀 비교 함수**: `React.memo`에 날짜 범위 비교 함수를 전달하여 해당 주(week)에 속하는 날짜만 변경되었을 때 리렌더링

**결과**: React Profiler에서 셀 단위 렌더링 비용 85% 절감, 800ms → 120ms로 개선.

</details>

**꼬리 질문**:
- table이 왜 느렸나요?
- `React.memo`의 커스텀 비교 함수 경험은?
- 800ms → 120ms를 React Profiler로 어떻게 측정했나요?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **table 성능 문제**: `<table>`은 기본적으로 `table-layout: auto`로 동작하여 모든 셀의 콘텐츠를 확인한 후 열 너비를 결정합니다. 하나의 셀이 변경되면 전체 테이블을 다시 레이아웃(reflow)합니다. `table-layout: fixed`로 일부 완화 가능하지만, React의 렌더링과 브라우저의 DOM 업데이트가 겹치면서 성능 병목이 발생했습니다.

- **커스텀 비교 함수**: `React.memo(WeekRow, (prevProps, nextProps) => { return prevProps.weekDates.every((d, i) => d.value === nextProps.weekDates[i].value && d.isSelected === nextProps.weekDates[i].isSelected) })` 형태로, 해당 주에 속하는 날짜 데이터만 비교하여 무관한 주의 리렌더링을 방지했습니다.

- **측정 방법**: React DevTools Profiler에서 "Record" 후 input 값을 변경하는 시나리오를 실행했습니다. Flamegraph에서 각 컴포넌트의 렌더 시간을 확인하고, before(전체 42셀 렌더)/after(해당 행 7셀만 렌더)를 비교했습니다. Commit 단위로 총 렌더 시간이 800ms에서 120ms로 감소한 것을 확인했습니다.

</details>

---

### Q5. PDF 뷰어에서 가상 스크롤링으로 대용량 문서 렌더링을 지원하고 메모리 사용량 70%를 절감한 방법은?
> **출제 의도**: 가상화(Virtualization) 구현 역량과 메모리 최적화
>
> **경력 연관**: 이그나이트 Admin CMS

<details>
<summary><b>✅ 답변</b></summary>

**구현 아키텍처**:

1. **가상 스크롤링 원리**: 전체 500+ 페이지를 한 번에 DOM에 렌더링하지 않고, 현재 뷰포트에 보이는 페이지(약 2~3개)와 위아래 버퍼(각 1~2개)만 실제 DOM에 렌더링합니다. 나머지 영역은 빈 placeholder(적절한 높이만 가진)로 채워 스크롤 위치를 유지합니다.

2. **페이지 높이 관리**: PDF 페이지마다 높이가 다를 수 있으므로, 최초 로딩 시 각 페이지의 메타데이터에서 높이를 계산하여 배열로 저장합니다. 스크롤 위치에 따라 이진 탐색으로 현재 보이는 페이지 인덱스를 빠르게 결정합니다.

3. **메모리 관리**: 뷰포트를 벗어난 페이지의 캔버스를 `null`로 해제하고, `pdf.js`의 페이지 객체도 cleanup하여 메모리를 반환합니다. IntersectionObserver로 페이지가 뷰포트에 진입/이탈할 때 렌더링/해제를 트리거합니다.

4. **측정**: Memory Profiler에서 힙 스냅샷을 비교했습니다. 가상화 적용 전 500페이지 로딩 시 힙 약 1.2GB → 적용 후 약 350MB로 70% 절감.

</details>

**꼬리 질문**:
- 뷰포트 밖의 요소 관리 방법은?
- PDF 페이지 높이가 가변적일 때 처리 방법은?
- 500페이지에서 빠른 페이지 이동(Jump to Page)은?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **뷰포트 밖 요소**: 실제 DOM에서 제거하고 동일 높이의 빈 `<div>`로 대체합니다. 이때 `will-change: transform`이나 `contain: strict`를 적용하여 브라우저가 해당 영역의 레이아웃을 독립적으로 처리하게 했습니다.

- **가변 높이 처리**: pdf.js의 `getPage()` → `getViewport({ scale })` 로 각 페이지의 실제 높이를 미리 계산하여 배열에 저장합니다. 누적 높이 배열(prefix sum)을 만들어 스크롤 위치로부터 현재 페이지를 O(log n) 이진 탐색으로 계산합니다.

- **Jump to Page**: 누적 높이 배열을 이용해 목표 페이지의 스크롤 위치를 즉시 계산하고, `scrollTo({ top: targetOffset })`로 이동합니다. 이동 후 해당 페이지를 우선 렌더링하고, 주변 페이지는 IntersectionObserver가 트리거할 때 렌더링합니다. 사용자가 빠르게 스크롤할 때는 debounce를 적용하여 중간 페이지의 불필요한 렌더링을 방지했습니다.

</details>

---

### Q6. Guard chain 병렬화로 API 호출 지연을 63% 개선한 아키텍처를 설명해주세요.
> **출제 의도**: 비동기 처리 아키텍처 설계와 측정 기반 최적화
>
> **경력 연관**: 이그나이트 딜러포탈 - HomePrefetcher 최적화

<details>
<summary><b>✅ 답변</b></summary>

**문제 상황**:
- 기존 Guard chain이 `useEffect` 기반 순차 구조: `AuthGuard` → `ConsentGuard` → Home 컴포넌트 마운트 → API 호출
- ConsentGuard 완료까지 실제 API 호출이 시작되지 않아 2.5~3초 지연 발생

**해결 아키텍처**:
```
기존: AuthGuard → ConsentGuard → Home → API 호출 (순차)
개선: AuthGuard → [ConsentGuard | HomePrefetcher] (병렬)
```

HomePrefetcher를 ConsentGuard의 sibling으로 배치했습니다. AuthGuard를 통과한 시점에서 인증 토큰이 확보되므로, ConsentGuard의 동의 검사와 무관하게 API 데이터를 미리 가져올 수 있습니다. `prefetchQuery`로 가져온 데이터는 React Query 캐시에 저장되어, 이후 Home 컴포넌트가 마운트되면 캐시에서 즉시 사용합니다.

**측정**: Playwright로 `page.on('request')` 이벤트를 캡처하여 ConsentGuard 완료 시점과 첫 API 요청 시점의 차이를 측정했습니다. Network throttling을 적용하여 3G/일반 네트워크 환경 모두에서 Before/After를 비교했습니다.

- 3G: 1873ms → 1207ms (36% 개선)
- 일반: 374ms → 140ms (63% 개선)

</details>

**꼬리 질문**:
- Guard chain에서 어떤 부분을 병렬화할 수 있었나요?
- Playwright 네트워크 타이밍 측정의 구체적인 코드 구조는?
- 3G 환경 시뮬레이션 설정 방법은?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **병렬화 가능 판단**: AuthGuard(인증)는 API 호출의 필수 전제조건이므로 순차 유지, ConsentGuard(동의)는 UI 표시에만 영향을 주고 API 호출 권한과는 무관하므로 API prefetch와 병렬화가 가능했습니다. 핵심은 "이 Guard가 실패하면 prefetch한 데이터가 무용지물인가?"를 기준으로 판단한 것입니다.

- **Playwright 측정 코드**: `page.on('request', req => timestamps.push({ url: req.url(), time: Date.now() }))` 형태로 요청 타이밍을 수집했습니다. ConsentGuard 완료는 특정 DOM 요소의 출현(`page.waitForSelector`)으로 감지하고, 해당 시점과 첫 API 요청의 시간 차이를 계산했습니다. 3회 반복 측정의 중앙값을 사용했습니다.

- **3G 시뮬레이션**: Playwright의 `page.route('**/*', route => { ... })` 또는 CDP(Chrome DevTools Protocol) 연결을 통해 `Network.emulateNetworkConditions`로 `download: 1.6 * 1024 * 1024 / 8, upload: 768 * 1024 / 8, latency: 150` 등의 조건을 설정했습니다.

</details>

---

## 번들 최적화

### Q7. Barrel 파일 제거로 빌드/테스트 시간을 12분 → 5분으로 줄인 경험을 설명해주세요.
> **출제 의도**: 모듈 시스템과 번들링 최적화 이해
>
> **경력 연관**: 마이리얼트립 TF / Frontend libs

<details>
<summary><b>✅ 답변</b></summary>

**문제 상황**: 공통 모듈 모노레포에서 Lint + Jest 실행 시 12분+ 소요. CI 파이프라인의 병목이었습니다.

**원인 분석**:
- 각 패키지의 `index.ts`(Barrel 파일)가 패키지 내 모든 모듈을 re-export
- Barrel 파일 간 순환 참조 발생: `@common/utils/index.ts` → `@common/hooks/index.ts` → `@common/utils/index.ts`
- Jest와 ESLint가 모듈을 해석할 때 순환 참조로 인해 모듈 해석 인스턴스가 무한에 가깝게 증가, 메모리와 시간을 대량 소비

**해결 방법**:
1. **Barrel 파일 제거**: `import { something } from '@common/utils'` → `import { something } from '@common/utils/something'`으로 직접 import
2. **ESLint 규칙 추가**: `no-restricted-imports` 규칙으로 Barrel 파일 import를 금지
3. **IDE 자동 import 설정**: VS Code의 auto-import가 직접 경로를 사용하도록 tsconfig `paths` 조정

**결과**: 빌드/테스트 시간 12분 → 5분 (CI 전후 비교), 메모리 사용량도 크게 감소.

</details>

**꼬리 질문**:
- Barrel 파일이 순환 참조를 유발하는 메커니즘은?
- Tree-shaking과 Barrel 파일의 관계는?
- 직접 import로 전환할 때 DX 저하를 어떻게 보완했나요?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **순환 참조 메커니즘**: `utils/index.ts`가 `utils/formatDate.ts`를 export하고, `formatDate.ts`가 `hooks/useFormat`을 import하며, `hooks/index.ts`가 다시 `utils/index.ts`의 무언가를 import하면 순환이 발생합니다. Barrel 파일은 패키지의 모든 것을 re-export하므로, 이런 순환이 쉽게 만들어집니다. 직접 import하면 실제 필요한 파일만 의존하므로 순환이 발생하기 어렵습니다.

- **Tree-shaking 관계**: Barrel 파일은 사이드 이펙트가 없다는 보장이 어려워 번들러가 안전하게 tree-shake하지 못할 수 있습니다. `package.json`의 `sideEffects: false`로 완화 가능하지만, Barrel 파일이 깊어지면 번들러가 전체 모듈 그래프를 탐색해야 해서 빌드 시간이 증가합니다.

- **DX 보완**: tsconfig의 `paths`를 세밀하게 설정하여 `@common/utils/formatDate` 형태의 import를 지원했고, VS Code의 auto-import가 직접 경로를 우선 추천하도록 설정했습니다. 또한 코드모드(codemod) 스크립트를 작성하여 기존 barrel import를 일괄 변환했습니다.

</details>

---

### Q8. Webpack HMR 메모리 누수 (30분 내 힙 2GB+) 문제를 어떻게 진단하고 해결했나요?
> **출제 의도**: 메모리 누수 진단 능력
>
> **경력 연관**: 마이리얼트립 Staynet

<details>
<summary><b>✅ 답변</b></summary>

**증상**: 개발 서버에서 30분 정도 작업하면 브라우저 탭이 느려지고, 결국 크래시. Chrome Task Manager에서 해당 탭 메모리가 2GB+ 도달.

**진단 과정**:
1. Chrome DevTools Memory 탭에서 힙 스냅샷을 5분 간격으로 3번 찍음
2. 스냅샷 비교(Comparison)에서 `system / JSArrayBufferData`와 관련 모듈 객체가 계속 증가하는 것을 확인
3. HMR 시 이전 모듈이 GC되지 않고 누적되는 패턴 발견
4. Webpack 설정에서 `output.filename`에 `[hash]` 옵션이 적용되어 있었음

**원인**: `[hash]`(현재는 `[fullhash]`)가 빌드마다 새로운 해시를 생성하여, HMR 시 이전 번들의 참조가 해제되지 않고 메모리에 누적. HMR은 변경된 모듈만 교체해야 하는데, 해시 기반 파일명으로 인해 이전 번들이 새 번들과 공존하는 상태가 됨.

**해결**: 개발 환경에서 `output.filename`의 `[hash]` 옵션을 제거하고, 프로덕션 빌드에서만 `[contenthash]`를 사용하도록 분리. HMR은 해시 없이 모듈 ID로 교체가 이루어지므로 이전 모듈이 정상적으로 GC됨.

</details>

**꼬리 질문**:
- 힙 스냅샷에서 어떤 패턴으로 누수를 확인했나요?
- hash 옵션이 번들을 누적시키는 원리는?
- HMR과 Full Reload의 트레이드오프는?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **누수 확인 패턴**: "Comparison" 뷰에서 두 스냅샷 간 `#New` 컬럼이 계속 증가하는 객체 유형을 찾습니다. Detached DOM 노드, 클로저에 의한 참조 유지, 이벤트 리스너 누적 등이 일반적인 패턴입니다. 이 경우는 모듈 시스템의 `__webpack_module_cache__`에 이전 버전 모듈이 계속 남아있었습니다.

- **hash 누적 원리**: `[hash]`는 전체 컴파일에 대한 해시이므로 HMR 때마다 새 해시가 생성됩니다. Webpack의 HMR 런타임이 새 해시 기반 청크를 로드하면서 이전 해시 기반 청크의 참조를 완전히 해제하지 못해, 두 버전의 모듈이 메모리에 공존하게 됩니다.

- **HMR vs Full Reload**: HMR은 상태를 유지하면서 변경된 모듈만 교체하여 DX가 좋지만, 메모리 누수나 상태 불일치 위험이 있습니다. Full Reload는 모든 상태가 초기화되어 깨끗하지만 DX가 떨어집니다. 실무에서는 HMR을 기본으로 사용하되, 특정 파일(라우팅 설정 등)은 Full Reload하도록 설정했습니다.

</details>

---

## 애니메이션 성능

### Q9. CSS 애니메이션과 JavaScript 애니메이션의 성능 차이를 설명하고, 실무에서 어떤 기준으로 선택했나요?
> **출제 의도**: 렌더링 파이프라인 이해와 애니메이션 성능 판단력
>
> **경력 연관**: 마이리얼트립 디자인시스템 (framer-motion)

<details>
<summary><b>✅ 답변</b></summary>

**브라우저 렌더링 파이프라인**:
```
JavaScript → Style → Layout → Paint → Composite
```

애니메이션 성능의 핵심은 **어느 단계부터 시작하느냐**입니다.

**Layout을 유발하는 속성 (가장 비쌈)**:
- `width`, `height`, `margin`, `padding`, `top`, `left` 등
- 변경 시 해당 요소 및 주변 요소의 레이아웃을 전부 재계산

**Paint만 유발하는 속성**:
- `background-color`, `color`, `border-color` 등
- Layout 재계산은 없지만 픽셀을 다시 그림

**Composite만 유발하는 속성 (가장 저렴)**:
- `transform`, `opacity`
- GPU에서 처리하므로 메인 스레드 차단 없음

**실무 기준**:
- 이동/크기 변환: `transform: translate/scale` → `top/left/width` 금지
- 페이드 인/아웃: `opacity` → `visibility/display` 전환 금지
- 복잡한 인터랙션(물리 기반, 제스처): framer-motion으로 JS 애니메이션 사용, 내부적으로 `transform`으로 최적화됨
- 단순 hover/transition: CSS `transition`으로 충분, JS 오버헤드 없음

마이리얼트립 디자인시스템의 바텀시트 슬라이드 애니메이션을 `bottom` 값 변경에서 `transform: translateY`로 전환하여 60fps를 안정적으로 유지했습니다.

</details>

**꼬리 질문**:
- Layout Thrashing이란 무엇이고, 어떻게 방지했나요?
- `will-change` 속성을 언제 사용하고 남용하면 왜 안 되나요?
- `requestAnimationFrame`을 직접 사용한 경험이 있나요?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **Layout Thrashing**: JS에서 레이아웃 속성(offsetHeight, scrollTop 등)을 읽으면 브라우저가 강제로 레이아웃을 계산합니다. 읽기와 쓰기가 교차하면 매번 재계산이 발생합니다. 방지법은 읽기를 모두 먼저 수행하고 쓰기를 일괄 처리하는 것입니다. 스크롤 이벤트 핸들러에서 `el.getBoundingClientRect()` 후 스타일을 변경하는 패턴이 대표적인 Thrashing 원인이었고, `IntersectionObserver`로 대체하여 해결했습니다.

- **`will-change` 주의점**: `will-change: transform`은 브라우저에게 "이 요소는 곧 변환될 것"이라고 힌트를 주어 GPU 레이어를 미리 생성합니다. 모든 요소에 적용하면 GPU 메모리가 과도하게 소비되고 오히려 성능이 저하됩니다. 애니메이션 시작 직전에 동적으로 추가하고 종료 후 제거하거나, 실제로 자주 변환되는 요소(슬라이더, 드로어)에만 제한적으로 사용했습니다.

- **`requestAnimationFrame` 활용**: 스크롤 기반 시차(parallax) 효과를 구현할 때 직접 사용했습니다. `scroll` 이벤트는 초당 수십 번 발생하므로 스로틀이 필요한데, `rAF`를 사용하면 브라우저의 렌더링 사이클에 맞춰 정확히 1프레임당 1번만 실행됩니다. `let ticking = false; window.addEventListener('scroll', () => { if (!ticking) { rAF(() => { updateParallax(); ticking = false; }); ticking = true; } })` 패턴으로 구현했습니다.

</details>

---

### Q10. framer-motion으로 애니메이션을 구현할 때 성능을 어떻게 관리했나요?
> **출제 의도**: 애니메이션 라이브러리 활용 역량과 성능 트레이드오프 이해
>
> **경력 연관**: 마이리얼트립 디자인시스템

<details>
<summary><b>✅ 답변</b></summary>

**framer-motion이 기본적으로 제공하는 최적화**:
- 모든 애니메이션을 내부적으로 `transform`/`opacity`로 처리 → Composite 단계만 유발
- `useMotionValue` + `useTransform`으로 React 리렌더링 없이 DOM 스타일을 직접 업데이트

**실무에서 추가로 관리한 포인트**:

**1) `AnimatePresence`와 목록 최적화**:
```tsx
// 잘못된 패턴: 목록 전체에 AnimatePresence
// 올바른 패턴: 개별 아이템에 layoutId 사용
<AnimatePresence>
  {items.map(item => (
    <motion.li key={item.id} layout layoutId={item.id}
      initial={{ opacity: 0 }} animate={{ opacity: 1 }} exit={{ opacity: 0 }}>
      {item.content}
    </motion.li>
  ))}
</AnimatePresence>
```
`layout` prop은 요소 위치 변경을 자동으로 애니메이션하는데, 목록 전체에 적용하면 모든 아이템이 매 렌더마다 위치를 재계산합니다. `layoutId`를 통해 공유 레이아웃 애니메이션으로 전환하여 대상 아이템만 애니메이션이 적용되도록 했습니다.

**2) `useReducedMotion` 대응**:
```tsx
const prefersReducedMotion = useReducedMotion();
const variants = prefersReducedMotion
  ? { hidden: {}, visible: {} }   // 애니메이션 없음
  : { hidden: { opacity: 0, y: 20 }, visible: { opacity: 1, y: 0 } };
```
운동 민감성 사용자 또는 저사양 기기에서 `prefers-reduced-motion` 미디어 쿼리에 따라 애니메이션을 비활성화했습니다. WCAG 2.1 AAA 기준이기도 합니다.

**3) 번들 사이즈**: framer-motion은 약 50KB(gzip)로 무겁습니다. 단순한 CSS transition으로 대체 가능한 곳은 framer-motion을 사용하지 않았고, 필요한 컴포넌트만 dynamic import로 지연 로딩했습니다.

</details>

**꼬리 질문**:
- `motion.div`를 많이 사용하면 성능에 어떤 영향이 있나요?
- `useMotionValue`와 `useState`의 차이는?
- 바텀시트 드래그-투-디스미스 구현 시 어떤 고려가 필요했나요?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **`motion.div` 남용 영향**: `motion.div`는 일반 `div`보다 무겁습니다. 내부적으로 MotionValue, 애니메이션 컨트롤러, 이벤트 리스너를 등록합니다. 스크롤 되는 긴 목록의 각 아이템에 `motion.div`를 사용하면 수백 개의 MotionValue 인스턴스가 생성됩니다. 정적인 요소(레이아웃이 고정된, 애니메이션이 없는)에는 일반 `div`를 사용하고, 실제로 애니메이션이 필요한 컨테이너 수준에만 `motion.div`를 적용했습니다.

- **`useMotionValue` vs `useState`**: `useState`는 값이 변경될 때 React 리렌더링을 트리거합니다. `useMotionValue`는 React 렌더 사이클 밖에서 값을 관리하여, 값이 변경되어도 리렌더링 없이 `useTransform`, `useSpring` 등으로 연결된 DOM 스타일을 직접 업데이트합니다. 드래그 중 매 프레임마다 값이 변경되는 제스처 애니메이션에서 `useState`를 사용하면 초당 60번 리렌더링이 발생하므로, `useMotionValue`가 필수입니다.

- **바텀시트 드래그-투-디스미스 구현**: `dragConstraints`로 드래그 범위를 제한하고, `onDragEnd`에서 속도(`velocity.y`)와 변위를 기준으로 닫을지 원위치할지 결정했습니다. 빠르게 아래로 스와이프(`velocity.y > 500`)하거나 절반 이상 내렸을 때(`y > height / 2`) 닫히도록 했습니다. 드래그 중 배경 오버레이의 `opacity`를 `useTransform(y, [0, height], [0.5, 0])`으로 연동하여 자연스럽게 사라지는 효과를 구현했습니다. iOS의 네이티브 시트 동작과 유사한 느낌을 내기 위해 `dragElastic: 0.2`로 바운스 효과도 추가했습니다.

</details>

---

### Q11. 스크롤 기반 애니메이션을 구현할 때 성능 문제를 어떻게 해결했나요?
> **출제 의도**: 스크롤 이벤트 최적화와 IntersectionObserver 활용
>
> **경력 연관**: 마이리얼트립 디자인시스템 / 기아 인증 중고차

<details>
<summary><b>✅ 답변</b></summary>

**scroll 이벤트의 문제점**:
- 스크롤 이벤트는 메인 스레드에서 동기적으로 실행되어 잦은 핸들러가 프레임 드롭을 유발
- `passive: false`인 스크롤 핸들러는 브라우저가 스크롤을 시작하기 전에 핸들러 실행을 기다려야 해서 스크롤 자체가 끊김

**패턴 1 - IntersectionObserver (요소 진입 시 1회성 애니메이션)**:
```tsx
// 뷰포트 진입 시 fade-in 애니메이션
const useScrollReveal = (ref: RefObject<Element>) => {
  const [isVisible, setIsVisible] = useState(false);

  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => { if (entry.isIntersecting) setIsVisible(true); },
      { threshold: 0.1 }
    );
    if (ref.current) observer.observe(ref.current);
    return () => observer.disconnect();
  }, []);

  return isVisible;
};
```
메인 스레드가 아닌 별도 스레드에서 교차 여부를 감지하므로 스크롤 성능에 영향 없음.

**패턴 2 - `passive` scroll + rAF (스크롤 연동 연속 애니메이션)**:
```typescript
// 패럴랙스, 진행 바 등 스크롤 값에 연속 반응하는 경우
let ticking = false;
window.addEventListener('scroll', () => {
  if (!ticking) {
    requestAnimationFrame(() => {
      updateAnimation(window.scrollY);
      ticking = false;
    });
    ticking = true;
  }
}, { passive: true }); // passive: true로 스크롤 차단 없음
```

**패턴 3 - framer-motion의 `useScroll` + `useTransform`**:
```tsx
const { scrollYProgress } = useScroll({ target: sectionRef });
const opacity = useTransform(scrollYProgress, [0, 0.5, 1], [0, 1, 0]);
// React 리렌더링 없이 스크롤 값을 opacity에 직접 연결
```

기아 인증 중고차 랜딩 페이지의 차량 스펙 섹션에서 스크롤 진입 시 카운트업 애니메이션을 IntersectionObserver로 구현했고, 페이지 상단의 진행 표시 바는 framer-motion `useScroll`로 구현했습니다.

</details>

**꼬리 질문**:
- `IntersectionObserver`의 `threshold`와 `rootMargin` 옵션을 어떻게 활용했나요?
- 스크롤 이벤트에 `throttle`보다 `rAF`를 선호하는 이유는?
- 저사양 기기에서 복잡한 스크롤 애니메이션을 어떻게 graceful degradation했나요?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **threshold와 rootMargin**: `threshold: 0.1`은 요소의 10%가 뷰포트에 들어올 때 트리거, `threshold: [0, 0.25, 0.5, 0.75, 1]`처럼 배열로 주면 각 구간마다 콜백이 호출되어 단계별 애니메이션에 활용합니다. `rootMargin: '0px 0px -100px 0px'`은 뷰포트 하단 100px 위에서 미리 트리거하여 사용자가 요소를 보기 직전에 애니메이션을 시작, 시각적으로 더 자연스러운 타이밍을 만들었습니다.

- **throttle vs rAF**: `throttle(fn, 16)`은 클럭 기반 16ms마다 실행하는데, 브라우저의 실제 렌더링 사이클과 맞지 않을 수 있습니다. 기기가 120Hz이면 8ms마다 프레임이 그려지는데 16ms 스로틀은 절반만 처리합니다. `rAF`는 브라우저가 다음 프레임을 그리기 직전에 정확히 한 번 콜백을 실행하므로 모니터 주사율에 자동으로 맞춰지고, 탭이 비활성화되면 자동으로 실행을 멈춰 배터리를 절약합니다.

- **Graceful Degradation**: `prefers-reduced-motion` 미디어 쿼리와 `navigator.hardwareConcurrency`(CPU 코어 수)를 조합하여 저사양 환경을 감지했습니다. `window.matchMedia('(prefers-reduced-motion: reduce)').matches`가 true이거나 코어 수가 4 미만이면 애니메이션 duration을 0으로 설정하거나 CSS 클래스로 대체했습니다. 핵심 정보는 애니메이션 없이도 즉시 표시되도록 했습니다(Content First).

</details>
