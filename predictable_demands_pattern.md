

# 1. 예측 범위 내의 요구사항
> Predictable Demands pattern, *다음은 OREILLY의 쿠버네티스 패턴 2장을 요약한 내용입니다.*

<!-- TOC -->

- [1. 예측 범위 내의 요구사항](#1-예측-범위-내의-요구사항)
  - [1.1. 문제](#11-문제)
  - [1.2. 해결](#12-해결)
  - [1.3. 런타임 의존성](#13-런타임-의존성)
    - [1.3.1. 파일 스토리지](#131-파일-스토리지)
    - [1.3.2. 포트](#132-포트)
    - [1.3.3. 설정(Configuration)](#133-설정configuration)
  - [1.4. 자원 프로파일 (QoS)](#14-자원-프로파일-qos)
  - [1.5. 파드 우선순위 (Priority)](#15-파드-우선순위-priority)
  - [1.6. QoS vs Pod priority](#16-qos-vs-pod-priority)
  - [1.7. 프로젝트 자원](#17-프로젝트-자원)

<!-- /TOC -->


`Predictable Demands pattern`은 `hard runtime`(물리적으로 필요한 런타임 환경) 의존성이나리소스 요구사항과 상관 없이, 애플리케이션 요구사항을 선언하는 방법이다. 해당 요구사항 선언은 **쿠버네티스가 클러스터 내에서 애플리케이션에 적합한 노드를 찾기 위해 반드시 필요하다.**

## 1.1. 문제

다양한 언어/프레임워크들로 생성된 애플리케이션을 컨테이너를 사용해 관리하게 되면 **컨테이너가 최적의 기능을 수행하는 데 필요한 자원량을 예측하기 어려워진다.** 

특정 자원량 외에도 애플리케이션 별로 특정 포트 번호를 사용해야 작동하는 서비스들도 존재한다.

이 처럼 `애플리케이션 런타임`은 리소스 요구사항, 데이터 스토리지, 애플리케이션 설정 같은 플랫폼 관리 기능이 필요하다.

## 1.2. 해결
> 해결: 컨테이너 런타임 요구사항을 미리 알려준다.

1. 효율적인 컨테이너 지능적인 배치(placement)
   - 모든 런타임 의존성과 자원 요구량, 우선순위가 미리 계산되면, 쿠버네티스는 클러스터 내에 컨테이너 실행 위치를 효과적으로 배치 가능하다.
1. 컨테이너 `자원 프로파일`이 제공 되면 용량 계획이 가능.
   - 성공적인 클러스터 관리를 위해선 서비스 자원 프로파일과 용량 계획을 장기적으로 함꼐 진행해야 함. (런타임 의존성과 관련)

## 1.3. 런타임 의존성

- `tl;dr`
  - 스토리지와 포트 번호 의존성은 파드가 스케줄링되는 위치를 제한한다.
  - 컨피그맵/시크릿 의존성은 파드가 시작하는 부분까지도 막을 수 있다.
  
### 1.3.1. 파일 스토리지
> 애플리케이션 상태를 저장하는 데 사용
1. emptyDir
   - 가장 간단한 볼륨 타입 
   - 일시적이며 파드가 종료되면 삭제

2. persistentVolumeClaim
   - 파드 레벨 스토리지
   - 파드 restart 후에도 데이터 저장


`스케줄러`는 파드가 요청한 볼륨 종류룰 판단하며, 만약 클러스터의 워커노드가 제공하지 않는 볼륨을 파드가 필요로 하면 **파드는 결코 스케줄링 되지 않는다.**

### 1.3.2. 포트
> 호스트 시스템의 특정 포트로 컨테이너 포트 노출을 요청

1. `hostProt`
   - 클러스터에서 각 노드에 해당 포트를 예약하고 노드 하나당 최대 하나의 파드만 스케줄링 되게 제한 한다.
   - 포트 충돌 때문에 **쿠버네티스 클러스터 노드 수만큼만 파드 확장 가능**

### 1.3.3. 설정(Configuration)

1. `ConfigMap` / `Secret`
   - 요청한 모든 컨피그맵이 생성되지 않으면 컨테이너는 노드에 스케줄링 될 수 있지만, 시작되지 않도록 막을 수도 있다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: random-generator-config
data:
  pattern: Predictable Demands
---
apiVersion: v1
kind: Pod
metadata:
  name: random-generator
spec:
  containers:
  - image: k8spatterns/random-generator:1.0
    name: random-generator
    env:
    - name: PATTERN
      valueFrom:
        # First Hard requirement for a config map to exist.
        configMapKeyRef:
          name: random-generator-config
          key: pattern
```


## 1.4. 자원 프로파일 (QoS)
> 쿠버네티스 컨텍스 내에서 compute resource라고 함은 컨테이너에 의해 요청/할당/소비 되는 무언가이다.


- 종류
  1. 압축 가능 자원(compressible resource)
     - cpu, 네트워크 대역폭 처럼 제어 가능
     - **너무 많이 소비할 경우, 병목 현상**
  2. 압축 불가능 자원(incompressible resource)
     - 메모리 처럼 제어 불가능
     - 너무 많이 소비하면 컨테이너 kill(애플리케이션에 할당된 메모리 해제 요청할 방법이 없기 때문)

쿠버네티스는 최소 자원량(request), 최대 자원량(limits)를 통하여 cpu/ 메모리 양을 지정할 수 있다. (linux의 soft/hard와 유사)

특히 reuqests는 **스케줄러가 파드를 노드에 배치시킬 때 사용**되며, 스케줄러는 해당 파드와 파드 안의 모든 컨테이너 요청 자원량을 합산해 충분히 수용 가능한 노드들만 고려한다.


```yaml

# DeploymentConfig for starting up the random-generator
apiVersion: apps/v1
kind: Deployment
metadata:
  name: random-generator
spec:
  replicas: 1
  selector:
    matchLabels:
      app: random-generator
  template:
    metadata:
      labels:
        app: random-generator
    spec:
      containers:
      - image: k8spatterns/random-generator:1.0
        name: random-generator
        ports:
        - containerPort: 8080
          protocol: TCP
        resources:
          # Initial resource request for CPU and memory
          requests:
            cpu: 100m
            memory: 100Mi
          # Upper limit until we want our application to grow at max
          limits:
            cpu: 200m
            memory: 200Mi
```
requests나 limits을 통해서 다음과 같은 여러 종류의 서비스 품질(QoS)를 지원한다.

1. 최선적 파드(Best Effort)
   - requests나 limits을 포함하지 않는다.
   - 가장 낮은 우선순위
   - `incompressible resource` 고갈 시 가장 먼저 죽는다.
2. 확장 가능 파드(Burstable)
   - requests와 limits의 값이 다르다. (보통 requests < limits)
   - 지정한 범위 만큼 자원 소비
   - `incompressible resource` 압박이 있을경우, Best Effort가 없다면 죽을 확률이 높다.
3. 보장된 파드(Guaranteed)
   - request = limits 동일하게 지정
   - 가장 우선순위가 높은 파드
   - 셋 중 가장 나중에 죽는다.

## 1.5. 파드 우선순위 (Priority)

- `PriorityClass`
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "매우 높은 우선순위의 pod Class"
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
  priorityClassName: high-priority # 자원에 적용될 우선순위 클래스
```

파드 우선순위 기능이 활성화되면 스케줄러가 파드를 노드에 배치하는 순서에 영향을 준다.

**파드를 배치하기에 충분한 용량을 가진 노드가 하나도 없다면 스케줄러는 자원을 확보하고 우선순위가 높은 파드를 배치하기 위해 노드에서 실행되고 있는 우선순위가 낮은 파드를 제거한다.**

## 1.6. QoS vs Pod priority
> Pod QoS와 Pod priority 서로 연관되지 않고, orthogonal(직교)하는 기능이다.

- QoS(서비스 품질)는 사용 가능한 컴퓨팅 자원이 낮을 때 노드의 안정성을 유지하기 위해 kubelet에 의해 주로 사용된다. kubelet은 파드를 축출(eviction, *다른 노드로 파드를 옮기기 위해 현재 노드에 있는 파드를 삭제하는 것*)하기 전에 1) QoS 2) PriorityClass를 순서로 판단한다. 
- 반면 스케줄러 eviction 로직은 선점 대상 선택 시, Priority만을 고려한다.

## 1.7. 프로젝트 자원
- 관련 키워드
  - ResourceQuota: 네임스페이스별 컴퓨팅 자원 제한
  - LimitRange: 최소/최대 자원량 설정
  - overcommit level: request/limit 비율 제어

requests와 limit차이가 크면 노드에 오버커밋할 가능성이 크고, 많은 컨테이너가 처음 요청 값보다 더 많은 자원을 동시에 필요로 할 경우 애플리케이션 성능이 낮아질 수 있다.