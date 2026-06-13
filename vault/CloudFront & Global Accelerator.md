---
service: Amazon CloudFront · AWS Global Accelerator
exams: [saa, dva, cloudops]
domains:
  - saa/high-performing-architectures
  - dva/development
  - cloudops/networking
status: learning
confidence: 1
tags:
  - service
  - saa/network
  - dva/network
  - cloudops/network
---

# CloudFront & Global Accelerator

> CDN과 글로벌 가속 — 오리진, 캐시 정책, OAC, 지리적 제한.

## 개요

둘 다 전 세계에 흩어진 AWS 엣지 로케이션과 내부망을 써서 사용자에게 더 빠르게 콘텐츠를 전달하는 서비스다. 다만 일하는 방식이 다르다.

- **CloudFront** — CDN(Content Delivery Network). 콘텐츠를 **엣지에 캐싱**해서 읽기 성능을 올린다. 전 세계 수백 개 PoP(엣지 로케이션). 전 세계에 퍼져 있어 DDoS 방어에 유리하고 Shield·WAF와 붙는다.
- **Global Accelerator** — 캐싱은 안 하고, **AWS 내부망으로 트래픽을 빠르게 흘려보내는** 가속기. 앱 앞에 고정 Anycast IP 2개를 두고, 사용자를 가까운 엣지로 보낸 뒤 내부망으로 원본 앱까지 전달한다.

## 설계 관점 (SAA) #saa

![[AWS Certified Solutions Architect Slides v47.pdf#page=335]]

### CloudFront — 오리진 종류

CloudFront가 콘텐츠를 가져오는 원본(origin)은 세 가지다.

- **S3 버킷**: 파일을 엣지에 캐싱. **OAC(Origin Access Control) + 버킷 정책**으로 보호해서, S3를 공개하지 않고도 CloudFront만 접근하게 한다. (CloudFront 통해 업로드도 가능)
- **VPC Origin**: private 서브넷에 있는 앱(private ALB·NLB·EC2)에 직접. 인터넷에 노출할 필요가 없다.
- **Custom Origin (HTTP)**: S3 정적 웹사이트(먼저 버킷을 정적 웹사이트로 켜야 함), 또는 아무 public HTTP 백엔드(예: public ALB).

### CloudFront vs S3 Cross-Region Replication — 헷갈리는 구분

| | CloudFront | S3 CRR |
| --- | --- | --- |
| 범위 | 전 세계 엣지 네트워크 | 리전마다 따로 설정 |
| 갱신 | TTL 동안 캐싱(하루 정도일 수도) | 거의 실시간 |
| 성격 | 읽기 캐시 | 복제본(읽기 전용) |
| 적합 | **어디서나 필요한 정적 콘텐츠** | **몇몇 리전에 저지연으로 둘 동적 콘텐츠** |

### CloudFront — 보안·캐시 관리

- **ALB·EC2를 오리진으로(public network)**: 엣지 로케이션의 public IP를 보안 그룹에서 열어줘야 한다. EC2 직접이면 EC2가 public이어야 하고, ALB를 두면 ALB는 public·EC2는 private으로 둘 수 있다.
- **Geo Restriction(지리적 제한)**: 국가 단위 allowlist(허용 목록) / blocklist(차단 목록). 국가 판별은 3rd party Geo-IP DB. 저작권 때문에 특정 국가만 막을 때.
- **Cache Invalidation(캐시 무효화)**: 오리진을 바꿔도 CloudFront는 TTL 끝나야 새 내용을 가져온다. 즉시 갱신하려면 무효화로 TTL을 건너뛴다. 전체(`*`) 또는 경로(`/images/*`) 단위.

### Global Accelerator — 동작과 용도

- **Unicast vs Anycast**: Unicast는 서버마다 다른 IP, **Anycast는 여러 서버가 같은 IP를 쓰고 클라이언트는 가장 가까운 곳으로** 라우팅된다. Global Accelerator는 앱마다 **Anycast IP 2개**를 만든다.
- Anycast IP로 들어온 트래픽이 가까운 엣지 → AWS 내부망 → 원본 앱으로 간다. EIP·EC2·ALB·NLB(public/private)와 동작.
- **일관된 성능**: 최저 지연으로 지능형 라우팅, 빠른 리전 failover. **IP가 안 바뀌어 클라이언트 캐시 문제 없음.**
- **Health Check**로 비정상 시 **1분 안에 failover** → 재해 복구에 좋음.
- 보안: **외부에 열어줄 IP가 2개뿐**, Shield로 DDoS 방어.

### CloudFront vs Global Accelerator — 무엇을 고르나

둘 다 AWS 글로벌 네트워크·엣지·Shield를 쓴다. 차이는 "캐싱하느냐"와 "무슨 프로토콜이냐"다.

| | CloudFront | Global Accelerator |
| --- | --- | --- |
| 방식 | 엣지에 **캐싱**해서 제공 | 엣지에서 **프록시**만(캐싱 안 함) |
| 콘텐츠 | 캐싱 가능 콘텐츠(이미지·영상) + 동적 콘텐츠(API 가속) | TCP/UDP 임의 앱 |
| 잘 맞는 경우 | 웹 콘텐츠 전달 | **비HTTP(게임 UDP, IoT MQTT, VoIP)**, **정적 IP가 필요한 HTTP**, 결정적이고 빠른 리전 failover |

## 개발 관점 (DVA) #dva

![[AWS Certified Developer Slides v44.pdf#page=281]]

%% 이 시험의 관점에서 배운 내용을 정리합니다. %%

## 운영 관점 (CloudOps) #cloudops

![[AWS Certified CloudOps Engineer Associate Slides v41.pdf#page=313]]

%% 이 시험의 관점에서 배운 내용을 정리합니다. %%

## 시험 함정

- 캐싱 가능한 웹 콘텐츠(이미지·영상)·API 가속 → **CloudFront**, TCP/UDP 임의 앱·정적 IP·게임/IoT/VoIP → **Global Accelerator** #exam/trap/cloudfront
- "어디서나 정적 콘텐츠" → CloudFront, "몇몇 리전에 동적 콘텐츠 저지연" → **S3 CRR** #exam/trap/cloudfront
- S3를 공개하지 않고 CloudFront로만 노출 → **OAC + 버킷 정책** #exam/trap/cloudfront
- Global Accelerator는 **Anycast IP 2개**, IP가 안 바뀜(클라이언트 캐시 문제 없음) #exam/trap/cloudfront
- 오리진 바꾼 즉시 반영하려면 TTL 안 기다리고 **Cache Invalidation** #exam/trap/cloudfront
- 국가 단위 접근 제어(저작권) → CloudFront **Geo Restriction** #exam/trap/cloudfront

## 관련 노트

[[S3]] · [[Route 53]] · [[ELB & Auto Scaling]] · [[VPC]]
