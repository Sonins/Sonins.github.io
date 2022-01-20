---
layout: post
title:  "Kubernetes Basic 4 : Troubleshooting"
categories: Kubernetes
---
# Trouble Shooting

Kubernetes 클러스터에 문제가 생겼을 때 원인을 찾고 문제를 해결할 방법을 찾는 방법에 대한 항목입니다.

## Log

클러스터에 문제가 생겼을 때는 로그를 봐야합니다. [공식 문서](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-cluster/)에도 나와있듯, Kubernetes 구성 요소들의 로그가 저장되어 있는 위치는 정해져 있습니다.

### Kubelet

Kubelet은 모든 노드에서 실행되는 요소로써, Kubernetes를 통해 같은 노드에 생성된 모든 컨테이너들을 관리합니다. Yaml 혹은 Json 파일로 작성된 Pod spec을 해석해 Pod를 생성하는 것도 Kubelet의 몫입니다.

Kubernetes가 systemd를 사용하는 경우, 호스트 노드에서 다음 명령어로 Log를 열람할 수 있습니다.

```bash
sudo journalctl -u kubelet
```

사용하지 않는 경우, `/var/log/kubelet.log` 에서 로그를 찾을 수 있습니다.

### Kube-apiserver

Kube-apiserver는 Control plane에서 작동하고 있는 요소로, kube-apiserver는 클러스터의 state를 검색하는데 프론트엔드 부분을 제공합니다. 다시 말해, kube-apiserver는 REST 요청을 받아서 클러스터의 저장된 상태를 반환합니다.

kube-apiserver도 Pod로 작동하므로, 다음 명령어로 Log를 볼 수 있습니다.

```bash
kubectl logs kube-apiserver-name
```

또한, `var/log/kube-apiserver.log`에도 로그가 저장되어 있습니다.

### etcd

kube-apiserver가 클러스터의 state를 검색하는 프론트엔드를 제공한다면, etcd는 실제로 클러스터의 state를 저장하는 백엔드를 제공합니다. 클러스터의 설정과 여러 상태를 키-값 쌍으로 저장합니다.

etcd는 배포 형태가 다양합니다. Static Pod로 배포되었을 때는,

```bash
kubectl logs etcd-name
```

그리고 컨테이너로 배포되었을 때는 `/var/log/containers/<Etcd container name>` 경로에서 로그를 찾을 수 있습니다.

### Pod Log

일반적인 모든 Pod의 로그는 다음 명령어로 열람할 수 있습니다.

```bash
kubectl logs <pod name>
```

이 명령어로 종료된 Pod의 로그 또한 살펴볼 수 있습니다.

## Cases

- [Etcd healthcheck 문제](https://sonins.github.io/kubernetes/troubleshooting/etcd/2021/12/07/etcd-HDD-problem.html)
