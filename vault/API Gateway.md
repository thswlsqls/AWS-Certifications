---
service: Amazon API Gateway
exams: [saa, dva]
domains:
  - saa/high-performing-architectures
  - dva/development
status: reviewing
confidence: 1
tags:
  - service
  - saa/serverless
  - dva/serverless
---

# API Gateway

> API 관리 — 통합 유형, 스테이지·배포, 인증(Cognito/IAM/Lambda Authorizer), 스로틀링.

## 개요

클라이언트와 백엔드 사이에 두는 **API 입구**. 서버리스 앱에서 클라이언트 → API Gateway → [[Lambda]] → [[DynamoDB]]로 이어지는 그림의 맨 앞 계층이다. 직접 인프라를 두지 않고도 REST API(와 WebSocket)를 노출하고, 그 앞에서 인증·요청 제한(throttling)·캐싱·버전 관리·요청/응답 변환을 한꺼번에 처리한다.

핵심 가치는 "백엔드 앞에 관리 계층을 한 겹 씌운다"는 것이다. Lambda뿐 아니라 사내 HTTP 엔드포인트나 다른 AWS 서비스 앞에 붙여, 그 자체로는 없는 인증·rate limiting·API 키 같은 기능을 더한다.

## 설계 관점 (SAA) #saa

![[AWS Certified Solutions Architect Slides v47.pdf#page=482]]

> 추천 학습 순서상 "큰 그림" 자리입니다. 스테이지·배포·매핑 템플릿 등 세부는 3단계(DVA)에서 채웁니다. 여기서는 SAA에 나오는 통합 유형·엔드포인트·인증만 정리합니다.

### 무엇을 해 주나

WebSocket 지원, API 버전 관리(v1·v2), 환경 분리(dev·test·prod), 인증/인가, API 키 발급과 요청 throttling, Swagger/OpenAPI로 빠르게 정의, 요청·응답 변환과 검증, SDK 생성, 응답 캐싱.

### 통합 유형 — 뒤에 무엇을 붙이나

- **Lambda**: REST API를 Lambda로 받는 가장 흔한 조합.
- **HTTP**: 사내 HTTP API나 [[ELB & Auto Scaling|ALB]] 같은 백엔드 앞에 둬서 rate limiting·캐싱·인증·API 키를 더한다.
- **AWS Service**: 다른 AWS API를 그대로 노출. 예를 들어 클라이언트 요청을 [[SQS & SNS & Kinesis|SQS]]에 넣거나 Step Functions를 시작. 인증을 붙이고 공개적으로 안전하게 노출하려는 목적.

### 엔드포인트 타입 3종 (단골)

| 타입 | 누가 접근 | 특징 |
| --- | --- | --- |
| **Edge-Optimized** (기본) | 전 세계 클라이언트 | 요청이 [[CloudFront & Global Accelerator\|CloudFront]] 엣지를 거쳐 지연이 준다. API Gateway 자체는 **한 리전**에만 있음 |
| **Regional** | 같은 리전 클라이언트 | CloudFront를 직접 붙여 캐싱·배포를 더 통제하고 싶을 때 |
| **Private** | **내 VPC 안에서만** | 인터페이스 VPC 엔드포인트(ENI)로만 접근. 접근 범위는 **resource policy**로 정의 |

### 보안 — 인증 3종 (단골)

- **IAM Roles**: 사내(AWS 내부) 애플리케이션에 적합.
- **Cognito**: 외부 사용자(모바일 등)의 신원. 클라이언트가 [[Cognito]]에서 토큰을 받아 API Gateway에 넘기면 게이트웨이가 검증.
- **Custom Authorizer (Lambda Authorizer)**: 내 로직으로 직접 인증.

**Custom Domain Name + HTTPS**는 ACM 인증서로 한다. 여기서 함정:

- **Edge-Optimized 엔드포인트면 인증서가 `us-east-1`에 있어야** 한다.
- **Regional 엔드포인트면 인증서가 그 API Gateway 리전**에 있어야 한다.
- [[Route 53]]에 CNAME 또는 A-alias 레코드를 만들어 연결.

## 개발 관점 (DVA) #dva

![[AWS Certified Developer Slides v44.pdf#page=657]]

SAA 섹션이 "뒤에 뭘 붙이고 어떤 엔드포인트·인증을 고르나"였다면, 여기서는 배포 단위(Stage)와 통합을 코드로 어떻게 다루는지, 캐싱·요청 제한·보안을 어떻게 설정하는지를 정리한다. DVA에서 가장 자주 나오는 단원이다.

### 배포(Deployment)와 스테이지(Stage)

- API Gateway에서 설정을 바꿔도 **배포(deployment)를 해야 실제로 반영**된다. 자주 헷갈리는 지점.
- 변경은 **Stage**(dev·test·prod 등 이름 자유)에 배포한다. 스테이지마다 설정 값이 따로 있고, **배포 이력이 남아 롤백** 가능.
- v1·v2처럼 호환 깨지는 변경은 스테이지를 나눠(`/v1`, `/v2`) 각각 다른 URL로 노출.

### Stage Variables — 환경별 설정값

API Gateway의 환경 변수 같은 것. 자주 바뀌는 설정값을 담는다.

- 쓰는 곳: **Lambda 함수 ARN**, HTTP 엔드포인트, 매핑 템플릿 파라미터.
- 용도: 스테이지마다 다른 백엔드를 가리키게 하거나, Lambda에 설정값 전달. Lambda에서는 `context` 객체로 들어온다. 형식은 `${stageVariables.변수명}`.
- **Stage Variable + Lambda Alias** 조합: 스테이지 변수가 Lambda alias를 가리키게 하면, API Gateway는 건드리지 않고 **Lambda alias만 바꿔** prod·test·dev가 각기 다른 버전을 호출하게 할 수 있다.

### Canary Deployment

특정 스테이지(보통 prod)에서 **트래픽 일부 %만 새 버전(canary)** 으로 보낸다. 지표·로그가 분리돼 모니터링하기 좋고, canary용 스테이지 변수를 따로 덮어쓸 수 있다. Lambda + API Gateway의 **블루/그린 배포** 방식.

### 통합 유형 (Integration Types) — 매핑 템플릿 필요 여부가 핵심

| 유형 | 매핑 템플릿 | 설명 |
| --- | --- | --- |
| **MOCK** | — | 백엔드로 안 보내고 API Gateway가 응답을 바로 돌려줌(테스트용) |
| **HTTP / AWS** | **필요** | integration request·response를 직접 구성하고 **매핑 템플릿**으로 요청·응답 데이터를 변환 |
| **AWS_PROXY (Lambda Proxy)** | **불필요** | 클라이언트 요청이 그대로 Lambda 입력으로. 헤더·쿼리스트링이 인자로 전달되고, 요청/응답 로직은 **Lambda가 책임** |
| **HTTP_PROXY** | **불필요** | HTTP 요청을 백엔드로 그대로 전달, 응답도 그대로 포워딩. 필요하면 헤더만 추가 |

→ "매핑 템플릿 없이 요청을 그대로 넘긴다" = **Proxy 유형**. "요청/응답을 변환·검증한다" = HTTP/AWS 유형 + 매핑 템플릿.

### Mapping Templates — 요청/응답 변환

AWS·HTTP 통합에서 요청·응답을 바꾼다. 쿼리스트링 이름 변경, body 내용 수정, 헤더 추가, 출력 결과 필터링. **VTL(Velocity Template Language)** 로 for·if를 쓴다. Content-Type을 `application/json`·`application/xml`로 지정. 대표 예: REST(JSON) 클라이언트와 **SOAP(XML)** 백엔드 사이에서 JSON↔XML 변환.

### 요청 검증(Request Validation)

백엔드로 넘기기 전에 API Gateway가 기본 검증을 한다. 실패하면 즉시 **400 에러**를 반환해 불필요한 백엔드 호출을 줄인다. URI·쿼리스트링·헤더의 필수 파라미터가 있는지, body가 지정한 **JSON Schema** 모델에 맞는지 확인. OpenAPI 정의 파일로 검증기를 설정(`params-only` / `all`).

### 응답 캐싱

- 백엔드 호출 수를 줄인다. **기본 TTL 300초**(최소 0, 최대 3600초).
- **캐시는 스테이지 단위**로 정의하고, **메서드별로 덮어쓰기** 가능. 캐시 암호화 옵션, 용량 0.5GB~237GB.
- 캐시는 비싸서 보통 prod에만. **캐시 무효화**: 전체 flush 가능하고, 클라이언트가 `Cache-Control: max-age=0` 헤더로 무효화하려면 **IAM 권한(`execute-api:InvalidateCache`)** 이 필요. 정책을 안 걸면 누구나 무효화할 수 있다.

### Usage Plans & API Keys — 유료 API 제공

API를 고객에게 상품으로 팔 때 쓴다.

- **Usage Plan**: 누가 어떤 스테이지·메서드에 접근하는지, 얼마나·얼마나 빠르게 쓰는지(throttle·quota)를 API 키 단위로 건다.
- **API Key**: 고객에게 나눠주는 문자열. 호출자는 `x-api-key` 헤더에 키를 넣어 보낸다.
- **설정 순서(시험 출제)**: ① API 만들고 메서드에 API 키 요구 설정 후 스테이지 배포 → ② API 키 생성/배포 → ③ throttle·quota 정한 Usage Plan 생성 → ④ **스테이지·API 키를 Usage Plan에 연결**.

### Throttling과 에러 코드

- 계정 한도: 모든 API 합쳐 **초당 10,000 요청**(soft limit, 증액 요청 가능). 초과 시 **429 Too Many Requests**(재시도 가능). Stage/Method limit이나 Usage Plan으로 더 세밀하게 제한. **한 API가 과부하면 다른 API까지 throttle** 될 수 있다(Lambda 동시성과 비슷).
- 4xx 클라이언트 오류: **400** Bad Request, **403** Access Denied·WAF 차단, **429** quota 초과·throttle.
- 5xx 서버 오류: **502** Bad Gateway(Lambda proxy가 잘못된 출력 반환 등), **503** Service Unavailable, **504** Integration Failure(타임아웃). **API Gateway 요청은 최대 29초**에서 타임아웃.

### CORS

다른 도메인에서 API를 호출할 때 필요. 브라우저의 **OPTIONS preflight** 요청에 `Access-Control-Allow-Methods`·`Access-Control-Allow-Headers`·`Access-Control-Allow-Origin` 헤더가 있어야 한다. 콘솔에서 켤 수 있다.

### 보안 3종 — 개발 관점 정리

(SAA 섹션의 선택 기준에 더해)

- **IAM Permissions**: 인증=IAM, 인가=IAM Policy. **Sig v4**로 자격증명을 헤더에 실어 보낸다. AWS 안의 사용자·역할에 적합. **Resource Policy**를 더하면 **교차 계정 접근**, 특정 소스 IP, VPC 엔드포인트 허용 가능.
- **Cognito User Pools**: 인증=Cognito가 사용자 수명주기·토큰 만료까지 관리, 인가=API Gateway 메서드. 별도 구현 불필요. 클라이언트가 토큰을 받아 넘기면 게이트웨이가 자동 검증.
- **Lambda Authorizer**(구 Custom Authorizer): 인증=외부, 인가=Lambda 함수. **토큰 기반**(JWT·OAuth bearer 토큰) 또는 **요청 파라미터 기반**(헤더·쿼리스트링·스테이지 변수). Lambda가 **IAM 정책을 반환**하고 그 결과가 **캐시**된다. 서드파티 토큰을 쓸 때 적합.

### HTTP API vs REST API

- **HTTP API**: 저지연·저비용. Lambda proxy·HTTP proxy·private 통합(데이터 매핑 없음). **OIDC·OAuth 2.0** 인가, CORS 기본 지원. 단 **Usage Plan·API Key 없음**, Resource Policy 없음.
- **REST API**: 기능 전부(Native OIDC/OAuth 2.0만 빠짐). Usage Plan·API Key·Resource Policy가 필요하면 REST API.

### WebSocket API

서버가 클라이언트로 정보를 **밀어 보내는**(push) 양방향 통신. 채팅·실시간 협업·멀티플레이어 게임·실시간 거래에 쓴다. URL은 `wss://...`.

- 연결 시 **connectionId**가 만들어지고 이후 메시지에 재사용(보통 DynamoDB에 저장).
- **서버→클라이언트**: Connection URL(`.../@connections/{connectionId}`)에 **HTTP POST**(IAM Sig v4)로 메시지 전송. **GET**은 연결 상태 조회, **DELETE**는 연결 끊기.
- **라우팅**: 들어온 JSON 메시지를 **route selection expression**(예: `$request.body.action`)으로 평가해 route key(`$connect`·`$disconnect`·`$default`·사용자 정의)에 맞는 백엔드로 보낸다. 매칭 없으면 `$default`.

### 모니터링

- **CloudWatch Logs**: 스테이지 단위로 켜고 로그 레벨(ERROR·DEBUG·INFO) 지정, 요청/응답 body 기록.
- **CloudWatch Metrics**(스테이지 단위): `CacheHitCount`·`CacheMissCount`, `Count`, **`IntegrationLatency`**(게이트웨이↔백엔드 시간) vs **`Latency`**(클라이언트가 본 전체 시간), `4XXError`·`5XXError`.
- **X-Ray**: 추적을 켜면 요청 흐름을 본다. **X-Ray + API Gateway + Lambda**로 전체 그림을 얻는다.

## 시험 함정

- 전 세계 사용자 지연 줄이기 → **Edge-Optimized**(기본, CloudFront 엣지 경유). API 자체는 한 리전에만 존재 #exam/trap/api-gateway
- **Edge-Optimized면 ACM 인증서가 `us-east-1`**, Regional이면 해당 API Gateway 리전에 있어야 함 #exam/trap/api-gateway
- VPC 안에서만 쓰는 API → **Private 엔드포인트**(인터페이스 VPC 엔드포인트), 접근은 resource policy로 제한 #exam/trap/api-gateway
- 외부(모바일) 사용자 인증 → **Cognito**, 사내 앱 → **IAM**, 직접 로직 → **Custom Authorizer** #exam/trap/api-gateway
- 기존 HTTP/ALB 백엔드에 rate limiting·API 키·인증만 얹고 싶다 → API Gateway HTTP 통합 #exam/trap/api-gateway
- 매핑 템플릿 **없이** 요청을 그대로 전달 → **Proxy 유형**(AWS_PROXY/HTTP_PROXY). 요청·응답 변환 필요 → HTTP/AWS 통합 + 매핑 템플릿(VTL) #exam/trap/api-gateway
- 설정을 바꿔도 **배포(deployment)를 해야 반영**됨. 환경별 설정은 **Stage Variables**(Lambda alias와 묶어 버전 전환) #exam/trap/api-gateway
- 응답 캐시 **기본 TTL 300초**(0~3600), **스테이지 단위**·메서드별 덮어쓰기. 클라이언트 캐시 무효화는 `Cache-Control: max-age=0` + IAM 권한 #exam/trap/api-gateway
- API Gateway 요청 **타임아웃 29초**. 502=Lambda proxy 잘못된 출력, 504=통합 실패/타임아웃, 429=throttle #exam/trap/api-gateway
- Usage Plan 구성 순서: API+키요구 배포 → API 키 생성 → Usage Plan 생성 → **스테이지·키를 플랜에 연결**. 호출자는 `x-api-key` 헤더 #exam/trap/api-gateway
- **Usage Plan·API Key가 필요하면 REST API**(HTTP API엔 없음). OIDC/OAuth2.0·저비용이면 HTTP API #exam/trap/api-gateway
- 서버가 클라이언트로 push·실시간 양방향 → **WebSocket API**. 서버→클라 전송은 `@connections/{connectionId}`에 HTTP POST #exam/trap/api-gateway
- 서드파티 토큰(JWT/OAuth) 인증 → **Lambda Authorizer**(IAM 정책 반환·캐시). 교차 계정 접근 → IAM + **Resource Policy** #exam/trap/api-gateway
- 백엔드 호출 전 요청 검증(필수 파라미터·JSON Schema) 실패 → API Gateway가 **400** 즉시 반환 #exam/trap/api-gateway

## 관련 노트

[[Lambda]] · [[Cognito]] · [[Step Functions & AppSync]] · [[DynamoDB]] · [[SQS & SNS & Kinesis]] · [[CloudFront & Global Accelerator]] · [[Route 53]] · [[ELB & Auto Scaling]] · [[CloudWatch & CloudTrail & Config]] · [[IAM]]
