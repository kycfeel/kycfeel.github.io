---
layout: post
title:  "RDO로 간편하게 OpenStack 설치하기"
date:   2017-03-01 03:22:25 +0900
categories: 시스템 운영과 배포
---

시작하며
========================

[RDO-Project](https://www.rdoproject.org)(또는 PackStack)는 CentOS (레드헷 기반 대부분의 배포판)에 OpenStack 플랫폼을 얹을 수 있는 가장 쉽고 빠른 방법이다. 현대의 시스템 엔지니어링은 그 규모와 복잡함이 날로 커져감에 따라 가능한 모든 부분을 자동화하는 것이 점점 더 중요해지고 있다. RDO는 OpenStack 기반 테스트를 진행하거나, 또는 상용 인프라를 구축할 때 좋은 선택이 될 수 있을 것이다.

작성 기준 환경은 아래와 같다.

```
* Controller Node (Compute를 제외한 모든 서비스 구동) 1대 - 192.168.0.1
* Compute Node 2대 - `192.168.1.2`, `192.168.1.3`
* CentOS 7.2
```

설치하기
========================

테스트 서버라면 상관 없지만, 혹시 상용 서비스를 위해 구축하는 시스템이라면 퍼포먼스와 안정성을 위해 사전에 꼭 디스크 RAID 설정을 권장한다. 꼭 비싼 하드웨어 RAID 컨트롤러를 구매할 필요는 없고. OS를 설치할 때 소프트웨어 레이드를 설정하면 충분하다. SSD 디스크를 사용할 경우 오히려 어중간한 하드웨어 RAID 컨트롤러보다 훨씬 안정적이고 속도도 잘 나온다고 하니 알아보길 권한다. 추후 필요가 있으면 이 부분 관련하여 따로 포스트를 작성하겠다.

일단 필요한 준비물들이 모두 Good-to-Go 상태라면, 아래 과정을 따라와보자.

```
sudo systemctl disable firewalld
sudo systemctl stop firewalld
sudo systemctl disable NetworkManager
sudo systemctl stop NetworkManager
sudo systemctl enable network
sudo systemctl start network
```

위 명령어는 `Controller`와 `Compute`노드 구분 없이 모든 서버에 사전 입력해준다. 읽어보면 단번에 알겠지만, 설치 중 문제가 생길 수 있는 요인들을 제거하는 거다. 클러스터링에 방해가 될 수 있는 `firewalld`를 정지하고, 서버용 CentOS 7까지 들어와 기껏 만져둔 네트워크 설정들을 다 뒤집어버리는 `NetworkManager`도 죽인다. 마지막으로 혹시 네트워크 기능이 재부팅 때마다 정지될 경우 아래 두 줄을 입력해 언제나 네트워크를 활성화 상태로 둔다.

이제 본격적으로 인스톨 과정에 들어가 보자. *Controller에서* 아래 명령어들로 인스톨 패키지를 단번에 다운로드할 수 있다. 다른 서버들은 그냥 놔두면 된다. 작성일 기준, 최신 OpenStack 릴리즈는 `ocata`지만, 안정성과 커뮤니지 지원 등을 생각해 2세대 전 버전인 `mitaka`를 사용하겠다. OpenStack은 6개월 단위로 신버전 릴리즈가 진행되고 있어, 정상적인 사용을 위해서는 1~2세대 전 버전을 선택해주는 편이 좋다. 1세대 전 버전인 `newton`도 얼마 전 테스트 결과 여러가지 버그들이 남아있어 거르겠다.

```
sudo yum install -y https://rdoproject.org/repos/rdo-release.rpm

sudo yum install -y centos-release-openstack-mitaka

sudo yum update -y

sudo yum install -y openstack-packstack
```

위 과정이 끝났다면 이제 인스톨러를 가동시킬 수 있다. 그런데 아무 설정 없이 바로 가동하면 당연히 우리가 원하는대로 구성이 불가능하다. 다행히 RDO는 자유롭게 설치 정보를 지정할 수 있는 스크립트를 읽어들일 수 있다. 바로 하나 생성해보기 전에, 혹시 `root` 계정 사용이 자유로운 환경이라면 앞으로의 과정은 `root`로 진행하는 것을 추천한다. 물론 보안 문제의 위험성은 잘 알고 있지만, 일반 계정에서 가끔 `sudo `를 통해 진행했을 시 권한 문제가 발생해 꼬여버리는 경우가 있었다. 정확한 문제 요인을 발견하면 포스트를 수정하도록 하겠다. 일단 아래 명령어를 통해 스크립트를 하나 생성해보자.

```
sudo packstack --gen-answer-file=스크립트이름.txt
```

방금 생성한 스크립트 파일에 들어가, 아래 값들을 편집해 준다.

  ```
  CONFIG_KEYSTONE_API_VERSION=v3 // v2.0을 사용해도 되나, 추후 여러가지 범용성을 위해 v3 선택.

  CONFIG_PROVISION_DEMO=n

  CONFIG_NEUTRON_OVS_BRIDGE_MAPPINGS=extnet:br-ex

  CONFIG_NEUTRON_OVS_BRIDGE_IFACES=br-ex:네트워크 인터페이스 이름

  CONFIG_NEUTRON_ML2_TYPE_DRIVERS=vxlan,flat

  CONFIG_CONTROLLER_HOST=192.168.1.1 // Controller의 관리 네트워크 IP

  CONFIG_COMPUTE_HOSTS=192.168.1.2, 192.168.1.3 // Compute 노드들의 관리 네트워크 IP

  CONFIG_NETWORK_HOSTS=192.168.1.1 // 상황에 따라 별도의 Network 노드를 둘 수도 있다. 여기서는 Controller 노드와 통합하겠다.
  ```

스크립트를 실행하기 전, [이곳](https://www.rdoproject.org/documentation/enabling-migrations/)을 참고해 각 서버간 ssh 키를 공유해주는 것을 강력 권장한다. RDO는 ssh를 통해 OpenStack의 구성원이 될 노드들에 설치를 진행하는데, 사전에 키를 공유해두지 않으면 설치 과정 내내 각 서버들의 비밀번호를 *계속* 물어볼 것이다. 스크립트 돌리고 커피라도 한잔 마시고 오고 싶다면 꼭 이 과정을 미리 거쳐둬라. 키를 미리 공유해두면 추후 Compute 노드간 인스턴스(VM)의 실시간 마이그레이션도 별도의 설정 없이 가능해지니 더더욱 해둬라.

준비가 완전히 끝났다면, `sudo packstack --answer-file=내 파일 이름.txt` 명령어로 스크립트를 실행한다. 이제 커피 한전 마시고 와도 좋다. 시간이 조금 걸릴 것이다. 아직 완전히 끝난 것은 아니니 이 웹페이지는 닫지 말고 잠깐 숨 돌리고 와라. 설치가 끝난 후 뵙겠다.

한숨 돌리고 왔고 스크립트가 다 돌아갔다면, 아래 과정으로 기본적인 네트워크 설정을 잡아주자. 원본 메뉴얼은 [이 곳](https://www.rdoproject.org/networking/neutron-with-existing-external-network/)을 확인하면 된다.

*Controller* 에서 `/etc/sysconfig/network-scripts/ifcfg-br-ex` 파일을 들여다보면, 아까 스크립트에 지정한 `br-ex` 브릿지의 정보가 보일 것이다. 혹시 뭔가 익숙하다고 느껴지면 당신의 감이 좋은 것이다. IP 주소를 비롯해 여러 설정 정보들이 `br-ex`와 연결한 네트워크 어댑터의 그것으로 바뀌지 않았는가? 이제 이전의 네트워크 어댑터는 `br-ex`의 짐꾼이 되었다. 앞으로의 설정은 여기서 만져주면 된다. 혹시 스크립트를 돌리기 전에 네트워크 어댑터에 설정을 입력하지 않아 `br-ex`도 공허하다면, 이제 만져주면 된다. 아래 예시 설정값을 참고해 만져주자.

```
DEVICE=br-ex // 브릿지 이름

DEVICETYPE=ovs // OpenStack에서 사용하는 브릿지는 OpenVSwitch(ovs)라는 기술을 사용한다.

TYPE=OVSBridge // 이곳도 위와 같은 의미.

BOOTPROTO=static // 서버의 IP가 막 바뀌면 사용할 수 없다. static(고정)으로 박아두자

IPADDR=192.168.0.1 // 브릿지의 (네트워크 어댑터의) IP

NETMASK=255.255.255.0 // IP의 서브넷 마스크

GATEWAY=192.168.122.1 // 브릿지의 (네트워크 어댑터의) 게이트웨이

DNS1=192.168.122.1  // 별도의 DNS를 굴리고 있지 않다면 8.8.8.8(Google DNS)나 사용하는 통신사의 것을 입력하자.

ONBOOT=yes // 부팅할 때 자동으로 브릿지도 작동
```

혹시 브릿지의 노예가 되어버린 기존의 네트워크 인터페이스는 어떻게 되었는지 궁금한가? `/etc/sysconfig/network-scripts/ifcfg-네트워크 인터페이스 이름` 에 들어가보자. 아마 아래같이 되어 있을 것이다. 불쌍한 녀석.

```
DEVICE=네트워크 인터페이스 이름

TYPE=OVSPort

DEVICETYPE=ovs

OVS_BRIDGE=br-ex

ONBOOT=yes
```

여기까지 확인했으면, `sudo systemctl restart network` 명령어로 네트워크를 재시작한다. 만약 위 과정에서 편집한 내용이 있다면 재시작되며 적용될 것이다.

이제 `openstackclient`에 접근해 기본적인 네트워크를 구성해줄 차례다. "내가 주인장이다" 라는 권한을 입증해주는 `OpenStack RC`파일은 RDO가 미리 생성해 두었다. 우리는 그걸 사용하기만 하면 된다. 기본 경로에 있다면, `. keystonerc_admin` 명령어로 client를 실행해 보자. 입력창에 무언가 변화가 있는가? 정상적인 것이다. 아래 과정을 계속 따라와보자.

```
neutron net-create external_network --provider:network_type flat --provider:physical_network extnet  --router:external
```

위 명령어는 OpenStack에서 네트워크를 관리하는 `neutron`에 `external_network`라는 외부로 연결할 flat(우리가 일상적으로 생각하는 가장 기본적인 네트워크 형태) 네트워크를 생성하라는 지시를 내린다.

```
neutron subnet-create --name public_subnet --enable_dhcp=False --allocation-pool=start=192.168.2.2,end=192.168.2.254 \
                        --gateway=192.168.2.1 external_network 192.168.2.1/24
```

명령어를 보고 대충 감이 왔겠지만, `public_subnet`이라는 `external_network`과 연결될 네트워크를 생성해 IP 범위를 지정해 주는 것이다. `external_network`와 연결될 것이니 당연히 네트워크 인터페이스가 접근할 수 있는 IP 범위를 지정해줘야 한다. 당연히 IP가 수시로 바뀔 수 있는 `dhcp`는 비활성화. 이 네트워크의 IP 범위는 `192.168.2,2`에서부터 `192.168.2.254`까지다. 서브넷 마스크는 `/24`.

```
neutron router-create router1

neutron router-gateway-set router1 external_network
```

가상의 `router1`이라는 라우터를 만들어서 아까 생성한 `external_network`와 연결시켰다. 이 작업을 하면서 `external_network`라는 네트워크와 연결되는 랜선을 `router1`이라는 유선공유기의 WAN 포트에 꼽았다라는 상상을 하면 더 직관적으로 이해될 것이다. WAN 포트에 선을 꼽았다면 이제 LAN 포트에도 꼽을 것이 필요하다. 일반적인 공유기에서 WAN에 인터넷 선을 꼽고, LAN에는 그 인터넷을 분배받을 장비 (컴퓨터)를 연결한다. 이것도 같은 개념이다. 다만 공유기에서는 LAN의 IP 범위를 기기 자체에서 지정해주지만, 이건 그런 기능이 없는 단순 라우터다. LAN의 IP 범위부터 연결까지 직접 해보도록 하자.

```
neutron net-create private_network

neutron subnet-create --name private_subnet private_network 192.168.3.0/24
```

`private_network`라는 LAN 네트워크를 만들어서 `192.168.3.1 ~ 192.168.3.254`까지의 IP 범위를 지정해줬다. 네트워크를 만들었으니 선을 꼽는 과정만 남았다.

```
neutron router-interface-add router1 private_subnet
```

자, 선도 제대로 꼽았다. 소프트웨어로 정의되는 네트워크의 가장 좋은 점은 선을 밟고 넘어지거나 헐렁하게 꼽혀 스트레스 상승에 기여하지 않는다는 점이다. 이걸로 기본적인 네트워크 설정까지 모두 끝났다.

브라우저에서 컨트롤러의 IP를 입력하고 접속해보자. 정상적으로 OpenStack 대시보드 (Horizon)이 반겨주고 로그인까지 문제가 없다면, 정상적으로 설치가 완료된 것이다. 기쁨을 만끽하며 이것저것 만져보자. 인스턴스를 생성해보고, 여러가지 마음이 가는대로 쑤셔보고 다녀도 좋다.

새로운 유저 (Project) 만들기
========================

이제 유저를 하나 만들어보자. 계속 `admin` 계정으로만 가지고 놀 수도 없는 노릇이니까.

```
openstack project create --enable internal

openstack user create --project internal --password foo --email bar@corp.com --enable internal

```

아무래도 Private 클라우드 목적으로 디자인된 OpenStack에서는 그 목적에 충실하게 `project`라는 단위를 사용한다. 겁먹지 말고 일단은 유저명이라고 생각하자. 우리는 `internal`이라는 `project`에 `foo`라는 비밀번호와 `bar@corp.com`이라는 더미 이메일 주소를 주고 생성과 동시에 활성화시켰다.

```
export OS_USERNAME=internal

export OS_TENANT_NAME=internal

export OS_PASSWORD=foo
```

위 명령어를 치면 터미널에서 우리가 방금 생성한 `internal` 유저로 로그인한 것과 같아진다. 이 상태로 원래는 위와 같이 `internal`만의 새로운 라우터나 네트워크를 생성하고 연결시킬 수 있어야 하는데, `keystone v3`부터 어딘가 변경점이 있는지 유저 입장에서 새로운 네트워크 생성이 안된다. 이 부분은 최대한 빨리 확인해서 포스트를 갱신해 두겠다. 터미널 권한을 다시 `admin`으로 바꿀려면 위 `export`명령어의 값들을 `admin`의 그것으로 바꿔 입력해주면 된다.
