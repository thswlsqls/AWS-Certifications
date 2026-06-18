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

SAA 섹션이 키 유형·정책·서비스 선택을 다뤘다면, 여기서는 개발자가 코드와 API에서 직접 만나는 것 — 큰 데이터를 어떻게 암호화하는지, 어떤 KMS API를 부르는지, 호출이 막히면 어떻게 하는지, 배포 템플릿에서 비밀을 어떻게 참조하는지 — 를 정리한다. DVA 보안 도메인(26%)의 핵심.

### 봉투 암호화(Envelope Encryption) — 4KB 넘는 데이터

- **KMS의 `Encrypt` API는 한 번에 4KB까지만** 암호화한다.
- **4KB가 넘는 데이터는 봉투 암호화**를 써야 하고, 그 중심 API가 **`GenerateDataKey`** 다. (시험: "4KB 초과 데이터 암호화 → GenerateDataKey / Envelope Encryption")
- 흐름(암호화): `GenerateDataKey` 호출 → KMS가 **평문 데이터 키(DEK)** 와 **CMK로 암호화된 DEK**를 같이 돌려준다 → 평문 DEK로 큰 파일을 직접 암호화(클라이언트 측) → 평문 DEK는 버리고, **암호화된 DEK + 암호화된 파일**을 묶어 저장.
- 흐름(복호화): 암호화된 DEK를 `Decrypt`로 풀어 평문 DEK를 얻고 → 그걸로 파일을 복호화.

### KMS Symmetric API 요약 (어느 걸 부르나)

- **Encrypt**: 4KB까지 KMS로 직접 암호화.
- **GenerateDataKey**: DEK 생성 — **평문 DEK + 암호화된 DEK 둘 다** 반환(바로 쓸 때).
- **GenerateDataKeyWithoutPlaintext**: **암호화된 DEK만** 반환(지금 말고 나중에 쓸 것 — 쓸 때 `Decrypt` 필요).
- **Decrypt**: 4KB까지 복호화(DEK 포함).
- **GenerateRandom**: 임의 바이트 문자열 반환.

### Encryption SDK & Data Key Caching

- **AWS Encryption SDK**가 봉투 암호화를 대신 구현해 준다. CLI로도 있고 Java·Python·C·JavaScript 지원. 암호화된 DEK를 암호문(ciphertext)에 함께 담아 돌려준다.
- **Data Key Caching**: 매번 새 DEK를 만들지 않고 **재사용**해 KMS 호출 수를 줄인다(보안과의 trade-off). `LocalCryptoMaterialsCache`로 max age·max bytes·max messages를 정한다.

### KMS Request Quotas — ThrottlingException

- 요청 한도를 넘기면 **`ThrottlingException`**. 대응은 **지수 백오프(exponential backoff)**.
- **암호 연산(Decrypt·Encrypt·GenerateDataKey 등)은 quota를 공유**하고, **AWS가 나 대신 부르는 호출(예: SSE-KMS)도 포함**된다. 그래서 SSE-KMS를 많이 쓰면 throttle이 날 수 있다.
- 대응: `GenerateDataKey`라면 **Encryption SDK의 DEK 캐싱**, 그래도 부족하면 **quota 증액 요청**.

### S3 Bucket Key — SSE-KMS 호출 줄이기

S3에 SSE-KMS를 쓸 때 KMS 호출이 많아 비싸지는 걸 푼다. **S3 Bucket Key**를 한 번 만들어 그걸로 객체들의 DEK를 생성하면 **S3→KMS 호출과 비용이 약 99% 감소**한다. CloudTrail에 찍히는 KMS 이벤트도 줄어든다.

### CloudWatch Logs 암호화

- 로그를 KMS 키로 암호화할 수 있고, **로그 그룹 단위**로 CMK를 연결한다.
- **콘솔로는 CMK를 연결할 수 없다.** CloudWatch Logs **API를 써야** 한다: 이미 있는 로그 그룹은 `associate-kms-key`, 없으면 `create-log-group`으로 만들 때 지정.

### CodeBuild의 비밀 처리

- 비밀을 **환경 변수에 평문으로 넣지 말 것.** 대신 환경 변수가 **Parameter Store 파라미터**나 **Secrets Manager 비밀**을 참조하게 한다.
- VPC 안 리소스에 접근하려면 CodeBuild에 **VPC 설정**을 지정해야 한다.

### CloudFormation에서 비밀 참조 — Dynamic References

CloudFormation 템플릿에서 외부에 저장된 값을 가져온다. `'{{resolve:service-name:reference-key}}'` 형식.

- **`ssm`**: SSM Parameter Store의 평문 값.
- **`ssm-secure`**: SSM Parameter Store의 SecureString.
- **`secretsmanager`**: Secrets Manager 비밀.
- RDS와 묶을 때: ① 템플릿에서 `AWS::SecretsManager::Secret`으로 비밀 생성 → ② RDS 인스턴스가 `{{resolve:secretsmanager:...}}`로 참조 → ③ `AWS::SecretsManager::SecretTargetAttachment`로 비밀과 DB를 연결(회전용). 또는 RDS/Aurora의 **`ManageMasterUserPassword: true`** 로 비밀 생성·회전을 통째로 맡길 수도 있다.

### AWS Nitro Enclaves

민감 데이터(PII·의료·금융)를 **격리된 컴퓨팅 환경**에서 처리. 컨테이너가 아니라 완전 격리된 VM이고, 영구 저장소·대화형 접근·외부 네트워킹이 없다. **Cryptographic Attestation**으로 승인된 코드만 실행하고, **KMS와 통합**돼 Enclave만 민감 데이터에 접근하게 한다. 용도: 개인 키 보호, 신용카드 처리, 안전한 다자간 연산.

## 운영 관점 (CloudOps) #cloudops

![[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=477]]

%% 이 시험의 관점에서 배운 내용을 정리합니다. %%

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
- KMS Encrypt는 **4KB까지**. 4KB 넘으면 **봉투 암호화 = GenerateDataKey API**. #exam/trap/kms
- GenerateDataKey는 평문+암호화 DEK 둘 다, **GenerateDataKeyWithoutPlaintext는 암호화 DEK만**(나중에 쓸 것). #exam/trap/kms
- KMS 호출 초과 → **ThrottlingException** → 지수 백오프. 암호 연산은 quota 공유, SSE-KMS 호출도 포함. GenerateDataKey면 **DEK 캐싱(Encryption SDK)**. #exam/trap/kms
- S3 SSE-KMS 호출·비용 99% 절감 → **S3 Bucket Key**. #exam/trap/kms
- CloudWatch Logs에 CMK 연결은 **콘솔 불가**, API(`associate-kms-key`/`create-log-group`)로만. #exam/trap/kms
- CodeBuild 비밀은 환경 변수에 평문 금지 → Parameter Store/Secrets Manager 참조. #exam/trap/security
- CloudFormation에서 비밀 참조 → Dynamic References: `ssm`(평문)·`ssm-secure`(SecureString)·`secretsmanager`(비밀). #exam/trap/security
- 민감 데이터를 격리 환경에서 처리(승인 코드만, KMS 통합) → **Nitro Enclaves**. #exam/trap/security

## 관련 노트

[[IAM]] · [[S3]] · [[RDS & Aurora & ElastiCache]] · [[CloudWatch & CloudTrail & Config]] · [[CloudFront & Global Accelerator]] · [[ELB & Auto Scaling]] · [[Route 53]] · [[CloudFormation]] · [[CICD]] · [[Lambda]]
