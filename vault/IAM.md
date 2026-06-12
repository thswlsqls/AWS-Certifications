---
service: AWS IAM
exams: [saa, dva, cloudops]
domains:
  - saa/secure-architectures
  - dva/security
  - cloudops/security-compliance
status: learning
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

%% 이 시험의 관점에서 배운 내용을 정리합니다. %%

## 시험 함정

- 그룹 안에 그룹은 못 넣는다. 그룹의 구성원은 user뿐이다. #exam/trap/iam
- EC2 위 애플리케이션에 권한이 필요하면 Access Key 하드코딩이 아니라 IAM Role을 붙인다. #exam/trap/iam
- Credentials Report는 계정 단위, Access Advisor는 user 단위 — 문제에서 "전체 사용자 현황"인지 "특정 사용자의 안 쓰는 권한"인지로 가른다. #exam/trap/iam
- IAM은 글로벌 서비스라 리전별로 user를 따로 만들지 않는다. #exam/trap/iam

%% 연습문제에서 틀리거나 헷갈린 지점을 위 형식으로 계속 추가합니다. 시험 직전 주에 `tag:#exam/trap` 검색으로 한 번에 모아 봅니다. %%

## 관련 노트

[[고급 ID 관리]] · [[KMS & 암호화]] · [[Organizations & 계정 관리]]
