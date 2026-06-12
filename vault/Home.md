---
tags:
  - home
---

# AWS 자격증 Vault

세 개의 AWS Associate 자격증을 **하나의 vault**에서 준비합니다.

| 시험          | 관점  | 태그          | 강의자료                                                                               |
| ----------- | --- | ----------- | ---------------------------------------------------------------------------------- |
| [[SAA-C03]] | 설계  | `#saa`      | [[AWS Certified Solutions Architect Slides v47.pdf\|SAA 슬라이드 (879p)]]              |
| [[DVA-C02]] | 개발  | `#dva`      | [[AWS Certified Developer Slides v44.pdf\|DVA 슬라이드 (913p)]]                        |
| [[SOA-C03]] | 운영  | `#cloudops` | [[AWS Certified CloudOps Engineer Associate Slides v41.pdf\|CloudOps 슬라이드 (734p)]] |

진도 현황: [[진도표.base|📊 진도표]]

## Vault의 원칙

1. **폴더가 아니라 링크로 정리합니다.** 시험별 폴더는 만들지 않습니다. S3는 세 시험 모두에 나오므로, [[S3]] 노트 하나만 두고 시험과의 관계는 태그·속성·링크로 표현합니다.
2. **하나의 지식은 한 곳에만.** 같은 내용을 시험별로 복사하면 일관성이 깨집니다. 서비스당 노트 1개, 그 안에 시험별 **관점 섹션**(설계 `#saa` / 개발 `#dva` / 운영 `#cloudops`)을 적층합니다.
3. **출제 비중에 비례해 공부합니다.** 각 시험 노트의 도메인 비중표와 [[진도표.base|진도표]]로 빈칸을 찾습니다.
4. **강의자료는 노트 안에 임베드합니다.** `![[슬라이드.pdf#page=N]]` — 정리와 출처가 같은 화면에 보입니다.
5. **함정은 그때그때 태그로.** 틀린 문제는 해당 서비스 노트의 "시험 함정" 섹션에 `#exam/trap/<주제>` 태그와 함께 기록합니다.

## 사용 규칙

- **새 서비스 노트**: [[Templates/서비스 노트 템플릿|템플릿]]을 적용해 vault 루트에 만듭니다.
- **태그 규칙**: 공백 불가, kebab-case 사용 (예: `#exam/trap/cost-optimization`). 중첩 태그는 상위 태그 검색 시 함께 수집됩니다 (`tag:#exam/trap` → 하위 전부).
- **진도 속성**: 노트 상단 속성에서 `status`(todo → learning → reviewing → done)와 `confidence`(1~5)를 갱신합니다.
- **저작권 주의**: 슬라이드 PDF가 포함된 이 vault를 공개 저장소에 올리지 마세요.

## 시험 직전 한 주 루틴

1. **함정 점검** — `tag:#exam/trap` 검색으로 모든 함정을 모아 보고, 반복해서 틀린 항목에 집중
2. **그래프로 빈 곳 메우기** — 그래프 뷰에서 외딴(orphaned) 노트를 찾아 기존 개념과 연결. 새 내용 추가가 아니라 연결 강화가 목표
3. **진도표 점검** — [[진도표.base|진도표]]에서 비중 큰 도메인의 `todo`/`learning` 노트부터 마무리

## 서비스 노트 지도

**공통 기반**: [[IAM]] · [[EC2]] · [[EBS & EFS]] · [[ELB & Auto Scaling]] · [[VPC]] · [[Route 53]] · [[S3]] · [[CloudFront & Global Accelerator]] · [[RDS & Aurora & ElastiCache]] · [[DynamoDB]] · [[Lambda]] · [[CloudWatch & CloudTrail & Config]] · [[KMS & 암호화]]

**SAA 중심**: [[솔루션 아키텍처 패턴]] · [[스토리지 추가 서비스]] · [[데이터 분석 서비스]] · [[머신러닝 서비스]] · [[재해 복구 & 마이그레이션]] · [[고급 ID 관리]]

**DVA 중심**: [[API Gateway]] · [[Cognito]] · [[SQS & SNS & Kinesis]] · [[컨테이너 서비스]] · [[Elastic Beanstalk]] · [[CloudFormation]] · [[CICD]] · [[CDK]] · [[Step Functions & AppSync]]

**CloudOps 중심**: [[Systems Manager]] · [[Organizations & 계정 관리]] · [[기타 서비스]]
