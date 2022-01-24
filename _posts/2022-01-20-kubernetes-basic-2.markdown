---
layout: post
title:  "Kubernetes Basic 2 : Kubernetes 객체"
categories: Kubernetes
---

# Kubernetes 객체

이 포스트에서는 Kubernetes 기본 객체들을 간단하게 소개합니다. 더 상세한 정보는 [Kubernetes 공식 문서 : Concept](https://kubernetes.io/ko/docs/concepts/)에서 제공하고 있습니다.

## Pod

![Pod](../assets/images/module_03_pods.svg)
Kubernetes의 가장 기본적인 워크로드입니다. Application이 Pod 형태로 Kubernetes 클러스터에 배포됩니다. Pod는 다음과 같은 특징들을 가집니다.

- Pod는 하나 이상의 컨테이너로 구성되어 있습니다.
- Pod는 컨테이너 수와 상관없이 하나의 IP를 가집니다.
- Kubernetes는 Pod가 오류로 인해 실패하면, 자동으로 Pod를 재생성합니다.  
- Pod는 영속성이 보장되지 않습니다. Pod에서 일어난 연산 결과는 영구적이지 않고, Pod의 IP 또한 Pod가 재시작할 때 마다 바뀝니다.

```yaml
# from https://kubernetes.io/ko/docs/concepts/workloads/pods/
# nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

위 Pod를 `kubectl apply -f nginx.yaml`로 생성하면 도커 이미지 `nginx:1.14.2`를 pull해서, 컨테이너로 배포합니다. `nginx:1.14.2`의 컨테이너는 Pod가 배포될 때 Pod의 일부가 됩니다.

### Reference

- [Kubernetes 공식 문서 : Pod](https://kubernetes.io/ko/docs/concepts/workloads/pods/)

## Service

![Pod temp IP](../assets/images/podiptemp.png)

Pod의 IP는 영구적이지 않고 재시작할 때마다 바뀌므로, IP를 기반으로 Pod에 접근하면 연결을 오래 유지할 수 없습니다.

![service](../assets/images/service.png)

이러한 문제를 해결하기 위해, Kubernetes는 IP에 상관없이 특정 Pod에 접근할 수 있도록 도메인 네임 주소를 제공하는 Service라는 객체를 제공합니다. 이 Service는 Pod 하나에만 접합하는 건 아니고, Pod 집합에 접합할 수도 있습니다. 이 경우 Service는 접합된 여러 개의 Pod 중 하나만을 선택해서 연결합니다.

이렇게 여러 개의 서버 중 하나를 선택해서 클라이언트와 연결해주는 것을 Reverse proxy라고 합니다. 여러 개의 서버를 추상화해 마치 하나의 서버와 연결하는 것처럼 연결을 반환합니다. 이 경우 Load balancing 등의 효과를 기대할 수 있습니다.

Service는 종류도 여러가지고, Kubernetes의 경우 네트워크 연결을 기본으로 클러스터를 관리하니 상세 내용이 복잡합니다. 상세 내용을 위해 읽어볼만한 문서들은 다음과 같습니다.

### Service host name

Kubernetes 내부에서 특정 서비스에 접근하기 위해서 사용해야 하는 호스트 네임(주소)는 다음과 같습니다.

```text
{service-name}.{namespace}.svc.cluster.local
```

아래 Yaml 파일로 생성된 서비스의 호스트 네임은 `my-service.my-namespace.svc.cluster.local`이 됩니다. 주소 `my-service.my-namespace.svc.cluster.local`로 접근하면, `my-service`에 접합된 Pod 집합에 연결할 수 있게 됩니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: my-namespace
spec:
  selector:
    app: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

이러한 서비스에 접합된 특정 단일 Pod `my-pod-0`에 접근할 때, 해당 Pod는 `my-pod-0.my-service.my-namespace.svc.cluster.local`이라는 이름을 가집니다. 이때 해당 Pod의 이름이 `my-pod-0`여야 합니다. (`metadata.name`)

### Reference

- [Kubernetes 공식 문서 : Service](https://kubernetes.io/ko/docs/concepts/services-networking/service/)
  - 서비스의 종류들과 그 작동 원리에 대해 설명하고 있는 문서입니다. 여러가지 시나리오에 대해 그에 맞는 서비스 종류를 제안하는데, 처음에는 서비스를 어떻게 생성하는지와, 서비스의 유형인 [ClusterIP/LoadBalancer/Nodeport의 차이](https://kubernetes.io/ko/docs/concepts/services-networking/service/#publishing-services-service-types) 먼저 봅시다.
- [Finda 기술블로그](https://medium.com/finda-tech/kubernetes-%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC-%EC%A0%95%EB%A6%AC-fccd4fd0ae6)
  - Pod to Pod, Pod to Service등 여러 가지의 연결 종류를 비교해놓은 블로그입니다. 이를 보면 Service의 동작 원리와, Kube-proxy가 이때 어떤 역할을 하는건지 등을 알아볼 수 있습니다.
- [Kubernetes 공식 문서 : 서비스 및 파드용 DNS](https://kubernetes.io/ko/docs/concepts/services-networking/dns-pod-service/)

## Deployment/StatefulSet

Deployment/Statefulset 모두 Pod 집합을 배포하는데 사용합니다. 일반적인 상황에서 Pod는 단일 Pod로 배포하지 않고 Deployment/Statefulset으로 배포합니다.

### Deployment

Deployment는 다수의 Pod를 배포하는 것에 집중합니다. Deployment로 어플리케이션을 배포하는 목적은 부하 분산과 버전 rollout 업데이트 등을 사용하는 것입니다.   예를 들어, Nginx 웹 서버를 Deployment로 배포한다면, 그 목적 중 하나는 Nginx 웹 서버의 인스턴스 수를 늘려서, 트래픽을 분산하기 위해서입니다.  

단일 Pod로 배포할 때도 Deployment를 사용하는 것은 여러 이점을 가집니다. Rollout Update를 이용해 어플리케이션 버전 업데이트를 수행할 때 Pod의 수를 늘려 무중단 배포를 지원하는 것은 그러한 이점 중 하나입니다.

### StatefulSet

#### Stateless

Kubernetes로 배포되는 모든 어플리케이션은 Stateless입니다. 일반적인 컨테이너가 그러하듯, Pod 또한 영속성이 보장되지 않습니다. Kubernetes는 Pod 하나를 영속성을 보장하기 위해 종료없이 운영하는 대신, 오류가 발생할 때마다 Pod를 재시작합니다. 재시작할 때마다 이전의 종료된 Pod의 연산 결과는 모두 사라집니다. 컨테이너안 데이터가 컨테이너가 종료되면 사라지는 것과 같습니다.

예를 들어, Mysql Pod를 볼륨 없이 배포했다고 생각해봅시다. 이렇게 배포된 Mysql은 쿼리 요청을 받아서 데이터를 생성하고 수정할 수 있지만, Pod가 종료됨과 동시에 DB안에 저장되어있던 데이터는 모두 사라질 것입니다. 오류가 날때마다 DB안의 데이터가 모두 사라진다면 정상적으로 데이터베이스를 운용할 수 없습니다. 이러한 어플리케이션은 Stateful을 보장받아야 합니다.

#### Statefulset

StatefulSet은 어플리케이션 다수의 Stateful을 관리하기 위해서 배포합니다. StatefulSet으로 배포된 Pod들은 순서 및 고유성이 보장됩니다. Stateful을 관리하기 위해서, StatefulSet은 각 Pod에 대한 Volume을 생성하기 위해 VolumeClaim 스펙을 명시해야 합니다.

```yaml
apiVersion: apps/v1
kind: StatefulSet
spec:
  selector:
    matchLabels:
      app: nginx # has to match .spec.template.metadata.labels
  serviceName: "nginx"
  replicas: 3 # by default is 1
...
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "my-storage-class"
      resources:
        requests:
          storage: 1Gi
```

위 경우처럼 말입니다. 위 nginx StatefulSet은 `volumeClaimTemplates`의 명시된 `my-storage-class`라는 이름의 storage class에 속하는 볼륨을 각 Pod에 붙입니다.
따라서 총 3개의 볼륨이 프로비저닝 될 것입니다.

### Reference

- [Kubernetes 공식 문서 : StatefulSet](https://kubernetes.io/ko/docs/concepts/workloads/controllers/statefulset/)
- [Kubernetes 공식 문서 : Deployment](https://kubernetes.io/ko/docs/concepts/workloads/controllers/deployment/)

## Secret/ConfigMap

Pod를 위한 데이터를 키-값 쌍으로 저장하는 객체입니다. ConfigMap은 데이터가 기밀이 아닐때, Secret은 기밀일때 사용합니다. 환경 변수, 커맨드-라인 인수 혹은 구성 파일을 설정할 때 주로 쓰입니다.
### Reference

- [Kubernetes 공식 문서 : ConfigMap](https://kubernetes.io/ko/docs/concepts/configuration/configmap/)
- [Kubernetes 공식 문서 : Secret](https://kubernetes.io/ko/docs/concepts/configuration/secret/)

## PersistentVolume/PersistentVolumeClaim

Pod로 배포된 모든 Container의 데이터는 일시적입니다. Pod가 종료됨과 동시에 사라집니다. Volume은 Pod의 종료 여부와 상관없이 데이터를 계속해서 유지합니다. Database 어플리케이션을 배포할 때를 예로 들면, 데이터베이스가 데이터를 저장할 위치를 PersistentVolume내로 지정하는 것이 합리적일 것입니다. (Pod가 종료/재시작 되어도 유지되니까요.)

### PersistentVolume

PersistentVolume은 실제로 공간이 할당된 저장공간 객체입니다. Docker도 볼륨이라는 객체를 지원하고 있지만, Kubernetes의 PersistentVolume에 비해 그 기능이 제한적입니다.

### PersistentVolumeClaim

PersistentVolume이 실제 저장 공간이라면, PersistentVolumeClaim은 그 공간을 사용할 수 있다는 점유권입니다. PersistentVolume이 집이라면, PersistentVolumeClaim은 집문서가 되는 것입니다. Pod등 워크로드는 PersistentVolumeClaim을 통해 PersistentVolume을 사용합니다.  

### Provisioning

워크로드가 PersistentVolumeClaim을 사용할 때, PersistentVolumeClaim에 맞게 PersistentVolume이 할당 되는 것을 Provisioning이라고 합니다. 할당 방식에 따라 동적 Provisioning과 정적 Proivisoning으로 나뉩니다.

#### Static Provisioning

정적 Provisioning은 매니페스트 파일이나 명령어 등으로 미리 생성한 PersistentVolume을 PersistentVolumeClaim이 요구하는 접근 모드와 크기에 맞게 할당합니다.

#### Dynamic Provisioning

동적 Provisioning은 Volume을 요구하는 워크로드가 PersistentVolumeClaim을 사용할 때 자동으로 PersistentVolume을 생성합니다.

### StorageClass

스토리지 유형을 표현할 때 StorageClass라는 Kubernetes 객체가 사용됩니다. PV가 동적으로 Provisioning 될 때 사용하는 Provisioner, 그리고 스토리지 클래스에 속해 생성될 Volume의 유형등 여러 설정 값을 지니고 있습니다.

### Lifecycle

PersistentVolume의 라이프사이클은 다음 4단계를 거칩니다.  

- Provisioning : PV를 생성하는 단계입니다.
- Binding : 생성된 PV가 PVC에 연결되는 단계입니다.
- Using : 워크로드가 실제로 PV를 사용하는 단계입니다.
- Reclaiming : 사용이 끝나고, PVC가 삭제될 때 남아있는 PV를 처리하는 단계입니다. 이 단계에서 PV는 삭제될 수도, 다른 PVC에 재사용 될 수도 있습니다.

### Reference

- [Kubernetes 공식 문서 : PersitentVolume](https://kubernetes.io/ko/docs/concepts/storage/persistent-volumes/)
- [Kubernetes 공식 문서 : 동적 Provisioning](https://kubernetes.io/ko/docs/concepts/storage/dynamic-provisioning/)
- [Kubernetes 공식 문서 : 스토리지 클래스](https://kubernetes.io/ko/docs/concepts/storage/storage-classes/)
