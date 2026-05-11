# 트러블슈팅 질문

### Q1. useEffect 남용으로 발생한 무한 루프를 어떻게 진단하고 해결했나요?
> **출제 의도**: React 안티패턴 인식과 해결 역량
>
> **경력 연관**: 마이리얼트립 렌터카/숙소 고도화

<details>
<summary><b>✅ 답변</b></summary>

**문제**: 레거시 코드에서 컴포넌트당 평균 3~4개의 useEffect가 사용되고 있었으며, 의존성 배열 관리 실수로 무한 루프, 불필요한 API 호출, 디버깅 난이도 증가.

**진단**:
1. React DevTools Profiler로 무한 렌더링 패턴 확인
2. useEffect의 의존성 배열에 매 렌더링마다 새로 생성되는 객체/배열이 포함된 경우 발견
3. useEffect 간 상태 업데이트 체인이 순환 의존성을 형성하는 케이스 다수

**대표 패턴과 해결**:

| 안티패턴 | 해결 방법 |
|---------|----------|
| useEffect로 파생 상태 계산 | `useMemo`로 대체 |
| useEffect로 이벤트 핸들링 | 이벤트 핸들러로 직접 처리 |
| useEffect 체인 (A→B→C) | 하나의 useEffect로 통합 또는 커스텀 훅으로 추상화 |
| useEffect에서 setState → 리렌더링 → useEffect 재실행 | 의존성 배열 정리 + useRef로 이전 값 비교 |

**결과**: 컴포넌트당 useEffect 평균 3.7개 → 0.7개. Jira 레이블 추적 기준 useEffect 관련 버그 약 80% 감소.

**원칙**: "useEffect는 외부 시스템과의 동기화에만 사용한다" (React 공식 문서의 You Might Not Need an Effect 원칙 적용)

</details>

---

### Q2. @tanstack/react-virtual Safari 크로스 브라우징 이슈를 오픈소스 PR로 해결한 과정을 설명해주세요.
> **출제 의도**: 오픈소스 문제 분석 및 기여 역량
>
> **경력 연관**: 마이리얼트립 통합숙소

<details>
<summary><b>✅ 답변</b></summary>

**증상**: Sentry에서 iOS Safari 유저 15%에서 가상 스크롤 목록이 제대로 표시되지 않는 에러 수집. 특정 스크롤 위치에서 아이템이 비어 보이거나 겹쳐 표시됨.

**분석 과정**:
1. **재현**: BrowserStack의 실제 iOS Safari 환경에서 이슈 재현
2. **원인 파악**: Safari의 `scrollTop`/`offsetHeight` 계산 방식이 Chrome과 다름. 서브픽셀 렌더링에서 소수점 처리 차이로 가상 스크롤의 위치 계산이 틀어짐
3. **코드 분석**: @tanstack/react-virtual 라이브러리 소스 코드를 fork하여 문제 지점 파악

**PR 작성**:
1. 최소 재현 코드(Reproduction)를 포함한 Issue 생성
2. 수정 코드 + 테스트 케이스를 포함한 PR 제출 ([PR #518](https://github.com/TanStack/virtual/pull/518))
3. 메인테이너와 코드 리뷰 후 머지

**배운 점**: 
- 오픈소스 기여 시 최소 재현 코드가 Issue 채택의 핵심
- 라이브러리 내부 구조를 이해하면 workaround가 아닌 근본 해결이 가능
- 사내 코드에서 fork 버전을 임시로 사용하다가 PR 머지 후 공식 버전으로 전환

</details>

---

### Q3. 캘린더 컴포넌트의 전체 리렌더링 문제를 어떻게 해결했나요?
> **출제 의도**: 렌더링 최적화 실무 역량
>
> **경력 연관**: 마이리얼트립 Staynet

<details>
<summary><b>✅ 답변</b></summary>

**문제**: table 기반 캘린더에서 하나의 셀에 input을 삽입하면 전체 테이블이 리렌더링. Performance 탭 측정 시 800ms+ 소요.

**원인 분석**:
1. `<table>` 구조에서 한 셀의 상태 변경이 부모 `<table>`을 통해 전체 셀에 전파
2. `<input>` onChange 이벤트가 상위 컴포넌트의 state를 업데이트하여 전체 re-render 유발

**해결 방법**:
1. **`<table>` → `<div>` 기반 가상화**: CSS Grid로 캘린더 레이아웃을 구성하여 각 행/셀을 독립적으로 렌더링
2. **React.memo 적용**: 각 셀 컴포넌트에 `React.memo`를 적용하여 props가 변경되지 않은 셀은 리렌더링 방지
3. **Row 단위 격리**: 변경 영향 범위를 단일 행으로 제한. 한 행의 셀 변경이 다른 행에 영향 없음

**결과**: 셀 단위 렌더링 비용 85% 절감 (800ms → 120ms, React Profiler 측정).

**교훈**: HTML 시맨틱 요소(`<table>`)가 반드시 최선은 아닙니다. 인터랙티브 UI에서는 렌더링 제어가 용이한 `<div>` 기반 구조가 성능상 유리할 수 있습니다. 접근성은 `role="grid"`, `role="gridcell"` ARIA 속성으로 보완했습니다.

</details>

---

### Q4. Barrel 파일 순환 참조로 CI가 12분 걸리던 문제를 어떻게 해결했나요?
> **출제 의도**: 빌드 성능 문제 진단 역량
>
> **경력 연관**: 마이리얼트립 FE libs TF

<details>
<summary><b>✅ 답변</b></summary>

**문제**: 공통 모듈 모노레포에서 Lint + Jest 실행 시 12분+ 소요. CI Actions 워크플로우 로그에서 비정상적인 메모리 사용량 확인.

**진단**:
1. Jest `--verbose` 모드에서 모듈 해석(module resolution) 로그 확인
2. Barrel 파일(`index.ts`)의 re-export 체인에서 순환 참조 발견:
   ```
   @mrt/utils/index.ts → @mrt/auth/index.ts → @mrt/http/index.ts → @mrt/utils/index.ts
   ```
3. 순환 참조로 인해 모듈 해석 인스턴스가 무한 생성되어 메모리 폭증

**원인**: 각 패키지의 `index.ts`가 패키지의 모든 export를 re-export하는 Barrel 파일 패턴. 패키지 A의 Barrel에서 패키지 B를 import하고, 패키지 B의 Barrel에서 다시 패키지 A를 import하면서 순환 발생.

**해결**:
1. Barrel 파일 제거
2. 직접 import 경로로 전환: `import { formatDate } from '@mrt/utils'` → `import { formatDate } from '@mrt/utils/date'`
3. ESLint 규칙으로 Barrel 파일 import 금지

**결과**: 12분 → 5분 (CI 전후 비교). 메모리 사용량도 정상 수준으로 복귀.

</details>

---

### Q5. 프론트엔드에서 메모리 누수를 어떻게 진단하나요?
> **출제 의도**: 브라우저 메모리 관리 역량
>
> **경력 연관**: 마이리얼트립 Staynet (HMR 메모리 누수), 이그나이트 (PDF 뷰어)

<details>
<summary><b>✅ 답변</b></summary>

**진단 도구와 방법**:

1. **Chrome DevTools Memory 탭**:
   - **힙 스냅샷(Heap Snapshot)**: 특정 시점의 메모리 상태를 촬영. 시간별 스냅샷을 비교(Comparison)하여 증가하는 객체 확인
   - **Allocation Timeline**: 메모리 할당 시점과 해제되지 않는 객체를 실시간 추적
   - **Allocation Sampling**: CPU 프로파일링처럼 메모리 할당을 함수별로 추적

2. **Performance Monitor**: 실시간으로 JS Heap Size, DOM Node 수, Event Listener 수를 모니터링

3. **일반적인 누수 패턴**:
   - `setInterval`/`setTimeout` 해제 안 됨 → `useEffect` cleanup에서 `clearInterval`
   - 이벤트 리스너 해제 안 됨 → `useEffect` cleanup에서 `removeEventListener`
   - Closure가 큰 객체를 참조 → 참조 해제 또는 WeakRef 사용
   - DOM 분리 후에도 JS에서 참조 유지(Detached DOM nodes)

**실제 사례 - PDF 뷰어**:
- 500페이지+ 문서를 렌더링하면 메모리 사용량이 계속 증가
- 가상 스크롤링으로 뷰포트에 보이는 페이지만 렌더링하여 메모리 사용량 70% 절감
- Memory Profiler 힙 스냅샷 비교로 개선 효과 검증

</details>

---

### Q6. SSR + 클라이언트 하이드레이션 불일치 에러를 어떻게 해결했나요?
> **출제 의도**: SSR 트러블슈팅 역량
>
> **경력 연관**: 이그나이트 Admin CMS

<details>
<summary><b>✅ 답변</b></summary>

**문제**: 서버에서 렌더링한 HTML과 클라이언트에서 하이드레이션 시 생성하는 HTML이 다르면 React가 경고를 발생시키고, 최악의 경우 전체 페이지를 다시 렌더링.

**일반적인 원인과 해결**:

| 원인 | 해결 |
|------|------|
| `Date.now()`, `Math.random()` 사용 | 서버/클라이언트에서 동일한 시드 사용 또는 클라이언트에서만 실행 |
| `window`, `localStorage` 참조 | `typeof window !== 'undefined'` 체크 또는 `useEffect`에서 실행 |
| 서버/클라이언트 환경변수 차이 | `NEXT_PUBLIC_` prefix 사용으로 통일 |
| 조건부 렌더링의 서버/클라이언트 결과 다름 | `useEffect`로 마운트 후 상태 변경, 또는 `suppressHydrationWarning` 사용 |

**Admin CMS에서의 실제 사례**: 
- RBAC 기반 메뉴 렌더링에서 서버는 권한 정보가 없어 전체 메뉴를 렌더링, 클라이언트는 권한에 따라 필터링하여 불일치 발생
- 해결: `getServerSideProps`에서 권한 정보를 서버에서 미리 가져와 동일한 결과를 렌더링하도록 수정

**SSR + 클라이언트 하이드레이션 최적화**: 초기 로딩 2.1s → 1.2s 달성. 서버에서 데이터를 포함한 HTML을 생성하고, 클라이언트에서 React Query의 `dehydratedState`로 캐시를 초기화하여 추가 API 호출 없이 즉시 인터랙션 가능.

</details>
