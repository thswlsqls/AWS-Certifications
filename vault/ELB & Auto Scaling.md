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
status: learning
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

%% 이 시험의 관점에서 배운 내용을 정리합니다. %%

## 시험 함정

- 고정 IP가 필요하면 NLB(+Elastic IP)다. ALB는 고정 호스트네임만 있고 고정 IP가 없다. #exam/trap/elb-asg
- 고정 IP와 L7 라우팅이 둘 다 필요하면 NLB 뒤에 ALB를 체이닝한다. #exam/trap/elb-asg
- 백엔드에서 클라이언트의 실제 IP는 `X-Forwarded-For` 헤더로 받는다. #exam/trap/elb-asg
- SNI는 ALB·NLB·CloudFront만 지원한다. CLB는 인증서 1개뿐이다. #exam/trap/elb-asg
- Cross-Zone LB 기본값: ALB는 켜짐(무료), NLB·GWLB는 꺼짐(켜면 유료). #exam/trap/elb-asg
- GWLB가 보이면 GENEVE 프로토콜, 포트 6081을 떠올린다. 서드파티 보안 어플라이언스 문제다. #exam/trap/elb-asg
- Sticky session은 부하 쏠림을 만들 수 있다. 분산이 안 된다는 증상의 원인으로 출제된다. #exam/trap/elb-asg
- ASG의 스케일링 지표는 인스턴스 하나가 아니라 그룹 전체 평균이다. #exam/trap/elb-asg

%% 연습문제에서 틀리거나 헷갈린 지점을 위 형식으로 계속 추가합니다. 시험 직전 주에 `tag:#exam/trap` 검색으로 한 번에 모아 봅니다. %%

## 관련 노트

[[EC2]] · [[Route 53]] · [[CloudWatch & CloudTrail & Config]] · [[컨테이너 서비스]] · [[KMS & 암호화]]
