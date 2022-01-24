---
layout: post
title: "Kubernetes Basic 0 : Kubernetes 구성 도구"
categories: Kubernetes
---

# Intro

Kubernetes Basic 포스트 시리즈는 Kubernetes/Infra에서 필요한 가장 기본적인 개념들만 간략하게 소개합니다. 실제로는 더 많은 내용들이 있고, 여기에 소개된 것은 극히 일부임을 밝힙니다. 이 문서의 목적은 복잡한 의사 결정을 최대한 줄이고 빠르게 Kubernetes를 사용하고 또 필요한 개념들을 소개하는데 있습니다. 더 상세한 내용은 인터넷에 공개된 공식 문서 혹은 각 개념에 대해 자세히 설명한 글들을 참조하는 것이 좋고, 이 시리즈에서도 그러한 글들의 주소들을 첨부하고 있습니다.

# Kubernetes Cluster 구성

이 챕터에서는 Kubernetes 클러스터를 맨 처음 구성할 때 어떠한 방식을 사용하고, 또 어떤 솔루션을 사용하고 있는지 소개합니다. 클러스터가 이미 구성되어 있더라도, 어떠한 방식으로 클러스터를 구성할 수 있는지, 또한 그 과정에서 어떤 솔루션을 채택하는지를 알아두면 도움이 될 것 같습니다.  

## Kubernetes 구성 도구

베어메탈 리눅스 서버가 여러 개 주어져 있는 상황에서, 어떤 방식으로 Kubernetes 클러스터를 구성할 수 있는지 소개합니다.

### Kubespray/Ansible

![Kubespray](../assets/images/kubespray.svg)

Kubeadm, Kops등 Kubernetes 클러스터를 구성하는 여러 방법이 있지만 Kubespray/Ansible playbook을 사용해서도 베어메탈 서버에 Kubernetes를 설치할 수 있습니다.

Kubespray/Ansible은 최초 클러스터 구성뿐 만 아니라, 노드가 추가될 때 마다 클러스터에 추가할 수도 있고, (이를 스케일링이라고 합니다.) 클러스터의 Kubernetes 버전 또한 Graceful 하게 업그레이드 할 수 있습니다.

상세한 구성 과정은 해당 문서에 설명하기에 너무 길고 복잡해 다른 글의 링크를 참조합니다.

- [Kubernetes 공식 문서 : Kubespray로 Kubernetes 설치하기](https://kubernetes.io/ko/docs/setup/production-environment/tools/kubespray/)
- [개인 블로그 : Kubespray 이용하여 Kubernetes 설치하기](https://www.whatwant.com/entry/Kubespray)

DNS, 컴포넌트 런타임 옵션등 여러가지를 선택할 수 있지만 클러스터를 처음 설치하는 경우는 먼저 기본값을 이용해서 설치해봅시다. 선택 항목 중의 하나인 CNI는 [CNI 플러그인](##CNI) 항목을 참고합니다.

### Kind

![kind](../assets/images/kind.png)

프로덕션 환경에서 Kubernetes를 설치하는 것도 중요하지만, 로컬 환경(대부분의 경우 본인의 랩탑/맥북일 것입니다.)에서 테스트, 혹은 학습을 위해 로컬 머신을 이용한 단일 클러스터/혹은 여러 가상머신으로 이루어진 클러스터를 구성하는 방법도 필요합니다. 이 경우 선택할 수 있는 좋은 솔루션 중에 Kind가 있습니다.  

[Kind installation](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)을 참고해서 Kind를 설치합니다.

설치 이후, 간단한 명령어로 Kubernetes 단일 클러스터 환경을 로컬에 빠르게 생성할 수 있습니다.

```bash
kind create cluster  # Default cluster context name is 'kind'
kind create cluster --name kind-2
```

또한, 다음과 같은 설정 파일을 이용해서 여러 개의 가상 노드로 구성된 클러스터를 구성할 수도 있습니다. [여러 환경 구성 예](https://kind.sigs.k8s.io/docs/user/quick-start/#advanced)를 살펴봅시다.

```yaml
# kind-example-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
# 하나의 컨트롤 플레인과 3개의 워커 노드로 이루어져 있습니다.
#
# API-server, 그리고 다른 컨트롤 플레인 구성 요소들은 control-plane 노드에 있을 것입니다.
#
# 테스트 환경이 아니라면 이를 이용할 필요는 없을 것입니다.
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
```

위 파일을 구성한뒤, 다음 명령어로 클러스터를 생성합니다.

```bash
kind create cluster --config kind-example-config.yaml
```

## CNI
CNI는 Container Network Interface의 줄임말입니다. CNI plugin 또한 시중에 여러가지 플러그인이 나와있지만, Cilium과 Metallb를 조합해서 사용할 수 있습니다. 자세한 개념은 복잡하므로, 많은 세부사항이 생략되었습니다. 이러한 솔루션을 사용하고 있음만 소개해드리려 합니다.

### Cilium
![cilium](../assets/images/cilium.png)  
[Finda 기술블로그](https://medium.com/finda-tech/kubernetes-%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC-%EC%A0%95%EB%A6%AC-fccd4fd0ae6)에 소개된 것 처럼, Kubernetes는 기본으로 Routing에 Iptable을 사용합니다. Cilium에서 사용하는 방식은 ebpf/xdp입니다. ebpf/xdp는 iptable을 대체할 커널 기술로 많이 주목을 받고 있습니다.

간단히 말해서 기존 Iptables/Netfilter에서 가지는 성능 한계를 ebpf/xdp에서는 해결하고 있습니다. [참고](https://cilium.io/blog/2018/04/17/why-is-the-kernel-community-replacing-iptables)

### Metallb

Metallb는 Baremetal 클러스터에서 작동하는 로드밸런서 솔루션입니다. 클러스터의 외부 IP를 할당해주기도 합니다. 따라서 LoadBalancer 유형 서비스에 할당된 외부 IP를 늘리기 위해서는 Metallb의 설정을 변경해야합니다.

### Reference

- [Kubernetes 공식 문서 : 네트워크 플러그인](https://kubernetes.io/ko/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)
- [K8s 에서의 eBPF/XDP 기반
고성능 & 고가용성 NAT 시스템](https://deview.kr/data/deview/session/attach/1000_T3_%EC%86%A1%EC%9D%B8%EC%A3%BC_Kubernetes%20%ED%81%B4%EB%9F%AC%EC%8A%A4%ED%84%B0%EC%97%90%EC%84%9C%EC%9D%98%20%EA%B3%A0%EC%84%B1%EB%8A%A5&%EA%B3%A0%EA%B0%80%EC%9A%A9%EC%84%B1%20NAT%20Networking%20%EC%8B%9C%EC%8A%A4%ED%85%9C.pdf)

## Storage Solution

[PersistentVolume/PersistentVolumeClaim](##PersistentVolume/PersistentVolumeClaim) 챕터에서 소개했던 [스토리지 클래스](###StorageClass)와 [동적 Provisioning](###Dynamic-provisioning)을 해주는 Provisioner를 제공하는 것이 Storage Solution입니다. 대표적으로 사용하기 쉬운 OpenEBS를 소개합니다.

### OpenEBS

![OpenEBS](../assets/images/OpenEBS.png)

Persistent Volume을 간단하게 생성하고 또 관리할 수 있는 스토리지 솔루션입니다. Mayastor, cStor, Jiva같은 분산 Replicated Persistent Volume 엔진도 지원하고 있지만, 제일 간단하게 사용할 수 있는 방법은 바로 [Local Provisioner를 이용한 로컬 볼륨](https://openebs.io/docs#local-volumes)입니다. 이를 이용해서 어플리케이션이 배포된 노드의 로컬에 볼륨을 생성해 레이턴시가 거의 없이 볼륨에 접근할 수 있도록 만듭니다.

로컬 볼륨을 사용하는 것은 Minio, Elasticsearch, MongoDB (Sharding)처럼 어플리케이션 레벨에서 데이터를 분산해서 저장하는 어플리케이션에 유리합니다. 어플리케이션 레벨에서 이미 데이터를 분산하고 있기 때문에 다른 노드의 로컬 볼륨에 접근할 이유가 없고, 데이터를 스토리지 레벨에서 복제할 필요가 없는 데다, Pod가 배포된 동일한 노드에 배포되는 볼륨이므로 I/O 레이턴시도 낮기 때문입니다.

그러나 반대로 어플리케이션에서 데이터를 분산해서 저장하지 않는 경우, 그래서 스토리지 레벨에서 복제를 할 필요가 있거나, 로컬에서만 접근되는 저장소를 만들면 안되는 경우에는 다른 형태의 볼륨을 선택해야 합니다. 예컨대, OpenEBS에서 지원하는 Mayastor, cStor, Jiva등의 엔진을 선택하면 됩니다.

## Service Mesh

![Complicated Service mesh](../assets/images/servicemeshcasestudies.png)

Service mesh는 Kubernetes 클러스터를 구성하는 Microservice간 통신을 담당합니다. 기존 Kubernetes의 기능들로도 Pod간 통신을 관리하는 것이 가능하지만, 문제는 Microservice의 숫자가 많아지면서 서비스가 복잡해지면 복잡해질수록 Kubernetes의 기존 기능으로는 관리하기가 힘들어집니다. Service mesh는 그런점을 보완하기 위해 나타난 솔루션입니다. Service Mesh로 Microservice간 네트워크 모니터링, Fault injection, Load balancing등을 수행할 수 있습니다.

### Istio

![Istio](../assets/images/Istio.png)

Istio는 Service mesh 및 Ingress/Engress를 지원하는 통합 솔루션입니다. Istio는 Service mesh의 일반적인 아키텍처로 채택되는 Sidecar 구성에서 Sidecar로 Envoy proxy를 injection 합니다. 또한, Grafana와 kiali등의 추가적인 도구로 Dashboard를 구성할 수 있습니다.