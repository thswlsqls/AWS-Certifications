---
service: AWS Organizations · Control Tower · Billing
exams: [saa, cloudops]
domains:
  - saa/cost-optimized-architectures
  - cloudops/security-compliance
status: learning
confidence: 1
tags:
  - service
  - saa/management
  - cloudops/management
---

# Organizations & 계정 관리

> 멀티 계정 전략 — OU, SCP, 통합 결제, Health Dashboard, 비용 도구.

## 개요

여러 AWS 계정을 한 묶음으로 관리하는 서비스. 회사가 커지면 계정을 하나만 쓰지 않고 팀·환경·프로젝트별로 계정을 나누는데(멀티 계정 전략), 그 계정들을 하나의 **Organization** 아래 모아 결제를 합치고, 계정 그룹(OU)마다 권한 상한을 거는 게 핵심이다.

- **Management Account**(관리 계정, 옛 master) 하나 + 나머지 **Member Account**들. 한 멤버 계정은 **딱 하나의 Organization**에만 속할 수 있다.
- 계정을 트리로 묶는 단위가 **OU(Organizational Unit)**. 루트 아래 OU를 중첩할 수 있다.
- 결제는 **통합 결제(Consolidated Billing)**로 한 결제 수단에 합치고, 권한은 **SCP(Service Control Policy)**로 OU/계정 단위에 상한을 건다.

## 설계 관점 (SAA) #saa

![[AWS Certified Solutions Architect Slides v47.pdf#page=623]]

### AWS Organizations — 멀티 계정의 뼈대

글로벌 서비스. 관리 계정에서 멤버 계정들을 만들고 묶는다. SAA에서 이 서비스가 비용·보안 두 축으로 나오므로 "통합 결제로 뭘 아끼나"와 "SCP가 어떻게 동작하나" 두 가지를 분리해서 기억한다.

**통합 결제(Consolidated Billing) — 비용 도메인의 핵심**

- 모든 계정 사용량을 합쳐 **하나의 결제 수단**으로 낸다.
- **볼륨 할인 합산**: EC2·S3 같은 사용량 기반 할인 구간을 계정별이 아니라 **조직 전체 합계**로 계산해 더 싼 구간에 들어간다.
- **예약 인스턴스(RI)·Savings Plans 할인을 계정 간 공유**: 한 계정이 산 RI를 안 쓰면 같은 조직의 다른 계정이 그 할인을 받는다. → "여러 계정인데 RI 비용을 아끼고 싶다" 시나리오의 답.
- 계정 생성을 **API로 자동화**할 수 있다.

**멀티 계정 전략의 이점 (왜 계정을 나누나)**

- 계정 하나에 VPC 여러 개를 두는 것보다, **계정 자체를 격리 경계**로 쓰면 권한·결제·장애 범위가 깔끔히 나뉜다.
- 결제용 **태그 표준**을 세우고, 모든 계정의 [[CloudWatch & CloudTrail & Config|CloudTrail]] 로그를 **중앙 S3 계정**으로, CloudWatch Logs를 **중앙 로깅 계정**으로 모은다.
- 관리 목적의 **크로스 계정 역할(Cross Account Role)**을 만들어 관리 계정에서 멤버 계정을 운영한다.

### OU 설계 — 무엇을 기준으로 계정을 묶나

같은 정책을 한 번에 걸 단위로 묶는 게 목적이다. 슬라이드가 든 세 가지 기준:

- **Business Unit**(부서별: Sales·Retail·Finance OU)
- **Environmental Lifecycle**(환경별: Prod·Dev·Test OU)
- **Project-Based**(프로젝트별: Project 1·2·3 OU)

정답이 하나는 아니고, "어느 묶음에 같은 SCP를 걸고 싶은가"로 고르면 된다.

### SCP (Service Control Policy) — 계정/OU 권한의 상한

OU나 계정에 붙여 그 안의 **사용자·역할이 쓸 수 있는 서비스를 제한**하는 정책. 권한을 더 주는 게 아니라 **상한을 깎는** 용도다.

- **관리 계정에는 SCP가 적용되지 않는다.** 관리 계정은 항상 풀 권한이라, 실수로 자기 발등을 찍지 않게 막아 둔 것이자 시험 단골 함정.
- IAM처럼 **기본은 전부 거부(deny by default)**. 어떤 동작이 허용되려면 **루트 → 대상 계정까지 경로상의 모든 OU에 명시적 Allow**가 있어야 한다. 중간 OU 하나라도 그 서비스를 허용하지 않으면 막힌다. 그래서 보통 루트에 `FullAWSAccess`를 깔고 하위에서 `Deny`로 빼는 식으로 쓴다.
- **명시적 Deny가 최우선**: 상위 OU에서 `Deny S3`를 걸면 그 아래 계정은 다른 데서 아무리 Allow를 받아도 S3를 못 쓴다.
- **두 가지 전략**:
  - **Blocklist**(블록리스트): `FullAWSAccess`로 다 열어 두고 특정 서비스만 `Deny`로 차단. (예: DynamoDB만 막기)
  - **Allowlist**(허용리스트): 기본은 다 막고 필요한 서비스(`ec2:*`, `cloudwatch:*` 등)만 `Allow`. 더 엄격하지만 관리가 번거롭다.
- **SCP vs Permission Boundary**: SCP는 **계정/OU 전체**의 상한, [[고급 ID 관리|Permission Boundary]]는 **사용자·역할 하나**의 상한. 둘 다 "권한을 깎기만 한다"는 점은 같고, 실제 권한 = 신원 정책 ∩ SCP ∩ Permission Boundary로 교집합이다. ([[고급 ID 관리]]의 IAM Policy Evaluation Logic 참고.)

### Tag Policies — 태그를 강제해 비용·접근을 통제

조직 전체 리소스의 **태그를 표준화**하는 정책.

- 태그 **키와 허용 값**을 정의한다(예: `CostCenter` 키에 `100`·`200`만 허용).
- **Cost Allocation Tags**(비용 배분 태그)와 **ABAC**(속성 기반 접근 제어)의 바탕이 된다.
- 지정한 서비스에서 **규칙에 어긋나는 태깅 작업을 차단**한다(태그가 아예 없는 리소스에는 효과 없음).
- 규정 위반 리소스 **리포트**를 뽑고, **EventBridge로 비규격 태그를 모니터링**한다.

## 운영 관점 (CloudOps) #cloudops

![[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=434]]

%% 이 시험의 관점에서 배운 내용을 정리합니다. %%

## 시험 함정

- SCP는 **관리 계정에는 적용되지 않는다**(관리 계정은 항상 풀 권한). #exam/trap/organizations
- SCP는 기본 거부라 **루트→대상 계정 경로의 모든 OU에 명시적 Allow**가 있어야 동작한다. 중간 OU에서 빠지면 막힌다. #exam/trap/organizations
- 여러 계정에서 RI·Savings Plans 비용을 아끼는 답 → **통합 결제(할인 계정 간 공유)**. #exam/trap/cost-optimization
- SCP = 계정/OU 전체 상한, Permission Boundary = 사용자 하나 상한. 범위를 바꿔 묻는다. #exam/trap/organizations
- "조직 멤버 계정만 S3 버킷 접근 허용" → 리소스 정책의 **`aws:PrincipalOrgID`** 조건(이건 [[고급 ID 관리]]에 정리). #exam/trap/organizations

## 관련 노트

[[IAM]] · [[고급 ID 관리]] · [[CloudWatch & CloudTrail & Config]]
