---
layout: post
title:  "라즈베리 파이로 Docker Swarm 클러스터 만들기"
date:   2017-08-24 02:04:20 +0900
categories: piluster
---

사실 처음부터 이 프로젝트를 하려고 한 건 아니였다. 혼자 시작한 프로젝트라면 자비로 모든 하드웨어를 사들여야 하는데, 필자가 그럴 돈이 어디 있는가. 그런데 우연히 학교에서 지원금 빵빵히 받아 프로젝트를 진행하던 친구가 기술 자문 겸 노가다꾼으로 초대해 주면서 열심히 노동하고 왔다. 사실 라즈베리 파이로 클러스터 만들어 가지고 노는 사람들은 이전부터 유튜브 등지에서 종종 봐 왔었고, 관심이 없는 것은 아니였기에 필자에게도 좋은 기회가 아니였나 싶다.

프로젝트의 목표는 저비용 + 저전력 + 작은 공간에서 분산 컴퓨팅 환경을 직접 구축하고 체험하는 것이다. 물론 이 작은 ARM 보드가 몇개 더 모인다고 해서 프로덕션 단계의 애플리케이션을 돌리는 건 어렵고, AWS 등의 상용 클라우드 서비스를 사용하는 편이 훨씬 더 간단하고 강력하지만, 직접 그와 비슷한 환경을 구축해본다는 것 자체에 의미를 뒀다.

piluster
========================
개발자의 본업은 이름 짓는 것이라고 종종 말해왔지만, 이번에는 도저히 시간을 투자할 수 없었다. 나는 한번의 작업을 위해 고속버스를 타고 슝슝슝 달려와야 하는 입장이라 도착도 하기 전에 피로가 한가득 쌓여있었고, 작업 중에도 역시나 생각하지도 않은 문제들이 펑펑 터져나와서 작명이라는 고도의 문학적이고 감성적인 일은 어려웠다. 결국 그냥 라즈베리 파이 + 클러스터 라고 해서 piluster로 퉁 쳤다. 당장 부를 이름이 없는데 어쩌겠는가 :).

![piluster_equidments](/_images/piluster_equidments.JPG)

사용한 장비는 위와 같다. 라즈베리 파이 3 5대, 짱짱하고 싼 마이크로 5핀 5개, 외부 전원 공급이 가능한 USB 허브, 그리고 적당한 공유기, 스위칭 허브 (물론 인터넷에서 USB OTG만을 통한 클러스터 구축 방법도 보긴 했지만, 여기서는 가장 범용적인 방법을 선택했다.)

각 파이에는 당시 최신 버전의 Raspbian Jessie LITE와 Docker Engine이 설치되었다. [이 곳](https://www.raspberrypi.org/downloads/raspbian/)과 [이 곳](https://www.raspberrypi.org/blog/docker-comes-to-raspberry-pi/)에서 설치 정보를 참고할 수 있다.

중간 과정
========================

사실 이전에 이미 Docker Swarm에 대해 [작성한 글](https://kycfeel.github.io/2017/03/27/Docker-Swarm으로-고래엮기/)이 있다. 그러기에 자세한 내용은 언급하지 않겠으나, 최소한의 구축 방법만 적으려 한다.

Leader로 삼을 라즈베리 파이에서 `docker swarm init` 명령어를 통해 Swarm 클러스터를 시작할 수 있다. 아마 아래와 같은 출력값이 나올 것이다.

```
Swarm initialized: current node (gjm2fxwcx29ha4rfzkw3zskvt) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token xxx \
    ip_address:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

저 중 `docker swarm join`으로 시작하는 출력값을 잘 복사해뒀다, worker로 삼을 파이들에 접속 후 입력하자. 그러면 고맙게도 알아서 차곡차곡 Swarm 클러스터에 노동자로 붙게 된다. 모두 끝났다면, Leader 노드에서 `docker node ls` 명령어를 입력해 제대로 클러스터가 구성되었는지 확인하자.

```
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
aaa *  rpi-01   Ready   Active        Leader
bbb    rpi-02   Ready   Active
ccc    rpi-03   Ready   Active
ddd    rpi-04   Ready   Active
eee    rpi-05   Ready   Active
```

위와 비슷하게 출력된다면 별 문제 없다는 뜻이다.

이제 클러스터가 묶였으니, 가장 중요한 애플리케이션을 올려볼 차례다. ARM용으로 빌드된 어떤 이미지라도 클러스터에 올릴 수 있지만, 필자는 편의상 직접 만든 [divergence](https://kycfeel.github.io/2017/05/24/내-자리비움을-발산하라) 웹 애플리케이션을 사용했다. 별도의 DB를 연결할 필요도 없어 여러모로 간편했기도 하고 말이다.

원하는 이미지를 worker 노드에 다운로드 받은 후, 아래 명령어를 통해 서비스를 생성해보자.

> 아래 명령어에 추가로 집어넣을 수 있는 옵션들은 [여기](https://docs.docker.com/engine/reference/commandline/service_create/)에서 확인할 수 있다.

```
docker service create --name 서비스_이름 -p 외부포트:내부포트 만든이/이미지_이름:1(컨테이너 갯수)
```

생성 후에는 `docker service ls` 명령어로 전체 서비스 목록을, `docker service ps 서비스_이름` 으로 원하는 서비스의 자세한 정보를 볼 수 있다.

```
ID            NAME      IMAGE              NODE     DESIRED STATE  CURRENT STATE            
id  서비스_이름.1  만든이/이미지_이름:1  rpi-01  Running        Running 3 seconds ago
```

위와 같이 정상적으로 서비스가 구동된 것을 확인한 후, curl이나 웹 브라우저 등을 통해 미리 지정한 포트로 접근을 해 보면 정상 가동되고 있는 것을 확인할 수 있다.

여기서 잠깐, 클러스터에 서비스가 딱 한 컨테이너로 실행되고 있는 상태에서 간단한 퍼포먼스 테스트를 해보았다. 사용한 툴은 [apib](https://github.com/apigee/apib). 소개 그대로 심플한 http 퍼포먼스 테스팅 툴이다. macOS에서 brew를 통해 간단히 설치할 수 있었다.

60초간 100명의 가상 접속자를 만들어 라즈베리 파이를 괴롭힌 결과, 아래와 같은 값을 얻었다.

<div align="center"><img src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/single_node_rpi.png?raw=true"/></div><br>

흠. 인터레스팅 하다. 무선 네트워크 간섭 등 여러가지 장애물이 껴 있어 신뢰도 높은 값이라고는 절대 지칭하지 못하겠지만, 추후 부하분산 상태에서의 값과 비교할 가치는 충분히 있어 보인다. 자 그럼, 바로 다음 값을 측정할 준비를 해보자.

방금 우리가 측정한 서비스는 컨테이너 1개로 돌아가고 있었다. 다시 말해, 클러스터를 묶지 않은 상태와 다를 것이 없다는 뜻이다. 다행히 Swarm 클러스터는 정말 간편하게 이미 돌아가는 서비스의 부하분산 설정을 할 수 있다. 아래 명령어를 따라와 보자.

```
docker swarm scale 서비스_이름=컨테이너_갯수
```

위에서 입력한 숫자만큼 내 서비스의 컨테이너 갯수가 늘어난다. Leader 포함 5개의 노드로 구성된 (Swarm 모드에서의 Leader 노드도 동시에 worker처럼 컨테이너를 돌릴 수 있다.) 클러스터에서 5를 지정하면 각 노드마다 하나씩의 컨테이너가 자동으로 자리잡게 된다. 그렇다고 한 노드에 하나의 컨테이너만 들어갈 수 있는 것은 아니라, 그 이상의 숫자도 얼마든지 지정 가능하다. 이 글에서는 편의를 위해 1노드 1컨테이너. 즉, 5개까지만 복제시켰다.

`docker service ls` 명령어 등을 통해 모든 컨테이너가 정상적으로 구동되고 있는 것을 확인했다면, 다시한번 퍼포먼스 테스트를 해볼 차례다. 위 단일 컨테이너 테스트와 동일한 조건값으로 진행하였다.

<div align="center"><img src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/multi_node_rpi.png?raw=true"/></div><br>

오호라. 확실히 평균 지연시간부터 최소 지연시간까지 모두 줄어든 것을 볼 수 있다. 위와 비교해 엄청나게 늘어난 초당 요청 횟수를 통해 여러 컨테이너로 부하분산이 되었다는 것을 확인할 수 있다. 조그만한 라즈베리 파이라도 클러스터링을 통해 확실한 퍼포먼스 향상을 꾀할 수 있다는 것을 직접 눈으로 봤다.

마무리
========================

위에서 언급했던 것처럼, 이제는 대부분의 경우에서 물리적 서버 환경을 직접 구축하는 것보다 상용 클라우드 서비스를 사용하는 것이 훨씬 경재적이고 강력한 시대가 되었다. 더군다나 비 x86 아키텍처에 성능도 떨어지는 라즈베리 파이 놀음은 전혀 쓸모없는 짓처럼 보일지도 모른다.

하지만, 아무리 훌륭하고 편한 솔루션들이 무수히 존재한다고 하더라도 이런 본질로 돌아가보는 행위가 가끔씩은 꼭 필요하다고 생각한다. 다 떠나서 손바닥 위의 작은 컴퓨터 보드 여러대가 네트워크를 타고 하모니를 이루는 광경을 직접 만들고 느끼는 건 정말로 황홀하지 않는가. 저렴한 가격과 적은 전기, 그리고 정말 적은 공간에서 분산 컴퓨팅을 직접 체험해본다는 취지는 훌륭하게 달성한 것 같다. 표면적인 데이터 측정은 끝났으니, 이제 더욱 Geek처럼 가지고 놀 일만 남았다.
