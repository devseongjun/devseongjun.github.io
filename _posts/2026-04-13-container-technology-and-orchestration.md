---
title: "컨테이너 기술과 Docker, 그리고 컨테이너 오케스트레이션"
categories:
  - Backend
  - DevOps
tags:
  - Container
  - Docker
  - Podman
  - LXC
  - Kubernetes
  - Orchestration
  - AutoScaling
  - SelfHealing
  - DeclarativeInfrastructure
---

애플리케이션을 배포하다 보면 "내 로컬에서는 잘 되는데 서버에서는 안 된다"는 문제에 자주 맞닥뜨린다.
런타임 버전이 다르거나, 환경 변수가 빠져 있거나, 의존 라이브러리 버전이 충돌하는 경우가 대표적이다.
컨테이너 기술은 바로 이런 환경 불일치 문제를 해결하기 위해 등장한 개념이다.

그런데 많은 사람들이 "컨테이너 = Docker"라고 이해하는 경향이 있다.
Docker가 컨테이너 생태계를 대중화한 것은 사실이지만, 컨테이너 기술 자체는 Docker 이전부터 존재했던 개념이다.
이 글에서는 컨테이너 기술의 본질과 Docker의 관계를 정리하고,
나아가 컨테이너가 많아졌을 때 필연적으로 필요해지는 오케스트레이션의 개념과 필요성을 살펴본다.

---

## 1. 컨테이너 기술: Docker 이전부터 존재한 개념

### 컨테이너 기술의 본질

컨테이너(Container)는 **프로세스를 격리된 환경에서 실행하는 기술**이다.
가상 머신(VM)이 하드웨어 수준의 가상화를 통해 OS 전체를 띄우는 방식과 달리,
컨테이너는 호스트 OS의 커널을 공유하면서 프로세스 수준에서 격리를 제공한다.

이 격리를 가능하게 하는 Linux 커널의 핵심 기능은 두 가지다.

- **Namespace**: 프로세스, 네트워크, 파일시스템, 사용자 등을 독립된 공간으로 분리
- **cgroup(Control Groups)**: CPU, 메모리, 디스크 I/O 등 리소스 사용량을 제한하고 모니터링

이 두 기능은 Linux 커널 자체에 내장된 기능으로, Docker가 만들어낸 개념이 아니다.

### Docker 이전의 컨테이너 기술

컨테이너 개념은 수십 년 전부터 발전해왔다.

2000년대 초 FreeBSD에서는 `jails`라는 기술로 프로세스와 파일시스템을 격리했고,
2008년에는 Linux LXC(Linux Containers)가 등장해 Namespace와 cgroup을 활용한 리눅스 컨테이너를 구현했다.
LXC는 현재도 사용되는 컨테이너 구현체로, 특히 인프라 레벨의 격리 환경이 필요한 곳에서 쓰인다.

2013년에 등장한 Docker는 이 LXC 위에서 시작했다가 이후 자체 런타임(`libcontainer`, 현재는 `containerd`)으로 전환했다.
Docker의 혁신은 컨테이너 기술 자체를 만든 것이 아니라,
**이미지(Image) 레이어 시스템, Dockerfile을 통한 선언적 빌드, Docker Hub를 통한 이미지 공유**라는
개발자 친화적인 워크플로우를 제공한 것이다.
이것이 컨테이너를 대중화한 Docker의 진짜 가치다.

### Docker 외의 컨테이너 구현 도구들

Docker는 컨테이너 기술을 구현한 여러 도구 중 하나일 뿐이다. 대표적인 대안은 다음과 같다.

**Podman**은 Red Hat이 주도하는 Docker 대안 도구다.
Docker와 CLI 명령어가 거의 동일해 `alias docker=podman`으로 바로 대체할 수 있다.
가장 큰 차이는 데몬리스(daemonless) 아키텍처로,
Docker처럼 백그라운드 데몬 프로세스 없이 컨테이너를 실행한다.
또한 루트(root) 권한 없이도 컨테이너를 실행할 수 있어 보안 측면에서 유리하다.

**LXC / LXD**는 앞서 언급한 컨테이너 기술의 원형에 가깝다.
Docker보다 훨씬 무거운 컨테이너(컨테이너 안에 온전한 Linux 배포판을 올리는 방식)를 지향하며,
VM처럼 다양한 서비스가 함께 돌아가는 환경을 컨테이너로 구성할 때 사용한다.

**containerd**는 Docker의 내부 컨테이너 런타임으로 시작했으나 지금은 독립 프로젝트로 분리되었다.
Kubernetes의 기본 컨테이너 런타임으로 채택되어 있으며,
Docker 없이도 컨테이너를 실행할 수 있는 저수준 런타임이다.

정리하면, **컨테이너 기술은 Linux 커널의 Namespace와 cgroup을 기반으로 한 프로세스 격리 개념**이며,
Docker는 이 기술을 개발자가 쉽게 사용할 수 있도록 구현한 도구 중 하나다.

---

## 2. 컨테이너 오케스트레이션: Docker 단독으로는 부족한 이유

### 컨테이너 오케스트레이션이란

컨테이너 오케스트레이션(Container Orchestration)은
**다수의 컨테이너를 여러 호스트에 걸쳐 자동으로 배포, 관리, 확장, 복구하는 시스템**이다.

Docker 단독 환경에서는 단일 호스트에서 컨테이너를 실행하고 관리하는 것이 중심이다.
개발 환경이나 소규모 서비스에서는 이것으로 충분하다.
하지만 운영 환경에서 수십, 수백 개의 컨테이너가 여러 서버에 분산되어 실행된다면,
Docker만으로는 감당하기 어려운 문제들이 나타난다.

현재 가장 널리 사용되는 컨테이너 오케스트레이션 도구는 **Kubernetes(K8s)** 다.
그 외에도 Docker Swarm, Apache Mesos 등이 있으며,
클라우드 환경에서는 AWS ECS, GKE, AKS 같은 관리형 서비스도 있다.

### 문제 1. 자동 확장 (Auto Scaling)

**Docker 단독 환경의 한계**

Docker만 사용하면 트래픽이 급증했을 때 컨테이너를 늘리는 작업이 수동이다.
`docker run`을 반복 실행하거나 `docker-compose scale`을 사용해야 하고,
어느 서버에 컨테이너를 추가할지도 사람이 판단해야 한다.

또한 트래픽이 줄어도 불필요하게 컨테이너가 계속 실행되어 자원이 낭비된다.
이를 줄이려면 다시 수동으로 컨테이너를 제거해야 한다.

**오케스트레이션이 해결하는 방식**

Kubernetes는 **HPA(Horizontal Pod Autoscaler)** 를 통해 CPU 사용률, 메모리, 커스텀 메트릭 등을 기준으로
컨테이너(Pod) 수를 자동으로 늘리거나 줄인다.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: user-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
```

위 설정은 `user-service`의 CPU 사용률이 60%를 넘으면 자동으로 Pod를 늘리고,
사용률이 낮아지면 다시 줄이도록 한다.
최소 2개에서 최대 10개 사이에서 자동으로 조절되므로 사람이 개입할 필요가 없다.

### 문제 2. 자가 복구 (Self-Healing)

**Docker 단독 환경의 한계**

Docker만 사용할 때 컨테이너가 충돌하거나 비정상 종료되면,
`--restart=always` 옵션으로 자동 재시작을 설정할 수 있다.
하지만 이는 단순히 프로세스를 다시 시작하는 것에 불과하다.

만약 컨테이너가 실행은 되어 있지만 실제로는 요청을 처리하지 못하는 상태(예: 데드락, 무한 루프, 의존 서비스 연결 실패)라면,
Docker는 이를 감지하지 못한다.
또한 특정 호스트 서버 자체가 장애가 나면 해당 서버의 컨테이너들을 다른 서버로 이동시키는 것도 불가능하다.

**오케스트레이션이 해결하는 방식**

Kubernetes는 컨테이너의 상태를 지속적으로 감시하고,
**liveness probe**와 **readiness probe**를 통해 컨테이너의 실제 건강 상태를 주기적으로 확인한다.

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
```

`livenessProbe`는 컨테이너가 살아있는지 확인하고, 실패 시 컨테이너를 재시작한다.
`readinessProbe`는 컨테이너가 요청을 받을 준비가 되었는지 확인하고,
준비가 안 된 경우 로드 밸런서에서 해당 컨테이너를 제외한다.

Spring Boot에서는 `spring-boot-starter-actuator`를 추가하면
`/actuator/health` 엔드포인트를 통해 애플리케이션 상태를 노출할 수 있어 이와 자연스럽게 연동된다.

또한 노드(서버) 자체가 다운되면 Kubernetes는 해당 노드의 컨테이너들을
정상 노드로 자동으로 재스케줄링한다. Docker 단독으로는 불가능한 동작이다.

### 문제 3. 선언적 인프라 (Declarative Infrastructure)

**Docker 단독 환경의 한계**

Docker 단독 환경에서는 컨테이너를 시작하고 관리하는 방식이 대부분 명령형(imperative)이다.
"이 컨테이너를 실행해라", "이 컨테이너를 중지해라"처럼 각 단계를 직접 지시하는 방식이다.

`docker-compose`로 어느 정도 선언적 구성이 가능하지만,
이것은 단일 호스트에 한정된 도구다.
여러 서버에 걸쳐 배포하거나, 버전을 롤링 업데이트하거나, 이전 버전으로 롤백하는 것은
별도의 스크립트나 수동 작업이 필요하다.

**오케스트레이션이 해결하는 방식**

Kubernetes는 **"현재 상태"가 아닌 "원하는 상태(desired state)"를 YAML로 선언**하는 방식을 사용한다.
사용자가 "컨테이너 3개를 실행한 상태"를 선언하면, Kubernetes가 현재 상태와 비교해 알아서 맞춰준다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
        - name: user-service
          image: user-service:1.2.0
          ports:
            - containerPort: 8080
```

이 YAML 파일을 적용하면 Kubernetes는 `user-service` 컨테이너 3개가 항상 실행되는 상태를 유지한다.
이미지를 `1.2.0`으로 변경해 다시 적용하면 롤링 업데이트가 자동으로 수행되며,
문제가 생기면 `kubectl rollout undo`로 이전 버전으로 즉시 롤백할 수 있다.

이 방식의 핵심 장점은 **인프라 설정 자체가 코드(YAML)로 관리**된다는 점이다.
Git으로 버전 관리하고, 변경 이력을 추적하고, 코드 리뷰를 통해 인프라 변경을 검토할 수 있다.
이를 **IaC(Infrastructure as Code)** 라고 부른다.

---

## 정리

컨테이너 기술은 Linux 커널의 Namespace와 cgroup을 기반으로 한 개념으로, Docker 이전부터 존재했다.
Docker는 이 기술 위에 개발자 친화적인 인터페이스와 이미지 생태계를 더해 컨테이너를 대중화한 도구이며,
Podman, LXC, containerd 등 다양한 대안 구현체도 존재한다.

컨테이너 수가 늘어나고 운영 환경이 복잡해지면 Docker 단독으로는 자동 확장, 자가 복구, 선언적 인프라 관리가 어렵다.
컨테이너 오케스트레이션은 이 세 가지 문제를 체계적으로 해결한다.
- **자동 확장**: 메트릭 기반으로 컨테이너 수를 자동으로 조절
- **자가 복구**: Probe를 통해 비정상 컨테이너를 감지하고 재시작하거나 트래픽에서 제외
- **선언적 인프라**: 원하는 상태를 코드로 선언하고 Kubernetes가 그 상태를 유지

결국 Docker는 컨테이너를 "만들고 실행하는 도구"이고,
Kubernetes와 같은 오케스트레이션 도구는 그 컨테이너들을 "운영 환경에서 안정적으로 관리하는 도구"다.
두 기술은 대체 관계가 아니라 **서로 다른 역할을 담당하는 상호 보완적인 도구**라고 이해하는 것이 중요하다.

---
