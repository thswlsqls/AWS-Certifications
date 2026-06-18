---
service: Amazon DynamoDB
exams: [saa, dva, cloudops]
domains:
  - saa/high-performing-architectures
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

# DynamoDB

> 서버리스 NoSQL — 파티션 키 설계, RCU/WCU, 인덱스(LSI/GSI), DAX, Streams.

## 개요

완전 관리형 **NoSQL** 데이터베이스. 관계형이 아니고, 여러 AZ에 자동 복제돼 가용성이 높다. 트래픽이 아무리 커져도(초당 수백만 요청, 수조 행) 일정한 한 자리 밀리초 응답을 내는 분산 DB라서, RDS로는 버거운 규모·서버리스 앱의 데이터 저장소로 쓴다. 유지보수·패치가 없고 IAM으로 권한을 건다.

구조는 **테이블 → 아이템(행) → 속성(열)**. 테이블마다 **Primary Key**를 만들 때 정하고, 아이템은 무한히 넣을 수 있다. 스키마가 고정이 아니라 속성을 나중에 추가할 수 있어 **스키마를 빠르게 바꿔야 하는** 경우에 맞는다. 아이템 하나는 최대 400KB.

## 설계 관점 (SAA) #saa

![[AWS Certified Solutions Architect Slides v47.pdf#page=470]]

> 추천 학습 순서상 "큰 그림" 자리입니다. 인덱스(LSI/GSI)·쓰기 패턴 등 세부는 3단계(DVA)에서 채웁니다. 여기서는 SAA 시나리오에 나오는 선택 기준만 정리합니다.

### Primary Key

- **Partition Key만**: 값이 고르게 흩어지도록 잡는다.
- **Partition Key + Sort Key**: 같은 파티션 키 안에서 정렬 키로 구분(예: User_ID + Game_ID).

### 용량 모드 — Provisioned vs On-Demand (단골)

테이블의 읽기·쓰기 처리량을 어떻게 관리하느냐다.

| | Provisioned (기본) | On-Demand |
| --- | --- | --- |
| 방식 | 초당 읽기/쓰기 수(RCU·WCU)를 **미리 지정** | **자동으로** 부하에 맞춰 오르내림 |
| 용량 계획 | 필요 | 불필요 |
| 비용 | 싸다. Auto Scaling도 가능 | 비싸다($$$) |
| 적합 | 트래픽이 **예측 가능**할 때 | **예측 불가·갑작스러운 급증** |

→ "트래픽을 예측 못 한다 / 스파이크가 튄다"면 On-Demand, 안정적이고 비용을 아끼려면 Provisioned.

### DAX — 읽기 캐시

- DynamoDB 전용 **인메모리 캐시**. 같은 항목을 자주 읽어 생기는 **읽기 혼잡(read congestion)**을 풀고, 캐시 적중 시 **마이크로초** 응답.
- 기존 DynamoDB API 그대로라 **앱 코드 수정이 필요 없다**. 기본 TTL 5분.
- **DAX vs [[RDS & Aurora & ElastiCache|ElastiCache]]**: 개별 객체 캐시·쿼리/스캔 결과 캐시는 **DAX**. 집계(aggregation) 결과를 따로 저장하는 건 ElastiCache. 시험에서 "DynamoDB 읽기를 캐싱"이면 DAX.

### Streams — 변경 이벤트 처리

테이블의 아이템 변경(생성·수정·삭제)을 **순서대로** 흘려보낸다. 변경에 반응해 이메일 보내기, 실시간 분석, 파생 테이블 적재, **크로스 리전 복제**, Lambda 호출 등에 쓴다.

- **DynamoDB Streams**: 보관 24시간, 소비자 수 제한. Lambda 트리거나 KCL 어댑터로 처리.
- **Kinesis Data Streams(신규)**: 보관 1년, 소비자 많음. Lambda·Kinesis Data Analytics·Firehose·Glue 등으로 처리.

### Global Tables — 멀티 리전

- 한 테이블을 여러 리전에서 **낮은 지연**으로 읽고 쓴다. **active-active 복제**(어느 리전에서나 읽기·쓰기).
- **전제조건: DynamoDB Streams를 켜야** 한다.

### TTL — 자동 만료

- 지정한 만료 타임스탬프가 지나면 아이템을 **자동 삭제**. 웹 **세션 데이터**, 최신 데이터만 남기기, 규제상 보관 기간 관리 등에 쓴다.

### 백업·S3 연동

- **PITR(Point-in-Time Recovery)**: 최근 35일 안의 임의 시점으로 복구. 켜두면 복구 시 **새 테이블**이 만들어진다.
- **On-demand 백업**: 명시적으로 지울 때까지 보관하는 풀 백업. AWS Backup으로 크로스 리전 복사 가능.
- **S3로 내보내기/가져오기**: PITR을 켜야 export 가능. S3에 내보내 [[S3|Athena]]로 분석하거나, S3에서 가져와 새 테이블 생성(쓰기 용량 소모 안 함). export/import는 **읽기/쓰기 용량에 영향 없음**.

## 개발 관점 (DVA) #dva

![[AWS Certified Developer Slides v44.pdf#page=603]]

SAA 섹션이 "언제 어떤 옵션을 고르나"였다면, 여기서는 코드에서 직접 만나는 것들을 정리한다. 용량을 숫자로 계산하는 법, API마다 무엇이 다른지, 동시 수정을 어떻게 막는지, 인덱스 두 종류를 언제 쓰는지가 DVA에서 가장 많이 나온다.

### 용량 단위 계산 (시험 단골 — 숫자 문제)

읽기·쓰기 처리량은 **RCU(읽기)** 와 **WCU(쓰기)** 라는 단위로 센다. Provisioned 모드면 이 값을 미리 지정하고, 계산 문제로 출제된다.

- **WCU 1개** = 1KB 이하 아이템을 **초당 1번 쓰기**. 아이템이 1KB를 넘으면 KB 단위로 **올림**해서 그만큼 더 쓴다.
  - 예: 2KB 아이템을 초당 10번 쓰기 → `10 × (2KB/1KB)` = **20 WCU**.
  - 예: 4.5KB 아이템을 초당 6번 → 5KB로 올림 → `6 × (5/1)` = **30 WCU**.
- **RCU 1개** = 4KB 이하 아이템을 **Strongly Consistent Read 초당 1번**, 또는 **Eventually Consistent Read 초당 2번**. 4KB 넘으면 4KB 단위로 **올림**.
  - 예: 4KB 아이템 Strongly read 초당 10번 → `10 × (4/4)` = **10 RCU**.
  - 예: 12KB 아이템 Eventually read 초당 16번 → `(16/2) × (12/4)` = **24 RCU**.
  - 예: 6KB 아이템 Strongly read 초당 10번 → 8KB로 올림 → `10 × (8/4)` = **20 RCU**.

올림 기준이 쓰기는 **1KB**, 읽기는 **4KB**라는 점, Eventually read가 RCU를 절반만 먹는다는 점이 함정 포인트다.

### 읽기 일관성 — Eventually vs Strongly

- **Eventually Consistent Read (기본)**: 쓰기 직후 읽으면 복제가 덜 끝나 옛 값이 나올 수 있다.
- **Strongly Consistent Read**: 쓰기 직후에도 최신 값을 보장. API 호출(`GetItem`, `BatchGetItem`, `Query`, `Scan`)에서 `ConsistentRead=true`로 켠다. 대신 **RCU를 2배** 먹는다.
- "방금 쓴 값을 반드시 읽어야 한다" → Strongly. 비용·처리량이 중요하고 약간의 지연 허용 → Eventually.

### Throttling — ProvisionedThroughputExceededException

지정한 RCU·WCU를 넘기면 이 예외가 난다. 원인과 해법:

- **Hot Key / Hot Partition**: 특정 파티션 키 하나에 읽기·쓰기가 몰림(인기 아이템 등). → 파티션 키를 더 고르게 분산, 읽기 쏠림이면 **DAX**.
- **너무 큰 아이템**: RCU·WCU가 아이템 크기에 비례하므로 큰 아이템은 더 많이 소모.
- **공통 대응**: **지수 백오프(exponential backoff)** 재시도 — AWS SDK에 이미 들어 있다. Provisioned 모드는 잠깐의 초과를 **Burst Capacity**로 흡수하지만, 그걸 다 쓰면 예외가 난다.

### 쓰기 API

- **PutItem**: 아이템 새로 만들거나 같은 Primary Key 아이템을 **통째로 덮어쓰기**. WCU 소모.
- **UpdateItem**: 일부 속성만 수정(없으면 새로 생성). **Atomic Counter**(숫자 속성을 조건 없이 증가)를 이걸로 구현.
- **조건부 쓰기(Conditional Writes)**: 조건을 만족할 때만 쓰기/수정/삭제하고 아니면 에러. 동시 접근 제어에 쓰고 **성능 영향 없다**. `PutItem`·`UpdateItem`·`DeleteItem`·`TransactWriteItems`에 `Condition expression`을 건다.
  - 연산자: `attribute_exists`, `attribute_not_exists`, `attribute_type`, `contains`, `begins_with`, `size`, `IN`, `between` 등.
  - `attribute_not_exists(partition_key)` → 기존 아이템을 **덮어쓰지 않게** 보장(없을 때만 생성).
  - **Condition Expression(쓰기 조건) vs Filter Expression(읽기 결과 거르기)** 을 구분하는 문제가 나온다.

### 읽기 API — GetItem / Query / Scan

- **GetItem**: Primary Key로 아이템 하나 읽기. `ProjectionExpression`으로 필요한 속성만 가져온다.
- **Query**: `KeyConditionExpression`으로 조회. **파티션 키는 `=`만**(필수), 정렬 키는 `=, <, <=, >, >=, between, begins_with`(선택). `FilterExpression`은 조회 후 **키가 아닌 속성**으로 추가 필터(키 속성엔 못 씀). 결과는 `Limit` 또는 **최대 1MB**까지, 페이지네이션 가능. 테이블·LSI·GSI에 모두 가능.
- **Scan**: 테이블 전체를 읽고 거른다 → **비효율, RCU 많이 먹음**. 한 번에 1MB. 빠르게 하려면 **Parallel Scan**(여러 워커가 구간 나눠 동시 스캔, 처리량·RCU 더 소모). `ProjectionExpression`·`FilterExpression`은 RCU에 영향 없다.
- "특정 키로 조회"면 Query, "전체 훑기"면 Scan인데 시험은 보통 **Scan을 피하라**는 쪽으로 유도한다.

### Batch 연산

API 호출 수를 줄여 지연을 아낀다. 내부적으로 병렬 처리되고, **일부만 실패할 수 있어** 실패분은 재시도해야 한다.

- **BatchWriteItem**: `PutItem`/`DeleteItem` 최대 **25개**, 최대 16MB(아이템당 400KB). **업데이트는 못 함**(UpdateItem 따로). 실패분은 `UnprocessedItems`로 돌려준다 → 지수 백오프나 WCU 증설.
- **BatchGetItem**: 여러 테이블에서 최대 **100개** 읽기, 16MB. 실패분은 `UnprocessedKeys`.

### LSI vs GSI (핵심)

기본 Primary Key 말고 다른 기준으로 조회하고 싶을 때 만드는 인덱스.

| | **LSI** (Local Secondary Index) | **GSI** (Global Secondary Index) |
| --- | --- | --- |
| 바꾸는 것 | **정렬 키만** 다른 걸로(파티션 키는 테이블과 동일) | **파티션 키(+정렬 키)** 를 새로 |
| 생성 시점 | **테이블 만들 때만** | 생성 후에도 **추가·수정 가능** |
| 개수 | 테이블당 최대 5개 | 더 유연 |
| 용량 | 테이블의 RCU·WCU를 **공유** | **자체 RCU·WCU를 따로 프로비저닝** |
| Throttling | 별도 고려 없음 | **GSI 쓰기가 막히면 메인 테이블 쓰기까지 막힘** |

→ GSI는 파티션 키 선택과 WCU 할당을 신중히. GSI 쓰기 스로틀이 메인 테이블로 번진다는 게 시험 포인트. 두 인덱스 모두 **Attribute Projection**(KEYS_ONLY / INCLUDE / ALL)으로 인덱스에 담을 속성을 고른다.

### 낙관적 잠금(Optimistic Locking)

아이템에 **version 속성**을 두고, "version이 내가 읽었을 때 값과 같을 때만 수정"하는 조건부 쓰기를 건다. 두 클라이언트가 동시에 같은 아이템을 고치면 한쪽만 성공한다. 내가 읽은 뒤 남이 바꾸지 않았음을 보장하는 방식.

### Transactions — ACID

여러 아이템(여러 테이블 포함)을 **전부 성공 아니면 전부 실패**로 묶는다. ACID 보장.

- 연산: **TransactGetItems**(읽기 묶음), **TransactWriteItems**(`PutItem`/`UpdateItem`/`DeleteItem` 묶음).
- **용량을 2배** 먹는다 — 아이템마다 prepare·commit 2단계라서. (시험 계산 단골)
  - 예: 5KB 아이템 트랜잭션 쓰기 초당 3번 → `3 × (5/1) × 2` = **30 WCU**.
  - 예: 5KB 아이템 트랜잭션 읽기 초당 5번 → 8KB 올림 → `5 × (8/4) × 2` = **20 RCU**.
- 용도: 금융 거래, 주문 처리, 멀티플레이어 게임처럼 "둘 다 되거나 둘 다 안 되거나"가 필요한 경우.

### Streams — 개발 디테일

(개념·소비처는 SAA 섹션 참고) 개발자가 챙길 점:

- 스트림에 담을 정보 선택: **KEYS_ONLY / NEW_IMAGE / OLD_IMAGE / NEW_AND_OLD_IMAGES**.
- 스트림은 Kinesis처럼 **샤드**로 구성되지만 샤드는 AWS가 자동 관리.
- **스트림을 켜기 전의 변경은 소급 기록되지 않는다.**
- Lambda로 처리하려면 **Event Source Mapping**을 정의(Lambda 권한 필요). 이 경우 Lambda는 **동기 호출**된다.

### PartiQL

DynamoDB용 **SQL 비슷한 문법**. `SELECT`/`INSERT`/`UPDATE`/`DELETE`를 쓰고 배치도 지원. 콘솔·NoSQL Workbench·API·CLI·SDK에서 실행. 단, SQL 전체가 아니라 일부 문만 된다.

### Write Sharding — Hot Partition 회피

파티션 키 값 종류가 적으면(예: 후보 A·B 둘뿐인 투표 앱) 파티션이 몰려 Hot Partition이 된다. 해법은 **파티션 키 값에 접미사를 붙여** 더 흩뜨리는 것 — **랜덤 접미사** 또는 **계산된 접미사**.

### 보안 — Fine-Grained Access Control

웹/모바일 앱에서 클라이언트가 DynamoDB에 **직접** 접근하게 할 때, Cognito Identity Pools나 Web Identity Federation으로 임시 자격증명을 받고 **IAM Role + Condition**으로 권한을 좁힌다.

- **LeadingKeys**: 파티션 키(= Primary Key) 기준으로 **자기 행만** 접근하게 제한(`dynamodb:LeadingKeys`에 `${cognito-identity.amazonaws.com:sub}`).
- **Attributes**: 사용자가 볼 수 있는 **특정 속성**만 제한.

이 밖에 VPC Endpoint로 인터넷 없이 접근, KMS 저장 암호화·TLS 전송 암호화, DMS로 MongoDB·Oracle·MySQL·S3에서 DynamoDB로 이관, 로컬 개발용 **DynamoDB Local**이 있다.

### CLI 페이지네이션

`--projection-expression`(가져올 속성), `--filter-expression`(반환 전 필터). 페이지네이션은 `--page-size`(한 번에 여러 API 호출로 나눠 받기, 기본 1000), `--max-items`(보여줄 최대 개수, `NextToken` 반환), `--starting-token`(이전 `NextToken`부터 이어 받기).

## 운영 관점 (CloudOps) #cloudops

> CloudOps(SOA-C03) v41 슬라이드에는 **DynamoDB 전용 운영 단원이 없다.** 데이터베이스 섹션(`Databases in AWS`, p.334~373)은 RDS·Aurora·ElastiCache만 다루고, p.374부터 모니터링 단원으로 넘어간다. 즉 이 시험을 위해 따로 읽을 DynamoDB 슬라이드가 없다(추천 학습 순서 #7의 p.334 링크는 RDS 단원을 가리킨다).

운영 시험에서 DynamoDB는 신뢰성 도메인의 보기로 가볍게 나오는 정도이고, 판단 기준은 이미 위 설계 관점(SAA)·개발 관점(DVA)에 정리돼 있다. 운영 각도에서 다시 짚을 것만 추린다.

- **용량·스케일링**: 트래픽이 예측 불가·급증이면 **On-Demand**, 예측 가능하고 비용을 아끼면 **Provisioned + Auto Scaling**. "스로틀이 나는데 손대지 않고 견디게" 만드는 운영 답은 이 둘 중 하나.
- **스로틀 대응**: `ProvisionedThroughputExceededException`은 Hot Partition·큰 아이템·용량 부족이 원인. 읽기 쏠림은 **DAX**, 쓰기 쏠림은 **write sharding**, 공통으로 지수 백오프(위 DVA 섹션).
- **가용성·복구**: 멀티 리전 active-active는 **Global Tables**(Streams 전제). 시점 복구는 **PITR(35일)**, 장기 보관은 **On-demand 백업**(AWS Backup으로 크로스 리전 복사).
- **모니터링**: DynamoDB도 다른 서비스처럼 CloudWatch로 처리량·스로틀·지연을 감시한다 — 구체적 방법은 [[CloudWatch & CloudTrail & Config]]에서 다룬다.

## 시험 함정

- 트래픽이 예측 불가·급증 → **On-Demand** 모드. 예측 가능·비용 절감 → **Provisioned**(+ Auto Scaling) #exam/trap/dynamodb
- "DynamoDB 읽기를 마이크로초로 캐싱, 코드 수정 없이" → **DAX**. ElastiCache는 집계 결과 저장 쪽 #exam/trap/dynamodb
- 멀티 리전에서 읽기·쓰기(active-active) → **Global Tables**, 단 **DynamoDB Streams 활성화가 전제** #exam/trap/dynamodb
- 세션 데이터 자동 정리 → **TTL** #exam/trap/dynamodb
- 아이템 최대 400KB, 더 큰 데이터는 S3에 두고 DynamoDB엔 포인터만 #exam/trap/dynamodb
- DynamoDB → S3 export는 **PITR을 먼저 켜야** 하고, export/import는 테이블 읽기/쓰기 용량을 소모하지 않음 #exam/trap/dynamodb
- 변경 이벤트에 반응(Lambda 호출·크로스 리전 복제) → **Streams**. 보관 길게·소비자 많이면 Kinesis Data Streams 옵션 #exam/trap/dynamodb
- 용량 계산: 쓰기 올림 기준 **1KB**, 읽기 올림 기준 **4KB**. Eventually read는 RCU **절반**, Strongly read는 **2배** #exam/trap/dynamodb
- 트랜잭션(TransactWriteItems/TransactGetItems)은 용량 **2배** 소모 #exam/trap/dynamodb
- **Condition Expression = 쓰기 조건**, **Filter Expression = 읽기 결과 거르기**. Query의 FilterExpression은 **키가 아닌 속성**에만 #exam/trap/dynamodb
- Query 파티션 키는 **`=` 연산자만** 가능, 정렬 키만 범위 연산 가능 #exam/trap/dynamodb
- **LSI**는 테이블 만들 때만·테이블 RCU/WCU 공유 / **GSI**는 나중에 추가 가능·자체 용량. **GSI 쓰기 스로틀이 메인 테이블 스로틀로 번진다** #exam/trap/dynamodb
- BatchWriteItem은 **업데이트 불가**(PutItem/DeleteItem만), 최대 25개. 실패분은 UnprocessedItems로 재시도 #exam/trap/dynamodb
- 낙관적 잠금 = **조건부 쓰기 + version 속성**. "수정 전 안 바뀌었는지 확인" #exam/trap/dynamodb
- DynamoDB Streams는 **켜기 전 변경을 소급 기록하지 않음**. Lambda 연동은 Event Source Mapping, 호출은 **동기** #exam/trap/dynamodb
- 클라이언트가 자기 행만 접근 → **Fine-Grained Access Control의 LeadingKeys**(Cognito Identity Pools + IAM Role Condition) #exam/trap/dynamodb

## 관련 노트

[[Lambda]] · [[RDS & Aurora & ElastiCache]] · [[API Gateway]] · [[SQS & SNS & Kinesis]] · [[S3]] · [[Cognito]] · [[CloudWatch & CloudTrail & Config]]
