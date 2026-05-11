# 디자인 시스템 질문

### Q1. 마이리얼트립 디자인 시스템을 6개 프로덕트 팀에 제공한 경험에서, 디자인 시스템의 성공 기준은 무엇이었나요?
> **출제 의도**: 디자인 시스템 전략과 조직 내 채택 역량
>
> **경력 연관**: 마이리얼트립 디자인시스템

<details>
<summary><b>✅ 답변</b></summary>

**성공 기준 3가지**:

1. **채택률**: 6개 프로덕트 팀 모두가 실제로 사용하고 있는지. 주간 npm 다운로드 수와 각 프로젝트의 import 비율로 측정했습니다.
2. **일관성**: 디자인 시안과 구현 결과물 간의 시각적 차이가 줄어들었는지. 디자이너-개발자 간 "이거 왜 달라요?" 이슈가 감소했는지.
3. **생산성**: 새 페이지/기능 구현 시 공통 컴포넌트 활용으로 개발 시간이 단축되었는지.

**도입 전후 변화**:
- UI 컴포넌트 개발 시간 약 40% 단축 (기존에 팀마다 버튼/모달을 각각 구현했던 것이 공통화)
- 디자인 QA 지적사항 약 60% 감소 (일관된 토큰 시스템으로 색상/간격 오차 제거)
- 신규 팀원 온보딩 시 "Storybook 보면서 작업하세요"로 설명 비용 절감

**팀별 요구사항 조율**: 월 1회 디자인 시스템 싱크 미팅을 운영하여 각 팀의 요구사항을 수집하고 우선순위를 협의했습니다. 범용 컴포넌트는 디자인 시스템에, 도메인 특화 컴포넌트는 각 팀 프로젝트에 두는 경계를 명확히 했습니다.

</details>

---

### Q2. Compound Component 패턴을 선택한 이유와 구현 방식을 설명해주세요.
> **출제 의도**: 컴포넌트 설계 패턴에 대한 깊은 이해
>
> **경력 연관**: 마이리얼트립 디자인시스템

<details>
<summary><b>✅ 답변</b></summary>

**선택 이유**: 소비하는 개발자에게 유연한 조합과 명시적인 구조를 제공하기 위해서입니다.

**예시 - Select 컴포넌트**:
```tsx
<Select value={value} onChange={onChange}>
  <Select.Trigger>{selectedLabel}</Select.Trigger>
  <Select.Content>
    <Select.Item value="a">옵션 A</Select.Item>
    <Select.Item value="b">옵션 B</Select.Item>
  </Select.Content>
</Select>
```

**구현 방식**:
1. 최상위 `Select` 컴포넌트에서 `Context`를 생성하여 open/close 상태, 선택값, 이벤트 핸들러를 공유
2. 하위 컴포넌트(`Trigger`, `Content`, `Item`)는 `useContext`로 부모의 상태에 접근
3. TypeScript로 `Select.Trigger` 등의 타입을 정확하게 정의하여 IDE 자동완성 지원

**패턴 비교**:
- **Props 전달 방식** (`<Select options={[...]} />`): 간단하지만, 커스터마이징(아이콘 삽입, 그룹 헤더 등)이 어려움
- **Render Props**: 유연하지만 코드가 복잡해지고 가독성이 떨어짐
- **Compound Component**: 유연성과 가독성의 균형. 내부 구조가 명시적으로 보이면서도 자유로운 조합 가능

</details>

**꼬리 질문**:
- 컴포넌트 간 암묵적 의존성의 단점 보완은?
- 타입 안전성 보장은?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **암묵적 의존성 보완**: `Select.Item`을 `Select` 바깥에서 사용하면 Context가 없어 에러가 발생합니다. 이를 개선하기 위해 커스텀 `useSelectContext` 훅에서 Context가 null인 경우 명확한 에러 메시지를 throw하도록 했습니다: "Select.Item은 반드시 Select 컴포넌트 안에서 사용해야 합니다."

- **타입 안전성**: `Select` 컴포넌트의 타입을 `React.FC & { Trigger: typeof Trigger; Content: typeof Content; Item: typeof Item }` 형태로 정의하여 IDE에서 `Select.`을 입력하면 하위 컴포넌트가 자동완성됩니다. 제네릭을 활용하여 `<Select<string>>` 형태로 value 타입을 전파하면 `onChange` 콜백의 파라미터 타입도 자동으로 추론됩니다.

</details>

---

### Q3. Figma 토큰을 CSS 변수로 자동 변환하는 파이프라인을 어떻게 구축했나요?
> **출제 의도**: 디자인 토큰 시스템 자동화 역량
>
> **경력 연관**: 이그나이트 딜러포탈

<details>
<summary><b>✅ 답변</b></summary>

**파이프라인 구조**:

```
Figma Tokens Plugin → JSON Export → Style Dictionary 변환 → CSS Variables 생성 → npm 배포
```

1. **Figma에서 토큰 관리**: Tokens Studio(Figma 플러그인)로 디자이너가 색상, 타이포그래피, 간격 등을 토큰으로 정의
2. **JSON Export**: 플러그인에서 토큰을 JSON 파일로 내보내어 Git 저장소에 커밋
3. **Style Dictionary 변환**: JSON 토큰을 CSS Custom Properties(`--color-primary: #0066FF`), SCSS 변수, JS 상수 등 여러 포맷으로 변환
4. **3개 브랜드 분기**: 현대/기아/제네시스의 토큰을 별도 파일로 관리하되, Semantic Token 이름은 동일하게 유지 (예: `--color-brand-primary`가 브랜드마다 다른 값)

**토큰 계층**:
```
Primitive: --blue-500: #0066FF
Semantic:  --color-brand-primary: var(--blue-500)  // 현대
           --color-brand-primary: var(--red-600)   // 기아
Component: --button-bg: var(--color-brand-primary)
```

컴포넌트는 Semantic 토큰만 참조하므로, 브랜드 전환 시 Primitive 매핑만 변경하면 전체 UI가 자동으로 업데이트됩니다.

</details>

**꼬리 질문**:
- Semantic Token과 Primitive Token의 계층 구조는?
- 디자이너가 Figma에서 토큰을 변경하면 코드까지 반영되는 플로우는?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **토큰 계층**: Primitive는 실제 값(`#0066FF`), Semantic은 용도 기반 별칭(`brand-primary`), Component는 컴포넌트 전용(`button-bg`). 개발자는 Component/Semantic 토큰만 사용하고, Primitive는 직접 참조하지 않습니다. 이렇게 하면 "파란색을 조금 더 진하게"라는 변경이 Primitive 한 곳만 수정하면 전체에 반영됩니다.

- **변경 반영 플로우**: 디자이너가 Tokens Studio에서 토큰 수정 → GitHub에 JSON 푸시 → CI가 Style Dictionary 빌드 트리거 → 변환된 CSS/JS 파일로 토큰 패키지 새 버전 배포 → 각 프로젝트에서 의존성 업데이트. Renovate/Dependabot으로 자동 PR이 생성되어 개발자가 리뷰 후 머지합니다.

</details>

---

### Q4. 상호작용 컴포넌트 중첩 시 `validateDOMNesting` 에러를 ContextAPI로 해결한 방법은?
> **출제 의도**: DOM 규격 이해와 런타임 에러 방지 전략
>
> **경력 연관**: 마이리얼트립 디자인시스템

<details>
<summary><b>✅ 답변</b></summary>

**문제**: `<button>` 안에 `<a>`, `<a>` 안에 `<button>` 등 인터랙티브 요소 중첩 시 React가 `validateDOMNesting` 경고를 발생시키고, 접근성과 동작에도 문제가 생깁니다.

**ContextAPI 해결 방식**:

1. **InteractiveContext 생성**: 인터랙티브 요소(`Button`, `Link`, `Clickable`)가 렌더링될 때 Context에 `isInsideInteractive: true`를 제공
2. **중첩 감지**: 하위 인터랙티브 컴포넌트가 렌더링될 때 Context를 읽어 부모에 인터랙티브 요소가 있으면 경고 또는 에러 발생
3. **자동 태그 전환**: 중첩이 감지되면 `<button>` 대신 `<div role="button" tabIndex={0}>`으로 렌더링하거나, 개발 환경에서 명확한 에러 메시지를 throw

```tsx
const InteractiveContext = createContext(false);

function Button({ children }) {
  const isNested = useContext(InteractiveContext);
  if (isNested && process.env.NODE_ENV === 'development') {
    console.error('Button은 다른 인터랙티브 요소 안에 중첩될 수 없습니다.');
  }
  return (
    <InteractiveContext.Provider value={true}>
      <button>{children}</button>
    </InteractiveContext.Provider>
  );
}
```

이 패턴을 블로그에 공유했고(zigae.com), 커뮤니티에서 비슷한 문제를 겪는 개발자들에게 참고가 되었습니다.

</details>

---

### Q5. Conventional Commits + Changesets 기반 자동 배포 파이프라인을 설명해주세요.
> **출제 의도**: 라이브러리 배포 자동화 역량
>
> **경력 연관**: 마이리얼트립 디자인시스템

<details>
<summary><b>✅ 답변</b></summary>

**워크플로우**:

1. **개발자가 변경사항 커밋**: Conventional Commits 형식 (`feat:`, `fix:`, `BREAKING CHANGE:`)
2. **Changeset 파일 생성**: PR에서 `npx changeset`으로 변경 내용과 영향 범위(MAJOR/MINOR/PATCH)를 기록
3. **PR 머지 후 자동화**:
   - CI가 changeset 파일을 읽어 버전 범핑 결정
   - `CHANGELOG.md` 자동 생성
   - npm 패키지 자동 배포
   - 슬랙 알림으로 새 버전과 변경사항 전파

**Semantic Versioning 기준**:
- **MAJOR**: Breaking Change (API 변경, props 삭제/이름 변경)
- **MINOR**: 새 컴포넌트/props 추가 (하위 호환)
- **PATCH**: 버그 수정, 스타일 미세 조정

**Breaking Change 관리**: MAJOR 업데이트 시 마이그레이션 가이드를 CHANGELOG에 포함하고, 최소 2주간 이전 버전을 유지하여 소비 팀에 전환 시간을 제공했습니다.

</details>

---

### Q6. 아토믹 디자인 패턴의 장단점과 실제 적용 경험을 설명해주세요.
> **출제 의도**: 컴포넌트 계층 설계 역량
>
> **경력 연관**: 브랜디 커머스 플랫폼

<details>
<summary><b>✅ 답변</b></summary>

**장점**:
- **재사용성**: Atoms(Button, Input)부터 작게 만들어 조합하므로 재사용이 용이
- **일관성**: 같은 Atom을 사용하므로 UI 일관성 유지
- **테스트 용이**: 작은 단위부터 독립적으로 테스트 가능

**단점 (실무에서 느낀 한계)**:
- **분류 모호성**: Molecules과 Organisms의 경계가 주관적. 팀원마다 다르게 분류하여 혼란
- **과도한 추상화**: 간단한 기능도 5계층(Atoms→Pages)을 거쳐야 한다는 강박이 생겨 불필요한 파일 분리
- **수정 비용**: 하위 Atom 변경이 상위 컴포넌트에 연쇄 영향

**브랜디에서의 적용**: PHP→React 전환 초기에 아토믹 패턴을 채택하여 컴포넌트 구조를 잡는 데 도움이 되었으나, 프로젝트가 커지면서 Feature-based 구조로 점진적으로 전환했습니다. 공통 UI 컴포넌트는 아토믹 계층을 유지하되, 비즈니스 로직이 포함된 컴포넌트는 도메인 기준으로 폴더를 구성했습니다.

</details>

---

### Q7. Scroll prevent가 iPad OS 13에서 미작동한 문제를 어떻게 해결했나요?
> **출제 의도**: 크로스 브라우징/디바이스 이슈 해결 역량
>
> **경력 연관**: 마이리얼트립 디자인시스템

<details>
<summary><b>✅ 답변</b></summary>

**문제**: 바텀시트/모달 오픈 시 `body`의 스크롤을 막아야 하는데, iPad OS 13에서 `overflow: hidden`이 동작하지 않는 이슈. Sentry 기준 해당 기기 사용자 8%.

**원인**: iPad OS 13부터 Safari가 데스크톱 모드를 기본으로 사용하여, User-Agent가 Mac으로 표시됩니다. 기존 코드가 User-Agent 기반으로 iOS를 감지하여 터치 이벤트 기반 스크롤 방지를 적용했는데, iPad가 감지되지 않아 일반 데스크톱 로직(overflow: hidden만 적용)이 실행된 것.

**해결**:
1. `navigator.maxTouchPoints > 0`으로 터치 지원 디바이스를 감지. iPad OS 13도 터치를 지원하므로 정확히 감지됨
2. 터치 디바이스에서는 `touchmove` 이벤트를 `preventDefault`하고, `body`에 `position: fixed; top: -{scrollY}px`를 적용
3. 모달 닫힐 때 저장된 scrollY 위치로 복원

```tsx
const isTouch = navigator.maxTouchPoints > 0;
// User-Agent 대신 feature detection으로 디바이스 구분
```

**교훈**: User-Agent 기반 감지는 브라우저/OS 업데이트에 취약하므로, 가능하면 feature detection을 사용해야 합니다.

</details>

**꼬리 질문**:
- 모달/바텀시트의 body scroll lock 일반적 접근법은?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

일반적인 접근법으로 3가지가 있습니다:

1. **overflow: hidden**: `body`에 적용. 가장 간단하지만 iOS Safari에서 완벽하지 않음
2. **position: fixed**: `body`를 fixed로 고정하고 현재 scrollY를 `-top`으로 설정. 닫을 때 scrollTo로 복원. iOS에서 안정적이지만 스크롤 위치 저장/복원 로직 필요
3. **touch-action: none + touchmove preventDefault**: 터치 이벤트를 직접 차단. 가장 확실하지만 모달 내부 스크롤도 막혀서, 모달 내부에서는 이벤트를 허용하는 조건부 로직 필요

실무에서는 `body-scroll-lock` 같은 라이브러리를 참고하되, iOS Safari의 바운스 스크롤까지 고려하면 position: fixed 방식이 가장 안정적이었습니다. CSS `overscroll-behavior: contain`은 모달 내부 스크롤이 body로 전파되는 것을 방지하지만, iOS Safari 지원이 늦어서 폴백이 필요했습니다.

</details>
