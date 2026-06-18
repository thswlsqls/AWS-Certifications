---
service: Elastic Load Balancing · Auto Scaling Group
exams:
  - saa
  - dva
  - cloudops
domains:
  - saa/resilient-architectures
  - saa/high-performing-architectures
  - dva/development
  - cloudops/reliability
status: reviewing
confidence: 1
tags:
  - service
  - saa/compute
  - dva/compute
  - cloudops/compute
---

# ELB & Auto Scaling

> 고가용성과 확장성의 핵심 — ALB/NLB/GWLB, 대상 그룹, ASG 정책.

## 개요

확장에는 두 방향이 있다. **수직 확장(scale up)** 은 인스턴스를 더 큰 걸로 바꾸는 것 — 분산이 어려운 DB(RDS, ElastiCache)가 주로 쓰고, 하드웨어 한계가 있다. **수평 확장(scale out)** 은 인스턴스 수를 늘리는 것 — 웹 애플리케이션의 기본이고, 이를 자동화하는 게 ASG와 ELB다.

**고가용성(HA)** 은 확장과는 다른 개념으로, 같은 애플리케이션을 **최소 2개 AZ**에 걸쳐 돌려 데이터센터 하나가 죽어도 살아남는 것을 말한다. ELB와 ASG 둘 다 multi-AZ로 구성할 수 있어서, 이 둘의 조합이 AWS에서 고가용성을 만드는 기본 패턴이다.

- **ELB** — AWS가 관리하는 로드 밸런서. 여러 인스턴스에 트래픽을 나누고, health check로 죽은 인스턴스를 빼고, 단일 DNS 진입점을 제공한다.
- **ASG** — 부하에 따라 인스턴스를 늘리고(scale out) 줄이는(scale in) 그룹. ASG 자체는 무료고 띄운 인스턴스 값만 낸다.

## 설계 관점 (SAA) #saa

![[AWS Certified Solutions Architect Slides v47.pdf#page=118]]

### Health Check

LB는 포트 + 경로(`/health`가 관례)로 인스턴스 상태를 확인하고, 응답이 200이 아니면 unhealthy로 보고 트래픽을 보내지 않는다. ALB·NLB에서 health check는 **target group 단위**다.

### 로드 밸런서 4종 — 어느 것을 고르는가

| 종류 | 계층 | 프로토콜 | 이런 문제의 답 |
| --- | --- | --- | --- |
| CLB (v1) | L4+L7 | HTTP·HTTPS·TCP | 구세대. 신규 설계의 답이 되는 일은 없음 |
| ALB (v2) | L7 | HTTP·HTTPS·WebSocket | HTTP 라우팅, 마이크로서비스·컨테이너(ECS) |
| NLB (v2) | L4 | TCP·UDP·TLS | 초고성능(수백만 req/s), **고정 IP가 필요할 때** |
| GWLB | L3 | IP (GENEVE, 포트 6081) | 서드파티 보안 어플라이언스(방화벽·IDS/IPS)를 트래픽 경로에 끼울 때 |

### ALB

- 라우팅 규칙으로 여러 target group에 나눈다: URL 경로(`/users` vs `/posts`), 호스트네임(`one.example.com`), 쿼리 스트링·헤더(`?Platform=Mobile`).
- 대상: EC2(ASG 연동), ECS task, Lambda, 사설 IP. 컨테이너의 동적 포트 매핑을 지원해서 ECS와 궁합이 좋다 — CLB라면 애플리케이션마다 LB를 따로 둬야 했다.
- 호스트네임은 고정이지만 **IP는 고정이 아니다.**
- LB가 연결을 종료하고 다시 맺기 때문에 백엔드는 클라이언트 IP를 직접 못 본다. 실제 IP는 **`X-Forwarded-For`** 헤더에 들어온다(포트는 `X-Forwarded-Port`, 프로토콜은 `X-Forwarded-Proto`).

### NLB

- **AZ당 고정 IP 하나**를 갖고 Elastic IP를 붙일 수 있다. "방화벽에 등록할 고정 IP가 필요하다"는 문제의 답.
- 대상: EC2, 사설 IP, 그리고 **ALB**. NLB 뒤에 ALB를 두면 고정 IP + L7 라우팅을 같이 얻는다.
- health check는 TCP·HTTP·HTTPS를 지원한다.

### LB의 Security Group 패턴

LB의 SG는 80·443을 전체에 열고, 백엔드 인스턴스의 SG는 소스를 **LB의 SG로 참조**해서 LB를 거친 트래픽만 받는다. 사용자가 인스턴스에 직접 못 닿게 하는 표준 구성.

### Sticky Session

같은 클라이언트를 항상 같은 인스턴스로 보내는 기능. CLB·ALB·NLB가 지원한다. 쿠키 기반이며 두 종류다 — LB가 만들어주는 duration-based 쿠키(ALB는 `AWSALB`), 애플리케이션이 만드는 custom 쿠키(예약된 이름 `AWSALB*`는 사용 불가). 세션 데이터를 잃으면 안 될 때 쓰지만, **부하가 한쪽으로 쏠릴 수 있다.**

### Cross-Zone Load Balancing

켜면 모든 AZ의 인스턴스에 고르게 분산하고, 끄면 각 LB 노드가 자기 AZ 안에서만 분산한다. **기본값과 과금이 LB마다 달라서 출제 포인트다.**

| LB | 기본값 | AZ 간 데이터 요금 |
| --- | --- | --- |
| ALB | **켜짐** | 무료 |
| NLB·GWLB | 꺼짐 | 켜면 유료 |
| CLB | 꺼짐 | 켜도 무료 |

### SSL 인증서와 SNI

- 인증서는 ACM으로 관리하거나 직접 올린다. HTTPS listener에는 기본 인증서를 하나 지정하고, 여러 도메인을 받으려면 인증서를 추가한다.
- **SNI**: 클라이언트가 SSL 핸드셰이크에서 접속하려는 호스트네임을 밝히면 서버가 맞는 인증서를 골라주는 방식. 이 덕에 LB 하나가 인증서 여러 개를 들 수 있다. **ALB·NLB·CloudFront만 지원, CLB는 안 된다** — CLB는 인증서 1개라 도메인마다 CLB를 따로 둬야 한다.

### Connection Draining / Deregistration Delay

인스턴스가 빠질 때(deregister) 진행 중인 요청이 끝나길 기다려주는 시간. CLB에서는 Connection Draining, ALB·NLB에서는 Deregistration Delay라 부른다. 1~3600초, **기본 300초**, 0이면 끈다. 요청이 짧은 서비스면 낮게 잡는다.

### ASG

- **min / desired / max** 용량을 정해두면 그 범위 안에서 인스턴스 수를 조절하고, unhealthy 인스턴스(ELB health check 연동 가능)를 종료하고 새로 띄운다.
- 인스턴스를 어떻게 만들지는 **Launch Template**에 담는다: AMI, 인스턴스 타입, User Data, EBS, SG, IAM Role 등. (구식 Launch Configuration은 deprecated.)
- 스케일링 판단 지표(CloudWatch)는 **그룹 전체의 평균**으로 계산된다. 잘 쓰는 지표: CPUUtilization, RequestCountPerTarget(인스턴스당 요청 수), Network In/Out.

### Scaling 정책 4가지

| 정책 | 동작 | 언제 |
| --- | --- | --- |
| Target Tracking | "평균 CPU를 40%로 유지" 식으로 목표값만 지정 | 가장 간단한 기본 선택 |
| Simple / Step | CloudWatch 알람이 울리면 N대 추가·제거 | 단계별로 세밀하게 제어할 때 |
| Scheduled | 정해진 시각에 용량 변경 | 부하 패턴을 미리 알 때(금요일 17시 등) |
| Predictive | 과거 부하를 분석해 미리 스케일 | 주기적 패턴을 자동으로 따라갈 때 |

스케일링 후에는 **cooldown(기본 300초)** 동안 추가 스케일링을 멈추고 지표가 안정되길 기다린다. 설정이 미리 구워진 AMI를 쓰면 기동이 빨라져 cooldown을 줄일 수 있다.

## 개발 관점 (DVA) #dva

![[AWS Certified Developer Slides v44.pdf#page=95]]

%% 이 시험의 관점에서 배운 내용을 정리합니다. %%

## 운영 관점 (CloudOps) #cloudops

![[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=71]]

설계(SAA) 섹션이 "어떤 LB·어떤 스케일링을 고르나"였다면, 운영 관점은 **돌고 있는 LB·ASG를 어떻게 모니터링하고, 장애를 진단하고, 무중단으로 갱신·복구하느냐**다. (LB 4종·target group·sticky session·cross-zone·SNI·scaling 정책 기초는 SAA 섹션 참고.)

### ELB Health Check 상태와 설정

- 타겟 상태: **Initial**(등록 중)·Healthy·Unhealthy·**Unused**(미등록)·**Draining**(등록 해제 중)·**Unavailable**(health check 비활성).
- 설정값: `HealthCheckProtocol`·`Port`·`Path`, `HealthCheckTimeoutSeconds`(기본 5), `HealthCheckIntervalSeconds`(30), `HealthyThresholdCount`(3회 성공→healthy), `UnhealthyThresholdCount`(5회 실패→unhealthy).
- **타겟 그룹에 unhealthy 타겟만 남으면 ELB는 그 unhealthy 타겟들로라도 라우팅한다**(fail-open).

### LB 에러 코드와 트러블슈팅

- **4XX = 클라이언트 문제**: 400(Bad Request·malformed), 401/403, 460(클라이언트가 연결 끊음), **463(`X-Forwarded-For` 헤더에 IP 30개 초과)**.
- **5XX = 서버 문제**: **500(ELB 자체 에러)**, 502(Bad Gateway), **503(Service Unavailable)**, 504(Gateway Timeout·서버 쪽 문제), 561.
- 메트릭 기반 진단:
  - **HTTP 503** → 응답할 AZ마다 **healthy 인스턴스가 있는지** 확인(`HealthyHostCount`).
  - **HTTP 504** → EC2의 **keep-alive timeout이 LB의 idle timeout보다 큰지** 확인.
  - HTTP 400 → 클라이언트가 보낸 요청이 HTTP 규격에 안 맞음.

### LB 모니터링·로깅

- 모든 LB 메트릭은 **CloudWatch로 직접 push**된다: `BackendConnectionErrors`, `Healthy/UnHealthyHostCount`, `HTTPCode_Backend_2XX/3XX`, **`HTTPCode_ELB_4XX`(클라이언트)·`HTTPCode_ELB_5XX`(LB 생성 에러)**, `Latency`, `RequestCount`, `RequestCountPerTarget`, **`SurgeQueueLength`**(healthy 인스턴스로 라우팅 대기 중인 요청, 최대 **1024**, ASG scale out 판단에 유용), **`SpilloverCount`**(surge queue가 가득 차 거부된 요청).
- **Access Logs**: LB 접근 로그를 **S3에 저장**(시간·클라이언트 IP·지연·요청 경로·서버 응답·Trace ID). **S3 저장 비용만** 들고, **ELB·EC2가 종료된 뒤에도 데이터가 남아** 컴플라이언스·감사에 유용. **이미 암호화**되어 저장된다.
- **ALB Request Tracing**: HTTP 요청마다 **`X-Amzn-Trace-Id`** 헤더를 붙여 단일 요청을 로그·분산 추적에서 따라간다. **ALB는 아직 X-Ray와 통합되지 않는다.**

### Target Group 고급 설정

- `deregistration_delay.timeout_seconds`(Connection Draining), `load_balancing.algorithm.type`, `stickiness.*`.
- **Slow Start Mode**: 새로 등록된 healthy 타겟에 **warm-up 시간**을 줘서 요청을 **점진적으로(선형 증가)** 보낸다. duration이 지나거나 타겟이 unhealthy되면 종료. **0이면 비활성**(기본은 등록 즉시 전량 분배).
- **요청 라우팅 알고리즘**: **Least Outstanding Requests**(처리 중 요청이 가장 적은 타겟 — ALB·CLB HTTP/HTTPS), **Round Robin**(균등 — ALB·CLB TCP), **Flow Hash**(프로토콜·src/dst IP·포트·TCP seq 기반, 연결 단위로 한 타겟에 고정 — NLB).
- **ALB Listener Rules**: 위에서부터 순서대로 처리(+ Default Rule). 액션은 **forward·redirect·fixed-response**, 조건은 host-header·http-request-method·path-pattern·source-ip·http-header·query-string.
- **Target Group Weighting**: 룰 하나에서 타겟 그룹별 **가중치**를 줘 트래픽 비율을 나눈다(예: blue 80% / green 20% — **blue/green 배포**).

### ASG — 무중단 갱신과 복구

- **Scaling Cooldown**: 스케일링 직후 **기본 300초** 동안 추가 스케일링을 멈추고 지표가 안정되길 기다린다. **미리 구운 AMI**를 쓰면 기동이 빨라 cooldown을 줄일 수 있다.
- **Instance Refresh**: **Launch Template을 갱신한 뒤 전체 인스턴스를 다시 만드는** 기능(`StartInstanceRefresh`). **Minimum Healthy Percentage**로 한 번에 교체할 비율을 정하고, **warm-up time**으로 새 인스턴스가 준비될 시간을 준다.
- **Warm Pools**: 앱 부팅이 길어 **scale-out이 느린 문제**를 해결. **미리 초기화된 인스턴스 풀**을 두고, scale-out 때 새로 띄우는 대신 풀에서 꺼내 쓴다. 설정: Minimum warm pool size, Max prepared capacity(기본 = ASG max), **Warm Pool Instance State(Running/Stopped/Hibernated)**. **Warm Pool 인스턴스는 ASG 스케일링 정책 메트릭에 잡히지 않는다.**
- **Lifecycle Hooks**: 인스턴스가 서비스에 들어가기 전(**Pending:Wait**)·종료되기 전(**Terminating:Wait**)에 추가 작업을 끼운다(cleanup·로그 추출·특수 health check). EventBridge·SNS·SQS와 통합.
- **SQS + ASG**: SQS 큐 길이(**`ApproximateNumberOfMessages`**) CloudWatch 메트릭 → Alarm → ASG 스케일. 큐가 쌓이면 처리 인스턴스를 늘리는 패턴.

### ASG Health Check와 트러블슈팅

- HA = **최소 2 AZ에 2 인스턴스**(multi-AZ ASG 구성 필수). Health check: **EC2 Status Checks·ELB Health Checks·Custom Health Checks**(`set-instance-health`로 직접 보고).
- **ASG는 unhealthy 인스턴스를 재부팅하지 않고, 종료한 뒤 새로 띄운다.** CLI: `set-instance-health`, `terminate-instance-in-auto-scaling-group`.
- 트러블슈팅:
  - "instances already running, launching failed" → **MaximumCapacity 한계 도달** → max 용량을 올린다.
  - 인스턴스 시작 실패 → **SG가 삭제됨** 또는 **키 페어가 삭제됨**.
  - **24시간 넘게 시작에 실패하면 ASG가 프로세스를 자동 중단**한다(administration suspension).
- **CloudWatch Metrics for ASG**(1분 간격): **ASG-level(opt-in, 수집 켜야 함)** — `GroupMinSize/MaxSize/DesiredCapacity`, `GroupInService/Pending/Standby/Terminating/TotalInstances`. **EC2-level(기본 활성)** — CPU 등(Basic 5분 / Detailed 1분).

### AWS Auto Scaling (그룹 밖의 통합 서비스)

ASG뿐 아니라 **여러 확장 가능 리소스**를 한 서비스로 스케일: EC2 ASG·**Spot Fleet**·**ECS**(desired count)·**DynamoDB**(WCU/RCU)·**Aurora**(read replica). **Scaling Plans**의 Dynamic scaling은 target tracking으로 가용성 40% / 균형 50% / 비용 70% 목표를 고를 수 있고, Predictive scaling은 부하를 예측해 미리 스케일한다.

## 시험 함정

- 고정 IP가 필요하면 NLB(+Elastic IP)다. ALB는 고정 호스트네임만 있고 고정 IP가 없다. #exam/trap/elb-asg
- 고정 IP와 L7 라우팅이 둘 다 필요하면 NLB 뒤에 ALB를 체이닝한다. #exam/trap/elb-asg
- 백엔드에서 클라이언트의 실제 IP는 `X-Forwarded-For` 헤더로 받는다. #exam/trap/elb-asg
- SNI는 ALB·NLB·CloudFront만 지원한다. CLB는 인증서 1개뿐이다. #exam/trap/elb-asg
- Cross-Zone LB 기본값: ALB는 켜짐(무료), NLB·GWLB는 꺼짐(켜면 유료). #exam/trap/elb-asg
- GWLB가 보이면 GENEVE 프로토콜, 포트 6081을 떠올린다. 서드파티 보안 어플라이언스 문제다. #exam/trap/elb-asg
- Sticky session은 부하 쏠림을 만들 수 있다. 분산이 안 된다는 증상의 원인으로 출제된다. #exam/trap/elb-asg
- ASG의 스케일링 지표는 인스턴스 하나가 아니라 그룹 전체 평균이다. #exam/trap/elb-asg
- **HTTP 503**(Service Unavailable) → AZ마다 **healthy 인스턴스 있는지**(HealthyHostCount). **HTTP 504** → EC2 keep-alive timeout > LB idle timeout 확인. #exam/trap/elb-asg
- LB 접근 로그는 **S3**에 저장(이미 암호화), ELB·EC2가 사라져도 보존. ALB 요청 추적은 **`X-Amzn-Trace-Id`**(X-Ray 통합은 아직 없음). #exam/trap/elb-asg
- `SurgeQueueLength`(최대 1024) 쌓임·`SpilloverCount`(거부) → 스케일 아웃 신호. #exam/trap/elb-asg
- **ASG는 unhealthy 인스턴스를 재부팅하지 않고 종료 후 재생성**한다. #exam/trap/elb-asg
- Launch Template 갱신 후 전체 인스턴스 교체 → **Instance Refresh**(Min Healthy %). scale-out 지연 해소 → **Warm Pools**. #exam/trap/elb-asg
- 종료/시작 직전에 cleanup·로그 추출 → **Lifecycle Hooks**(Pending:Wait / Terminating:Wait). #exam/trap/elb-asg
- ASG가 **24시간 넘게 인스턴스 시작에 실패**하면 프로세스 자동 중단(administration suspension). 시작 실패 흔한 원인: SG·키페어 삭제, MaximumCapacity 도달. #exam/trap/elb-asg
- ASG-level 메트릭(GroupDesiredCapacity 등)은 **opt-in**(메트릭 수집을 켜야 보임). #exam/trap/elb-asg

%% 연습문제에서 틀리거나 헷갈린 지점을 위 형식으로 계속 추가합니다. 시험 직전 주에 `tag:#exam/trap` 검색으로 한 번에 모아 봅니다. %%

## 관련 노트

[[EC2]] · [[Route 53]] · [[CloudWatch & CloudTrail & Config]] · [[컨테이너 서비스]] · [[KMS & 암호화]] · [[SQS & SNS & Kinesis]] · [[S3]] · [[기타 서비스]]
