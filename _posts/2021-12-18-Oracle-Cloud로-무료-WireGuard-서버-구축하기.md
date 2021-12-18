---
layout: post
title:  "Oracle Cloud로 무료 WireGuard 서버 구축하기"
date:   2021-12-18 20:40:45 +0900
categories: General DevOps
---

평생 무료 클라우드 컴퓨팅 서비스라니. 도대체 그런 게 어떻게 가능할까 싶은데, 오라클이 그걸 해냈다. 기존의 AWS나 GCP 같은 전통적 (?) 퍼블릭 클라우드 서비스도 프리 티어나 무료 크레딧 등을 제공했었지만, 대부분 기간 제한이 있고 크레딧을 전부 사용하면 자동으로 과금이 시작되는 구조였다. 사실 그것만이라도 감지덕지하며 사용했던 것이 엇그제 같은데, 이제 그 "프리 티어"가 평생 유지되며 제한없이 사용할 수 있는 서비스까지 등장했다. AWS의 이야기는 아니고, GCP나 DigitalOcean의 이야기도 아닌, 바로 Oracle Cloud의 이야기다.

AWS 같이 프리 티어라고 완전히 저사양 서비스만 제공하는 것이 아닌, ARM CPU를 사용해야 하는 제약이 있지만 Oracle Cloud는 무려 CPU 4 Core와 24GB RAM을 제공하는 인스터스까지 사용할 수 있다. 사실 ARM CPU를 사용하는 것이 딱히 제약이라고 말할 수도 없다. 요즘의 ARM 프로세서는 퍼포먼스와 에너지 사용량 측면에서 모두 기존 x64 CPU를 압도하고 있다. 애플리케이션을 ARM64 아키텍쳐로 새로 빌드해야 하는 것만 제외하면, 오히려 업그레이드라고 생각할 수도 있는 것이다.

처음 이 평생 무료 클라우드 서비스에 대해 들었을 때, 바로 든 생각은 바로 상시 사용할 수 있는 VPN 서버를 구축하는 것이였다. 물론 기존에 다룬 [라즈베리 파이로 OpenVPN 서버를 구축하는 글](https://kycfeel.github.io/2017/07/10/%EB%9D%BC%EC%A6%88%EB%B2%A0%EB%A6%AC-%ED%8C%8C%EC%9D%B4%EB%A1%9C-OpenVPN-%ED%99%98%EA%B2%BD-%EA%B5%AC%EC%B6%95%ED%95%98%EA%B8%B0/) 과 같이 이미 개인용 VPN 서버를 구축해 집에서 사용하고 있긴 하지만, 어디까지나 개인 인터넷에 개인 IP를 사용해야 한다는 점은 변하지 않는다. 만약 VPN을 사용해 안전하지 않은 네트워크에서의 통신의 암호화를 넘어, 더 나은 익명화와 개인정보 보호를 생각하고 있다면 나와 직접 연관되지 않은 퍼블릭 IP를 사용할 필요가 있다. 더군다나 한국의 인터넷 검열을 미꾸라지처럼 빠져나가려면, 해외 IP를 사용할 수 있어야 한다.

이런 점을 생각했을 때, 평생 무료 인스턴스를 제공하고 해외 여러곳에 데이터 센터가 존재하는 Oracle Cloud는 굉장히 매력적인 선택지가 될 수 밖에 없다. 한국에서 가까운 일본에도 2곳의 데이터 센터를 가지고 있으니, 더할 나위 없다.

그럼, 빠르게 본론으로 넘어가 Oracle Cloud의 일본 데이터센터에 무료 인스턴스를 생성하고, OpenVPN 서버를 구축해 보자.

# Oracle Cloud에 인스턴스 생성하기

Oracle Cloud에 빠르게 회원가입을 한 후, Compute - Instances 탭에서 새 가상 인스턴스를 생성할 수 있다. 

![os_to_ubuntu](https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/os_to_ubutnu?raw=true)

먼저 OS부터 지정해 보자. "Always Free-eligible"에 해당되는 이미지 중 어떤 것을 골라도 비용은 청구되지 않지만, 우리는 빠른 VPN 서버의 구축을 위해 Ubuntu를 선택하자.

![instance_spec_configuration](https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/instance_spec_configuration?raw=true)

이제 가장 중요한 사양 설정이 남아있다. 기본 설정인 AMD 인스턴스를 사용하면 겨우 1vCore에 1GB 사양만을 제공받지만, ARM 기반 인스턴스를 선택하면 최대 4vCore와 24GB RAM까지 할당받을 수 있다. 

![instance_free_spec_limit_screenshot](https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/instance_free_spec_limit_screenshot?raw=true)

평생 무료 조건으로 제공받기에는 실로 엄청난 사양이 아닐 수 없다. 우리는 그냥 입 꾹 다물고, 감사합니다 하면서 받으면 된다.

한번에 4vCore와 24GB를 할당하지 않고, 작은 사양으로 쪼개서 여러 인스턴스를 생성할 수도 있는 것으로 안다. 2vCore와 12GB RAM을 가진 인스턴스 2개를 생성하는 식으로 말이다. 고로 필요에 맞게 사양을 지정해 주자. 필자는 오직 VPN 용도로만 Oracle Cloud를 사용할 예정이기에, 한번에 4vCore와 24GB RAM을 몰빵했다.

![instance_configuration_done](https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/instance_configuration_done?raw=true)

설정을 마무리하고 인스턴스를 생성하면 위와 같이 접속에 필요한 공인 IP와 계정 이름 등이 보일 것이다. 인스턴스 생성 중 추가한 PEM 키와 함께 SSH 접속을 해보자.

```
ssh -i <PEM_KEY_경로> ubuntu@<내_인스턴스_공인_IP>
```

정상적으로 접속된다면 아래와 같이 쉘 메인 화면이 보일 것이다.

![shell_connected](https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/shell_connected?raw=true)

# OpenVPN 서버 설정하기

이제 인스턴스는 잘 생성되었고, WireGuard 서버만 구축하면 된다. WireGuard 서버를 구축하는 법은 여러가지가 있겠지만, 필자는 평소 애용하는 툴인 [PiVPN](https://www.pivpn.io)을 사용할 예정이다.

WireGuard는 기존의 VPN의 표준이였던 OpenVPN을 넘어서는, VPN 프로토콜의 신흥 강자로 떠오르는 기술이다. 더 작은 오버헤드로 인한 더 나은 속도, 덜 복잡한 코드베이스로 인한 쉬운 코드 취약점 파악과 디버깅, 그리고 모바일 환경에 유리한 더 나은 네트워크 전환 처리까지, 거의 모든 부분에서 OpenVPN의 상위 호환으로 기능하는 그 특정 덕분이다. 덕분에, 기존에 OpenVPN을 사용했거나 사용 예정이라면, 별 고민 없이 WireGuard로 기술만 다르게 선택하면 나머지는 OpenVPN을 사용할 때와 큰 차이 없이 더 나은 사용자 경험을 누릴 수 있어 유저 입장에서는 더할나위 없는 새로운 기술이다. 필자도 이번에는 WireGuard를 선택해서 글을 진행하고자 한다. 

이름만 들어서는 라즈베리 파이를 위한 VPN 구축 도구 같지만, 사실 Debian 기반 리눅스 배포판에서는 어떻게 실행해도 전부 잘 동작하는 유연한 도구다. 라즈베리 파이용으로 디자인된 만큼 ARMv7 / ARM64V8 아키텍쳐도 문제 없고, 일반 AMD64 인스턴스나 심지어 IBM Z 메인프레임 규격인 S390X에서도 너무나 쌩쌩하게 잘 동작한다. 개인적으로 Debian 환경에서 가장 쉽고 빠르게 WireGuard 구축할 수 있는 도구라고 생각한다. 개발자 분에게 압도적인 감사를 표한다.

설치와 구축은 아래 명령어 한 줄로 시작할 수 있다.

```
curl -L https://install.pivpn.io | bash
```

> 혹시 WireGuard가 아니라 OpenVPN의 사용을 원할 때는, 필자가 이전에 작성한 [이 글](https://kycfeel.github.io/2017/07/10/%EB%9D%BC%EC%A6%88%EB%B2%A0%EB%A6%AC-%ED%8C%8C%EC%9D%B4%EB%A1%9C-OpenVPN-%ED%99%98%EA%B2%BD-%EA%B5%AC%EC%B6%95%ED%95%98%EA%B8%B0/)을 참고해 보시기 바란다.

설치 과정 중 복잡한 내용은 없다. OpenVPN과 WireGuard중 사용하고 싶은 걸 선택하고 (이 글에서는 WireGuard를 사용), 내가 VPN에 사용할 DNS 정보나, OpenVPN이 사용할 포트 등을 지정해 주면 끝난다.

![WireGuard-port-select](https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/WireGuard-port-select?raw=true)

Port는 기본으로 UDP 51820이 선택된다. 필요할 경우 변경해주자.

![pivpn-unattended-upgrade](https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/pivpn-unattended-upgrade?raw=true)

설치 중간에 Debian의 Unattended Upgrade를 사용할 것이냐 물어본다. 만약 약간의 다운타임은 크게 문제 없이 넘길 수 있고, 항상 최선 버전을 유지하는 걸 중요하게 생각한다면 Yes를 눌러 켜 주는 걸 추천하고, 그게 아닌 대부분의 유저에게는 No를 눌러 허용하지 않는 걸 추천한다. 이론만 생각한다면 언제나 모든 패키지를 항상 최신으로 유지하는 게 가장 좋지만, 그만큼 예상하지 못했던 호환성 문제 등을 경험할 수 있다. 특수한 경우가 아닌 이상 업데이트는 직접 관리해주는 걸 추천한다.

이후, WireGuard에서 사용할 유저 정보를 생성해 주자.

```
pivpn -a
```

![pivpn-wireguard-user-add](https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/pivpn-wireguard-user-add?raw=true)

간단히 사용할 유저네임만 입력하면 생성이 완료된다.


![pivpn-wireguard-qr-code](https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/pivpn-wireguard-qr-code?raw=true)


재미있는 점은 계정이 생성된 이후에, QR코드를 통해 쉽게 모바일 기기에 추가할 수 있다는 점이다. 아래 명령어를 통해 QR코드를 생성해 보자.

```
pivpn -qr
```

![pivpn-wireguard-mobile-connected](https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/pivpn-wireguard-mobile-connected.png?raw=true)

QR코드를 스캔해 VPN 설정을 추가하고 나면 위와 같이 쉽게 연결할 수 있다.

![pivpn-wireguard-ip-changed](https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/pivpn-wireguard-ip-change.pngㅔㅜㅎ?raw=true)

마지막으로 내 IP 주소가 정상적으로 변경되었는지 확인 한번만 해주자.

# 마치며

이제 우리는 일상에서의 트래픽 암호화 + 한국 인터넷 검열 우회를 위해 유료 VPN 서비스를 찾아나서지 않아도 되게 되었다. 각종 제한이 걸린 상태로 설치되어 있던 무료 상업 VPN 서비스도 지워도 된다. 이제 안전하게, 또 확실하게 나만의 VPN 서버를 사용해서 걱정없이 인터넷을 자유롭게 날아다닐 차례다. 

이 글이 일상에서의 약간의 더 프라이버시를 원하는 많은 사람들에게 도움이 되었으면 좋겠다.

