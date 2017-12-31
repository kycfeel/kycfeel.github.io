---
layout: post
title:  "Portainer를 통한 Docker 초점잡기"
date:   2017-12-30 16:20:45 +0900
categories: Docker
---
자, 2017년 마지막 날까지 까만 바탕에 하얀 글씨만 들여다보고 있는 당신을 위해 [Portainer](https://portainer.io)라는 가볍고 빠른 웹 기반 Docker 환경 관리 도구를 들고왔다.

필자도 평소 꽤 CLI 환경을 애용하는 편이지만, 가끔은 눈으로 보이는 인터페이스가 미친듯이 끌린다. 마치 할리우드 해커가 된 것처럼 이쁘고 깔끔한 그래픽을 바라보며 폼 잡고 싶어진다. 그래서 로컬에서 구동하는 Docker를 사용할 때는 종종 [Kitematic](https://kitematic.com)이라는 GUI 도구도 같이 사용했었는데, 원격 서버에서 구동되는 Docker를 사용할 때는 마땅한 도구를 찾지 못해 선택권 없이 CLI만 주구장창 사용했다. 사실 어지간한 도구들은 사용 전 이것저것 신경써야 할 것들도 많고 그 결과물도 썩 마음에 들지 않아 더 CLI를 선호했기도 한데, Portainer는 사전 설정도 간단하고 사용도 편해 상당히 만족하며 사용하고 있다.

속살
==========
날씨도 추우니 빠르게 속살을 까고 내려가자. 아래는 Portainer 공식 웹페이지에서 제공하는 설치 과정이다. 일반적인 단일 노드 환경에서 Docker를 사용한다면, 아래 두 명령어면 충분하다. Portainer가 사용하는 데이터 볼륨을 만들고, 호스트의  `/var/run/docker.sock` 을 컨테이너와 직접 연결한 후 9000포트를 통해 외부와 커뮤니케이션 시켜준다.

```
$ docker volume create portainer_data
$ docker run -d -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer
```
만약 다중 노드로 구성된 Docker Swarm 환경에서 Portainer를 사용하고 싶다면, 아래를 참고하면 된다.

```
$ docker service create \
--name portainer \
--publish 9000:9000 \
--replicas=1 \
--constraint 'node.role == manager' \
--mount type=bind,src=//var/run/docker.sock,dst=/var/run/docker.sock \
portainer/portainer \
-H unix:///var/run/docker.sock
```

Easy peasy lemon squeezy는 바로 이럴 때 사용하는 말이다. 컨테이너 구동이 끝났다면, 브라우저에서 해당 노드의 IP + 9000포트로 접속하면 된다.

<div align="center"><img src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/portainer-config.png?raw=true"/></div><br/>

간단한 계정 생성 과정을 거친 후, 위와 같이 Local과 Remote를 선택하는 창이 나오는데, 이름에서 유추할 수 있듯 Local은 Portainer가 설치된 시스템의 Docker 그 자체에 접근하는 것이고 (이를 위해 `docker run`시 `-v` 옵션을 통해 시스템의 `/var/run/docker.sock`을 마운트해 줬다.), Remote는 다른 곳에 위치한 Docker 머신을 관리할 수 있는 옵션이다.

대부분의 경우 Docker가 돌아가는 시스템 그 자체에 Portainer를 설치했을 것이니, Local을 선택하자. 원격지 Docker 머신을 사용하고 싶은 경우, Docker 데몬의 API 주소 (원격 서버 주소 + 2375 혹은 2376)로 Endpoint를 설정하면 된다.

<div align="center"><img src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/portainer-main.png?raw=true"/></div><br/>

모든 필요 설정을 마치면, 드디어 무언가 할 수 있는 콘솔창이 마중나온다. 이제 더 이상 설명할 건 없고, 하나하나 눌러보며 이전에 Docker CLI나 기타 GUI 도구로 진행했던 작업을 이어가면 된다. 혹시 단일 컨테이너로 실행했더라도 해당 시스템이 Docker Swarm의 Manager 노드일 경우 그 Swarm 클러스터의 전반적인 상태와 구조까지 시각화해서 보여주니 상당히 편하다.

한가지 아쉬운 점이라면, 각 컨테이너의 리소스 사용량을 그래프화시켜 실시간으로 보여주는 기능도 있으면서 정작 상태 이상 알림 기능은 없어 본격적인 모니터링 용도로는 사용하기 애매하다는 것인데, 추후 업데이트에서 대응할 계획이 있다고 운영진이 공식 입장을 내놓았으니 이를 기다려봐도 좋을 것 같다.

마치며
==========

사실 해가 가기 전 캐노니컬의 MAAS 솔루션 관련 포스트를 마무리하고 개고생한 2017년 넋두리 글이라도 쓸까 했는데, 생각보다 MAAS의 잔버그가 많아 모두 내년으로 미뤄버렸다. 그래도 어찌저찌 다른 글로 공백은 매웠으니 뭐 만족한다.

몇시간 남지 않은 2017년 모두 잘 마무리 할 수 있길 바란다. 
