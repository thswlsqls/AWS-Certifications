# AWS-Certifications

AWS Associate 자격증 3종을 준비하기 위한 **Obsidian vault** 저장소입니다. 코드 프로젝트가 아니며, 모든 노트는 한국어로 작성되어 있습니다.

| 시험 | 관점 | 태그 |
| --- | --- | --- |
| SAA-C03 (Solutions Architect) | 설계 | `#saa` |
| DVA-C02 (Developer) | 개발 | `#dva` |
| SOA-C03 (CloudOps Engineer) | 운영 | `#cloudops` |

## 저장소 구조

```
AWS-Certifications/
├── vault/                  # Obsidian vault (Obsidian에서 이 폴더를 열기)
│   ├── 001 Home.md         # vault 규칙과 서비스 노트 지도 — 시작점
│   ├── 002 추천 학습 순서.md  # SAA → DVA → CloudOps 학습 순서 가이드
│   ├── 003 SAA-C03.md      # 시험 노트 (도메인 비중표, 슬라이드 목차)
│   ├── 004 DVA-C02.md
│   ├── 005 SOA-C03.md
│   ├── S3.md, EC2.md, …    # 서비스 노트 (vault 루트에 평평하게 배치)
│   ├── 진도표.base          # Obsidian Bases 진도 추적기
│   ├── Templates/          # 서비스 노트 템플릿
│   └── Attachments/        # 강의 슬라이드 PDF (gitignore — 직접 세팅, 아래 참고)
└── slides/                 # 슬라이드 PDF 사본 (gitignore, vault 외부 보관용)
```

## 세팅 가이드: 슬라이드 PDF 준비

이 저장소에는 **노트만** 들어 있습니다. Udemy 유료 강의의 슬라이드 PDF는 저작권 문제로 `.gitignore`에 의해 제외되어 있으므로, 해당 강의를 결제한 수강생이 본인의 강의 자료를 직접 넣어야 vault가 완성됩니다.

1. 이 저장소를 클론하고 Obsidian에서 `vault/` 폴더를 엽니다.
2. 수강 중인 Udemy 강의의 강의 자료(course resources)에서 슬라이드 PDF를 다운로드합니다.
3. PDF를 `vault/Attachments/`에 **아래 파일명 그대로** 넣습니다. 노트의 임베드(`![[...pdf#page=N]]`)가 이 파일명을 참조하므로, 파일명이 다르면 (버전이 달라 다운로드한 이름이 다른 경우 포함) 파일명을 아래에 맞춰 변경해야 합니다.

   | 시험 | 파일명 |
   | --- | --- |
   | SAA-C03 | `AWS Certified Solutions Architect Slides v47.pdf` |
   | DVA-C02 | `AWS Certified Developer Slides v44.pdf` |
   | SOA-C03 | `AWS Certified CloudOps Engineer Associate Slides v41.pdf` |

4. 끝입니다. 노트 곳곳의 슬라이드 임베드와 시험 노트의 슬라이드 목차가 자동으로 연결됩니다.

> **버전 주의**: 노트의 페이지 참조(`#page=N`)는 위 버전 기준입니다. 강의 자료가 더 최신 버전이면 파일명을 맞춰도 페이지 번호가 어긋날 수 있습니다.
>
> `slides/` 폴더는 vault 외부 보관용 사본일 뿐이므로 채우지 않아도 vault 동작에는 영향이 없습니다. 노트에서 링크하지도 마세요.

## 핵심 원칙: 서비스당 노트 1개

S3처럼 세 시험 모두에 등장하는 서비스도 노트는 **하나만** 둡니다. 시험별 폴더나 노트 복제는 만들지 않고, 시험과의 관계는 노트 안에서 표현합니다.

- **관점 섹션**: `## 설계 관점 (SAA) #saa`, `## 개발 관점 (DVA) #dva`, `## 운영 관점 (CloudOps) #cloudops`
- **프론트매터**: `exams`, `domains`, `status`(todo → learning → reviewing → done), `confidence`(1~5) — `진도표.base`가 이 속성을 쿼리하므로 정확하게 유지해야 합니다.
- **시험 함정**: 틀린 문제는 해당 서비스 노트의 "시험 함정" 섹션에 `#exam/trap/<주제>` 태그와 함께 한 줄로 기록합니다.

## 사용 방법

1. Obsidian에서 `vault/` 폴더를 vault로 엽니다.
2. `001 Home.md`에서 시작해 시험 노트와 서비스 노트 지도를 따라갑니다. 공부 순서는 `002 추천 학습 순서.md`를 참고합니다.
3. 새 서비스 노트는 `Templates/서비스 노트 템플릿.md`를 적용해 vault 루트에 만듭니다.
4. 공부하면서 노트 상단의 `status`와 `confidence`를 갱신하면 `진도표.base`에 진도가 반영됩니다.
5. 강의자료는 `![[슬라이드.pdf#page=N]]` 형태로 노트 안에 임베드합니다.

## ⚠️ 저작권 주의

강의 슬라이드 PDF는 Udemy 유료 강의의 라이선스 자료이므로 `.gitignore`로 저장소에서 제외되어 있습니다. **PDF를 커밋하거나 어디에도 업로드하지 마세요.** 본인이 결제한 강의에서 받은 자료를 로컬에서만 사용해야 합니다.
