---
service: Amazon VPC
exams: [saa, dva, cloudops]
domains:
  - saa/secure-architectures
  - dva/security
  - cloudops/networking
status: learning
confidence: 1
tags:
  - service
  - saa/network
  - dva/network
  - cloudops/network
---

# VPC

> 가상 네트워크 — 서브넷, 라우팅 테이블, NAT, 보안 그룹/NACL, 피어링, 엔드포인트.

## 개요

VPC(Virtual Private Cloud)는 AWS 안에 만드는 내 전용 가상 네트워크다. 이 노트는 SAA 슬라이드의 VPC 단원 전체를 담는다. 큰 그림은 세 덩어리다.

- **주소와 구획** — CIDR로 IP 범위를 정하고, VPC를 서브넷으로 나눈다. 퍼블릭 서브넷은 인터넷과 직접 통하고, 프라이빗 서브넷은 직접 통하지 않는다.
- **인터넷·서비스 연결** — IGW(양방향 인터넷), NAT(프라이빗의 아웃바운드만), VPC 엔드포인트(인터넷 안 거치고 AWS 서비스로). 라우팅 테이블이 이 경로를 정한다.
- **보안과 하이브리드** — SG/NACL로 트래픽을 거르고, VPN·Direct Connect·Transit Gateway로 온프레미스나 다른 VPC와 잇는다.

## 설계 관점 (SAA) #saa

![[AWS Certified Solutions Architect Slides v47.pdf#page=700]]

### CIDR과 IP 주소

CIDR은 IP 범위 표기법이다. `Base IP/서브넷마스크` 꼴이며 마스크가 **작을수록 범위가 넓다**. 외울 기준점: `/32`=1개, `/28`=16개, `/26`=64개, `/24`=256개, `/16`=65,536개, `/0`=전체. SG 규칙에서 `0.0.0.0/0`은 모든 IP, `x.x.x.x/32`는 한 IP다.

사설 IP 범위(이 안에서만 VPC를 만들 수 있다): `10.0.0.0/8`(대형망), `172.16.0.0/12`(**AWS 기본 VPC가 이 범위**), `192.168.0.0/16`(가정용). 나머지는 전부 공인 IP.

### VPC와 서브넷

- VPC는 **리전당 최대 5개**(소프트 한도), VPC당 CIDR 최대 5개. 각 CIDR 크기는 **최소 /28(16개) ~ 최대 /16(65,536개)**. 사설 범위만 허용되고, **다른 네트워크(회사망 등)와 CIDR이 겹치면 안 된다**(피어링·VPN이 막힌다).
- 서브넷은 **AZ에 묶인다.** AWS가 각 서브넷에서 **IP 5개를 예약**한다(처음 4개 + 마지막 1개). 예: `10.0.0.0/24`면 `.0`(네트워크), `.1`(VPC 라우터), `.2`(DNS), `.3`(예약), `.255`(브로드캐스트).
- **시험 단골 계산**: EC2 29개가 필요하면 `/27`(32−5=27<29)로는 부족, `/26`(64−5=59)이 필요. "필요 IP 수 + 5"를 수용하는 마스크를 고른다.
- **기본 VPC**: 새 계정에 자동 생성. 서브넷 지정 없이 띄운 EC2가 여기 들어가고, 인터넷 연결과 퍼블릭 IPv4·퍼블릭/프라이빗 DNS 이름을 받는다.

### 인터넷 연결: IGW / NAT

- **Internet Gateway (IGW)** — VPC가 인터넷과 통하게 한다. VPC와 **1:1**(VPC 하나에 IGW 하나). IGW만 붙인다고 끝이 아니라 **라우팅 테이블도 고쳐야** 인터넷이 된다(퍼블릭 서브넷의 정의가 사실상 "0.0.0.0/0 → IGW 라우트가 있는 서브넷").
- **NAT** — 프라이빗 서브넷의 EC2가 **아웃바운드로만** 인터넷에 나가게 한다(인바운드는 막음). 두 방식:
  - **NAT Instance**(구식, 시험엔 나옴) — 퍼블릭 서브넷에 직접 띄운 EC2. **Source/Destination Check를 꺼야** 하고, Elastic IP를 붙이며, SG를 직접 관리. 고가용성이 기본이 아니라 ASG·스크립트로 직접 구성해야 함. Bastion Host로도 쓸 수 있음.
  - **NAT Gateway**(관리형) — AWS가 운영, **특정 AZ에 생성**되고 Elastic IP 사용. IGW가 있어야 함(프라이빗→NATGW→IGW). 5~100Gbps 자동 확장, **SG 관리 불필요**. AZ 장애에 대비하려면 **AZ마다 NATGW를 따로** 둔다(AZ가 죽으면 그 AZ는 NAT가 필요 없으므로 cross-AZ failover는 불필요).
  - 선택: 고가용성·운영 부담 최소·고대역폭이면 **NAT Gateway**. NAT Gateway는 Bastion Host로 못 쓰고 SG가 없다.

### Security Group vs NACL (매우 자주 나옴)

| 구분 | Security Group | NACL |
| --- | --- | --- |
| 적용 단위 | **인스턴스(ENI)** | **서브넷** |
| 규칙 | **allow만** | allow + **deny** |
| 상태 | **Stateful**(응답 트래픽 자동 허용) | **Stateless**(응답도 규칙으로 명시 — ephemeral 포트 주의) |
| 평가 | 모든 규칙 종합 | 번호 순(낮은 번호 우선), **첫 매치로 결정** |

- NACL은 서브넷당 1개, 새 서브넷은 기본 NACL(모두 허용). 직접 만든 NACL은 **기본이 모두 거부**. 규칙 번호 1~32766, 100단위 증가 권장, 마지막 `*`는 거부. **특정 IP를 서브넷 레벨에서 차단**할 때 NACL을 쓴다(SG는 deny가 없어서 못 함).
- **Ephemeral 포트**: 클라이언트는 정해진 포트로 보내고 응답은 임시 포트(예: Linux 32768~60999)로 받는다. NACL은 stateless라 응답용 임시 포트 범위를 아웃바운드/인바운드에 따로 열어야 한다.

### VPC Peering / Endpoints

- **VPC Peering** — 두 VPC를 AWS 사설망으로 연결해 한 네트워크처럼 쓴다. **CIDR이 겹치면 안 되고**, **전이(transitive)되지 않는다**(A-B, B-C가 있어도 A-C는 따로 맺어야 함). **각 VPC의 라우팅 테이블을 고쳐야** 통신된다. 다른 계정·리전 간에도 가능.
- **VPC Endpoints (PrivateLink)** — AWS 서비스에 **인터넷을 안 거치고 사설망으로** 접근. IGW·NATGW가 필요 없어진다. 두 종류:
  - **Gateway Endpoint** — 라우팅 테이블의 타깃으로 등록. **S3·DynamoDB만** 지원, **무료**, SG 없음. **시험에서는 거의 항상 Gateway가 정답.**
  - **Interface Endpoint** — ENI(사설 IP)를 만들고 SG를 붙임. 대부분의 AWS 서비스 지원, **유료(시간+GB)**. 온프레미스(VPN·DX)나 다른 VPC·리전에서 접근해야 할 때 선택.

### VPC Flow Logs

VPC·서브넷·ENI 단위로 IP 트래픽 정보를 기록. 연결 문제 트러블슈팅에 쓰고, S3·CloudWatch Logs·Kinesis Data Firehose로 보낸다. ELB·RDS·ElastiCache·NATGW 같은 관리형 인터페이스 트래픽도 잡는다. 로그의 **ACTION 필드(ACCEPT/REJECT)**로 SG/NACL 문제를 진단한다: 인바운드 REJECT면 NACL이나 SG, 인바운드 ACCEPT인데 아웃바운드 REJECT면 (stateful인 SG는 응답을 자동 허용하므로) **NACL 문제**다. S3 → Athena, 또는 CloudWatch Logs Insights로 분석.

### 하이브리드 연결: VPN / Direct Connect / Transit Gateway

- **Site-to-Site VPN** — 온프레미스와 VPC를 잇는다. AWS 쪽 **VGW(Virtual Private Gateway)** + 고객 쪽 **CGW(Customer Gateway)**. **공중 인터넷을 타지만 IPsec으로 암호화**. 라우팅 테이블에서 **Route Propagation을 켜야** 하고, 온프레미스에서 EC2로 ping 하려면 SG 인바운드에 ICMP 허용. 빠르게(수 시간) 구성 가능.
- **VPN CloudHub** — 여러 사이트를 VGW 하나에 모으는 저비용 hub-and-spoke. 공중망 VPN.
- **Direct Connect (DX)** — 온프레미스에서 VPC로 가는 **전용 사설 회선**. 큰 데이터·일관된 성능·하이브리드에 적합하지만 **데이터가 암호화되지 않는다**(사설일 뿐). 암호화하려면 **DX + VPN(IPsec)**. 신규 구성에 **보통 한 달 이상** 걸린다. Dedicated(1~400Gbps) vs Hosted(50Mbps~25Gbps, 온디맨드 증감).
  - **DX 복원력**: DX가 끊기면 **Site-to-Site VPN을 백업**으로 두는 게 저렴한 대안(DX 회선을 하나 더 까는 건 비쌈).
  - **Direct Connect Gateway** — 여러 리전의 여러 VPC를 한 DX로 연결할 때.
- **Transit Gateway** — 수천 개 VPC·온프레미스를 **전이적으로(transitive)** 잇는 hub-and-spoke 라우터. 리전 리소스(리전 간 피어링 가능), RAM으로 계정 간 공유, 라우팅 테이블로 어느 VPC끼리 통신할지 제한. **IP Multicast를 지원하는 유일한 서비스.** Site-to-Site VPN ECMP로 대역폭을 합칠 수 있다.
- **Traffic Mirroring** — ENI의 트래픽을 복사해 ENI나 NLB로 보내 보안 어플라이언스로 검사(콘텐츠 검사·위협 탐지).

### IPv6와 Egress-only IGW

- **AWS의 모든 IPv6는 공인이며 인터넷 라우팅이 된다**(사설 범위 없음). VPC/서브넷에서 **IPv4는 끌 수 없고**, IPv6를 켜면 dual-stack으로 동작.
- **IPv4 트러블슈팅**: 서브넷에 EC2를 못 띄우면 IPv6 부족이 아니라 **IPv4가 동났기 때문**(IPv6 공간은 매우 큼). 해결은 서브넷에 새 IPv4 CIDR 추가.
- **Egress-only Internet Gateway** — IPv6 전용. NAT Gateway의 IPv6 버전으로, 인스턴스가 IPv6로 아웃바운드만 하고 인터넷에서 들어오는 IPv6 연결은 막는다. 라우팅 테이블 수정 필요.

### 네트워킹 비용 (비용 도메인)

- **같은 AZ·사설 IP 통신은 무료.** 공인/Elastic IP를 쓰거나 AZ를 넘으면 과금($0.01~), 리전 간은 더 비쌈($0.02~). 절약하려면 **사설 IP·같은 AZ**를 쓴다(단 같은 AZ는 고가용성과 트레이드오프).
- **Egress(밖으로 나가는) 트래픽이 비싸고 ingress는 보통 무료.** 트래픽을 최대한 AWS 안에 두고, S3 같은 경우 **같은 리전 Gateway Endpoint는 무료**, NAT Gateway 경유는 시간+처리량 과금이라 더 비싸다.

### AWS Network Firewall

VPC **전체**를 L3~L7로 보호. VPC 간·인바운드·아웃바운드·DX/VPN 트래픽을 모두 검사. 내부적으로 Gateway Load Balancer를 쓰고, 수천 개 규칙(IP/포트·프로토콜·도메인 리스트·정규식)으로 allow/drop/alert. Firewall Manager로 여러 VPC에 일괄 적용. (앞 보안 단원의 WAF=L7 앱 보호와 구분 — Network Firewall은 VPC 네트워크 전체.)

## 개발 관점 (DVA) #dva

![[AWS Certified Developer Slides v44.pdf#page=196]]

%% 이 시험의 관점에서 배운 내용을 정리합니다. %%

## 운영 관점 (CloudOps) #cloudops

![[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=585]]

%% 이 시험의 관점에서 배운 내용을 정리합니다. %%

## 시험 함정

- 서브넷은 AWS가 IP 5개를 예약한다. 필요 IP 수 + 5를 수용하는 마스크를 골라야 한다(29개 → /27 불가, /26). #exam/trap/vpc
- IGW만 붙여서는 인터넷이 안 된다. 라우팅 테이블에 0.0.0.0/0 → IGW를 추가해야 한다. #exam/trap/vpc
- NAT Gateway는 AZ에 묶인다. 고가용성은 AZ마다 NATGW를 따로 둬서 얻는다. NAT Instance는 Source/Dest Check를 꺼야 동작. #exam/trap/vpc
- SG는 stateful·instance·allow only, NACL은 stateless·subnet·allow+deny. 특정 IP 차단은 NACL(SG엔 deny 없음). #exam/trap/vpc
- NACL은 stateless라 ephemeral 포트 응답을 따로 열어야 한다. #exam/trap/vpc
- VPC Peering은 CIDR이 겹치면 안 되고 전이되지 않으며, 양쪽 라우팅 테이블을 고쳐야 한다. #exam/trap/vpc
- VPC Endpoint: S3·DynamoDB는 Gateway(무료, 라우팅 테이블), 나머지는 Interface(ENI·SG·유료). 시험은 대개 Gateway. #exam/trap/vpc
- Direct Connect는 사설이지만 암호화 안 됨. 암호화하려면 DX + VPN. 신규 구성은 한 달 이상. #exam/trap/vpc
- VPN은 공중망+IPsec 암호화, DX는 전용 사설망. DX 백업은 Site-to-Site VPN이 저렴. #exam/trap/vpc
- Transit Gateway만 IP Multicast 지원. 전이적 피어링이 필요하면 Peering이 아니라 TGW. #exam/trap/vpc
- Flow Logs의 ACTION에서 인바운드 ACCEPT인데 아웃바운드 REJECT면 SG가 아니라 NACL 문제(SG는 stateful). #exam/trap/vpc
- AWS에서 IPv4는 끌 수 없고 모든 IPv6는 공인. IPv6 아웃바운드 전용은 Egress-only IGW. #exam/trap/vpc

## 관련 노트

[[EC2]] · [[Route 53]] · [[재해 복구 & 마이그레이션]] · [[KMS & 암호화]] · [[CloudWatch & CloudTrail & Config]] · [[S3]] · [[ELB & Auto Scaling]] · [[Organizations & 계정 관리]]
