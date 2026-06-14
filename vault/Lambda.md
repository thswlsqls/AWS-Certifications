---
service: AWS Lambda
exams: [saa, dva, cloudops]
domains:
  - saa/high-performing-architectures
  - dva/development
  - cloudops/deployment-automation
status: learning
confidence: 1
tags:
  - service
  - saa/serverless
  - dva/serverless
  - cloudops/serverless
---

# Lambda

> 서버리스 컴퓨팅 — 동시성, 레이어, 환경 변수, 이벤트 소스, 권한 모델.

## 개요

서버리스는 "서버가 없다"가 아니라 **서버를 내가 관리·프로비저닝·신경 쓰지 않는다**는 뜻이다. 처음엔 FaaS(Function as a Service)를 가리켰고 Lambda가 그 시작이었지만, 지금은 관리형 DB·메시징·스토리지까지 폭넓게 묶어 부른다.

Lambda는 **짧게 도는 함수**를 이벤트가 올 때만(on-demand) 실행하고, 확장은 자동으로 된다. EC2(계속 떠 있고 RAM·CPU에 묶이고 확장에 손이 감)와 대비된다. 요청 수와 실행 시간(GB-초)으로만 과금해서 보통 매우 싸다. 함수에 RAM을 더 주면 CPU·네트워크 성능도 같이 올라간다.

AWS 서버리스 묶음의 전형: 사용자가 [[S3]](정적 콘텐츠) · [[API Gateway]](REST API) · [[Cognito]](로그인)로 들어오고, API Gateway 뒤에 **Lambda**가 붙고, 그 뒤에 [[DynamoDB]]가 데이터를 받는 그림.

## 설계 관점 (SAA) #saa

![[AWS Certified Solutions Architect Slides v47.pdf#page=441]]

> 이 행은 추천 학습 순서상 "큰 그림만" 보는 자리입니다. 함수 작성·배포 세부는 3단계(DVA)에서 채웁니다. 여기서는 SAA에 나오는 한계값과 아키텍처 판단만 정리합니다.

### 언어와 컨테이너 이미지

Node.js·Python·Java·C#(.NET)·Ruby를 기본 지원하고, Custom Runtime API로 Rust·Go도 가능. 컨테이너 이미지로도 돌릴 수 있지만 그 이미지는 **Lambda Runtime API를 구현해야** 한다. 그냥 임의의 Docker 이미지를 돌리는 거라면 ECS/Fargate가 낫다.

### 알아둘 한계값 (리전 단위)

- 메모리 **128MB ~ 10GB**(1MB 단위). RAM을 올리면 CPU·네트워크도 좋아진다.
- 최대 실행 시간 **900초(15분)**. 이보다 오래 걸리는 작업엔 부적합.
- 환경 변수 4KB, `/tmp` 디스크 512MB~10GB.
- **동시 실행(concurrency) 1000**(상향 가능, 지원 티켓).
- 배포 패키지: 압축 .zip 50MB, 압축 해제 250MB.

### 동시성과 Throttling (단골)

- 함수 단위로 **Reserved Concurrency**(예약 동시성 = 상한)를 둘 수 있다.
- 동시성 한도를 넘는 호출은 **Throttle**된다. 처리 방식이 호출 유형에 따라 다르다:
  - **동기 호출**(API Gateway·ALB·SDK 등): `ThrottleError 429`를 즉시 반환.
  - **비동기 호출**(S3 이벤트 등): 자동 재시도 후 안 되면 **DLQ**로. throttle(429)·시스템 오류(5xx)는 최대 6시간까지 재시도하고, 재시도 간격은 1초에서 최대 5분까지 지수적으로 늘어난다.
- **함정**: 한 함수의 예약 동시성을 안 두면 그 함수가 계정 전체 1000을 다 써버려 **다른 함수가 throttle**될 수 있다.

### 콜드 스타트 / Provisioned Concurrency

- **Cold Start**: 새 인스턴스가 뜰 때 코드 로드 + 핸들러 밖 초기화가 일어나 **첫 요청 지연**이 크다.
- **Provisioned Concurrency**: 호출 전에 미리 동시성을 띄워 둬 콜드 스타트를 없앤다. Application Auto Scaling으로 일정·목표 사용률에 맞춰 관리.
- **SnapStart**(Java·Python·.NET): 초기화된 상태의 스냅샷을 캐싱해 추가 비용 없이 최대 10배 빠르게.

### 네트워킹 — VPC 접근 (자주 나옴)

- **기본값: Lambda는 내 VPC 밖(AWS 소유 VPC)에서 실행**된다. 그래서 사설 서브넷의 RDS·ElastiCache·내부 ELB에 **닿지 못한다**(퍼블릭 DynamoDB 같은 건 됨).
- 사설 자원에 접근하려면 함수를 **VPC에 배치**(VPC ID·서브넷·보안 그룹 지정). 그러면 Lambda가 그 서브넷에 **ENI**를 만든다.
- **RDS Proxy**: 함수가 많아지면 DB 커넥션이 폭증한다. RDS Proxy로 커넥션을 풀링·공유해 확장성·가용성(failover 시간 66% 단축)·보안(IAM 인증, Secrets Manager)을 높인다. **RDS Proxy는 퍼블릭 접근이 안 되므로 Lambda를 반드시 VPC 안에 둬야** 한다.

### 엣지에서 실행 — CloudFront Functions vs Lambda@Edge

[[CloudFront & Global Accelerator|CloudFront]]에 코드를 붙여 사용자 가까이에서 돌려 지연을 줄이는 것. 둘 중 선택이 출제된다.

| | CloudFront Functions | Lambda@Edge |
| --- | --- | --- |
| 언어 | JavaScript | Node.js · Python |
| 규모 | **초당 수백만** 요청 | 초당 수천 요청 |
| 트리거 | Viewer Request/Response | Viewer + **Origin** Request/Response |
| 실행 시간 | < 1ms | 5~10초 |
| 메모리 | 2MB | 128MB~10GB |
| 네트워크·파일시스템·요청 본문 접근 | 불가 | **가능** |
| 쓰는 경우 | 캐시 키 정규화, 헤더 조작, URL 재작성, 간단한 토큰(JWT) 검증 | 외부 서비스 호출·SDK 의존, 무거운 처리, 요청 본문 접근 |

→ 가볍고 초고빈도면 CloudFront Functions, 외부 연동·무거운 로직이면 Lambda@Edge.

### 데이터베이스가 Lambda를 부르는 경우

- **RDS/Aurora → Lambda 호출**: DB 안에서 Lambda를 불러 데이터 이벤트를 처리(예: INSERT 후 이메일 발송). **RDS for PostgreSQL·Aurora MySQL**만 지원. DB에서 Lambda로 나가는 통신 경로(퍼블릭·NAT GW·VPC 엔드포인트)와 호출 권한(Lambda Resource-based Policy + IAM)이 필요.
- **RDS Event Notifications**: DB **인스턴스 자체**의 상태 변화(생성·중지·시작 등) 알림. 데이터 내용은 안 알려준다. 최대 5분 지연, SNS나 EventBridge로 받는다.

## 개발 관점 (DVA) #dva

![[AWS Certified Developer Slides v44.pdf#page=527]]

%% 이 시험의 관점에서 배운 내용을 정리합니다. %%

## 운영 관점 (CloudOps) #cloudops

![[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=214]]

%% 이 시험의 관점에서 배운 내용을 정리합니다. %%

## 시험 함정

- "Lambda가 RDS·ElastiCache 등 사설 자원에 접근 못 함" → 기본은 **VPC 밖**. 함수를 **VPC에 배치**해야 ENI가 생겨 접근 가능 #exam/trap/lambda
- 15분(900초)을 넘는 작업은 Lambda로 못 한다 → ECS/Fargate·Batch 등 고려 #exam/trap/lambda
- 한 함수가 throttle되어 다른 함수까지 영향 → **Reserved Concurrency**로 함수별 상한을 둬 격리 #exam/trap/lambda
- 첫 요청만 느리다(콜드 스타트) → **Provisioned Concurrency**(또는 Java/Python/.NET이면 SnapStart) #exam/trap/lambda
- 동기 호출 throttle은 429 즉시 반환, 비동기는 재시도 후 **DLQ**행 — 둘을 구분 #exam/trap/lambda
- Lambda가 DB 커넥션을 너무 많이 연다 → **RDS Proxy**(단, Lambda를 VPC 안에 둬야 함) #exam/trap/lambda
- 엣지: 가볍고 초당 수백만이면 **CloudFront Functions**, Origin 트리거·외부 호출·요청 본문 접근이면 **Lambda@Edge** #exam/trap/lambda

## 관련 노트

[[API Gateway]] · [[DynamoDB]] · [[SQS & SNS & Kinesis]] · [[Step Functions & AppSync]] · [[S3]] · [[Cognito]] · [[CloudFront & Global Accelerator]] · [[RDS & Aurora & ElastiCache]] · [[VPC]]
