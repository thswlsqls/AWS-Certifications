---
service: AWS CloudFormation
exams: [dva, cloudops]
domains:
  - dva/deployment
  - cloudops/deployment-automation
status: reviewing
confidence: 1
tags:
  - service
  - dva/deployment
  - cloudops/deployment
---

# CloudFormation

> IaC의 표준 — 템플릿 구조(Parameters/Resources/Outputs), 스택, 변경 세트, 드리프트.

## 개요

인프라를 **코드로 선언**하는 표준 도구(Infrastructure as Code). "보안 그룹 하나, 그걸 쓰는 EC2 두 대, Elastic IP 두 개, S3 버킷, 그 앞의 ELB가 필요하다"를 템플릿에 적으면, CloudFormation이 **올바른 순서로** 내가 적은 **그대로의 설정**으로 만들어 준다.

- **수동 생성이 없으니** 통제가 쉽고, 템플릿을 **Git으로 버전 관리**하며, 변경을 코드 리뷰로 본다.
- 스택의 리소스마다 식별자 태그가 붙어 **스택 비용을 한눈에** 본다. dev 환경을 밤에 지웠다 아침에 다시 만드는 식의 절약도 가능.
- **선언형**이라 생성 순서를 내가 짤 필요가 없고, 인프라를 통째로 부쉈다 다시 만들 수 있다. [[Elastic Beanstalk]]·SAM·[[CDK]]도 내부적으로 CloudFormation을 쓴다.

## 개발 관점 (DVA) #dva

![[AWS Certified Developer Slides v44.pdf#page=379]]

### 동작 방식

- 템플릿은 **S3에 올라가고** CloudFormation이 그걸 참조한다. 기존 템플릿은 **수정 못 하고**, 바꾸려면 **새 버전을 다시 업로드**한다.
- **Stack**은 이름으로 식별된다. **스택을 지우면 그 스택이 만든 리소스가 전부 삭제**된다.
- 배포: 수동(Infrastructure Composer·콘솔에서 파라미터 입력) vs 자동(YAML 파일 + AWS CLI `create-stack`이나 CD 도구). 자동화하려면 후자.

### 템플릿 구성 요소

- **Resources**(필수): 만들 AWS 리소스. 템플릿의 핵심.
- **Parameters**: 템플릿에 넣는 동적 입력값.
- **Mappings**: 템플릿 안에 박아 둔 정적 변수.
- **Outputs**: 만든 것에 대한 참조(다른 스택에서 가져다 쓸 수 있음).
- **Conditions**: 조건에 따라 리소스를 만들지 말지.
- 그 외 `AWSTemplateFormatVersion`, `Description`. 헬퍼로 **References**와 **Functions**.

### Resources

리소스는 **필수**이고 서로 참조할 수 있다. 700종이 넘고 식별자는 **`service-provider::service-name::data-type-name`**(예: `AWS::EC2::Instance`). 다 외울 수 없으니 **문서를 찾아 쓰는** 게 정석. 리소스 개수를 동적으로 만들려면 **Macros·Transform**, CloudFormation이 아직 지원 안 하는 리소스는 **Custom Resources**로 푼다.

### Parameters

회사 전체에서 템플릿을 **재사용**하거나 미리 정할 수 없는 입력에 쓴다. 타입으로 잘못된 값을 걸러 실수를 막는다.

- 타입: `String`·`Number`·`CommaDelimitedList`·`List<Number>`·**AWS-Specific Parameter**(계정의 기존 값과 대조해 검증)·**SSM Parameter**(Parameter Store에서 값을 가져옴).
- 설정: `AllowedValues`·`AllowedPattern`(regex)·`Min/MaxLength`·`Min/MaxValue`·`Default`·`ConstraintDescription`·**`NoEcho`**(비밀번호처럼 값을 안 보이게).
- 판단: **설정이 앞으로 바뀔 것 같으면 Parameter로** 빼라 — 템플릿을 다시 안 올려도 값만 바꾸면 된다.
- 참조: **`Fn::Ref`**(YAML 단축 **`!Ref`**). 다른 리소스도 참조 가능.
- **Pseudo Parameters**(기본 제공): `AWS::AccountId`·`AWS::Region`·`AWS::StackId`·`AWS::StackName`·`AWS::NotificationARNs`·`AWS::NoValue`.

### Mappings

값을 미리 다 아는 경우의 정적 변수. 리전·AZ·계정·환경(dev/prod)·AMI 타입처럼 변수로 추려지는 것에 쓴다. **AMI는 리전마다 달라서** Mappings가 잘 맞는다. 접근은 **`Fn::FindInMap [MapName, TopLevelKey, SecondLevelKey]`**.

- **Mappings vs Parameters**: 값이 미리 정해져 있고 리전·환경 등으로 추려지면 **Mappings**(더 안전), 정말 사용자마다 다른 값이면 **Parameters**.

### Outputs와 Cross-Stack 참조

만든 값을 내보내(export) 다른 스택에서 가져다 쓴다. 예: 네트워크 스택이 VPC ID·Subnet ID를 export하면 앱 스택이 가져온다.

- `Outputs`에 `Export: Name:`으로 내보내고, 받는 쪽은 **`Fn::ImportValue`**로 가져온다.
- **export한 값을 누가 import하고 있으면 그 스택은 삭제할 수 없다.**

### Conditions

조건(환경·리전·파라미터 값)에 따라 리소스/Output을 만들지 결정. `Conditions:`에 `!Equals` 등으로 정의하고(`Fn::And`·`Fn::Equals`·`Fn::If`·`Fn::Not`·`Fn::Or`), 리소스에 `Condition: 이름`을 달아 적용한다.

### Intrinsic Functions (시험 — "must know")

- **`Ref`**: Parameter 값 또는 리소스의 **물리 ID**(예: EC2 ID).
- **`Fn::GetAtt`**: 리소스의 **속성**(예: EC2의 `PublicDnsName`). 어떤 속성이 있는지는 문서를 본다.
- **`Fn::FindInMap`**: Mappings 값 조회.
- **`Fn::ImportValue`**: 다른 스택이 export한 값 가져오기.
- **`Fn::Base64`**: 문자열을 Base64로. 예: EC2 **UserData**에 스크립트 전달.
- **Condition Functions**: `Fn::If`·`Fn::Not`·`Fn::Equals` 등.
- (그 외 `Fn::Join`·`Fn::Sub`·`Fn::Select`·`Fn::Split`·`Fn::Cidr`·`Fn::GetAZs` 등)

### Rollback

- **스택 생성 실패**: 기본은 **전부 롤백(삭제)**. 원인을 보려면 rollback을 끄고 디버깅하는 옵션.
- **스택 업데이트 실패**: **직전의 정상 상태로 자동 롤백**. 로그로 원인 확인.
- **롤백마저 실패**하면 리소스를 수동으로 고친 뒤 **`ContinueUpdateRollback`**(콘솔/CLI `continue-update-rollback`).

### DeletionPolicy — 삭제·제거 시 동작

리소스를 템플릿에서 지우거나 스택을 삭제할 때 그 리소스를 어떻게 할지. 데이터 보존용 안전장치.

- **Delete(기본)**: 같이 삭제. 단 **S3 버킷은 비어 있지 않으면 Delete가 안 된다.**
- **Retain**: CloudFormation이 지워도 **남긴다**(모든 리소스에 가능).
- **Snapshot**: 지우기 전에 **마지막 스냅샷**을 뜬다. 지원: EBS Volume·ElastiCache·RDS·Redshift·Neptune·DocumentDB 등.

### 안전장치 — Stack Policy · Termination Protection · Service Role · Capabilities

- **Stack Policy**: 스택 **업데이트 중** 어떤 리소스에 어떤 변경을 허용할지 정하는 JSON. 의도치 않은 업데이트로부터 보호. **정책을 걸면 기본은 전부 보호**되니, 바꾸고 싶은 리소스엔 명시적 **ALLOW**가 필요.
- **Termination Protection**: 스택의 **실수 삭제**를 막는다.
- **Service Role**: CloudFormation이 내 대신 리소스를 만들도록 주는 IAM Role. 사용자에게 리소스 권한을 직접 안 줘도 스택을 다루게 해 **최소 권한**을 지킨다. 사용자는 **`iam:PassRole`** 권한이 필요.
- **Capabilities**: 템플릿이 **IAM 리소스를 만들면** `CAPABILITY_IAM`(이름을 직접 지정하면 `CAPABILITY_NAMED_IAM`)를 줘야 한다. **Macros·Nested Stacks**가 있으면 `CAPABILITY_AUTO_EXPAND`. 안 주면 **`InsufficientCapabilitiesException`**.

### Custom Resources

CloudFormation이 아직 지원 안 하는 리소스, 외부(온프레미스·서드파티) 리소스, 또는 create/update/delete 때 **커스텀 스크립트**(예: **S3 버킷을 비우고 나서** 삭제 — 비어 있지 않은 버킷은 못 지우므로)를 돌릴 때. `Custom::이름` 타입으로 정의하고, **Lambda(가장 흔함) 또는 SNS**가 뒤를 받친다. **`ServiceToken`**(Lambda ARN·SNS ARN, 같은 리전)으로 어디에 요청을 보낼지 지정.

### StackSets

**여러 계정·여러 리전**에 스택을 한 번의 작업으로 생성·수정·삭제. StackSet을 업데이트하면 연결된 모든 stack instance가 전부 갱신된다. **AWS Organization 전체 계정**에 적용 가능하고, **관리자 계정(또는 위임 관리자)만** StackSet을 만들 수 있다.

## 운영 관점 (CloudOps) #cloudops

![[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=138]]

운영 시험은 템플릿 문법(위 DVA 섹션)보다 **이미 배포된 스택을 운영하다 생기는 문제**를 묻는다. 수동 변경 감지(Drift), 여러 계정·리전 배포(StackSets)의 권한 모델, ASG를 무중단으로 갱신하는 UpdatePolicy, EC2 구성 완료 신호(cfn-signal), 그리고 실패 상태값별 대처가 핵심이다.

### Drift — 수동 변경 감지

CloudFormation은 인프라를 만들어 주지만 **누군가 콘솔에서 직접 바꾼 것까지 막지는 못한다.** 예를 들어 스택이 만든 보안 그룹의 SSH 소스를 콘솔에서 `0.0.0.0/0`으로 바꿔 버리면 템플릿과 실제가 어긋난다. **Drift Detection**은 템플릿이 기대하는 상태와 실제 리소스를 비교해 어긋난(drifted) 리소스를 찾아낸다. **스택 전체** 또는 **개별 리소스** 단위로 돌릴 수 있다. → "수동 변경으로 인프라가 템플릿과 달라졌는지 확인"이면 Drift.

### StackSet 심화 — 권한 모델·Organizations·Drift

기본 개념(여러 계정·리전 한 번에 배포)은 DVA 섹션에 있고, 운영에서는 **권한을 어떻게 거느냐**가 출제된다.

- **Self-managed Permissions**: 관리자 계정과 대상 계정 **양쪽에 IAM Role을 직접 만든다**(신뢰 관계 설정). 관리자엔 `AWSCloudFormationStackSetAdministrationRole`, 대상엔 `AWSCloudFormationStackSetExecutionRole`. Organization에 속하지 않은 계정에도 배포할 때.
- **Service-managed Permissions**: **AWS Organizations**가 관리하는 계정에 배포. StackSet이 **IAM Role을 알아서 만든다**(Organizations와 **trusted access** 활성화 필요, **all features** 켜야 함). 앞으로 Organization에 **새로 추가되는 계정에도 자동 배포**(Automatic Deployments)된다.
- **StackSet Drift Detection**: 각 stack instance가 연결된 스택에 drift 검사를 돌린다. 리소스가 어긋나면 그 스택·stack instance·StackSet 모두 drifted로 본다. 단, **StackSet이 아니라 스택에 직접 가한 CloudFormation 변경은 drift로 치지 않는다**(관리 밖 수동 변경만 잡는다).
- **StackSet 트러블슈팅**: stack instance가 **`OUTDATED`** 상태 → 대상 계정 권한 부족, S3 버킷처럼 **전역 유일해야 하는 이름 충돌**, 관리자-대상 간 **신뢰 관계 없음**, 대상 계정의 **한도(quota) 초과** 중 하나.

### UpdatePolicy — ASG 무중단 갱신

ASG의 Launch 설정 등을 바꿀 때 인스턴스를 **어떻게 교체할지** 정한다(운영에서 자주 출제).

- **`AutoScalingRollingUpdate`**: 기존 ASG 안에서 인스턴스를 **배치(batch) 단위로** 교체. `MaxBatchSize`(한 번에 몇 개), `MinInstancesInService`(교체 중 최소 가동 수), `PauseTime`, **`WaitOnResourceSignals`**(새 인스턴스가 cfn-signal로 준비 완료를 알릴 때까지 대기). 용량을 크게 안 늘리고 점진 교체.
- **`AutoScalingReplacingUpdate`**(`WillReplace: true`): **새 ASG를 통째로 만들어** 기존 것을 대체. **두 ASG가 동시에 떠 있을 EC2 용량**이 필요. 업데이트 실패 시 새 ASG는 지우고 **기존 ASG를 그대로 유지**(롤백이 깔끔).

### EC2 구성과 완료 신호 — cfn-init · cfn-signal · WaitCondition

UserData 스크립트만으로는 "설정이 길어지고, 재생성 없이 상태를 바꾸기 어렵고, 성공했는지 알 수 없다"는 한계가 있다. **CloudFormation Helper Scripts**(Amazon Linux 기본 탑재, 아니면 yum/dnf 설치)로 푼다.

- **`AWS::CloudFormation::Init`**(리소스의 **Metadata**에 둠): `packages`·`groups`·`users`·`sources`·`files`·`commands`·`services`를 **이 순서로** 선언형으로 구성. 복잡한 EC2 설정을 읽기 쉽게 만든다.
- **`cfn-init`**: 인스턴스가 메타데이터를 받아 패키지 설치·파일 생성·서비스 시작을 수행. 로그는 **`/var/log/cfn-init.log`**.
- **`cfn-signal` + `WaitCondition`**: cfn-init 직후 `cfn-signal`로 **성공/실패를 CloudFormation에 알린다.** 리소스에 **`CreationPolicy`**(`ResourceSignal`의 `Timeout`·`Count`)를 달면 신호가 올 때까지(또는 타임아웃까지) 스택 진행을 막는다. EC2·ASG에 쓴다.
- **"WaitCondition이 신호를 못 받았다"** 트러블슈팅: AMI에 helper scripts가 있는지, cfn-init·cfn-signal이 실제로 돌았는지(`/var/log/cloud-init.log`·`cfn-init.log`), 로그를 보려면 **실패 시 rollback을 꺼야** 인스턴스가 안 지워진다, 그리고 인스턴스가 **인터넷에 닿는지**(public이면 IGW, private이면 NAT — `curl -I https://aws.amazon.com`로 확인).

### Dynamic References — 외부 비밀·설정 참조

템플릿에 비밀번호를 박지 않고, **Parameter Store나 Secrets Manager의 값을 배포 시점에 끌어온다**. 형식은 **`{{resolve:service-name:reference-key}}`**.

- **`ssm`**(평문 SSM 파라미터), **`ssm-secure`**(SSM SecureString), **`secretsmanager`**(Secrets Manager 비밀).
- **RDS 비밀번호** 다루는 두 방법: ① **`ManageMasterUserPassword: true`** — RDS·Aurora가 Secrets Manager에 비밀을 **알아서 만들고 자동 교체(rotation)**까지 관리(`MasterUserSecret.SecretArn`을 Output으로 노출). ② **Dynamic Reference** — `AWS::SecretsManager::Secret`으로 비밀을 만들고 RDS가 `{{resolve:secretsmanager:...}}`로 참조, `SecretTargetAttachment`로 둘을 연결해 rotation까지.

### Nested Stacks vs Cross Stacks

- **Cross Stacks**: 스택끼리 **생명주기가 다를 때**. `Outputs` export + `Fn::ImportValue`로 VPC ID 같은 값을 **여러 스택에 넘긴다**.
- **Nested Stacks**: ALB 구성·보안 그룹처럼 **반복되는 패턴을 재사용**할 때. 부모(root) 스택에 종속된 부품이라 공유하지 않으며, **업데이트는 항상 부모 스택을 갱신**해 처리. 모범 사례로 권장.

### 배포·삭제 운영 — DependsOn · OnFailure · Tags

- **`DependsOn`**: 리소스 생성 **순서**를 강제(A를 B 뒤에 만들기). `!Ref`·`!GetAtt`로 참조하면 자동으로 걸리지만, 참조가 없는 의존은 명시한다.
- **Stack 생성 실패 시 동작(`OnFailure`)**: **`ROLLBACK`(기본)** 만든 것 삭제 / **`DO_NOTHING`** 다 남기고 `CREATE_FAILED`로 둠(디버깅용) / **`DELETE`** 전부 삭제. CLI `create-stack --on-failure`.
- **트러블슈팅 상태값**: **`DELETE_FAILED`** — 비어 있지 않은 S3·인스턴스가 남은 보안 그룹처럼 **먼저 비워야 지워지는 리소스** 때문(Custom Resource로 비우거나 `DeletionPolicy: Retain`으로 건너뜀). **`UPDATE_ROLLBACK_FAILED`** — 외부 변경·권한 부족·ASG 신호 부족 등 → 수동 수정 후 **`ContinueUpdateRollback`**.
- **리전을 옮겼더니 실패**: 그 리전에 **서비스가 있는지**, **AMI ID(리전마다 다름)**, 박아 둔 **리전별 값(ARN 등)**, **전역 유일 이름(S3 버킷)** 충돌을 확인. EC2 **private DNS 이름**처럼 콘솔에서도 못 정하는 값은 템플릿으로도 못 정한다.
- **Stack-Level Tags**: 스택에 단 태그는 **지원되는 모든 리소스로 자동 전파**돼 비용·관리 분류에 쓰인다(최대 50개).

## 시험 함정

- 여러 **계정·리전**에 한 번에 스택 배포 → **StackSets**(Organization 전체, 관리자 계정만 생성) #exam/trap/cloudformation
- 스택 삭제 시 리소스 보존 → **DeletionPolicy: Retain**, 마지막 백업 → **Snapshot**(RDS·EBS·ElastiCache 등). 기본은 Delete이고 **비어 있지 않은 S3는 Delete 실패** #exam/trap/cloudformation
- 비어 있지 않은 S3를 지워야 함·미지원 리소스·생성 시 커스텀 스크립트 → **Custom Resources**(Lambda/SNS, `ServiceToken`) #exam/trap/cloudformation
- 템플릿이 **IAM 리소스**를 만들면 `CAPABILITY_IAM`(이름 지정 시 `CAPABILITY_NAMED_IAM`), Macro/Nested면 `CAPABILITY_AUTO_EXPAND`. 안 주면 **InsufficientCapabilitiesException** #exam/trap/cloudformation
- 다른 스택의 값 가져오기 → export(`Outputs`) + **`Fn::ImportValue`**. import 중이면 export한 스택 삭제 불가 #exam/trap/cloudformation
- 리소스의 **물리 ID**는 `Ref`, **속성(예: PublicDnsName)**은 `Fn::GetAtt`. UserData 인코딩은 `Fn::Base64` #exam/trap/cloudformation
- 사용자에게 리소스 권한 없이 스택만 다루게 → **Service Role**(사용자는 `iam:PassRole` 필요) #exam/trap/cloudformation
- 미리 정해진 값(리전별 AMI 등) → **Mappings + Fn::FindInMap**. 사용자별 입력 → **Parameters** #exam/trap/cloudformation
- 비밀번호 파라미터 가리기 → **NoEcho: true** #exam/trap/cloudformation
- 업데이트 중 특정 리소스 보호 → **Stack Policy**(걸면 기본 전부 보호, 명시적 ALLOW 필요). 스택 실수 삭제 방지 → **Termination Protection** #exam/trap/cloudformation
- 롤백마저 실패 → 수동 수정 후 **ContinueUpdateRollback** #exam/trap/cloudformation
- **콘솔에서 수동으로 바꿔 템플릿과 어긋났는지** 확인 → **Drift Detection**(스택/리소스 단위). StackSet 직접 CloudFormation 변경은 drift 아님 #exam/trap/cloudformation
- StackSet 권한: Organization 안이면 **Service-managed**(Role 자동 생성, trusted access·all features, 새 계정 자동 배포), 밖이면 **Self-managed**(양쪽 Role 직접 생성) #exam/trap/cloudformation
- stack instance가 **`OUTDATED`** → 대상 계정 권한 부족·전역 이름 충돌(S3)·신뢰 관계 없음·한도 초과 #exam/trap/cloudformation
- ASG 갱신: 같은 ASG에서 배치 교체 **AutoScalingRollingUpdate**(`WaitOnResourceSignals`로 신호 대기) vs 새 ASG 통째 교체 **AutoScalingReplacingUpdate**(`WillReplace`, 2배 용량 필요) #exam/trap/cloudformation
- 복잡한 EC2 설정은 **AWS::CloudFormation::Init**(Metadata) + **cfn-init**, 구성 완료 신호는 **cfn-signal** + **CreationPolicy/WaitCondition**(Count·Timeout). 로그 `/var/log/cfn-init.log` #exam/trap/cloudformation
- 비밀을 템플릿에 안 박고 참조 → **Dynamic References** `{{resolve:ssm/ssm-secure/secretsmanager:...}}`. RDS는 **ManageMasterUserPassword: true**로 Secrets Manager 자동 생성·rotation #exam/trap/cloudformation
- 반복 패턴 재사용 → **Nested Stacks**(부모 갱신으로 업데이트), 생명주기 다른 스택에 값 전달 → **Cross Stacks**(export + ImportValue) #exam/trap/cloudformation
- 생성 실패 시 **OnFailure**: ROLLBACK(기본)·**DO_NOTHING**(남겨서 디버깅)·DELETE. 생성 순서 강제는 **DependsOn** #exam/trap/cloudformation

## 관련 노트

[[CDK]] · [[Elastic Beanstalk]] · [[Systems Manager]] · [[Lambda]] · [[KMS & 암호화]] · [[S3]] · [[Organizations & 계정 관리]] · [[ELB & Auto Scaling]] · [[EC2]] · [[CloudWatch & CloudTrail & Config]]
