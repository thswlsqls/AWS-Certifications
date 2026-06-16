---
service: Amazon CloudWatch · CloudTrail · AWS Config · X-Ray
exams: [saa, dva, cloudops]
domains:
  - saa/resilient-architectures
  - dva/troubleshooting
  - cloudops/monitoring-logging
status: learning
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

%% 이 시험의 관점에서 배운 내용을 정리합니다. %%

## 운영 관점 (CloudOps) #cloudops

![[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=374]]

%% 이 시험의 관점에서 배운 내용을 정리합니다. %%

## 시험 함정

- EC2 메모리(RAM)·디스크 사용량은 CloudWatch 기본 지표에 없다. Unified Agent나 Custom Metric으로 올려야 한다. #exam/trap/monitoring
- 로그를 S3로 보낼 때 S3 Export는 최대 12시간 지연·비실시간, 실시간이면 Logs Subscriptions. #exam/trap/monitoring
- CloudWatch Logs Insights는 쿼리 엔진이지 실시간 처리 엔진이 아니다. #exam/trap/monitoring
- CloudTrail Management Events는 기본 기록, Data Events(S3 객체·Lambda Invoke)는 기본 미기록 — 따로 켜야 한다. #exam/trap/monitoring
- CloudTrail 보관은 90일, 장기 보관·분석은 S3 + Athena. #exam/trap/monitoring
- AWS Config Rules는 행동을 막지 못한다(deny 없음). 사전 차단은 IAM/SCP 몫. #exam/trap/monitoring
- CloudTrail은 글로벌, AWS Config는 리전 단위 서비스. #exam/trap/monitoring

## 관련 노트

[[Systems Manager]] · [[ELB & Auto Scaling]] · [[기타 서비스]] · [[SQS & SNS & Kinesis]] · [[S3]] · [[KMS & 암호화]] · [[Organizations & 계정 관리]]
