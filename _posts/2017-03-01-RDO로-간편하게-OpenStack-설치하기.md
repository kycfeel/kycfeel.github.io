이 문서를 시작으로 앞으로 작성될 모든 설명 문서는 그 전달성과 이해도를 높히기 위해 '~니다' 말투를 사용합니다.

시작하며
========================

현대 클라우드 환경의 사실상 표준이 되는 OpenStack 기술은 이미 전세계 수많은 개인과 단체들이 각종 서비스에 사용하며 충분히 검증되었습니다.

작성 기준 환경은 아래와 같습니다.

* Controller Node (Compute를 제외한 모든 서비스 구동) 1대 - HDD Software RAID 1 구성
* Compute Node 2대 - SSD Software RAID 5 구성
* CentOS 7.x 설치됨

> [인프라 구성도](https://gitlab.com/geekstack_io/openstack-infrastructure/blob/master/deploy/_images/infrastructure.png)

RDO-Project
========================

RDO Project는 Red Hat 기반 Linux (CentOS, Fedora, RHEL 등)에 OpenStack을 설치하는 것들 도와주는 자동화 소프트웨어, 그리고 그 커뮤니티를 지칭합니다.

물론 Fuel 등 PXE 레벨에서부터 자동화가 가능한 다른 도구들도 존재하지만, 사전 Software RAID 구축을 위해 RDO를 선택했습니다.

아래 링크에서 자세한 RDO에 관한 자세한 정보를 확인하실 수 있습니다.

> [RDO-Project 공식 웹사이트](https://www.rdoproject.org)


1. Software RAID 설정
========================

여러 개의 물리적 디스크를 하나의 논리적 디스크로 정의해 사용하는 RAID (Redundant Array of Independent Disks) 기술은 전체적인 데이터의 안정성과 그 성능을 향상시키기 위해 꼭 필요합니다.

데이터를 나누는 다양한 방법들이 존재하지만, 이 문서에서는 RAID 5 레벨을 사용할 것입니다.

별도의 물리적인 RAID 컨트롤러는 사용하지 않습니다. SSD 기반 RAID에서는 어중간한 중저가 RAID 컨트롤러가 오히려 성능을 하락시킬 수 있기 때문입니다.

RAID에 관한 자세한 정보는 아래 링크를 확인해 주세요.

> [위키백과 RAID 문서](https://ko.wikipedia.org/wiki/RAID)

OS 설치 과정 중 Software RAID 설정 과정은 아래 동영상을 참조해 주세요.

RAID 1, 5 모두 중간 과정은 동일합니다.

> [OS 설치 중 RAID 5 설정](https://www.youtube.com/watch?v=xmi8bQau9GY)


2. RDO로 OpenStack 설치하기
========================

RAID 구성된 깨끗한 CentOS 서버들이 준비되었다면, 아래 과정을 통해 손쉽게 OpenStack을 설치할 수 있습니다.

> [RDO QuickStart](https://www.rdoproject.org/install/quickstart/)

순서대로 진행하다 문서 맨 아래의 `sudo packstack --allinone` 차례에서 잠깐 멈춰 주세요.

위 명령어 대신 `sudo packstack --gen-answer-file=파일이름.txt` 명령어를 입력해 줍시다.

이 명령어는 PackStack 인스톨러 구동 관련 설정을 관리할 수 있는 스크립트를 생성해 줍니다.

방금 생성한 스크립트 파일에 들어가, 아래의 값들을 다음과 같이 수정합니다.

* CONFIG_KEYSTONE_API_VERSION=v3 // v2.0을 사용해도 되나, 추후 여러가지 범용성을 위해 v3 선택.
* CONFIG_PROVISION_DEMO=n
* CONFIG_NEUTRON_OVS_BRIDGE_MAPPINGS=extnet:br-ex
* CONFIG_NEUTRON_OVS_BRIDGE_IFACES=br-ex:네트워크 어댑터 이름
* CONFIG_NEUTRON_ML2_TYPE_DRIVERS=vxlan,flat
* CONFIG_CONTROLLER_HOST=IP 주소 // OpenStack 컨트롤러가 될 서버의 아이피
* CONFIG_COMPUTE_HOSTS=IP 주소 // Compute 노드가 될 서버의 아이피
* CONFIG_NETWORK_HOSTS=IP 주소 // Network 노드가 될 서버의 아이피

> [스크립트를 실행하기 전, 설치되는 장비간 ssh 키를 공유해두면 설치 과정 내내 비밀번호를 반복 입력하지 않아도 됩니다.](https://www.rdoproject.org/documentation/enabling-migrations/)

> 위의 키 공유 과정을 거치면, 추후 Compute 노드간 인스턴스 마이그레이션이 가능해집니다. 여러모로 필요하니 꼭 설정해 주세요.

준비가 완전히 끝났다면, `sudo packstack --answer-file=내 파일 이름.txt` 명령어로 스크립트를 실행합니다.

설치 과정이 끝나면 아래 링크를 참조해 다음으로 진행합시다.

(아래 링크의 첫번째 명령어는 넘어가 주세요. 이미 우리가 스크립트 편집 과정에서 적용한 것들입니다.)

> [RDO work with your existing network](https://www.rdoproject.org/networking/neutron-with-existing-external-network/)

위 과정까지 끝날 경우. 브라우저에서 컨트롤러의 IP를 입력하고 접속해 보세요.

정상적으로 OpenStack 대시보드 (Horizon)이 반겨주고 로그인까지 문제가 없다면, 정상적으로 설치가 완료된 것입니다.

잠깐 이것저것 만져보세요. 인스턴스를 생성해보고, 여러가지 마음이 가는대로 쑤셔보고 다녀도 좋습니다.

3.
========================
