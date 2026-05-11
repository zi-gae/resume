# 웹뷰 & 네이티브 브릿지 질문

### Q1. 웹뷰 네이티브 브릿지 컴포넌트(바텀시트, iOS DatePicker)를 어떻게 구현했나요?
> **출제 의도**: 웹뷰-네이티브 통신 구현 역량
>
> **경력 연관**: 마이리얼트립 디자인시스템

<details>
<summary><b>✅ 답변</b></summary>

**아키텍처**: 웹뷰에서 네이티브 기능을 호출하고, 네이티브에서 결과를 웹뷰로 전달하는 양방향 브릿지입니다.

**통신 방식**:
```
[웹 → 네이티브]
- iOS: window.webkit.messageHandlers.{name}.postMessage(data)
- Android: window.Android.{name}(JSON.stringify(data))

[네이티브 → 웹]
- 네이티브가 window.{callbackName}(result)를 실행
```

**바텀시트 구현**:
1. 웹에서 `bridge.openBottomSheet({ title, items, selectedIndex })` 호출
2. 네이티브 바텀시트 UI 표시 (웹뷰보다 자연스러운 애니메이션, 제스처 지원)
3. 사용자 선택 후 네이티브가 콜백으로 선택 결과 전달
4. 웹에서 콜백 데이터를 받아 UI 업데이트

**iOS DatePicker 구현**:
- 웹의 `<input type="date">`는 iOS에서 기본 DatePicker가 열리지만, 커스터마이징이 제한적
- 네이티브 DatePicker를 브릿지로 호출하여 최소/최대 날짜, 초기값, 로케일 등을 세밀하게 제어
- 선택 완료 시 ISO 형식 날짜 문자열로 웹에 전달

**브릿지 추상화**:
```typescript
const bridge = {
  isNative: () => !!window.webkit || !!window.Android,
  openDatePicker: (options) => {
    if (bridge.isNative()) {
      return nativeDatePicker(options);  // 네이티브 호출
    }
    return webDatePicker(options);  // 웹 폴백
  }
};
```

네이티브 환경이 아니면 웹 폴백 컴포넌트를 렌더링하여, 데스크톱 브라우저에서도 동작합니다.

</details>

**꼬리 질문**:
- 브릿지 호출 시 에러 핸들링은?
- 네이티브 앱 버전별 브릿지 호환성 관리는?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **에러 핸들링**: 브릿지 호출에 타임아웃을 설정했습니다. 네이티브가 응답하지 않으면(앱 크래시, 브릿지 미구현 등) 5초 후 타임아웃으로 에러를 반환하고 웹 폴백을 실행합니다. 또한 `window.webkit` 존재 여부를 먼저 확인하여 브릿지가 없는 환경(일반 브라우저)에서의 에러를 방지합니다.

- **버전 호환성**: 네이티브 앱이 웹 로드 시 User-Agent 또는 쿠키에 앱 버전을 포함하여 전달합니다. 웹에서 버전을 파싱하여 해당 버전에서 지원하는 브릿지 기능만 호출합니다. 미지원 기능은 웹 폴백으로 처리합니다. 새로운 브릿지 기능 추가 시, 네이티브 앱 강제 업데이트 없이도 기존 사용자가 에러 없이 사용할 수 있도록 방어적 코딩이 필수입니다.

</details>

---

### Q2. 웹뷰에서 네이티브 앱과 인증 상태를 공유하는 방법을 설명해주세요.
> **출제 의도**: 웹뷰 인증 아키텍처
>
> **경력 연관**: 이그나이트 딜러포탈 / 마이리얼트립

<details>
<summary><b>✅ 답변</b></summary>

**일반적인 접근법**:

1. **쿠키 주입 방식**: 네이티브 앱이 웹뷰 로드 전에 인증 쿠키를 주입
   - `WKWebView.configuration.websiteDataStore`를 통해 쿠키를 설정
   - 웹뷰가 로드되면 쿠키가 이미 존재하므로 즉시 인증된 상태
   - 장점: 웹 코드 변경 없음. 단점: 쿠키 동기화 타이밍 이슈

2. **브릿지 방식**: 웹에서 브릿지를 통해 네이티브에 토큰 요청
   - `bridge.getAuthToken()` 호출 → 네이티브가 토큰 반환 → 웹이 API 요청에 사용
   - 장점: 명시적, 토큰 갱신 시점 제어 가능. 단점: 웹 코드에 네이티브 의존성

3. **URL 파라미터 방식**: 웹뷰 URL에 토큰을 쿼리 파라미터로 전달
   - 보안 취약 (URL 히스토리에 토큰 노출). 가급적 사용하지 않음

**딜러포탈에서의 적용**: 브릿지 방식을 채택했습니다. 네이티브 앱이 이미 자체 인증을 완료한 상태에서 웹뷰를 열기 때문에, 웹에서 별도 로그인 없이 브릿지를 통해 토큰을 가져옵니다. 토큰 만료 시에도 브릿지를 통해 네이티브에 갱신을 요청하고, 새 토큰을 받아 사용합니다.

</details>

---

### Q3. 웹뷰 성능 최적화를 위한 전략을 설명해주세요.
> **출제 의도**: 웹뷰 환경의 특수한 성능 고려사항
>
> **경력 연관**: 이그나이트 / 마이리얼트립

<details>
<summary><b>✅ 답변</b></summary>

**웹뷰 환경의 특수성**:
- 네이티브 브라우저 대비 리소스 제한 (메모리, CPU)
- 네이티브 앱 내에서 웹을 렌더링하므로 초기 로딩이 체감상 더 느림
- 네트워크 조건이 다양 (3G/LTE/WiFi)

**최적화 전략**:

1. **번들 사이즈 최소화**: 
   - Tree-shaking 확인, 불필요한 폴리필 제거
   - Dynamic import로 페이지별 코드 분할
   - 이미지는 WebP + `next/image` 최적화

2. **프리로딩**: 
   - 네이티브 앱 시작 시 웹뷰에 필요한 정적 리소스를 미리 캐싱
   - `<link rel="preconnect">`로 API 서버 연결 미리 수립

3. **렌더링 최적화**: 
   - SSR로 초기 HTML을 즉시 표시, 하이드레이션으로 인터랙션 활성화
   - 스켈레톤 UI로 로딩 중에도 레이아웃이 완성된 느낌 제공

4. **캐싱**: 
   - Service Worker로 정적 리소스 캐싱 (오프라인 대응)
   - React Query의 persistQueryClient로 이전 데이터를 로컬에 저장하여 재방문 시 즉시 표시

</details>

---

### Q4. 웹뷰에서 발생하는 크로스 브라우징 이슈를 어떻게 대응하나요?
> **출제 의도**: 다양한 웹뷰 엔진 대응 역량
>
> **경력 연관**: 마이리얼트립 디자인시스템 / 통합숙소

<details>
<summary><b>✅ 답변</b></summary>

**웹뷰 엔진 차이**:
- **iOS**: WKWebView (Safari 기반). iOS 버전에 따라 지원 기능 차이
- **Android**: Chrome Custom Tabs 또는 System WebView. 제조사/버전별 파편화
- **인앱 브라우저**: 카카오톡, 네이버, 페이스북 등 앱 내 브라우저는 또 다른 제한

**대응 전략**:

1. **Feature Detection 우선**: User-Agent가 아닌 기능 존재 여부로 분기
   ```javascript
   // ❌ User-Agent 기반 (불안정)
   if (navigator.userAgent.includes('iPhone')) { ... }
   // ✅ Feature Detection (안정적)
   if ('IntersectionObserver' in window) { ... }
   ```

2. **Polyfill 전략**: `@babel/preset-env`의 `useBuiltIns: 'usage'`로 타겟 브라우저에 필요한 폴리필만 자동 포함

3. **실제 사례 - @tanstack/react-virtual Safari 이슈**:
   - Sentry에서 iOS 유저 15% 영향 확인
   - 가상 스크롤이 Safari에서 스크롤 위치 계산 오류 발생
   - 오픈소스 라이브러리에 직접 PR 제출하여 해결 ([PR #518](https://github.com/TanStack/virtual/pull/518))

4. **테스트**: BrowserStack에서 주요 디바이스/OS 조합을 테스트하고, Sentry에서 디바이스별 에러 비율을 모니터링하여 우선순위를 정합니다.

</details>

---

### Q5. 웹뷰에서 딥링크와 유니버설 링크는 어떻게 처리하나요?
> **출제 의도**: 앱-웹 네비게이션 통합 이해
>
> **경력 연관**: 마이리얼트립

<details>
<summary><b>✅ 답변</b></summary>

**개념 구분**:
- **딥링크**: 앱 내 특정 화면으로 직접 이동하는 URL 스킴 (`myapp://product/123`)
- **유니버설 링크**: 일반 HTTP URL이 앱이 설치되어 있으면 앱으로, 아니면 웹으로 열리는 방식 (`https://www.example.com/product/123`)

**웹뷰에서의 처리**:

1. **웹 → 네이티브 화면 이동**:
   - 웹뷰 내에서 네이티브 화면(지도, 카메라 등)으로 이동 필요 시 브릿지를 통해 네이티브 네비게이션 호출
   - `bridge.navigate({ screen: 'NativeMap', params: { lat, lng } })`

2. **외부 링크 처리**:
   - 웹뷰 내에서 외부 URL 클릭 시, 네이티브가 인터셉트하여 인앱 브라우저 또는 외부 브라우저로 열기
   - 결제 URL(PG사)은 반드시 외부 브라우저에서 처리 (웹뷰 내 결제는 보안 제약)

3. **공유 링크**:
   - 사용자가 콘텐츠를 공유할 때 유니버설 링크 형태로 URL 생성
   - 수신자가 앱이 설치되어 있으면 해당 화면으로 직접 이동, 미설치 시 웹으로 열림

</details>

---

### Q6. 웹뷰 디버깅은 어떻게 하나요?
> **출제 의도**: 웹뷰 개발/디버깅 환경 이해
>
> **경력 연관**: 전체 경력

<details>
<summary><b>✅ 답변</b></summary>

**iOS 웹뷰 디버깅**:
1. Safari → 개발자 도구 → 시뮬레이터/실기기의 웹뷰를 선택하여 Web Inspector 사용
2. Console, Network, Elements 탭 등 일반 웹 개발과 동일한 도구 사용 가능
3. 실기기 디버깅 시 Mac에 USB 연결 + Safari 원격 디버깅 활성화 필요

**Android 웹뷰 디버깅**:
1. `chrome://inspect`에서 디바이스의 웹뷰를 감지
2. `WebView.setWebContentsDebuggingEnabled(true)` 설정이 활성화되어 있어야 함
3. Chrome DevTools로 일반 웹과 동일하게 디버깅

**프로덕션 환경 디버깅**:
- Sentry: JavaScript 에러 + 스택트레이스 + 디바이스 정보
- Datadog RUM: 사용자 세션 리플레이, 네트워크 타이밍, 에러 컨텍스트
- 커스텀 로깅: 브릿지 호출/응답을 콘솔에 로깅하여 통신 문제 진단

**어려운 점**: 인앱 브라우저(카카오톡, 네이버 등)는 Web Inspector를 연결할 수 없어, 에러 재현이 어렵습니다. 이 경우 Sentry의 디바이스 정보와 브레드크럼(사용자 행동 기록)을 최대한 활용하여 원인을 추적합니다.

</details>

---

### Q7. 웹뷰와 네이티브 간 Safe Area, 키보드 처리는 어떻게 하나요?
> **출제 의도**: 웹뷰 레이아웃 실무 이슈 대응
>
> **경력 연관**: 마이리얼트립 디자인시스템

<details>
<summary><b>✅ 답변</b></summary>

**Safe Area 처리**:
- iOS 노치/홈 인디케이터 영역을 고려하여 콘텐츠가 잘리지 않도록 해야 합니다
- `env(safe-area-inset-top)`, `env(safe-area-inset-bottom)` CSS 환경 변수 사용
- `<meta name="viewport" content="viewport-fit=cover">`를 설정하여 전체 화면 모드에서 safe area 값을 활용

```css
.bottom-bar {
  padding-bottom: calc(16px + env(safe-area-inset-bottom));
}
```

**키보드 처리**:
- 가상 키보드가 올라오면 웹뷰의 viewport가 줄어들어 레이아웃이 깨질 수 있음
- **문제**: `position: fixed`인 하단 버튼이 키보드에 가려지거나 키보드 위로 올라감
- **해결**: `visualViewport` API로 실제 표시 영역 크기를 감지하여 동적 조정

```javascript
window.visualViewport?.addEventListener('resize', () => {
  const keyboardHeight = window.innerHeight - window.visualViewport.height;
  // 키보드 높이만큼 하단 요소 위치 조정
});
```

- iOS에서 `input` 포커스 시 자동 스크롤이 과도하게 발생하는 이슈는 `scrollIntoViewIfNeeded` 대신 수동 스크롤 제어로 해결했습니다.

</details>

---

### Q8. 웹뷰에서 네이티브 뒤로가기(Back Navigation)와 웹 히스토리 충돌은 어떻게 처리하나요?
> **출제 의도**: 네이티브-웹 네비게이션 통합 실무 역량
>
> **경력 연관**: 마이리얼트립 / 이그나이트

<details>
<summary><b>✅ 답변</b></summary>

**문제 상황**: 네이티브 앱의 뒤로가기 버튼(Android 하드웨어 버튼, iOS 제스처)을 누르면, 웹 히스토리가 있어도 네이티브 스택에서 웹뷰 자체를 닫아버리는 현상이 발생합니다.

**해결 전략**:

1. **브릿지로 뒤로가기 이벤트 인터셉트**:
   - 네이티브가 뒤로가기 이벤트를 웹에 먼저 전달
   - 웹에서 `window.history.length > 1`이면 `history.back()`을 호출하고 네이티브에 "처리됨"을 응답
   - 웹 히스토리가 없으면 네이티브가 직접 뒤로가기(웹뷰 닫기) 처리

```typescript
// 네이티브로부터 뒤로가기 이벤트 수신
window.onNativeBackPressed = () => {
  if (window.history.length > 1) {
    window.history.back();
    return true; // 웹이 처리했음을 네이티브에 알림
  }
  return false; // 네이티브가 처리하도록 위임
};
```

2. **popstate 이벤트 관리**:
   - SPA에서 모달/드로어가 열린 경우, 뒤로가기 시 모달만 닫히도록 히스토리에 더미 엔트리를 push
   - 모달 open: `history.pushState({ modal: true }, '')`
   - 뒤로가기: `popstate` 이벤트에서 state를 확인하여 모달 닫기 처리

3. **실제 적용 사례**:
   - 마이리얼트립 숙소 검색 필터가 풀스크린 모달로 열릴 때, 네이티브 뒤로가기로 필터를 닫을 수 있도록 구현했습니다.
   - 브릿지가 없는 환경(일반 브라우저)에서는 `popstate`만으로 동작하도록 폴백 처리했습니다.

</details>

---

### Q9. 웹뷰 보안 취약점과 대응 방법을 설명해주세요.
> **출제 의도**: 웹뷰 환경의 보안 인식 및 실무 대응
>
> **경력 연관**: 이그나이트 딜러포탈 (GDPR/CCPA 대응)

<details>
<summary><b>✅ 답변</b></summary>

**웹뷰 특유의 보안 위협**:

1. **JavaScriptInterface 노출 (Android)**:
   - `@JavascriptInterface`로 노출된 메서드는 웹뷰 내 모든 JS에서 접근 가능
   - 악성 스크립트가 주입되면 네이티브 기능 무단 호출 위험
   - **대응**: 허용된 출처(origin)의 요청만 처리하도록 네이티브 측에서 검증

2. **XSS → 브릿지 탈취**:
   - 웹 콘텐츠에 XSS가 발생하면, 공격자가 브릿지를 통해 네이티브 기능 악용 가능
   - **대응**: CSP(Content Security Policy) 헤더 설정, 사용자 입력 sanitize

3. **로컬 파일 접근**:
   - `file://` 스킴을 허용하면 기기 내 파일에 접근 가능한 취약점
   - **대응**: `allowFileAccess = false`, `allowFileAccessFromFileURLs = false` 설정

4. **네트워크 트래픽 감청**:
   - 일부 오래된 Android WebView는 자체 인증서 검증이 취약
   - **대응**: Certificate Pinning 적용, HTTPS 강제

**딜러포탈에서의 보안 실무**:
- 글로벌 서비스 특성상 GDPR/CCPA 규정 대응이 필요했습니다.
- 국가별 동의 수집 여부를 프론트엔드 미들웨어로 구현하여, 동의 없이는 추적 스크립트(GA, Datadog RUM)가 로드되지 않도록 처리했습니다.
- 브릿지 호출 시 요청 출처를 검증하여 외부 도메인에서의 브릿지 접근을 차단했습니다.

</details>

---

### Q10. 웹뷰에서 파일 업로드/다운로드를 어떻게 처리하나요?
> **출제 의도**: 웹뷰 파일 처리 실무 경험
>
> **경력 연관**: 이그나이트 SaaS Admin CMS (PDF 뷰어, 문서 처리)

<details>
<summary><b>✅ 답변</b></summary>

**파일 업로드**:

웹뷰에서 `<input type="file">`은 플랫폼별로 동작이 다릅니다.

- **iOS**: `WKWebView`는 기본적으로 파일 선택을 지원하지만, 카메라/갤러리 접근 권한을 네이티브에서 설정해야 합니다.
  - `Info.plist`에 `NSCameraUsageDescription`, `NSPhotoLibraryUsageDescription` 추가 필요
- **Android**: `onShowFileChooser` 콜백을 구현해야 파일 선택 다이얼로그가 동작합니다. 미구현 시 파일 선택 자체가 안 됨

**파일 다운로드**:

웹뷰는 기본적으로 파일 다운로드를 지원하지 않아 브릿지 처리가 필요합니다.

```typescript
// 웹에서 다운로드 요청
bridge.downloadFile({
  url: fileUrl,
  filename: 'document.pdf',
  mimeType: 'application/pdf'
});
```

- 네이티브가 실제 파일을 다운로드하여 기기 저장소 또는 공유 시트(Share Sheet)로 처리
- 진행률을 브릿지 콜백으로 웹에 전달하여 Progress UI 표시

**SaaS Admin CMS PDF 뷰어에서의 적용**:
- 500페이지 이상 대용량 PDF를 웹뷰 내에서 직접 렌더링하기 위해 가상 스크롤링 기반 PDF 뷰어를 구현했습니다.
- 모바일 웹뷰에서는 메모리 제한으로 전체 페이지를 한 번에 렌더링할 수 없어, 뷰포트 기준 ±2 페이지만 활성화하고 나머지는 DOM에서 제거하는 방식으로 메모리 사용량을 70% 절감했습니다.

**꼬리 질문**: 이미지 촬영 후 즉시 업로드하는 플로우는?

네이티브 카메라 접근은 브릿지로 요청하고, 촬영 완료 후 base64 또는 임시 파일 URL로 웹에 전달합니다. 웹에서는 받은 데이터를 FormData로 변환하여 API에 업로드합니다. 네이티브 카메라를 사용하면 웹 `<input capture>`보다 더 나은 UX(저조도 모드, 연속 촬영 등)를 제공할 수 있습니다.

</details>

---

### Q11. 웹뷰 White Screen(빈 화면) 이슈를 어떻게 대응하나요?
> **출제 의도**: 웹뷰 로딩 실패 원인 파악 및 복구 전략
>
> **경력 연관**: 전체 경력 (Sentry, Datadog 활용)

<details>
<summary><b>✅ 답변</b></summary>

**White Screen의 주요 원인**:

1. **JS 번들 로드 실패**: 네트워크 불안정, CDN 장애, 캐시 불일치
2. **JS 런타임 에러**: Unhandled exception으로 React 렌더링 중단
3. **메모리 부족**: 저사양 기기에서 대용량 번들 파싱 중 OOM
4. **웹뷰 프로세스 크래시**: 안드로이드에서 `onRenderProcessGone` 이벤트 없이 백지 상태

**탐지 전략**:

```typescript
// ErrorBoundary로 렌더링 에러 포착
class AppErrorBoundary extends React.Component {
  componentDidCatch(error, info) {
    Sentry.captureException(error, { extra: info });
    // 네이티브에 에러 알림 → 앱 레벨 복구 UI 표시
    bridge.notifyError({ type: 'white-screen', message: error.message });
  }
}
```

- **Sentry**: `beforeSend` 훅에서 white screen 여부를 `document.body.children.length === 0`으로 감지하여 별도 태깅
- **Datadog RUM**: 세션 리플레이로 white screen 발생 직전 사용자 행동 파악

**복구 전략**:

1. **네이티브 복구**: 네이티브 앱이 일정 시간 내 웹뷰가 렌더링되지 않으면 웹뷰 reload 또는 에러 페이지 표시
2. **Service Worker 캐시**: 오프라인/CDN 장애 시 이전 버전 번들을 Service Worker가 제공하여 일부 기능 유지
3. **번들 분할**: 초기 번들 크기를 최소화하여 파싱 중 OOM 가능성 감소

</details>

---

### Q12. 웹뷰가 앱 포그라운드/백그라운드로 전환될 때 어떻게 처리하나요?
> **출제 의도**: 앱 라이프사이클에 따른 웹뷰 상태 관리
>
> **경력 연관**: 이그나이트 딜러포탈 (인증 토큰 관리)

<details>
<summary><b>✅ 답변</b></summary>

**백그라운드 전환 시 문제**:

1. **토큰 만료**: 앱이 백그라운드에 오래 있다가 복귀하면 액세스 토큰이 만료된 상태
2. **데이터 stale**: 오랫동안 백그라운드에 있던 후 복귀 시 표시 중인 데이터가 오래된 상태
3. **타이머/인터벌 불일치**: `setInterval`이 백그라운드에서 throttle되어 타이밍 로직 오류

**처리 방법**:

1. **브릿지로 앱 상태 수신**:
```typescript
// 네이티브 → 웹 앱 상태 전달
window.onAppStateChange = (state: 'foreground' | 'background') => {
  if (state === 'foreground') {
    // 토큰 유효성 재확인
    authService.validateAndRefreshToken();
    // stale 데이터 재조회
    queryClient.invalidateQueries({ stale: true });
  }
};
```

2. **Page Visibility API 활용** (브릿지 없는 환경 폴백):
```typescript
document.addEventListener('visibilitychange', () => {
  if (document.visibilityState === 'visible') {
    // 포그라운드 복귀 처리
  }
});
```

3. **딜러포탈 인증 처리**:
   - 앱 복귀 시 A 토큰 유효성을 먼저 검증하고, 만료 시 B 토큰도 연쇄 재발급하는 흐름을 구현했습니다.
   - 백그라운드 복귀를 감지하면 대기 중인 API 요청들을 일시 홀드하고, 토큰 갱신 완료 후 일괄 재시도합니다.

</details>

---

### Q13. 웹뷰에서 애니메이션 성능을 어떻게 최적화하나요?
> **출제 의도**: 웹뷰 렌더링 성능 실무 최적화 역량
>
> **경력 연관**: 마이리얼트립 디자인시스템 (framer-motion, 바텀시트)

<details>
<summary><b>✅ 답변</b></summary>

**웹뷰 애니메이션의 어려움**:
- 네이티브 대비 GPU 합성 레이어 제어가 제한적
- 저사양 기기에서 JS 스레드와 Main 스레드 모두 부하가 집중되면 프레임 드롭 발생
- `60fps` 유지 기준으로 한 프레임 = 16.67ms인데, JS 연산이 길면 애니메이션이 끊김

**최적화 전략**:

1. **GPU 합성 가속 유도**:
   - `transform`, `opacity`만 애니메이션에 사용 (layout/paint를 유발하는 `top`, `height` 지양)
   - `will-change: transform`으로 레이어 사전 승격 (남용 금지 — 메모리 소모)

2. **JS 연산 최소화**:
   - `framer-motion`의 `layout` 애니메이션은 내부적으로 FLIP 기법을 사용해 layout 재계산 비용을 줄임
   - `requestAnimationFrame`으로 JS 기반 애니메이션을 렌더링 사이클에 동기화

3. **네이티브 위임**:
   - 바텀시트, 모달 등 핵심 인터랙션은 네이티브 컴포넌트를 브릿지로 호출
   - 마이리얼트립 디자인시스템에서 바텀시트를 웹 구현과 네이티브 브릿지 두 가지로 제공한 이유가 이것입니다. 네이티브 바텀시트는 제스처와 물리 기반 스프링 애니메이션이 훨씬 자연스럽습니다.

4. **Reduced Motion 대응**:
```css
@media (prefers-reduced-motion: reduce) {
  /* 시각적 민감도가 있는 사용자를 위해 애니메이션 비활성화 */
  * { animation: none !important; transition: none !important; }
}
```

</details>
