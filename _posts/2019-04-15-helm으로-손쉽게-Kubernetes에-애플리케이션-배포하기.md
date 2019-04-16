---
layout: post
title:  "Helm으로 손쉽게 Kubernetes에 애플리케이션 배포하기"
date:   2019-04-15 19:46:45 +0900
categories: Kubernetes
---

<div align="center"><img src="https://cdn-images-1.medium.com/max/1200/0*j5qOv3O6AE-xnUGD.png"/></div><br/>

쿠버네티스에 배포하는 모든 종류의 리소스 (디플로이먼트, 서비스, 인그레스 등)은 모두 `yaml` 파일에 작성된 폼을 기반으로 구성된다. 사용자가 할 일은 열심히 `yaml` 파일을 작성한 후, `kubectl apply` 명령어로 파일을 적용시키기만 하면 끝.

그런데, 본격적으로 쿠버네티스에 이것저것 올리다 보면 뭔가 아쉬울 때가 있다. 한 애플리케이션을 위해 여러 `yaml` 파일을 적용해야 하는 경우, 비슷한 뼈대를 바탕으로 해 다른 일을 처리하는 애플리케이션을 만드는 경우 등 생각보다 기본 `yaml` 파일을 단순히 적용하는 것 이상의 "무언가"에 갈증을 느끼게 된다. 

오늘 들고 온 [`Helm`](https://helm.sh)이 바로 여러분이 찾던 그 "무언가" 다. 너무나도 매끄럽게 잘 작동하는 패키지 매니저로서, `Helm`을 위해 미리 준비한 `Helm Chart`만 있으면 마치 `.exe` 프로그램을 설치하듯, 쿠버네티스라는 내 컴퓨터에 내 애플리케이션을 뚝딱 설치할 수 있다.

깨끗한 쿠버네티스 클러스터를 준비한 후, 아래 글을 쭉 따라와보자. 곧 `Helm`의 매력에 푹 빠질 것이다. 

Helm 설치하기
======

마치 쿠버네티스 클러스터에 접근하기 위해서는 `kubectl`이 필요하듯, `Helm`을 쿠버네티스 클러스터에 설치하고 애플리케이션을 배포하기 위해서는 `Helm` 클라이언트가 필요하다.
 
macOS에서는 `brew`를 통해 아래 명령어로 간단히 설치할 수 있다. 다른 OS에서는 [이 가이드](https://helm.sh/docs/using_helm/#installing-helm) 를 참고하자.

> 만약 로컬에 `kubectl`이 설치되어 있지 않을 경우, 아래 명령어가 `kubectl`까지 같이 설치합니다. 설치되어 있을 경우라도 최신 버전으로 자동으로 업데이트 하니 만약 구버전 `kubectl` 또는 `helm`이 필요할 경우 특정 버전을 지정하여 설치하시기 바랍니다.

```
brew install kubernetes-helm
```

설치가 완료된 후 `helm version` 명령어를 통해 정상적으로 설치가 진행되었는지 꼭 확인하자. 지금은 서버 측에 `helm`이 준비되지 않아 `Client:`로 시작하는 로컬 `Helm` 정보만 출력될 것인데, 정상이다. 

클라이언트가 제대로 준비되었다면, 이제 쿠버네티스에 `Helm`을 설치할 차례다. `Helm`은 `Tiller`라는 특수한 Pod를 통해 쿠버네티스 위에서 리소스를 생성하고 지우는데, 자유롭게 쿠버네티스를 쥐락펴락 할 수 있어야 하기 때문에 그에 맞는 권한을 할당해줄 필요가 있다.

아래 코드를 `yaml` 파일에 저장해 `kubectl apply` 하자. 아래 `yaml`은 `Tiller`라는 `cluster-admin`에 연결된 새로운 권한을 생성한다.

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
```

이제 `Tiller`를 쿠버네티스 위에 올리면서, 방금 생성한 권한을 부여해 보자.

```
helm init --service-account tiller
```

완전히 `Tiller` Pod가 준비될 때까지 잠깐 기다린 후, `helm version` 명령어를 다시 실행해 서버 측에도 완전히 `Helm`이 준비되었는지 확인하자. 준비가 되었다면 아래와 같은 값이 출력될 것이다.

> 필자는 구버전 `helm`이 설치되어 있는 테스트 환경에서 명령어를 실행해 버전이 `v2.9.1`로 표시되었다. 글을 작성하는 현재 최신 `helm`은 `v2.13.0`이다.

```
$ helm version
Client: &version.Version{SemVer:"v2.9.1", GitCommit:"20adb27c7c5868466912eebdf6664e7390ebe710", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.9.1", GitCommit:"20adb27c7c5868466912eebdf6664e7390ebe710", GitTreeState:"clean"}
```


Helm Chart로 애플리케이션 실행하기
======

`Helm`이 준비되었다면, 이제 `Helm Chart`가 필요하다. 마치 윈도우 컴퓨터에 설치하는 `.exe` 패키지처럼, 쿠버네티스라는 컴퓨터를 위한 애플리케이션 패키지를 구해오는 것이다.

만약 실행하려고 하는 것이 `ElasticSearch`나 `Jenkins` 같은 잘 알려진 오픈소스 애플리케이션일 경우, 대부분 해당 제작사에서 `Helm Chart`를 같이 제작해 배포한다. 이런 `Chart`들은 [`Helm Stable Repo`](https://github.com/helm/charts/tree/master/stable) 에서 확인할 수 있다. 

`Helm` 패키지 설치는 `helm install` 명령어를 통해 이루어진다, 적당한 패키지를 고르지 못했다면, 필자가 만든 데모용 애플리케이션을 설치해보며 어떻게 `Helm`을 사용할지 감을 잡을 수 있다.

먼저 [이 링크](../.post_attachments/helm-helloworld.tgz)에서 데모 `Helm Chart`를 다운로드 받은 후, 압축을 풀고, 적당한 텍스트 에디터로 열어보도록 하자. 구조는 아래와 같을 것이다.

> `Helm Chart` 설치는 `tgz`로 압축된 상태에서도 가능하지만, 이번에는 실습을 위해 압축을 해제했습니다. 물론 압축 해제된 폴더 그 자체로도 설치가 가능합니다. 

```
├── Chart.yaml
├── charts
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   └── deployment.yaml
└── values.yaml
```
아래 명령어를 통해 쿠버네티스에 올려보도록 하자.

쿠버네티스의 모든 동작이 `yaml` 파일을 기반으로 이루어지는 것처럼, `Helm Chart`도 결국에는 `yaml` 파일들의 집합체이다. `Helm Chart` 설치를 진행하면 `templates` 폴더 안에 있는 모든 `yaml`을 실행한다. 예시를 확인하기 위해 `deployment.yaml` 파일을 열어보자. 내용은 아래와 같을 것이다.

```
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: helm-helloworld
  labels:
    app: helm-helloworld
    chart: helm-helloworld
spec:
  replicas: 1
  selector:
    matchLabels:
      app: helm-helloworld
  template:
    metadata:
      labels:
        app: helm-helloworld
    spec:
      containers:
        - name: helm-helloworld-container
          image: "alpine:3.6"
          env:
          - name: PRINT_VALUES
            value: {{.Values.printValues}}
          command: ["/bin/sh","-c"]
          args: 
          - while true; do echo $PRINT_VALUES; sleep 5; done;
```

일반적으로 보던 디플로이먼트를 생성하는 `yaml` 같아 보이지만, 못 보던 무언가가 `env` 칸에 있다. 여기서 `Helm`의 진가 중 하나가 들어난다. 바로 파라미터화다. 같은 `yaml` 파일이여도 동적으로 어떤 값을 넘겨주냐에 따라 다른 결과물을 만들 수 있다. 그리고 그 값들은 `Helm Chart` 최상위 경로에 있는 `values.yaml`에 정의된다.

`values.yaml` 파일을 열어보면 아래와 같은 값이 먼저 정의되어 있을 것이다.

```
printValues: "hello world!"
```

`printValues` 값이 `hello world!` 로 정의되어 있다. 대충 감을 잡으셨겠지만, 이대로 `Helm Chart`를 쿠버네티스에 설치하면 `hello world!`를 5초에 한번씩 출력하게 될 것이다. 출력 문구를 바꾸고 싶다면, 이걸 수정하면 된다. 

일단 대충 구조를 파악했으니, 쿠버네티스에 설치부터 해보자. 아래 명령어를 참고하자.

```
helm install --name helm-demo --namespace default <압축해제한_HELM_경로> 
```

우리는 방금 필자가 만든 데모용 `Helm Chart`를 `helm-demo` 라는 설치 이름으로 `default` 네임스페이스에 설치하였다. 

이제 `kubectl get pods` 명령어로 방금 실행한 애플리케이션의 Pod 이름을 확인한 후, `kubectl logs pod/<확인한_POD_이름>` 명령어를 통해 로그를 긁어와보자.

```
$ kubectl logs helm-helloworld-8d6f9c5db-84pjv
hello world!
hello world!
hello world!
hello world!
hello world!
```

정상적으로 Pod가 실행되어 로그를 출력하고 있는 것을 확인할 수 있다. `kubectl get deployments` 명령어로 디플로이먼트도 제대로 생성되었는지 체크하자.

`Helm`으로 새로운 애플리케이션을 생성해 봤으니, 이제 업그레이드도 해보자. 하는 김에 출력될 문구를 바꿔봐도 좋겠다. 

```
helm upgrade helm-helloworld --set printValues="hello helm!" <압축해제한_HELM_경로> 
```

잠시 기다린다면 새로운 Pod가 새로운 값을 읽고 시작되어, 아래와 같이 `hello world!` 대신 `hello helm!` 이라는 문구를 출력할 것이다. 

```
$ kubectl logs helm-helloworld-8d6f9c5db-5p1bj
hello helm!
hello helm!
hello helm!
hello helm!
hello helm!
```

이처럼 `Helm`은 애플리케이션 설치와 업그레이드 시 동적으로 값을 바꿀 수 있게 해준다. 이를 잘 응용해 업그레이드 시 도커 이미지의 버전을 바꾸는 등의 사용도 얼마든지 가능하다. 만약 업그레이드를 하는데 이전에 넘긴 값을 그대로 쓰고싶다면 `--reuse-values` 파라미터를 같이 넘기면 된다. 

자. 작동 되는것도 잘 봤고 적당히 가지고 놀았으니 이제 깔끔하게 정리할 차례다. 현재 설치되어 있는 `Helm Chart`들은 `helm ls` 명령어로 확인할 수 있다. 우리가 설치한 `Helm Chart`의 이름은 `helm-helloworld`이니 아래 명령어를 통해 완전히 삭제할 수 있다.

```
helm del --purge helm-helloworld
```

완전히 삭제된 후, 다시 `helm ls` 명령어를 실행해 다시한번 확인해보는 것도 좋다.


더 알고싶어요!
======

방금까지 첫 `Helm` 사용에 필요한 건 대충 다 해봤다. 이제 본격적으로 배포에 `Helm`을 사용하고 싶으신 분들은 아래 글들을 참고하면 도움이 될 것이다. 

- MS 공식 `Helm` 사용 가이드. 위에서 다룬 것처럼 깡통 상태에서부터 `Helm`을 셋업하고 쓸 수 있도록 도와준다. - [https://docs.microsoft.com/ko-kr/azure/aks/kubernetes-helm](https://docs.microsoft.com/ko-kr/azure/aks/kubernetes-helm)

- Helm 공식 사용 안내 페이지. - [https://helm.sh/docs/using_helm/](https://helm.sh/docs/using_helm/)
