---
service: Amazon EC2
exams: [saa, dva, cloudops]
domains:
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

# EC2

> AWS의 기본 컴퓨팅 서비스. 인스턴스 유형, 구매 옵션, AMI, 배치 그룹까지.

## 개요

EC2는 AWS에서 가상 서버를 빌리는 서비스다(IaaS). EC2 하나만 말하는 게 아니라 가상 머신(EC2) + 가상 드라이브(EBS) + 로드 밸런서(ELB) + 오토 스케일링(ASG)이 한 묶음으로 다뤄진다.

인스턴스를 만들 때 고르는 것: OS, CPU, RAM, 스토리지(네트워크형인 EBS·EFS 또는 하드웨어 직결인 Instance Store), 네트워크 카드, 방화벽 역할의 Security Group, 첫 부팅 때 실행할 스크립트인 User Data.

인스턴스 이름은 `m5.2xlarge`처럼 읽는다 — `m`은 인스턴스 클래스, `5`는 세대, `2xlarge`는 클래스 안에서의 크기.

## 설계 관점 (SAA) #saa

![[AWS Certified Solutions Architect Slides v47.pdf#page=41]]

- [[AWS Certified Solutions Architect Slides v47.pdf#page=78|EC2 Associate — IP·배치 그룹·ENI·Hibernate (p.78)]]

### 인스턴스 유형 — 워크로드에 맞춰 고르기

| 유형 | 특징 | 이런 문제에 고른다 |
| --- | --- | --- |
| General Purpose (t, m) | 컴퓨팅·메모리·네트워크 균형 | 웹 서버, 코드 저장소 |
| Compute Optimized (c) | 고성능 프로세서 | 배치 처리, 미디어 트랜스코딩, HPC, 게임 서버 |
| Memory Optimized (r, x) | 큰 데이터셋을 메모리에서 처리 | 인메모리 DB, 분산 캐시, BI용 실시간 처리 |
| Storage Optimized (i, d) | 로컬 스토리지에 높은 순차 I/O | OLTP, 데이터 웨어하우스, 분산 파일 시스템 |

### User Data

첫 부팅에 **한 번만** 실행되는 부트스트랩 스크립트. root 권한으로 돈다. 업데이트·소프트웨어 설치 같은 초기 설정 자동화에 쓴다. stop 후 start해도 다시 실행되지 않는다.

### Security Group

인스턴스 앞단의 방화벽. 동작 방식이 곧 출제 포인트다.

- **allow 규칙만** 있다. deny 규칙은 못 쓴다.
- 기본값: 인바운드 전부 차단, 아웃바운드 전부 허용.
- 인스턴스 여러 개에 같은 SG를 붙일 수 있고, 리전·VPC 조합에 묶인다.
- 소스에 IP 대신 **다른 Security Group을 참조**할 수 있다. 인스턴스끼리 통신을 열 때 IP를 몰라도 된다.
- 트래픽이 차단되면 인스턴스까지 도달하지 않는다. 그래서 **접속이 timeout이면 SG 문제, connection refused면 애플리케이션 문제**다.
- SSH 접근용 SG는 따로 분리해 관리하는 게 좋다.

외워야 하는 포트: 22(SSH·SFTP), 21(FTP), 80(HTTP), 443(HTTPS), 3389(RDP, Windows 원격 접속).

### 구매 옵션 — 비용 문제의 단골

| 옵션 | 할인 | 언제 고르는가 |
| --- | --- | --- |
| On-Demand | 없음(최고가) | 짧고 예측 불가능한 워크로드. 약정 없음 |
| Reserved (1·3년) | 최대 ~72% | 안정적으로 오래 쓰는 워크로드(예: DB). 인스턴스 유형·리전·테넌시·OS를 고정 |
| Convertible Reserved | 더 적음 | Reserved인데 도중에 유형·OS·테넌시를 바꿀 수 있어야 할 때 |
| Savings Plans | Reserved 수준 | "시간당 $X 사용"을 약정. 패밀리·리전은 고정, 크기·OS·테넌시는 유연 |
| Spot | 최대 90% | 중단을 견디는 워크로드(배치, 데이터 분석). 언제든 회수될 수 있음 |
| Dedicated Host | — (최고가) | 물리 서버를 통째로. BYOL 라이선스, 강한 컴플라이언스 요구 |
| Dedicated Instance | — | 하드웨어를 다른 고객과 안 나누면 됨. 배치 제어는 불가 |
| Capacity Reservation | **없음** | 특정 AZ에 용량만 확보. 안 써도 On-Demand 요금이 나감 |

- Reserved는 Regional/Zonal 범위가 있고, Zonal만 AZ 용량을 확보한다.
- On-Demand 과금: Linux·Windows는 첫 1분 이후 초 단위, 나머지 OS는 시간 단위.

### Spot 자세히

- max price를 정해두고, 현재 spot price가 그보다 낮은 동안 인스턴스를 쓴다. spot price가 넘어서면 **2분 유예** 후 stop 또는 terminate.
- **Spot Request를 취소해도 떠 있는 인스턴스는 안 꺼진다.** 순서는 요청 취소 → 인스턴스 종료. (persistent 요청을 안 끄고 인스턴스만 죽이면 다시 떠버린다.)
- **Spot Fleet**: Spot + (선택) On-Demand를 섞어 목표 용량을 채우는 기능. 할당 전략 중 `priceCapacityOptimized`(용량 많은 풀 중 최저가 선택)가 대부분의 워크로드에 권장.

### IP와 ENI

- 인스턴스는 기본으로 사설 IP + 공인 IP를 받는데, **stop → start 하면 공인 IP가 바뀐다.**
- 고정 공인 IP가 필요하면 **Elastic IP**. 계정당 기본 5개, 한 번에 인스턴스 하나에만 붙는다. 다만 슬라이드도 가급적 피하라고 한다 — 대신 DNS 이름을 쓰거나 로드 밸런서 뒤에 두는 게 좋은 설계다.
- **ENI(Elastic Network Interface)**: 가상 네트워크 카드. 인스턴스와 독립적으로 만들어 옮겨 붙일 수 있어서 장애 시 failover에 쓴다. **특정 AZ에 묶인다.**

### Placement Group — 배치 전략 3가지

| 전략 | 배치 | 트레이드오프 | 용도 |
| --- | --- | --- | --- |
| Cluster | 한 AZ 안에 밀집 | 저지연·10Gbps 네트워크 ↔ AZ 장애 시 전부 죽음 | 빨리 끝내야 하는 빅데이터 작업, HPC |
| Spread | 인스턴스마다 다른 하드웨어 | 동시 장애 위험 최소 ↔ **AZ당 7개 제한** | 서로 격리돼야 하는 핵심 인스턴스 |
| Partition | 파티션(랙 묶음) 단위 분산 | AZ당 최대 7파티션, 수백 인스턴스까지 | Hadoop, Cassandra, Kafka |

### EC2 Hibernate

stop 대신 hibernate하면 RAM 상태를 root EBS 볼륨에 써두고 멈춘다. 다시 시작할 때 OS 부팅 없이 메모리 상태가 그대로 살아나서 빠르다. 조건: **root 볼륨이 암호화된 EBS여야 하고**, RAM 150GB 미만, 베어메탈 불가, 최대 60일까지만.

## 개발 관점 (DVA) #dva

![[AWS Certified Developer Slides v44.pdf#page=39]]

%% 이 시험의 관점에서 배운 내용을 정리합니다. %%

## 운영 관점 (CloudOps) #cloudops

![[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=9]]

- [[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=28|AMI (p.28)]]

%% 이 시험의 관점에서 배운 내용을 정리합니다. %%

## 시험 함정

- 접속 timeout은 Security Group 문제, connection refused는 애플리케이션 문제다. #exam/trap/security-group
- Security Group에는 allow 규칙만 있다. deny가 필요하면 SG가 아니라 NACL 쪽 이야기다. #exam/trap/security-group
- Spot Request를 취소해도 인스턴스는 안 꺼진다. 요청 취소 → 인스턴스 종료 순서. #exam/trap/spot
- Spot은 중요 작업이나 DB에 쓰지 않는다. 회수되면 끝이다. #exam/trap/spot
- 공인 IP는 stop/start 시 바뀐다. 고정이 필요하면 Elastic IP, 더 나은 답은 DNS나 로드 밸런서. #exam/trap/ec2-networking
- ENI는 AZ에 묶인다. 다른 AZ의 인스턴스로는 못 옮긴다. #exam/trap/ec2-networking
- Spread placement group은 AZ당 7개 제한이 있다. 그 이상 분산하려면 Partition. #exam/trap/placement-group
- Capacity Reservation은 할인이 없고, 인스턴스를 안 띄워도 On-Demand 요금이 나간다. #exam/trap/cost-optimization
- Hibernate는 root EBS 볼륨이 암호화돼 있어야 하고, RAM 150GB 미만, 최대 60일. #exam/trap/ec2

%% 연습문제에서 틀리거나 헷갈린 지점을 위 형식으로 계속 추가합니다. 시험 직전 주에 `tag:#exam/trap` 검색으로 한 번에 모아 봅니다. %%

## 관련 노트

[[EBS & EFS]] · [[ELB & Auto Scaling]] · [[VPC]] · [[Systems Manager]]
