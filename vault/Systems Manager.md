---
service: AWS Systems Manager
exams: [cloudops]
domains:
  - cloudops/deployment-automation
  - cloudops/security
status: reviewing
confidence: 1
tags:
  - service
  - cloudops/management
---

# Systems Manager

> EC2·온프레미스를 대규모로 운영 — Session Manager, Run Command, Patch Manager, Parameter Store, Automation 등 기능 묶음.

## 개요

EC2와 온프레미스 서버를 **한곳에서 대규모로 관리**하는 도구 묶음(SSM). 인프라 상태를 파악하고, 문제를 찾고, **패치를 자동화**해 컴플라이언스를 맞춘다. Windows·Linux 모두 지원, CloudWatch·Config와 붙고, **서비스 자체는 무료**(돌리는 리소스 비용만).

쓰려면 두 가지가 전제다 — 관리할 대상에 **SSM Agent**가 떠 있어야 하고(Amazon Linux 2·일부 Ubuntu AMI엔 기본 설치), EC2에는 **SSM 동작을 허용하는 IAM Role**이 있어야 한다. "인스턴스가 SSM에 안 잡힌다"는 문제는 대개 이 둘(에이전트·IAM) 중 하나다.

## 운영 관점 (CloudOps) #cloudops

![[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=40]]

CloudOps에서 가장 깊게 다루는 서비스다. 기능이 많지만 시험은 **"이 상황이면 어느 기능"**을 묻는다. 접속(Session Manager)·명령 실행(Run Command)·패치(Patch Manager)·설정/비밀(Parameter Store)·작업 자동화(Automation)를 구분해 둔다.

### 동작 전제 — SSM Agent와 IAM

- 제어 대상에 **SSM Agent** 설치(Amazon Linux 2·일부 Ubuntu 기본 탑재). 안 잡히면 보통 에이전트 문제.
- EC2 인스턴스 role에 **`AmazonSSMManagedInstanceCore`** 정책: 인스턴스 등록, Session Manager·Documents·Parameters 접근, Run Command 수신, Patch Manager, Inventory·Compliance 보고, heartbeat.
- **Default Host Management Configuration (DHMC)**: 켜면 **EC2 Instance Profile 없이** EC2를 관리 인스턴스로 자동 구성. 권한 없이 식별만 하는 **Instance Identity Role**을 쓴다. **IMDSv2 필수**(IMDSv1 미지원), SSM Agent 설치 필요. Session Manager·Patch Manager·Inventory 자동 활성화, 에이전트 자동 최신화. **리전별로** 켠다.

### Documents · Run Command · Automation

- **SSM Documents**: **JSON/YAML**로 파라미터·액션을 정의한 스크립트. AWS 제공 문서가 많다. Run Command·State Manager·Patch Manager·Automation의 공통 재료.
- **Run Command**: 문서(스크립트)나 명령을 **여러 인스턴스에 한 번에** 실행(Resource Groups 기준). **SSH 불필요**, IAM·CloudTrail 통합, Rate/Error Control. 출력은 콘솔·S3·CloudWatch Logs, 상태는 SNS 알림, EventBridge로 호출 가능.
- **Automation**: EC2·AWS 리소스의 흔한 유지보수·배포 작업을 자동화(재시작, AMI 생성, EBS 스냅샷 등). **Automation Runbook**(Automation 타입 SSM Document)으로 정의. 트리거: 콘솔/CLI/SDK·EventBridge·**Maintenance Windows(스케줄)**·**AWS Config(규칙 교정)**. 예: Source AMI를 패치해 **Golden AMI**를 만들고 Launch Template을 갱신해 ASG를 instance refresh.

### 접속 — Session Manager

EC2·온프레미스에 **안전한 셸**을 연다. **SSH 접근·bastion host·SSH 키가 전혀 필요 없다**(인바운드 22번도 안 연다). Linux·macOS·Windows 지원.

- 접근 제어는 **IAM**: 어떤 사용자·그룹이 어떤 인스턴스(태그로 제한)에 붙을지, 실행 가능한 명령까지 제한. 인스턴스엔 **IAM Instance Profile(`AmazonSSMManagedInstanceCore`)** 만 있으면 된다.
- 연결·실행 명령을 로깅, **세션 로그는 S3·CloudWatch Logs**로. **CloudTrail이 `StartSession`** 이벤트를 가로챈다.
- **SSH vs Session Manager**: SSH는 SG에 22번 인바운드가 필요하지만, Session Manager는 **인바운드 규칙이 필요 없다.** (EC2 노트의 EC2 Instance Connect와 혼동 주의 — 그쪽은 22번을 연다.)

### Patch Manager

관리 인스턴스 **패치 자동화**(OS·앱·보안 업데이트). EC2·온프레미스, Linux·macOS·Windows. 온디맨드 또는 **Maintenance Windows**로 스케줄. 스캔 후 **컴플라이언스 리포트(누락 패치)** 생성, S3로 전송 가능.

- **Patch Baseline**: 어떤 패치를 설치/거부할지 정의. **기본은 critical·security 패치만**, 출시 후 며칠 내 자동 승인. **Pre-defined**(AWS 관리, 수정 불가 — `AWS-RunPatchBaseline`) vs **Custom**(승인/거부·커스텀 저장소 지정).
- **Patch Group**: 인스턴스 묶음을 특정 Patch Baseline에 연결(환경별 dev/test/prod). 태그 키는 **`Patch Group`**. **인스턴스는 한 Patch Group에만**, **Patch Group은 한 Patch Baseline에만** 등록된다.
- Maintenance Windows로 패치할 땐 **Rate Control**로 동시에 패치할 대상 수·비율을 제한.

### Parameter Store

설정·비밀을 **안전하게 저장**. **KMS로 선택적 암호화**(SecureString). 서버리스·확장·내구성, **버전 추적**, IAM 보안, EventBridge 알림, CloudFormation 통합.

- **계층(Hierarchy)**: `/부서/앱/환경/키` 식 경로. **`GetParameters`**(여러 개) / **`GetParametersByPath`**(경로 하위 전체) API. `/aws/reference/secretsmanager/...`로 **Secrets Manager 참조**, `/aws/service/...`로 최신 AMI 같은 **public 파라미터**.
- **Standard vs Advanced**: Standard는 **10,000개·값 4KB·정책 없음·무료**, Advanced는 **100,000개·8KB·정책 가능·월 $0.05**.
- **Parameter Policies(Advanced 전용)**: 파라미터에 **TTL(만료일)**을 줘 비밀번호 등을 강제로 갱신/삭제. `Expiration`(삭제), `ExpirationNotification`·`NoChangeNotification`(EventBridge 알림). 여러 정책 동시 적용 가능.

### 인벤토리·상태 관리·기타

- **Fleet Manager**: AWS·온프레미스·엣지·IoT 노드를 **중앙에서 원격 관리**(상태·헬스·성능, 트러블슈팅, Windows RDP/Session Manager). SSM Agent 필수, `AmazonSSMManagedInstanceCore` 또는 DHMC.
- **Inventory**: 관리 인스턴스의 **메타데이터 수집**(설치 소프트웨어·OS 드라이버·설정·업데이트·실행 서비스). 콘솔에서 보거나 **S3 저장 후 Athena·QuickSight**로 분석. 다중 계정·리전, Custom Inventory.
- **State Manager**: 인스턴스를 **정의한 상태로 유지**(부트스트랩, 패치 스케줄). **State Manager Association**으로 원하는 상태(예: 포트 22 닫힘, 안티바이러스 설치)와 적용 스케줄을 정의하며 SSM Document를 쓴다.
- **Distributor**: 관리 인스턴스에 **소프트웨어 패키징·배포**. Distributor Package(SSM Document, S3 저장, OS별 zip + JSON manifest). 일회성은 **Run Command**, 스케줄은 **State Manager**(`AWS-ConfigureAWSPackage`).
- **OpsCenter**: 운영 이슈를 **한 곳에서 보고·조사·교정**(여러 서비스를 옮겨다닐 필요 없이). **OpsItems**(조사·교정 필요한 이슈)로 모으고 **추천 Runbook**을 제시. 소스: CloudWatch·SSM Incident Manager·EventBridge·DevOps Guru·Config·Security Hub → OpsCenter → SNS.

## 시험 함정

- 인스턴스가 SSM에 안 잡힘 → 대개 **SSM Agent 미설치** 또는 **IAM(`AmazonSSMManagedInstanceCore`) 누락**. #exam/trap/ssm
- Instance Profile 없이 EC2를 SSM 관리 → **Default Host Management Configuration**(IMDSv2 필수, 리전별). #exam/trap/ssm
- **bastion·SSH 키·22번 포트 없이** 안전하게 셸 접속 → **Session Manager**(IAM Instance Profile만, 인바운드 규칙 불필요). #exam/trap/ssm
- 여러 인스턴스에 SSH 없이 명령·스크립트 실행 → **Run Command**(출력 S3/CloudWatch Logs, EventBridge 트리거). #exam/trap/ssm
- 재시작·AMI 생성 등 정형 작업 자동화·Config 규칙 교정 → **Automation Runbook**. #exam/trap/ssm
- 패치: **Patch Baseline=무엇을 패치**(기본 critical·security), **Patch Group=어떤 인스턴스 묶음**(태그키 `Patch Group`). 인스턴스는 한 그룹, 그룹은 한 베이스라인. #exam/trap/ssm
- 설정·비밀 저장 + 버전·계층 + KMS 암호화 → **Parameter Store**. 만료(TTL)·알림 정책은 **Advanced tier**만. #exam/trap/ssm
- 관리 인스턴스의 설치 소프트웨어·설정 메타데이터 수집 → **Inventory**(S3+Athena/QuickSight). 원하는 상태 유지 → **State Manager**. #exam/trap/ssm

## 관련 노트

[[EC2]] · [[CloudFormation]] · [[CloudWatch & CloudTrail & Config]] · [[KMS & 암호화]] · [[ELB & Auto Scaling]] · [[기타 서비스]] · [[IAM]]
