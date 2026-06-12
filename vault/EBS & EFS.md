---
service: Amazon EBS · EFS · Instance Store
exams: [saa, dva, cloudops]
domains:
  - saa/resilient-architectures
  - dva/development
  - cloudops/reliability
status: learning
confidence: 1
tags:
  - service
  - saa/storage
  - dva/storage
  - cloudops/storage
---

# EBS & EFS

> EC2 인스턴스 스토리지 — EBS 볼륨 유형, 스냅샷, EFS, Instance Store 비교.

## 개요

EC2 인스턴스에 스토리지를 붙이는 방법은 세 가지고, 셋을 구분하는 게 이 주제의 전부다.

- **EBS** — 네트워크로 연결되는 블록 스토리지. "네트워크 USB 스틱"이라 생각하면 된다. 기본적으로 한 번에 인스턴스 하나에만 붙고, 특정 AZ에 묶인다. 인스턴스를 종료해도 데이터가 남는다.
- **Instance Store** — 호스트 하드웨어에 직결된 디스크. I/O는 가장 빠르지만 인스턴스를 stop하면 데이터가 사라진다(ephemeral).
- **EFS** — 관리형 NFS. 여러 AZ의 인스턴스 수백 대가 동시에 마운트할 수 있는 공유 파일 시스템.

## 설계 관점 (SAA) #saa

![[AWS Certified Solutions Architect Slides v47.pdf#page=93]]

### EBS 기본 동작

- 네트워크 드라이브라서 약간의 지연이 있고, 대신 인스턴스에서 떼어 다른 인스턴스에 빨리 옮겨 붙일 수 있다.
- **AZ에 묶인다.** us-east-1a의 볼륨은 us-east-1b 인스턴스에 못 붙인다. 다른 AZ로 옮기려면 스냅샷을 떠서 그쪽에 복원한다.
- 용량(GB)과 IOPS를 미리 정해서 만들고, 쓴 만큼이 아니라 **잡아둔 만큼** 과금된다. 용량은 나중에 늘릴 수 있다.
- **Delete on Termination**: 인스턴스 terminate 시 root 볼륨은 기본으로 같이 삭제되고, 추가로 붙인 볼륨은 기본으로 남는다. terminate 후에도 root를 보존해야 하는 문제가 나오면 이 속성을 끄는 게 답이다.

### EBS Snapshot

특정 시점의 볼륨 백업. detach 없이도 뜰 수 있지만 권장은 detach 후. **AZ·리전 간 복사가 되므로 볼륨을 옮기는 수단**이기도 하다. 백업은 I/O를 쓰므로 트래픽이 많을 때는 피한다.

- **Snapshot Archive** — 75% 싼 보관 계층. 복원에 24~72시간 걸린다.
- **Recycle Bin** — 실수로 지운 스냅샷을 복구할 수 있게 보존 규칙(1일~1년)을 건다.
- **Fast Snapshot Restore (FSR)** — 복원한 볼륨의 첫 사용 지연을 없앤다. 비싸다.

### AMI

인스턴스의 OS·소프트웨어·설정을 통째로 구운 이미지. 같은 구성을 빠르게 복제할 때 쓴다. **리전 단위**로 만들어지고 리전 간 복사할 수 있다. 인스턴스에서 AMI를 만들면(권장: stop 후) EBS 스냅샷도 같이 생성된다. 출처는 셋: AWS 제공 Public AMI, 직접 만든 AMI, Marketplace AMI.

### Instance Store

EBS의 네트워크 성능으로 부족할 때 쓰는 하드웨어 직결 디스크. IOPS가 수백만 단위로, EBS와 자릿수가 다르다. 대신 **stop하면 데이터가 사라지므로** 버퍼·캐시·임시 데이터 용도다. 하드웨어 장애 시 데이터 유실 위험이 있고 백업·복제는 직접 챙겨야 한다.

### EBS 볼륨 유형 6종

| 유형 | 종류 | 핵심 수치 | 언제 고르는가 |
| --- | --- | --- | --- |
| gp3 | SSD | 기본 3,000 IOPS·125 MiB/s → **크기와 무관하게** 16,000 IOPS·1,000 MiB/s까지 증설 | 대부분의 워크로드, 부트 볼륨 |
| gp2 | SSD | 크기에 IOPS 연동(3 IOPS/GiB, max 16,000), 소형은 3,000까지 버스트 | 구형. IOPS 올리려면 크기를 키워야 함 |
| io1 | SSD | max 64,000 IOPS(Nitro) / 32,000(그 외), 크기와 IOPS 독립 | 16,000 IOPS 초과, DB 워크로드 |
| io2 Block Express | SSD | 4 GiB~64 TiB, max 256,000 IOPS, 1ms 미만 지연 | 최고 성능이 필요할 때 |
| st1 | HDD | max 500 MiB/s·500 IOPS | 빅데이터, 로그 처리 — 처리량 중심 |
| sc1 | HDD | max 250 MiB/s·250 IOPS | 접근 빈도 낮고 비용 최소화 |

- **부트 볼륨은 SSD 계열(gp2/gp3, io1/io2)만 가능하다. HDD는 안 된다.**
- 문제에서 "16,000 IOPS 이상"이 나오면 io1/io2 + Nitro 인스턴스 조합이다.

### EBS Multi-Attach

io1/io2 한정으로, **같은 AZ 안에서** 한 볼륨을 최대 **16개** 인스턴스에 동시에 붙인다. 모든 인스턴스가 읽기·쓰기 가능하므로 동시 쓰기는 애플리케이션이 책임져야 하고, 클러스터를 인지하는 파일 시스템이 필요하다(XFS·EXT4 같은 일반 파일 시스템은 안 됨).

### EBS 암호화

암호화된 볼륨을 만들면 저장 데이터, 인스턴스와의 전송 구간, 스냅샷, 그 스냅샷으로 만든 볼륨까지 전부 암호화된다. [[KMS & 암호화|KMS]] 키(AES-256)를 쓰고 지연 영향은 거의 없다.

이미 있는 비암호화 볼륨은 바로 암호화로 못 바꾼다. 절차: 스냅샷 생성 → 스냅샷을 **복사하면서 암호화** → 그 스냅샷으로 새 볼륨 생성 → 인스턴스에 교체 장착.

### EFS

관리형 NFS(NFSv4.1). 멀티 AZ에 걸쳐 수백 인스턴스가 동시 마운트하는 공유 스토리지로, WordPress 같은 콘텐츠 공유가 대표 용도다.

- **Linux(POSIX) 전용. Windows AMI에는 못 붙인다.**
- 접근 제어는 Security Group으로 한다.
- 용량 계획이 필요 없다 — 자동으로 늘고 쓴 만큼 과금. 단가는 gp2의 3배 수준이라 스토리지 티어로 줄인다.
- **Performance Mode** (생성 시 결정): General Purpose(기본, 지연에 민감한 웹 서버·CMS) / Max I/O(지연은 늘지만 병렬 처리량 큼 — 빅데이터, 미디어 처리).
- **Throughput Mode**: Bursting(용량 비례) / Provisioned(용량과 무관하게 처리량 지정) / Elastic(워크로드 따라 자동 조절 — 예측 불가능할 때).
- **Storage Class**: Standard / Infrequent Access(저장은 싸고 조회에 비용) / Archive(연 몇 회 접근, 50% 더 저렴). lifecycle policy로 "N일 미접근 시 이동"을 건다. 가용성은 Standard(멀티 AZ, 운영용) vs One Zone(단일 AZ, 개발용).

### EBS vs EFS vs Instance Store — 고르는 기준

한 인스턴스의 디스크면 EBS, 여러 인스턴스가 공유하는 파일 시스템이면 EFS, 최고 I/O의 임시 공간이면 Instance Store.

## 개발 관점 (DVA) #dva

![[AWS Certified Developer Slides v44.pdf#page=72]]

%% 이 시험의 관점에서 배운 내용을 정리합니다. %%

## 운영 관점 (CloudOps) #cloudops

![[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=225]]

%% 이 시험의 관점에서 배운 내용을 정리합니다. %%

## 시험 함정

- EBS는 AZ에 묶인다. 다른 AZ로 옮기는 방법은 스냅샷 → 복원뿐이다. #exam/trap/ebs-efs
- terminate 시 root 볼륨은 기본 삭제, 추가 볼륨은 기본 유지. 반대로 알고 있으면 틀린다. #exam/trap/ebs-efs
- gp2는 크기를 키워야 IOPS가 오르고, gp3는 크기와 무관하게 IOPS를 올린다. 16,000 IOPS를 넘기면 io1/io2. #exam/trap/ebs-efs
- HDD(st1·sc1)는 부트 볼륨이 될 수 없다. #exam/trap/ebs-efs
- Multi-Attach는 io1/io2 한정, 같은 AZ, 최대 16개 인스턴스. gp3로는 안 된다. #exam/trap/ebs-efs
- EFS는 Linux(POSIX) 전용이다. Windows 인스턴스 공유 스토리지 문제의 답이 아니다. #exam/trap/ebs-efs
- Instance Store는 stop하면 데이터가 사라진다. 영구 데이터를 두면 안 된다. #exam/trap/ebs-efs
- 비암호화 EBS 볼륨은 그 자리에서 암호화로 못 바꾼다. 스냅샷 복사 시 암호화하는 절차를 묻는다. #exam/trap/encryption

%% 연습문제에서 틀리거나 헷갈린 지점을 위 형식으로 계속 추가합니다. 시험 직전 주에 `tag:#exam/trap` 검색으로 한 번에 모아 봅니다. %%

## 관련 노트

[[EC2]] · [[S3]] · [[KMS & 암호화]] · [[스토리지 추가 서비스]]
