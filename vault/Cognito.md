---
service: Amazon Cognito
exams: [saa, dva]
domains:
  - saa/secure-architectures
  - dva/security
status: learning
confidence: 1
tags:
  - service
  - saa/security
  - dva/security
---

# Cognito

> 애플리케이션 사용자 인증 — User Pools(인증)와 Identity Pools(AWS 자격 증명) 구분.

## 개요

웹·모바일 앱의 **사용자에게 신원(identity)을 주는** 서비스. [[IAM]]이 AWS 계정 내부 사용자·역할을 다룬다면, Cognito는 앱을 쓰는 **외부 최종 사용자**(앱 가입자, 모바일 사용자)를 다룬다.

두 갈래가 있고 이 구분이 핵심이다.

- **Cognito User Pools (CUP)**: 앱의 **로그인 기능**. 사용자가 누구인지 인증하고 토큰을 발급한다. = 사용자 디렉터리.
- **Cognito Identity Pools (Federated Identities)**: 인증된 사용자에게 **임시 AWS 자격증명**을 줘서 S3·DynamoDB 같은 AWS 리소스에 직접 접근하게 한다. = 권한 부여.

둘은 같이 쓰기도 한다. User Pools로 로그인 → 그 신원을 Identity Pools에 넘겨 AWS 자격증명을 받는 식.

## 설계 관점 (SAA) #saa

![[AWS Certified Solutions Architect Slides v47.pdf#page=489]]

> 추천 학습 순서상 "큰 그림" 자리입니다. 토큰·트리거 등 세부는 3단계(DVA)에서 채웁니다. 여기서는 SAA에 나오는 두 풀의 구분과 선택 기준만 정리합니다.

### Cognito User Pools (CUP) — 인증

앱을 위한 **서버리스 사용자 디렉터리**. 사용자명(또는 이메일)/비밀번호 로그인, 비밀번호 재설정, 이메일·전화번호 인증, **MFA**, 그리고 Facebook·Google·SAML 같은 외부 신원 연동(Federated Identities)을 제공한다.

- **통합**: [[API Gateway]], [[ELB & Auto Scaling|Application Load Balancer]]와 붙는다.
  - API Gateway 앞: 클라이언트가 CUP에서 토큰을 받아 API Gateway에 넘기면 게이트웨이가 토큰을 검증하고 백엔드(Lambda)로 보낸다.
  - ALB 앞: ALB의 리스너 규칙에서 CUP로 인증을 시킨 뒤 대상 그룹(EC2·Lambda·컨테이너)으로 보낸다.

### Cognito Identity Pools — AWS 자격증명

"사용자"에게 **임시 AWS 자격증명**을 발급해 AWS 서비스에 직접(또는 API Gateway 경유) 접근하게 한다.

- 사용자 출처: Cognito User Pools, 서드파티 로그인(Google·Facebook·SAML·OpenID Connect) 등.
- 자격증명에 적용되는 **IAM 정책은 Cognito에서 정의**한다. 인증된 사용자와 게스트 사용자에 대한 **기본 IAM 역할**을 둘 수 있다.
- 정책을 **user_id 기준으로 세분화**할 수 있다. 예: DynamoDB에서 **자기 행만** 읽고 쓰게 하는 Row Level Security — `dynamodb:LeadingKeys`를 `${cognito-identity.amazonaws.com:sub}`(그 사용자의 ID)에 묶는다.

### Cognito vs IAM — 언제 쓰나

슬라이드가 짚는 판단 기준: **"수백 명 규모의 사용자", "모바일 사용자", "SAML로 인증"** 같은 상황이면 IAM이 아니라 Cognito다. IAM 사용자는 AWS를 직접 다루는 소수(직원·시스템)용이고, 앱의 최종 사용자가 많고 외부에서 들어오면 Cognito로 푼다.

## 개발 관점 (DVA) #dva

![[AWS Certified Developer Slides v44.pdf#page=763]]

%% 이 시험의 관점에서 배운 내용을 정리합니다. %%

## 시험 함정

- "앱 사용자 로그인·가입·MFA" → **User Pools**. "그 사용자에게 임시 AWS 자격증명을 줘 S3·DynamoDB 접근" → **Identity Pools**. 둘을 구분 #exam/trap/cognito
- "모바일 사용자 수백~수천 명에게 AWS 리소스 접근 권한" → IAM 사용자가 아니라 **Cognito (Identity Pools)** #exam/trap/cognito
- 사용자가 DynamoDB에서 **자기 데이터만** 접근 → Identity Pools + IAM 정책의 `${cognito-identity.amazonaws.com:sub}`로 Row Level Security #exam/trap/cognito
- API Gateway/ALB 앞단 인증을 외부 사용자로 → **Cognito User Pools** 통합 #exam/trap/cognito

## 관련 노트

[[API Gateway]] · [[IAM]] · [[고급 ID 관리]] · [[ELB & Auto Scaling]] · [[DynamoDB]] · [[Lambda]]
