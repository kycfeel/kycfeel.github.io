---
layout: post
title:  "스스로 Kubernetes 클러스터 구축하기"
date:   2018-10-14 18:25:45 +0900
categories: Kubernetes
---
<div align="center"><img src="https://cdn.dribbble.com/users/1714527/screenshots/4245728/kubernetes.gif"/></div><br/>

자. 새로운 주제의 새로운 시작이다.

단순한 흥미 목적이 됐건, 프로덕션 서비스를 꿈꾸고 있던 간에, 모든 일의 시작은 환경 구축이다. 정말 다행인 건, 쿠버네티스가 점점 유명해지고 활발하게 쓰이는 만큼 사용도 편하게 할 수 있도록 다양한 도구와 서비스가 나오고 있다는 점이다. 요즘은 AWS나 GCE같은 퍼블릭 클라우드 사업자에서 원클릭 쿠버네티스 클러스터 서비스도 제공하고 있다.

그렇지만, 첫 단추부터 잘 꿰어야 한다고 하지 않나. 설령 내 클라우드 사업자가 쿠버네티스 서비스를 제공하지 않더라도, 혹은 온프레미스 환경을 사용해야 한다고 해도 문제업이 쿠버네티스 클러스터를 구축할 수 있도록, 오늘 우리는 [kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm/) 이라는 도구를 사용할 것이다.


본격적인 구축에 앞서...
======

kubeadm은 가장 빠르고 간단하게 미니멀한 쿠버네티스 클러스터를 구축할 수 있는 쿠버네티스 공식 인증 도구다. 많은 사람들과 가이드에서도 kubeadm 사용을 권장하고 있으니 혹여나 여기서 다루는 내용이 비표준일 것이라는 걱정은 안 해도 된다.

필자의 테스트 환경은 아래와 같다.

```
- Ubuntu 16.04 1vCore / 1GB - master
- Ubuntu 16.04 1vCore / 2GB - node1 (slave)
= Ubuntu 16.04 1vCore / 2GB - node2 (slave)
```

잠깐. master? node? 생소한 단어가 막 나온다. 아니, 얼핏 개념은 알 것이다. master가 책임자고 node가 일한다고 말이다.

<div align="center"><img src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/kubernetes_basic_diagram.png?raw=true"/></div><br/>

바로 그거다. 쿠버네티스에서도 크게 달라지는 것은 없다. master에는 쿠버네티스 전체를 제어할 수 있는 API 서버나, 쿠버네티스 클러스터에 관한 정보를 저장하는 [etcd](https://github.com/etcd-io/etcd) 저장소 등이 구동된다. node가 진짜 일꾼이다. 우리가 앞으로 올릴 애플리케이션 컨테이너는 바로 node에서 구동될 것이다. 이 개념을 감안해 본인의 환경에 맞게 적절하게 테스트 환경의 리소스를 조절하길 바란다.

master 구성하기
======

먼저 master에 ssh 접속한 후, 다음 명령어를 통해 필요한 패키지를 설치해보자.

```
sudo apt-get update
sudo apt-get install -y docker.io

apt-get update && apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -

cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF

apt-get update
apt-get install -y kubelet kubeadm kubectl
```

자. 방금 우리는 명령어 몇 줄을 통해 깡통 머신에 도커 데몬과 쿠버네티스 클러스터에 필요한 필수 패키지들을 설치했다. 이제 이 녀석을 master로 탈바꿈시켜야 하는데, 그것마져도 간단하다. 다음 한 줄이면 된다.

```
kubeadm init --apiserver-advertise-address <master_프라이빗_IP> --apiserver-cert-extra-sans <master_퍼블릭_IP> --pod-network-cidr=10.244.0.0/16
```

간단하다고는 했는데, 뭐가 좀 많다? 하나하나 짚어보자. `--apiserver-advertise-address`는 master의 메인 IP 주소. node들이 직접 붙을 바로 그 주소다. 당연히 보안상 프라이빗 네트워크를 사용하는 편이 좋으니, master 서버의 프라이빗 IP를 기입하자. `--apiserver-cert-extra-sans`는 추가적으로 인증에 사용할 IP 주소. 외부에서 쿠버네티스 클러스터에 접근할 수 있는 편이 관리 측면에서 월등하게 편리하니, master의 퍼블릭 IP를 기입한다.

`--pod-network-cidr` 는 조금 다른 녀석인데, 쿠버네티스 클러스터에서 구동되는 모든 구성요소 (컨테이너) 등은 자체적인 오버레이 네트워크를 통해 서로 통신한다. 쿠버네티스가 자체적으로 제공하는 오버레이 네트워크 솔루션은 없고, 서드파티를 사용해야 하는데 그 중 우리가 사용할 것은 가장 대중적으로 사용되는 [Flannel](https://github.com/coreos/flannel) 이다. `10.244.0.0/16`이 바로 Flannel이 요구하는 CIDR 범위임으로 init시 지정해줘야 추후 사용에 무리가 없다.

초기 셋업 과정들이 자동으로 진행된 후, 완료되었다는 메시지와 함께 `kubadm join` 으로 시작하는 명령어가 출력되었을 것이다. 이것을 복사해서 한쪽에 잘 보관해두자. 눈치가 빠른 분들은 알아차렸겠지만 추후 이 명령어를 사용해 node들을 master에 연결시킬 것이다.

거의 다 왔다. 방금 만든 쿠버네티스 master에 오버레이 네트워크 플러그인만 설치하면 된다. 바로 위에서 다룬 Flannel을 말하는 것이다.

```
cp /etc/kubernetes/admin.conf $HOME/
chown $(id -u):$(id -g) $HOME/admin.conf
export KUBECONFIG=$HOME/admin.conf
```

위 과정을 통해 `KUBECONFIG` 파일을 사용할 수 있도록 준비시킨다. 쿠버네티스 클러스터의 접속 정보가 담긴 일종의 열쇠라고 생각해도 무방하다.

```
sysctl net.bridge.bridge-nf-call-iptables=1
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.10.0/Documentation/kube-flannel.yml
```

오버레이 네트워크가 작동할 수 있도록 방화벽을 약간 만져주고, 플러그인을 설치해 보자. 여기에 쓰인 `kubectl apply -f` 명령어는 앞으로도 자주 쓸 녀석이다. 쿠버네티스의 대부분의 애플리케이션 생성, 기타 구성 과정 등은 yaml 파일에 저장된 구성 정보를 읽어 이루어진다. 방금 우리는 온라인 저장소에서 파일을 가져와 바로 쿠버네티스 클러스터가 읽게 만들었다.

이제 쿠버네티스의 뿌리가 갖춰졌으니, 이제 간단히 커뮤니케이션을 해보자. 아래 과정을 통해 쿠버네티스 API에서 현재의 클러스터 구성 상태를 불러올 수 있다.

```
kubectl get nodes
```

나의 master 노드와 현재 상태 (Ready) 가 보이는가? 잘 따라오고 있다!

클러스터에 node 추가하기
======

기본 베이스는 master와 다르지 않다. 동일한 Docker 데몬, 동일한 kubelet 등 쿠버네티스 의존성을 요구한다.

node가 될 서버에서 아래 과정을 통해 필요한 물밑작업을 진행하자.

```
sudo apt-get update
sudo apt-get install -y docker.io

apt-get update && apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -

cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF

apt-get update
apt-get install -y kubelet kubeadm kubectl
```

여기까지 따라왔다면 다른 것은 딱 하나, 최초의 master에서는 init 과정을 통해 클러스터를 생성했다면, 이제 node에서는 join 커멘드로 이미 생성한 클러스터에 참여하면 된다.

위에서 언급한 join 커멘드 복사본을 가져와 node에 붙혀넣기한다. 다음과 같은 형태로 존재한다.

```
kubeadm join <MASTER_IP:PORT> --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

잠시 시간이 지나고, master에서 `kubectl get node` 커멘드를 실행하면 우리가 추가한 node까지 같이 출력되는 모습을 볼 수 있다. Ready 사인이 들어와 있으면 사용 준비 완료.

클러스터 시각화하기
======

> 로컬 머신에서 쿠버네티스 클러스터에 접근하고 싶을 경우, 서버에 있는 KUBECONFIG 파일을 옮긴 후 열어 프라이빗 IP가 적혀있는 `server` 부분을 master의 퍼블릭 IP로 교체해 주어야 합니다. kubectl 설치가 따로 필요할 경우, [이 곳](https://kubernetes.io/docs/tasks/tools/install-kubectl/)을 참조하세요.

방금 우리는 사용 가능한 쿠버네티스 클러스터를 완성시켰다! 그래, 다 좋은데, 이제 뭘 해야 할까? 아직 쿠버네티스에 전혀 익숙하지 않은데, 이 검은 창만 보며 모든 일을 진행해야 하는 걸까?

다행히 아니다. 쿠버네티스는 아래와 같은 멋진 [공식 웹 UI](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/)를 제공한다. 다만 따로 설치해야 할 뿐. 하지만 쿠버네티스를 처음 사용하는 사람에게는 좋은 실습과 적응 과정이 될 수 있다. 자, 따라오자.

<div align="center"><img src="https://d33wubrfki0l68.cloudfront.net/e6bda94ebf94cc460db5cdc42bbfdb8f95f5f7ce/fd28b/images/docs/ui-dashboard.png"/></div><br/>

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```

이제 쿠버네티스 시스템이 알아서 필요한 파일을 다운로드받아 대시보드를 구동시킬 것이다. 위의 `kubectl apply -f` 명령어는 영어 그대로, kubectl을 통해 file (`-f`)을 적용시키는 일을 한다. 방금 예시처럼, 앞으로 대부분의 쿠버네티스 리소스 생성은 완성된 yaml 파일을 통해 이루어질 것이다. 일단 파일을 완성만 하면 그 어떤 쿠버네티스 환경에서도 똑같이 구동된다. Infrastructure-as-Code가 피부로 확 체감되는 순간이다.

얘기는 나중에. 기껏 대시보드를 설치했으니 직접 접속해보자.

쿠버네티스에서는 대시보드의 접속에 [kube-proxy](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/)를 사용할 것을 권장한다. 쿠버네티스 API를 통해, 컨테이너 네트워크에 직접 붙어 `localhost`를 통해 내부 구성요소들에 접속할 수 있게 해주는 컨셉인데, 이를 사용하면 로컬 머신에서의 컨테이너 (혹은 서비스) 접근을 위해 외부 네트워크로 포트를 열지 않아도 된다. 쿠버네티스 대시보드도 마찬가지로 외부에 그냥 오픈할 경우 여러가지 보안 문제가 생길 수 있다. 조금 불편하더라도 프록시를 사용해 접속하도록 하자.

아래 명령어로 바로 kube-proxy를 구동할 수 있다.

```
kubectl proxy (&를 뒤에 붙혀 백그라운드 구동 가능)
```

별다른 에러가 발생하지 않는다면, 이제 내 로컬 머신이 클러스터 내부 네트워크에 붙었다는 소리다. 웹브라우저에서 아래 주소로 접근하면 대시보드가 반갑게 환영해줄 것이다.

```
https://<master-ip>:<apiserver-port>/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
```
<div align="center"><img src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/kubernetes-dashboard-hello.png?raw=true"/></div><br/>

빰. 어렵게 어렵게 만났다. 근데 잠깐만, 로그인을 하라고 한다. kubeconfig 파일 또는 토큰을 입력해야 하는데, 아마 지금 가지고있는 kubeconfig를 넣어도 로그인이 안될 것이다. 파일 안에 토큰 정보가 들어있지 않기 때문이다. 우리는 어드민 권한으로 로그인하고 싶으니, 어드민 권한을 가진 클러스터 내부 계정을 하나 만들어 토큰값을 뽑아보자. 아래의 코드를 긁어 `yaml` 형태로 저장한 후, 클러스터에 적용하는 것부터 시작하자.

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-admin
  namespace: kube-system

---

apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: dashboard-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: dashboard-admin
  namespace: kube-system
```

모든 yaml 파일 적용은 조금 전 해봤던 것과 같이, `kubectl apply -f <파일_경로>` 명령어를 사용하면 된다.

콘솔에 무언가 만들어졌다고 주르륵 떴는데, 토큰값은 따로 보이지 않는다. 괜찮다, 정상이다. 이제 계정만 만들어졌을 뿐, 토큰은 우리가 뽑아야 한다.

```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep dashboard-admin | awk '{print $1}')
```

위 명령어를 통해 토큰값을 얻을 수 있다. 값을 복사한 후 대시보드 로그인 창에 넣어보면 정상적으로 메인 화면이 나오는 것을 볼 수 있다. 일일히 토큰값을 복사하는게 귀찮다면 kubeconfig 파일에 입력할 수도 있는데, 아래의 예시를 따라하자. 파일 형태는 조금 다를 수 있으나 위치는 동일하다.

```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: (...)
    server: (...)
  name: (...)
contexts:
- context:
    cluster: (...)
    user:(...)
  name: (...)
current-context: (...)
kind: Config
preferences: {}
users:
- name: (...)
  user:
    token:
    <여기에 토큰값>
    exec:
      (...)
```

정상적으로 입력했다면, 로그인 창에서 kubeconfig 파일을 선택하는 것으로 접근이 가능할 것이다.


Hello, World?
======

방금 전 과정까지 해서 우리는 직접 깡통 머신에 쿠버네티스를 설치하고, 클러스터를 구성하고, 대시보드 애플리케이션 구동과 내부 계정 생성까지 모두 해보았다. 아직은 어벙벙 하겠지만, 앞으로의 쿠버네티스 사용 대부분은 방금 우리가 거쳐온 방법들을 사용한다. 역시 맛보기로는 직접 만들어보는 것처럼 좋은 일이 없다.

이제 막 클러스터가 만들어졌으니, 무언가 자꾸 올려보고 싶은 건 당연하다. 아래의 링크들을 참조하면 여러가지 데모 애플리케이션을 직접 실행하며 감을 잡을 수 있을 것이다.

마지막으로 이 글을 정독한 모두에게, 클러스터 오케스트레이션 세상에 첫 발을 내딛을 걸 환영한다고 박수를 쳐주고 싶다. 처음 접근은 어려울 지 몰라도, 한번 익숙해지면 이전의 서버 운영과는 차원이 다른 편안함과 유연함을 맛볼 수 있을 것이라 장담한다.

- 쿠버네티스 워드프레스 실행하기 : https://kubernetes.io/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/

- 쿠버네티스에 Postgres 데이터베이스 구축하기 : https://severalnines.com/blog/using-kubernetes-deploy-postgresql

그리고 다음에 다룰 내용에 관한 글도 하나 첨부한다. 기존의 쿠버네티스 yaml 파일도 쓸만은 하지만, 너무나 정적인 탓에 같은 애플리케이션 (또는 이미지) 기반으로 여러 환경을 만들어야 할 경우 그 환경들을 위한 모든 yaml 파일을 만들어야 하는 등 단점이 명확한데, Helm 이라는 패키지 매니저가 그런 문제를 해결해줄 수 있다. 자세한 내용은 다음에 다루겠다.

- 쿠버네티스 패키지 매니저, Helm 사용하기 : https://daemonza.github.io/2017/02/20/using-helm-to-deploy-to-kubernetes/
