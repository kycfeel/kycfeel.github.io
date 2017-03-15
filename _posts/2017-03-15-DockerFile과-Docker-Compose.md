---
layout: post
title:  "DockerFile과 Docker-Compose"
date:   2017-03-15 15:04:20 +0900
categories: Docker
---

[이전 포스트](https://kycfeel.github.io/2017/03/14/어서오세요-Docker의-세계에/)에서 기본적인 Docker의 개념과 설치법, 이미지 등을 다뤘다. 아직 안 읽어보셨다면 참고하시길.

DockerFile, 그리고 Docker-Compose
========================

<div align="center"><img src="https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/compose.png?raw=true"/></div>

마치 Docker 이미지와 컨테이너의 관계가 그랬듯, DockerFile과 Docker-Compose는 땔래야 땔 수 없는 관계이다. 우선 DockerFile 이미지를 실행하면서 특정 작업까지 같이 처리하게 해주는 도구고, Docker-Compose는 다수의 컨테이너를 쉽게 실행할 수 있게 도와주는 도구이다. 다시 말해, Docker-Compose로 컨테이너를 자동 생성한 후, DockerFile로 생성한 컨테이너 안에 자동으로 세팅 작업까지 돌아가게 할 수 있다는 것이다.

지금부터 Docker 공식 [메뉴얼](https://docs.docker.com/compose/gettingstarted/)을 참고해 DockerFile과 Docker-Compose를 모두 사용하는 예제를 작성할 것인데, 아직 Docker-Compose를 설치하지 않은 여러분들은 아래 과정을 먼저 밟고 따라와주시면 고맙겠다.

Docker-Compose 설치하기
========================

이전 포스트와 동일하게 필자는 CentOS에서 설치를 진행하였지만, 이번에는 GitHub에서 직접 파일을 끌어올 것이라 다른 리눅스 배포판들도 설치 방법이 동일하다. 그냥 따라오셔도 무방할 것 같다. 원본 메뉴얼은 [이곳](https://docs.docker.com/compose/install/)에서 확인할 수 있다.

```
curl -L "https://github.com/docker/compose/releases/download/1.11.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

`curl`로 Docker-Compose의 GitHub 저장소에서 필요한 파일을 끌어온다. 혹시라도 `/usr/local/bin` 경로에 권한 문제가 있어 접근이 안 되시는 분들은 위 명령어를 입력하기 전 `sudo -i` 명령어로 `root` 권한을 받은 후 진행하시길 바란다.

```
chmod +x /usr/local/bin/docker-compose
```

파일 끌어오기가 완료되었다면, `chmod` 명령어로 해당 경로에 실행 권한을 준다.

여기까지 끝났다면, `docker-compose --version` 명령어를 입력해보자. 정상적으로 Docker-Compose의 버전이 출력된다면 설치 성공.

본론
========================

이제 *정말로* DockerFile과 Docker-Compose를 모두 활용해 무언가 생성해볼 시간이다. Redis가 실행되는 컨테이너 위에 Flask가 돌아가는 환경을 간단히 만들어 보도록 하겠다.

```
mkdir composetest
cd composetest
```

먼저 `composetest`라는 실험을 진행할 폴더를 만들고, 그 안으로 들어간다. 이제 원하는 텍스트 에디터로 `app.py`라는 새로운 파일을 만들어 아래의 내용으로 채워넣는다.

```
from flask import Flask
from redis import Redis

app = Flask(__name__)
redis = Redis(host='redis', port=6379)

@app.route('/')
def hello():
    count = redis.incr('hits')
    return 'Hello World! I have been seen {} times.\n'.format(count)

if __name__ == "__main__":
    app.run(host="0.0.0.0", debug=True)
```

코드를 대충 읽어보면 알겠지만, Flask가 당신에게 인사를 해주며 몇 번 마주쳤다고 친절하게 알려까지 주는 귀여운 코드다. 그대로 저장하면 된다.

다시 `requirements.txt`라는 파일을 하나 더 만든 후, 아래 내용을 넣어준다. 무엇이 이 애플리케이션에 필요한지 명시해주는 파일이다.

```
flask
redis
```

마찬가지로 저장하고 나오면 된다.

이제 DockerFile을 작성해보자. `Dockerfile`이라는 파일을 만들어, 아래처럼 내용을 집어넣으면 OK.

```
FROM python:3.4-alpine # Python 3.4를 사용하는 이미지를 기준으로 시작.
ADD . /code # 현재 위치(.)을 이미지의 /code 위치에 넣기.
WORKDIR /code # 작업 위치를 /code로 설정.
RUN pip install -r requirements.txt # 아까 requirements.txt에서 명시한 필요한 소프트웨어 설치.
CMD ["python", "app.py"] # 컨테이너의 기본실행 명령어를 python app.py로 설정.
```

위 스크립트 내용에 주석을 달아뒀으니 참고 바란다.

자. 컨테이너가 실행되면 어떤 내부작업이 돌아갈 것인지 DockerFile에 명시해 뒀으니, 이제 컨테이너 자체를 실행하는 Docker-Compose를 설정해 보자. `docker-compose.yml`이라는 새로운 파일을 생성하고 아래 내용을 넣어보자.

```
version: '2' # Docker-Compose 버전 2 사용.
services: # 컨테이너별 서비스 정의.
  web: # 웹서버 부분.
    build: . # 현 위치의 DockerFile을 사용해 이미지 빌드.
    ports: # 웹서버 포트.
     - "5000:5000"
    volumes: # 웹서버 저장소.
     - .:/code
  redis: # Redis 부분.
    image: "redis:alpine"
```

Docker는 서비스별로 컨테이너를 분리시키는 것을 좋아한다. 그게 컨테이너 기술의 의의기도 하니까 말이다. 웹서버와 Redis 컨테이너를 따로따로 만들어 보겠다. DockerFile과 마찬가지로 위 스크립트에 주석을 달아뒀으니 참고하길 바란다.

준비는 모두 끝났다. 바로 가동시켜 보자. `docker-compose up` 명령어를 치면 끝. 컨테이너가 모든 준비를 스스로 마친 것 같으면 웹 브라우저를 키고 `http://localhost:5000` 주소로 접속해보자. 아래같은 실행값이 보이면 성공.

![dockercomposeruned](https://github.com/kycfeel/kycfeel.github.io/blob/master/_images/dockercomposeruned.png?raw=true)

이곳에서는 간단한 웹 애플리케이션을 실행하는 것만 보여줬지만, 이것을 응용해 활용할 수 있는 분야는 무궁무진하다. 내 개인, 또는 팀의 서비스가 Docker에 담기는 상상을 해봐라. 그리고 그로 얻을 수 있는 모든 이점을 상상해보자. 모 블로그의 표현처럼 Docker는 단순 생산성 뿐만 아니라 인간의 상상력 자체를 자극하는 도구가 분명하다.

다음 포스트에서는 Docker에서 팍팍 밀어주는 클러스터링 툴인 Docker-Swarm에 대해 다뤄보려 한다. 모두 수고 많으셨다.
