---
service: AWS IAM
exams: [saa, dva, cloudops]
domains:
  - saa/secure-architectures
  - dva/security
  - cloudops/security-compliance
status: reviewing
confidence: 1
tags:
  - service
  - saa/security
  - dva/security
  - cloudops/security
---

# IAM

> AWS 자원 접근 권한을 관리하는 글로벌 서비스. 세 시험 모두의 공통 기반.

## 개요

IAM은 "누가 AWS에서 무엇을 할 수 있는가"를 관리하는 서비스다. 리전을 선택하지 않는 **글로벌 서비스**라서, 만든 사용자와 정책은 모든 리전에 똑같이 적용된다.

- **Root account** — 계정을 만들 때 자동 생성. 계정 초기 설정 외에는 쓰지도, 공유하지도 않는다.
- **User** — 조직의 실제 사람 한 명. "물리적 사용자 1명 = AWS user 1명"이 원칙이다.
- **Group** — user를 묶는 단위. **그룹 안에는 user만 들어가고, 그룹을 중첩할 수 없다.** user는 그룹에 안 속해도 되고, 여러 그룹에 동시에 속할 수도 있다.
- **Policy** — 권한을 정의하는 JSON 문서. user나 group에 붙인다.
- **Role** — 사람이 아니라 **AWS 서비스**(EC2, Lambda 등)에 권한을 줄 때 쓰는 신원.

## 설계 관점 (SAA) #saa

![[AWS Certified Solutions Architect Slides v47.pdf#page=24]]

### Policy 구조

권한 부여의 원칙은 **최소 권한(least privilege)** — 필요한 만큼만 준다. Policy JSON은 `Version`(항상 `"2012-10-17"`)과 `Statement`(필수)로 구성되고, 각 statement는 다음으로 이루어진다.

| 필드 | 의미 |
| --- | --- |
| `Sid` | statement 식별자 (선택) |
| `Effect` | `Allow` 또는 `Deny` |
| `Principal` | 이 정책이 적용되는 계정/user/role |
| `Action` | 허용·거부할 동작 목록 (예: `s3:GetObject`) |
| `Resource` | Action이 적용되는 자원의 ARN |
| `Condition` | 정책이 발동하는 조건 (선택) |

정책 상속: 그룹에 붙인 정책은 그 그룹의 모든 user에게 적용된다. 그룹을 거치지 않고 user에게 직접 붙이는 정책은 **inline policy**라고 부른다.

### AWS 접근 3가지 경로와 자격 증명

| 경로 | 보호 수단 |
| --- | --- |
| Management Console | 비밀번호 + MFA |
| CLI | Access Key |
| SDK (코드에서 호출) | Access Key |

- Access Key ID는 username, Secret Access Key는 password에 해당한다. **비밀번호처럼 절대 공유하지 않는다.**
- 비밀번호는 password policy로 통제한다: 최소 길이, 문자 종류 강제, 본인 변경 허용, 만료 기간, 재사용 금지.
- MFA = 아는 것(비밀번호) + 가진 것(기기). 비밀번호가 털려도 계정은 뚫리지 않는다는 게 핵심 효용. 기기 종류: Virtual MFA(Google Authenticator·Authy, 한 폰에 토큰 여러 개), U2F 보안 키(YubiKey, 키 하나로 여러 user 지원), Hardware Key Fob(GovCloud용 별도 제품 존재).

### IAM Role — 서비스에 권한 주기

EC2 인스턴스 위의 애플리케이션이 AWS API를 호출해야 한다면, Access Key를 인스턴스에 넣는 게 아니라 **IAM Role을 인스턴스에 붙인다**. 시험에서 "EC2에 자격 증명을 안전하게 주는 방법"은 거의 항상 Role이 답이다. 대표 사례: EC2 Instance Role, Lambda Function Role, CloudFormation Role.

### 감사 도구 2개 구분

- **IAM Credentials Report** — **계정 단위**. 모든 user와 각자의 자격 증명 상태(비밀번호·키 사용 현황)를 목록으로 뽑는다.
- **IAM Access Advisor** — **user 단위**. 그 user에게 부여된 서비스 권한과 마지막 사용 시점을 보여준다. 안 쓰는 권한을 찾아 정책을 줄일 때(최소 권한) 쓴다.

"계정 전체 자격 증명 현황 → Credentials Report, 특정 user의 안 쓰는 권한 정리 → Access Advisor"로 구분한다.

## 개발 관점 (DVA) #dva

![[AWS Certified Developer Slides v44.pdf#page=21]]

%% 이 시험의 관점에서 배운 내용을 정리합니다. %%

## 운영 관점 (CloudOps) #cloudops

![[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=526]]

운영 시험은 user·group·role 기본기(위 설계 관점)보다 **권한을 어떻게 제한·위임하고, 누가 무엇을 할 수 있는지 검증·감사하느냐**를 묻는다. 보안 16% 도메인이지만 멀티 계정 운영과 엮여 자주 나온다.

### 권한 상한: Permission Boundary vs SCP

둘 다 "여기까지만"이라는 **권한의 천장**을 정하는 도구다. 실제로 허용되는 권한은 천장과 부여된 정책의 **교집합**이라, 정책에서 `Allow`해도 천장 밖이면 못 한다.

- **Permission Boundary** — managed policy로 **user 또는 role 한 개**의 최대 권한을 정한다(group에는 못 붙인다). 슬라이드 예시: 경계는 `s3/cloudwatch/ec2`만 허용하는데 정책은 `iam:CreateUser`만 허용 → 교집합이 비어 **결과 권한 없음**.
- **SCP**(Organizations) — **계정 단위** 천장. (자세한 건 [[Organizations & 계정 관리]].)
- 고를 때: **특정 user 한 명**만 묶고 싶으면 Permission Boundary, **계정 전체**를 묶고 싶으면 SCP. 대표 용도는 권한 위임 — 관리자가 아닌 사람에게 "이 경계 안에서는 IAM user를 직접 만들어 써라"를 허용하되 **스스로 관리자가 되는(권한 상승) 건 막는** 것.

### IAM Access Analyzer — 외부 공유 탐지

내 자원이 **신뢰 영역(Zone of Trust = 내 계정 또는 내 Organization) 밖으로 공유됐는지**를 찾아 finding으로 띄운다. 대상은 S3 버킷, IAM Role, KMS 키, Lambda 함수·레이어, SQS 큐, Secrets Manager 비밀. "버킷이 외부에 열려 있는지 점검" 같은 문제의 답.

### Identity Federation — IAM user 없이 외부 신원으로 접근

외부 사용자가 임시 role을 떠맡아(assume) AWS에 접근하게 한다. **IAM user를 안 만들어도 되는** 게 핵심. 어떤 방식인지로 갈린다.

| 상황 | 선택 |
| --- | --- |
| 기업 AD/ADFS 등 SAML 2.0 IdP를 콘솔·CLI에 연결 | SAML Federation |
| IdP가 SAML 2.0 호환이 아님 | Custom Identity Broker (브로커가 적절한 IAM policy를 결정) |
| 모바일·웹 앱에서 클라이언트가 AWS 자원에 직접 접근 | Cognito Federated Identity Pool |
| Facebook·Google 같은 OIDC로 로그인 (Web Identity Federation) | AWS는 권장 안 함 — **Cognito 사용 권고** |

### AWS STS — 임시 자격 증명 발급

임시·제한 권한을 발급하는 서비스. 토큰 수명은 **15분~1시간**(만료 시 갱신). API를 상황별로 외운다.

- **AssumeRole** — 같은 계정(보안 강화) 또는 **다른 계정(cross-account)**의 role을 떠맡는다.
- **AssumeRoleWithSAML** — SAML으로 로그인한 사용자용.
- **AssumeRoleWithWebIdentity** — Facebook·Google·OIDC 로그인용. **AWS는 대신 Cognito 권고.**
- **GetSessionToken** — MFA가 걸린 user나 root용.

**Cross-account access**: 대상 계정에 role을 만들고 누가 떠맡을 수 있는지 지정 → 소스 계정 쪽에서 `AssumeRole`로 임시 자격 증명을 받아 그 계정의 자원에 접근. 운영 계정의 S3 버킷을 개발 계정 개발자에게 열어주는 그림이 전형적이다.

### IAM Policy Simulator — 적용 전 권한 검증

정책을 실제로 붙이기 **전에** "이 user/group/role이 `s3:PutObject`를 할 수 있나"를 시험·디버깅한다. identity 기반 정책·resource 기반 정책·Permission Boundary·SCP를 모두 반영해 계산하므로, "왜 접근이 막혔나"를 추적하는 트러블슈팅 도구로 쓴다.

## 시험 함정

- 그룹 안에 그룹은 못 넣는다. 그룹의 구성원은 user뿐이다. #exam/trap/iam
- EC2 위 애플리케이션에 권한이 필요하면 Access Key 하드코딩이 아니라 IAM Role을 붙인다. #exam/trap/iam
- Credentials Report는 계정 단위, Access Advisor는 user 단위 — 문제에서 "전체 사용자 현황"인지 "특정 사용자의 안 쓰는 권한"인지로 가른다. #exam/trap/iam
- IAM은 글로벌 서비스라 리전별로 user를 따로 만들지 않는다. #exam/trap/iam
- Permission Boundary는 user·role에만 붙고 group에는 못 붙인다. 실제 권한은 경계와 정책의 교집합이라, 정책에서 Allow해도 경계 밖이면 거부된다. #exam/trap/iam
- "특정 user 한 명만 제한" → Permission Boundary, "계정 전체 제한" → SCP. 문제에서 범위가 한 명인지 계정인지로 가른다. #exam/trap/iam
- STS API 매칭: SAML 로그인 → AssumeRoleWithSAML, OIDC/소셜 로그인 → AssumeRoleWithWebIdentity(단 AWS는 Cognito 권고), MFA → GetSessionToken. #exam/trap/iam
- "자원이 외부에 공유됐는지 점검" → IAM Access Analyzer, "적용 전 정책이 동작하는지 테스트" → IAM Policy Simulator로 구분한다. #exam/trap/iam

%% 연습문제에서 틀리거나 헷갈린 지점을 위 형식으로 계속 추가합니다. 시험 직전 주에 `tag:#exam/trap` 검색으로 한 번에 모아 봅니다. %%

## 관련 노트

[[고급 ID 관리]] · [[KMS & 암호화]] · [[Organizations & 계정 관리]]
