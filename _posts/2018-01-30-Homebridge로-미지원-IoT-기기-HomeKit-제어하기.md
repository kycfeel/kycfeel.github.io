---
layout: post
title:  "Homebridge로 미지원 IoT 기기 HomeKit 제어하기"
date:   2018-01-30 22:42:45 +0900
categories: System Engineering
---

AliExpress에서 몇개월 전 할인에 혹해 충동구매한 Yeelight의 스마트 컬러 LED 전구가 해를 넘어서도 책장 위에서 먼지만 쌓이고 있는 것을 보고, 드디어 사용할 마음을 먹었다. IKEA에서 싸고 이쁜 플로어스탠드 하나를 주문하고 잠깐 생각해보니, 조명의 제어가 걱정되기 시작했다.

물론 Yeelight 측에서 기본적으로 제공하는 앱을 통해 제어가 가능하지만, 조금 더 힙하고 맛깔나는 방법을 원했다. 마치 Apple의 유튜브 광고에 나오는 [이런](https://www.youtube.com/watch?v=4nbhfrQfRRE) 것을 말이다. 아쉽게도, 내가 구매한 이 조명은 Apple에게 어떠한 HomeKit 인증도 받지 못했지만, 지금이 어떤 세상인가. [Homebridge](https://github.com/nfarina/homebridge)라는 HomeKit 에뮬레이션 서버가 필자를 구재할 것이니.

나는 편법이 좋아요
=========

Homebridge는 Node.js 기반으로 만들어진 홈킷 에뮬레이션 서버다. 다시 말해, Apple HomeKit 액세서리 인증을 받지 못한 IoT 기기라도 이 Homebridge를 통해 HomeKit으로 제어할 수 있다는 소리다. IoT를 좋아하는 친구에게 이전에 잠깐 말은 들어본 적 있지만, 직접 검색해보니 생각보다 정말 다양한 플러그인들이 존재했다. 물론 필자가 사용하려 하는 Yeelight 컬러 전구를 위한 플러그인도 있다. 망설일 게 뭔가. 당장 실천에 옮기자.

>이 글에서는 Yeelight만 예시로 다루지만, 플러그인만 존재한다면 비슷한 과정을 거쳐 얼마든지 다양한 기기를 연결할 수 있다. 필자가 다른 기기들도 가지고 있었다면 다뤄봤을 수도 있겠지만, 안타깝게도 필자의 주머니는 얇다 :(

필자는 추가적인 자원 소모를 막기 위해, 이미 로컬넷에서 작동하고 있던 라즈베리 파이에 Docker 컨테이너 형태로 Homebridge를 구동했다. 미리 빌드된 이미지가 존재하니, 아래 명령어를 통해 바로 컨테이너를 띄워보자. 네이티브 구동을 원할 경우 위 Homebridge GitHub 링크를 타고 들어가 README를 참고하면 된다.

>라즈베리 파이가 아닌 일반 x86 기기에서 구동 시, 이미지 이름 뒤의 `:raspberry-pi`만 지우면 된다. 알아서 x86용 `:latest` 이미지를 받아올 것이다.

```
docker run \
  --net=host \
  --name=homebridge \
  -e PUID=1001 -e PGID=1001 \
  -e TZ=Asia/Seoul \
  -v </호스트/연결/경로>:/homebridge \
  oznu/homebridge:raspberry-pi
```

홈 앱과의 연결을 위해 컨테이너는 호스트 네트워크에 직접 연결시켜 줬다. PUID와 PGID는 기본 Docker 유저, 그룹을 뜻하는 `1001`, 타임존은 서울로 설정하면 된다. 편리한 Homebridge 설정을 위해 원하는 호스트의 특정 경로를 컨테이너의 `/homebridge`와 마운트하자.

컨테이너가 정상적으로 켜졌다면 아래와 같은 QR 코드가 콘솔창에 나타날 것인데, 애플 기기에서 홈 앱을 실행해 액세서리 추가 메뉴에서 이 비슷한 QR코드를 스캔하자.

<div align="center"><img src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/homebridge-qr.png?raw=true"/></div><br/>

Homebridge가 홈 앱에 정상적으로 추가되었다면, 다시 콘솔로 돌아와 Yeelight 플러그인을 설치해야 한다. 다행히 복잡하지는 않고, `docker exec homebridge yarn add homebridge-yeelight` 한 줄로 직접 컨테이너에 명령을 보내 플러그인을 간단히 설치할 수 있다.

플러그인도 설치했으니 이제 Yeelight를 정상적으로 인식할 수 있도록, 장비 목록에 추가만 해주면 된다. `/homebridge` 경로와 마운트했던 호스트 위치로 이동해보자. 이 경로에는 Homebridge 앱의 모든 설정 파일들이 동기화된다. `config.json` 파일을 열어, 아래와 같이 편집하자. 상단의 브릿지와 관련된 설정은 무시하고, 하단 `platforms`만 건들면 된다.

```
    "platforms": [
        {
            "platform" : "yeelight",
            "name" : "yeelight"
        }
    ]
```

여기까지 하면 다 된 것 같은데, 귀찮은 과정이 하나 더 남아있다. Yeelight가 Homebridge와 통신할 수 있게, Yeelight의 개발자 모드를 켜줘야 한다. 하다하다 이젠 전구의 개발자 모드라니. 직접 타이핑하면서도 뭔가 어색하다.

Yeelight 앱을 통해 전구와 연결된 스마트폰에서, 하단 설정 메뉴를 누르면 '긱 모드'라는 버튼이 하나 보일 것이다. 살짝 밀어 켜주면 된다. 이전 버전에서는 개발자 모드라고 제대로 출력이 되었는데, 나름 센스를 부려 Geek 모드로 개명을 시킨 것 같다. 맞는 말이긴 하다. 전구에서까지 개발자 모드를 키고 있는 사람들이 Geek이 아니고 무엇이겠는가.

>필자의 경우 처음부터 손에 바로 잡히는 거리에 Android 기기가 있어 Android용 Yeelight 앱에서 진행했는데, 해외 포럼에서 iOS용 Yeelight 앱은 긱 모드 버튼이 보이지 않는다는 말도 오가는 것 같다. 진행에 참고하길 바란다.

<div align="center"><img src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/yeelight-geekmode.jpg?raw=true"/></div><br/>

여기까지 따라왔으면, 정말 다 끝났다. 전구를 켜둔 상태로 아까 설정을 마친 Homebridge 컨테이너를 재시작하면, 자동으로 Yeelight를 감지해 서로 연결될 것이다. 못 미더우면 콘솔 로그를 들여다보자. 애플 홈 앱에서도 알아서 새로 추가한 전구가 보일 건데, 혹시라도 계속 팝업이 안될 경우 Homebridge를 홈 앱에서 지웠다 다시 추가해보자. 다시 잘 보일 것이다.

>홈에서 지운 후 다시 추가 시도 시 '이미 추가된 액세서리입니다.' 와 같은 메시지가 뜨며 진행이 막힐 수도 있다. 이럴 경우 `/homebridge` 경로의 `accessories` 폴더와 `persist` 폴더를 삭제한 후 다시 Homebridge를 시작해보자. 필자도 이렇게 오류를 해결했다.

시리야, 불 꺼.
=========

<div align="center"><img src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/home-main.jpg?raw=true"/></div><br/>

<div align="center"><img src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/home-color.jpg?raw=true"/></div><br/>

Yeelight 앱에서와 같이, 홈 앱과 시리를 통해 전원부터 빛 색깔, 밝기까지 모두 컨트롤 가능한 것을 확인할 수 있다. 여기서 혹시 외부 네트워크에서의 기기 원격 제어는 어떻게 하냐 물어본다면, iPad나 Apple TV를 구매하시라고 미리 답을 드린다.

농담이 아니라, 이 두 기기는 Apple이 직접 권장하는 (그리고 유일한) HomeKit 중계 서버로 동작하며, 여러가지 자동화나 외부 연결도 이 녀석들을 통해야만 가능하다. 배보다 배꼽이 더 커질 것 같다. 아직 필자는 스탠드 조명 하나만 연결되어 있기에 로컬넷 한정 제어로도 만족한다. 여차하면 구형 iPad를 가지고 있으니 활용해볼 수도 있을 것 같고 말이다.

무엇보다 이제 당당히 외칠 수 있다.

*시리야, 불 꺼.*
