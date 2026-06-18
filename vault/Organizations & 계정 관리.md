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

회사가 커지면 계정을 하나로 쓰지 않고 여러 개로 나눈다. 이 노트는 그 **멀티 계정 환경을 묶고·통제하고·비용을 보는** 도구 모음이다.

- **AWS Organizations** — 여러 계정을 한 조직으로 묶는다. 결제를 하나로 합치고(Consolidated Billing), OU로 계정을 그룹 짓고, SCP로 각 계정이 할 수 있는 일의 상한선을 건다.
- **AWS Control Tower** — Organizations 위에서 멀티 계정 환경을 베스트 프랙티스대로 자동 셋업·통제한다. OU·SCP를 알아서 깔아 준다.
- **AWS Service Catalog** — 관리자가 미리 승인한 제품(CloudFormation 템플릿)만 사용자가 self-service로 띄우게 해, 제멋대로 비표준 리소스를 만드는 걸 막는다.
- **AWS Health Dashboard** — AWS 쪽 장애·예정 작업이 내 계정·리소스에 영향을 주는지 알려준다.
- **비용 도구** — Billing Alarms·Cost Explorer·Budgets·Cost and Usage Report·Compute Optimizer로 비용을 보고·예측하고 줄인다.

핵심 구분 하나: **Organizations는 계정을 묶고 결제·정책을 관리, Control Tower는 그걸 자동·표준으로 깔아 주는 상위 도구**다.

## 설계 관점 (SAA) #saa

![[AWS Certified Solutions Architect Slides v47.pdf#page=622]]

%% 이 시험의 관점에서 배운 내용을 정리합니다. %%

## 운영 관점 (CloudOps) #cloudops

![[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=434]]

CloudOps에서는 멀티 계정을 실제로 운영하는 관점 — 조직 정책(SCP)·중앙 거버넌스·가용성 알림·비용 통제 — 을 묻는다. 보안 16% + 배포 자동화·신뢰성에 걸쳐 있다.

### AWS Health Dashboard

AWS의 상태를 보는 두 화면을 구분해야 한다.

- **Service History** (옛 Service Health Dashboard) — 전체 리전·전체 서비스의 일반적인 상태와 과거 이력. RSS 구독 가능. "AWS 전체가 지금 괜찮은가".
- **Your Account** (옛 Personal Health Dashboard, PHD) — **내 계정·내 리소스에 직접 영향**을 주는 이벤트만 보여준다. 알림·교정 가이드·사전 통지(예정 작업)를 준다. **글로벌 서비스**이고 **Organization 전체 데이터를 한데 모아** 볼 수 있다. "내 리소스가 받는 영향".
- **Health Event Notifications** — EventBridge로 Health 이벤트에 자동 반응한다. **Account 이벤트**(내 계정 리소스 영향)와 **Public 이벤트**(특정 리전의 서비스 가용성) 둘 다 잡아 Lambda·SNS·SQS·KDS로 보낸다. 예: 노출된 IAM 액세스 키를 Lambda로 자동 삭제, retirement 예정 EC2를 자동 재시작.

### AWS Organizations — 구조와 결제

- **management account(= payer account)**는 조직의 주인. 나머지는 **member account**이고 **한 조직에만** 속한다. Root OU 아래로 OU를 중첩한다(부서별·환경별·프로젝트별).
- **Consolidated Billing** — 모든 계정을 결제상 한 계정처럼 본다. 사용량을 합산해 볼륨 할인(EC2·S3 등)을 받고, **Reserved Instance·Savings Plans 할인을 계정끼리 공유**한다.
  - RI/SP 할인 공유는 **payer가 계정별로 끄고 켤 수 있다.** 두 계정이 할인을 나누려면 **양쪽 모두 공유가 켜져 있어야** 한다.

### Service Control Policies (SCP) — 가장 출제 잦음

- OU나 계정에 붙여 그 안의 IAM 사용자·역할이 할 수 있는 일의 **상한선(가드레일)**을 정한다. 권한을 주는 게 아니라 **최대 한도를 제한**한다.
- **management account에는 SCP가 적용되지 않는다** — 항상 전체 관리 권한. (그래서 운영 워크로드를 management account에 두지 말라는 베스트 프랙티스가 나온다.)
- **기본은 전부 deny.** IAM처럼, **Root → 대상 계정까지 경로상의 모든 OU에 명시적 allow**가 있어야 그 동작이 허용된다. 새 OU·계정엔 기본으로 `FullAWSAccess`가 붙는다.
- **Blocklist 전략**(FullAWSAccess + 특정 동작만 Deny)과 **Allowlist 전략**(전체 Deny 후 필요한 것만 Allow)이 있다. 자주 나오는 예: 특정 리전 외 전부 거부(`aws:RequestedRegion`), 특정 태그가 없으면 `ec2:RunInstances` 거부.

### 조직 차원의 다른 통제 수단

- **`aws:PrincipalOrgID` 조건키** — 리소스 기반 정책(예: S3 버킷 정책)에 걸어, **조직에 속한 계정의 principal만** 접근하게 한다. 계정 ID를 일일이 나열할 필요가 없다.
- **Tag Policies** — 조직 전체에서 태그 키·허용 값을 표준화한다. `enforced_for`로 지정 서비스의 비준수 태깅을 막고(태그 없는 리소스엔 효과 없음), 비준수 리소스 리포트를 만든다. Cost Allocation Tags·ABAC와 함께 쓴다.
- **멀티 계정 베스트 프랙티스** — CloudTrail을 전 계정에 켜 중앙 S3 계정으로 모으고, CloudWatch Logs도 중앙 로깅 계정으로 보내고, 관리용 Cross-Account Role을 둔다.

### Control Tower와 Service Catalog

- **Control Tower** — 멀티 계정 환경을 몇 번 클릭으로 베스트 프랙티스대로 셋업하고, **guardrail**로 정책을 계속 관리한다. 위반을 탐지·교정하고 대시보드로 준수 상태를 본다. **내부적으로 Organizations를 자동 구성**해 OU와 SCP를 깐다. "표준에 맞는 멀티 계정을 빠르게 세팅·통제" → Control Tower.
- **Service Catalog** — 관리자가 **CloudFormation 템플릿을 Product로 등록**하고 Portfolio로 묶어 IAM 권한으로 접근을 통제한다. 사용자는 **인가된 제품만 self-service로 런치**해, 항상 준수·태깅된 스택을 얻는다. Portfolio는 다른 계정·조직과 공유 가능(원본과 동기화되는 reference 공유 vs 복사본 배포). **TagOptions Library**로 런치되는 제품의 태그를 표준화한다.

### 비용·최적화 도구 (CloudOps 비용 관점)

- **Billing Alarms** — 청구 지표는 **us-east-1에만 저장**된다(다른 리전에서 못 만든다). 전 세계 합산 **실제** 비용 기준이며 예상 비용이 아니다.
- **Cost Explorer** — 비용·사용량을 시각화. 전체·월·시간·리소스 단위로 분석하고, 최적 Savings Plan을 추천하며, **과거 사용량 기반으로 최대 18개월 예측**한다.
- **AWS Budgets** — 비용이 예산을 넘으면 알림. 4종(Usage·Cost·Reservation·Savings Plans), 예산당 **SNS 알림 5개**, Cost Explorer와 같은 필터. 처음 2개 무료.
- **Cost Allocation Tags** — 비용을 세분해 추적. **`aws:` 접두사는 AWS 자동 생성 태그**, **`user:` 접두사는 사용자 정의 태그**.
- **Cost and Usage Report (CUR)** — **가장 상세한** 비용·사용 데이터. S3로 일일 export하고 Athena·Redshift·QuickSight로 분석한다. "가장 깊은 비용 분석" → CUR.
- **Compute Optimizer** — ML로 CloudWatch 지표를 분석해 right-sizing을 추천(EC2·ASG·EBS·Lambda·Fargate ECS·Aurora/RDS). 최대 25% 절감. 보려면 `ComputeOptimizerReadOnlyAccess` 권한. **새 인스턴스가 안 보이면 지표가 부족한 것 — 30시간 이상 켜 두면** 추천이 생긴다.
- **Billing Conductor** — 청구서를 **보여주고 배분하는 방식만** 커스터마이즈(pro forma 청구서, 부서·고객별 분할, chargeback/showback). **실제 AWS 청구액은 바뀌지 않는다.**

## 시험 함정

- **SCP는 management account에 적용되지 않는다** — 항상 전체 권한. 운영 워크로드를 management account에 두지 말 것. #exam/trap/organizations
- SCP는 **기본 전부 deny**, Root→대상 계정 경로의 **모든 OU에 명시적 allow**가 있어야 허용(IAM과 동일). #exam/trap/organizations
- RI·Savings Plans 할인 공유는 **양쪽 계정 모두 공유 ON**이어야 한다. payer가 계정별로 끌 수 있음. #exam/trap/cost-optimization
- Health Dashboard: **Service History**(AWS 전체 상태) vs **Your Account/PHD**(내 계정·리소스 영향). 혼동 주의. #exam/trap/monitoring
- 조직 내 계정만 리소스 접근 허용 → 리소스 정책에 **`aws:PrincipalOrgID`** 조건키(계정 ID 나열 불필요). #exam/trap/security
- **Billing 지표는 us-east-1에만 저장** — 청구 알림은 거기서 만든다. 전 세계 합산 실제 비용. #exam/trap/cost-optimization
- Cost Explorer는 **최대 18개월 비용 예측**. 가장 상세한 비용 데이터는 **CUR**(S3+Athena). #exam/trap/cost-optimization
- Compute Optimizer에 새 EC2가 안 보이면 지표 부족 — **30시간 이상 가동** 후 추천 생성. 권한은 `ComputeOptimizerReadOnlyAccess`. #exam/trap/cost-optimization
- Service Catalog의 Product는 **CloudFormation 템플릿** — 관리자 승인 제품만 self-service 런치. #exam/trap/organizations
- Control Tower는 **Organizations 위에서** OU·SCP를 자동 구성하는 상위 도구. "표준 멀티 계정 빠른 셋업" → Control Tower. #exam/trap/organizations
- Billing Conductor는 **표시·배분만** 바꾼다 — 실제 청구액은 그대로(pro forma). #exam/trap/cost-optimization
- Cost Allocation Tags: `aws:`(AWS 생성) vs `user:`(사용자 정의) 접두사. #exam/trap/cost-optimization

## 관련 노트

[[IAM]] · [[고급 ID 관리]] · [[CloudWatch & CloudTrail & Config]] · [[CloudFormation]] · [[기타 서비스]]
