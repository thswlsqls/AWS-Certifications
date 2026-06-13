---
service: Amazon Route 53
exams: [saa, dva, cloudops]
domains:
  - saa/high-performing-architectures
  - dva/development
  - cloudops/networking
status: learning
confidence: 1
tags:
  - service
  - saa/network
  - dva/network
  - cloudops/network
---

# Route 53

> DNS와 라우팅 정책(단순·가중치·지연시간·장애조치·지리적) 관리 서비스.

## 개요

도메인 이름(www.example.com)을 IP 주소로 바꿔주는 DNS 서비스다. AWS의 관리형 DNS이면서 동시에 도메인을 직접 사는 등록 대행(Domain Registrar) 역할도 한다. "Authoritative DNS"라는 말은 도메인 주인인 내가 DNS 레코드를 직접 만들고 고칠 수 있다는 뜻이다.

- AWS 서비스 중 **유일하게 100% 가용성 SLA**를 보장한다.
- 이름의 "53"은 전통적인 DNS 포트 번호에서 따왔다.
- 리소스가 살아 있는지 검사(Health Check)하고, 그 결과에 따라 트래픽을 자동으로 다른 곳으로 넘길 수 있다.

기본 DNS 용어만 짚고 가면, 레코드는 **Hosted Zone**(레코드를 담는 그릇) 안에 들어간다. 레코드 하나는 도메인 이름, 타입(A·AAAA·CNAME·NS 등), 값(IP 등), 라우팅 정책, TTL로 이뤄진다. 시험에서 반드시 아는 타입은 A(이름→IPv4), AAAA(이름→IPv6), CNAME(이름→다른 이름), NS(이 존을 맡는 네임서버)다.

## 설계 관점 (SAA) #saa

![[AWS Certified Solutions Architect Slides v47.pdf#page=193]]

### Hosted Zone — Public vs Private

레코드를 담는 그릇. 둘의 차이는 "누가 이 이름을 물어보느냐"다.

- **Public Hosted Zone**: 인터넷에서 오는 질의에 답한다. 공개 도메인(application1.mypublicdomain.com)용.
- **Private Hosted Zone**: VPC 안에서만 통하는 사설 도메인(application1.company.internal)용. VPC 내부 리소스끼리 이름으로 서로를 찾을 때 쓴다. 인터넷에서는 안 보인다.
- 둘 다 존 하나당 월 $0.50.

### TTL — 캐싱 시간

클라이언트가 응답을 얼마나 오래 캐싱할지 정하는 값.

- **높은 TTL(예: 24시간)**: Route 53 질의가 줄어 비용이 낮지만, 레코드를 바꿔도 한동안 옛 값이 돌아다닌다.
- **낮은 TTL(예: 60초)**: 질의가 늘어 비용이 더 들지만, 레코드 변경이 빨리 반영된다.
- 시험 포인트: **Alias 레코드를 빼면 모든 레코드에 TTL이 필수**다. Alias는 AWS가 TTL을 알아서 관리해서 내가 못 정한다.

### CNAME vs Alias — 가장 많이 나오는 구분

ALB·CloudFront 같은 AWS 리소스는 `lb1-1234.us-east-2.elb.amazonaws.com` 같은 AWS 호스트네임을 준다. 이걸 내 도메인(myapp.mydomain.com)에 연결할 때 둘 중 무엇을 쓰느냐가 시험 단골이다.

| | CNAME | Alias |
| --- | --- | --- |
| 가리키는 대상 | 아무 호스트네임이나 | **AWS 리소스만** |
| **Zone Apex(루트 도메인)** | **안 됨** (example.com ✗, www.example.com ✓) | **됨** (example.com ✓) |
| 비용 | — | **무료** |
| TTL | 직접 설정 | **설정 불가**(AWS가 관리) |
| 타입 | CNAME | 항상 **A/AAAA** |
| Health Check | — | 네이티브 지원 |

- 핵심 한 줄: **루트 도메인(example.com)을 AWS 리소스에 붙이려면 CNAME이 안 되니 Alias를 써야 한다.** 이게 함정 문제의 90%.
- Alias는 리소스의 IP가 바뀌어도 자동으로 따라간다.
- **Alias 대상**: ELB, CloudFront, API Gateway, Elastic Beanstalk 환경, S3 정적 웹사이트, VPC Interface Endpoint, Global Accelerator, 같은 존의 Route 53 레코드.
- **Alias로 못 거는 것: EC2 DNS 이름.** (EC2는 CNAME이나 A 레코드로)

### 라우팅 정책 — 시나리오 매칭

"Routing"이라는 말에 속지 말 것. DNS는 트래픽을 직접 흘려보내지 않고, **질의에 어떤 값을 답할지**만 정한다.

| 정책 | 언제 쓰나 | 핵심 |
| --- | --- | --- |
| **Simple** | 단일 리소스로 보냄 | 값을 여러 개 주면 **클라이언트가 그중 랜덤 선택**. Health Check 연결 불가 |
| **Weighted** | 리소스별 트래픽 비율 제어. 카나리(새 버전 일부 테스트), 리전 간 분산 | 가중치/전체합 비율. 합이 100일 필요 없음. weight 0이면 그 리소스로 안 보냄. **전부 0이면 모두 균등** |
| **Latency-based** | 사용자 지연시간이 최우선일 때 | 사용자↔**AWS 리전** 간 지연으로 판단. 독일 사용자가 더 빠르면 미국으로 갈 수도 |
| **Failover** | 액티브-패시브 DR | Primary에 **Health Check 필수**. 죽으면 Secondary로 |
| **Geolocation** | 사용자 **위치** 기반(언어별 사이트, 콘텐츠 제한) | 대륙·국가·미국 주 단위. **Default 레코드 꼭 만들 것**(매칭 없을 때 대비). Latency-based와 헷갈리지 말 것 |
| **Geoproximity** | 위치 기반 + 특정 리소스로 트래픽 **쏠리게** 조정 | bias로 영역 크기 조정(확장 1~99, 축소 -1~-99). **Route 53 Traffic Flow** 필요 |
| **IP-based** | 클라이언트 IP(CIDR) 기반. 특정 ISP 사용자를 특정 엔드포인트로 | CIDR 목록 ↔ 엔드포인트 매핑 |
| **Multi-Value** | 여러 리소스로 보내되 죽은 건 빼고 | Health Check 연결 가능, **건강한 레코드 최대 8개** 반환. **ELB 대체재가 아님** |

- **Latency vs Geolocation 구분**: Latency는 "빠른 쪽", Geolocation은 "사용자가 있는 위치 그대로". 문제에서 "지연시간 최소화"면 Latency, "독일 사용자는 무조건 독일 서버/독일어 페이지"면 Geolocation.

### Health Check — 자동 장애 조치의 기반

리소스가 살아 있는지 검사해서, 죽으면 DNS 응답에서 빼버리는 기능. 세 종류가 있다.

1. **엔드포인트 감시**: 전 세계 약 15개 health checker가 검사. 임계값 기본 3회, 간격 기본 30초(10초로 줄이면 비용 ↑). HTTP·HTTPS·TCP 지원. **2xx·3xx 응답이면 통과.** checker의 **18% 이상**이 정상이라고 하면 정상 판정. 응답 본문 앞 **5120바이트**까지 텍스트로 통과/실패를 판단할 수도 있음. 주의: **Route 53 health checker IP 대역을 방화벽에서 열어줘야** 함.
2. **Calculated Health Check**: 여러 Health Check를 OR·AND·NOT으로 묶어 하나로. 자식 최대 256개. 일부를 점검 중이어도 전체가 죽지 않게 할 때.
3. **CloudWatch Alarm 감시**: Health Check가 알람 자체를 본다. **사설(private) 리소스용 우회로**가 핵심.

- **HTTP Health Check는 public 리소스만 가능.** Route 53 checker는 VPC 밖에 있어서 사설 엔드포인트(Private Hosted Zone, 온프레미스)에 직접 못 닿는다.
- 사설 리소스를 감시하려면: CloudWatch Metric → CloudWatch Alarm → 그 알람을 보는 Health Check, 이렇게 우회한다.

### Registrar vs DNS Service, 그리고 Hybrid DNS

- **도메인 등록 대행(Registrar)과 DNS 서비스는 별개**다. GoDaddy에서 도메인을 사고 DNS 관리는 Route 53으로 할 수 있다. 이때 등록 대행 쪽에서 **NS 레코드를 Route 53 네임서버로 바꿔주면** 된다(Hosted Zone 먼저 생성 → NS 갱신).
- **Hybrid DNS**: VPC 안(Route 53 Resolver)과 내 네트워크(온프레미스 등)의 DNS를 서로 풀어주는 구성.
  - **Inbound Endpoint**: 내 DNS Resolver(온프레미스)가 AWS 쪽 이름(EC2, Private Hosted Zone)을 물어볼 수 있게 한다. (외부 → AWS 방향)
  - **Outbound Endpoint**: Route 53 Resolver가 내 DNS Resolver로 질의를 넘긴다. (AWS → 외부 방향)

## 개발 관점 (DVA) #dva

![[AWS Certified Developer Slides v44.pdf#page=165]]

%% 이 시험의 관점에서 배운 내용을 정리합니다. %%

## 운영 관점 (CloudOps) #cloudops

![[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=539]]

%% 이 시험의 관점에서 배운 내용을 정리합니다. %%

## 시험 함정

- 루트 도메인(example.com)을 AWS 리소스에 연결 → CNAME 불가, **Alias** 필수 #exam/trap/route53
- Alias로 **EC2 DNS 이름은 못 건다**(ELB·CloudFront 등만) #exam/trap/route53
- TTL은 **Alias만 설정 불가**, 나머지 레코드는 필수 #exam/trap/route53
- "지연시간 최소화"는 **Latency-based**, "사용자 위치 그대로"는 **Geolocation** — 헷갈리지 말 것 #exam/trap/route53
- Geolocation은 **Default 레코드**를 안 만들면 매칭 안 되는 위치에서 응답이 빈다 #exam/trap/route53
- **사설 리소스 Health Check**는 직접 불가 → CloudWatch Alarm을 보는 Health Check로 우회 #exam/trap/route53
- Failover는 Primary에 **Health Check 필수** #exam/trap/route53
- Multi-Value는 ELB 대체재가 아니다(클라이언트 측 분산일 뿐) #exam/trap/route53

## 관련 노트

[[VPC]] · [[CloudFront & Global Accelerator]] · [[ELB & Auto Scaling]]
