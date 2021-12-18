---
layout: post
title:  "Homebridge로 샤오미 공기청정기 HomeKit 제어하기"
date:   2020-10-10 11:27:45 +0900
categories: IoT
---

이전에 [Homebridge를 사용해 미지원 IoT 기기를 애플 HomeKit을 통해 제어하는 법](https://kycfeel.github.io/2018/01/30/Homebridge%EB%A1%9C-%EB%AF%B8%EC%A7%80%EC%9B%90-IoT-%EA%B8%B0%EA%B8%B0-HomeKit-%EC%A0%9C%EC%96%B4%ED%95%98%EA%B8%B0/)에 대해 다룬 적 있다. 당시에는 필자에게 있는 IoT 기기가 Yeelight 전구밖에 없었는데, 이후 샤오미에서 나온 공기청정기 (Mi Air 2s)도 추가로 들여 사용하고 있다. 

Yeelight와 똑같이 샤오미 브랜딩을 달고 출시한 제품답게, 와이파이를 통한 원격 제어와 프로그래밍이 가능한 모델이다. 혹시나 싶어 웹을 조금 찾아보니, Homebridge에서 Mi Air 모델도 제어할 수 있게 해주는 서드파티 플러그인이 존재했다. 이미 HomeKit의 마법같은 편리함을 맛 본 몸인데, 어떻게 더 망설일 수 있으랴. 바로 적용해보기로 했다.

# 어떤 플러그인을 사용해야 하나요?

서드파티로 기능을 구현한 만큼 플러그인이 딱 하나만 존재하는 것은 아니다. 몇 개의 Mi Air 지원 플러그인이 온라인 상에 존재하는 것은 알지만, 필자는 [seikan/homebridge-mi-air-purifier](https://github.com/seikan/homebridge-mi-air-purifier) 플러그인을 사용했다.

# 설치 방법

만약 Homebridge 서버를 미리 설정하지 않았다면, 필자의 [이전 글](https://kycfeel.github.io/2018/01/30/Homebridge%EB%A1%9C-%EB%AF%B8%EC%A7%80%EC%9B%90-IoT-%EA%B8%B0%EA%B8%B0-HomeKit-%EC%A0%9C%EC%96%B4%ED%95%98%EA%B8%B0/)을 참조해 설정부터 진행하길 바란다.

Homebridge 서버를 사용할 준비를 마쳤다면, 아래의 명령어를 통해 `homebridge-mi-air-purifier` 플러그인을 컨테이너에 설치하자. 

```
docker exec homebridge npm install -g homebridge-mi-air-purifier
```

설치가 완료되었다면, 이제 Homebridge의 `config.json` 파일에 아래와 같이 Mi Air를 위한 설정을 집어넣어야 한다.

아래의 탬플릿을 `config.json`의 `platforms` array에 넣으면 된다.

```
{
    "platform": "MiAirPurifierPlatform",
    "deviceCfgs": [{
        "type": "MiAirPurifier2S",
        "ip": "<MY_MI_AIR_IP>",
        "token": "<MY_TOKEN>",
        "airPurifierDisable": false,
        "airPurifierName": "MiAir2S",
        "silentModeSwitchDisable": false,
        "silentModeSwitchName": "MiAir2S Silent Mode Switch",
        "temperatureDisable": false,
        "temperatureName": "MiAir2S Temperature",
        "humidityDisable": false,
        "humidityName": "MiAir2S Humidity",
        "buzzerSwitchDisable": true,
        "buzzerSwitchName": "MiAir2S Buzzer Switch",
        "ledBulbDisable": true,
        "ledBulbName": "MiAir2S LED Switch",
        "airQualityDisable": false,
        "airQualityName": "MiAir2S AirQuality"
    }]
}
```

이 설정을 통해 내 Mi Air의 종류, 커스텀 이름, LED 작동 여부 등 세부적인 사항을 모두 조종할 수 있다.

그런데, `deviceCfgs.ip`와 `deviceCfgs.token` 2개 값이 비어있는 것이 보인다. 

그렇다. 예상하는 것과 같이 위 2 값은 내가 가지고 있는 Mi Air의 내부 IP와 고유 토큰을 집어 넣어야 한다.

Mi Air의 내부 IP는 내 공유기 설정 페이지 등에 접속해 쉽게 확인할 수 있을 것인데, 문제는 저 토큰이겠다. 도대체 어디서 기기 토큰을 얻어야 할까?

# Mi Air 제어 토큰 얻기

본래는 [`miio`](https://github.com/aholstenson/miio)와 같은 라이브러리를 통해 토큰값을 쉽게 얻을 수 있었지만, 최근에는 모종의 보안 패치가 진행됨에 따라 더 이상 그 방법을 사용할 수 없게 되었다. 

이를 대신하는 일종의 편법으로, 보안 문제가 있는 구 버전의 안드로이드 Mi Home 앱을 사용함으로서 토큰을 추출할 수 있다.

사실 이렇게까지 해야 하나 싶지만... 티끌만큼 더 윤택해질 우리의 삶을 위해 잠깐만 눈 딱 감자.

토큰을 추천할 수 있는 Mi Home의 버전은 `v5.4.49`다. 이 버전 이후의 Mi Home은 보안 문제가 패치되어 더 이상 토큰을 인할 수 없으니 기억하자.

[Apkmirror](https://www.apkmirror.com/apk/xiaomi-inc/mihome/mihome-5-4-49-release/) 같은 APK 덤프 사이트에서 설치 파일을 구해 안드로이드 기기에 설치한 후, 평소 Mi Home 앱을 사용하는 것과 같이 로그인을 진행하면 된다. 성공적으로 로그인이 된 후 내 기기 목록을 확인할 수 있으면 준비 완료.

이제 안드로이드의 파일 관리자 앱을 열어, 아래의 경로로 이동하자.

```
~/SmartHome/logs/plug_DeviceManager
```

상위 경로는 기기마다 차이가 있을 수 있으니, 검색 기능 등을 통해 `SmartHome` 이라는 폴더를 찾아보자.

경로로 이동했다면 아래 스크린샷과 같이 텍스트 파일이 보일 것이다. 열어보자.

> 필자는 해당 작업을 올해 초에 진행했기에, 파일 이름이 1월 30일 날짜로 태깅되어 있으니 참고하자.

![mi_home_path_screenshot.png](https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/mi_home_path_screenshot.png?raw=true)

파일을 열어보면 json 형태로 되어 있는 내 기기 목록 정보가 담겨있을 것이다. 검색 기능을 통해 `token` 키워드를 검색하자. 아래와 같이 토큰 정보를 확인할 수 있을 것이다.

![mi_home_token_screenshot.png](https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/mi_home_token_screenshot.png?raw=true)

위에서 Mi Air의 내부 IP를 아직 확인하지 않았다면, 같은 파일에서 IP까지 같이 확인할 수 있으니 긁어가자.

# 서버에 토큰 적용하기

이제 다시 서버의 `config.json` 파일을 열어, 아까 공란으로 남겨준 Mi Air IP와 토큰을 집어넣자. 완성된 형태는 아래와 비슷할 것이다.

> '******' 부분은 필자가 임의로 마스킹 처리한 민감 정보다. 

```

{
    "bridge": {
        "name": "Homebridge ******",
        "username": "******",
        "port": ******,
        "pin": "******"
    },
    "accessories": [],
    "platforms": [
        {
            "platform" : "yeelight",
            "name" : "yeelight"
        },
        {
            "platform": "MiAirPurifierPlatform",
            "deviceCfgs": [{
                "type": "MiAirPurifier2S",
                "ip": "192.168.******",
                "token": "******",
                "airPurifierDisable": false,
                "airPurifierName": "MiAir2S",
                "silentModeSwitchDisable": false,
                "silentModeSwitchName": "MiAir2S Silent Mode Switch",
                "temperatureDisable": false,
                "temperatureName": "MiAir2S Temperature",
                "humidityDisable": false,
                "humidityName": "MiAir2S Humidity",
                "buzzerSwitchDisable": true,
                "buzzerSwitchName": "MiAir2S Buzzer Switch",
                "ledBulbDisable": true,
                "ledBulbName": "MiAir2S LED Switch",
                "airQualityDisable": false,
                "airQualityName": "MiAir2S AirQuality"
                }]
        }
    ]
}

```

이제 살포시 저장해주고, 적용을 위해 Homebridge 서버를 재시작하자. 

```
docker restart homebridge
```

이제 조금만 기다리면 내 애플 Home 앱에 공기청정기가 빰 하고 머리를 내밀 것이다. 아래 스크린샷처럼 말이다!

![mi_air_on_apple_home_screenshot](https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/mi_air_on_apple_home_screenshot.png?raw=true)

단순한 공기청정기 전원 제어 뿐만 아니라, 온습도계와 연동되어 집 안의 온도와 습도까지 확인할 수 있다. 이제보니 따로 온도계 습도계 설치할 비용도 아꼈네.

만약 Homebridge 서버에 정상적으로 연결되지 않는다면 서버 로그를 꼭 살펴보자.

```
docker logs homebridge
```

무언가 디버깅할 수 있는 단서를 줄 것이다. 굿 럭.

# 마치며

10년, 20년 전만 해도 공상과학 영화 속의 일이였던 홈 오토메이션이 이제 몇만원도 하지 않는 저렴한 중국산 기기를 통해서까지 이루어질 수 있다는 건 분명한 격세지감이다. 

더군다나 애플 HomeKit에 연동해 통합까지 할 수 있다니! 분명 우리는 좋은 시대에 살고 있다. 코로나니 뭐니 하며 전 세계가 전래없는 재난 상황에 빠져있는 2020년이지만, 이런 작은 것들에 감탄하고 행복을 느끼면서 앞으로 나아갈 수 있는 해도 2020년이다. 

생각해보니 이 글이 올해 처음으로 블로그에 작성한 글이 되었다. 바쁘다는 핑계, 정신 없는 한 해여서 그랬다는 핑계를 대고 싶지만 그냥 심적 여유가 없었다. 일상 생활조차도 심적으로 고갈되어서 허덕이는 삶을 살았는데 한가롭게 글 쓸 여유가 어디서 나올까.

다행히 요즘은 개선이 있다. 항우울제가 도움이 되는 건지 단순히 시간이 약이라 그런 건진 모르겠지만, 분명 한 발자국씩 앞으로 나아가고 있다. 앞으로 글도 조금 더 쓸 수 있으리라.
