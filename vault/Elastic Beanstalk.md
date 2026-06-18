---
service: AWS Elastic Beanstalk
exams: [dva]
domains:
  - dva/deployment
status: reviewing
confidence: 1
tags:
  - service
  - dva/deployment
---

# Elastic Beanstalk

> 애플리케이션 배포 PaaS — 배포 정책(All at once/Rolling/Immutable/Blue-Green) 암기 필수.

## 개요

애플리케이션을 AWS에 올리는 일을 **개발자 시각**으로 묶어 주는 PaaS. 대부분의 웹 앱은 결국 ALB + Auto Scaling Group + EC2 + RDS라는 같은 구조를 쓰는데, 개발자는 그걸 일일이 세팅하지 않고 **코드만 올리면 실행되길** 원한다. Elastic Beanstalk이 용량 프로비저닝·로드 밸런싱·스케일링·헬스 모니터링·인스턴스 구성을 알아서 하고, 개발자는 **애플리케이션 코드만** 책임진다.

- 내부적으로 우리가 이미 아는 [[EC2]]·[[ELB & Auto Scaling|ASG·ELB]]·[[RDS & Aurora & ElastiCache|RDS]]를 쓴다. 관리형이지만 **설정은 전부 내가 통제**할 수 있다.
- **Beanstalk 자체는 무료, 그 아래 인스턴스 비용만 낸다.**
- **내부 동작은 [[CloudFormation]]에 의존**한다. 그래서 `.ebextensions`로 ElastiCache·S3 같은 CloudFormation 리소스를 같이 띄울 수 있다.

## 개발 관점 (DVA) #dva

![[AWS Certified Developer Slides v44.pdf#page=354]]

### 구성 요소

- **Application**: 환경·버전·설정을 묶은 단위.
- **Application Version**: 코드의 한 버전(이터레이션).
- **Environment**: 한 application version을 실행하는 AWS 리소스 묶음. **한 번에 한 버전만** 돈다. dev·test·prod처럼 여러 개 만들 수 있다.

### 환경 Tier — Web Server vs Worker

- **Web Server Tier**: ELB + ASG로 HTTP 요청을 받는 일반 웹 환경.
- **Worker Tier**: **SQS 큐 메시지를 당겨(pull) 처리**하는 환경. **SQS 메시지 수에 따라 스케일**한다. Web Server Tier가 큐에 작업을 넣고 Worker Tier가 처리하는 식으로 분리한다.
- → "오래 걸리는 비동기 작업을 분리" → Worker Tier + SQS.

### 배포 모드

- **Single Instance**: EC2 1대 + Elastic IP. **dev에 적합.**
- **High Availability with Load Balancer**: ALB + ASG + Multi-AZ RDS(Master/Standby). **prod에 적합.**

### 배포 정책 (시험 핵심 — 6종 비교)

업데이트를 어떻게 굴릴지 고르는 문제다. 다운타임·비용·롤백 난이도가 갈린다.

| 정책 | 동작 | 다운타임 | 추가 비용 | 비고 |
| --- | --- | --- | --- | --- |
| **All at once** | 전부 한 번에 교체 | **있음**(잠깐 끊김) | 없음 | 가장 빠름. dev·빠른 반복용 |
| **Rolling** | 한 번에 몇 대(bucket)씩 교체 | 없음 | 없음 | 잠깐 **용량 미달**로 운영. 두 버전 공존 |
| **Rolling with additional batches** | 새 인스턴스를 추가로 띄워 batch 교체 | 없음 | **소액** | **용량 유지**하며 교체. 두 버전 공존. prod에 적합 |
| **Immutable** | 새 ASG에 새 인스턴스로 전부 배포 후 한꺼번에 교체 | 없음 | **높음(용량 2배)** | 가장 오래 걸림. **롤백 빠름**(새 ASG만 버리면 됨). prod에 적합 |
| **Blue/Green** | 새 환경(green)에 배포 후 전환 | 없음 | 높음 | EB의 **직접 기능 아님**. Route 53 가중치로 일부 트래픽 → 검증 후 **swap URL** |
| **Traffic Splitting** | 임시 ASG에 새 버전, **트래픽 일부 %**만 보냄 | 없음 | 있음 | **카나리 테스트**. 헬스 보고 문제면 **자동 롤백(빠름)** |

핵심 갈림길: 다운타임 허용·빠름 → All at once. 용량 유지하며 무중단 → Rolling with additional batches. 무중단 + 빠른 롤백(prod) → Immutable. 환경 통째로 바꿔 검증 후 전환 → Blue/Green. 새 버전에 소량 트래픽으로 카나리 → Traffic Splitting.

### 배포·운영 디테일

- **배포 과정**: 의존성 기술(Python `requirements.txt`, Node.js `package.json`) → 코드를 **zip**으로 → 콘솔이나 CLI로 새 버전 업로드 → 배포. EB가 각 EC2에 zip을 풀고 의존성을 받아 앱을 띄운다.
- **EB CLI**: `eb create`·`eb deploy`·`eb status`·`eb health`·`eb logs`·`eb config`·`eb terminate` 등. 자동 배포 파이프라인에 쓴다.
- **Lifecycle Policy**: application version은 **최대 1,000개**까지. 안 지우면 더는 배포가 안 된다. **시간 기준** 또는 **개수(공간) 기준**으로 오래된 버전을 정리한다. **현재 쓰는 버전은 안 지운다.** S3의 source bundle은 안 지우는 옵션도 있다.
- **`.ebextensions`**: UI에서 만지는 설정을 코드로 둔다. 소스 루트의 **`.ebextensions/`** 디렉터리에 **`.config`**(YAML/JSON) 파일. `option_settings`로 기본값을 바꾸고, RDS·ElastiCache·DynamoDB 같은 리소스를 추가할 수 있다. **여기서 만든 리소스는 환경이 사라지면 같이 지워진다.**

### Cloning과 마이그레이션

- **Cloning**: 환경을 같은 설정으로 복제(LB 타입·설정, RDS 타입, 환경 변수 보존 — 단 **RDS 데이터는 보존 안 됨**). "test 버전을 같은 구성으로 띄울 때" 쓴다.
- **LB 타입 변경**: 환경 생성 후에는 **ELB 타입을 못 바꾼다**(설정만 변경 가능). 바꾸려면 새 환경을 만들어 배포하고 **CNAME swap이나 Route 53 변경**. (clone으로는 LB 타입을 못 바꾸니 새로 만들어야 한다.)
- **RDS 분리**: RDS를 Beanstalk이 띄우면 **dev/test엔 편하지만 prod엔 위험**하다 — DB 수명이 환경 수명에 묶여 환경을 지우면 DB도 사라진다. prod는 **RDS를 따로 만들고 연결 문자열만** EB에 준다. 이미 묶인 걸 떼는 마이그레이션: ① RDS 스냅샷 → ② RDS를 **삭제 보호** → ③ RDS 없는 새 EB 환경을 만들어 기존 RDS를 가리킴 → ④ CNAME swap/Route 53 변경 → ⑤ 옛 환경 종료(RDS는 안 지워짐) → ⑥ DELETE_FAILED 상태의 CloudFormation 스택 삭제.

## 시험 함정

- 무중단 + 빠른 롤백(prod) → **Immutable**(새 ASG, 용량 2배·가장 느림). 무중단 + 용량 유지·저비용 → **Rolling with additional batches** #exam/trap/beanstalk
- 다운타임 감수하고 가장 빠르게 → **All at once**(dev용). 그냥 Rolling은 배포 중 **용량 미달** #exam/trap/beanstalk
- 새 버전에 트래픽 일부만 보내는 카나리 + 자동 롤백 → **Traffic Splitting**. 환경 통째로 바꿔 검증 후 전환 → **Blue/Green(swap URL, EB 직접 기능 아님)** #exam/trap/beanstalk
- SQS 메시지를 당겨 처리·메시지 수로 스케일 → **Worker Environment Tier** #exam/trap/beanstalk
- 설정을 코드로 두고 리소스 추가 → 소스 루트 **`.ebextensions/`의 `.config`(YAML/JSON)**. 여기 만든 리소스는 환경 삭제 시 같이 삭제 #exam/trap/beanstalk
- prod에서 DB가 환경과 함께 지워지면 안 됨 → RDS를 **Beanstalk 밖에서 따로 생성**하고 연결 문자열만 전달 #exam/trap/beanstalk
- 환경 생성 후 **ELB 타입은 변경 불가** → 새 환경 만들어 CNAME swap/Route 53 #exam/trap/beanstalk
- application version은 **최대 1,000개**, 안 지우면 배포 막힘 → **Lifecycle Policy**(시간/개수 기준) #exam/trap/beanstalk
- Beanstalk은 내부적으로 **CloudFormation**에 의존 #exam/trap/beanstalk

## 관련 노트

[[CloudFormation]] · [[CICD]] · [[컨테이너 서비스]] · [[EC2]] · [[ELB & Auto Scaling]] · [[RDS & Aurora & ElastiCache]] · [[SQS & SNS & Kinesis]] · [[Route 53]]
