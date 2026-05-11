# DevOps & CI/CD 질문

### Q1. GitLab CI/CD로 Blue-Green 배포를 구축하여 배포 사이클을 2주→3일로 단축한 경험을 설명해주세요.
> **출제 의도**: CI/CD 파이프라인 설계 및 배포 전략
>
> **경력 연관**: 이그나이트 Admin CMS

<details>
<summary><b>✅ 답변</b></summary>

**기존 문제**: 수동 배포 + 단일 환경으로 배포 시 다운타임 발생. 롤백에 30분+ 소요되어 배포 빈도를 줄이게 됨 → 변경사항이 누적되어 한 번 배포에 리스크 증가.

**Blue-Green 배포 구조**:
```
Blue (현재 운영) ← 트래픽
Green (새 버전 대기)

1. Green 환경에 새 버전 배포
2. 헬스체크 통과 확인
3. 로드밸런서의 트래픽을 Green으로 전환
4. 문제 발생 시 즉시 Blue로 롤백 (5분 이내)
```

**GitLab CI/CD 파이프라인**:
```yaml
stages:
  - build       # Docker 이미지 빌드 + ECR 푸시
  - test        # 단위/통합 테스트 + 커버리지 80% 게이트
  - deploy-stg  # 스테이징 배포 + E2E 테스트
  - deploy-prod # 프로덕션 Blue-Green 배포
```

**태그 기반 배포**: `release/YYMMDD` 태그 푸시 시 자동으로 프로덕션 배포 트리거. 테스트 커버리지 80% 미만이면 배포 파이프라인 중단.

**결과**: 배포 사이클 2주 → 3일, 롤백 시간 30분 → 5분. 빈번한 소규모 배포로 리스크 감소.

</details>

**꼬리 질문**:
- Blue-Green vs Canary 배포의 선택 기준은?
- 테스트 커버리지 80% 기준의 근거는?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **선택 기준**: Blue-Green은 전체 트래픽을 한 번에 전환하므로 구현이 단순하고 롤백이 즉각적. Canary는 트래픽을 점진적으로 전환(1% → 5% → 25% → 100%)하여 리스크를 더 세밀하게 제어할 수 있지만 인프라 복잡도가 높음. 딜러포탈은 사용자 수가 상대적으로 적고(B2B), 빠른 롤백이 더 중요했기에 Blue-Green을 선택했습니다. B2C 대규모 서비스라면 Canary가 더 적합할 수 있습니다.

- **커버리지 80% 근거**: 100%는 비현실적이고(UI 이벤트 핸들러 등 테스트 비용 대비 효과가 낮은 코드 존재), 60% 미만은 핵심 로직도 누락될 수 있음. 80%는 업계에서 일반적으로 권장하는 수준이고, 비즈니스 크리티컬 로직(결제, 인증)은 별도로 100% 커버리지를 요구했습니다.

</details>

---

### Q2. PR 시 Lighthouse CI 기반 성능 회귀 방지 파이프라인을 어떻게 구축했나요?
> **출제 의도**: 자동화된 품질 게이트 구축 역량
>
> **경력 연관**: 마이리얼트립 통합숙소

<details>
<summary><b>✅ 답변</b></summary>

**구성**:
1. **GitHub Actions 워크플로우**: PR이 생성/업데이트될 때마다 자동 실행
2. **Lighthouse CI 실행**: 주요 페이지(홈, 상품 목록, 상품 상세)에 대해 Lighthouse 감사 실행
3. **임계값 체크**: Performance, Accessibility, Best Practices, SEO 점수가 설정된 임계값 미만이면 PR 블로킹
4. **결과 리포트**: PR 코멘트에 점수와 이전 빌드 대비 변화를 자동으로 게시

**임계값 설정 (lighthouserc.js)**:
```javascript
assertions: {
  'categories:performance': ['error', { minScore: 0.85 }],
  'categories:accessibility': ['error', { minScore: 0.9 }],
  'first-contentful-paint': ['warn', { maxNumericValue: 2000 }],
  'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
}
```

**실효성**: 이미지 최적화를 빠뜨리거나, 번들 사이즈가 크게 증가하는 PR을 머지 전에 자동 감지. 성능이 의도적으로 변경되는 경우(새 기능 추가 등)에는 PR 설명에 사유를 기재하고 임계값을 일시적으로 조정.

</details>

---

### Q3. Docker 기반 AWS ECS/Fargate 배포 환경을 어떻게 구축했나요?
> **출제 의도**: 컨테이너 기반 배포 인프라 이해
>
> **경력 연관**: 브랜디 커머스 / 마이리얼트립 통합숙소

<details>
<summary><b>✅ 답변</b></summary>

**전체 아키텍처**:
```
코드 푸시 → CI/CD → Docker 빌드 → ECR 이미지 푸시 → ECS Task Definition 업데이트 → Fargate 태스크 실행
```

**Docker 최적화**:
1. **멀티스테이지 빌드**: 빌드 스테이지와 실행 스테이지 분리. 빌드 의존성(devDependencies)이 최종 이미지에 포함되지 않아 이미지 사이즈 감소
2. **레이어 캐싱**: `package.json`과 `yarn.lock`을 먼저 COPY하여 의존성 레이어를 캐싱. 소스 코드 변경 시 의존성 재설치 없이 빌드
3. **.dockerignore**: `node_modules`, `.git`, 테스트 파일 등 불필요한 파일 제외

**ECS/Fargate 설정**:
- **Task Definition**: CPU/메모리 할당, 환경변수, 헬스체크 설정
- **Service**: 최소/최대 태스크 수, 오토스케일링 정책, 로드밸런서 연결
- **Fargate**: 서버 관리 없이 컨테이너 실행. 트래픽에 따라 자동 스케일

**브랜디에서의 경험**: PHP 모놀리스에서 Next.js 컨테이너로 전환하면서, 기존 EC2 기반 인프라에서 ECS/Fargate 기반으로 마이그레이션했습니다. Fargate를 선택한 이유는 서버 관리 오버헤드 제거와 프론트엔드 팀의 인프라 운영 부담 최소화였습니다.

</details>

---

### Q4. Webpack HMR 메모리 누수를 어떻게 진단하고 해결했나요?
> **출제 의도**: 빌드 도구 트러블슈팅 역량
>
> **경력 연관**: 마이리얼트립 Staynet

<details>
<summary><b>✅ 답변</b></summary>

**증상**: 개발 서버를 30분 이상 사용하면 힙 메모리가 2GB+까지 증가하여 브라우저 탭이 크래시.

**진단 과정**:
1. Chrome DevTools Memory 탭에서 힙 스냅샷을 시간별로 3회 촬영
2. 스냅샷 비교(Comparison view)에서 HMR 업데이트 시마다 이전 번들이 해제되지 않고 계속 누적되는 것을 확인
3. Webpack 설정에서 `output.filename`에 `[hash]` 옵션이 개발 환경에서도 적용된 것이 원인

**원인**: `[hash]`로 파일명이 매번 변경되면, Webpack HMR이 이전 모듈을 정리하지 못하고 새 모듈을 계속 추가. 프로덕션에서는 캐시 무효화를 위해 필요하지만, 개발 환경에서는 불필요.

**해결**:
```javascript
output: {
  filename: isDev ? '[name].js' : '[name].[contenthash].js',
}
```

개발 환경에서 `[hash]` 제거 후 메모리 누수 해소. HMR이 정상적으로 이전 모듈을 교체하게 됨.

**교훈**: 프로덕션 최적화 설정(해시, 압축 등)을 개발 환경에 그대로 적용하면 DX 문제가 발생할 수 있으므로, 환경별 설정을 명확히 분리해야 합니다.

</details>

---

### Q5. CloudFront 캐시 전략을 어떻게 수립하여 정적 자원 응답을 210ms→85ms로 개선했나요?
> **출제 의도**: CDN 캐시 전략 실무 이해
>
> **경력 연관**: 이그나이트 딜러포탈

<details>
<summary><b>✅ 답변</b></summary>

**기존 문제**: 모든 정적 자원에 동일한 캐시 정책 적용. 자주 변경되는 HTML과 거의 변경되지 않는 JS/CSS/이미지에 같은 TTL 적용되어 캐시 효율이 낮음.

**자원 유형별 캐시 전략**:

| 자원 | Cache-Control | 이유 |
|------|--------------|------|
| HTML | `no-cache, must-revalidate` | 항상 최신 버전 체크 (304 응답으로 빠른 검증) |
| JS/CSS (`[contenthash]`) | `public, max-age=31536000, immutable` | 파일명에 해시 포함으로 내용이 바뀌면 URL도 변경 → 1년 캐시 |
| 이미지 | `public, max-age=86400` | 24시간 캐시, CDN에서 직접 서빙 |
| API 응답 | `no-store` | 캐시 금지, 항상 원본 서버에서 가져옴 |

**CloudFront Behavior 설정**:
- `/_next/static/*` 패턴에 1년 캐시 + `immutable`
- `/*.html` 패턴에 no-cache
- Origin Shield 활성화로 Origin 서버 부하 감소

**측정**: CloudFront Metrics에서 캐시 히트율 60% → 92%로 개선, 평균 응답 시간 210ms → 85ms.

</details>

**꼬리 질문**:
- `immutable` 헤더의 의미와 효과는?
- 캐시 무효화는 어떻게?

<details>
<summary><b>✅ 꼬리 답변</b></summary>

- **immutable**: 브라우저에게 "이 리소스는 절대 변경되지 않으니, 재검증(revalidation)도 하지 마라"고 알려주는 지시어입니다. `max-age` 내에서도 새로고침 시 조건부 요청(If-Modified-Since)을 보내는데, `immutable`은 이것도 방지합니다. `[contenthash]`로 파일명이 내용 기반으로 결정되므로, 내용이 같으면 URL이 같고 다르면 URL이 달라져 `immutable`을 안전하게 사용할 수 있습니다.

- **캐시 무효화**: 일반적으로 캐시 무효화가 필요 없습니다. 빌드 시 새로운 해시가 생성되면 새 URL이므로 이전 캐시와 무관합니다. HTML은 no-cache라 항상 최신 버전을 가져오고, 새 HTML이 새 해시의 JS/CSS를 참조합니다. 긴급 시에는 CloudFront Invalidation API로 특정 경로의 캐시를 강제 삭제할 수 있지만, 비용이 발생하므로 최후의 수단으로만 사용합니다.

</details>
