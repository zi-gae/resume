# AI & 자동화 질문

### Q1. SonarQube Auto-Fix 파이프라인에서 AI 코드 수정의 범위를 어떻게 제한했나요?
> **출제 의도**: AI 기반 코드 자동화에서의 안전성 설계
>
> **경력 연관**: 이그나이트 SonarQube Auto-Fix

<details>
<summary><b>✅ 답변</b></summary>

**문제**: 초기 느슨한 프롬프트로 Claude가 관련 없는 코드까지 수정하는 문제 발생. 한 파일의 이슈를 수정하면서 다른 파일의 코드 스타일을 변경하거나, 불필요한 리팩토링을 수행.

**프롬프트 제약 설계**:

1. **허용 파일 목록 명시**: 수정 대상 파일만 나열하고, "이 파일 외에는 절대 수정하지 마라" 명시
2. **금지 항목 구체화**:
   - ❌ 코드 스타일 변경 (포맷팅, 네이밍)
   - ❌ 관련 없는 리팩토링
   - ❌ import 순서 변경
   - ❌ 주석 추가/삭제
3. **수정 범위 제한**: "SonarQube 이슈에 해당하는 코드 라인만 수정. 해당 라인의 로직을 변경할 때 필요한 최소한의 주변 코드만 변경 허용"

**비대화형 실행**: Claude Code CLI를 `--non-interactive` 모드로 실행하여, 인간의 확인 없이 자동 실행. 이 때문에 수정 범위 제한이 더욱 중요했습니다.

**검증**: Auto-Fix MR의 변경 파일이 이슈 파일과 정확히 일치하는지 CI에서 자동 검증. 불일치 시 MR에 경고 레이블 자동 부착.

</details>

---

### Q2. Dynamic Child Pipeline으로 병렬 처리를 구현한 아키텍처를 설명해주세요.
> **출제 의도**: CI/CD 파이프라인 고급 설계 역량
>
> **경력 연관**: 이그나이트 SonarQube Auto-Fix

<details>
<summary><b>✅ 답변</b></summary>

**배경 문제**: 단일 잡에서 모든 SonarQube 이슈를 순차 처리하면 GitLab 15분 타임아웃 초과.

**Dynamic Child Pipeline 구조**:

```
Parent Pipeline (스케줄 트리거, 매일 07:00)
├── Stage 1: SonarQube API에서 이슈 수집 (최대 10,000건)
├── Stage 2: 파일 기준으로 JIRA 티켓 그루핑
├── Stage 3: JIRA 티켓 수에 따라 child-pipeline.yml 동적 생성
└── Stage 4: trigger로 Child Pipeline 실행

Child Pipeline (동적 생성)
├── Job 1: JIRA-001 (파일 A, B의 이슈 수정 → MR 생성)
├── Job 2: JIRA-002 (파일 C의 이슈 수정 → MR 생성)
├── Job 3: JIRA-003 (파일 D, E, F의 이슈 수정 → MR 생성)
└── ... (JIRA 티켓 수만큼 병렬)
```

**핵심 설계**:
1. **동적 YAML 생성**: Python 스크립트가 JIRA 티켓 수에 따라 `.gitlab-ci-child.yml`을 동적으로 생성
2. **잡 격리**: 한 잡이 실패해도 다른 잡은 정상 완료. 실패한 잡만 재시도
3. **브랜치 전략**: 최신 `release/YYMMDD` 브랜치를 자동 감지하여 각 잡이 독립 브랜치 생성 → cherry-pick → MR

**성과**: 처리 속도 5배 향상, 동시 처리 건수 10배 증가 (FO/BO 병렬 + 티켓별 병렬).

</details>

---

### Q3. Claude Code 커스텀 스킬 7종의 설계 원칙과 효과를 설명해주세요.
> **출제 의도**: AI 코딩 도구 활용 및 팀 생산성 개선 역량
>
> **경력 연관**: 이그나이트 Claude Code 자동화

<details>
<summary><b>✅ 답변</b></summary>

**스킬 분류**:

| 카테고리 | 스킬 | 용도 |
|---------|------|------|
| 구현 | react-component-implementation | 팀 컨벤션에 맞는 React 컴포넌트 생성 |
| 구현 | tanstack-query-pattern | React Query 훅 패턴 자동 생성 |
| 구현 | form-validation-pattern | React Hook Form + Zod 폼 패턴 생성 |
| 협업 | review-checklist | 코드 리뷰 체크리스트 자동 생성 |
| 협업 | jira-ticket-implementation | JIRA 티켓 기반 구현 가이드 |
| 자동화 | gitlab-mr-draft | MR 설명 자동 생성 |
| 자동화 | playwright-automation | E2E 테스트 시나리오 생성 |

**설계 원칙**:
1. **팀 컨벤션 임베딩**: 스킬에 팀의 코드 컨벤션, 디렉토리 구조, 네이밍 규칙을 포함하여 AI가 팀 스타일에 맞는 코드를 생성
2. **단일 책임**: 하나의 스킬은 하나의 작업만 수행. 복잡한 작업은 스킬 조합으로 해결
3. **검증 가능한 출력**: 생성된 코드가 TypeScript 컴파일, ESLint, 테스트를 통과하는지 스킬 내에서 검증

**효과**: 
- 코드 리뷰 지적 사항 감소 (컨벤션이 스킬에 내장되어 있으므로)
- 신규 팀원 온보딩 시간 단축 (스킬을 통해 팀의 패턴을 자연스럽게 학습)

</details>

---

### Q4. 역할 분리 Agent 3종을 어떻게 설계하고, 작업 중첩을 어떻게 방지했나요?
> **출제 의도**: AI Agent 시스템 설계 역량
>
> **경력 연관**: 이그나이트 Claude Code 자동화

<details>
<summary><b>✅ 답변</b></summary>

**Agent 3종 역할**:

| Agent | 역할 | 허용 작업 | 금지 작업 |
|-------|------|----------|----------|
| frontend-implementer | 구현 | 컴포넌트/훅/유틸 생성, 테스트 작성 | 기존 코드 리팩토링, 아키텍처 변경 |
| reviewer | 검증 | 코드 리뷰, 버그 발견, 개선 제안 | 직접 코드 수정 |
| refactor-agent | 개선 | 코드 리팩토링, 성능 개선, 패턴 적용 | 새 기능 추가, 비즈니스 로직 변경 |

**중첩 방지 전략**:
1. **책임 경계 명문화**: 각 Agent의 AGENTS.md에 역할, 허용 작업, 금지 작업을 명확히 정의
2. **입력/출력 분리**: implementer가 생성한 코드 → reviewer가 검토 → refactor-agent가 개선. 순차적 파이프라인
3. **파일 소유권**: implementer는 새 파일만 생성, refactor-agent는 기존 파일만 수정. reviewer는 코멘트만 생성

**실무 워크플로우**:
```
JIRA 티켓 수신
→ implementer: 티켓 기반 구현
→ reviewer: 구현 결과 리뷰 (코멘트 생성)
→ implementer: 리뷰 반영
→ refactor-agent: 코드 품질 개선 (선택적)
→ MR 생성
```

**성과**: JIRA 티켓 수신 → 코드 구현 → 리뷰 → MR 생성 전 단계 자동화로 개발자 투입 공수 97% 절감.

</details>

---

### Q5. Playwright MCP 연동으로 E2E 테스트를 자동화한 경험을 설명해주세요.
> **출제 의도**: MCP(Model Context Protocol) 활용 역량
>
> **경력 연관**: 이그나이트 Claude Code 자동화

<details>
<summary><b>✅ 답변</b></summary>

**구성**:
- **Playwright MCP**: Claude Code가 브라우저를 직접 제어하여 E2E 테스트 실행
- **GitLab MCP**: Claude Code 내에서 MR 생성, 조회

**하이브리드 인증 플로우**:
```
1. Playwright가 로그인 페이지 접근
2. ID/PWD 자동 입력 (환경변수에서 읽기)
3. OTP 입력 화면에서 대기 → 개발자가 수동 입력
4. 인증 완료 후 인증 상태를 storageState로 저장
5. 이후 테스트에서 저장된 인증 상태를 재사용 → 반복 로그인 제거
```

**E2E 테스트 시나리오 자동화**:
1. Claude Code가 JIRA 티켓의 요구사항을 분석
2. playwright-automation 스킬을 사용하여 테스트 시나리오 생성
3. Playwright MCP로 브라우저에서 시나리오 실행 및 검증
4. GitLab CI/CD 파이프라인에 통합하여 PR 시 E2E 검증 자동 실행

**storageState 활용**: 브라우저의 쿠키와 localStorage를 JSON 파일로 저장. 테스트 실행 시 이 파일을 로드하여 로그인 과정을 건너뜁니다. OTP가 필요한 서비스에서 매번 수동 인증 없이 테스트를 실행할 수 있게 된 핵심 기법입니다.

</details>

---

### Q6. JIRA label 기반으로 별도 DB 없이 처리 이력을 추적한 방법을 설명해주세요.
> **출제 의도**: 경량 상태 관리 설계
>
> **경력 연관**: 이그나이트 SonarQube Auto-Fix

<details>
<summary><b>✅ 답변</b></summary>

**문제**: SonarQube 이슈를 매일 수집하여 JIRA 티켓을 생성하는데, 이미 처리한 이슈에 대해 중복 티켓이 생성되면 안 됨. 별도 DB를 구축하면 인프라 관리 부담 증가.

**해결 - JIRA Label 활용**:

```
1. SonarQube 이슈 key: "squid:S1234_FileA.tsx_L42"
2. JIRA 티켓 생성 시 label에 이슈 key를 포함: "sonar-squid:S1234_FileA.tsx_L42"
3. 다음 실행 시 JIRA API로 해당 label이 있는 티켓이 존재하는지 확인
4. 존재하면 건너뛰기, 없으면 새 티켓 생성
```

**장점**:
- 별도 DB/파일 없이 JIRA 자체가 상태 저장소 역할
- JIRA UI에서 label 필터로 Auto-Fix 관련 티켓만 조회 가능
- 티켓이 삭제되면 자동으로 다음 실행에서 재생성 (자가 복구)

**한계와 보완**:
- JIRA API 호출 횟수 증가 (이슈당 1회 조회). 500건씩 페이지네이션으로 최적화
- label 문자열 길이 제한이 있어, 이슈 key를 해시로 축약하는 방안도 고려했으나 가독성을 위해 원본 유지

</details>
