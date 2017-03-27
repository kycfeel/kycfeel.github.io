---
layout: post
title:  "Docker Swarm으로 고래엮기"
date:   2017-03-27 22:25:10 +0900
categories: 시스템 운영과 배포
---

Docker Swarm?
===================

<div align="center"><img src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/kawaiiwhales.gif?raw=true"/></div><br>

[서버 오케스트레이션](https://en.wikipedia.org/wiki/Orchestration_(computing)) 이라는 애매모호한 용어가 있다. 혹시 백엔드 쪽에 관심이 있는 분이라면 인터넷 어디선가 한번쯤 들어봤을 수도 있겠다. 이게 쓰는 사람에 따라 의미가 조금씩 달라지는 용어라 위에 '애매모호' 라고 표현을 헀는데, 기본적으로 *IT 인프라의 자동화* 라는 뜻을 가지고 있다. 오늘 다룰 Docker Swarm이 바로 "IT 인프라의 자동화"를 할 수 있게 도와주는 오케스트레이션 도구다.

당연히 Docker 컨테이너를 기반으로 오케스트레이션을 구현해주는 도구가 이것만 있는 것은 아니다. 다만, Docker Swarm은 무려 Docker의 내장 기능이라 기존에 사용하던 Docker의 다른 기능들, 그리고 명령어들과 눈물나게 잘 어우러지는 말 그대로의 오케스트라를 보여 주는 덕에 뒤도 안보고 선택하게 되었다. 그런데 이런 내장 기능들이 다 그러하듯, 정말 깔끔하게 필요한 기능들 이상은 제공하지 않는다는 단점 아닌 단점은 존재한다. 혹시 Docker Swarm 이상의 기능이 필요한 분들은 아쉽지만 뒤로가기를 눌러 구글로 돌아가길 바란다.

백문이 불어일견이라고, 아래 이미지를 보자. 기본이 되는 Docker Swarm 인프라의 구성도다.

<div align="center"><img src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/swarmstructure.png?raw=true"/></div><br>

아마 한눈에 어느정도 이해가 갈 것이다. 위의 `Swarm Manager`가 아래 Docker 데몬이 돌아가는 `Worker` 들을 중앙 관제하는 방식으로 구성되어 있는데, 관리자는 `Swarm Manager`에 터미널이나 Docker Compose 등으로 명령을 내려 아래 서버들까지 한번에 컨트롤 할 수 있다. 이러한 Master가 Slave들을 조련하는 중앙 관제 방식은 비단 Docker Swarm 뿐만 아니라 대부분의 오케스트레이션화가 진행된 인프라에 공통적으로 쓰이는 '정석'이니 지금 익숙해지면 나중에 편할 것이다.

설치하기
===================

아래 과정을 진행하며 [이 포스트](https://subicura.com/2017/02/25/container-orchestration-with-docker-swarm.html)를 많이 참고했다. 개인적으로 한국어로 작성된 Docker Swarm 관련 포스트 중 가장 잘 쓰여진 글이라 생각한다. 작성자님 감사합니다.

위에 올려둔 인프라 구성도 (스크린샷)를 그대로 따라갈 것인데, Docker Swarm이 설치될 테스트 환경은 [Vagrant](https://www.vagrantup.com)라는 VM 자동화 도구를 사용하여 구축하겠다. [필자의 GitHub에](https://) 모든 것이 준비된 Vagrant 패키지를 올려뒀으니, 다운로드받아 실행만 하시면 된다. 위 링크를 달아둔 포스트의 작성자님이 미리 만들어둔 것을 가져와 약간 수정한 것이다. 지금 사용한 Vagrant는 곧 따로 포스트를 작성할 예정이다. 완성되면 이곳에 연결해 두겠다.

구성도처럼 우리 Docker Swarm 클러스터의 Manager는 `core-01`가 될 것이다. `vagrant ssh core-01` 명령어로 `core-01`과 터미널 연결을 한 후, 아래 명령을 실행하여 Docker Swarm을 시작해 보자.

```
docker swarm init --advertise-addr 172.17.8.101
```

자, 순식간에 Docker Swarm 클러스터가 시작되었다. 아마 아래와 유사한 출력값이 나왔을 거다.

```
Swarm initialized: current node (y01bcscl168g9npunryt0fv5w) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-2hkjkxqnqt5dmizyr8lr59elc3dsuwoc5c2kgzbzont2pilcua-auleg4b3rtj3hkgudm4gqhj2o \
    172.17.8.101:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

Docker Swarm이 시작되었고 이 노드(`core-01`)이 Manager라는 사실, 그리고 Worker들은 어떻게 추가해야 하는지 설명해 주고 있다. 너무 순조롭고 친절해서 눈물이 나올 지경이다. 클러스터링이 이렇게 쉬운 일이 될줄 누가 알았겠는가.

눈물은 조금 이따 마저 흘리도록 하고, 일단 시키는대로 Worker 노드들도 추가해 보자. 나머지 노드 (`core-02`, `core-03`, `core-04`) 들에 접속하여 위 `docker swarm join` 명령어를 복사 & 붙혀넣기 해보자. `This node joined a swarm as a worker.` 이라는 짧고 강력하게 정상적으로 Worker가 되었다는 사실을 알려준다.

이제 `core-01`로 돌아가 `docker node ls` 명령어를 입력해보자. 이 Swarm 클러스터에 어떤 노드들이 있는지 한눈에 확인할 수 있다. 방금 추가한 Worker 노드들이 정상적으로 표시된다면 제대로 진행한 것이다.

아직 할 일이 하나 남았다. 우리가 `core-01`로 돌아온 것은 클러스터의 상태를 확인하려는 목적도 있었지만, 위의 구성도에 보이는 것처럼 `Manage` 노드를 하나 더 추가하려 온 것이다. 혹시 처음 `docker swarm init` 명령어로 클러스터를 시작했을 때, 출력값 맨 아래에 Manager를 추가하는 법도 알려줬던 것을 눈채챘을지 모르겠다. `docker swarm join-token manager` 명령어를 입력하면 Worker 추가 명령어와 마찬가지로 아래처럼 인증 토크값과 함께 Manager 추가 명령어를 알려준다.

```
To add a manager to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-2hkjkxqnqt5dmizyr8lr59elc3dsuwoc5c2kgzbzont2pilcua-7g5mef5gjj8zlr6kesyzqk83i \
    172.17.8.101:2377

```

이걸 복사한 후, `core-05`로 접속하자.

`core-05`에서 위 명령어를 붙혀넣기만 하면 "This node joined a swarm as a manager." 라는 메시지와 함께 Manager 추가도 끝난다. 축하한다. 3대의 Worker 노드, 그리고 2대의 Manager 노드로 구성된 고가용 클러스터가 완성되었다. 이제 아래로 넘어가 이 덕심을 자극하는 클러스터로 무언가 돌려 보도록 하자.

애플리케이션 올려보기
===================

Docker Hub에서 웹 애플리케이션 (Flask 데모)을 하나 끌어와 클러스터에 올려보자. Manager 노드에 접속하여 아래 명령어를 쳐보자. Docker Hub에 올라가 있는 `p0bailey/docker-flask`을 끌어와 `80:80` 포트로 실행하는 명령어이다.

```
docker service create --name flaskontesting \ # flaskontesting에는 원하는 이름 입력
  -p 80:80 \ # 연결할 포트
  p0bailey/docker-flask # Docker 이미지의 이름
```

별다른 경고 없이 무언가 알 수 없는 값 (필자의 경우 `3pyk3b8bo9t01r8ct2fhtregk`)이 출력되었다면 정상적으로 서비스가 실행되고 있다는 뜻이다. 방금 출력된 값은 우리가 이번에 실행한 서비스의 ID값이다. `docker service ls` 명령어로 현재 실행되고 있는 서비스를 확인할 수 있다.

조금 더 자세한 정보를 원한다면 `docker service ps 서비스이름` 명령어를 입력해보자. 필자의 경우 아래같은 출력값이 나왔다.

```
ID            NAME              IMAGE                         NODE     DESIRED STATE  CURRENT STATE           ERROR  PORTS
1iqh5ti6ditt  flaskontesting.1  p0bailey/docker-flask:latest  core-05  Running        Running 10 minutes ago       
```

현재 서비스의 상태, 서비스의 ID값, 이름, 사용하고 있는 이미지, 그리고 서비스가 돌아가고 있는 Swarm 노드 등의 정보가 자세히 출력된다. 그런데 여기서 궁금한 점이 생긴다. 왜 Manager 노드인 `core-05` 에서 서비스가 돌아가고 있는 것으로 나오고, 또 접속은 어떻게 해야 하는 건가? 정말 `core-05`의 IP주소를 일일히 치고 들어가야 하는 걸까?

필자도 처음에는 어버버 했었다. 지금부터 하나하나 설명할 것이니 너무 어려워 하지 마라. 먼저, 기본적으로 Manager는 단순히 Worker를 관리하는 매니저의 역할만 하는 것이 아니라, 본인이 일도 같이 하는 'Manager + Worker'의 상태이다. 따라서 설령 내 Swarm 클러스터에 Worker 노드가 하나도 없다고 해도 정상적으로 애플리케이션을 실행할 수 있다. 물론 Manager 노드가 매니저의 일만 하도록 지시할 수도 있는데, [이 글](http://stackoverflow.com/questions/39144752/docker-swarm-managers-acting-nodes) 을 참고해 보길 바란다. Manager 노드가 Worker의 일도 같이 한다는 것과, 추가로 Manager 노드에게 매니저의 일만 하도록 설정하는 법이 잘 설명되어 있다.

두번째로, 접속할 때 `core-05`의 IP를 일일히 확인하고 입력할 필요는 전혀 없다. Docker Swarm은 기본적으로 [Ingress](https://docs.docker.com/engine/swarm/ingress/)라는 오버레이 메시 네트워크를 사용한다. 말이 어려운데, 그냥 Swarm 노드간 가상의 네트워크를 만들어 서로 알아서 연결된다는 소리다. 그리고 이 네트워크에 연결된 노드들에는 신기한 일이 일어나는데, 만약 서비스를 실행하며 80 포트를 열었다면, 모든 노드의 80 포트가 같이 열리는 것이다. 이게 무슨 소리냐면 일단 서비스가 돌아만 가면 `core-01`로 접속하던 `core-02`로 접속하던 같은 서비스에 연결된다는 말이다.

정말 그럴까? 직접 접속해보자. 위에 말했듯 필자의 Flask 데모 애플리케이션은 `core-05` (172.17.8.105) 에서 돌아가고 있다. 먼저 `core-01` (172.17.8.101) 로 접속을 해 보겠다.

<div align="center"><img src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/core01test.png?raw=true"/></div><br>

문제 없이 접속이 된다. 그럼 `core-02` (172.17.8.102) 는 어떨까?

<div align="center"><img src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/core02test.png?raw=true"/></div><br>

마찬가지로 아무 문제 없이 접속이 되는 것을 확인할 수 있다. 놀랍지 않은가? 이것은 서버 오케스트레이션이 추구하는 일 중 하나이기도 하다. 수백 ~ 수천대의 서버들이 작동하는데 그 중 웹 서버는 대체 어디서 돌아가는지 관리자가 알고 있어야 한다면 상상만 해도 끔찍할 것이다. 오케스트레이션이 잘 진행된 환경이라면 그 모든 과정을 인프라가 스스로 결정하고 사용자에게 매끄럽게 연결해 줄 것이다. 관리자의 흰머리와 피부 상태도 훨씬 좋아지지 않을까.

부하 분산하기
===================

그럼 잠깐 다른 상황을 가정해 보겠다. 우리 웹 애플리케이션의 매력을 알아챈 사용자들이 미친듯이 몰리는 바람에, 서버가 골로 가기 직전의 지옥도가 펼쳐지고 있다고 해보자. 이럴 때는 서비스가 돌아가는 컨테이너를 여러개 만들어 부하를 분산시킬 수 있다. Docker Swarm 답게 역시 간단한 명령어 조금으로 구현이 가능하다.

```
docker service scale 서비스이름=5
```

위 명령어로 지정한 서비스의 컨테이너를 5개 더 만들어 부하 분산을 할 수 있다. `서비스이름 scaled to 5` 라는 메시지가 출력됬을 것이다. 혹시 내 Swarm 클러스터의 노드가 5개 미만이더라도 한 노드에 컨테이너 2개 이상을 배치해 문제 없이 가동시킬 수 있다. `docker service ps 서비스이름` 명령어를 입력해서 진행 상태 등을 확인할 수 있다. 아래는 필자의 출력값이다.

```
7mgur7xeylq2  flaskontesting.1      p0bailey/docker-flask:latest  core-05  Running        Running 45 seconds ago                                          
owqgn5g7ognl  flaskontesting.2      p0bailey/docker-flask:latest  core-01  Running        Running 4 minutes ago                                       
xwwn6blyxs8r  flaskontesting.3      p0bailey/docker-flask:latest  core-02  Running        Running 52 seconds ago                                      
ypk6ht23mig3  flaskontesting.4      p0bailey/docker-flask:latest  core-03  Running        Running 4 minutes ago                                                                   
itv3ms2jm3i2  flaskontesting.5      p0bailey/docker-flask:latest  core-04  Running        Running 3 minutes ago                                       
```

잘 돌아가고 있는 것을 확인할 수 있다. 혹시라도 일부 컨테이너가 시작을 실패한 경우 `docker service update 서비스이름 --force` 명령어로 강제 재시작 시킬 수 있다. 물론 명령어는 `update` 이지만 아무 변경점 없이 `--force` 깃발을 달아 재시작만 할 수 있다. 자세한 내용은 [여기](https://github.com/docker/docker/issues/24413) 참조.


일단 Swarm을 통한 부하 분산까지 다뤄봄으로서 기본적인 개념과 사용법은 익혔다. 물론 위 내용만으로도 정말 많은 일을 할 수 있겠지만, 조금 더 깊히 내려가보도록 하겠다. 여기까지 해서 잠깐 글을 끊고, 다음 포스트에 이어서 '심화' 과정을 다뤄보겠다. 너무 글이 길어지면 몸도 마음도 지치는 법이다. 바로 다음 글로 넘어가지 말고 커피 한잔씩 하고 오도록 하자. 모두 수고하셨다.
