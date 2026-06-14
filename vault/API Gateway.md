---
service: Amazon API Gateway
exams: [saa, dva]
domains:
  - saa/high-performing-architectures
  - dva/development
status: learning
confidence: 1
tags:
  - service
  - saa/serverless
  - dva/serverless
---

# API Gateway

> API 관리 — 통합 유형, 스테이지·배포, 인증(Cognito/IAM/Lambda Authorizer), 스로틀링.

## 개요

클라이언트와 백엔드 사이에 두는 **API 입구**. 서버리스 앱에서 클라이언트 → API Gateway → [[Lambda]] → [[DynamoDB]]로 이어지는 그림의 맨 앞 계층이다. 직접 인프라를 두지 않고도 REST API(와 WebSocket)를 노출하고, 그 앞에서 인증·요청 제한(throttling)·캐싱·버전 관리·요청/응답 변환을 한꺼번에 처리한다.

핵심 가치는 "백엔드 앞에 관리 계층을 한 겹 씌운다"는 것이다. Lambda뿐 아니라 사내 HTTP 엔드포인트나 다른 AWS 서비스 앞에 붙여, 그 자체로는 없는 인증·rate limiting·API 키 같은 기능을 더한다.

## 설계 관점 (SAA) #saa

![[AWS Certified Solutions Architect Slides v47.pdf#page=482]]

> 추천 학습 순서상 "큰 그림" 자리입니다. 스테이지·배포·매핑 템플릿 등 세부는 3단계(DVA)에서 채웁니다. 여기서는 SAA에 나오는 통합 유형·엔드포인트·인증만 정리합니다.

### 무엇을 해 주나

WebSocket 지원, API 버전 관리(v1·v2), 환경 분리(dev·test·prod), 인증/인가, API 키 발급과 요청 throttling, Swagger/OpenAPI로 빠르게 정의, 요청·응답 변환과 검증, SDK 생성, 응답 캐싱.

### 통합 유형 — 뒤에 무엇을 붙이나

- **Lambda**: REST API를 Lambda로 받는 가장 흔한 조합.
- **HTTP**: 사내 HTTP API나 [[ELB & Auto Scaling|ALB]] 같은 백엔드 앞에 둬서 rate limiting·캐싱·인증·API 키를 더한다.
- **AWS Service**: 다른 AWS API를 그대로 노출. 예를 들어 클라이언트 요청을 [[SQS & SNS & Kinesis|SQS]]에 넣거나 Step Functions를 시작. 인증을 붙이고 공개적으로 안전하게 노출하려는 목적.

### 엔드포인트 타입 3종 (단골)

| 타입 | 누가 접근 | 특징 |
| --- | --- | --- |
| **Edge-Optimized** (기본) | 전 세계 클라이언트 | 요청이 [[CloudFront & Global Accelerator\|CloudFront]] 엣지를 거쳐 지연이 준다. API Gateway 자체는 **한 리전**에만 있음 |
| **Regional** | 같은 리전 클라이언트 | CloudFront를 직접 붙여 캐싱·배포를 더 통제하고 싶을 때 |
| **Private** | **내 VPC 안에서만** | 인터페이스 VPC 엔드포인트(ENI)로만 접근. 접근 범위는 **resource policy**로 정의 |

### 보안 — 인증 3종 (단골)

- **IAM Roles**: 사내(AWS 내부) 애플리케이션에 적합.
- **Cognito**: 외부 사용자(모바일 등)의 신원. 클라이언트가 [[Cognito]]에서 토큰을 받아 API Gateway에 넘기면 게이트웨이가 검증.
- **Custom Authorizer (Lambda Authorizer)**: 내 로직으로 직접 인증.

**Custom Domain Name + HTTPS**는 ACM 인증서로 한다. 여기서 함정:

- **Edge-Optimized 엔드포인트면 인증서가 `us-east-1`에 있어야** 한다.
- **Regional 엔드포인트면 인증서가 그 API Gateway 리전**에 있어야 한다.
- [[Route 53]]에 CNAME 또는 A-alias 레코드를 만들어 연결.

## 개발 관점 (DVA) #dva

![[AWS Certified Developer Slides v44.pdf#page=657]]

%% 이 시험의 관점에서 배운 내용을 정리합니다. %%

## 시험 함정

- 전 세계 사용자 지연 줄이기 → **Edge-Optimized**(기본, CloudFront 엣지 경유). API 자체는 한 리전에만 존재 #exam/trap/api-gateway
- **Edge-Optimized면 ACM 인증서가 `us-east-1`**, Regional이면 해당 API Gateway 리전에 있어야 함 #exam/trap/api-gateway
- VPC 안에서만 쓰는 API → **Private 엔드포인트**(인터페이스 VPC 엔드포인트), 접근은 resource policy로 제한 #exam/trap/api-gateway
- 외부(모바일) 사용자 인증 → **Cognito**, 사내 앱 → **IAM**, 직접 로직 → **Custom Authorizer** #exam/trap/api-gateway
- 기존 HTTP/ALB 백엔드에 rate limiting·API 키·인증만 얹고 싶다 → API Gateway HTTP 통합 #exam/trap/api-gateway

## 관련 노트

[[Lambda]] · [[Cognito]] · [[Step Functions & AppSync]] · [[DynamoDB]] · [[SQS & SNS & Kinesis]] · [[CloudFront & Global Accelerator]] · [[Route 53]] · [[ELB & Auto Scaling]]
