---
service: AWS CDK · AWS SAM
exams: [dva]
domains:
  - dva/deployment
status: reviewing
confidence: 1
tags:
  - service
  - dva/deployment
---

# CDK

> CloudFormation 위에 얹는 두 프레임워크 — SAM(서버리스용 선언형 YAML)과 CDK(프로그래밍 언어로 모든 리소스). 둘 다 CloudFormation으로 합성된다.

## 개요

[[CloudFormation]]을 직접 쓰는 대신, 그 위에 얹어 더 편하게 인프라를 정의하는 두 도구를 묶어 정리한다. **둘 다 마지막엔 CloudFormation 템플릿을 만들어 배포**하는 게 공통점이다.

- **SAM (Serverless Application Model)**: 서버리스(Lambda·API Gateway·DynamoDB) 중심. 짧은 **YAML**로 쓰면 복잡한 CloudFormation으로 펼쳐 준다. Lambda를 빠르게 시작할 때 좋다.
- **CDK (Cloud Development Kit)**: 모든 AWS 서비스 대상. **프로그래밍 언어**(JavaScript/TypeScript·Python·Java·.NET)로 인프라를 코드처럼 짠다. 인프라와 앱 런타임 코드를 같이 배포할 수 있어 Lambda·컨테이너(ECS/EKS)에 잘 맞는다.

갈림길: **서버리스를 선언형 YAML로 빠르게** → SAM. **익숙한 언어로 모든 리소스를, 앱 코드와 함께** → CDK.

## 개발 관점 (DVA) #dva

![[AWS Certified Developer Slides v44.pdf#page=750]]

### AWS SAM — 개념과 배포 흐름

![[AWS Certified Developer Slides v44.pdf#page=738|p.738]]

서버리스 앱 개발·배포 프레임워크. 설정이 전부 **YAML**이고, CloudFormation의 기능(Outputs·Mappings·Parameters·Resources)을 그대로 쓸 수 있다.

- **템플릿 표시**: 맨 위에 **`Transform: 'AWS::Serverless-2016-10-31'`** 헤더가 있으면 SAM 템플릿. 리소스 타입은 **`AWS::Serverless::Function`·`AWS::Serverless::Api`·`AWS::Serverless::SimpleTable`**.
- **배포 흐름**: SAM 템플릿(YAML) + 앱 코드 → **transform**으로 CloudFormation 템플릿 생성 → zip해서 **S3 업로드** → **ChangeSet 생성·실행** → CloudFormation 스택(Lambda·API Gateway·DynamoDB)이 만들어진다.
- **명령**: `sam build`(로컬 빌드) → `sam package`(선택) → **`sam deploy`**.

### SAM Accelerate (`sam sync`)

AWS로 배포할 때 지연을 줄이는 기능. 핵심은 **코드만 바꿀 땐 CloudFormation을 우회**한다는 점.

- `sam sync` (옵션 없음): 코드 + 인프라 동기화.
- **`sam sync --code`**: 인프라 업데이트 없이 **코드 변경만** 동기화. **CloudFormation을 우회**(service API 직접 호출)해 수초 만에 반영.
- `sam sync --code --resource AWS::Serverless::Function`: Lambda 함수와 의존성만.
- `sam sync --code --resource-id <이름>`: 특정 리소스만.
- **`sam sync --watch`**: 파일 변경을 감시해 자동 동기화. 설정 변경이 섞이면 `sam sync`, 코드만이면 `sam sync --code`를 자동 선택.

### SAM Policy Templates

Lambda 함수에 권한을 붙이는 **미리 만들어진 정책 템플릿** 목록. 직접 IAM 정책을 쓰지 않아도 된다. 예: **`S3ReadPolicy`**(S3 객체 읽기), **`SQSPollerPolicy`**(SQS 큐 폴링), **`DynamoDBCrudPolicy`**(CRUD = create/read/update/delete).

### SAM과 CodeDeploy — Lambda 점진 배포

SAM은 **CodeDeploy를 내부적으로 써서** Lambda를 업데이트한다(트래픽 시프트). 배포 전후 검증과 자동 롤백이 핵심.

- **`AutoPublishAlias`**: 새 코드가 배포되면 새 버전을 발행하고 **alias를 새 버전으로** 옮긴다.
- **`DeploymentPreference`**: **Canary·Linear·AllAtOnce** 중 트래픽 이동 방식.
- **`Alarms`**: 지정한 CloudWatch Alarm이 울리면 **롤백**.
- **`Hooks`**: 트래픽 이동 **전(PreTraffic)·후(PostTraffic)** 에 검증용 Lambda를 돌려 배포를 확인.

### SAM 로컬 기능

로컬에서 서버리스를 띄워 테스트한다.

- **`sam local start-lambda`**: Lambda를 흉내 내는 로컬 엔드포인트. 자동화 테스트에 쓴다.
- **`sam local invoke`**: payload로 함수를 한 번 호출하고 끝. 테스트 케이스 생성에 유용.
- **`sam local start-api`**: 함수들을 호스팅하는 로컬 HTTP 서버(API Gateway). 함수 변경이 자동 reload.
- **`sam local generate-event`**: S3·API Gateway·SNS·Kinesis·DynamoDB 등의 **샘플 이벤트 payload** 생성.
- 함수가 AWS API를 호출하면 **`--profile`** 옵션을 맞게 줘야 한다.

### SAM 여러 환경 (`samconfig.toml`)

`samconfig.toml`에 dev·prod 등 환경별 스택 이름·S3 버킷·리전·파라미터를 정의하고, **`sam deploy --config-env dev`** 처럼 환경을 골라 배포한다.

### AWS CDK — 개념

![[AWS Certified Developer Slides v44.pdf#page=749|p.749]]

익숙한 **프로그래밍 언어**(JS/TS·Python·Java·.NET)로 인프라를 정의한다. 코드는 **`cdk synth`** 로 CloudFormation 템플릿(JSON/YAML)으로 "컴파일"된다.

- 인프라 정의(예: VPC·ECS 클러스터)와 **앱 런타임 코드를 함께 배포**할 수 있어 Lambda·ECS/EKS 컨테이너에 잘 맞는다.
- 흐름: CDK 앱(Constructs) → **CDK CLI `cdk synth`** → CloudFormation 템플릿 → CloudFormation 배포.
- **CDK vs SAM**: SAM은 서버리스 중심·선언형 JSON/YAML·Lambda 빠른 시작. CDK는 모든 AWS 서비스·프로그래밍 언어. **둘 다 CloudFormation을 바탕으로** 동작.
- **CDK + SAM**: **SAM CLI로 CDK 앱을 로컬 테스트**할 수 있는데, 먼저 **`cdk synth`** 로 템플릿을 만든 뒤 `sam local invoke -t <템플릿> <함수>` 식으로 쓴다.

### CDK Constructs — L1 / L2 / L3

**Construct**는 CloudFormation 스택을 만드는 데 필요한 것을 캡슐화한 컴포넌트. 리소스 하나(S3 버킷)일 수도, 여러 관련 리소스(큐 + 컴퓨팅)일 수도 있다. **AWS Construct Library**가 모든 AWS 리소스용 Construct를 담고, **Construct Hub**에 AWS·서드파티·커뮤니티 Construct가 더 있다. 추상화 수준에 따라 3단계.

| 레벨 | 별칭 | 특징 |
| --- | --- | --- |
| **L1** | CFN Resources | CloudFormation에 직접 있는 모든 리소스. CloudFormation Resource Spec에서 주기 생성. 이름이 **`Cfn`** 으로 시작(`CfnBucket`). **모든 속성을 직접 설정**해야 함 |
| **L2** | — | 한 단계 높은 **intent-based API**. 기능은 L1과 비슷하나 **기본값·boilerplate 제공**(속성을 다 몰라도 됨). 편의 메서드 제공(`bucket.addLifeCycleRule()`) |
| **L3** | Patterns | **여러 관련 리소스**를 묶어 흔한 작업을 한 번에. 예: `aws-apigateway.LambdaRestApi`(Lambda 백엔드 API Gateway), `aws-ecs-patterns.ApplicationLoadBalancerFargateService`(ALB + Fargate) |

### CDK 주요 명령

`npm install -g aws-cdk-lib`(CLI·라이브러리 설치) · `cdk init app`(새 프로젝트) · **`cdk synth`**(CloudFormation 템플릿 합성·출력) · **`cdk bootstrap`**(아래) · `cdk deploy`(스택 배포) · `cdk diff`(로컬 CDK와 배포된 스택 차이) · `cdk destroy`(스택 삭제).

### CDK Bootstrapping

CDK 앱을 배포하기 **전에** 필요한 리소스를 깔아 두는 과정. **AWS Environment = 계정 + 리전**.

- **`CDKToolkit`이라는 CloudFormation 스택**이 만들어지고, 파일 저장용 **S3 Bucket**과 배포 권한용 **IAM Role**을 담는다.
- **새 환경(계정·리전)마다** `cdk bootstrap aws://<account>/<region>` 을 한 번씩 실행해야 한다.
- 안 하면 **"Policy contains a statement with one or more invalid principal"** 에러.

### CDK Testing

CDK 앱은 **CDK Assertions Module**에 Jest(JS)·Pytest(Python)를 붙여 테스트한다. 특정 리소스·규칙·조건·파라미터가 있는지 검증.

- **Fine-grained Assertions**(흔함): 템플릿의 특정 부분을 검사(이 리소스가 이 속성·값을 갖는지).
- **Snapshot Tests**: 합성된 템플릿을 **저장해 둔 baseline과 비교**.
- 템플릿 가져오기: `Template.fromStack(MyStack)`(CDK로 만든 스택), `Template.fromString(mystring)`(CDK 밖에서 만든 스택).

## 시험 함정

- SAM은 **서버리스·선언형 YAML·Lambda 빠른 시작**, CDK는 **모든 서비스·프로그래밍 언어**. 둘 다 결국 **CloudFormation**으로 배포 #exam/trap/cdk
- SAM 템플릿 식별: 맨 위 **`Transform: AWS::Serverless-2016-10-31`** 헤더 + `AWS::Serverless::Function/Api/SimpleTable` #exam/trap/cdk
- 인프라는 그대로 두고 **코드만 수초 안에** 반영 → **`sam sync --code`**(CloudFormation 우회) #exam/trap/cdk
- SAM의 Lambda 점진 배포: **CodeDeploy** 사용, `AutoPublishAlias` + `DeploymentPreference`(Canary/Linear/AllAtOnce) + `Hooks`(Pre/PostTraffic 검증) + `Alarms`(롤백) #exam/trap/cdk
- Lambda에 권한 줄 때 직접 IAM 정책 대신 → **SAM Policy Templates**(S3ReadPolicy·SQSPollerPolicy·DynamoDBCrudPolicy) #exam/trap/cdk
- 로컬에서 Lambda·API Gateway 실행·이벤트 생성 → **`sam local`**(start-lambda·invoke·start-api·generate-event) #exam/trap/cdk
- CDK Construct: **L1=Cfn\*(속성 전부 수동)**, **L2=기본값·메서드 제공**, **L3=Patterns(여러 리소스 묶음)** #exam/trap/cdk
- CDK 배포 전 **`cdk bootstrap`** 필요 → CDKToolkit 스택(S3+IAM Role). 안 하면 **"invalid principal"** 에러 #exam/trap/cdk
- 코드를 CloudFormation 템플릿으로 합성 → **`cdk synth`**. SAM CLI로 CDK 로컬 테스트 시 먼저 `cdk synth` #exam/trap/cdk
- CDK 테스트 = **CDK Assertions Module** + Jest/Pytest. Fine-grained Assertions(흔함) vs Snapshot Tests #exam/trap/cdk

## 관련 노트

[[CloudFormation]] · [[CICD]] · [[Lambda]] · [[API Gateway]] · [[DynamoDB]] · [[컨테이너 서비스]]
