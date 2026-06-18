---
service: Amazon Cognito
exams: [saa, dva]
domains:
  - saa/secure-architectures
  - dva/security
status: reviewing
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

SAA 섹션이 "User Pool과 Identity Pool을 어떻게 구분하나"였다면, 여기서는 개발자가 실제로 만나는 것 — 로그인 후 받는 토큰의 정체, 인증 흐름에 끼워 넣는 Lambda, 의심스러운 로그인 차단, Identity Pool이 임시 자격증명을 받아오는 방식 — 을 정리한다.

### User Pools (CUP) — 기능과 토큰

앱을 위한 **서버리스 사용자 데이터베이스**. 사용자명(또는 이메일)/비밀번호 로그인, 비밀번호 재설정, 이메일·전화번호 인증, **MFA**, Facebook·Google·SAML 등 외부 신원 연동을 제공한다. **다른 곳에서 자격증명이 유출된 사용자를 차단**하는 기능도 있다.

- **로그인하면 JWT(JSON Web Token)를 돌려준다.** 이게 핵심.

### JWT 토큰 — 디코딩과 검증

CUP가 발급하는 JWT는 **Header · Payload · Signature** 세 부분(Base64 인코딩).

- **Signature를 반드시 검증해야** 토큰을 믿을 수 있다. CUP가 발급한 토큰의 유효성은 라이브러리로 검증한다.
- Payload에는 사용자 정보가 담긴다: **`sub`(Cognito DB의 사용자 ID, UUID)**, email, given_name, phone_number, `cognito:username`, `cognito:groups`, `exp`(만료), `iat`(발급 시각) 등. `sub` UUID로 Cognito/OIDC에서 사용자 상세를 다시 가져올 수 있다.

### Lambda Triggers — 인증 흐름에 코드 끼우기

CUP는 인증 흐름의 특정 시점에 **Lambda를 동기 호출**한다. 흐름을 커스터마이즈하고 싶을 때 쓴다.

- 인증 이벤트: **Pre Authentication**(로그인 요청 수락/거부 검증), **Post Authentication**(분석용 로깅), **Pre Token Generation**(토큰 클레임 추가/제거).
- 가입: **Pre Sign-up**(가입 요청 검증), **Post Confirmation**(환영 메시지·로깅), **Migrate User**(기존 사용자 디렉터리에서 이전).
- 메시지: **Custom Message**(메시지 커스터마이즈·현지화).

### Hosted Authentication UI

Cognito가 제공하는 **가입·로그인 화면**을 앱에 붙일 수 있다. 소셜 로그인·OIDC·SAML 연동의 토대가 되고, **로고와 CSS를 커스터마이즈**할 수 있다.

- **커스텀 도메인을 쓰려면 ACM 인증서가 `us-east-1`에 있어야** 한다. 도메인은 콘솔의 "App Integration"에서 정의.

### Adaptive Authentication — 위험 기반 인증

의심스러운 로그인이면 **차단하거나 MFA를 추가로 요구**한다. Cognito가 로그인 시도마다 **위험 점수(low/medium/high)** 를 매긴다(같은 기기·위치·IP인지 등으로 판단). 위험이 감지될 때만 2차 MFA를 띄운다. 유출된 자격증명·계정 탈취도 점검하고, **CloudWatch Logs**에 로그인 시도·위험 점수·실패한 챌린지를 남긴다.

### Identity Pools — 임시 AWS 자격증명을 받는 방식

(SAA 섹션의 Row Level Security에 더해) 개발자가 챙길 점:

- **자격증명 출처**: 퍼블릭 공급자(Login with Amazon·Facebook·Google·Apple), Cognito User Pool, OIDC·SAML 공급자, **Developer Authenticated Identities**(직접 만든 로그인 서버), 그리고 **인증 안 한 게스트(guest) 접근**도 가능.
- **IAM 자격증명 획득 방식**: Identity Pool이 **STS의 `AssumeRoleForWebIdentity` API**로 임시 자격증명을 받아온다. 그래서 그 **IAM Role에는 Cognito Identity Pool을 신뢰하는 trust policy**가 있어야 한다.
- **역할 선택**: 인증된 사용자·게스트용 **기본 IAM Role**을 두고, 사용자 ID 기준 규칙으로 역할을 고른다. **정책 변수(policy variables)** 로 사용자별 접근을 나눈다 — 예: S3는 `${cognito-identity.amazonaws.com:sub}` 접두사로 자기 폴더만, DynamoDB는 `dynamodb:LeadingKeys`로 자기 행만.

### User Pools vs Identity Pools — 한 줄 정리

- **CUP = 인증(authentication, 신원 확인)**: 사용자 DB, 소셜·OIDC·SAML 연동, Hosted UI, Lambda Triggers, MFA·Adaptive Authentication.
- **CIP = 인가(authorization, 접근 제어)**: 사용자에게 AWS 자격증명 발급, 게스트 가능, IAM Role·정책에 매핑.
- **CUP + CIP = 인증 + 인가**. 둘을 묶으면 "User Pool로 로그인 → 그 신원을 Identity Pool에 넘겨 AWS 자격증명 획득"이 된다.

## 시험 함정

- "앱 사용자 로그인·가입·MFA" → **User Pools**. "그 사용자에게 임시 AWS 자격증명을 줘 S3·DynamoDB 접근" → **Identity Pools**. 둘을 구분 #exam/trap/cognito
- "모바일 사용자 수백~수천 명에게 AWS 리소스 접근 권한" → IAM 사용자가 아니라 **Cognito (Identity Pools)** #exam/trap/cognito
- 사용자가 DynamoDB에서 **자기 데이터만** 접근 → Identity Pools + IAM 정책의 `${cognito-identity.amazonaws.com:sub}`로 Row Level Security #exam/trap/cognito
- API Gateway/ALB 앞단 인증을 외부 사용자로 → **Cognito User Pools** 통합 #exam/trap/cognito
- CUP 로그인은 **JWT** 반환. 토큰을 믿으려면 **Signature 검증** 필수, Payload의 `sub`가 사용자 ID(UUID) #exam/trap/cognito
- 인증 흐름에 커스텀 로직(가입 검증·환영 메시지·토큰 클레임 수정) → **Lambda Triggers**(동기 호출) #exam/trap/cognito
- 의심스러운 로그인만 MFA 요구·차단 → **Adaptive Authentication**(위험 점수 low/med/high) #exam/trap/cognito
- Hosted UI 커스텀 도메인은 **ACM 인증서가 `us-east-1`** 에 있어야 함 #exam/trap/cognito
- Identity Pool은 **STS `AssumeRoleForWebIdentity`** 로 임시 자격증명 획득. 그 IAM Role엔 **Cognito Identity Pool 신뢰 정책** 필요 #exam/trap/cognito
- 로그인 안 한 사용자에게도 제한된 AWS 접근 → Identity Pool의 **게스트(unauthenticated) 접근** #exam/trap/cognito
- 직접 만든 로그인 서버로 인증한 사용자에게 AWS 자격증명 → **Developer Authenticated Identities** #exam/trap/cognito

## 관련 노트

[[API Gateway]] · [[IAM]] · [[고급 ID 관리]] · [[ELB & Auto Scaling]] · [[DynamoDB]] · [[Lambda]]
