---
service: Amazon SQS · SNS · Kinesis
exams: [saa, dva]
domains:
  - saa/resilient-architectures
  - dva/development
status: learning
confidence: 1
tags:
  - service
  - saa/integration
  - dva/integration
---

# SQS & SNS & Kinesis

> 애플리케이션 분리(decoupling)의 핵심 — 큐, 발행/구독, 실시간 스트리밍 비교.

## 개요

앱이 여러 개로 나뉘면 서로 통신해야 한다. 통신 방식은 두 가지다.

- **동기(synchronous)**: 앱이 앱을 직접 호출. 트래픽이 갑자기 몰리면(평소 10건인데 갑자기 1000건) 받는 쪽이 못 버틴다.
- **비동기(asynchronous)**: 중간에 큐·토픽·스트림을 두고 앱을 떼어 놓는다(decoupling). 중간 서비스가 앱과 별개로 알아서 확장하므로 트래픽 급증을 흡수한다.

이 비동기 중간 계층을 맡는 세 서비스를 비교하는 단원이다.

- **SQS** — 큐(queue) 모델. 생산자가 메시지를 넣고, 소비자가 **끌어와서(pull)** 처리한 뒤 지운다. 한 메시지는 한 소비자가 처리.
- **SNS** — 발행/구독(pub/sub) 모델. 한 토픽에 발행하면 구독자 전원에게 **밀어준다(push)**. 한 메시지를 여러 수신자가 받음.
- **Kinesis** — 실시간 스트리밍 모델. 대량의 작은 데이터를 모으고, 여러 소비자가 읽고, **나중에 다시 읽을 수도 있다(replay)**.

## 설계 관점 (SAA) #saa

![[AWS Certified Solutions Architect Slides v47.pdf#page=375]]

### SQS — 큐로 앱 분리하기

생산자가 `SendMessage`로 메시지를 넣고, 소비자가 `ReceiveMessage`로 끌어와 처리한 뒤 `DeleteMessage`로 지운다. **지우기 전까지 메시지는 큐에 남는다.**

- **Standard 큐**: 처리량 무제한. 단, **순서 보장 안 됨(best effort)** + **중복 가능(at-least-once)**. 메시지 보관은 기본 4일, 최대 14일. 메시지 한 건 최대 256KB.
- **FIFO 큐**: 순서 보장 + 중복 제거(Deduplication ID). 대신 처리량 제한 — 배치 없이 300 msg/s, 배치 쓰면 3000 msg/s. 순서는 **Message Group ID** 단위로 묶인다.
- 소비자는 한 번에 최대 10건씩 받고, 여러 대로 늘려 **수평 확장**할 수 있다(처리량 증가).

**Visibility Timeout(가시성 타임아웃)** — 자주 나온다.

- 소비자가 메시지를 받으면 그 메시지는 잠시 **다른 소비자에게 안 보이게** 숨겨진다. 기본 30초.
- 이 시간 안에 처리·삭제를 못 하면 메시지가 다시 보이게 되어 **다른 소비자가 또 처리한다(중복 처리)**.
- 처리에 시간이 더 필요하면 소비자가 `ChangeMessageVisibility`로 시간을 늘린다. 너무 길게 잡으면 소비자가 죽었을 때 재처리가 늦어지고, 너무 짧으면 중복이 늘어난다.

**Long Polling** — 큐가 비었을 때 소비자가 메시지를 기다리는 시간(1~20초, 20초 권장). 빈 응답을 받는 API 호출 수를 줄여 비용·지연을 낮춘다. Short Polling보다 권장.

**SQS로 분리하는 전형 패턴**:

- **큐 길이 기반 Auto Scaling**: `ApproximateNumberOfMessages` 지표를 [[CloudWatch & CloudTrail & Config\|CloudWatch]] Alarm으로 감시하다가, 쌓이면 [[ELB & Auto Scaling\|ASG]]로 소비자 인스턴스를 늘린다.
- **DB 쓰기 버퍼**: 트래픽이 몰리면 DB 쓰기가 유실될 수 있으니, 요청을 일단 큐에 넣고(infinitely scalable) 뒤쪽 앱이 자기 속도로 꺼내 DB에 넣는다.
- **계층 분리**: 프런트엔드와 백엔드 사이에 큐를 두면 양쪽이 독립적으로 확장된다.

보안: 전송 중 HTTPS, 저장 시 [[KMS & 암호화\|KMS]] 암호화, 접근 제어는 IAM 정책. **SQS Access Policy**(S3 버킷 정책과 비슷)로 다른 계정이나 다른 서비스(SNS·S3)가 큐에 쓰도록 허용한다.

### SNS — 한 메시지를 여러 곳에

수신자가 많을 때, 발행자가 각 수신자를 직접 호출하면 결합도가 높아진다. SNS는 발행자가 **토픽 하나에만** 보내고, 구독자들이 그 토픽을 구독해 알림을 받는 구조다.

- 구독자 종류: SQS, Lambda, [[CloudWatch & CloudTrail & Config\|Kinesis Data Firehose]], 이메일, SMS/모바일 푸시, HTTP(S) 엔드포인트.
- 토픽당 구독 최대 1250만, 계정당 토픽 최대 10만.
- CloudWatch Alarm·ASG·S3 이벤트·CloudFormation 등 많은 AWS 서비스가 SNS로 직접 알림을 보낸다.
- **FIFO 토픽**: SQS FIFO처럼 순서·중복 제거 지원. 구독자로 SQS FIFO 큐를 둘 수 있다.

**Fan-Out 패턴** — 이 단원에서 제일 중요한 시나리오.

- SNS 토픽에 한 번 발행 → 구독 중인 **여러 SQS 큐가 모두 받는다.**
- 완전히 분리되고 **데이터 유실이 없다**(각 SQS가 보관·재처리·지연 처리를 담당). 나중에 구독 큐를 더 붙이기도 쉽다.
- 받는 쪽 **SQS 큐의 Access Policy가 SNS의 쓰기를 허용**해야 한다.
- 다른 리전의 SQS 큐로도 전달 가능(Cross-Region).
- 대표 활용: **S3 이벤트를 여러 큐로** 보내기. S3는 같은 (이벤트 타입 + prefix) 조합에 이벤트 규칙을 하나만 둘 수 있어서, 같은 이벤트를 여러 큐에 보내려면 S3 → SNS → 여러 SQS로 fan-out 한다.
- 순서+중복 제거+fan-out이 동시에 필요하면 **SNS FIFO + SQS FIFO** 조합.

**Message Filtering** — 구독마다 JSON 필터 정책을 걸어 일부 메시지만 받게 한다(예: 주문 상태가 Cancelled인 것만). 필터가 없는 구독은 모든 메시지를 받는다.

### Kinesis — 실시간 스트리밍

클릭 스트림·IoT·로그처럼 작은 데이터가 끊임없이 들어오는 것을 모은다. SQS·SNS와 달리 **데이터를 보관하고 여러 소비자가 같은 데이터를 다시 읽을 수 있는 것**이 핵심 차이.

- **Kinesis Data Streams**: 실시간 수집·저장.
  - 보관 최대 365일, 소비자가 **다시 읽기(replay)** 가능, 데이터는 만료 전까지 **삭제 불가**.
  - 같은 **Partition ID**끼리는 순서 보장(= shard 단위 순서).
  - 용량 모드 둘 — **Provisioned**(shard 수를 직접 정함, shard당 입력 1MB/s·출력 2MB/s, shard당 과금) / **On-demand**(용량 관리 불필요, 최근 30일 피크 기준 자동 확장).
- **Amazon Data Firehose**(옛 이름 Kinesis Data Firehose): 스트리밍 데이터를 **목적지에 적재**하는 관리형 서비스.
  - 목적지: S3, Redshift, OpenSearch, 서드파티(Splunk·Datadog 등), 커스텀 HTTP.
  - **Near Real-Time**(크기·시간 기준으로 버퍼링), 자동 확장, 서버리스, **데이터 저장 안 함**, **replay 불가**.
  - Lambda로 변환(CSV→JSON 등), Parquet/ORC 변환·압축 지원.

### 셋 중 무엇을 고르나 (시험 단골 비교)

| | SQS | SNS | Kinesis Data Streams |
| --- | --- | --- | --- |
| 모델 | 큐 (소비자가 pull) | 발행/구독 (push) | 실시간 스트리밍 |
| 한 메시지 수신자 | 한 소비자 | 모든 구독자 | 여러 소비자 (각자 읽음) |
| 처리 후 데이터 | **삭제됨** | **유실**(미전달 시) | X일간 **보관**, replay 가능 |
| 순서 | FIFO 큐만 | FIFO 토픽만 | shard(Partition) 단위 |
| 용량 관리 | 불필요 | 불필요 | Provisioned / On-demand |
| 쓰는 경우 | 작업 분리·버퍼링 | 한 이벤트를 여러 곳에 알림 | 빅데이터·분석·ETL·실시간 처리 |

### Amazon MQ

SQS·SNS는 AWS 전용 프로토콜이다. 온프레미스에서 **MQTT·AMQP·STOMP·OpenWire·WSS** 같은 표준 프로토콜을 쓰던 앱을 클라우드로 옮길 때, 앱을 SQS/SNS용으로 고쳐 쓰는 대신 그대로 붙일 수 있게 해주는 관리형 메시지 브로커(RabbitMQ·ActiveMQ)다.

- SQS/SNS만큼 확장되지는 않는다. 서버에서 돌고, **Multi-AZ로 failover** 구성 가능(스토리지는 [[EBS & EFS\|EFS]] 공유).
- 큐 기능(~SQS)과 토픽 기능(~SNS)을 모두 가진다.
- 시험에서 "**기존 앱을 표준 프로토콜 그대로 마이그레이션**"이 나오면 Amazon MQ.

## 개발 관점 (DVA) #dva

![[AWS Certified Developer Slides v44.pdf#page=427]]

%% 이 시험의 관점에서 배운 내용을 정리합니다. %%

## 시험 함정

- "한 이벤트를 여러 시스템이 받되 유실되면 안 된다" → SNS 단독이 아니라 **SNS + SQS Fan-Out**(SQS가 보관·재처리 담당) #exam/trap/messaging
- "처리한 데이터를 나중에 다시 읽어야 한다(replay)" → SQS·SNS는 불가, **Kinesis Data Streams**만 가능 #exam/trap/messaging
- Firehose는 **replay·데이터 저장이 안 되는** near real-time 적재 서비스. "실시간 수집·재처리"는 Data Streams, "S3·Redshift로 적재"는 Firehose #exam/trap/messaging
- 메시지가 두 번 처리되는 문제 → **Visibility Timeout**이 처리 시간보다 짧기 때문. `ChangeMessageVisibility`로 연장 #exam/trap/messaging
- 순서·중복 제거가 필요하면 **FIFO**(SQS FIFO / SNS FIFO). 단 처리량 제한(300, 배치 3000 msg/s)을 기억 #exam/trap/messaging
- SNS·S3가 SQS 큐에 쓰게 하려면 **SQS Access Policy**로 허용해야 한다(IAM 역할이 아님) #exam/trap/messaging
- "표준 프로토콜(MQTT·AMQP 등) 쓰는 기존 앱을 그대로 마이그레이션" → SQS/SNS 재설계가 아니라 **Amazon MQ** #exam/trap/migration

## 관련 노트

[[Lambda]] · [[솔루션 아키텍처 패턴]] · [[ELB & Auto Scaling]] · [[CloudWatch & CloudTrail & Config]] · [[KMS & 암호화]] · [[S3]]
