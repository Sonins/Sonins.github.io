---
layout: post
title:  "Kubernetes Basic 3 : Application Operator"
categories: Kubernetes
---
## Application Operator

Kubernetes가 지원하는 기능 중 하나는 CustomResource라는 사용자 지정 객체를 만들 수 있다는 점입니다. CustomResource들을 Kubernetes 객체로 해석하고 지속적으로 모니터링하는 Kubernetes Operator 또한 지원합니다. CustomResourceDefinition으로 CustomResource가 어떤 것인지 지정할 수 있고, Kubernetes operator는 이를 참고하여 CustomResource를 Kubernetes 객체로 해석해서 생성합니다.  

여러 어플리케이션 개발사들은 이를 이용해서 어플리케이션을 Kubernetes Operator 형태로 개발해서 배포하고 있습니다. MongoDB를 예로 들어봅시다. MongoDB 개발사 측은 MongoDB kubernetes operator를 개발해서 배포하고 있습니다. MongoDB 개발사에서 만든 MongoDB operator와 CustomResourceDefinition을 클러스터내에 배포하면, Pod, Deployment 같은 Kubernetes 객체들처럼 MongoDB 객체 또한 조작할 수 있게 됩니다. `kubectl get mongodb` 같은 명령어를 쓸 수 있게 되는것이죠.

이 항목에서는, Operator & CustomResource 패턴으로 개발된 여러 어플리케이션을 소개합니다. 만약 필요한 어플리케이션이 Kubernetes Operator를 지원하는지, 그리고 그것이 사용할만한지 알고 싶은 경우 이 항목을 참고하면 됩니다. 이 항목에 찾는 어플리케이션이 없는 경우, [operatorhub](https://operatorhub.io/)에 많은 Kubernetes Operator 목록이 있습니다.

### MongoDB

MongoDB Operator 종류입니다.

#### Percona MongoDB Operator

![Percona MongoDB](../assets/images/pmdb.png)

- [Percona MongoDB Operator 공식 홈페이지](https://www.percona.com/doc/kubernetes-operator-for-psmongodb/index.html)
- [깃헙](https://github.com/percona/percona-server-mongodb-operator)

Percona MongoDB Operator는 무료로 제공되는 MongoDB Operator입니다. 특별한 Enterprise 버전이 아니더라도 Sharding등 여러 MongoDB의 기능을 지원하고, 모니터링 솔루션 또한 통합해서 제공하고 있습니다.  

Percona MongoDB Operator의 문제점은 커뮤니티의 크기가 많이 작다는 것입니다. 깃헙에 Issue 항목에 여러 버그 리포트가 올라가있는 여타 많은 다른 어플리케이션과는 달리 공식 포럼에서만 커뮤니티가 활성화 되어있습니다.

#### MongoDB Community/Enterprise Kubernetes Operator

![MongoDB](https://camo.githubusercontent.com/0806fae66e28fd4ce9baf881d083ed47cdff1a3b401fe0a8a21652d6fa9a33cb/68747470733a2f2f6d6f6e676f64622d6b756265726e657465732d6f70657261746f722e73332e616d617a6f6e6177732e636f6d2f696d672f4c6561662d466f7265737425343032782e706e67)

- [MongoDB Community Operator 깃헙](https://github.com/mongodb/mongodb-kubernetes-operator)
- [MongoDB Enterprise Operator 깃헙](https://github.com/mongodb/mongodb-enterprise-kubernetes)

MongoDB 공식 Kubernetes Operator입니다. 공식 Operator이다보니 커뮤니티가 활성화 되어있다는 장점이 있습니다.

그러나 Community Operator의 경우 Sharding 등 여러 기능이 제한되어 있는 반면에, Enterprise Operator는 비용을 요구한다는 단점이 있습니다.

### Minio

Minio Operator 종류입니다.

#### Minio Operator

![Minio](https://raw.githubusercontent.com/minio/minio/master/.github/logo.svg?sanitize=true)

- [Minio Operator 깃헙](https://github.com/minio/operator)

Minio 공식 Kubernetes Operator입니다. 특이한 점은 CR/CRD와 오퍼레이터만 클러스터에 배포하는 것이 아니라, `kubectl minio`라는 플러그인을 kubectl에 추가합니다. Minio tenant라는 객체로 Minio 저장소를 배포할 수 있습니다. 모니터링 또한 제공하고 있습니다.

### Elasticsearch

Elasticsearch Operator 종류입니다.

#### ECK (Elastic cloud on Kubernetes)

![ECK](../assets/images/elastic-logo.svg)

- [ECK Quickstart](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-quickstart.html)

Elasticsearch 공식 Kubernetes Operator입니다. ECK 스택의 경우 Elasticsearch 뿐만 아니라 Kibana 또한 포함되어 있습니다. 배포 과정이 쉽고 간편합니다.

### Mysql

Mysql Operator 종류입니다.

#### Percona XtraDB

![Xtradb-logo](../assets/images/kubernetes-xtradb-logo.png)

- [Percona XtraDB Operator 공식 홈페이지](https://www.percona.com/doc/kubernetes-operator-for-pxc/index.html)

여러 개의 Mysql 서버 인스턴스를 배포할 수 있는 XtraDB operator입니다. Mysql Pod간에 데이터를 복제합니다.

사용해본 경험으로 미루어 보았을 때, 오류가 좀 많았던것 같습니다만, Mysql에서 이만큼 스케일링을 지원하는 Operator가 시중에 없습니다.