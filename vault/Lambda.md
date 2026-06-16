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

DVA는 함수를 **어떻게 호출하고, 권한을 주고, 배포·버전 관리하느냐**를 묻는다(개발 도메인의 핵심). 설계 관점에서 정리한 한계값·동시성·콜드스타트·VPC·엣지는 반복하지 않는다.

### 호출 모델 3가지 (가장 중요)

**① 동기(Synchronous)** — 결과를 바로 받는다.

- 트리거: CLI·SDK·**API Gateway**·**ALB**. 에러 처리(재시도·지수 백오프)는 **호출하는 쪽(client)** 책임.
- **ALB 통합**: 함수를 **Target Group**에 등록. ALB가 HTTP↔JSON을 변환(`requestContext`·`httpMethod`·`queryStringParameters`·`body`·`isBase64Encoded`). **Multi-Value Headers**를 켜면 같은 키의 값이 **배열**로 온다. ALB가 함수를 부르려면 Lambda **Resource-based Policy**가 `elasticloadbalancing`을 허용해야 한다.

**② 비동기(Asynchronous)** — 결과를 안 기다린다.

- 트리거: **S3·SNS·EventBridge**·CodeCommit·CodePipeline 등. 이벤트는 Lambda 내부 **Event Queue**에 쌓인다.
- **자동 재시도 3회**(1분 뒤, 다시 2분 뒤). 그래서 **처리는 idempotent**해야 한다(중복 실행 대비). 재시도 시 CloudWatch Logs에 같은 로그가 중복으로 남는다.
- 실패분은 **DLQ(SNS·SQS)** 또는 **Destination**으로 보낸다.

**③ Event Source Mapping (ESM)** — Lambda가 소스를 **폴링**한다.

- 대상: **Kinesis Data Streams·SQS(+FIFO)·DynamoDB Streams**. 공통점은 "레코드를 끌어와야 한다"는 것. 가져온 배치로 함수를 **동기 호출**.
- **스트림(Kinesis·DynamoDB)**: shard마다 iterator. 기본은 함수가 에러를 내면 **배치 전체를 성공할 때까지 재처리**하고, 순서 보장을 위해 **해당 shard 처리를 멈춘다**. parallelization으로 shard당 최대 10배치 병렬(파티션 키 단위 순서는 유지). 저트래픽이면 batch window로 모아서.
- **SQS**: Long Polling으로 폴링, batch size 1~10. **권장: 큐 Visibility Timeout을 함수 타임아웃의 6배**로. **DLQ는 Lambda가 아니라 SQS 큐에 설정한다**(Lambda DLQ는 비동기 전용). FIFO는 같은 GroupID 순서 보장.
- **스케일링**: 스트림은 shard당 1호출, SQS Standard는 분당 +60 인스턴스(최대 1000배치 동시), SQS FIFO는 활성 메시지 그룹 수만큼.

### Event·Context 객체

핸들러는 `(event, context)`를 받는다.

- **Event**: 처리할 입력 데이터(JSON → Python이면 dict). 호출한 서비스 정보가 담긴다.
- **Context**: 실행 환경 정보 — `aws_request_id`, `function_name`, `invoked_function_arn`, `memory_limit_in_mb`, `log_group_name`, `log_stream_name`.

### Destinations

- **비동기 호출**: 성공/실패 각각에 목적지를 지정 — **SQS·SNS·Lambda·EventBridge bus**. AWS는 이제 DLQ보다 **Destination을 권장**(둘 다 동시 사용 가능).
- **ESM**: 버려진 배치 → SQS·SNS.

### 권한 — Execution Role vs Resource-based Policy

- **Execution Role(IAM 역할)**: 함수가 **다른 AWS 서비스에 접근**할 권한. 관리형 정책 예: `AWSLambdaBasicExecutionRole`(CloudWatch Logs), `AWSLambdaKinesisExecutionRole`, `AWSLambdaDynamoDBExecutionRole`, `AWSLambdaSQSQueueExecutionRole`, `AWSLambdaVPCAccessExecutionRole`(VPC ENI), `AWSXRayDaemonWriteAccess`. **ESM도 이벤트를 읽을 때 이 역할을 쓴다.** 함수당 역할 하나가 권장.
- **Resource-based Policy**: **다른 계정·서비스가 이 함수를 호출**하게 허용(S3 버킷 정책과 비슷). 예: S3·ALB가 함수를 부를 때.

### 환경 변수

키-값(문자열). 코드를 안 고치고 동작을 바꾼다. **비밀값을 KMS로 암호화**(Lambda 서비스 키 또는 내 CMK)해 담을 수 있다.

### Execution Context와 /tmp (성능 함정)

- **Execution Context**: 핸들러 밖에서 초기화한 것(DB 커넥션·SDK·HTTP 클라이언트)을 담는 임시 런타임. 다음 호출이 **재사용**해 초기화 시간을 아낀다. → **DB 연결 같은 무거운 초기화는 핸들러 밖에서** 한 번만(핸들러 안에서 매번 연결하면 느림).
- **/tmp**: 큰 파일 다운로드·디스크 작업용. 최대 **10GB**, 컨텍스트가 살아 있는 동안 유지되는 임시 캐시. 영구 보관은 S3, 암호화는 KMS Data Key.

### 의존성·패키징·배포 형식

- **의존성**: 라이브러리를 코드와 함께 zip(Node는 `node_modules`, Python은 `pip --target`, Java는 `.jar`). **50MB 미만이면 직접 업로드, 넘으면 S3 경유.** 네이티브 라이브러리는 Amazon Linux에서 컴파일. **AWS SDK는 기본 포함.**
- **Layers**: 무거운 의존성을 분리해 여러 함수가 **재사용**, 커스텀 런타임도 제공. **함수당 5개·총 250MB.**
- **Container Image**: 최대 10GB 이미지를 ECR에서. **반드시 Lambda Runtime API를 구현**해야 한다(임의 Docker면 ECS/Fargate). 로컬 테스트는 RIE(Runtime Interface Emulator).
- **CloudFormation 배포**: **inline**(`Code.ZipFile`, 의존성 못 넣음) vs **S3 경유**(`S3Bucket`·`S3Key`·`S3ObjectVersion`). **S3 코드만 바꾸고 이 키들을 안 바꾸면 CloudFormation이 함수를 갱신하지 않는다**(시험 함정). 다른 계정 배포는 S3 버킷 정책으로 get/list 허용.

### Versions & Aliases (배포 도메인 핵심)

- 작업 중인 건 **`$LATEST`(가변)**. 게시하면 **Version**이 생긴다. **Version = 코드 + 설정, 불변, 고유 ARN.**
- **Alias**: Version을 가리키는 **가변 포인터**(dev·test·prod). **가중치로 카나리 배포**(예: prod alias가 V1 95% + V2 5%). 이벤트 트리거·Destination은 alias에 안정적으로 건다. **Alias는 다른 Alias를 가리킬 수 없다.**
- **Lambda + CodeDeploy**: alias의 트래픽 전환을 자동화(SAM에 통합). **Linear**(N분마다 일정 비율 증가), **Canary**(X% 먼저 → 100%), **AllAtOnce**(즉시). **Pre/Post Traffic Hook**으로 배포 전후 함수 상태 점검. **AppSpec.yml**: Name·Alias·CurrentVersion·TargetVersion.

### Function URL

- 함수에 붙는 **전용 HTTPS 엔드포인트**(`https://<id>.lambda-url.<region>.on.aws`, 안 바뀜). 브라우저·curl·Postman으로 호출.
- **퍼블릭 인터넷 전용**(PrivateLink 미지원). **alias나 `$LATEST`에만** 붙고 다른 Version엔 안 됨. Resource-based Policy·CORS 지원, Reserved Concurrency로 스로틀.
- **AuthType NONE**(인증 없는 공개, 단 Resource Policy가 공개를 허용해야) vs **AWS_IAM**(`lambda:InvokeFunctionUrl` 필요. **같은 계정은 Identity OR Resource 정책, 크로스 계정은 둘 다 ALLOW**).

### 로깅·모니터링·추적

- **CloudWatch Logs**: 실행 로그(역할에 쓰기 권한 필요). **Metrics**: Invocations·Duration·ConcurrentExecutions·Errors·Throttles·**Iterator Age**(스트림 지연).
- **X-Ray Active Tracing**: 설정에서 켜면 데몬을 알아서 띄운다. 역할에 `AWSXRayDaemonWriteAccess`, 코드에 X-Ray SDK. 환경 변수 `_X_AMZN_TRACE_ID`·`AWS_XRAY_CONTEXT_MISSING`·`AWS_XRAY_DAEMON_ADDRESS`. ([[CloudWatch & CloudTrail & Config|X-Ray 상세]])
- **CodeGuru Profiling**: 런타임 성능 인사이트(Java·Python). 켜면 프로파일러 Layer·환경 변수·`AmazonCodeGuruProfilerAgentAccess` 정책이 붙는다.

### RAM·CPU 튜닝

RAM 128MB~10GB(1MB 단위). RAM을 올리면 vCPU도 늘어난다 — **1,792MB에서 vCPU 1개**, 그 이상은 멀티스레딩을 써야 이득(최대 6 vCPU). **CPU 바운드(연산 무거움)면 RAM을 올려라.** Timeout 기본 3초·최대 900초.

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
- 호출 3분류: 동기(API GW·ALB, 에러처리 클라이언트), 비동기(S3·SNS, 3회 재시도→DLQ/Destination), **ESM**(Kinesis·SQS·DynamoDB Streams를 Lambda가 폴링) #exam/trap/lambda
- **SQS를 ESM으로 쓸 때 DLQ는 Lambda가 아니라 SQS 큐에 설정**(Lambda DLQ는 비동기 전용). 큐 Visibility Timeout은 함수 타임아웃의 6배 권장 #exam/trap/lambda
- **$LATEST·Version은 불변, Alias는 가변 포인터**. 카나리는 alias 가중치. Alias는 다른 alias 못 가리킴 #exam/trap/deployment
- CloudFormation S3 배포에서 **코드만 바꾸고 S3Key/S3ObjectVersion을 안 바꾸면 함수가 갱신 안 됨** #exam/trap/lambda
- Execution Role = 함수가 다른 서비스 접근(ESM 읽기도 이걸 씀), Resource-based Policy = 다른 계정·서비스가 함수 호출 #exam/trap/lambda
- DB 커넥션 등 무거운 초기화는 **핸들러 밖**에서(Execution Context 재사용). 핸들러 안에서 매번 연결하면 느림 #exam/trap/lambda
- RAM 1,792MB에서 vCPU 1개, CPU 바운드면 RAM을 올려 성능 확보 #exam/trap/lambda
- Container Image는 **Lambda Runtime API 구현 필수**(임의 Docker는 ECS/Fargate). Layer는 함수당 5개·총 250MB #exam/trap/lambda
- Function URL은 퍼블릭 인터넷 전용(PrivateLink 미지원), alias·$LATEST에만. AuthType: 같은 계정 OR·크로스 계정 AND #exam/trap/lambda
- CodeDeploy 트래픽 전환: Linear/Canary/AllAtOnce + Pre/Post Traffic Hook #exam/trap/deployment

## 관련 노트

[[API Gateway]] · [[DynamoDB]] · [[SQS & SNS & Kinesis]] · [[Step Functions & AppSync]] · [[S3]] · [[Cognito]] · [[CloudFront & Global Accelerator]] · [[RDS & Aurora & ElastiCache]] · [[VPC]] · [[CICD]] · [[CloudFormation]] · [[CloudWatch & CloudTrail & Config]] · [[컨테이너 서비스]] · [[ELB & Auto Scaling]] · [[KMS & 암호화]]
