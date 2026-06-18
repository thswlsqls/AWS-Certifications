---
service: AWS Step Functions · AppSync · Amplify
exams: [dva]
domains:
  - dva/development
status: reviewing
confidence: 1
tags:
  - service
  - dva/serverless
---

# Step Functions & AppSync

> 강의의 "Other Serverless" 단원 3종 — 워크플로 오케스트레이션(Step Functions), GraphQL 관리형 API(AppSync), 모바일·웹 앱 개발 도구 묶음(Amplify).

## 개요

서로 다른 세 서비스인데 강의가 "Other Serverless" 한 단원에 묶어 둔다. 셋 다 서버 관리 없이 쓰는 서버리스 계열이라는 공통점뿐이고, 역할은 다르다.

- **Step Functions**: 여러 단계로 이뤄진 작업을 **상태 머신(state machine)**으로 그려 순서·분기·재시도·대기를 관리한다. Lambda 여러 개를 코드로 직접 엮는 대신 워크플로로 빼낸다.
- **AppSync**: **GraphQL**을 쓰는 관리형 API 서비스. 클라이언트가 필요한 데이터만 골라 가져오고, 여러 데이터 소스를 한 번에 묶는다.
- **Amplify**: 모바일·웹 앱을 빠르게 만들도록 인증·데이터·호스팅을 묶어 주는 도구 모음. "모바일·웹용 [[Elastic Beanstalk]]"으로 이해하면 된다.

## 개발 관점 (DVA) #dva

![[AWS Certified Developer Slides v44.pdf#page=787]]

### Step Functions — 상태 머신

워크플로를 **상태 머신으로 모델링**한다(워크플로 하나당 상태 머신 하나). 정의는 **JSON**으로 쓰고, 콘솔에서 워크플로 그림·실행 과정·이력을 본다. 시작은 **SDK 호출·[[API Gateway]]·EventBridge(CloudWatch Event)**로 건다. 주문 처리, 데이터 처리 등 순서가 있는 작업에 쓴다.

**상태(State) 종류**

- **Task**: 실제 일을 하는 상태. ① **AWS 서비스 하나 호출**(Lambda 함수, Batch 작업, ECS 작업 후 완료 대기, DynamoDB 항목 삽입, SNS·SQS 발행, 다른 Step Functions 워크플로 실행) 또는 ② **Activity 실행**(EC2·ECS·온프레미스의 워커가 Step Functions를 폴링해 일을 받아 처리하고 결과를 돌려줌).
- **Choice**: 조건을 보고 분기(또는 기본 분기).
- **Fail / Succeed**: 실패·성공으로 실행 종료.
- **Pass**: 일을 하지 않고 입력을 그대로 출력으로 넘기거나 고정 데이터 주입.
- **Wait**: 일정 시간 또는 특정 시각까지 지연.
- **Map**: 단계를 동적으로 반복.
- **Parallel**: 병렬 분기 시작.

**에러 처리 — Retry와 Catch (앱 코드 말고 상태 머신에서)**

어떤 상태든 런타임 에러가 날 수 있다(상태 머신 정의 문제·Task 실패(Lambda 예외)·일시적 네트워크 문제). 이걸 **애플리케이션 코드가 아니라 상태 머신 정의 안의 Retry·Catch로** 처리한다.

- 예약 에러 코드: **`States.ALL`**(모든 에러), **`States.Timeout`**(TimeoutSeconds 초과 또는 heartbeat 없음), **`States.TaskFailed`**(실행 실패), **`States.Permissions`**(권한 부족).
- **Retry**(Task·Parallel state): 위에서 아래로 평가. `ErrorEquals`(에러 종류 매칭)·`IntervalSeconds`(첫 재시도 전 지연)·`BackoffRate`(재시도마다 지연 배수)·`MaxAttempts`(**기본 3**, **0이면 재시도 안 함**). 최대 시도를 넘기면 **Catch가 작동**.
- **Catch**(Task·Parallel state): 위에서 아래로 평가. `ErrorEquals`·`Next`(보낼 상태)·**`ResultPath`**(Next 상태로 넘길 입력을 정함, 예: `$.error`로 입력에 에러를 끼워 보냄).

**Wait for Task Token — 외부를 기다리기**

Task에서 **Task Token이 돌아올 때까지 워크플로를 멈춰** 둔다. 다른 AWS 서비스·**사람 승인**·서드파티 연동·레거시 시스템을 기다릴 때. **Resource 필드에 `.waitForTaskToken`을 붙이면** 그 Task는 토큰이 **`SendTaskSuccess`/`SendTaskFailure`**로 돌아올 때까지 일시 정지한다.

**Activity Tasks**

Task의 일을 **Activity Worker**(EC2·Lambda·모바일 등)가 처리하게 한다. 워커가 **`GetActivityTask`**로 작업을 폴링하고, 끝나면 **`SendTaskSuccess`/`SendTaskFailure`**로 응답. Task를 살려 두려면 `TimeoutSeconds`를 두고 **`SendTaskHeartBeat`**를 `HeartBeatSeconds` 안에 주기적으로 보낸다. 길게 잡으면 **최대 1년**까지 대기 가능.

**Standard vs Express**

| 항목 | Standard(기본) | Express |
| --- | --- | --- |
| 최대 실행 시간 | **최대 1년** | **최대 5분** |
| 실행 모델 | **Exactly-once** | At-least-once(비동기) / At-most-once(동기) |
| 실행 속도 | 2,000/초 이상 | 100,000/초 이상 |
| 이력 | 90일 또는 CloudWatch | CloudWatch Logs |
| 과금 | 상태 전이 수 | 실행 수·시간·메모리 |
| 용도 | **비멱등 작업**(결제 등) | IoT 수집·스트리밍·모바일 백엔드 |

- Express의 **비동기**: 워크플로 완료를 안 기다림(결과는 CW Logs), 즉시 응답 불필요(메시징), **멱등성 직접 관리**.
- Express의 **동기**: 워크플로 완료를 기다림, 즉시 응답 필요(마이크로서비스 오케스트레이션), **API Gateway·Lambda에서 호출** 가능.

### AWS AppSync — GraphQL 관리형 API

![[AWS Certified Developer Slides v44.pdf#page=801|p.801]]

**GraphQL**을 쓰는 관리형 서비스. GraphQL은 클라이언트가 **필요한 데이터만 정확히** 가져오게 하고, **여러 소스의 데이터를 한 번에 묶는다**. 모든 건 **GraphQL 스키마 하나를 업로드**하는 데서 시작한다.

- **데이터 소스(Resolver로 연결)**: DynamoDB·Aurora·OpenSearch, **Lambda(→ 무엇이든)**, HTTP(→ 퍼블릭 HTTP API).
- **실시간**: WebSocket 또는 MQTT on WebSocket로 데이터를 받는다. 모바일 앱의 **로컬 데이터 접근·동기화**도 지원.
- 모니터링은 CloudWatch Metrics & Logs.
- **보안 — 인가 4종**: **`API_KEY`**, **`AWS_IAM`**(IAM 사용자·역할·교차 계정), **`OPENID_CONNECT`**(OIDC 공급자·JWT), **`AMAZON_COGNITO_USER_POOLS`**.
- **커스텀 도메인 + HTTPS**가 필요하면 AppSync 앞에 **CloudFront**를 둔다.

### AWS Amplify — 모바일·웹 앱 도구 묶음

![[AWS Certified Developer Slides v44.pdf#page=805|p.805]]

모바일·웹 앱을 빠르게 만드는 도구 모음. **"모바일·웹용 Elastic Beanstalk"**. 데이터 저장·인증·스토리지·ML 같은 필수 기능을 AWS 서비스로 받쳐 주고, React·Vue·iOS·Android·Flutter용 프런트엔드 라이브러리를 준다. **Amplify CLI** 또는 **Amplify Studio**(시각적 풀스택)로 만들고 배포.

- **인증 (`amplify add auth`)**: **Cognito**를 끌어다 씀. 가입·인증·계정 복구, MFA·소셜 로그인, 사전 제작 UI, 세분화된 인가.
- **DataStore (`amplify add api`)**: **AppSync + DynamoDB**를 끌어다 씀. 로컬 데이터로 작업하고 **클라우드에 자동 동기화**(복잡한 코드 없이), GraphQL 기반, 오프라인·실시간, Amplify Studio로 시각적 데이터 모델링.
- **Amplify Hosting**: 웹 앱 빌드·호스팅, **CICD**(build·test·deploy), Pull Request 미리보기, 커스텀 도메인, 모니터링, 리다이렉트·헤더, 비밀번호 보호.
- **E2E 테스트**: `amplify.yml`의 **test phase**에서 엔드투엔드 테스트를 돌려 운영 배포 전 회귀를 잡는다. **Cypress** 연동.

## 시험 함정

- 여러 Lambda·단계를 순서·분기·재시도로 엮는 워크플로 → **Step Functions**(상태 머신, JSON 정의) #exam/trap/step-functions
- 재시도·실패 경로는 **앱 코드가 아니라 상태 머신의 Retry·Catch**에. `MaxAttempts` 기본 **3**, **0이면 재시도 안 함**, 초과 시 **Catch** 작동 #exam/trap/step-functions
- 예약 에러 코드: **`States.ALL`·`States.Timeout`·`States.TaskFailed`·`States.Permissions`** #exam/trap/step-functions
- 사람 승인·외부 시스템 대기 → Resource에 **`.waitForTaskToken`** + `SendTaskSuccess`/`SendTaskFailure`로 재개 #exam/trap/step-functions
- 외부 워커(EC2·온프레미스)가 처리 → **Activity Task**(`GetActivityTask` 폴링, `SendTaskHeartBeat`로 최대 1년 대기) #exam/trap/step-functions
- **Standard**(최대 1년·exactly-once·결제 등 비멱등) vs **Express**(최대 5분·고속·IoT/스트리밍, 동기는 API Gateway/Lambda 호출·멱등성 직접 관리) #exam/trap/step-functions
- 클라이언트가 필요한 데이터만·여러 소스 묶기·모바일 오프라인 동기화 → **AppSync(GraphQL)**, 스키마 업로드로 시작 #exam/trap/appsync
- AppSync 인가 4종: **API_KEY·AWS_IAM·OPENID_CONNECT·AMAZON_COGNITO_USER_POOLS**. 커스텀 도메인+HTTPS는 앞에 **CloudFront** #exam/trap/appsync
- "모바일·웹 앱을 빠르게, 인증은 Cognito·데이터는 AppSync+DynamoDB" → **Amplify**(DataStore 자동 동기화, Hosting은 CICD·PR 미리보기, E2E는 Cypress) #exam/trap/amplify

## 관련 노트

[[Lambda]] · [[API Gateway]] · [[DynamoDB]] · [[Cognito]] · [[SQS & SNS & Kinesis]] · [[컨테이너 서비스]] · [[CloudFront & Global Accelerator]] · [[Elastic Beanstalk]]
