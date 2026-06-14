---
service: Amazon DynamoDB
exams: [saa, dva, cloudops]
domains:
  - saa/high-performing-architectures
  - dva/development
  - cloudops/reliability
status: learning
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

%% 이 시험의 관점에서 배운 내용을 정리합니다. %%

## 운영 관점 (CloudOps) #cloudops

![[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=334]]

%% 이 시험의 관점에서 배운 내용을 정리합니다. %%

## 시험 함정

- 트래픽이 예측 불가·급증 → **On-Demand** 모드. 예측 가능·비용 절감 → **Provisioned**(+ Auto Scaling) #exam/trap/dynamodb
- "DynamoDB 읽기를 마이크로초로 캐싱, 코드 수정 없이" → **DAX**. ElastiCache는 집계 결과 저장 쪽 #exam/trap/dynamodb
- 멀티 리전에서 읽기·쓰기(active-active) → **Global Tables**, 단 **DynamoDB Streams 활성화가 전제** #exam/trap/dynamodb
- 세션 데이터 자동 정리 → **TTL** #exam/trap/dynamodb
- 아이템 최대 400KB, 더 큰 데이터는 S3에 두고 DynamoDB엔 포인터만 #exam/trap/dynamodb
- DynamoDB → S3 export는 **PITR을 먼저 켜야** 하고, export/import는 테이블 읽기/쓰기 용량을 소모하지 않음 #exam/trap/dynamodb
- 변경 이벤트에 반응(Lambda 호출·크로스 리전 복제) → **Streams**. 보관 길게·소비자 많이면 Kinesis Data Streams 옵션 #exam/trap/dynamodb

## 관련 노트

[[Lambda]] · [[RDS & Aurora & ElastiCache]] · [[API Gateway]] · [[SQS & SNS & Kinesis]] · [[S3]] · [[Cognito]]
