---
title: "AWS RDS와 EC2 직접 운영 DB 비교, GitHub Actions Trigger 이해하기"
categories:
  - Backend
  - DevOps
tags:
  - AWS
  - RDS
  - EC2
  - Database
  - GitHubActions
  - CI/CD
  - Trigger
  - Workflow
---

백엔드 애플리케이션을 운영하다 보면 데이터베이스와 배포 자동화는 거의 항상 함께 고민하게 된다.
데이터베이스는 직접 서버에 설치해서 운영할 수도 있고, AWS RDS처럼 관리형 서비스를 사용할 수도 있다.
또한 GitHub Actions를 사용하면 코드 변경, Pull Request, 배포 요청 등 다양한 이벤트를 기준으로 CI/CD 파이프라인을 자동 실행할 수 있다.

이 글에서는 먼저 **AWS RDS를 사용하는 주요 이점과 EC2에 직접 데이터베이스를 설치해 운영하는 방식과의 차이점**을 정리한다.
이어서 **GitHub Actions Workflow에서 사용할 수 있는 Trigger 유형과 각 Trigger가 적합한 CI/CD 시나리오**를 살펴본다.

---

## 1. AWS RDS를 활용하는 주요 이점

### RDS란?

`RDS(Relational Database Service)`는 AWS에서 제공하는 관리형 관계형 데이터베이스 서비스다.
MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, Amazon Aurora 같은 엔진을 선택해 사용할 수 있다.

직접 EC2에 DB를 설치하면 운영자가 OS, DB 설치, 패치, 백업, 장애 복구, 모니터링을 모두 관리해야 한다.
반면 RDS는 데이터베이스 운영에 필요한 많은 작업을 AWS가 대신 관리해준다.

즉, RDS의 핵심 가치는 단순히 "DB 서버를 만들어준다"가 아니라
**운영 부담을 줄이고 안정적인 데이터베이스 환경을 빠르게 구성할 수 있게 해준다**는 점이다.

---

### 1. 자동 백업과 복구

RDS는 자동 백업(Automated Backup)과 특정 시점 복구(Point-in-Time Recovery)를 지원한다.
설정한 보존 기간 안에서는 장애나 실수로 데이터가 손상되었을 때 원하는 시점으로 복구할 수 있다.

EC2에 직접 DB를 설치한 경우에는 백업 스크립트 작성, 백업 파일 저장 위치, 보존 정책, 복구 테스트까지 직접 관리해야 한다.
백업은 만들어두는 것보다 실제로 복구 가능한 상태인지 검증하는 것이 더 중요하기 때문에 운영 부담이 크다.

RDS를 사용하면 이러한 백업과 복구 체계를 비교적 쉽게 구성할 수 있다.

---

### 2. 고가용성 구성

RDS는 `Multi-AZ` 배포를 통해 고가용성(High Availability)을 제공한다.
Primary DB 인스턴스에 장애가 발생하면 다른 Availability Zone에 있는 Standby 인스턴스로 자동 Failover가 가능하다.

EC2에서 직접 DB를 운영한다면 Replication 구성, 장애 감지, Failover 처리, DNS 전환 등을 직접 설계해야 한다.
특히 장애 상황에서는 수동 대응 시간이 길어질 수 있고, 잘못된 Failover는 데이터 정합성 문제로 이어질 수 있다.

RDS의 Multi-AZ는 이런 복잡한 장애 대응 흐름을 관리형 기능으로 제공한다는 점에서 큰 장점이 있다.

---

### 3. 패치와 유지보수 부담 감소

DB 운영에서는 보안 패치, 마이너 버전 업데이트, OS 유지보수가 중요하다.
EC2에 DB를 직접 설치하면 운영자가 직접 패치 일정을 잡고, 패치 후 호환성 문제를 확인해야 한다.

RDS는 Maintenance Window를 설정해 정해진 시간에 패치가 적용되도록 관리할 수 있다.
물론 모든 업데이트가 완전히 자동으로 끝나는 것은 아니며, 운영 환경에서는 사전 검증이 필요하다.
하지만 EC2 직접 운영에 비해 반복적인 유지보수 부담은 크게 줄어든다.

---

### 4. 모니터링과 운영 기능 제공

RDS는 CloudWatch와 연동되어 CPU 사용률, 메모리, 디스크 사용량, Connection 수, IOPS 같은 지표를 확인할 수 있다.
또한 Performance Insights를 사용하면 느린 쿼리나 병목 지점을 분석하는 데 도움을 받을 수 있다.

EC2 직접 운영에서도 모니터링은 가능하지만, DB 지표 수집 도구와 로그 수집 체계를 별도로 구성해야 한다.
운영 초기에는 이 구성이 누락되기 쉽고, 장애가 발생한 뒤에야 필요한 지표가 없다는 사실을 알게 되는 경우도 많다.

RDS는 기본적인 운영 가시성을 빠르게 확보할 수 있다는 장점이 있다.

---

### 5. 확장성

RDS는 인스턴스 타입 변경, 스토리지 확장, Read Replica 생성 등을 통해 확장성을 제공한다.
읽기 트래픽이 많은 서비스에서는 Read Replica를 만들어 조회 부하를 분산할 수 있다.

EC2에서 직접 운영하는 DB도 확장이 불가능한 것은 아니다.
하지만 Replication 설정, 서버 증설, 스토리지 확장, 장애 처리까지 직접 구성해야 하므로 난이도가 높다.

RDS는 확장 작업의 많은 부분을 관리형 기능으로 제공해 운영 복잡도를 낮춘다.

---

## 2. RDS와 EC2 직접 운영 DB의 차이점

### 관리 책임의 차이

가장 큰 차이는 **책임 범위**다.

RDS를 사용하면 AWS가 인프라와 DB 운영의 상당 부분을 관리한다.
운영자는 스키마 설계, 쿼리 튜닝, 인덱스 설계, 파라미터 설정, 보안 그룹, 접근 제어, 비용 최적화에 더 집중할 수 있다.

반면 EC2에 직접 DB를 설치하면 운영자가 거의 모든 것을 책임진다.
OS 설치, DB 설치, 백업, 패치, 장애 복구, 모니터링, 보안 설정, 스토리지 관리까지 직접 다뤄야 한다.

---

### 비교 정리

| 항목 | RDS | EC2 직접 운영 DB |
| --- | --- | --- |
| 설치와 구성 | 콘솔/CLI로 빠르게 생성 | OS와 DB 직접 설치 |
| 백업 | 자동 백업, Snapshot 지원 | 직접 스크립트와 저장소 구성 |
| 장애 복구 | Multi-AZ, 자동 Failover 지원 | 직접 Replication과 Failover 구성 |
| 패치 | Maintenance Window 기반 관리 | OS/DB 패치 직접 수행 |
| 모니터링 | CloudWatch, Performance Insights 연동 | 별도 모니터링 도구 구성 필요 |
| 커스터마이징 | 제한적 | OS와 DB 레벨까지 자유롭게 제어 가능 |
| 운영 난이도 | 상대적으로 낮음 | 상대적으로 높음 |
| 비용 구조 | 관리형 서비스 비용 포함 | 인프라 비용은 낮을 수 있으나 운영 비용 증가 |

---

## 3. RDS가 적합하지 않을 수 있는 상황

RDS는 많은 경우 좋은 선택이지만 항상 정답은 아니다.
관리형 서비스이기 때문에 편리함을 얻는 대신 일부 제어권을 포기해야 한다.

### 1. OS 또는 DB 내부 설정을 깊게 제어해야 하는 경우

RDS에서는 DB 서버의 OS에 직접 접근할 수 없다.
특정 OS 패키지 설치, 파일시스템 레벨 튜닝, DB 엔진 내부 파일 직접 수정이 필요한 경우에는 RDS가 적합하지 않을 수 있다.

이런 경우 EC2에 직접 DB를 설치하면 더 높은 수준의 커스터마이징이 가능하다.

---

### 2. 특수한 DB Extension이나 설정이 필요한 경우

RDS는 지원하는 Extension과 설정 범위가 제한될 수 있다.
예를 들어 PostgreSQL의 특정 Extension, 비표준 플러그인, 커스텀 엔진 설정이 필요한 경우에는 RDS에서 지원 여부를 반드시 확인해야 한다.

지원되지 않는 기능이 핵심 요구사항이라면 EC2 직접 운영이나 다른 DB 서비스를 고려해야 한다.

---

### 3. 비용 최적화가 매우 중요한 소규모 서비스

작은 토이 프로젝트나 학습용 서비스에서는 RDS 비용이 부담될 수 있다.
특히 사용량이 적은데도 상시 실행되는 DB 인스턴스 비용이 발생한다.

이 경우에는 EC2 한 대에 애플리케이션과 DB를 함께 두거나, 더 저렴한 DB 옵션을 선택하는 것이 현실적일 수 있다.
다만 운영 서비스라면 백업, 보안, 장애 복구 비용까지 함께 고려해야 한다.

---

### 4. 벤더 종속성을 피해야 하는 경우

RDS는 AWS 생태계와 강하게 통합되어 있다.
CloudWatch, IAM, Security Group, Snapshot, Multi-AZ 같은 기능은 편리하지만 AWS 환경에 묶이는 측면도 있다.

멀티 클라우드 전략이 중요하거나 특정 클라우드에 종속되지 않아야 하는 조직이라면,
직접 운영 방식이나 Kubernetes 기반 DB 운영 방식을 검토할 수 있다.

---

## 4. GitHub Actions Trigger란?

GitHub Actions에서 `Trigger`는 Workflow를 실행시키는 이벤트 조건이다.
즉, "언제 CI/CD 파이프라인을 실행할 것인가"를 정하는 설정이다.

Workflow 파일은 보통 `.github/workflows/*.yml` 경로에 작성하고,
`on` 키워드를 사용해 Trigger를 정의한다.

```yaml
name: CI

on:
  push:
    branches:
      - main
```

위 설정은 `main` 브랜치에 push가 발생했을 때 Workflow를 실행한다.

---

## 5. GitHub Actions의 주요 Trigger 유형과 CI/CD 시나리오

### 1. push

`push` Trigger는 브랜치나 태그에 커밋이 push될 때 실행된다.
가장 일반적인 CI Trigger 중 하나다.

```yaml
on:
  push:
    branches:
      - main
      - develop
```

적합한 시나리오는 다음과 같다.

- `develop` 브랜치에 push될 때 테스트 실행
- `main` 브랜치에 merge된 후 배포 파이프라인 실행
- 특정 태그가 push될 때 Release Workflow 실행

예를 들어 `main` 브랜치에 push될 때 Docker Image를 빌드하고 운영 서버에 배포하는 방식으로 사용할 수 있다.

---

### 2. pull_request

`pull_request` Trigger는 Pull Request가 열리거나 업데이트될 때 실행된다.
코드가 main 브랜치에 병합되기 전에 품질을 검증하는 데 적합하다.

```yaml
on:
  pull_request:
    branches:
      - main
```

적합한 시나리오는 다음과 같다.

- PR 생성 시 Gradle Test 실행
- Checkstyle, SpotBugs 같은 정적 분석 실행
- 테스트가 통과해야 Merge 가능하도록 Branch Protection과 연동

Spring Boot 프로젝트라면 PR마다 `./gradlew test`를 실행해 서비스 로직 변경이 기존 기능을 깨뜨리지 않았는지 확인할 수 있다.

---

### 3. workflow_dispatch

`workflow_dispatch`는 사용자가 GitHub UI에서 직접 Workflow를 실행할 수 있게 하는 수동 Trigger다.
입력값도 받을 수 있어 운영 배포나 특정 환경 배포에 자주 사용된다.

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: "Deploy environment"
        required: true
        default: "dev"
```

적합한 시나리오는 다음과 같다.

- 운영 배포를 사람이 승인한 뒤 수동 실행
- 특정 환경(dev, staging, prod)을 선택해 배포
- 긴급 Rollback Workflow 실행

자동 배포가 부담스러운 운영 환경에서는 `workflow_dispatch`를 사용해 배포 타이밍을 사람이 통제할 수 있다.

---

### 4. schedule

`schedule` Trigger는 Cron 표현식을 사용해 정해진 시간에 Workflow를 실행한다.

```yaml
on:
  schedule:
    - cron: "0 18 * * *"
```

GitHub Actions의 Cron은 UTC 기준으로 동작한다.
따라서 한국 시간으로 매일 새벽 3시에 실행하려면 UTC 기준 전날 18시로 설정해야 한다.

적합한 시나리오는 다음과 같다.

- 매일 새벽 정기 배치 실행
- 주기적인 의존성 취약점 검사
- 매주 테스트 리포트 생성
- 오래된 Docker Image나 Artifact 정리

정기적으로 반복되어야 하지만 코드 변경과 직접 관련 없는 작업에 적합하다.

---

### 5. release

`release` Trigger는 GitHub Release가 생성, 수정, 게시될 때 실행된다.
버전 릴리즈와 배포를 연결할 때 사용하기 좋다.

```yaml
on:
  release:
    types:
      - published
```

적합한 시나리오는 다음과 같다.

- Release가 published 되었을 때 운영 배포 실행
- 릴리즈 노트 생성 후 Artifact 업로드
- 버전 태그 기준 Docker Image 빌드

서비스 버전을 명확히 관리하고 싶을 때 Release Trigger를 사용하면 배포 이력을 추적하기 쉽다.

---

### 6. repository_dispatch

`repository_dispatch`는 GitHub 외부 시스템에서 API를 호출해 Workflow를 실행할 수 있게 하는 Trigger다.

```yaml
on:
  repository_dispatch:
    types:
      - deploy-from-external-system
```

적합한 시나리오는 다음과 같다.

- 외부 배포 시스템에서 GitHub Actions 실행
- 다른 저장소의 빌드 완료 후 현재 저장소 배포 실행
- Slack Bot이나 운영 도구에서 특정 Workflow 실행

마이크로서비스 구조에서 여러 저장소 간 파이프라인을 연결할 때 유용하다.

---

### 7. workflow_run

`workflow_run` Trigger는 다른 Workflow가 완료된 뒤 후속 Workflow를 실행할 때 사용한다.

```yaml
on:
  workflow_run:
    workflows:
      - CI
    types:
      - completed
```

적합한 시나리오는 다음과 같다.

- CI Workflow 성공 후 CD Workflow 실행
- 테스트 완료 후 배포 승인 단계로 이동
- 빌드 Workflow와 배포 Workflow를 분리해 관리

CI와 CD를 하나의 파일에 모두 넣을 수도 있지만,
프로젝트가 커지면 Workflow를 분리하고 `workflow_run`으로 연결하는 방식이 관리에 유리할 수 있다.

---

### 8. pull_request_target

`pull_request_target`은 PR과 관련된 Trigger지만, 일반 `pull_request`와 다르게 base repository의 권한과 컨텍스트에서 실행된다.
따라서 Fork PR에 댓글을 달거나 Label을 붙이는 자동화에 사용할 수 있다.

하지만 보안상 주의가 필요하다.
외부 기여자의 코드를 checkout해서 실행하면 Secret이 노출될 위험이 있다.

적합한 시나리오는 다음과 같다.

- Fork PR에 자동 Label 부여
- PR 제목 또는 본문 형식 검사
- 기여자에게 안내 댓글 작성

테스트나 빌드처럼 PR의 코드를 직접 실행해야 하는 작업은 일반적으로 `pull_request`를 사용하는 것이 안전하다.

---

## 6. Trigger 선택 기준 정리

| Trigger | 적합한 시나리오 |
| --- | --- |
| push | 브랜치 push 후 빌드, 테스트, 배포 |
| pull_request | Merge 전 테스트와 코드 품질 검증 |
| workflow_dispatch | 수동 배포, 긴급 Rollback, 운영 승인 배포 |
| schedule | 정기 배치, 취약점 검사, 리포트 생성 |
| release | 버전 릴리즈 기준 배포 |
| repository_dispatch | 외부 시스템과 연동된 Workflow 실행 |
| workflow_run | CI 완료 후 CD 실행 같은 후속 파이프라인 |
| pull_request_target | Fork PR 관리 자동화, Label/Comment 처리 |

---

## 마무리

RDS는 백업, 장애 복구, 패치, 모니터링, 확장 같은 데이터베이스 운영 작업을 관리형 서비스로 제공한다.
EC2에 직접 DB를 설치하는 방식보다 운영 부담이 낮고 안정적인 구성을 빠르게 만들 수 있다는 장점이 있다.
다만 OS 레벨 제어, 특수한 DB 설정, 비용 최적화, 벤더 종속성 측면에서는 RDS가 적합하지 않을 수 있다.

GitHub Actions Trigger는 CI/CD 파이프라인의 실행 시점을 결정한다.
`push`와 `pull_request`는 일반적인 CI에 적합하고,
`workflow_dispatch`, `release`, `workflow_run`은 배포 흐름을 통제하는 데 유용하다.
또한 `schedule`과 `repository_dispatch`는 코드 변경이 아닌 시간이나 외부 시스템 이벤트를 기준으로 자동화를 구성할 때 활용할 수 있다.

결국 좋은 운영 환경을 만들기 위해서는 인프라와 자동화를 따로 보는 것이 아니라,
**데이터베이스는 안정적으로 운영하고, 배포 과정은 예측 가능하게 자동화하는 것**이 중요하다.

