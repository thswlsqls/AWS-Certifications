---
exam: SOA-C03
exam-name: AWS Certified CloudOps Engineer – Associate
slides: AWS Certified CloudOps Engineer Associate Slides v41.pdf
tags:
  - exam
---

# SOA-C03 — CloudOps Engineer Associate

> 운영 관점의 시험. 모니터링, 비용 최적화, 문제 해결에 초점을 둡니다.
> 강의자료: [[AWS Certified CloudOps Engineer Associate Slides v41.pdf|CloudOps 슬라이드 v41 (734p)]]

> [!warning] SysOps → CloudOps 개편
> 기존 SysOps Administrator(SOA-C02)에서 명칭과 도메인 구조가 바뀌었습니다. 컨테이너, IaC, 멀티 계정·멀티 리전 아키텍처가 추가되었으므로 **이전 버전 자료를 사용하지 마세요.**

## 도메인 비중

> 공부한 시간이 아니라 **출제 비중에 비례해** 점수를 얻습니다. 비중 큰 도메인의 빈칸부터 채우세요.

| 도메인 | 비중 | `domains` 속성 값 |
| --- | --- | --- |
| 모니터링, 로깅, 분석, 문제 해결 | **22%** | `cloudops/monitoring-logging` |
| 신뢰성 및 비즈니스 연속성 | **22%** | `cloudops/reliability` |
| 배포, 프로비저닝, 자동화 | **22%** | `cloudops/deployment-automation` |
| 네트워킹 및 콘텐츠 전송 | **18%** | `cloudops/networking` |
| 보안 및 규정 준수 | **16%** | `cloudops/security-compliance` |

진도 현황은 [[진도표.base|진도표]]의 **CloudOps** 뷰에서 확인합니다.

## 슬라이드 섹션 목차

| 페이지 | 섹션 | 서비스 노트 |
| --- | --- | --- |
| [[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=9\|p.9]] | Amazon EC2 for CloudOps | [[EC2]] |
| [[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=28\|p.28]] | AMI | [[EC2]] |
| [[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=40\|p.40]] | Systems Manager | [[Systems Manager]] |
| [[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=71\|p.71]] | High Availability & Scalability | [[ELB & Auto Scaling]] |
| [[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=138\|p.138]] | AWS CloudFormation | [[CloudFormation]] |
| [[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=214\|p.214]] | AWS Lambda | [[Lambda]] |
| [[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=225\|p.225]] | EC2 Storage & Data Management | [[EBS & EFS]] |
| [[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=244\|p.244]] | Amazon S3 | [[S3]] |
| [[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=268\|p.268]] | Amazon S3 – Advanced | [[S3]] |
| [[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=286\|p.286]] | Amazon S3 – Security | [[S3]] |
| [[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=295\|p.295]] | Advanced Storage Solutions | [[스토리지 추가 서비스]] |
| [[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=313\|p.313]] | Amazon CloudFront | [[CloudFront & Global Accelerator]] |
| [[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=334\|p.334]] | Databases in AWS | [[RDS & Aurora & ElastiCache]] · [[DynamoDB]] |
| [[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=374\|p.374]] | Monitoring, Audit & Performance | [[CloudWatch & CloudTrail & Config]] |
| [[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=434\|p.434]] | AWS Account Management | [[Organizations & 계정 관리]] |
| [[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=469\|p.469]] | Disaster Recovery | [[재해 복구 & 마이그레이션]] |
| [[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=477\|p.477]] | Security & Compliance | [[KMS & 암호화]] |
| [[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=526\|p.526]] | Identity | [[IAM]] |
| [[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=539\|p.539]] | Amazon Route 53 | [[Route 53]] |
| [[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=585\|p.585]] | Amazon VPC | [[VPC]] |
| [[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=672\|p.672]] | Other Services | [[기타 서비스]] |
| [[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=721\|p.721]] | Exam Review & Tips | — |

## 시험 직전 체크

1. `tag:#exam/trap` 검색 → `#cloudops` 가 붙은 함정부터 점검
2. 그래프 뷰에서 연결이 빈약한(외딴) 노트 확인
3. [[진도표.base|진도표]]에서 비중 22% 도메인 3개의 빈칸부터 채우기
