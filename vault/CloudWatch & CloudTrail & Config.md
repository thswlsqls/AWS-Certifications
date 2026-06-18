---
service: Amazon CloudWatch · CloudTrail · AWS Config · X-Ray
exams: [saa, dva, cloudops]
domains:
  - saa/resilient-architectures
  - dva/troubleshooting
  - cloudops/monitoring-logging
status: reviewing
confidence: 1
tags:
  - service
  - saa/monitoring
  - dva/monitoring
  - cloudops/monitoring
---

# CloudWatch & CloudTrail & Config

> 관측성 3종 — 지표·로그·경보(CloudWatch), API 감사(CloudTrail), 구성 준수(Config). DVA는 X-Ray 포함.

## 개요

세 서비스는 "관측성(observability)"이라는 한 묶음으로 묶이지만 답하는 질문이 다르다. 시험에서는 시나리오가 어느 질문에 해당하는지로 서비스를 고른다.

- **CloudWatch** — 지금 시스템이 잘 돌아가는가? 지표·로그·경보·대시보드로 성능을 보고, 임계값을 넘으면 알림을 보내거나 자동 조치를 한다.
- **CloudTrail** — 누가 언제 무슨 작업(API 호출)을 했는가? 계정 안의 모든 API 호출 기록을 남기는 감사 로그. 리소스가 사라졌으면 여기부터 본다.
- **AWS Config** — 리소스 설정이 규정에 맞는가, 시간이 지나며 어떻게 바뀌었는가? 설정 변경 이력과 준수 여부를 기록한다. **막지는 못하고 기록·평가만 한다.**

여기에 **EventBridge**(옛 CloudWatch Events)가 붙는다. 위 서비스들이 만든 이벤트(스케줄, API 호출, 비준수 등)를 받아 Lambda·SNS·SQS 같은 대상으로 흘려보내는 라우터 역할이다.

## 설계 관점 (SAA) #saa

![[AWS Certified Solutions Architect Slides v47.pdf#page=578]]

### CloudWatch Metrics

모든 AWS 서비스가 지표를 자동으로 보낸다. 지표는 namespace에 속하고, dimension(인스턴스 ID·환경 등 속성)을 최대 30개까지 붙일 수 있다. **EC2 메모리(RAM) 사용량은 기본 지표에 없다** — 이건 Custom Metric으로 직접 올리거나 CloudWatch Unified Agent를 깔아야 한다. SAA 단골 함정.

Metric Streams를 켜면 지표를 Kinesis Data Firehose로 거의 실시간(near-real-time) 흘려보내 S3·Redshift·OpenSearch나 Datadog 같은 외부 도구로 보낼 수 있다.

### CloudWatch Logs

로그는 **로그 그룹(보통 애플리케이션 단위) → 로그 스트림(인스턴스·컨테이너·파일 단위)** 구조다. 보관 기간을 따로 정하지 않으면 영구 보관이므로 비용 문제로 출제된다 (1일~10년 또는 무기한 설정 가능). 기본 암호화되며 직접 만든 KMS 키로도 암호화할 수 있다.

로그를 다른 곳으로 내보내는 두 경로의 차이가 핵심이다.

- **S3 Export** — `CreateExportTask` API. 로그가 export 가능해지기까지 **최대 12시간**이 걸리고 실시간이 아니다. "실시간으로 로그를 S3/분석으로 보내라"는 문제면 오답.
- **Logs Subscriptions** — Subscription Filter로 거른 로그를 Kinesis Data Streams·Firehose·Lambda로 **실시간** 전달. 여러 계정·리전의 로그를 한 곳(KDS → Firehose → S3)으로 모으는 중앙 집계도 이걸로 한다. 다른 계정으로 보내는 Cross-Account Subscription도 가능.

로그 분석은 **CloudWatch Logs Insights** — 전용 쿼리 언어로 저장된 로그를 검색·집계한다. 단 "쿼리 엔진이지 실시간 엔진이 아니다." 실시간 처리가 필요하면 Subscriptions를 쓴다.

EC2 로그는 기본적으로 CloudWatch로 가지 않는다. 인스턴스에 **CloudWatch Agent**를 깔아야 한다.
- **Logs Agent** — 옛 버전, CloudWatch Logs로만 보냄.
- **Unified Agent** — RAM·프로세스 같은 시스템 레벨 지표까지 수집하고, 설정을 SSM Parameter Store로 중앙 관리. "EC2 메모리 지표를 모니터링하려면?" → Unified Agent.

### CloudWatch Alarms

상태는 OK / ALARM / **INSUFFICIENT_DATA** 셋뿐. 고해상도 커스텀 지표의 평가 주기는 10초·30초 또는 60초의 배수.

경보 대상(target)으로 출제되는 것:
- **EC2 Action** — 인스턴스 stop·terminate·reboot·recover
- **Auto Scaling Action**
- **SNS** — 여기서 사실상 무엇이든 연결

**Composite Alarm**은 여러 경보의 상태를 AND/OR로 묶는다. 단일 지표가 아니라 "경보들의 경보"라서, 잡음(alarm noise)을 줄여 진짜 문제일 때만 알리고 싶을 때 답이 된다.

경보는 Logs의 **Metric Filter**(로그에서 "ERROR" 발생 횟수 등)를 지표로 삼아 만들 수도 있다.

### EC2 Instance Recovery (자주 나옴)

상태 점검 종류를 구분해야 한다.
- **Instance status** — EC2 VM(소프트웨어) 점검
- **System status** — 밑단 하드웨어 점검
- **Attached EBS status** — 붙은 EBS 볼륨 점검

`StatusCheckFailed_System` 경보로 EC2 Instance Recovery를 걸면, 하드웨어 장애 시 인스턴스를 같은 사설 IP·퍼블릭 IP·Elastic IP·메타데이터·배치 그룹으로 복구한다. "하드웨어 장애에서 IP를 그대로 유지하며 자동 복구" 시나리오의 답.

### CloudWatch Network Synthetic Monitor

AWS의 앱과 온프레미스 데이터센터 사이 네트워크 문제(패킷 손실·지연·jitter)를 탐지한다. **에이전트 설치가 필요 없고**, Direct Connect나 S2S VPN을 거쳐 ICMP/TCP로 테스트한다.

### EventBridge (옛 CloudWatch Events)

두 가지 트리거 방식: **스케줄(cron)**과 **이벤트 패턴**(예: IAM 루트 사용자 로그인, S3 객체 업로드, CodeBuild 실패 등에 반응). 대상은 Lambda·SQS·SNS·Step Functions·ECS Task 등 다양하다.

- **Event Bus** 종류: Default(AWS 서비스) · Partner(Zendesk·Datadog 등 SaaS) · Custom(자체 앱). 이벤트 버스는 Resource-based Policy로 다른 계정·리전에서 접근하게 할 수 있어, 조직 전체 이벤트를 한 계정/리전으로 모으는 데 쓴다.
- 이벤트를 **archive**했다가 **replay**할 수 있다.
- **CloudTrail + EventBridge** 조합이 시험에 자주 나온다. CloudTrail이 기록한 API 호출(예: DynamoDB `DeleteTable`, 보안 그룹 인바운드 규칙 수정 `AuthorizeSecurityGroupIngress`)을 EventBridge가 잡아 SNS로 알린다. "특정 위험한 API 호출을 감지해 알림" → CloudTrail로 호출을 잡고 EventBridge로 라우팅.

### CloudWatch Insights 계열 (이름·용도만 매칭)

- **Container Insights** — ECS·EKS·Fargate·EC2 위 쿠버네티스의 지표·로그. EKS는 컨테이너화된 에이전트 필요.
- **Lambda Insights** — 서버리스 앱 트러블슈팅. Lambda Layer로 제공.
- **Contributor Insights** — 로그에서 "Top-N 기여자"(예: 트래픽이 가장 많은 IP, 에러를 가장 많이 내는 URL)를 찾음.
- **Application Insights** — 앱과 연관 리소스 문제를 자동 대시보드로. 결과는 EventBridge·SSM OpsCenter로 전송.

### CloudTrail

거버넌스·감사·규정 준수용. **기본으로 켜져 있고**, 계정 안의 모든 API 호출(콘솔·SDK·CLI·AWS 서비스) 이력을 남긴다. CloudWatch Logs나 S3로 보낼 수 있다. Trail은 **기본이 전체 리전**, 단일 리전으로 좁힐 수도 있다. "리소스가 누가 지웠는지 모르게 사라졌다" → CloudTrail부터 확인.

이벤트 종류 구분이 출제 포인트:
- **Management Events** — 리소스에 대한 설정 작업(IAM `AttachRolePolicy`, EC2 `CreateSubnet` 등). **기본으로 기록됨.** 리소스를 바꾸지 않는 Read와 바꾸는 Write로 나눌 수 있다.
- **Data Events** — S3 객체 단위 작업(`GetObject`·`PutObject`·`DeleteObject`), Lambda `Invoke`. 양이 많아 **기본으로는 기록 안 됨**. S3 객체 접근을 감사하려면 따로 켜야 한다.
- **CloudTrail Insights Events** — 비정상 활동(과도한 리소스 프로비저닝, 서비스 한도 도달, IAM 작업 폭주 등) 탐지. 평소 management 이벤트로 기준선을 만든 뒤 write 이벤트를 계속 분석해 이상치를 잡고, S3·EventBridge로 내보낸다.

보관: CloudTrail 안에서는 **90일**. 더 오래 보관·분석하려면 S3로 보내 Athena로 쿼리한다. "90일 넘는 감사 로그 보관/장기 분석" → S3 + Athena.

### AWS Config

리소스 설정의 준수 여부와 변경 이력을 기록한다. "SSH가 아무 데서나 열려 있나?", "버킷이 퍼블릭인가?", "ALB 설정이 시간에 따라 어떻게 바뀌었나?" 같은 질문에 답한다.

- **리전 단위 서비스.** 여러 리전·계정으로 결과를 모을(aggregate) 수 있다. 설정 데이터를 S3에 저장해 Athena로 분석 가능.
- **Config Rules** — AWS 관리형 규칙(75개 이상)이나 Lambda로 짠 커스텀 규칙. 설정이 바뀔 때마다 또는 정기적으로 평가. **규칙은 행동을 막지 못한다(deny 없음)** — 비준수를 탐지·기록할 뿐. "위반을 사전에 차단"하려는 문제면 Config가 아니라 IAM/SCP 쪽. 무료 티어 없음.
- **Remediation** — 비준수 리소스를 SSM Automation Document로 자동 교정(예: 미사용 IAM 액세스 키 비활성화). 재시도 횟수 설정 가능.
- **Notification** — 비준수를 EventBridge로 라우팅하거나 SNS로 설정 변경·준수 상태 알림.

### CloudWatch vs CloudTrail vs Config (한 줄 정리)

- **CloudWatch** — 성능 지표·대시보드, 이벤트·알림, 로그 집계/분석.
- **CloudTrail** — 계정 안 모든 사람의 API 호출 기록. 특정 리소스 trail 정의 가능. **글로벌 서비스.**
- **Config** — 설정 변경 기록, 규칙으로 준수 평가, 변경·준수 타임라인.

예: 로드 밸런서 하나를 두고 — CloudWatch는 연결 수·에러율 모니터링, Config는 SG 규칙·SSL 인증서 항상 붙어 있는지(준수) 추적, CloudTrail은 누가 LB를 바꿨는지 추적.

## 개발 관점 (DVA) #dva

![[AWS Certified Developer Slides v44.pdf#page=469]]

DVA는 관측성을 **코드·SDK로 어떻게 다루느냐**와, 분산 앱을 디버깅하는 **X-Ray**를 묻는다(트러블슈팅 도메인의 중심). 설계 관점에서 다룬 지표·로그·경보·CloudTrail은 반복하지 않고, 개발자가 직접 만지는 부분만 더한다.

### CloudWatch — 개발자가 만지는 부분

- **Custom Metrics (`PutMetricData`)**: EC2 메모리·디스크·로그인 사용자 수처럼 기본에 없는 지표를 직접 올린다. dimension으로 세분(Instance.id, Environment.name). **StorageResolution**: Standard 60초 / High Resolution 1·5·10·30초(고비용). **중요: 과거 2주 ~ 미래 2시간 범위의 데이터만 받으므로 EC2 시계가 맞아야 한다.**
- **EC2 Detailed Monitoring**: 기본 지표는 5분 간격, **Detailed Monitoring(유료)을 켜면 1분 간격**. ASG를 더 빨리 확장하고 싶을 때. 프리 티어 10개.
- **Metric Filter**: 로그에서 패턴(예: "ERROR" 횟수, 특정 IP)을 지표로 만든다. **소급 적용 안 됨**(필터 만든 뒤 발생한 이벤트만 집계). 최대 3 dimension.
- **경보 테스트**: `aws cloudwatch set-alarm-state --alarm-name ... --state-value ALARM`로 경보·알림을 강제로 발동해 테스트한다.
- **Synthetics Canary**: API·URL·웹사이트를 주기적으로 호출해 **고객보다 먼저 문제를 잡는다**. Node.js·Python으로 짜고 headless Chrome으로 동작. 청사진: Heartbeat Monitor, API Canary, Broken Link Checker, Visual Monitoring, Canary Recorder, GUI Workflow Builder. (Route 53 장애 조치 같은 자동 대응과 엮인다.)
- **EventBridge Schema Registry**: 이벤트 버스의 이벤트를 분석해 **스키마를 추론**하고, 그 구조에 맞는 **코드 바인딩을 생성**(버전 관리)해 애플리케이션이 이벤트 형식을 미리 알게 한다.

### AWS X-Ray — 분산 추적(distributed tracing)

마이크로서비스는 로그만으로 디버깅하기 어렵다. X-Ray는 요청 하나가 거쳐 간 서비스들을 **service map**으로 그려, 어디서 느려지고 어디서 에러가 나는지 시각적으로 보여준다.

**개념(용어 구분이 출제 포인트)**

- **Segment**: 각 앱/서비스가 보내는 추적 단위. **Subsegment**: 더 자세한 내부 단위.
- **Trace**: segment들을 모아 만든 end-to-end 경로.
- **Sampling**: X-Ray로 보내는 요청 수를 줄여 비용을 낮춘다.
- **Annotation**: 키-값. **인덱스되어 필터로 검색 가능**. ↔ **Metadata**: 키-값이지만 **인덱스 안 됨(검색 불가)**, 부가 정보 보관용.

**활성화 방법**

1. 코드(Java·Python·Go·Node.js·.NET)에 **X-Ray SDK**를 넣는다(설정 변경 수준). SDK가 AWS 호출·HTTP·DB(MySQL·PostgreSQL·DynamoDB)·SQS 호출을 캡처.
2. **X-Ray 데몬**을 설치하거나 AWS 통합을 켠다. 데몬은 UDP 패킷을 받아 1초마다 X-Ray로 배치 전송. **앱에 X-Ray 쓰기 IAM 권한이 있어야 한다.**

**Sampling Rules** (코드 수정 없이 변경 가능)

- 기본: **초당 첫 요청 1건(reservoir) + 그 이상 요청의 5%(rate)**.
- 커스텀 규칙으로 reservoir·rate를 조정(예: 특정 URL만 전부 추적하는 디버깅 규칙).

**API** — 데몬이 쓰는 것

- 쓰기: **`PutTraceSegments`**(세그먼트 업로드), `PutTelemetryRecords`, `GetSamplingRules`. 관리형 정책 **`AWSXrayWriteOnlyAccess`**.
- 읽기: `BatchGetTraces`, `GetServiceGraph`, `GetTraceSummaries`(필터로 trace ID·annotation 조회 → 전체는 BatchGetTraces로), `GetTraceGraph`.

**통합과 트러블슈팅**

- 호환: Lambda·Elastic Beanstalk·ECS·ELB·API Gateway·EC2·온프레미스.
- **EC2에서 안 될 때**: EC2 IAM 역할 권한 확인 + **X-Ray 데몬이 돌고 있는지** 확인.
- **Lambda에서 켜기**: 실행 역할에 `AWSX-RayWriteOnlyAccess`, 코드에 X-Ray import, **Active Tracing 활성화**.
- **Beanstalk**: 플랫폼에 데몬 포함, `XRayEnabled: true` 또는 `.ebextensions/xray-daemon.config`. (Multicontainer Docker엔 데몬 미제공.)
- **ECS**: X-Ray 컨테이너를 **Daemon**으로 띄우거나 **Sidecar**로. **Fargate는 Sidecar만** 가능.

### AWS Distro for OpenTelemetry (ADOT)

오픈소스 OpenTelemetry의 AWS 배포판. **코드 변경 없이 auto-instrumentation 에이전트**로 추적·지표를 수집해 **X-Ray·CloudWatch·Prometheus·파트너 도구로 동시에** 보낸다. **오픈소스 표준으로 통일**하거나 **추적을 여러 목적지에 동시 전송**하고 싶으면 X-Ray에서 ADOT로 옮긴다.

### CloudTrail + EventBridge (트러블슈팅 패턴)

위험한 API 호출을 감지해 알린다: 사용자가 `DeleteTable`·`AuthorizeSecurityGroupIngress` 같은 호출 → CloudTrail이 기록 → EventBridge 규칙이 잡아 → SNS로 알림. (CloudTrail 이벤트 종류·90일 보관·S3+Athena는 설계 관점 참고.)

### CloudTrail vs CloudWatch vs X-Ray (한 줄)

- **CloudTrail** — 누가 어떤 API를 호출했나(감사, 무단 호출·변경 원인).
- **CloudWatch** — 지표(모니터링)·로그(저장)·경보(알림).
- **X-Ray** — 분산 시스템에서 요청을 추적, 지연·에러·병목 분석, service map.

## 운영 관점 (CloudOps) #cloudops

![[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=374]]

CloudOps는 같은 서비스를 더 깊게 운영하는 시험이라, 여기서는 설계·개발 관점에서 이미 다룬 지표·로그·경보·CloudTrail 이벤트 종류·Config 기본은 반복하지 않는다. 운영 시험에만 새로 나오거나 깊어지는 부분만 모은다. 도메인이 모니터링 22% · 신뢰성 22% · 배포 자동화 22%로 고르게 퍼져 있으니, 이 세 서비스가 자동 대응(EventBridge → SSM/Lambda)과 어떻게 엮이는지가 핵심이다.

### 모니터링 깊이 — 새로 나오는 것

- **CloudWatch Anomaly Detection** — 정적 임계값(static threshold) 대신, 지표의 과거 데이터로 정상 범위를 학습한 모델을 만들어 그 밖으로 벗어나면 경보를 울린다. CPU처럼 시간대별로 오르내리는 지표에 고정 임계값을 걸기 애매할 때 답이 된다. 특정 기간·이벤트를 학습에서 제외할 수도 있다.
- **CloudWatch Logs Data Protection** — 로그에 섞여 들어온 민감정보(이메일·비밀번호·카드번호·주민번호 등)를 ML로 찾아 마스킹한다. Data Protection Policy에 Data Identifier를 지정(100종 이상 기본 제공 + 커스텀). 마스킹은 Logs Insights·Metric Filter·Subscription Filter에서 적용되고, **`logs:Unmask` 권한이 있는 사용자만 원본을 볼 수 있다.** 민감정보가 탐지되면 **`LogEventsWithFindings`** 지표로 알림을 걸 수 있다.
- **CloudWatch Internet Monitor vs Network Synthetic Monitor** — 둘 다 에이전트가 필요 없지만 보는 곳이 다르다. **Internet Monitor**는 AWS 위 앱과 **인터넷 최종 사용자**(도시·통신망 ASN·클라이언트 위치) 사이 문제를 AWS 글로벌 네트워크 데이터로 본다. **Network Synthetic Monitor**는 AWS와 **온프레미스 데이터센터** 사이를 Direct Connect/S2S VPN으로 ICMP·TCP 테스트한다. "최종 사용자 체감 지연" → Internet Monitor, "DC 사이 패킷 손실" → Network Synthetic Monitor.
- **Synthetics Canary in a VPC** — VPC 안 엔드포인트도 카나리로 감시할 수 있는데, **VPC에 DNS Resolution과 DNS Hostnames가 켜져 있어야 한다.** 카나리가 CloudWatch로 지표를 보내는 경로는 둘 중 하나다 — 퍼블릭 인터넷이면 NAT Gateway 경유, 내부망으로만 보내려면 CloudWatch용 VPC(Interface) Endpoint 경유.
- **Container Insights Enhanced Visibility** — 기본은 클러스터·서비스 단위 지표만 준다. **Enhanced Visibility**를 켜야 task·container 단위까지 내려간다. 컨테이너 하나의 과도한 리소스 사용·스로틀링을 추적해야 하면 이걸 켠다. (ECS·Fargate·EKS·ROSA 지원, sidecar 불필요.)

### 신뢰성·운영 자동화 — 한도 감시와 이벤트 자동 대응

- **Service Quotas CloudWatch Alarms** — 서비스 한도(예: Lambda 동시 실행 수)에 가까워지면 알림. 한도 증가 요청을 미리 넣거나 리소스를 줄이게 한다. 대안으로 **Trusted Advisor**도 Service Limits 체크(약 50개) 결과를 CloudWatch로 보내 경보를 걸 수 있다. "한도에 닿기 전에 알림" 시나리오의 답.
- **EventBridge Pipes** — 소스(DynamoDB/Kinesis Stream·SQS·MQ·MSK·Kafka) 하나를 타깃 하나로 잇는 **노코드 1:1 통합**. 중간에 Filter로 거르고 Enrichment(Lambda·Step Functions·API Gateway)로 가공한다. 소스를 여러 대상에 뿌리는 이벤트 버스와 달리, 한 소스→한 타깃 파이프라인일 때 쓴다.
- **EventBridge Retries & DLQ** — 대상이 안 떠 있거나 네트워크 문제로 전달이 실패하면 **Retry Policy**(기본 최대 24시간·185회)로 재시도하고, 끝내 못 보낸 이벤트는 **SQS Dead Letter Queue**로 보내 나중에 처리한다.
- **EventBridge → SSM Automation** — EventBridge 대상으로 SSM Automation Document를 걸어, 스케줄이나 특정 이벤트(예: EC2 상태 변경)에 자동 실행(부트스트랩·교정)을 건다.
- **EventBridge Cross-account Targets** — 다른 계정으로 이벤트를 보낼 때, 대상이 **이벤트 버스면 양쪽 다 Resource-based Policy**, 대상이 **SQS·SNS·Lambda·API Gateway·KDS 같은 일반 서비스면 보내는 쪽 Execution IAM Role + 받는 쪽 Resource-based Policy**가 필요하다. 조직 전체 이벤트를 한 계정으로 모을 때 쓴다.

### 감사 무결성과 멀티 계정

- **CloudTrail Log File Integrity Validation** — CloudTrail이 S3에 넣은 로그가 전달 후 **변조·삭제됐는지** 확인하는 기능. 매시간 **Digest File**(지난 한 시간 로그들의 해시 목록)을 같은 버킷 다른 폴더에 남긴다. 해시는 SHA-256, 서명은 SHA-256 with RSA. 로그 버킷 자체는 버킷 정책·버저닝·MFA Delete·암호화·Object Lock으로 보호한다. "감사 로그가 위변조되지 않았음을 증명" → 이 기능.
- **CloudTrail은 실시간이 아니다** — API 호출 후 **이벤트는 15분 이내**, **S3로 로그 파일은 5분마다** 전달. EventBridge로 API 호출에 반응하는 자동화를 짤 때 이 지연을 감안한다.
- **CloudTrail Organizations Trail** — Organization 전체 계정의 이벤트를 한 trail로 남긴다. 같은 이름의 trail이 모든 계정에 생기고, **멤버 계정은 이 trail을 지우거나 바꿀 수 없다(보기만)**. 중앙 감사용.

### 준수 자동 교정 (Config Remediation)

설계 관점에서 다룬 Config Rules·Remediation을 운영 시각에서 더 본다.

- **Remediation** — 비준수 리소스를 SSM Automation Document로 자동 교정. AWS 관리형 문서를 쓰거나 커스텀(Lambda 호출도 가능). 교정 후에도 여전히 비준수면 **Remediation Retries**(예: 5회)로 재시도. 자주 나오는 관리형 문서: `AWS-DisableIncomingSSHOnPort22`(SSH 포트 닫기), `AWS-ConfigureS3BucketLogging`(버킷 로깅 켜기), `AWSConfigRemediation-RevokeUnusedIAMUserCredentials`(미사용 키 비활성화).
- **Config Aggregator** — 중앙 한 계정(aggregator 계정)에서 여러 계정·리전의 규칙·준수 상태를 한눈에 모아 본다. **Organizations를 쓰면 계정별 개별 승인(Authorization)이 필요 없다.** 규칙 자체는 각 소스 계정에 만들어야 하고, 여러 계정에 규칙을 한 번에 배포하려면 **CloudFormation StackSets**를 쓴다.
- **알림** — 비준수를 EventBridge로 보내 Lambda·SNS·SQS로 자동 대응하거나, SNS로 설정 변경·준수 상태를 통지(전체 이벤트가 오므로 SNS Filtering이나 클라이언트단 필터로 걸러야 한다).

## 시험 함정

- EC2 메모리(RAM)·디스크 사용량은 CloudWatch 기본 지표에 없다. Unified Agent나 Custom Metric으로 올려야 한다. #exam/trap/monitoring
- 로그를 S3로 보낼 때 S3 Export는 최대 12시간 지연·비실시간, 실시간이면 Logs Subscriptions. #exam/trap/monitoring
- CloudWatch Logs Insights는 쿼리 엔진이지 실시간 처리 엔진이 아니다. #exam/trap/monitoring
- CloudTrail Management Events는 기본 기록, Data Events(S3 객체·Lambda Invoke)는 기본 미기록 — 따로 켜야 한다. #exam/trap/monitoring
- CloudTrail 보관은 90일, 장기 보관·분석은 S3 + Athena. #exam/trap/monitoring
- AWS Config Rules는 행동을 막지 못한다(deny 없음). 사전 차단은 IAM/SCP 몫. #exam/trap/monitoring
- CloudTrail은 글로벌, AWS Config는 리전 단위 서비스. #exam/trap/monitoring
- X-Ray **Annotation은 인덱스되어 필터 검색 가능, Metadata는 인덱스 안 됨**(검색 불가). #exam/trap/monitoring
- X-Ray가 EC2에서 안 됨 → IAM 역할 권한 + **X-Ray 데몬 실행** 확인. Lambda는 실행 역할 `AWSX-RayWriteOnlyAccess` + **Active Tracing**. #exam/trap/monitoring
- X-Ray Sampling 기본값: 초당 첫 1건(reservoir) + 추가분 5%(rate). 코드 수정 없이 규칙 변경. #exam/trap/monitoring
- Fargate에서 X-Ray는 **Sidecar 컨테이너만**(Daemon 불가). #exam/trap/monitoring
- `PutMetricData`는 과거 2주~미래 2시간만 허용 — EC2 시계가 틀리면 거부된다. #exam/trap/monitoring
- Metric Filter는 **소급 적용 안 됨**(만든 뒤 이벤트만). EC2 1분 지표는 Detailed Monitoring. #exam/trap/monitoring
- 오픈소스 표준 통일·다중 목적지 전송이 필요하면 X-Ray → **ADOT(AWS Distro for OpenTelemetry)**. #exam/trap/monitoring
- 시간대별로 오르내리는 지표에 고정 임계값이 애매하면 정적 임계값 대신 **Anomaly Detection**(학습된 정상 범위 기반 경보). #exam/trap/monitoring
- 로그 민감정보 마스킹은 Logs **Data Protection Policy**, 원본 열람은 **`logs:Unmask`** 권한자만. 탐지 알림은 `LogEventsWithFindings` 지표. #exam/trap/security
- **Internet Monitor**(AWS↔인터넷 최종 사용자)와 **Network Synthetic Monitor**(AWS↔온프레미스 DC, DX/VPN) 혼동 주의. 둘 다 에이전트 불필요. #exam/trap/monitoring
- VPC 안 엔드포인트에 Synthetics Canary를 걸려면 VPC에 **DNS Resolution + DNS Hostnames** 활성화 필수. #exam/trap/monitoring
- Container Insights는 기본이 클러스터·서비스 단위, task·container 단위까지 보려면 **Enhanced Visibility**. #exam/trap/monitoring
- 서비스 한도 근접 알림 → **Service Quotas CloudWatch Alarm** 또는 **Trusted Advisor + CW Alarm**(Service Limits 체크 약 50개). #exam/trap/reliability
- EventBridge 전달 실패: **Retry Policy 기본 24시간·185회**, 끝내 실패하면 **SQS DLQ**. #exam/trap/reliability
- EventBridge Cross-account: 대상이 이벤트 버스면 **양쪽 Resource-based Policy**, 일반 서비스(SQS·SNS·Lambda 등)면 **보내는 쪽 Execution Role + 받는 쪽 Resource Policy**. #exam/trap/reliability
- 감사 로그 위변조 증명 → CloudTrail **Log File Integrity Validation**(Digest File, SHA-256 / SHA-256 with RSA). #exam/trap/security
- CloudTrail은 실시간 아님 — **이벤트 15분 이내, S3 로그 파일 5분마다**. EventBridge 자동화 설계 시 감안. #exam/trap/monitoring
- Organizations Trail은 멤버 계정이 **수정·삭제 불가(보기만)**. #exam/trap/security
- Config Aggregator는 **Organizations 사용 시 계정별 개별 승인 불필요**, 규칙 다계정 배포는 **CloudFormation StackSets**. #exam/trap/reliability
- Config 자동 교정은 **SSM Automation Document**로, 실패 시 **Remediation Retries**. (예: `AWS-DisableIncomingSSHOnPort22`) #exam/trap/reliability

## 관련 노트

[[Systems Manager]] · [[ELB & Auto Scaling]] · [[기타 서비스]] · [[SQS & SNS & Kinesis]] · [[S3]] · [[KMS & 암호화]] · [[Organizations & 계정 관리]] · [[CloudFormation]]
