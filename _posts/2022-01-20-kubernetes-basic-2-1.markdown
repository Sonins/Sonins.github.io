---
title:  "Kubernetes Basic 2-1 : Statefulset vs Deployment"
categories: Kubernetes
---

# StatefulSet vs Deployment

Pod 집합을 배포할 때 Deployment와 StatefulSet은 구분하기 까다로울수 있습니다. 둘 다 여러 개의 Pod를 배포한다는 공통점이 있는 반면, 차이점이 무엇인지는 와닿지 않기 때문입니다. 이 둘을 구분하는 대표적인 차이점 중 하나는 StatefulSet은 Deployment와 달리 각 Pod의 고유성을 보장한다는 것입니다. 어떤 것인지 살펴봅시다.

## Pod의 이름

Deployment로 배포하면 각 Pod의 이름은 랜덤으로 결정됩니다.

```bash
NAME                                READY     STATUS    RESTARTS   AGE       LABELS
nginx-deployment-75675f5897-7ci7o   1/1       Running   0          18s       app=nginx,pod-template-hash=3123191453
nginx-deployment-75675f5897-kzszj   1/1       Running   0          18s       app=nginx,pod-template-hash=3123191453
nginx-deployment-75675f5897-qqcnn   1/1       Running   0          18s       app=nginx,pod-template-hash=3123191453
```

반면, StatefulSet으로 배포하면 각 Pod의 이름이 일정하게 정해져 있습니다.

```bash
NAME      READY     STATUS    RESTARTS   AGE
web-0     1/1       Running   0          1m
web-1     1/1       Running   0          1m
web-2     1/1       Running   0          1m
```

Deployment로 배포된 Pod는 재시작 할때마다 이름이 바꾸지만, StatefulSet의 경우에는 그렇지 않습니다. 이는 다음을 뜻합니다.

- Deployment의 Pod 하나인 `nginx-deployment-75675f5897-xxxxx`는 종료될 때 다른 Pod인 `nginx-deployment-75675f5897-yyyyy`로 대체됩니다. (Stateless)
- StatefulSet의 Pod 하나인 `web-n`은 종료될 때 대체되지 않고 똑같은 Pod를 재시작합니다. 이름이 같으므로, `web-n.my-service.my-namespace.svc.cluster.local`로 접근하면 연결은 Pod가 재시작되더라도 같은 상태의 서버에 접속할 수 있습니다. (Stateful)  
  
  [Mysql StatefulSet 배포 예](https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/#understanding-stateful-pod-initialization)를 봅시다. 분산 배포를 지원하지 않는 Mysql을 Data replication을 이용해 분산된 Pod 형식으로 배포하고 있습니다. Pod `mysql-N+1`은 `mysql-N`에서 데이터를 복제해오는 방식입니다. 이러한 동작 방식을 지원하기 위해 StatefulSet은 일관된 Pod 이름을 지원합니다. 이를 이용해서 어플리케이션을 배포할 때, 각 Pod가 의도된 순서로 배포될 수 있게 할 수 있습니다.  
  
  또 다른 예를 살펴봅시다. 어떤 어플리케이션은 Master-Slave 구조를 가집니다. 이러한 어플리케이션을 배포할 때, StatefulSet으로 배포하여 `pod-0`는 master로, `pod-1`, `pod-2`는 slave로 지정할 수 있습니다. 이름이 바뀌지 않으므로, `pod-0`는 재시작 되더라도, 특별한 master 교체가 없다면 계속 master로 동작하게 됩니다. 다만 이는 어플리케이션 레벨에서 해당 동작이 구현되어 있어야 합니다.

## Volume

Deployment의 경우 생성 시 [PVC](###PersistentVolumeClaim)가 의무적이지 않습니다. 반면에 StatefulSet의 경우 의무적으로 `volumeClaimTemplate`으로 PVC를 지정해줘야 합니다.

- Deployment의 경우 각 Pod가 재시작시 다른 Pod로 대체되므로 볼륨 접합이 의무적이지 않습니다.
- StatefulSet의 경우 각 Pod가 재시작하더라도 상태를 유지해야하므로 Persistent Volume을 의무적으로 Pod당 하나씩 접합해야 합니다.