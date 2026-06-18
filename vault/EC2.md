---
service: Amazon EC2
exams: [saa, dva, cloudops]
domains:
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

운영 시각에서는 "이미 떠 있는 인스턴스를 어떻게 고치고·모니터링하고·복구하느냐"를 본다. 설계(SAA) 섹션의 인스턴스 유형·배치 그룹·Hibernate 위에, 접속 문제 해결·메트릭 수집·상태 점검·자동 복구·AMI 운영이 더해진다.

### 인스턴스 타입 변경

**EBS 기반 인스턴스에서만** 된다. **stop → Instance Settings에서 Change Instance Type → start.** (실행 중에는 못 바꾼다.)

### 접속 문제 해결 (SSH troubleshooting)

- **"Unprotected private key file"**: pem 키 파일 권한이 너무 열려 있음 → **`chmod 400`**.
- **"Host key not found" / "Permission denied" / "Connection closed"**: 로그인 **OS 사용자명이 틀림**(예: Amazon Linux는 `ec2-user`, Ubuntu는 `ubuntu`).
- **"Connection timed out"**: 네트워크 차단 문제. 원인 후보 — **Security Group**·**NACL** 설정, 서브넷 **route table**(VPC 밖 트래픽이 IGW로 가는지), **공인 IPv4 없음**, 인스턴스 **CPU 부하**가 높음.

### SSH vs EC2 Instance Connect vs EIC Endpoint

- **EC2 Instance Connect**: 콘솔/CLI로 접속할 때 **AWS IP 범위에서 1회용 SSH 공개키를 60초간** 인스턴스에 푸시한다. SG에 **AWS IP 범위(EC2 Instance Connect)**의 22번을 열어 둬야 한다.
- **EC2 Instance Connect Endpoint (EIC Endpoint)**: **프라이빗 서브넷의 EC2**에 안전하게 접속. **IGW·NAT·인터넷이 필요 없다.** EIC Endpoint SG는 대상으로 **아웃바운드 SSH 허용**, EC2 인스턴스 SG는 **EIC Endpoint SG로부터 인바운드 SSH 허용**.

### CloudWatch 메트릭과 Unified Agent

- **AWS 제공 메트릭**: **Basic Monitoring(기본, 5분 간격)** vs **Detailed Monitoring(유료, 1분 간격)**. CPU·Network·Disk·Status Check 포함.
- **EC2 기본 메트릭 항목**: CPU(Utilization + Credit Usage/Balance), Network(In/Out), Status Check, Disk(**Instance Store 전용**). → **RAM은 EC2 기본 메트릭에 없다.**
- **Custom metric**: 직접 push. Basic Resolution(1분) / High Resolution(1초까지). RAM·앱 레벨 메트릭은 여기로. **EC2 인스턴스 role의 IAM 권한**이 맞아야 한다.
- **Unified CloudWatch Agent**: 가상 서버(EC2·온프레미스)에 깔아 **RAM·프로세스·디스크 사용량 같은 시스템 메트릭**과 **로그(CloudWatch Logs)**를 보낸다. **에이전트 없이는 인스턴스 내부 로그가 CloudWatch Logs로 안 간다.** 설정은 **SSM Parameter Store로 중앙 관리**, 기본 namespace는 **`CWAgent`**.
  - **procstat 플러그인**: 개별 **프로세스**의 사용량 수집(Linux·Windows). 대상 선택은 `pid_file`·`exe`(RegEx)·`pattern`(RegEx). 메트릭은 **`procstat`** 접두사.

### Status Check와 자동 복구

자동 점검 3종으로 하드웨어·소프트웨어 문제를 가린다.

| 점검 | 무엇을 보나 | 해결 |
| --- | --- | --- |
| **System Status Check** | AWS 측 문제(물리 호스트 장애·전원 등). **Personal Health Dashboard**로 예정 정비 확인 | **stop & start**(새 호스트로 이전) |
| **Instance Status Check** | 인스턴스 소프트웨어·네트워크 설정(잘못된 네트워크 구성, 메모리 고갈 등) | **reboot** 또는 설정 변경 |
| **Attached EBS Status Check** | 연결된 EBS 볼륨(도달성·I/O) | **reboot** 또는 볼륨 교체 |

- **메트릭(1분 간격)**: `StatusCheckFailed_System`·`StatusCheckFailed_Instance`·`StatusCheckFailed_AttachedEBS`·`StatusCheckFailed`(전체).
- **자동 복구 — 방식 차이가 출제됨**:
  - **CloudWatch Alarm 복구**: `StatusCheckFailed_System`을 감시해 복구. **같은 private/public IP·EIP·메타데이터·Placement Group 유지**. SNS 알림 가능.
  - **Auto Scaling Group**: min/max/desired=1로 두면 인스턴스를 다시 띄워 복구하지만, **private/elastic IP는 유지되지 않는다.**

### Instance Scheduler

**CloudFormation으로 배포하는 AWS 솔루션**(서비스 아님). 비용 절감(최대 70%)을 위해 **업무시간 외 EC2를 자동 stop/start**. EC2·ASG·RDS 지원. **스케줄은 DynamoDB 테이블**에서 관리하고, **태그 + Lambda**로 stop/start. 교차 계정·교차 리전 가능.

### AMI 운영

**AMI = EC2 인스턴스를 커스터마이즈(소프트웨어·OS·설정·모니터링)한 이미지.** 사전 패키징이라 부팅이 빠르다. **특정 리전용**이고 리전 간 복사 가능. 출처는 Public(AWS) / 직접 만든 것 / Marketplace.

- **AMI 만드는 과정**: 인스턴스 커스터마이즈 → **stop(데이터 무결성)** → AMI 빌드(**EBS 스냅샷도 같이 생성**) → 그 AMI로 다른 인스턴스 시작.
- **No-Reboot 옵션**: 끄지 않고 AMI 생성. **기본은 비활성**(AWS가 파일시스템 무결성 위해 인스턴스를 끈 뒤 생성). **No-Reboot를 켜면 OS 버퍼가 디스크로 flush되지 않아** 무결성이 떨어진다.
- **AWS Backup으로 AMI**: AWS Backup은 스냅샷 뜰 때 **재부팅을 안 한다(no-reboot)** → 파일시스템 무결성이 보장된 AMI를 못 만든다. 무결성이 필요하면 **EventBridge + Lambda + `CreateImage`(reboot 파라미터)** 로 만든다.
- **AZ 간 이동**: create image → 다른 AZ에서 launch/restore.
- **Cross-Account 공유**: 다른 계정에 공유해도 **소유권은 그대로**. **암호화 안 된 볼륨**이거나 **고객 관리 키(CMK)로 암호화된 볼륨**만 공유 가능하고, 암호화됐으면 **CMK도 함께 공유**(대상 계정에 `kms:DescribeKey`·`CreateGrant`·`Decrypt`·`GenerateDataKey`·`ReEncrypt` 권한).
- **Cross-Account 복사**: 공유받은 AMI를 **복사하면 복사본의 소유자가 된다.** 소스 소유자가 EBS 스냅샷 read 권한(암호화면 키)도 줘야. 복사하면서 **자기 CMK로 재암호화** 가능(Decrypt: 소스 CMK, Encrypt: 대상 CMK).
- **EC2 Image Builder**: AMI(또는 컨테이너 이미지) 생성·유지·검증·테스트를 **자동화**. 스케줄(주간·패키지 업데이트 시) 실행, **서비스 자체는 무료**(기반 리소스만 과금). Builder 인스턴스로 빌드 → 새 AMI → Test 인스턴스로 검증 → 여러 리전에 배포.
- **AMI in Production**: **IAM 정책으로 사전 승인된 AMI(특정 태그)에서만** 인스턴스를 띄우게 강제(`ec2:ResourceTag/...` 조건). **AWS Config**와 묶어 미승인 AMI로 뜬 비준수 인스턴스를 찾는다.

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
- 인스턴스 타입 변경은 **EBS 기반만**, **stop 후** 변경. #exam/trap/ec2
- **RAM은 EC2 기본 메트릭에 없다** → 메모리·디스크 사용량·내부 로그는 **Unified CloudWatch Agent**로 수집(설정은 SSM Parameter Store). #exam/trap/cloudwatch
- **System Status Check 실패 = 호스트 문제 → stop & start**(새 호스트 이전). Instance Check = reboot. #exam/trap/status-check
- 자동 복구로 **같은 IP/EIP 유지** → CloudWatch Alarm 기반 복구. **ASG는 IP 유지 안 됨**. #exam/trap/status-check
- 프라이빗 EC2에 IGW/NAT 없이 SSH → **EC2 Instance Connect Endpoint(EIC Endpoint)**. #exam/trap/ec2-networking
- 무결성 보장 AMI는 **reboot하며 생성**. No-Reboot는 버퍼 미flush. **AWS Backup은 no-reboot**라 부적합 → EventBridge+Lambda+CreateImage(reboot). #exam/trap/ami
- 암호화 AMI를 다른 계정에 공유 → **CMK도 공유**(kms:DescribeKey·CreateGrant·Decrypt·GenerateDataKey·ReEncrypt). #exam/trap/ami
- 승인된 AMI에서만 인스턴스 시작 강제 → **IAM 정책(태그 조건)** + 비준수 탐지는 **AWS Config**. #exam/trap/ami

%% 연습문제에서 틀리거나 헷갈린 지점을 위 형식으로 계속 추가합니다. 시험 직전 주에 `tag:#exam/trap` 검색으로 한 번에 모아 봅니다. %%

## 관련 노트

[[EBS & EFS]] · [[ELB & Auto Scaling]] · [[VPC]] · [[Systems Manager]] · [[CloudWatch & CloudTrail & Config]] · [[KMS & 암호화]]
