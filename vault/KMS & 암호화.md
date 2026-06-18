---
service: AWS KMS · SSM Parameter Store · Secrets Manager
exams: [saa, dva, cloudops]
domains:
  - saa/secure-architectures
  - dva/security
  - cloudops/security-compliance
status: learning
confidence: 1
tags:
  - service
  - saa/security
  - dva/security
  - cloudops/security
---

# KMS & 암호화

> 암호화 키 관리 — KMS 키 유형, 봉투 암호화, Parameter Store vs Secrets Manager.

## 개요

이 노트는 SAA 슬라이드의 "AWS Security & Encryption" 단원 전체를 담는다. 암호화의 세 형태부터 시작한다.

- **전송 중 암호화(in flight, TLS/SSL)** — 보내기 전에 암호화하고 받은 뒤 복호화. 중간자 공격(MITM)을 막는다. HTTPS가 이것.
- **서버 측 저장 암호화(at rest)** — 서버가 받은 뒤 암호화해 저장하고, 보낼 때 복호화. 키를 서버가 관리하며 접근할 수 있어야 한다. S3 SSE 등.
- **클라이언트 측 암호화** — 클라이언트가 암호화하고 서버는 절대 복호화하지 못한다. 받는 쪽 클라이언트만 푼다. 봉투 암호화(Envelope Encryption)를 쓴다.

핵심 갈림길은 **"키를 누가 관리하느냐"**다. AWS가 전부 관리(무료) → 내가 키 정책만 관리(KMS Customer Managed Key) → 전용 하드웨어로 내가 완전히 관리(CloudHSM) 순으로 통제권이 커진다. 시험은 규정 요구사항을 주고 이 중 하나를 고르게 한다.

## 설계 관점 (SAA) #saa

![[AWS Certified Solutions Architect Slides v47.pdf#page=651]]

### AWS KMS

"AWS 서비스에서 암호화"라고 하면 대부분 KMS다. IAM과 통합되어 접근을 제어하고, 키 사용 내역을 [[CloudWatch & CloudTrail & Config|CloudTrail]]로 감사할 수 있다. **비밀을 평문으로(특히 코드에) 저장하지 말 것** — KMS API로 암호화해 환경 변수·코드에 넣는다.

KMS 키 유형(키를 누가 소유·관리하느냐):
- **AWS Owned Keys** — 무료. SSE-S3·SSE-SQS·SSE-DDB 같은 기본 키.
- **AWS Managed Key** — 무료. `aws/서비스명` 형태(예: `aws/rds`, `aws/ebs`).
- **Customer Managed Key** — KMS에서 생성하거나 직접 import. 월 $1 + API 호출 비용. 키 정책을 직접 통제할 수 있어 시험에 자주 나온다.

대칭/비대칭 구분:
- **Symmetric (AES-256)** — 암복호화에 같은 키. AWS 서비스 통합은 전부 대칭. **키 평문에 절대 접근 못 하고 반드시 KMS API를 호출해야 쓴다.**
- **Asymmetric (RSA·ECC)** — 공개키(암호화)/개인키(복호화) 쌍. 공개키는 다운로드 가능, 개인키는 평문 접근 불가. **KMS API를 못 부르는 AWS 외부 사용자가 암호화해야 할 때** 쓴다.

키 회전(rotation):
- AWS Managed Key — **1년마다 자동**.
- Customer Managed Key — 자동(켜야 함) + 온디맨드.
- Imported Key — **수동만 가능**(alias를 새 키로 옮기는 방식).

**KMS Key Policy** — KMS 키 접근 제어. S3 버킷 정책과 비슷하지만 차이가 있다: **정책 없이는 키에 접근할 수 없다.** 정책을 안 주면 기본 정책이 만들어져 루트 사용자(=계정 전체)에 완전 접근을 준다. 커스텀 정책으로 어떤 사용자·역할이 쓰고 관리할지 정하며, **교차 계정 접근**에도 쓴다.

KMS는 **리전에 묶인다.** EBS 스냅샷을 다른 리전으로 복사하면 그 리전의 키로 다시 암호화(re-encrypt)해야 한다. 교차 계정 스냅샷 공유는 ① Customer Managed Key로 암호화 → ② 키 정책으로 대상 계정 허용 → ③ 스냅샷 공유 → ④ 대상에서 자기 계정 키로 복사한다.

**KMS Multi-Region Keys** — 여러 리전에 같은 키 ID·키 자료를 갖는 동일한 키(Primary + Replica). 한 리전에서 암호화하고 다른 리전에서 복호화할 수 있어 **re-encrypt나 교차 리전 API 호출이 필요 없다.** 단 글로벌이 아니고 각 키는 독립적으로 관리된다. 용도: 글로벌 클라이언트 측 암호화, Global DynamoDB·Global Aurora의 클라이언트 측 필드 암호화. (DynamoDB는 DynamoDB Encryption Client, Aurora는 AWS Encryption SDK로 특정 속성만 암호화하면 DB 관리자조차 그 필드를 못 본다.)

**S3 복제 암호화 주의**: 암호화 안 된 객체와 SSE-S3 객체는 기본 복제, SSE-C도 복제 가능. **SSE-KMS 객체는 옵션을 따로 켜야** 하고 대상 키 지정·키 정책 조정·`kms:Decrypt`(원본)+`kms:Encrypt`(대상) 역할이 필요하며 KMS throttling이 날 수 있다.

### SSM Parameter Store

설정값과 비밀을 저장하는 곳. KMS로 선택적 암호화(SecureString), 서버리스·확장성·버전 추적·IAM 접근 제어·EventBridge 알림·CloudFormation 통합. `/부서/앱/환경/키` 식 계층 구조로 관리하고 `GetParameters`/`GetParametersByPath`로 읽는다. `/aws/reference/secretsmanager/...` 경로로 Secrets Manager 비밀도 참조할 수 있다.

티어: **Standard**(파라미터 1만 개, 값 4KB, 무료, 파라미터 정책 없음) vs **Advanced**(10만 개, 8KB, 유료, **파라미터 정책 사용 가능**). 파라미터 정책은 비밀번호 같은 값에 TTL(만료일)을 걸어 갱신·삭제를 강제하거나(Expiration), 만료 전/변경 없음을 EventBridge로 알린다.

### Secrets Manager vs Parameter Store

**Secrets Manager**는 비밀 저장 전용의 새 서비스. 핵심 차이는 **X일마다 비밀을 강제 회전(rotation)**하고, 회전 시 Lambda로 새 비밀을 자동 생성한다는 점. **RDS(MySQL·PostgreSQL·Aurora)와 통합**되어 DB 자격증명 회전에 주로 쓴다. 비밀은 KMS로 암호화. Multi-Region Secrets로 리전 간 복제(읽기 복제본 → 독립 승격)도 된다.

판단 기준: **자동 회전·RDS 통합이 필요하면 Secrets Manager**, 단순 설정·저렴한 비밀 저장이면 Parameter Store.

### ACM (AWS Certificate Manager)

TLS 인증서를 발급·관리·배포. **퍼블릭 인증서는 무료**, 자동 갱신(만료 60일 전). ELB(CLB·ALB·NLB)·CloudFront·API Gateway에 붙인다. **EC2에는 못 쓴다**(인증서를 추출할 수 없어서). 퍼블릭 인증서 요청 시 검증은 **DNS 검증(자동화에 유리, Route 53 CNAME)** 또는 이메일 검증. import한 인증서는 **자동 갱신이 안 되고** ACM이 만료 45일 전부터 EventBridge로 매일 알림을 보낸다(Config의 `acm-certificate-expiration-check` 규칙도 있음).

### CloudHSM

KMS는 AWS가 암호화 **소프트웨어**를 관리하지만, CloudHSM은 AWS가 암호화 **하드웨어**(전용 Hardware Security Module)를 제공하고 **키는 내가 완전히 관리한다.** 변조 방지 장치, FIPS 140-2 Level 3, 대칭·비대칭 모두 지원. 무료 티어 없음. 클러스터는 Multi-AZ로 펼쳐 고가용성. **SSE-C와 함께 쓰기 좋다**(직접 키를 들고 있어야 하므로).

KMS vs CloudHSM 요점:
- 테넌시: KMS는 멀티 테넌트, CloudHSM은 **싱글 테넌트(전용)**.
- 키 관리: KMS는 AWS Owned/Managed/Customer, CloudHSM은 **Customer Managed만**.
- 접근: KMS는 IAM, CloudHSM은 **사용자를 직접 만들고 권한 관리**.
- "규정상 키를 전용 하드웨어로 직접 관리해야 한다" → CloudHSM. KMS Custom Key Store로 CloudHSM을 KMS 뒤에 둘 수도 있다.

### 네트워크·위협 보호 서비스 (역할 매칭이 출제)

- **WAF (Web Application Firewall)** — **Layer 7(HTTP)** 공격 방어(SQL injection·XSS). ALB·API Gateway·CloudFront·AppSync·Cognito User Pool에 붙음. Web ACL 규칙: IP Set(최대 1만 개), geo-match(국가 차단), **Rate-based rule(DDoS 방어)**. Web ACL은 리전 단위(CloudFront는 예외=글로벌). **NLB(Layer 4)는 미지원** → 고정 IP가 필요하면 Global Accelerator + ALB에 WAF.
- **Shield** — DDoS 방어. **Standard는 모든 고객에 무료**(Layer 3/4: SYN/UDP flood 등). **Advanced는 월 $3,000**, EC2·ELB·CloudFront·Global Accelerator·Route 53 보호, 24/7 DDoS 대응팀(DRP), 트래픽 폭증 요금 보호, Layer 7 WAF 규칙 자동 생성.
- **Firewall Manager** — **AWS Organization 전체 계정**에 보안 규칙(WAF·Shield Advanced·SG·Network Firewall·Route 53 DNS Firewall)을 일괄 적용. **새로 만들어지는 리소스에도 자동 적용**(규정 준수에 유리). 리전 단위.
- 셋의 관계: 세밀한 단일 리소스 보호는 **WAF 단독**, 여러 계정에 걸쳐 WAF를 적용·자동화하려면 **Firewall Manager + WAF**, 잦은 DDoS·전담 지원이 필요하면 **Shield Advanced**.
- **GuardDuty** — ML 기반 위협 탐지. **클릭 한 번으로 활성화, 소프트웨어 설치 불필요.** 입력은 **CloudTrail 이벤트(관리/S3 데이터)·VPC Flow Logs·DNS Logs**(+선택: EKS·RDS·EBS·Lambda·S3). 탐지 결과를 EventBridge로 보내 Lambda·SNS로 알림. **암호화폐 채굴 공격 전용 finding**이 있다.
- **Inspector** — 자동 보안 평가. **대상은 EC2·ECR 컨테이너 이미지·Lambda 함수뿐.** EC2는 SSM 에이전트로 OS 취약점·의도치 않은 네트워크 노출을 점검, 패키지 취약점은 CVE DB 기준. 위험 점수로 우선순위. Security Hub·EventBridge로 전송.
- **Macie** — ML·패턴 매칭으로 **S3의 민감 데이터(PII)**를 찾아 알림. EventBridge 연동.

## 개발 관점 (DVA) #dva

![[AWS Certified Developer Slides v44.pdf#page=832]]

%% 이 시험의 관점에서 배운 내용을 정리합니다. %%

## 운영 관점 (CloudOps) #cloudops

![[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=477]]

설계 관점에서 각 보안 서비스의 역할(WAF·Shield·GuardDuty·Inspector·Macie·KMS 키 유형 등)은 이미 정리했다. 운영 시험은 거기에 **여러 계정의 보안·준수를 한곳에 모으는 도구**와 **키를 실제로 회전·삭제·감시하는 운영 디테일**을 더한다. 보안 16% 도메인의 몸통이다.

### 중앙 보안·감사 도구 (멀티 계정)

- **Security Hub** — 여러 계정의 보안을 한 대시보드에서 보고 자동 점검하는 **중앙 도구**. Config·GuardDuty·Inspector·Macie·IAM Access Analyzer·Systems Manager·Firewall Manager·Health의 finding을 모은다. **먼저 AWS Config를 켜야** 동작한다. Organizations에 **위임 관리자(Delegated Administrator)** 계정을 지정하면 전 계정에 자동 활성화되고, CIS AWS Benchmark 같은 표준으로 전 계정을 한 번에 점검한다. 결과는 EventBridge로 보내거나 **Amazon Detective**로 넘겨 원인을 파고든다.
- **Audit Manager** — AWS 사용 내역을 계속 감사해 **규정 준수 보고서**를 자동으로 만든다. 사전 정의 프레임워크(CIS·GDPR·HIPAA·PCI DSS·SOC 2)를 골라 범위를 정하면 증거를 자동 수집해 감사용 리포트를 낸다. "감사·인증 대비 증거 수집 자동화" → Audit Manager.
- **Trusted Advisor** — 설치 없이 계정을 **6개 범주(비용·성능·보안·내결함성·서비스 한도·운영 우수성)**로 점검·권고한다. **전체 점검과 Support API는 Business·Enterprise 지원 플랜**에서만. ServiceLimitUsage 지표 → CloudWatch Alarm → SNS로 한도 임박 알림을 자동화하고, **Organizational View**(관리 계정에서 Trusted Access 활성화)로 전 계정 점검을 모아 본다.
- 세 도구 정리: **Trusted Advisor**는 계정 단위 모범 사례 점검, **Security Hub**는 보안 finding 통합·자동 점검, **Audit Manager**는 규정 준수 증거·리포트.

### 보안·규정 준수 로깅

규정 대응에는 서비스별 보안·감사 로그를 모아 둔다: CloudTrail(API 호출)·Config(설정 준수)·CloudWatch Logs(보관)·VPC Flow Logs(VPC IP 트래픽)·ELB Access Logs·CloudFront Logs·WAF Logs. **S3에 모으면 Athena로 분석**하고, 비용 절감은 Glacier로 옮긴다. 로그 버킷은 암호화·IAM·버킷 정책·MFA로 보호한다.

### KMS 운영 — 회전·삭제·감시

- **자동 회전 주기 커스터마이즈**: AWS Managed Key는 1년 고정 자동. **Customer-Managed 대칭 키**는 자동 회전을 켜고 주기를 **90~2560일(기본 365일)**로 조절한다. 회전해도 **Key ID는 그대로**(backing key만 바뀜)고 이전 키는 살아 있어 옛 데이터를 복호화한다.
- **On-Demand 회전**: Customer-Managed 대칭 키를 즉시 회전. 자동 회전이 꺼져 있어도 되고 기존 자동 일정도 바꾸지 않는다(호출 횟수 제한 있음).
- **수동 회전**: 자동 회전 대상이 아닌 키(비대칭 키, import 키)는 새 키를 만들어 **alias를 새 키로 옮긴다**(`UpdateAlias`). 새 키는 Key ID가 다르다.
- **키 삭제 주의**: 바로 못 지우고 **7~30일 대기(Pending deletion)** 후 삭제된다. 대기 중에는 **암호화 작업에 못 쓰고**(SSE-KMS로 암호화한 S3 객체 복호화 불가) 회전도 멈춘다. 대기 중 취소 가능. 확신이 없으면 **삭제 대신 비활성화(disable)**를 권한다.
- **삭제 예정 키 사용 감시**: CMK가 "Pending deletion"인데 암호화 작업을 시도하면 거부되는데, 이걸 CloudTrail → CloudWatch Logs **Metric Filter("is pending deletion")** → CloudWatch Alarm → SNS로 알린다. 키 삭제로 서비스가 깨지기 전에 잡는 운영 패턴.
- **암호화된 EBS 볼륨의 키 변경**: 볼륨의 암호화 키는 직접 못 바꾼다. **스냅샷을 떠서 새 키로 새 볼륨을 만든다.**
- **암호화된 RDS DB 스냅샷 공유**: 다른 계정과 공유하려면 **먼저 그 CMK를 Key Policy로 대상 계정에 공유**해야 한다(EBS 교차 계정 스냅샷과 같은 원리).

### Secrets Manager·ACM 운영 (회전·만료 감시)

- **Secrets Manager 회전 모니터링**: CloudTrail이 회전 관련 이벤트를 **non-API 서비스 이벤트**로 기록한다 — `RotationStarted`·`RotationSucceeded`·`RotationFailed`·`RotationAbandoned`(자동 대신 수동 변경) 등. CloudWatch Logs·Alarm과 엮어 회전 실패를 알린다. 회전이 깨지면 회전용 **Lambda의 CloudWatch Logs**를 보고 디버깅한다.
- **Parameter Store 회전**: 자체 회전 기능이 없어 **EventBridge 스케줄 → Lambda**로 값을 바꾼다. (Secrets Manager는 서비스가 직접 Lambda를 호출해 회전.)
- **ACM 인증서 만료 감시**: **import한 인증서는 자동 갱신이 안 된다.** ACM이 **만료 45일 전부터 매일 만료 이벤트를 EventBridge**로 보내고, AWS Config의 관리형 규칙 **`acm-certificate-expiration-check`**로도 점검한다. API Gateway에 붙일 때, **Edge-Optimized는 인증서가 us-east-1(CloudFront와 같은 리전)**에, **Regional은 API와 같은 리전에 import**돼 있어야 한다.

## 시험 함정

- KMS 키는 리전에 묶인다. EBS 스냅샷을 다른 리전에 복사하면 그 리전 키로 re-encrypt 필요. 글로벌 동일 키가 필요하면 Multi-Region Keys. #exam/trap/kms
- 대칭 KMS 키는 평문 접근 불가, 반드시 KMS API 호출로 사용. AWS 외부에서 암호화해야 하면 비대칭(공개키 다운로드). #exam/trap/kms
- AWS Managed Key 회전은 1년 자동, Imported Key는 수동 회전만 가능. #exam/trap/kms
- ACM 인증서는 EC2에 못 붙인다. ELB·CloudFront·API Gateway만. import 인증서는 자동 갱신 안 됨. #exam/trap/security
- 자동 회전·RDS 통합이면 Secrets Manager, 단순·저렴한 비밀이면 Parameter Store. #exam/trap/security
- CloudHSM은 싱글 테넌트 전용 하드웨어로 키를 직접 관리. "전용 하드웨어로 키 관리" 요구면 KMS 아닌 CloudHSM. #exam/trap/kms
- WAF는 Layer 7(ALB·API Gateway·CloudFront), NLB(Layer 4)는 미지원. 고정 IP는 Global Accelerator + ALB. #exam/trap/security
- Shield Standard는 무료·자동(L3/4), Advanced는 월 $3,000(전담팀·요금 보호·L7). #exam/trap/security
- GuardDuty 입력은 CloudTrail·VPC Flow Logs·DNS Logs. Inspector는 EC2·ECR·Lambda 취약점. Macie는 S3의 PII. 역할 혼동 주의. #exam/trap/security
- Security Hub는 **먼저 AWS Config를 켜야** 동작. 여러 계정 finding 통합 → Organizations 위임 관리자. #exam/trap/security
- 중앙 도구 구분: Trusted Advisor(계정 모범 사례 6범주)·Security Hub(보안 finding 통합)·Audit Manager(준수 증거·리포트). #exam/trap/security
- KMS 커스텀 대칭 키 자동 회전 주기는 **90~2560일(기본 365)**, 회전해도 Key ID 동일(backing key만 교체). #exam/trap/kms
- KMS 키 삭제는 **7~30일 대기(Pending deletion)** — 그동안 암호화 작업 불가. 확신 없으면 삭제 대신 disable. #exam/trap/kms
- "Pending deletion 키 사용 시도 알림" → CloudTrail → CW Logs Metric Filter → Alarm → SNS. #exam/trap/kms
- 암호화 EBS 볼륨의 KMS 키는 직접 변경 불가 — 스냅샷 떠서 새 키로 새 볼륨 생성. #exam/trap/kms
- 암호화된 RDS 스냅샷 교차 계정 공유는 **CMK를 Key Policy로 먼저 공유**해야 함. #exam/trap/kms
- ACM API Gateway: **Edge-Optimized 인증서는 us-east-1**, **Regional은 API와 같은 리전**에 있어야 함. #exam/trap/security
- Secrets Manager 회전은 CloudTrail **non-API 이벤트**(RotationStarted/Succeeded/Failed/Abandoned)로 감시. Parameter Store 회전은 EventBridge+Lambda. #exam/trap/security

## 관련 노트

[[IAM]] · [[S3]] · [[RDS & Aurora & ElastiCache]] · [[CloudWatch & CloudTrail & Config]] · [[CloudFront & Global Accelerator]] · [[ELB & Auto Scaling]] · [[Route 53]] · [[Systems Manager]] · [[Organizations & 계정 관리]]
