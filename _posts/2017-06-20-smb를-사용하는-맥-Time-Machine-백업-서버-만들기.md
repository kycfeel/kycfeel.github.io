---
layout: post
title:  "SMB를 사용하는 macOS Time Machine 백업 서버 만들기"
date:   2017-06-20 15:20:10 +0900
categories: Linux
---

<div align="center"><img src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/timemachine.jpg?raw=true"/></div><br>

애플의 macOS는 [Time Machine](https://support.apple.com/ko-kr/HT201250) 이라는 환상적인 백업 솔루션을 지원한다. '타임 머신' 이라는 이름에 걸맞게, 각 백업 시점의 스냅샷을 찍어 나중에 언제라도 원하는 시점으로 슝 돌아갈 수 있는 놀라운 녀석이다. 문제는 또 애플 아니랄까봐, 본격적으로 기능 사용을 위해서는 [요런](https://www.apple.com/kr/airport-time-capsule/) 비싸고 성능 떨어지는 물건을 사야 한다는 거다. ~~애플의 Airport 라우터 시리즈는 정말 최악이다. 가격은 하늘을 찌르는데 유저 편의는 손톱만큼도 신경 쓰지 않는다. NAT 규칙 하나를 변경하기 위해 풀 리부팅이 필요하다고 하면 믿겠는가.~~

저걸 살 돈이면 이름있는 브랜드의 새로운 4K 모니터를 마련할 수도 있다. 그럼 어떻게 할까? 백업을 하지 말아야 할까? 아니다. 모든 시스템에는 예외없이 백업이 필요하다. 지갑을 여는 대신, 공돌이 기질을 아낌없이 뽑아내 DIY 원격 백업 시스템을 만들어 보자.

준비물
===================

필자는 RAID 5 디스크가 연결된 Ubuntu 16.04 LTS 로 동작하는 서버를 사용했다. 비슷한 사용 환경을 구축할 수 있는 어떤 Linux 배포판이나 디스크라도 상관 없다. 다만, 어디까지나 백업 데이터가 저장되는 서버인 만큼 안정성을 제 1순위로 생각하여 잘 준비하도록 하자.

그리고 시작하기 전 "왜 애플 기기의 백업 데이터 전송인데 AFP가 아니라 SMB를 사용하지?" 라고 의문이 들 수도 있을 것인데, 애플은 이미 OS X 매버릭스 시절부터 기본 파일 전송 프로토콜을 AFP에서 SMB2로 변경했다. 프로토콜별 퍼포먼스 차이를 자세히 꿰고 있지는 않지만, 기존 AFP보다 SMB2가 의미 있을 정도로 빠르고 효율적이라는 판단이 나와 변경을 감행한 것 같다. 물론 최신 macOS High Sierra에서도 레거시 AFP의 지원은 계속되고 있어 사용 자체는 가능하지만, 이왕 새로 만드는데 더 뛰어난 기술을 사용하는 편이 좋지 않겠나.

Samba 서버 세팅하기
===================

이미 유닉스 계열 OS들의 파일 공유 표준은 `Samba`가 자리를 차지하고 있다. 알고 있을수도 있지만, 윈도우에서 사용되던 SMB (Server Message Block)을 다시 구현해 오픈소스화 시킨 소프트웨어다. 위에서 파일 공유의 표준 자리를 차지하고 있다고 한 만큼 이미 많은 Linux 배포판의 패키지 관리 도구들에서 빠르게 설치할 수 있도록 준비를 마쳐뒀다.

```
sudo apt-get install samba samba-common-bin
```

그럼 바로 Ubuntu 배포판 기준으로, 위 명령어를 통해 필요한 소프트웨어를 먼저 설치하도록 하자.

다음으로 SMB를 통한 접속에 사용할 계정을 만들 거다. 기존 유닉스 계정과는 별개로 취급해야 하니 나중에라도 햇갈리지 말 것.

```
sudo smbpasswd -a 유저명
```

위 명령어로 진행할 수 있다.

이제 `Samba` 서버의 설정을 조금 만져주면 된다. 정말 별로 안 남았다. 설정 파일의 경로는 `/etc/samba/smb.conf` 다. 선호하는 에디터로 해당 파일을 열었으면, 아래를 참고해 주시길 바란다.

```
#======================= Global Settings =======================

[global]

## Browsing/Identification ###

# Change this to the workgroup/NT-domain name your Samba server will part of
   workgroup = WORKGROUP

# server string is the equivalent of the NT Description field
	server string = %h server (Samba, Ubuntu)

# Windows Internet Name Serving Support Section:
# WINS Support - Tells the NMBD component of Samba to enable its WINS Server
#   wins support = no

# WINS Server - Tells the NMBD components of Samba to be a WINS Client
# Note: Samba can be either a WINS Server, or a WINS Client, but NOT both
;   wins server = w.x.y.z

# This will prevent nmbd to search for NetBIOS names through DNS.
   dns proxy = no

...

```

아마 위와 같은 설정 스크립트가 쫘르륵 나올 것인데, 분량에 겁먹지 말고 찬찬히 읽어보면 전혀 어려울 건 없다. 나머지는 딱히 안 건들어도 되고, 아래의 내용만 `[global]` 태그 아래에 추가해주자.

```
path = 내가_공유할_저장경로
valid users = 유저명
writable = yes
```

공유 프로토콜을 수동 설정하고 싶을 경우 (SMB의 버전 등), 아래의 내용을 넣으면 된다.

```
min protocol = SMB1
max protocol = SMB3
```

위는 최소 사용 프로토콜을 `SMB1`, 최대 프로토콜을 `SMB3`로 설정한 예시다. 본인의 입맛에 맞게 적당히 수정해서 사용하자.

쉽다. 사실 이 설정 파일에서 공유 경로의 접근 퍼미션도 설정할 수 있긴 한데, 개인적으로는 `chown` 등의 명령어를 통해 따로 설정하는 것을 추천한다. 이런 중요한 설정은 특정 소프트웨어에 의존성이 생기면 영 좋지 않다.

모두 저장하고 나왔으면, 이제 아래 명령어로 `Samba` 서버를 가동해보자.

```
sudo systemctl enable smbd
sudo systemctl start smbd
```

방화벽을 사용중일 경우, SMB 프로토콜의 기본 포트인 `445`를 허용해주자. Ubuntu 기준 `sudo ufw allow 445` 명령어다. 추가적으로 서버 상단에 공유기 등이 달려있다면 알아서 허용해주도록 하자.

이제 접속할 클라이언트 장비에서 우리 서버로 접속 테스트를 해보자. macOS 기준 아래 독 바의 Finder 아이콘 오른쪽 클릭 -> 서버에 연결 -> 나오는 주소창에 `smb://서버주소` 로 접속 가능하다. 아래 이미지를 참고하자.

<div align="center"><img src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/smbconnection.png?raw=true"/></div><br>
계정 정보를 물어볼 때는 처음에 설정했던 SMB 전용 계정을 입력하자. 유닉스 계정과 햇갈리지 말고.

짠. 제대로 진행했다면 아마 정상적으로 접속이 될 것이다. 이제 SMB를 사용하는 파일 서버는 완성되었다. 그럼 Time Machine 백업은 어떻게 할까?

Time Machine 백업 이미지 만들기
===================

사실 OS X 매버릭스 시절부터 기본 파일공유 프로토콜이 SMB2로 변경되었다고 해도, 그대로 SMB 파일 서버를 Time Machine 백업 서버로 지정할 수는 없다.(또는 어렵다. 구글링을 열심히 해 봐도 관련 정보가 정말 드물다.) 그래서 우리는 약간의 꼼수를 쓸 꺼다. 바로 별도의 백업 이미지를 만들어 SMB 서버에 담는 것. 이렇게 하면 비단 SMB뿐만 아니라 어떤 종류의 파일공유 프로토콜을 사용하는 서버에도 Time Machine 백업이 가능하다. 마음껏 응용해주면 좋겠다.

이미지를 만들기 전, 아래 명령어로 지원하지 않는 네트워크 드라이브로의 Time Machine 백업을 허용해주자. 사실 '지원하지 않는 네트워크 드라이브' 라는 말과 달리 여전히 AFP를 통한 파일 서버가 아니면 아무 설정 없이 Time Machine 유틸리티에 드라이브가 뜨지 않는다. 필자도 처음에 기대했었는데 그게 아니더라. 그래서 우리가 별도의 이미지를 만드는 것이다. 이 명령어를 입력해줘야 이미지를 통한 꼼수 백업이라도 가능해지니 눈물을 흘리며 엔터를 누르자.

```
sudo defaults write com.apple.systempreferences TMShowUnsupportedNetworkVolumes 1
```

이제 정말로 이미지를 생성하기 위해 아래의 명령어를 입력하자. 입력하기 전 `cd` 명령어로 생성을 원하는 경로로 이동 후 진행하면 더 좋다.

```
sudo hdiutil create -size 300g -type SPARSEBUNDLE -fs "HFS+J" TimeMachine.sparsebundle
```

명령어를 읽어보면 알겠지만, 직접 최대 용량과 파일 시스템 등을 지정해줄 수 있다. 적당히 본인에게 맞게 수정하도록 하자. 그리고 혹시라도 300GB의 이미지를 생성하면 바로 300GB를 떡하니 차지하고 있는 것은 아닐지 걱정하지 않아도 된다. macOS의 `sparsebundle`은 일종의 동적 디스크라, 용량이 들어올수록 점점 늘어난다. 다시 말해, 처음에는 작다는 말이니까 용량 걱정은 하지 말자.

이제 이미지가 정상적으로 모습을 보일 것인데, 이걸 아까 연결해둔 SMB 서버에 올리기만 하면 끝이다. Finder 창을 연 후 쭉 끌어 놓아주자.

전부 다 올라갔다면, 이제 서버 위에 있는 이미지를 더블클릭해 마운트해 보자. 약간의 시간이 지나면 둥 하는 소리와 함께 좌측에 내 이미지가 외장 디스크처럼 마운트되어 있는 것을 볼 수 있다.

이제 백업할 시간이다. 다시 터미널을 열어, 아래 명령어로 Time Machine 백업 경로를 강제 지정해주자. 이렇게 안 해주면 인식을 못하더라. 슬프게.

```
sudo tmutil setdestination /Volumes/TimeMachine
```
이제 Time Machine 유틸리티를 실행하면 정상적으로 우리의 이미지가 안녕 하고 손을 흔들어줄 것이다. 남은 것은 백업 단추를 누르고 커피라도 한잔 마시고 오는 것. 백업이 끝나면 일반적인 외장 디스크를 마운트 해제하는 것처럼 추출 버튼을 누르면 된다. 나중에 백업을 할 때는 위와 마찬가지로 마운트만 해주면 되고.

팁
===================

처음 Time Machine 백업을 돌릴 때는 저어어어엉말 느리다. 컴퓨터의 말 그대로의 모든 것을 백업해야 하니 이해는 가지만, 그래도 느리다. 이유는 애플이 백업 중에도 다른 작업에 지장이 가지 않게 일종의 리밋을 걸어뒀기 때문.

느린 속도에 속이 터진다면 아래 명령어로 쓰로틀을 풀어주자. 만약 재부팅할 경우 자동 적용 해제되니 참고하시길.

```
sudo sysctl debug.lowpri_throttle_enabled=0
```

눈에 띄게 속도가 향상됬다면 씩 웃어주자.
