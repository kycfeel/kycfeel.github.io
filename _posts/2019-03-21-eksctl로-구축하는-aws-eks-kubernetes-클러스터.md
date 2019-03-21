---
layout: post
title:  "eksctl로 구축하는 AWS EKS Kubernetes 클러스터"
date:   2019-03-21 19:48:45 +0900
categories: Kubernetes
---

<div align="center"><img src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/awsekslogo.jpg?raw=true"/></div><br/>

[이전 글](./2018-10-14-스스로-Kubernetes-클러스터-구축하기.md) 에서 kubeadm을 사용해 베어메탈 쿠버네티스 클러스터를 구축하는 법을 다루었는데, 아무래도 실 서비스 환경에서는 조금 더 관리할 것이 덜하고 안정적인 무언가가 필요하다.

그런 생각을 하는 사람은 필자와 우리 독자 뿐만 아니라, 쿠버네티스를 프로덕션에 사용하고 싶어하는 모든 개발자들이 하고 있다. 개발자가 상상한다면 반드시 결과물이 나오는 법. 이미 세상에는 구글, 아마존, MS 등이 제공하는 다양한 관리형 쿠버네티스 서비스들이 나와있다.

그 중 오늘 우리는 아마존에서 제공하는 [AWS EKS]()를 사용해 클러스터를 구축해볼 것인데, 이게 좀 불편하다. 좀 많이 불편하다. 분명 불굴의 아마존이 제공하는 안정적인 서비스이고, 기존에 익숙하게 사용하던 여러 AWS 서비스들과 연동된다는 점은 참 좋은데, 초기 구축이 타 관리형 쿠버네티스와 비교해 많이 불편하다.

그래서 등장한 도구가 바로 [`eksctl`](https://github.com/weaveworks/eksctl)이다. 이미 쿠버네티스 네트워크 플러그인인 `Weave Net`을 선보여 익숙한 분들도 있을 Weaveworks가 제작한 CLI 도구다. EKS 클러스터의 생성과 업그레이드, 삭제까지 모두 간단한 명령어 몇 줄로 해결할 수 있게 해주는 사랑스러운 놈이다.

아래로 진행하기 전, 사용할 수 있는 AWS 계정과 EC2 / EKS 접근이 가능한 IAM 엑세스 키, 그리고 `aws-cli`를 설치해두길 바란다. `eksctl`은 설치되어 있는 `aws-cli`에 의존해 계정 인증을 하고 작업을 진행한다.

그래서 언제 써보나요?
======
<div align="center"><img src="https://eksctl.io/logo/eksctl.png"/></div><br/>

지금부터 쓸꺼다. 바로 지금.

macOS 기준으로 Homebrew를 사용한다면, 아래 명령어로 쉽게 설치할 수 있다.

```
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
```

타 Unix-like 운영체제에서는 아래 명령어로 설치할 수 있다.

```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

모든 설치가 완료되었다면, 이제 바로 클러스터 하나 뚝딱 만들 수 있다. 아래는 기본적인 파라미터가 포함된 예제 명령어다.

> 더 많은 옵션은 [eksctl 공식 문서](https://eksctl.io)에서 찾아볼 수 있다.

```
eksctl create cluster --name=<클러스터_이름> --nodes=<worker_node_갯수> --node-type=<EC2_인스턴스_종류> --region=<AWS_REGION> --kubeconfig=<KUBECONFIG_저장_경로>
```

꽤나 직관적이다. 이제 한 10여분 기다리면 별다른 함정(?) 없이, 내가 지정한 AWS 리전에 지정한 이름, Worker 갯수, KUBECONFIG까지 깔끔하게 생성해 저장해준다.

생성이 완료된 클러스터의 KUBECONFIG 파일을 열어보면 아래와 같은 값들이 있을 것이다.

```
(...)

current-context: (...)
kind: Config
preferences: {}
users:
- name: (...)
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1alpha1
      args:
      - token
      - -i
      - (...)
      command: aws-iam-authenticator
      env: null
    
```

아래에 [`aws-iam-authenticator`](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/install-aws-iam-authenticator.html) 가 보이는가? AWS EKS 클러스터가 계정 인증 처리를 위해 사용하는 소프트웨어다.

우리가 쿠버네티스 명령어를 실행하면 바로 저 `aws-iam-authenticator`가 계정 인증을 마친 후 실제 쿠버네티스 API에 전달될 것이다. `eksctl`과 마찬가지로, `aws-cli`에 로그인된 엑세스 키 정보를 사용한다.

macOS 기준으로 아래처럼 설치하면 된다. 아래 링크가 너무 오래되었거나 타 운영체제를 사용할 경우 [이 링크](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/install-aws-iam-authenticator.html)를 참조하면 좋다.

```
curl -o aws-iam-authenticator https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-07-26/bin/darwin/amd64/aws-iam-authenticator

chmod +x ./aws-iam-authenticator

cp ./aws-iam-authenticator $HOME/bin/aws-iam-authenticator && export PATH=$HOME/bin:$PATH

echo 'export PATH=$HOME/bin:$PATH' >> ~/.bash_profile
```

이제 `aws-iam-authenticator help`를 테스트로 실행해봐 제대로 작동하는지 확인하자. 정상적으로 도움말이 뜬다면 성공.

`aws-cli`까지 제대로 로그인 된 것을 확인했다면, 아래 명령어로 KUBECONFIG를 지정한 후, 간단히 클러스터 정보를 받아와보자.

```
export KUBECONFIG=<내_KUBECONFIG_파일_경로>

kubectl get node
```

별다른 계정 오류 등이 발생하지 않고 아래와 같이 내 클러스터의 Worker 노드 정보가 뜬다면 정상적으로 연결된 것이다. 

```
NAME                      STATUS   ROLES    AGE   VERSION
(...).compute.internal    Ready    <none>   1d   v1.11.5
(...).compute.internal   Ready    <none>   1d   v1.11.5
(...).compute.internal    Ready    <none>   1d   v1.11.5
```

AWS EKS는 생성 과정만 조금 다를 뿐, 한번 생성되면 일반적인 Kubernetes와 아주 똑같다. 이제 아래 문서들을 참고해 웹 대시보드나 Helm 등을 설치하고 본격적으로 내 워크로드들을 올릴 수 있다. 

- [AWS EKS에 Kubernetes 웹 대시보드 설치하기](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/dashboard-tutorial.html)

- [AWS EKS에 Helm 패키지 관리자 설치하기](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/helm.html) 

클러스터 정보 업그레이드
===

AWS EKS에서 사용자가 신경써야 할 것은 Worker가 전부다. Master (API) 는 AWS가 알아서 잘 관리하며, 만약 문제가 있을 경우 사용자 모르게 알아서 고친다. 우리는 Worker 노드만 우리의 워크로드에 맞게 잘 조율해주면 된다.

EKS의 Worker 노드들은 AWS Cloudformation 에 정의된 `Nodegroup`에 의해 관리된다. 지금 내 AWS 콘솔의 Cloudformation 에 들어가보면 `eksctl`이 자동으로 생성한 Nodegroup이 보일 것이다. 이 템플릿이 EC2 Auto Scaling Group도 만들어 알아서 뚝딱 Worker도 올리고, VPC도 만들고, 이것저것 필요한 건 다 한다.

그러므로 자연스럽게 클러스터 정보를 업그레이드할때도 `Nodegroup`을 업그레이드해야 얘가 알아서 나머지를 반영하는 구조가 된다. 어려울 것 같지만 `eksctl`을 사용하면 의외로 쉽게 처리할 수 있다.

> 마찬가지로 더 많은 옵션은 [eksctl 공식 문서](https://eksctl.io)에서 찾아볼 수 있다.

```
eksctl create nodegroup --cluster=<클러스터_이름> --region=<AWS_REGION> --nodes=<worker_node_갯수> --node-ami=<EKS_AMI_ID, 혹은 auto> --node-type=<EC2_인스턴스_종류>
```

이 명령어는 내가 지정한 클러스터에 새로운 설정을 담은 새로운 `Nodegroup`을 만들어준다. `Nodegroup`은 곧 자동으로 새로운 Worker들을 생성해 클러스터 붙혀준다. 다른 타입의 Worker들을 만들고 싶거나 쿠버네티스 버전 (AMI) 를 업그레이드 하고 싶을 경우 들에 유용하다. 이전 `Nodegroup`이 필요가 없어졌다면, 아래 명령어로 삭제하면 된다.

```
eksctl delete nodegroup --cluster=<클러스터_이름> --region=<AWS_REGION>
```

클러스터 삭제하기
======

클러스터 잘 가지고 놀았고 이제 치우고 싶을 때, 아래 명령어로 쉽게 삭제할 수 있다.

```
eksctl delete cluster --cluster=<클러스터_이름> --region=<AWS_REGION>
```

끝!

방금까지 `eksctl`을 사용해 EKS를 전반적으로 둘러보고 맛까지 보았다. 이제 내 진짜 프로덕션 환경을 생각해 다시 클러스터를 세워, 덜 머리아프게 고가용성 워크로드를 돌려보자. `eksctl`로 복잡한 쿠버네티스 조금이라도 쉽고 직관적이게 사용하셨으면 좋겠다. 화이팅.

나는 고급 옵션이 필요해요
======

대부분의 경우 위의 명령어로 "범용적인" 클러스터를 만들 수는 있으나, 조금 더 고급 옵션이 필요한 분들은 꼭 계신다. 그런 분들을 위해 몇가지 준비했다. 물론 위에 언급했던 것처럼 [공식 문서](https://eksctl.io)에서 필요한 것은 다 얻을 수 있지만, 몇가지 정리해봤다. 아래 Flag들은 따로 언급하지 않는 한 클러스터 첫 생성과 `Nodegroup` 생성 둘 다에 쓰일 수 있다.

```
--node-ami=<EKS_AMI_ID, 혹은 auto>
```

`Nodegroup` 생성에서 사용하고 싶은 EKS AMI를 지정할 수 있다.

```
--ssh-access --ssh-public-key=<내_SSH_PUBLIC_키>.pub
```

기본적으로 eksctl이 만든 Worker 노드들은 SSH 접속이 막혀있다. 이 명령어를 사용하면 *클러스터 생성 시* 내가 사용하고 싶은 키를 연결할 수 있다. `--ssh-public-key=<내_AWS에_등록된_키>` 로 이미 내 EC2 리전에 등록된 SSH 키를 바로 연결도 가능하다.

```
--node-volume-size=50
```

기본적으로 EKS의 Worker 노드 저장공간은 20GB다. 대부분의 경우 실 워크로드에는 EBS 볼륨은 연결해 사용하니 크게 지장은 없지만, 만약 높은 저장공간이 필요한 경우 이 명령어처럼 GB 단위를 직접 기입해 늘릴 수 있다.

```
--node-private-networking
```

Worker 노드들을 Private VPC에 위치시키고 싶은 경우, 이 명령어를 통해 Private VPC를 생성해 해당 `Nodegroup`의 노드들을 안에 위치시킬 수 있다.

```
eksctl create cluster \
  --vpc-private-subnets=subnet-0ff156e0c4a6d300c,subnet-0426fb4a607393184 \
  --vpc-public-subnets=subnet-0153e560b3129a696,subnet-009fa0199ec203c37
```

이미 만들어둔 VPC를 사용해 클러스터를 생성하고 싶은 경우, 이 명령어처럼 VPC 서브넷 아이디를 직접 지정해 만들 수 있다.

다른 옵션들이 궁금하다면 위에 걸어둔 링크를 타고 공식 문서를 살펴보자.
