---
service: Amazon RDS · Aurora · ElastiCache
exams: [saa, dva, cloudops]
domains:
  - saa/resilient-architectures
  - dva/development
  - cloudops/reliability
status: reviewing
confidence: 1
tags:
  - service
  - saa/database
  - dva/database
  - cloudops/database
---

# RDS & Aurora & ElastiCache

> 관계형 DB와 인메모리 캐시 — 다중 AZ, 읽기 전용 복제본, 백업·복원.

## 개요

- **RDS** — SQL을 쓰는 관계형 DB의 관리형 서비스. 엔진은 Postgres, MySQL, MariaDB, Oracle, SQL Server, IBM DB2, Aurora. EC2에 직접 DB를 깔 때와 달리 프로비저닝·OS 패치·백업(특정 시점 복원)·모니터링·Multi-AZ를 AWS가 해준다. 대신 **인스턴스에 SSH는 못 한다.**
- **Aurora** — AWS가 만든 클라우드 전용 DB 엔진. MySQL·Postgres와 드라이버 호환이라 앱은 그대로 쓰면서, RDS MySQL 대비 5배 성능을 내세운다. 가용성 구조가 RDS와 근본적으로 다르다.
- **ElastiCache** — 관리형 Redis/Memcached. 인메모리 캐시로 DB 읽기 부하를 줄이고, 세션 저장소로 애플리케이션을 stateless로 만든다.

## 설계 관점 (SAA) #saa

![[AWS Certified Solutions Architect Slides v47.pdf#page=160]]

### RDS Storage Auto Scaling

스토리지가 모자라면 자동으로 늘려주는 기능. Maximum Storage Threshold(상한)만 정해두면 된다. 조건: 여유 공간이 10% 미만이고, 그 상태가 5분 이상 지속되고, 마지막 증설 후 6시간이 지났을 때. 부하를 예측하기 어려운 워크로드에 쓴다.

### Read Replica vs Multi-AZ — 가장 많이 나오는 구분

| | Read Replica | Multi-AZ |
| --- | --- | --- |
| 목적 | **읽기 확장** | **가용성(DR)** |
| 복제 | **ASYNC** (eventual consistency) | **SYNC** |
| 위치 | 같은 AZ·다른 AZ·다른 리전, 최대 15개 | 다른 AZ의 standby 1대 |
| 접근 | SELECT 전용 엔드포인트. 앱이 연결 문자열을 바꿔야 함 | **standby는 읽기도 못 한다.** DNS 하나로 자동 failover라 앱 수정 없음 |

- Read Replica의 대표 시나리오: 운영 DB에 영향 없이 리포팅·분석 쿼리를 돌릴 곳이 필요할 때. replica를 독립 DB로 승격할 수도 있다.
- 복제 트래픽 비용: **같은 리전이면 AZ가 달라도 무료**, cross-region은 유료.
- Multi-AZ failover 조건: AZ 장애, 네트워크 단절, 인스턴스·스토리지 장애. 스케일링 용도가 아니다.
- Single-AZ → Multi-AZ는 무중단이다. modify만 하면 내부적으로 스냅샷 → 새 AZ에 복원 → 동기화가 일어난다.
- Read Replica 자체를 Multi-AZ로 구성할 수도 있다(DR용).

### RDS Custom

Oracle·SQL Server 한정으로, 관리형의 장점을 유지하면서 **기반 OS와 DB에 접근(SSH, SSM Session Manager)** 할 수 있는 변형. 커스터마이징 전에 Automation Mode를 끄고 스냅샷을 떠두는 게 권장 절차다. "관리형인데 OS 접근이 필요하다"는 문제의 답.

### Aurora 아키텍처

- 스토리지: 데이터를 **3개 AZ에 6벌** 복제. 쓰기는 6벌 중 4벌, 읽기는 3벌이 살아 있으면 동작한다. 자가 복구(self-healing)에 수백 볼륨에 스트라이핑.
- 10GB 단위로 **최대 256TB까지 자동 증설**. 용량 계획이 필요 없다.
- master 1대가 쓰기를 받고, **최대 15개 read replica**(복제 지연 10ms 미만). master 장애 시 failover가 30초 안에 끝난다.
- 클라이언트는 **Writer Endpoint**(master를 가리킴)와 **Reader Endpoint**(replica들에 연결 분산)를 쓴다. replica는 auto scaling이 된다. 특정 replica 부분집합만 묶는 **Custom Endpoint**도 만들 수 있다(예: 분석 쿼리는 큰 인스턴스 replica로만).
- 비용은 RDS보다 20%쯤 비싸지만 효율이 좋다.

### Aurora 파생 기능

- **Aurora Serverless** — 사용량 기반 자동 기동·스케일링, 초 단위 과금. 간헐적이거나 예측 불가능한 워크로드용.
- **Aurora Global Database** — primary 리전 1개(읽기/쓰기) + 최대 10개 secondary 리전(읽기 전용). **복제 지연 1초 미만, 리전 승격 RTO 1분 미만.** cross-region DR 문제의 권장 답.
- **Aurora Machine Learning** — SQL로 SageMaker·Comprehend 예측을 호출(사기 탐지, 추천 등).
- **Babelfish** — Aurora PostgreSQL이 SQL Server의 T-SQL을 알아듣게 해주는 기능. SQL Server 앱을 코드 수정 거의 없이 Aurora로 이전할 때.
- **Aurora Cloning** — copy-on-write로 기존 클러스터에서 새 클러스터를 빠르게 만든다. 운영 DB에 영향 없이 스테이징 DB를 만들 때. 스냅샷 복원보다 빠르다.

### 백업과 복원

- **자동 백업**: 매일 풀 백업 + 5분마다 트랜잭션 로그 → 가장 오래된 백업부터 5분 전까지 **특정 시점 복원(PITR)**. 보존 1~35일. RDS는 0으로 끌 수 있지만 **Aurora는 끌 수 없다.**
- **수동 스냅샷**: 원하는 만큼 보존. **중지된 RDS도 스토리지 요금은 계속 나가므로**, 오래 세워둘 거면 스냅샷 뜨고 지웠다가 나중에 복원하는 게 싸다.
- 백업·스냅샷 복원은 항상 **새 DB를 만든다.**
- 온프레미스 이전: 백업 파일을 S3에 올려 RDS MySQL로 복원. Aurora MySQL은 Percona XtraBackup으로 만든 백업이어야 한다.

### 보안

- 저장 암호화는 [[KMS & 암호화|KMS]]로, **생성 시점에 정해야 한다.** master가 비암호화면 replica도 암호화 못 한다. 이미 만든 비암호화 DB는 스냅샷 → 암호화 복사 → 복원 절차로 바꾼다(EBS와 같은 패턴).
- 전송 구간은 기본 TLS 지원. 비밀번호 대신 **IAM 인증**으로 접속할 수도 있다.
- 네트워크 접근은 Security Group, 감사 로그는 CloudWatch Logs로 보낸다.

### RDS Proxy

DB 앞에 두는 관리형 연결 풀. 연결을 모아 공유해서 DB 부담(CPU·RAM, 열린 연결 수)을 줄인다. **Lambda처럼 연결이 폭주하는 클라이언트**가 대표 용도. failover 시간을 최대 66% 줄이고, IAM 인증 강제 + Secrets Manager 자격 증명 보관이 가능하다. **public 접근은 안 되고 VPC 안에서만** 쓴다.

### ElastiCache

- 캐시를 두면: 읽기 부하가 DB에서 캐시로 빠지고, 세션을 캐시에 두면 앱이 stateless가 된다. 단 **애플리케이션 코드 수정이 많이 필요하다** — 켜기만 하면 되는 기능이 아니다.
- 캐시 무효화 전략이 반드시 필요하다. 패턴: **Lazy Loading**(읽을 때 캐시에 적재, 오래된 데이터 가능), **Write Through**(쓸 때 캐시도 갱신, 항상 최신), **Session Store**(TTL로 만료).

| | Redis | Memcached |
| --- | --- | --- |
| 가용성 | **Multi-AZ + 자동 failover**, read replica | 없음(복제 없음) |
| 영속성 | AOF 영속화, 백업·복원 | 비영속 |
| 구조 | Sets·**Sorted Sets**(게임 리더보드의 답) | 멀티스레드, 샤딩으로 분할 |
| 보안 | IAM 인증, Redis AUTH 토큰, SSL | SASL 인증 |

고가용성·영속성·리더보드가 나오면 Redis, 단순 분산 캐시에 멀티스레드면 Memcached다.

## 개발 관점 (DVA) #dva

![[AWS Certified Developer Slides v44.pdf#page=138]]

%% 이 시험의 관점에서 배운 내용을 정리합니다. %%

## 운영 관점 (CloudOps) #cloudops

![[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=334]]

운영 강의는 Read Replica vs Multi-AZ, Aurora 6벌 복제, RDS Proxy, Redis vs Memcached 같은 구조를 SAA와 같게 다룬다(위 설계 관점으로 갈음). 운영 시험이 새로 묻는 건 **어떤 지표로 감시하고, 언제 어떻게 스케일하고, 문제를 어떻게 진단하느냐**다. "지표 이름 + 높을 때의 처방"을 외우는 게 핵심이다.

### RDS 모니터링 — CloudWatch · Enhanced Monitoring · Performance Insights

세 가지가 보는 위치가 다르다(시험에서 구분).

- **CloudWatch Metrics**: **하이퍼바이저에서** 수집(DB 바깥). `DatabaseConnections`·`SwapUsage`·`ReadIOPS`/`WriteIOPS`·`ReadLatency`/`WriteLatency`·`DiskQueueDepth`·`FreeStorageSpace`.
- **Enhanced Monitoring**: **DB 인스턴스 안의 에이전트에서** 수집. **프로세스·스레드별 CPU 사용**을 봐야 할 때(CloudWatch의 하이퍼바이저 지표로는 안 보임). 50개+ CPU·메모리·파일시스템·디스크 I/O 지표.
- **Performance Insights**: DB 부하를 시각화·분석. 부하를 **By Waits**(병목 리소스: CPU·IO·lock), **By SQL**(문제 쿼리), **By Hosts**(부하 큰 서버), **By Users**(부하 큰 사용자)로 쪼개 본다. **`DBLoad`** = 활성 세션 수. 어떤 쿼리가 DB를 누르는지 찾는다. Proactive Recommendation은 유료.

### RDS 로그 · 이벤트 알림

- **RDS Database Log Files**: General·Audit·Error·Slow Query 로그를 **CloudWatch Logs**로 보내고, **Metric Filter**로 에러 패턴을 잡아 **CloudWatch Alarm → SNS**로 DBA에게 알린다.
- **RDS Events & Event Subscriptions**: DB 인스턴스·스냅샷·파라미터/보안 그룹 관련 이벤트(예: 상태가 pending→running, 생성·failover)를 **SNS로 구독**해 알림받는다. **Event Source**(인스턴스·SG 등)와 **Event Category**(creation·failover 등)를 지정. **EventBridge로도** 전달된다.

### RDS Multi-AZ Failover와 백업 운영

- **Failover가 일어나는 경우**: primary **장애**·**OS 소프트웨어 패치**·**네트워크 단절**·**modify(인스턴스 타입 변경)**·**busy/무응답**·**스토리지 장애**, **AZ 장애**, 그리고 **Reboot with failover**로 수동 유발.
- **Backup vs Snapshot(운영 디테일)**: 자동 백업은 연속·PITR(유지보수 윈도에 수행). **스냅샷은 I/O를 써서 DB가 수 초~수 분 멈출 수 있다** — 단 **Multi-AZ면 standby에서 떠 master에 영향 없다.** 첫 스냅샷만 full, 이후 증분. **수동 스냅샷은 만료가 없고**, DB 삭제 시 **final snapshot**을 뜰 수 있다.
- **Snapshot 공유**: **수동 스냅샷만** 계정 간 공유(자동은 먼저 복사). **비암호화 또는 CMK로 암호화된 것만** 공유되고, 암호화 스냅샷을 공유하면 **그 CMK도 공유**해야 하며 대상 계정에 KMS 권한(`kms:DescribeKey`·`CreateGrant`·`Decrypt` 등)이 필요하다.

### Aurora 운영 — Backtracking · 지표

- **Aurora Backtracking**: 클러스터를 **시간상 앞뒤로 되감기(최대 72시간)**. **새 클러스터를 안 만드는 in-place 복구**라 빠르다. **Aurora MySQL만.** (스냅샷 복원·PITR과 구분 — 그쪽은 새 클러스터를 만든다.)
- **Aurora CloudWatch 지표**: **`AuroraReplicaLag`**(primary→replica 복제 지연)·`AuroraReplicaLagMaximum`/`Minimum`(클러스터 내 최대·최소). **lag이 크면** 어떤 replica에 붙느냐에 따라 사용자가 다른 데이터를 본다(eventual consistency). 그 외 `DatabaseConnections`·`InsertLatency`.

### ElastiCache 스케일링 — Redis · Memcached

- **Redis (Cluster Mode Disabled)**: **수평** = read replica 추가/제거(**최대 5개**), **수직** = 더 큰/작은 노드 타입(내부적으로 새 node group 만들고 복제 후 DNS 갱신).
- **Redis (Cluster Mode Enabled)**: **Online Scaling**(무중단, 성능 약간 저하) vs **Offline Scaling**(중단되지만 노드 타입 변경·엔진 업그레이드 등 추가 설정 가능). 수평은 **Resharding**(샤드 추가/제거)·**Shard Rebalancing**(키 공간 균등 분배).
- **Memcached**: 클러스터 **1~40 노드**(soft limit). 수평은 노드 추가/제거 + **Auto Discovery**(configuration endpoint으로 모든 노드를 자동 식별 — 앱이 노드를 일일이 몰라도 됨). **수직 스케일은 새 클러스터를 만들어 옮긴다(노드는 빈 채로 시작).**

### 캐시 모니터링 지표 (시험 — 높을 때의 처방)

Redis·Memcached 공통으로 자주 나온다.

- **`Evictions`**: 메모리가 차서 만료 안 된 항목을 쫓아낸 수. **처방: eviction policy(LRU 등) 설정, 더 큰 노드(수직)나 노드 추가(수평).**
- **`CPUUtilization`**: 호스트 CPU. **처방: 더 큰 노드나 노드 추가.**
- **`SwapUsage`**: **50MB를 넘기면 안 된다.** 처방: 예약 메모리를 충분히.
- **`CurrConnections`**: 동시 연결 수가 비정상이면 **앱 동작을 점검.**
- Redis는 추가로 `DatabaseMemoryUsagePercentage`·`ReplicationLag`·`ReplicationBytes`, Memcached는 `FreeableMemory`.

## 시험 함정

- Read Replica는 ASYNC·읽기 확장, Multi-AZ는 SYNC·가용성. 문제가 "성능"을 물으면 replica, "장애 대비"를 물으면 Multi-AZ다. #exam/trap/rds
- Multi-AZ의 standby는 읽기 트래픽도 못 받는다. 읽기 분산은 Read Replica의 몫이다. #exam/trap/rds
- Read Replica로 읽으려면 앱이 연결 문자열을 바꿔야 한다. Multi-AZ failover는 DNS가 같아서 앱 수정이 없다. #exam/trap/rds
- Read Replica 복제 비용은 같은 리전이면 무료, cross-region이면 유료다. #exam/trap/cost-optimization
- 중지된 RDS도 스토리지 요금이 나간다. 장기 중지는 스냅샷 후 삭제가 답이다. #exam/trap/cost-optimization
- RDS 암호화는 생성 시점에만 켤 수 있다. 기존 DB는 스냅샷을 암호화 복사해서 복원한다. #exam/trap/encryption
- 자동 백업은 RDS에서는 끌 수 있지만(보존 0) Aurora에서는 끌 수 없다. #exam/trap/rds
- Lambda가 DB 연결을 고갈시키는 문제의 답은 RDS Proxy다. RDS Proxy는 public 접근이 안 된다. #exam/trap/rds
- 게임 리더보드·실시간 순위는 Redis Sorted Sets다. Memcached에는 없다. #exam/trap/elasticache
- 캐시 가용성·영속성이 요구되면 Redis, 멀티스레드 단순 캐시면 Memcached. #exam/trap/elasticache
- RDS 모니터링 위치 구분: **CloudWatch=하이퍼바이저**, **Enhanced Monitoring=인스턴스 내 에이전트**(프로세스/스레드별 CPU). 부하 쿼리 분석은 **Performance Insights**(Waits·SQL·Hosts·Users, DBLoad=활성 세션). #exam/trap/rds
- RDS 이벤트 알림 → **Event Subscriptions**(SNS, Event Source+Category)·EventBridge. DB 로그는 CloudWatch Logs→Metric Filter→Alarm→SNS. #exam/trap/rds
- RDS 스냅샷은 **수동만 공유** 가능(자동은 복사 먼저), 암호화 스냅샷 공유 시 **CMK도 공유** + 대상 계정 KMS 권한. #exam/trap/rds
- **Aurora Backtracking**: 최대 72시간 되감기, **새 클러스터 안 만듦(in-place)**, **Aurora MySQL만**. 스냅샷/PITR은 새 클러스터 생성. #exam/trap/rds
- Aurora **AuroraReplicaLag**가 크면 replica마다 다른 데이터(eventual consistency). #exam/trap/rds
- 캐시 **Evictions↑** → eviction policy(LRU)·노드 키우기/추가. **SwapUsage는 50MB 넘기면 안 됨**. #exam/trap/elasticache
- Redis 수평 스케일 read replica **최대 5**. Memcached 수직 스케일은 **새 클러스터(빈 채 시작)**, 노드 자동 인식은 **Auto Discovery**(configuration endpoint). #exam/trap/elasticache

%% 연습문제에서 틀리거나 헷갈린 지점을 위 형식으로 계속 추가합니다. 시험 직전 주에 `tag:#exam/trap` 검색으로 한 번에 모아 봅니다. %%

## 관련 노트

[[DynamoDB]] · [[VPC]] · [[KMS & 암호화]] · [[Lambda]] · [[재해 복구 & 마이그레이션]] · [[CloudWatch & CloudTrail & Config]] · [[SQS & SNS & Kinesis]]
