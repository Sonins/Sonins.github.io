---
layout: post
title:  "Etcd health check 문제"
categories: Kubernetes Troubleshooting etcd
---
<!--
소개
 - 어떤 주제인가?

증상
 - 어떤 장애가 일어났는가?
 - 눈에 띄는 로그가 무엇이 있는가?

원인
 - Etcd가 SSD/HDD/Nvme 중에 어디에 설치되어야 하는가?
 - health check가 Etcd io 속도에 어떤 영향을 받나?

해결 방법
 - 드라이브를 바꿀 수 있다면 해결책은 어떤게 있는가?
 - 바꿀 수 없다면 해결책은 어떤게 있는가?
-->
## etcd
etcd는 Go 언어 기반 분산 시스템 키-값 저장소이다. Kubernetes는 etcd에 작업 스케줄링과 서비스 검색을 위한 클러스터 설정값을 저장한다.
Kubernetes의 kube-apiserver는 etcd가 살아있는지 주기적으로 health check을 날린다. etcd가 죽거나 문제가 생기면 health check가 실패하고, 그러면 kube-apiserver 또한 꺼지게 된다.
kube-apiserver에 문제가 생기면 kubernetes를 제어할 수 없다. 이 말은 kubectl 명령어도 안먹고, apiserver에 직접 접근하는 것도 불가능하다는 이야기다. 
오늘은 etcd의 성능 문제로 이 kube-apiserver와 etcd간 health check에 문제가 생겼던 이야기를 할 것이다.

---

## 증상
### 장애
내가 겪었던 장애는 kube-apiserver가 몇 분 주기로 종료되는 것이었다. 당연히 분 주기로 apiserver가 꺼지면 kubernetes도 제어가 안되므로 큰 문제다. Pod등의 객체의 CRUD가 안되기 때문이다.
### Log
아래 표시된 로그들은 실제 오류 환경에서 가져온 것이 아니므로 부정확 할 수 있다. 그러나 최대한 기억에서 가깝게 인터넷에서 로그들을 가져왔다.
먼저 apiserver가 꺼지므로 apiserver의 로그를 살펴본다. 많은 로그중에 다음 로그가 눈에 띄었다.
```
Readiness probe failed: Get "https://xx.xx.xx.xx:6443/healthz": context deadline exceeded
```
Readiness probe가 context deadline exceeded 라는 이유로 실패했다는 로그다.  
실제로 kube-apiserver의 /healthz 엔드포인트에 GET 요청을 날려보면 (브라우저로 접속하면) 때때로 다음과 같은 실패 메시지가 출력되었다.
```
[+]ping ok
[+]log ok
[-]etcd failed: reason withheld
[+]poststarthook/start-kube-apiserver-admission-initializer ok
[+]poststarthook/generic-apiserver-start-informers ok
[+]poststarthook/priority-and-fairness-config-consumer ok
[+]poststarthook/priority-and-fairness-filter ok
[+]poststarthook/start-apiextensions-informers ok
[+]poststarthook/start-apiextensions-controllers ok
[+]poststarthook/crd-informer-synced ok
[+]poststarthook/bootstrap-controller ok
[+]poststarthook/rbac/bootstrap-roles ok
[+]poststarthook/scheduling/bootstrap-system-priority-classes ok
[+]poststarthook/priority-and-fairness-config-producer ok
[+]poststarthook/start-cluster-authentication-info-controller ok
[+]poststarthook/aggregator-reload-proxy-client-cert ok
[+]poststarthook/start-kube-aggregator-informers ok
[+]poststarthook/apiservice-registration-controller ok
[+]poststarthook/apiservice-status-available-controller ok
[+]poststarthook/kube-apiserver-autoregistration ok
[+]autoregister-completion ok
[+]poststarthook/apiservice-openapi-controller ok
healthz check failed
```
healthcheck 중 하나인 etcd healthcheck가 실패했고 그 이유는 생략되었다는 것이다. 실패했으면 실패한거지 이유까지 안보여줄 건 또 무엇인가??? 라는 생각이 들었지만 안보여준다면 뭐 어쩔 수 없는 것이고, 어쨌건 문제는 해결해야한다. kube-apiserver의 healthcheck 실패의 원인이 etcd임을 확인했으므로 다음은 etcd의 로그를 살펴보았다.  
kube-apiserver 로그를 살펴볼 때 context deadline exceeded라는 말이 눈에 띄었었는데, etcd 서버 로그에서도 똑같은 로그가 출력되는 것이 보인다.
```
etcdserver: read-only range request "key:\"/registry/health\" " with result "error:context deadline exceeded" took too long (2.000041013s) to execute
```
해석하자면, "/registry/health로 들어온 read-only 요청에 context deadline exceeded라는 결과를 내는데 2.000...초라는 시간이 걸렸고, 이건 너무 오래 걸린 것이다." 라는 뜻이다. 저 2초가 조금 안되는 시간이 눈에 띈다. 2초가 아주 조금 지난 시간이 너무 오래 걸린 시간이라면, 제한이 2초임은 쉽게 짐작할 수 있다. etcd가 context deadline인 2초내 health check에 응답하지 못해 문제가 발생한 것이다.  
로그를 살펴본 결과, 모종의 이유로 etcd가 2초 이내에 health check 응답을 하지 못해, kube-apiserver의 readiness probe가 실패한 것이 계속해서 kube-apiserver가 재시작 된 이유였다. 그렇다면 etcd가 2초 이내 health check 응답을 하지 못한 이유를 살펴보아야 문제를 해결할 수 있을 것이다.

---

## 원인
### etcd 성능 문제
```
Disks

Fast disks are the most critical factor for etcd deployment performance and stability.
...
When possible, back etcd’s storage with a SSD. A SSD usually provides lower write latencies and with less variance than a spinning disk, thus improving the stability and reliability of etcd.
...
```
[etcd's Hardware recommendation](https://etcd.io/docs/v3.3/op-guide/hardware/#disks)

위에 써져있는 바와 같이 etcd의 성능은 디스크의 속도가 매우 결정적인 요소이다. 그 이유는 etcd가 결국 저장소 어플리케이션이기 때문이다. 일반적으로 다른 데이터베이스들이 그러하듯, etcd 또한 디스크 쓰기/읽기 속도에 결정적인 영향을 받는다. 그러므로 etcd 개발사 측에서는 etcd 스토리지로 가능하면 SSD를 사용할 것을 권장하고 있다.  
<b>문제는 그때 당시 사용하던 운영용 스토리지가 HDD였다는 사실이다.</b> 사실 서버 운영체제가 깔려있는 스토리지가 HDD면 초래할 문제는 이것 외에도 많기 때문에, 어떤 회사던지 프로덕션 환경에서 HDD를 백업용이 아니라 운영용으로 사용하는 곳은 아마 없을 것이다. 그렇지만 어찌되었건 스토리지가 HDD였기 때문에 etcd 성능이 불안정해지고, 결과적으로 트래픽이 많이 생기면 health check에 etcd가 제때 (타임 아웃 이전에) 응답하지 못하는 경우가 생기는 것이다. 그러므로 etcd health check가 실패했던 것이었다.  
사실 인터넷에서도 health check에 대한 이야기를 찾아보면, 대부분의 경우 (서버/응답 객체에 장애가 발생하는 특별한 경우를 제외하면) 성능이 주 원인이라는 이야기가 많다. 보통은 메모리나 CPU 성능을 주로 꼽지만, 이번에는 디스크 성능이 문제였다.

### kube-apiserver의 healthcheck duration
로그를 살펴보았을 때 2초라는 제한이 있었음을 확인할 수 있었다. 그렇다면 이 2초를 어디서 설정하는지 확인하고, 이 설정을 바꿀 수만 있다면 etcd 성능이 좀 불안정 하더라도 kube-apiserver가 재시작 되는 일은 없을 것이다. 이는 kubernetes 공식 문서에서 확인 할 수 있었다.
```
--etcd-healthcheck-timeout duration     Default: 2s

The timeout to use when checking etcd health.
```
[kube-apiserver docs](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/#options)  
공식 문서에 표시된 것처럼, `--etcd-healthcheck-timeout` 이라는 옵션이 etcd health check 응답 시간의 제한을 설정하고 있고, 기본값이 2초임을 확인 할 수 있다. 기본값에서 딱히 건드리지 않았기 떄문에 제한이 2초가 된 것이다. 그렇다면, 이 옵션을 2초보다 더 긴 시간으로 지정하면, kube-apiserver가 그 전보다는 더 낮은 빈도로 종료될 것이다.  
물론 healthcheck timeout이 있는데는 이유가 있다. kube-apiserver가 계속 재시작 된다는 이유로 timeout의 제한을 풀어버리거나, 터무니 없이 긴 시간으로 지정하면, health check 자체가 의미가 없어질 것이다.

---

## 해결
### 근본적인 해결책
근본적인 해결책은 역시 etcd 서버가 동작하는 스토리지를 SSD로 바꾸는 것이다. SSD로 바꾼다는 것이 문제를 해결할 때에는 OS가 HDD에 깔려있으므로 root 파일 시스템이 마운트 된 디스크 자체를 바꿔야 한다고 생각했는데, 지금 생각해보니 몇 가지 과정을 거치면 OS는 내버려두고 etcd 서버만 SSD 위에서 작동할 수 있을 것 같다. <u>다만, 시도해보지 않았으므로 테스트는 필요하다.</u> 실제로는 이 방법으로 문제를 해결하지 않았다.  
해결 과정은 다음과 같다.  
1. SSD를 파일 시스템에 마운트 한다. `sudo mount /dev/sdb1 /var/lib/ssd`  
당연히 `/var/lib/ssd`는 자의적인 예시 경로다.
2. etcd를 잠시 끈다. etcd는 static pod이므로 컨트롤 플레인 호스트 노드의 `/etc/kubernetes/manifest/etcd.yaml`를 잠시 제거하는 것으로 etcd pod는 종료된다. 진짜로 제거하면 안되고 HOME 폴더등에 잠시 옮겨준다.
3. migration 과정을 거친다. 단순히 copy & paste 일 수도 있고, 다른 방식이 더 있을 수 있다. [etcd migration](https://stackoverflow.com/questions/54342705/in-a-kubernetes-cluster-is-there-a-way-to-migrate-etcd-from-external-to-interna) 링크에 migration 과정이 기술되어 있다. 이 경우에는 `/var/lib/ssd/etcd`에 migration 해야겠다.
4. `etcd.yaml`의 etcd-data 볼륨의 path를 `/var/lib/ssd/etcd`로 고쳐준다. hostPath 이므로 host 서버에 존재하는 `/var/lib/ssd/etcd` 경로의 migration 된 데이터들을 마운트 할 것이다. 자세한것은 [etcd.yaml](https://zetawiki.com/wiki//etc/kubernetes/manifests/etcd.yaml)에 작성되어 있는 `etcd.yaml` 파일의 형식을 확인하면 이해하기 쉬울 것 같다.
5. 다시 etcd를 실행한다. 수정된 `etcd.yaml`를 `/etc/kubernetes/manifest/etcd.yaml`에 넣어주면 자동으로 실행될 것이다.  

이 과정을 거치면 SSD를 사용하는 etcd가 재배포 될 것이다. 그러나 테스트는 해본 적이 없다.  
물론 그냥 OS를 다 밀고 SSD에 모든걸 다시 설치하는 방법도 있다.
### 실제로 해결한 방법
healthcheck timeout을 두 배로 늘리는 것이다. `/etc/kubernetes/manifest`에는 etcd 뿐만 아니라 kube-apiserver의 스펙도 기술되어 있다. 파일 이름은 `/etc/kubernetes/manifest/kube-apiserver.yaml`이다. 이 방법의 경우 더 간단한데, `/etc/kubernetes/manifest/kube-apiserver.yaml`에 있는 `--etcd-healthcheck-duration` 옵션을 2초에서 4초로 늘려주면, kubernetes가 수정을 감지하고 자동으로 kube-apiserver를 수정된 스펙에 맞게 재실행한다. 내 기억으로는 `--etcd-healthcheck-duration` 옵션이 기본으로는 없었고 새로 추가해줘야 했던 것으로 기억한다. `--etcd-healthcheck-duration=4s`가 되겠다.