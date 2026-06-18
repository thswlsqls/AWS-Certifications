---
service: CodeCommit · CodePipeline · CodeBuild · CodeDeploy
exams: [dva]
domains:
  - dva/deployment
status: reviewing
confidence: 1
tags:
  - service
  - dva/deployment
---

# CICD

> AWS의 CI/CD 파이프라인 — 각 Code 서비스의 역할과 연계, CodeDeploy 배포 구성.

## 개요

코드를 손으로 올리는 단계가 많을수록 실수가 잦다. CICD는 그 과정을 **자동화**해서, 코드를 저장소에 push하면 빌드·테스트를 거쳐 AWS에 배포되게 만든다. 단계마다 검증을 끼우고 필요하면 사람 승인도 받는다.

용어 두 개를 나눠 둔다.

- **CI(Continuous Integration)**: 개발자가 코드를 자주 push하면 빌드·테스트 서버가 바로 검사해 피드백을 준다. 버그를 일찍 잡는다. → CodeBuild.
- **CD(Continuous Delivery)**: 검증을 통과한 빌드를 자주, 빠르게 배포할 수 있게 한다. "3개월에 한 번 릴리스"에서 "하루에 여러 번"으로. → CodeDeploy.

AWS가 묶어 주는 Code 계열 서비스:

- **CodeCommit** — 코드 저장(Git 저장소).
- **CodePipeline** — 전체 파이프라인을 자동으로 엮는 오케스트레이션.
- **CodeBuild** — 빌드·테스트.
- **CodeDeploy** — EC2·온프레미스·Lambda·ECS에 배포.
- **CodeArtifact** — 빌드 의존성(패키지) 저장·공유.
- **CodeGuru** — 머신러닝 기반 코드 리뷰·성능 분석.

## 개발 관점 (DVA) #dva

![[AWS Certified Developer Slides v44.pdf#page=701]]

### 전체 그림 (Technology Stack)

코드 → 빌드·테스트 → 배포·프로비저닝 순서로 흐르고, 각 칸을 AWS 서비스나 서드파티가 채운다.

- Source: CodeCommit / GitHub / Bitbucket
- Build·Test: CodeBuild / Jenkins
- Deploy·Provision: CodeDeploy, [[Elastic Beanstalk]]
- 전체를 엮는 오케스트레이션: **CodePipeline**

### AWS CodeCommit

AWS가 호스팅하는 **프라이빗 Git 저장소**. 코드가 AWS 계정 안에만 있어 보안·규정 측면에서 유리하고, 저장소 크기 제한이 없으며 Jenkins·CodeBuild 등과 붙는다. (실무에선 신규 생성이 막혔지만 DVA 시험엔 그대로 나온다.)

- **인증(Authentication)**: **SSH Keys**(IAM 콘솔에서 사용자가 등록) 또는 **HTTPS**(AWS CLI Credential helper / Git Credentials).
- **인가(Authorization)**: **IAM 정책**으로 사용자·역할의 저장소 권한을 준다.
- **암호화**: 저장 시 **KMS로 자동 암호화**, 전송 시 HTTPS/SSH로 암호화.
- **교차 계정 접근**: SSH 키나 AWS 자격증명을 **공유하지 말고**, IAM Role + **STS `AssumeRole`** 로 푼다.
- **CodeCommit vs GitHub**: 둘 다 Pull Request(코드 리뷰)·CodeBuild 연동·SSH/HTTPS 인증을 지원. 차이는 **보안 주체**(CodeCommit은 IAM 사용자·역할, GitHub는 GitHub 사용자)와 **호스팅**(CodeCommit은 AWS 관리, GitHub는 GitHub 호스팅 또는 Enterprise 자체 호스팅), UI(GitHub가 더 풍부).

### AWS CodePipeline

CICD를 엮는 **시각적 워크플로**. 단계(stage)로 구성되고 각 단계는 순차/병렬 액션을 가질 수 있다.

- 단계별 가능한 서비스: **Source**(CodeCommit·ECR·S3·Bitbucket·GitHub) → **Build**(CodeBuild·Jenkins 등) → **Test**(CodeBuild·Device Farm) → **Deploy**(CodeDeploy·Elastic Beanstalk·CloudFormation·ECS·S3) → **Invoke**(Lambda·Step Functions).
- **수동 승인(Manual approval)**을 어느 단계에든 끼울 수 있다.
- **Artifacts**: 각 단계가 만든 산출물은 **S3 버킷에 저장**되고 다음 단계로 넘어간다.
- **트러블슈팅**:
  - 파이프라인/액션/단계 **상태 변화**는 **CloudWatch Events(EventBridge)** 로 잡는다(예: 실패한 파이프라인·취소된 단계에 이벤트).
  - 단계가 실패하면 파이프라인이 멈추고 콘솔에서 원인을 본다.
  - 액션을 수행 못 하면 연결된 **IAM Service Role**의 권한(IAM 정책)이 충분한지 확인.
  - API 호출 감사는 **CloudTrail**.

### AWS CodeBuild

완전관리형 **CI 서비스**. 서버를 직접 두지 않고(빌드 큐 없이 스케일), 소스를 컴파일·테스트하고 패키지를 만든다. **빌드에 걸린 시간(분) 단위로 과금**, 내부적으로 **Docker**로 재현 가능한 빌드를 돌린다.

- **Source**: CodeCommit·S3·Bitbucket·GitHub.
- **빌드 정의 `buildspec.yml`**: **코드 루트**에 둬야 한다.
  - `env`: 환경 변수 — `variables`(평문), **`parameter-store`**(SSM Parameter Store), **`secrets-manager`**(Secrets Manager).
  - `phases`: **install**(의존성 설치) → **pre_build**(빌드 전 마무리, 예: docker login) → **build**(실제 빌드) → **post_build**(마무리, 예: zip).
  - `artifacts`: S3에 올릴 산출물(**KMS로 암호화**).
  - `cache`: 재사용할 파일(보통 의존성)을 S3에 캐시해 다음 빌드를 빠르게.
- **로그**: S3 / CloudWatch Logs. 모니터링은 CloudWatch Metrics(빌드 통계), **EventBridge**(실패 감지·알림), CloudWatch Alarms(임계치).
- 빌드 환경: Java·Ruby·Python·Go·Node.js·Android·.NET Core·PHP, 그리고 **Docker**(원하는 환경 직접 확장).
- 기본은 AWS가 관리하는 네트워크에서 돌지만, **VPC 안에서** 실행해 사설 리소스(RDS·ElastiCache 등)에 접근하게 할 수 있다.

### AWS CodeDeploy

애플리케이션 배포를 자동화하는 서비스. **`appspec.yml`** 파일로 배포 방식을 정의한다. 배포 대상마다 고를 수 있는 옵션이 다른 게 핵심.

**대상 플랫폼별 배포 방식**

| 대상 | 배포 방식 | 비고 |
| --- | --- | --- |
| **EC2 / 온프레미스** | **In-place** 또는 **Blue/Green** | 대상에 **CodeDeploy Agent** 필요. 속도: AllAtOnce(다운타임 최대)·HalfAtATime(용량 50%↓)·OneAtATime(가장 느림·영향 최소)·Custom(%) |
| **Lambda** | 트래픽 시프트 | alias 트래픽을 옮김. **SAM에 통합**. Linear(N분마다 증가)·Canary(X%만 먼저 후 100%)·AllAtOnce |
| **ECS** | **Blue/Green만** | 새 Task Definition 배포. **ALB 필수**. Linear·Canary·AllAtOnce |

- **CodeDeploy Agent**(EC2/온프레미스): 대상에 떠 있어야 한다. **Systems Manager로 자동 설치·갱신** 가능. EC2는 배포 번들을 받기 위해 **S3 접근 IAM 권한** 필요.
- **ASG 배포**: **In-place**(기존 인스턴스 갱신, ASG가 새로 띄우는 인스턴스도 자동 배포됨) / **Blue/Green**(새 ASG를 만들어 설정 복사, 옛 인스턴스 유지 기간 선택, **ELB 필수**).
- **롤백**: 자동(배포 실패 시 또는 **CloudWatch Alarm** 임계치 충족 시) 또는 수동. **롤백 비활성화**도 선택 가능. 롤백은 마지막 정상 리비전을 **새 배포로 다시 올리는 것**(예전 버전을 복원하는 게 아님).

### AWS CodeArtifact

빌드에 필요한 **패키지(의존성) 저장·공유** 서비스(artifact management). 직접 아티팩트 서버를 운영하던 일을 대신한다.

- **Maven·Gradle·npm·yarn·twine·pip·NuGet** 같은 의존성 도구와 연동. 개발자와 **CodeBuild**가 여기서 의존성을 바로 받아 온다.
- **퍼블릭 저장소를 프록시**해 캐시할 수 있다(npm·PyPI·Maven 등).
- **EventBridge 연동**: 패키지 버전이 생성·수정·삭제되면 이벤트 발생 → Lambda·Step Functions·SNS·SQS·CodePipeline 트리거(예: 최신 보안 패치로 **재빌드·재배포**).
- **Resource Policy**: 다른 계정에 접근을 허용. 한 principal은 저장소의 **패키지를 전부 읽거나 전혀 못 읽거나** 둘 중 하나(부분 권한 없음).

### Amazon CodeGuru

머신러닝 기반 **자동 코드 리뷰 + 애플리케이션 성능 추천**. 두 기능으로 나뉜다.

- **CodeGuru Reviewer**: 정적 분석으로 결함·보안 취약점·찾기 어려운 버그·리소스 누수를 잡는다(**개발 단계**). **Java·Python** 지원, GitHub·Bitbucket·CodeCommit 연동.
- **CodeGuru Profiler**: 실행 중(**운영 단계**) 애플리케이션의 동작을 분석해 CPU 과다 사용·비효율 코드를 찾고 비용을 줄인다. heap summary, **이상 탐지(Anomaly Detection)**, 오버헤드 최소.
- Profiler **Agent 설정**: `MaxStackDepth`(프로파일에 담을 스택 최대 깊이), `MemoryUsageLimitPercent`, `MinimumTimeForReportingInMilliseconds`, `ReportingIntervalInMilliseconds`, `SamplingIntervalInMilliseconds`(줄이면 샘플링 빈도↑).

## 시험 함정

- CI = 자주 push·빌드·테스트(**CodeBuild**), CD = 자주 배포(**CodeDeploy**). 전체를 엮는 건 **CodePipeline** #exam/trap/cicd
- **`buildspec.yml`(CodeBuild)**: 코드 **루트**에 위치. **`appspec.yml`(CodeDeploy)**: 배포 방식 정의. 둘을 혼동 금지 #exam/trap/cicd
- buildspec의 비밀값은 **`parameter-store`(SSM)** / **`secrets-manager`** 로 주입, 빌드 캐시·아티팩트는 **S3**(아티팩트는 KMS 암호화) #exam/trap/cicd
- CodeDeploy 대상별 옵션: EC2/온프레미스는 **In-place/Blue-Green** + 속도(AllAtOnce·HalfAtATime·OneAtATime), **ECS는 Blue/Green만(ALB 필수)**, Lambda·ECS는 **Linear/Canary** #exam/trap/cicd
- EC2/온프레미스 배포엔 **CodeDeploy Agent** 필요(SSM으로 자동 설치). 인스턴스는 번들을 받기 위해 **S3 접근 권한** 필요 #exam/trap/cicd
- ASG **Blue/Green** 배포는 새 ASG 생성 + **ELB 필수** #exam/trap/cicd
- CodeDeploy 롤백 = 마지막 정상 리비전을 **새 배포로 재배포**(복원이 아님). 자동 롤백 트리거는 배포 실패 또는 **CloudWatch Alarm** #exam/trap/cicd
- 파이프라인 상태 변화 감지 → **CloudWatch Events(EventBridge)**. 액션 실패 시 → **IAM Service Role 권한** 확인. API 감사 → **CloudTrail** #exam/trap/cicd
- CodeCommit 교차 계정 접근 → 자격증명 공유 말고 **IAM Role + STS `AssumeRole`**. 저장소는 **KMS 자동 암호화** #exam/trap/cicd
- 의존성(패키지) 저장·공유, 퍼블릭 저장소 프록시 → **CodeArtifact**. CodeBuild가 여기서 의존성 fetch #exam/trap/cicd
- 코드 결함·보안 정적 분석 → **CodeGuru Reviewer**(Java·Python). 운영 중 성능·CPU·비용 → **CodeGuru Profiler** #exam/trap/cicd

## 관련 노트

[[CloudFormation]] · [[Elastic Beanstalk]] · [[컨테이너 서비스]] · [[Lambda]] · [[Systems Manager]] · [[KMS & 암호화]] · [[CloudWatch & CloudTrail & Config]] · [[IAM]]
