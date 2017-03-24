---
layout: post
title:  "Docker Swarm으로 고래엮기"
date:   2017-03-23 16:25:22 +0900
categories: Docker
---

Docker Swarm?
===================

<div align="center"><img src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/kawaiiwhales.gif?raw=true"/></div><br>

[서버 오케스트레이션](https://en.wikipedia.org/wiki/Orchestration_(computing)) 이라는 애매모호한 용어가 있다. 혹시 백엔드 쪽에 관심이 있는 분이라면 인터넷 어디선가 한번쯤 들어봤을 수도 있겠다. 이게 쓰는 사람에 따라 의미가 조금씩 달라지는 용어라 위에 '애매모호' 라고 표현을 헀는데, 기본적으로 *IT 인프라의 자동화* 라는 뜻을 가지고 있다. 오늘 다룰 Docker Swarm이 바로 "IT 인프라의 자동화"를 할 수 있게 도와주는 오케스트레이션 도구다.

당연히 Docker 컨테이너를 기반으로 오케스트레이션을 구현해주는 도구가 이것만 있는 것은 아니다. 다만, Docker Swarm은 무려 Docker의 내장 기능이라 기존에 사용하던 Docker의 다른 기능들, 그리고 명령어들과 눈물나게 잘 어우러지는 말 그대로의 오케스트라를 보여 주는 덕에 뒤도 안보고 선택하게 되었다. 그런데 이런 내장 기능들이 다 그러하듯, 정말 깔끔하게 필요한 기능들 이상은 제공하지 않는다는 단점 아닌 단점은 존재한다. 혹시 Docker Swarm 이상의 기능이 필요한 분들은 아쉽지만 뒤로가기를 눌러 구글로 돌아가길 바란다.

백문이 불어일견이라고, 아래 이미지를 보자. 기본이 되는 Docker Swarm 인프라의 구성도다.

<div align="center"><img src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/swarmexample.jpg?raw=true"/></div><br>

구성도가 깔끔해서 아마 한눈에 어느정도 이해가 갈 것이다. 위의 `Swarm Manager`가 아래 Docker 데몬이 돌아가는 `node` 들을 중앙 관제하는 방식으로 구성되어 있는데, 관리자는 `Swarm Manager`에 터미널이나 Docker Compose 등으로 명령을 내려 아래 서버들까지 한번에 컨트롤 할 수 있다. 이러한 Master가 Slave들을 조련하는 중앙 관제 방식은 비단 Docker Swarm 뿐만 아니라 대부분의 오케스트레이션화가 진행된 인프라에 공통적으로 쓰이는 '정석'이니 지금 익숙해지면 나중에 편할 것이다.

직접 사용해보기
===================
