---
layout: post
title:  "YateBTS를 사용하는 홈메이드 GSM 네트워크 구축하기"
date:   2017-12-02 18:35:45 +0900
categories: dokupe
---

[이전 프로젝트 포스트](https://kycfeel.github.io/2017/11/08/모두를-위한-모바일-네트워크-dokupe) 에서 온갖 폼은 다 잡아 놨으니, 이제 회수할 차례다. 지난 한 주간 틈날 때마다 방에서 전자파를 뿜어내며 시간을 보냈다. 네트워크 구축에 필요한 하드웨어도 배송 받았고, 어느 정도의 테스트도 끝마친 상태다. "모두를 위한 모바일 네트워크" 라는 프로젝트 취지에 맞게, 나 스스로가 국토를 뒤덮을 수 없으니 인터넷의 불특정 다수에게 과정을 공유하는 것이 맞다고 생각한다. 특히 이와 관련된 한국어 자료는 전무하다 싶을 정도니, 한국어 독자들에게는 분명 도움이 될 것이다.

> 주의! 이 프로젝트와 관련 포스트를 참고해 발생한 모든 종류의 불이익은 독자에게 있습니다. 적합한 법적 절차를 거치지 않았을 경우, 외부와 격리된 테스트 환경에서만 운용하고 전파가 외부로 새어나가지 않도록 주의하여야 합니다.

하드웨어 준비물
==========

- *라즈베리 파이 3 모델 B.*

  국내 인터넷 쇼핑몰 등지에서 쉽게 구할 수 있는 최신 라즈베리 파이 모델이다. 원활한 작업을 위해 16GB 이상의 마이크로SD와 적절한 전력을 공급해줄 수 있는 충전 케이블 / 전원 공급원을 같이 준비하자.

  > 휴대성과 적은 전력 소모를 걱정하지 않는다면 기존에 가지고 있던 x86 데스크탑이나 노트북 등을 사용해도 좋다. 오히려 퍼포먼스 부분에서는 더 권장한다. 라즈베리 파이는 기본 성능 자체도 낮을 뿐더러 USB 3.0을 지원하지 않아 추후 네트워크 구축 후 데이터 패킷 통신 시 (GPRS) 간혈적인 병목 현상이 발생할 수 있다.

- *BladeRF x40*

  필자는 몇해 전 킥스타터에서 성공적으로 모금을 마친 Nuand 사의 소프트웨어 정의 라디오 (SDR) 플랫폼 BladeRF x40를 구축에 사용했다. 비슷한 기능을 제공하는 장비 중 가장 저렴한 가격대와 안정적인 품질을 보여주니 개인적으로는 가장 추천한다. 한국에서 리셀링하는 업체는 프리미엄을 잔뜩 붙였으니 [공식 홈페이지](https://www.nuand.com) 에서 주문할 것. 한국 직배송도 지원한다.

  추가로, 수요층이 한정된 물건이라 서드파티 케이스를 구하기 힘드니 구매할 때 공식 홈페이지에서 케이스까지 같이 구매하는 편이 좋다. 물론 DIY 능력자라면 스스로 만들어 써도 된다.

- *SMA 규격을 준수하는 안테나*

  본인의 용도에 맞게 적절한 출력을 보장하는 SMA 규격 안테나를 준비해두자. BladeRF는 전이중 통신 플랫폼이니 당연하지만 안테나 2개가 필요하다. Nuand 공식 홈페이지에서 BladeRF와 함께 판매하는 2dBi 출력의 안테나가 있으니 귀찮으면 같이 사도 무방하다. 다만, 눈물 나오게 빈약한 전파 커버리지는 감안해야 한다.

기타 라즈베리 파이에 연결할 모니터나 키보드 등 작업에 도움을 줄 수 있는 물건들은 알아서 구비하길 바란다.

> 가입자 식별에 사용하는 SIM 카드는 이번 포스트에서는 사용하지 않는다. 가입자 인증 기능을 구현하고 싶은 독자는 Amazon이나 AliExpress 등에서 공 SIM 카드를 별도로 구매하면 된다. [이런 제품](https://www.aliexpress.com/item/16-in-1-Max-SIM-Cell-Phone-Magic-Super-Card-Integrate-Backup-all-your-Sims/32810074014.html?spm=2114.search0104.3.49.sURS5K&ws_ab_test=searchweb0_0,searchweb201602_3_10152_10151_10065_10344_10068_10345_10342_10343_10340_10341_10541_10562_10084_10083_10307_10539_10312_10059_10313_10314_10534_10533_100031_10604_10603_10103_10594_10557_10596_10595_10107_10142,searchweb201603_25,ppcSwitch_5&btsid=1a322392-cfdb-4b37-90c9-5e973b6667cb&algo_expid=9e44d0f2-9fef-4e18-b8ad-ebc3ff7af2b5-8&algo_pvid=9e44d0f2-9fef-4e18-b8ad-ebc3ff7af2b5&rmStoreLevelAB=5)과 [이런 리더기](https://www.aliexpress.com/item/Hot-Super-SIM-Card-Reader-Writer-Cloner-Edit-Copy-Backup-GSM-CDMA-USB/32711438816.html?spm=2114.search0104.3.18.4xvmkL&ws_ab_test=searchweb0_0,searchweb201602_3_10152_10151_10065_10344_10068_10345_10342_10343_10340_10341_10541_10562_10084_10083_10307_10539_10312_10059_10313_10314_10534_10533_100031_10604_10603_10103_10594_10557_10596_10595_10107_10142,searchweb201603_25,ppcSwitch_5&btsid=712eaa15-a899-46bd-97ef-7d6e60303880&algo_expid=16f554c6-8929-47af-892a-01e846e98e39-2&algo_pvid=16f554c6-8929-47af-892a-01e846e98e39&rmStoreLevelAB=5) 조합으로 맞추면 될 것이다.

소프트웨어 작업
==========

라즈베리 파이의 OS는 [라즈비안 Jessie Lite 마지막 릴리즈](http://downloads.raspberrypi.org/raspbian_lite/images/raspbian_lite-2017-07-05/) 를 사용한다. 사용할 YateBTS 소프트웨어가 Ubuntu 14.04 / Debian 8을 기반으로 하기 때문. 현재 업로드되는 최신 OS인 라즈비안 Stretch를 설치하면 소프트웨어 설치 과정 중간에 오류가 발생한다. YateBTS 프로젝트 자체도 정상적으로 유지되고 있고, 포럼도 활발한 상태인데 최신 OS 기반 대응은 해줄 생각을 안 하고 있어 아쉬울 따름이다. 그래도 아쉬운 사람이 발품 팔아야 하지 않겠나. 일단 그대로 따라가자.

위에서 언급한 YateBTS는 적합한 트랜시버와 컴퓨터만 있다면 누구나 모바일 네트워크를 구축할 수 있게 해주는 오픈소스 프로젝트다. 우리는 적합한 트랜시버 (BladeRF) 와 덜 적합하지만 적당히 굴러는 가는 컴퓨터 (라즈베리 파이)를 이미 가지고 있으니 YateBTS를 사용해 당장 셀룰러 신호를 뿜어낼 수 있다.

그리고 희소식 하나, BladeRF x40의 펌웨어와 YateBTS, 그리고 각종 필요한 유틸리티들을 자동으로 설치해주는 자동화 스크립트를 가져왔다. 혹시라도 이후 과정에 기다리고 있을 험난한 소프트웨어 설치 작업을 걱정하고 계셨던 분들이 있으시다면 안심하셔도 된다. 여러분이 할 일은 스크립트를 다운로드 받아 실행만 하면 된다.

아래 과정을 통해 라즈베리 파이에서 스크립트를 실행해 보자.

> 아래 과정에서 사용될 스크립트는 Matthew May와 Brendan Harlow님이 챔플레인 대학의 SEC-440(System Security) 학위 프로그램에 사용한 자동화 스크립트로 최신 YateBTS를 구동할 수 있도록 필자가 직접 커스터마이즈 하였습니다.

```
//필자의 GitHub에서 스크립트 다운로드.
wget https://raw.githubusercontent.com/kycfeel/dokupe/master/auto_yatebts_deploy.sh

//다운받은 스크립트에 실행 권한 추가.
chmod +x ./auto_yatebts_deploy.sh

//sudo 권한으로 스크립트 실행.
sudo ./auto_yatebts_deploy.sh
```

이제 커피 한잔 마시며 기다리면 된다! 첫 실행 시 내 네트워크의 이름을 물어보는데 알아서 잘 입력해주고, 중간 BladeRF 펌웨어 설치 시 BladeRF가 연결되어 있지 않다면 USB로 연결해달라는 안내문만 하나 뜰 뿐, 완전히 자동으로 YateBTS가 설치되니 걱정 푹 놓고 있으면 된다. 성능이 떨어지는 라즈베리 파이 특성 상 소프트웨어 make 과정 등에서 상당한 시간을 소요하니, 인내심을 가지자.

설치가 완료되었다면 라즈베리 파이 내 계정의 홈 폴더에 `StartYateBTS.sh` 라는 스크립트가 생성됬을 것이다. `sudo ./StartYateBTS` 명령어로 YateBTS를 실행해 보자. 자동으로 소프트웨어가 가동되고, BladeRF의 신호 송수신 LED가 깜빡거릴 것이다. 콘솔창의 화면에 `MBTS Ready` 라는 메시지가 나오면 가동 완료.

고맙게도 YateBTS는 대부분의 네트워크 설정 과정을 GUI로 진행할 수 있도록 웹 인터페이스를 제공한다. 왜 생 Yate 엔진만 사용하지 않고 YateBTS를 같이 사용하는지 지금부터 피부로 느끼게 될 것이다. 생각보다 상당히 편하다. 같은 네트워크에 연결된 컴퓨터에서 브라우저를 키고 `라즈베리_파이의_IP/nib` 주소로 접속하면 된다. 아마 별 문제 없이 웹 인터페이스가 마중을 나올 거다.

<div align="center"><img src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/yatebts-main.png?raw=true"/></div><br/>

바로 웹 인터페이스의 `BTS Configuration -> GSM` 메뉴로 들어가서 가장 중요한 GSM 네트워크의 설정을 진행해보자.

<div align="center"><img src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/yatebts-config.png?raw=true"/></div><br/>

위 이미지는 필자의 세팅값인데, EGSM900 - #1000 (930.2 MHz) 규격으로 네트워크를 구축하고 있고, MCC와 MNC 코드는 각각 테스트 네트워크를 의미하는 001과 01로 설정되어 있다. 하단의 전파 최소 / 최대 강도는 모두 중간 수준인 35, 네트워크를 식별명은 프로젝트 이름을 그대로 따 dokupe로 설정했다.

원할 경우, 바로 옆의 GPRS 메뉴로 넘어가 `Enable` 값에 체크만 해 주면 GSM 음성 통화 / SMS와 더불어 GPRS 데이터 통신도 가능해진다. 외부 인터넷 연결까지 생각한다면 GGSN 메뉴의 `Firewall.Enable` 값을 `no firewall`로 변경해주자.

일반적인 GSM 네트워크는 우리에게도 익숙한 USIM (SIM) 카드를 통해 가입자를 식별한다. 혹시 설치 스크립트를 유심히 살펴봤다면 항목 중 `pySIM` 이라는 소프트웨어가 있었을 건데, 이게 바로 라즈베리 파이를 통해 SIM 카드를 프로그래밍 할 수 있게 해주는 도구다. 웹 인터페이스의 `Subscribers -> Manage SIMs` 메뉴를 통해 간단히 사용할 수 있지만, 이번에는 SIM 카드 없이 주변의 모든 GSM 대응 휴대전화가 네트워크에 연결할 수 있는 방법을 소개하려 한다. SIM 카드에 투자하는 추가 비용이 없는 것은 좋지만 출력이 높아질 경우 제 3자의 휴대전화가 내 네트워크에 자동 연결되어 버릴 수도 있으니 언제나 주의해야 한다. 테스트 환경에서만 사용할 것. ~~사실 지금 사용 가능한 공 SIM 카드와 리더기가 없다 읍읍~~

그렇다고 복잡한 건 없고, 단순히 `Subscribers -> List Subscribers` 메뉴의 `Regexp` 값을 `.*` 으로만 고정하면 SIM 카드가 없어도 누구나 네트워크에 연결이 가능해진다. 이 정도면 만지면 이제 최소한의 네트워크 설정이 완료되었다.

사용하기
==========

이제 휴대전화를 꺼내 `수동 네트워크 검색` (또는 비슷한 기능을 하는 메뉴) 에 들어가 스캔을 돌려보면 `KT`나 `KOR SK Telecom` 등의 익숙한 통신사와 함께 `Test PLMN 1-1` (혹은 비슷한 이름의) 처음 보는 통신사가 같이 검색될 것인데, 이게 우리가 만든 모바일 네트워크다.

<div align="center"><img src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/yatebts-search.jpg?raw=true"/></div><br/>

> 처음 스크립트를 돌릴 때 설정했던 네트워크 이름은 왜 안나오나 궁금해 할 수도 있는데, YateBTS의 커뮤니티 버전은 커스텀 네트워크 이름의 출력을 지원하지 않는 것 같다. 해외 포럼 등에서도 계속 오고가는 이야기이며, 틈날 때마다 리서치 중이니 혹시 우회 방법이나 대체 해결책이 보이면 글을 업데이트 하겠다. `Test PLMN 1-1`는 우리가 설정한 MNC, MCC 코드에 지정되어 있는 기본 네트워크 이름이다.

정상적으로 휴대전화가 네트워크에 연결되었다면, 아래와 같이 몇초 지나지 않아 YateBTS 시스템에서 자동으로 보내는 문자메시지가 수신될 것이다. 웰컴 메시지는 Yate 시스템의 버전에 따라 얼마든지 형태가 달라질 수 있다. 혹시 내가 수신한 메시지의 양식이 다르더라도, 그 내용은 아주 비슷할 것이니 별 걱정하지 않아도 된다.

<div align="center"><img src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/yatebts-welcome.jpg?raw=true"/></div><br/>

> 상단의 웰컴 메시지는 Yate 6 기준 `/usr/local/share/yate/scripts/nipc.js` 파일 안의 `sms_registration` 변수 안에 담겨있다. 자유롭게 커스터마이즈 해보자!

수신된 문자 메시지에는 내 모바일 네트워크에서의 고유 전화번호도 같이 적혀 있다. 이제 이 번호로 다른 기기들과의 전화 / 문자메시지를 주고받을 수 있다. 남는 휴대전화가 더 있다면, 네트워크에 연결해 서로 전화를 걸어보고 문자를 발송해 보자. 별 문제 없이 잘 진행될 것이다.

IP를 할당받고 인터넷에 연결하는 일도 전혀 어려울 건 없다. 평소 휴대전화에서 데이터 네트워크에 접속하는 것처럼, 설정의 `셀룰러 데이터` 메뉴에서 데이터 통신을 허용해주기만 하면 된다. 평소에 보던 LTE나 3G 마크 대신 GPRS라는 안 익숙한 친구가 불쑥 튀어나올 것이다. 물론 기본 수십 ~ 수백 mbps 속도를 뽑아내는 LTE 네트워크를 사용하던 우리한테는 이 GPRS라는 친구는 영 미덥지 않다. 무려 이론상 최고 속도 114kbps를 보장해주는 2000년대 2.5G 기술이니까 말이다. 그래도 뭐, 일단 인터넷에 연결할 수 있다는 것 만으로도 감사하자. '불가능한 것' 과 '불편한 것' 의 차이는 어마어마하게 크다.

결론
==========

독자분들 중 시대가 어느 시대인데, 왜 GSM 네트워크를 선택했냐고 궁금해하시는 분들도 분명 있을 것이다. 필자도 이왕 만들거 속도 빵빵한 LTE 네트워크를 사용할 수 있었으면 좋았겠지만 현실적으로 봤을 때, 그리고 우리 네트워크의 취지를 생각할 때 GSM과 GPRS 조합이 가장 적당하다고 판단했다.

일단, GSM 이상의 모바일 네트워크 (UMTS 등)는 더 높은 하드웨어 사양을 요구한다. 다시말해, 라즈베리 파이로 구축하는 휴대용 네트워크는 어림도 없다는 소리다. 노트북이라는 대안이 있기는 하지만, 비용 측면이나 전력 소모량, 공간 사용 측면에서 비교가 되지 못한다. 하나 더 이유를 꼽자면, 오직 GSM 네트워크만 SIM을 통한 가입자 인증 없이 주변 모든 휴대기기를 네트워크에 연결시킬 수 있다. UMTS 이상 모바일 네트워크들은 필수적으로 사전에 사용자 정보를 SIM에 프로그래밍 해둬야 한다. 유사시 네트워크가 제 기능을 하기 위해서는 최대한 간단하고 편하게 사람들 사이에 퍼져야 한다. 평시에도 국내에서는 공 SIM 카드 자체를 구할 수 없을 뿐더러, 일일이 그 많고 많은 사람들을 위한 SIM 카드를 프로그래밍 하고, 또 나눠준다는 것 자체가 비현실적이다. 사용할 유저가 없는 네트워크는 유령 전파가 될 뿐이다.

물론, 여유가 있다면 [OpenBTS-UMTS](http://openbts.org/w/index.php?title=OpenBTS-UMTS) 프로젝트나 [openLTE](http://openlte.sourceforge.net) 프로젝트 등을 참고해 GSM보다 빠르고 현대적인 네트워크를 얼마든지 구축할 수 있다. 관심이 있다면 해당 사이트를 잘 열람해보자. 필자도 시간과 자원이 남는다면, 이런 프로젝트들을 활용한 네트워크를 구축해보고, 글을 남길 의향이 있다.

여기까지 따라오신 분들, 혹은 단순 흥미로 읽어내려가신 분들, 모두 수고하셨다. 이제 우리 손에 모바일 네트워크 하나쯤은 들고 집에 갈 수 있게 되었다. 각종 재난 상황에서, 또는 혹시라도 세상에 '네트워크는 공공재' 라는 중요한 사실을 다시 각인시켜줄 필요가 있을 때, 오늘의 삽질이 결실을 맺을 수 있길 바란다.
