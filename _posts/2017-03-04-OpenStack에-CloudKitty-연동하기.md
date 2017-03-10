---
layout: post
title:  "OpenStack에 CloudKitty 연동하기"
date:   2017-03-04 11:02:24 +0900
categories: OpenStack
---

CloudKitty?
========================

[CloudKitty](https://wiki.openstack.org/wiki/CloudKitty) 는 OpenStack을 위한 Rating-as-a-Service 프로젝트다. 다시 말해, 기존 OpenStack에 구현되어 있지 않은 Pricing이나, Billing 같은 기능을 추가해 사용할 수 있다는 거다. OpenStack의 [Ceilometer](https://wiki.openstack.org/wiki/Telemetry) 와 연동되어 사용량에 따라 과금 정책을 잡는 등 퍼블릭 서비스를 위한 발판으로 삼을 수 있다. 아무래도 OpenStack 자체가 프라이빗 클라우드를 생각하고 디자인된 플랫폼이라 CloudKitty같은 프로젝트는 마이너한 편이고, 한국어 자료는 더더욱 기대할 수 없어 이 포스트가 도움이 될 수 있을 것이다.

CloudKitty 설치하기
========================

최근 리뉴얼된 [공식 설치 가이드](http://cloudkitty.readthedocs.io/en/latest/installation.html) 를 참고해 Ubuntu 16.04 LTS 기준으로 작성되었다. 아래 과정으로 진행하기 전 적당한 사양의 (1vCore에 1GB만 되어도 테스트 목적으로 충분) VM을 준비해주자.

과거에는 `git`에서 직접 소스를 다운받아 인스톨하는 과정이 필요했었는데, 이제 `apt-get`으로 간편하게 설치가 가능해졌다. 아래 명령어로 repo를 추가해 주자. 권한 문제가 있으면 명령어 앞에 `sudo`를 적절히 붙혀준다.

```
apt install software-properties-common
add-apt-repository ppa:objectif-libre/cloudkitty
```

VM의 패키지를 업데이트 해준다.

```
apt update && apt dist-upgrade
```

이제 CloudKitty를 설치할 수 있다. 뭘 망설이고 있는가? 당장 받자.

```
apt-get install cloudkitty-api cloudkitty-processor cloudkitty-dashboard
```

정말 간단하게 명령어 몇 줄로 모든 설치과정이 끝났다. 예전 `git`에서 소스 가져와 설치하던 때만 하더라도 각종 의존성 문제에 파이썬 오류에 별 난리가 다 났었는데 이건 참 좋아진 것 같다.

CloudKitty 설정하기
========================

`/etc/cloudkitty/cloudkitty.conf` 에서 설정 스크립트를 건들 수 있다. 아래는 `Keystone v3` 기준 설정 예제다. 만약 `v2.0` 기준 예제가 필요하다면 위에 링크를 달아둔 CloudKitty 공식 메뉴얼에서 확인할 수 있다.

```
[DEFAULT]
verbose = True
log_dir = /var/log/cloudkitty

[oslo_messaging_rabbit]
rabbit_userid = openstack // rabbit 서비스 userid
rabbit_password = RABBIT_PASSWORD // rabbit 서비스 password
rabbit_host = RABBIT_HOST // rabbit 서비스가 돌아가는 호스트 IP (OpenStack-Controller)
rabbit_port = 5672

[ks_auth]
auth_type = v3password
auth_protocol = http
auth_url = http://localhost:5000/v3 // localhost 부분을 OpenStack-Controller IP 로 변경
identity_uri = http://localhost:35357/v3 // localhost 부분을 OpenStack-Controller IP 로 변경
username = cloudkitty
password = CK_PASSWORD // 사용할 password
project_name = service
user_domain_name = default
project_domain_name = default
debug = True

[keystone_authtoken]
auth_section = ks_auth

[database]
connection = mysql://cloudkitty:DB 비밀번호@localhost/cloudkitty // 아래의 DB 설정 후 정보 집어넣기

[keystone_fetcher]
auth_section = ks_auth
keystone_version = 3

[tenant_fetcher]
backend = keystone

[collect]
collector = ceilometer
period = 3600
services = compute, volume, network.bw.in, network.bw.out, network.floating, image // Ceilometer로 측정할 정보들

[ceilometer_collector]
auth_section = ks_auth
```

주요 파라미터들에 주석을 달아뒀으니 설정에 참고하면 된다. 아마 스크립트를 찬찬히 살펴보다 보면 `[database]` 라는 항목이 있었을 건데 지금부터 MySql에 CloudKitty용 DB를 하나 판 후, 위 스크립트에 집어넣어 보겠다.

```
mysql -uroot -p << EOF
CREATE DATABASE cloudkitty;
GRANT ALL PRIVILEGES ON cloudkitty.* TO 'cloudkitty'@'localhost' IDENTIFIED BY '사용할 비밀번호';
EOF
```

위 명령어로 Mysql에 DB를 만든 후, 다시 스크립트로 돌아가 `[database]` 칸을 방금 생성한 DB 정보로 수정한다. 저장한 후, 아래 명령어로 DB를 동기화하자.

```
cloudkitty-dbsync upgrade
cloudkitty-storage-init
```

Keystone 설정하기
========================

물론 CloudKitty가 OpenStack과 연결되어 Rating 서비스를 처리하려면, 인증 체계인 Keystone을 거쳐야 한다. 우리가 위에 작성했던 스크립트 내용 중 `[ks_auth]`라는 파라미터가 존재했는데, 인증할 `username`을 cloudktity로 설정해 뒀으나 정작 우리는 그런 계정을 Keystone에 등록한 적이 없지 않은가? 그래서 그걸 지금부터 할 것이다. 혹시 자동으로 CloudKitty가 등록해 주지 않을까 환상을 품었다면 안타깝지만 그런 건 없다.

`keystoneclient` 로 접속해 아래 명령어로 필요한 계정을 생성해보자. *OpenStack-Controller로 돌아와* RDO 설치 기준 `keystonerc_admin` 파일이 있는 곳에서 `. keystonerc_admin` 명령어로 접속할 수 있다.

```
openstack user create cloudkitty --password 위에서 지정한 비밀번호 --email cloudkitty@localhost
openstack role add --project service --user cloudkitty admin
```

이제 `rating` 이라는 새로운 역할을 만들어 cloudkitty를 묶어줄 거다.

```
openstack role create rating
openstack role add --project service --user cloudkitty rating
```

위 명령어들의 `--project` 는 본인의 상황에 맞게 골라주면 된다. 여기서는 `service`를 사용했다.

이제 마지막으로 Rating 서비스의 API Endpoint를 만들어 줘야 한다. 아래 명령어로 쉽게 생성할 수 있다.

```
openstack service create rating --name cloudkitty \
    --description "OpenStack Rating Service"
openstack endpoint create rating --region RegionOne \
    public http://OpenStack-Controller IP주소:8889
openstack endpoint create rating --region RegionOne \
    admin http://OpenStack-Controller IP주소:8889
openstack endpoint create rating --region RegionOne \
    internal http://OpenStack-Controller IP주소:8889
```

CloudKitty 서비스 시작하기
========================

잘 따라오셨다. 정말 마지막으로 CloudKitty 서비스를 어떻게 끄고 키는지만 알고 가자. CloudKitty를 설치한 VM에서 다른 앱들과 같이 `systemctl` 명령어로 제어할 수 있다.

```
systemctl start cloudkitty-api.service
systemctl start cloudkitty-processor.service
```

위의 모든 설정이 끝나고 CloudKitty 서비스도 정상적으로 굴러가고 있는가? 이제 OpenStack 대시보드로 접속해 보자. 로그인까지 마치면 이전에는 보지 못한 `Rating` 이라는 메뉴가 보일 것이다. 인스턴스 생성 창에는 가격 표시도 생긴다. 이제 이걸 자유롭게 주물러 보는 일만 남았다. 여러분의 몫이다.

마치며
========================

사실 이 정보를 찾는 사람이 얼마나 될 지는 모르겠다. CloudKitty라는 프로젝트가 워낙 마이너할 뿐더러 영어도 아니고 한국어로 작성된 이 포스트에 관심을 가지는 사람은 필자밖에 없을지도 모른다. 그래도 혹시 알까, 누군가 이 포스트를 읽고 도움을 받을지.
