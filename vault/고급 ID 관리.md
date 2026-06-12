---
service: STS · IAM Identity Center · Directory Service
exams: [saa, dva]
domains:
  - saa/secure-architectures
  - dva/security
status: learning
confidence: 2
tags:
  - service
  - saa/security
  - dva/security
---

# 고급 ID 관리

> IAM 기본을 넘어서는 신원 관리 — 임시 자격 증명(STS), 고급 정책 기법(조건·경계·평가 로직), 멀티 계정 SSO(IAM Identity Center), 온프레미스 연동(Directory Service).

## 개요

- **STS (Security Token Service)**: AWS 리소스에 대한 **제한적이고 임시적인** 접근 권한을 발급합니다. 역할 전환(AssumeRole)의 기반.
- **권한 평가 로직**: ① 명시적 **Deny**가 있으면 즉시 거부 → ② **Allow**가 있으면 허용 → ③ 둘 다 없으면 **기본 거부**. 모든 정책 문제의 출발점.
- **IAM Identity Center** (구 AWS Single Sign-On): 조직의 모든 AWS 계정 + 비즈니스 앱(Salesforce 등)에 대한 **단일 로그인(SSO)**.
- **Directory Service**: Microsoft Active Directory를 AWS에서 쓰는 세 가지 방식 (Managed AD / AD Connector / Simple AD).
- AWS Organizations·SCP·Control Tower도 강의에서는 같은 섹션에 나오지만, 정리는 [[Organizations & 계정 관리]]에서 합니다.

## 설계 관점 (SAA) #saa

![[AWS Certified Solutions Architect Slides v47.pdf#page=622]]

### IAM 조건 (Conditions) — p.630~633

| 조건 키 | 용도 |
| --- | --- |
| `aws:SourceIp` | API 호출을 보내는 **클라이언트 IP** 제한 |
| `aws:RequestedRegion` | API 호출 **대상 리전** 제한 |
| `ec2:ResourceTag` | 리소스에 붙은 **태그** 기준 제한 |
| `aws:MultiFactorAuthPresent` | **MFA 강제** |
| `aws:PrincipalOrgID` | **리소스 기반 정책**에서 특정 Organization 소속 계정만 허용 (예: S3 버킷을 조직 멤버에게만 개방) |

- S3의 권한 수준 구분: `s3:ListBucket`은 **버킷 수준**(`arn:aws:s3:::test`), `s3:GetObject`/`PutObject`/`DeleteObject`는 **객체 수준**(`arn:aws:s3:::test/*`) — ARN에 `/*`가 붙는지로 구분.

### IAM 역할 vs 리소스 기반 정책 — p.634~636

- 크로스 계정 접근 방법 두 가지: ① 리소스 기반 정책 부착(예: S3 버킷 정책) ② 역할을 프록시로 사용(AssumeRole).
- **핵심 차이**: 역할을 맡으면(assume) **원래 권한을 포기하고** 역할의 권한만 갖습니다. 리소스 기반 정책은 **원래 권한을 유지**한 채 접근합니다.
  - 예: 계정 A의 사용자가 계정 A의 DynamoDB를 스캔해서 계정 B의 S3에 덤프 → 원래 권한이 필요하므로 **리소스 기반 정책**이 답.
- 리소스 기반 정책 지원: S3 버킷, SNS 토픽, SQS 큐 등.
- **EventBridge 보안**: 규칙이 실행되면 대상에 대한 권한이 필요 — Lambda·SNS·SQS·S3·API Gateway는 **리소스 기반 정책**, EC2 Auto Scaling·Systems Manager Run Command·ECS Task는 **IAM 역할** 사용.

### IAM 권한 경계 (Permission Boundaries) — p.637~638

- **사용자와 역할에만** 지원 (그룹에는 불가).
- 관리형 정책으로 IAM 엔터티가 가질 수 있는 **최대 권한의 상한**을 설정. 실제 권한 = IAM 정책 ∩ 권한 경계.
- 사용 사례: 개발자가 스스로 정책을 관리하게 하되 **권한 상승(privilege escalation)을 차단**, 비관리자에게 IAM 사용자 생성 위임.
- SCP가 **계정 전체**를 제한한다면, 권한 경계는 **특정 사용자 하나**를 제한할 때 유용. Organizations SCP와 조합 가능.

### IAM Identity Center — p.641~645

- 한 번의 로그인으로 접근: Organization 내 **모든 AWS 계정**, 비즈니스 클라우드 앱(Salesforce, Box, Microsoft 365), SAML 2.0 앱, EC2 Windows 인스턴스.
- 자격 증명 공급자(IdP): **내장 Identity Store** 또는 서드파티(Active Directory, OneLogin, Okta…).
- **Permission Set**: 하나 이상의 IAM 정책 묶음. 사용자·그룹에 할당해 계정별 접근을 정의 (예: Developers 그룹 → Dev 계정엔 FullAccess, Prod 계정엔 ReadOnlyAccess).
- **ABAC (속성 기반 접근 제어)**: Identity Store에 저장된 사용자 속성(cost center, 직함, 지역 등)으로 세분화된 권한. 권한은 한 번 정의하고 **속성만 바꿔서** 접근을 조정.

### Active Directory & Directory Service — p.646~648

| 옵션 | 특징 | 온프레미스 AD 연결 |
| --- | --- | --- |
| **AWS Managed Microsoft AD** | AWS 안에 자체 AD 생성, 사용자를 AWS에서 관리, MFA 지원 | **양방향 신뢰(trust)** 연결 가능 |
| **AD Connector** | 온프레미스 AD로 리디렉션하는 **게이트웨이(프록시)**, MFA 지원 | 사용자는 온프레미스에서 관리 |
| **Simple AD** | AWS의 AD 호환 관리형 디렉터리 | 온프레미스 AD와 **연결 불가** |

- AD 기초 용어: 객체(사용자·컴퓨터·프린터·보안 그룹)를 **트리**로 조직, 트리의 묶음이 **포리스트(forest)**.
- Identity Center 연동: AWS Managed Microsoft AD는 **즉시 연동**, 자체 관리 디렉터리는 양방향 신뢰 또는 AD Connector 경유.

> Organizations(p.623~629) · Control Tower(p.649~650)는 → [[Organizations & 계정 관리]]

## 개발 관점 (DVA) #dva

![[AWS Certified Developer Slides v44.pdf#page=810]]

### STS API 정리 — p.811~814

임시 자격 증명은 **15분 ~ 1시간** 유효. 발급 결과물: Access ID + Secret Key + Session Token + 만료 시각.

| API | 용도 |
| --- | --- |
| `AssumeRole` | 같은 계정 또는 **크로스 계정**의 역할 맡기 |
| `AssumeRoleWithSAML` | **SAML**로 로그인한 사용자에게 자격 증명 발급 |
| `AssumeRoleWithWebIdentity` | 웹 IdP(Google, Facebook, OIDC) 로그인 사용자용 — **AWS는 비권장, [[Cognito]] Identity Pools 사용 권장** |
| `GetSessionToken` | **MFA**용 임시 자격 증명 (사용자 또는 루트 계정) |
| `GetFederationToken` | 연동(federated) 사용자용 임시 자격 증명 |
| `GetCallerIdentity` | 현재 API 호출에 사용된 IAM 사용자/역할 정보 조회 |
| `DecodeAuthorizationMessage` | API 거부 시 **오류 메시지 디코딩** |

- 역할 맡기 절차: ① 역할 정의 ② 어떤 보안 주체(principal)가 접근 가능한지 정의 ③ STS `AssumeRole`로 자격 증명 획득.
- **STS + MFA**: `GetSessionToken` 사용 + IAM 정책에 `aws:MultiFactorAuthPresent:true` 조건.

### IAM 베스트 프랙티스 — p.815~817

- 루트 자격 증명은 절대 사용 금지, 루트 계정에 MFA 활성화.
- **최소 권한** 부여 — 서비스에 `*` 접근을 주는 정책 금지.
- CloudTrail로 사용자 API 호출(특히 **Denied**) 모니터링.
- IAM 키를 개인 PC·온프레미스 서버 외의 머신에 **절대 저장 금지** — 온프레미스 서버는 STS로 임시 자격 증명을 받는 것이 모범 사례.
- **서비스마다 전용 역할**: EC2, Lambda, ECS Task(`ECS_ENABLE_TASK_IAM_ROLE=true`), CodeBuild 각각 자체 역할. 애플리케이션/함수당 역할 1개 — **역할 재사용 금지**.

### 권한 평가 로직 — p.818~823

1. 명시적 **DENY** 있음 → 즉시 **거부**
2. **ALLOW** 있음 → **허용**
3. 그 외 → **거부** (기본값)

IAM 정책과 S3 버킷 정책은 **합집합(union)으로 평가**됩니다:

| IAM 역할 (EC2에 부착) | S3 버킷 정책 | 결과 |
| --- | --- | --- |
| RW 허용 | 없음 | ✅ 읽기/쓰기 가능 |
| RW 허용 | 해당 역할에 **명시적 Deny** | ❌ 불가 (Deny 우선) |
| 권한 없음 | 해당 역할에 명시적 RW Allow | ✅ 가능 (합집합) |
| **명시적 Deny** | 명시적 RW Allow | ❌ 불가 (Deny 우선) |

### 동적 정책 & 정책 유형 — p.824~826

- 사용자별 S3 홈 폴더(`/home/<user>`) 부여: 사용자마다 정책을 만들면 확장 불가 → 정책 변수 **`${aws:username}`** 하나로 해결.
- 정책 3유형:
  - **AWS 관리형**: AWS가 유지보수, 신규 서비스/API 반영. 파워 유저·관리자용.
  - **고객 관리형**: **모범 사례**. 재사용 가능, 버전 관리 + 롤백, 중앙 변경 관리.
  - **인라인**: 정책과 보안 주체가 1:1. 주체를 삭제하면 정책도 삭제됨.

### iam:PassRole — p.827~829

- 많은 서비스 설정 시 IAM 역할을 **서비스에 넘겨주는**(pass) 작업이 필요 (설정 시 1회). 이후 서비스가 그 역할을 맡아(assume) 동작.
- 예: EC2 인스턴스, Lambda 함수, ECS Task, CodePipeline에 역할 전달.
- 필요한 권한: **`iam:PassRole`** (보통 `iam:GetRole`과 함께).
- 아무 서비스에나 넘길 수 있나? **No** — 역할의 **신뢰 정책(trust policy)** 이 허용하는 서비스에만 전달 가능.

## 시험 함정

- 웹 IdP 로그인엔 `AssumeRoleWithWebIdentity`가 아니라 **Cognito Identity Pools**가 정답인 경우가 많음 (AWS 공식 비권장) #exam/trap/advanced-identity
- 권한 경계는 사용자·역할 전용 — **그룹에는 적용 불가** #exam/trap/advanced-identity
- 명시적 Deny는 어떤 Allow보다 우선 — IAM 정책이 허용해도 버킷 정책의 Deny면 차단 #exam/trap/advanced-identity
- 역할을 맡으면 **원래 권한을 잃는다** — 양쪽 권한이 동시에 필요하면 리소스 기반 정책 #exam/trap/advanced-identity
- Simple AD는 온프레미스 AD와 **연결할 수 없다** (연결이 필요하면 Managed AD의 trust 또는 AD Connector) #exam/trap/advanced-identity
- STS 임시 자격 증명 유효 기간은 **15분~1시간** #exam/trap/advanced-identity
- PassRole은 IAM 권한이고, 실제 전달 가능 여부는 역할의 **trust policy**가 결정 #exam/trap/advanced-identity

## 관련 노트

[[IAM]] · [[Cognito]] · [[Organizations & 계정 관리]] · [[KMS & 암호화]]
