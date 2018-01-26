---
layout: post
title:  "Docker Swarm으로 프로덕션 아키텍처 세우기"
date:   2018-01-26 18:25:45 +0900
categories: Docker
---

2010년대 중반 백엔드 업계는 일종의 '클러스터 붐'을 맞았다. Docker를 비롯해 비교적 사용이 간편하며 쉽게 규모를 확장할 수 있는 도구들이 등장하고, AWS나 GCP 등 대표적 클라우드 서비스 제공자들도 비슷한 기능들을 자체적으로 제공하기 시작하면서 전 세계 수많은 기업들이 자신들의 서비스를 클러스터화 시키기 시작했다. 당장 필자도 [이전에](https://kycfeel.github.io/2017/03/27/Docker-Swarm으로-고래엮기/) Docker가 기본으로 제공하는 Swarm 기능을 통해 간단히 컨테이너를 클러스터화 하는 방법에 대해 다뤄본 적이 있지 않은가.

그럼, 왜 그럴까? 모바일 기기와 초고속 인터넷의 폭발적인 보급으로 온라인 서비스 접속 수요도 그만큼 높아지고 있는데, 이를 쉽게 해결해줄 도구들이 잔뜩 등장하고 있으니 다들 물 만난 물고기가 된게 가장 표면적인 이유가 아닐까 싶다.

그런 흐름에 맞춰, 이번 글에서는 점점 더 높은 수요를 안정적으로 감당해야 하는 프로덕션 상황을 위해 로컬 Swarm 클러스터와 외부 퍼블릭 클라우드가 함께 동작하는 하이브리드 아키텍처를 직접 세워보려 한다.

알맞은 도구의 선택?
======
<div align="center"><img src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/kubevsswarm.png?raw=true"/></div><br/>

혹시 이 글을 읽기 전 Docker를 이용한 클러스터화, 더 나아가 서비스 인프라의 [오케스트레이션](https://en.wikipedia.org/wiki/Orchestration_(computing))을 위해 사용할 수 있는 도구들을 미리 알아본 적 있다면 백이면 백 그 중 어떤 것을 골라야 잘 골랐다고 소문이 날지 고민하고 있을 것이다.

대표적으로 Google의 [Kubernetes](https://kubernetes.io)와 Docker에서 자체적으로 제공하는 Swarm 두 개가 도마 위에 오를 것 같은데, Swarm은 Docker에서 아무것도 추가 설치하지 않아도 기본적으로 제공한다는 누구도 이길 수 없는 막강한 장점이 있다. 이는 관리 명령어나 서비스 구조 등도 단일 Docker 머신과 크게 다르지 않다는 뜻이 되기도 한다.

사실 Kubernetes가 이것저것 지원하는 것도 많고, 실제 사용하고 있는 기업도 더 많지만 한정된 시간과 자원 안에 비슷한 결과물을 만들기에는 Swarm이 최고라 생각한다. 당연히 Swarm도 나름 프로덕션급 오케스트레이션 도구임을 표방하고 있고, 점점 사용자도 늘어나고 있어 세팅의 간편함을 제외하더라도 딱히 어딘가 떨어지거나 하지 않는다. 우리 모두의 행복한 삽질을 위해, Swarm으로 밀고 나가보자.

Swarm 클러스터 구성
======

위에서 침이 마르도록 칭찬했지만, Swarm 클러스터 구성은 입에서 Easy Peasy Lemon Squeezy가 튀어나올 정도로 쉽고 편하다.

아래가 필자의 테스트 셋업인데, 본인의 상황에 맞게 적당히 응용하면 된다.

>빠르고 편한 세팅을 위해 [Subicura](https://subicura.com/2017/02/25/container-orchestration-with-docker-swarm.html) 님의 미리 빌드된 Vagrant 이미지를 활용했다. Swarm 클러스터 구성 방법 자체는 필자의 [이전 글](https://kycfeel.github.io/2017/03/27/Docker-Swarm으로-고래엮기/)을 참조 바란다.

<div align="center"><img src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/swarmarch.png?raw=true"/></div><br/>

`core-01` 은 Manager, 나머지 `core-02`와 `core-03`은 Worker 노드다. 프로덕션 상황을 감안하여 웹 서버와 DB가 함께 구동되는 WordPress를 테스트 애플리케이션으로 사용하고, 추가로 원활한 서비스를 위한 [traefik](https://traefik.io) 리버스 프록시와 관리 편의를 고려해 [Portainer](https://portainer.io)까지 같이 구동한다. 만약 본인의 팀 / 회사에서 구동 예정인 애플리케이션이 따로 존재할 경우, 중간 과정을 대체하거나 따로 수정해도 무방하다.

사전 작업
======

원하는 애플리케이션을 구동하기 전에, 위에 잠깐 언급한 리버스 프록시와 편의장치 먼저 세팅하는 편이 좋다. 특히 리버스 프록시의 경우 미리 설정하지 않으면 애플리케이션을 구동해도 외부와 통신할 통로 마련이 안되니, 당연하기도 하지만 말이다.

<div align="center"><img
 src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/portainer_logo.png?raw=true"/></div><br/>

Swarm 환경이 제대로 구축되었다면, Manager 노드에서 아래 명령을 통해 Portainer를 구동하자.

> Portainer에 관련된 자세한 정보는 [공식 홈페이지](https://portainer.io)와 필자의 [이전 글](https://kycfeel.github.io/2017/12/31/Portainer를-통한-Docker-초점잡기/)을 참고하면 된다.

```
docker service create \
--name portainer \
--publish 9000:9000 \
--replicas=1 \
--constraint 'node.role == manager' \
--mount type=bind,src=//var/run/docker.sock,dst=/var/run/docker.sock \
--mount type=bind,src=//opt/portainer,dst=/data \
portainer/portainer \
-H unix:///var/run/docker.sock
```

여기서 주의할 점이, Portainer는 무조껀 Manager 노드에서만 구동해야 한다는 점이다. 명령어를 읽어보면 알겠지만, 작동에 Docker 데몬과의 연결이 필요하다. Swarm에서의 Worker 노드는 말 그대로의 Slave일 뿐, 서비스나 클러스터의 제어 작업 등은 불가하니 이런 서비스의 경우 애초에 실행 자체가 막혀버린다.

이후 웹 브라우저를 통해 9000 포트로 접속하면 Portainer의 웹 콘솔 화면이 나온다. 물론 애플리케이션 구동에 필수는 아니지만, 전체 클러스터 구성 상태 확인이나 실행중인 서비스 제어 등 CLI보다 편리한 인프라 관리가 가능해지니 애용하면 삶이 조금 더 편해지지 않을까.

<div align="center"><img
 src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/traefik_logo.png?raw=true"/></div><br/>

 다음은 traefik이다. 물론 더 유명하고 당장 사용할 수 있는 리버스 프록시들은 많이 존재하지만, traefik은 가볍고 컨테이너 환경에 최적화된 힙스터같은 녀석이라 이번 기회에 손을 벌려봤다. 실제로 프로덕션 환경에서도 잘 사용되는 소프트웨어니, 크게 걱정하지는 않아도 된다. 물론 모험 없이 nginx 등을 사용해도 무방할 것 같다.

 아래 명령을 통해 traefik을 구동해보자. 마찬가지로 traefik도 Docker 데몬과의 연결을 필요로 하기에, Manager 노드에서만 구동된다.

 ```
 docker service create \
    --name traefik \
    --constraint=node.role==manager \
    --publish 80:80 --publish 8080:8080 \
    --mount type=bind,source=/var/run/docker.sock,target=/var/run/docker.sock \
    --network traefik-net \
    traefik \
    --docker \
    --docker.swarmmode \
    --docker.domain=traefik \
    --docker.watch \
    --web
```

명령어를 자세히 들여다보면 `traefik-net` 이라는 새로운 네트워크를 생성하는 것을 알 수 있다. 이는 기존의 `ingress` 네트워크와는 별도로, 구동할 애플리케이션이 리버스 프록시를 거쳐 외부와 통신하게 해줄 독립 네트워크다. 추후 애플리케이션 구동 시 따로 네트워크를 `traefik-net`으로 잡아줄 것이니, 미리 알고 넘어가자.

각설하고, 웹 브라우저의 8080포트로 접속하면 아래와 같은 웹 콘솔에 접근할 수 있다. 애플리케이션을 구동하면 `Providers` 메뉴에 관련 정보가 표시되고, `Health` 메뉴에서 HTTP 상태 코드나 평균 응답 시간이 예쁜 그래프를 입고 나타날 것이다.

<div align="center"><img
 src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/traefik_console.png?raw=true"/></div><br/>

잠깐 다음으로 넘어가기 전에, 우리 traefik이 미친 쥐인지, 애완용 햄스터인지 확인 한번 하고 넘어가면 정신건강에 좋을 것 같은데 말이다. traefik 메뉴얼에서 직접 권장하는 테스트용 웹 서비스인 [whoami]()를 통해 접속 테스트를 진행하자. 아래 명령어를 사용하면 된다.

```
docker service create \
    --name whoami0 \
    --label traefik.port=80 \
    --label traefik.frontend.rule="Host:<Manager_노드의_IP주소>"
    --network traefik-net \
    emilevauge/whoami"
```

<div align="center"><img
 src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/traefik-provider.png?raw=true"/></div><br/>

다시 traefik 콘솔을 살펴보자. `Providers` 메뉴에 방금 실행한 `whoami0` 서비스가 나타났을 것이다. 웹 브라우저로 80 포트에 접속할 경우 정상적으로 `whoami0` 서비스까지 닿을 수 있다.

<div align="center"><img
 src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/whoami0.png?raw=true"/></div><br/>

다행히 애완용 햄스터였다 :)

데이터 보존에 관하여
 =====

 이제야 원하는 애플리케이션을 구동하는 차례인가! 하면 아쉽게도 아직 아니다. 중요한 일이 조금 더 남았다. 많은 애플리케이션은 데이터를 저장할 DB, 그리고 기타 쌓이는 자료를 위한 별도의 저장공간을 요구한다. 이런 녀석들을 Stateful 애플리케이션 이라고도 하는 것 같던데, 컨테이너 기술은 별도 기록 / 의존 데이터가 필요없는 Stateless 애플리케이션을 우선적으로 고려해 디자인되었다. 물론 Docker가 제공하는 로컬 볼륨 기능이나, 컨테이너 내부 RW 공간이 존재는 하지만, 이 방법으로는 Swarm 상태에서 다른 컨테이너와 데이터 공유가 불가하고, 특히 내부 RW 공간을 사용했을 경우 기존 컨테이너를 삭제할 시 데이터까지 그대로 소멸되어 버린다는 치명적인 단점이 존재한다.

 이러한 문제는 Swarm이 등장한 초창기부터 지금까지 끊임없이 논의되고 있고, 사용자가 Swarm의 여러 장점들을 모두 포기하더라도 Kubernetes로 넘어가는 가장 큰 이유가 되기도 하는데(Kubernetes의 경우 자체적인 저장공간 오케스트레이션을 지원한다), 다행히 [Ceph](https://ceph.com)나 [Gluster](https://www.gluster.org)같은 분산 파일 시스템을 마운트해 사용하거나, [Flocker](https://github.com/ClusterHQ/flocker), [REX-Ray](https://rexray.thecodeteam.com)같은 Docker 볼륨 드라이버 / 플러그인이 개발되는 등 다양한 해결책이 마련되어 있다.

 이번과 같은 테스트, 또는 소규모 환경에서 Ceph 등을 사용하는 것은 배보다 배꼽이 더 큰 행위라 생각해 제외하고, Docker 플러그인 형태로 제공되는 REX-Ray와 AWS의 [S3](https://aws.amazon.com/ko/s3/), [RDS](https://aws.amazon.com/ko/rds/) 서비스를 이용해 어느 컨테이너에서나 접근할 수 있는 저장 공간, 그리고 DB를 구현하려 한다.

 > 사실 Flocker의 경우 공유 볼륨을 위한 별도 서버가 필요 없다는 장점 때문에 가장 먼저 사용해보려 했으나, 이를 개발한 [ClusterHQ](https://github.com/ClusterHQ) 팀이 최근에 셧다운되어 대부분의 자료가 사라지는 바람에 더 접근이 용의한 REX-Ray를 선택했다. 다행히 팀은 셧다운됬어도 Flocker의 개발은 계속 이어간다는 것 같으니 조만간 따로 사용해본 후 글을 남기도록 하겠다.

가장 먼저, 우리의 Swarm 시스템이 AWS 자원과 통신할 수 있도록 API 키를 발급받아야 한다. AWS는 [IAM](https://docs.aws.amazon.com/ko_kr/IAM/latest/UserGuide/introduction.html)이라는 접속인증 관리 서비스를 제공한다. 당장 우리가 필요한 것은 S3 저장소의 접근이니, 별도 사용자 추가를 눌러 `AmazonS3FullAccess` 권한을 먹여주면 된다. 물론 모든 서비스 접근이 가능한 root 키도 존재하지만, 보안을 생각해 사용하지 않는다.

계정 생성 후 나오는 엑세스 키는 잘 모셔두고, 이제 Swarm 시스템에 플러그인을 설치해 보자. REX-Ray는 여러 퍼블릭 파일 시스템에 연결할 수 있게 해주는 다양한 플러그인들을 제공하는데, 그 중 우리는 [s3fs](https://hub.docker.com/r/rexray/s3fs/) 플러그인을 사용할 것이다. 이름 그대로, AWS가 제공하는 가장 간단한 형태의 저장공간 S3를 서버에서 파일 시스템(fs)으로도 사용할 수 있게 해주는 녀석이다.

Master 노드에 접속하고, 아래 명령어로 플러그인을 설치하자.

```
docker plugin install rexray/s3fs \
S3FS_ACCESSKEY=<엑세스_키> \
S3FS_SECRETKEY=<시크릿_키> \
--grant-all-permissions=true
```

<div align="center"><img
  src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/aws_s3.png?raw=true"/></div><br/>

이제 AWS S3와 내 Swarm 시스템이 연결되었다. 만약 이전에 사용하던 S3 버킷이 있을 경우, `docker volume ls` 명령어를 통해 내 모든 버킷을 확인할 수 있을 것이다. 설령 그렇다고 해도 이미 사용중인 버킷을 덮어쓸 수는 없으니, 우리 Swarm만을 위한 새로운 버킷을 하나 만들어 보자.

AWS의 S3 웹 콘솔에 들어가면, 클릭 몇 번으로 간단히 버킷을 생성할 수 있다. 버킷 이름이나 기타 속성 등을 지정해준 후, 다시 Master 노드로 돌아오자. `docker volume ls` 명령어를 입력해 방금 생성한 버킷이 제대로 보이는지 확인했다면, 이제 DB를 생성할 차례다.

<div align="center"><img
  src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/aws_rds.png?raw=true"/></div><br/>

마찬가지로 AWS 웹 콘솔의 RDS 메뉴로 들어가면, MySQL을 필두로 하는 다양한 관계형 DB 인스턴스를 생성할 수 있다. MariaDB 또는 MySQL 중 취향에 맞는 것을 선택하고, DB 식별 이름과 계정 등을 설정해주면 된다. AWS에서 처음 DB를 생성하는 경우 약간의 보안 정책과 한글 인코딩 정책을 손봐줘야 하는데, 깔끔하게 정리된 [이 포스트](http://blog.saltfactory.net/how-to-start-mysql-on-aws/)를 참조하면 좋을 것 같다. 필자도 도움을 많이 받았다. 이후 생성된 DB의 세부정보를 확인하면 엔드포인트 URI가 적혀있는데, 모니터 한쪽에 잘 띄워두도록 하자. 곧 쓸 꺼다.

이제 모든 준비가 끝났다. 정말로 애플리케이션을 실행해 보자. 아래 명령어를 적당히 수정해 콘솔에 넣으면 된다.

```
docker service create \
--name wordpress \
--mount type=volume,source=<S3_버킷_이름>,target=/var/www/html \
-e WORDPRESS_DB_HOST=<RDS_엔드포인트_URI>:3306 \
-e WORDPRESS_DB_NAME=<RDS_식별_이름> \
-e WORDPRESS_DB_USER=<RDS_DB_유저명> \
-e WORDPRESS_DB_PASSWORD=<RDS_DB_비밀번호> \
-e WORDPRESS_TABLE_PREFIX=wp_ \
--label traefik.frontend.rule="Host:<접속에_사용할_URL>" \
--label traefik.port=80 \
--network traefik-net \
--constraint=node.role==worker \
--replicas 2 \
wordpress
```

하나하나 천천히 살펴보자. S3 버킷을 WordPress 서비스의 기본 데이터 경로인 `/var/www/html`과 연결하고, DB 호스트와 이름을 조금 전 RDS에서 생성한 그것으로 설정했다. `--label` 태그를 통해 traefik을 거쳐 우리 웹 서비스에 접속에 사용할 URL을 설정하고(테스트 환경에서는 Master 노드의 IP주소로 설정해도 무방), 80 포트를 허용했다. 통신할 네트워크도 물론 `traefik-net`으로 고정해줬다. 컨테이너가 Worker 노드에서만 구동될 수 있도록 규칙을 잡고, 기본적으로 2개의 컨테이너가 작동하도록 해두었다. 이미지는 최신 버전의 공식 `wordpress` 이미지를 사용한다.

더 이상 기다릴 건 없다. 웹 브라우저를 키고 80 포트로 접속하자. WordPress 설치 화면이 나온다면, 우린 모든 역경을 이겨냈다는 뜻이다. 이제 당신의 애플리케이션도 훌륭한 퍼포먼스와 높은 안정성을 자랑하게 되었다.

마지막으로 `/var/www/html`의 데이터가 정상적으로 S3 버킷에 저장되는지, DB의 접속은 제대로 되는지, 일부 노드가 다운되도 정상적인 서비스 접속이 가능한지 등 필수 기능 체크는 꼭 거치길 바란다.

뭔가 아쉬운데...
=====

필자의 다른 글을 살펴봤다면 알겠지만, 휴대전화 기지국도 직접 만들 정도로 외부 의존성을 가능하면 줄이려고 하는 편인데, 이번에는 투자하는 시간과 노력 대비 결과물을 생각해 사용이 간편한 AWS로 데이터 저장 파트를 대체했다. 의도하지는 않았지만 독자가 원할 경우 컴퓨팅까지 AWS EC2로 대체하는 과정이 좀 더 편해졌을 수도 있겠다.

그래도 뭔가 아쉬운 느낌이 든다. 아직 보충할 일거리가 조금 더 남았지 않을까. 애플리케이션의 SSL 통신이나 외부 클라우드 의존 없는 100% 로컬 아키텍처 대체법 등이 될 것 같은데, 추후 별도의 글로 작성할 생각이다. 글이 업로드되면 이곳에도 링크를 달아 두겠다.
