---
layout: post
title:  "Google Analytics로 GitHub 페이지 분석하기"
date:   2017-03-03 02:38:30 +0900
categories: Jekyll과 GitHub 페이지
---

Google Analytics?
========================

Google에서는 [Analytics](https://www.google.com/analytics/) 라는 강력한 무료 방문자 분석 서비스를 제공한다. 멋진 웹 그래프도 확인할 수 있고, 적용법도 JS 코드만 집어넣으면 되니 아주 간단하다. 복잡할 것 없는 포스트이니 빠르게 따라오면 될 것 같다.

사전 준비
========================

위 Analytics 서비스 웹사이트로 들어가 초기 세팅을 해둔다. 누구나 하나쯤 있는 Google 계정으로 로그인한 후, 추적할 대상 (Analytics는 웹사이트 뿐만 아니라 Admob 광고나 Android 앱 등 여러가지 대상 추적을 지원한다.)을 선택하는 등 시키는대로 해주면 된다. 마지막에 JS로 작성된 추적 코드를 주는데, 그걸 복사하고 잘 가지고 있자. 물론 지금 복사하지 않아도 나중에 언제든지 Analytics 대시보드에 들어와 복사할 수 있다.

추적 코드 집어넣기
========================

여기가 핵심이다. Analytics 웹사이트에는 추적할 모든 페이지에 추적 코드를 삽입하라고 겁준다. 여기서 쫄면 안된다. 정말 "모든" 웹사이트에 코드를 넣을 일은 없다. GitHub 페이지에 테마를 설치한 적이 있다면 아마 `_includes`라는 폴더가 존재할 것이다. 여기에 `google_analytics.html`이라는 파일을 하나 만들어 준다. 그리고 아래처럼 코드를 집어넣어 보자

  ```
    <!-- Google Analytics -->
    <script>
      (function(i,s,o,g,r,a,m){i['GoogleAnalyticsObject']=r;i[r]=i[r]||function(){
      (i[r].q=i[r].q||[]).push(arguments)},i[r].l=1*new Date();a=s.createElement(o),
      m=s.getElementsByTagName(o)[0];a.async=1;a.src=g;m.parentNode.insertBefore(a,m)
      })(window,document,'script','https://www.google-analytics.com/analytics.js','ga');

      ga('create', '내 추적 ID', 'auto');
      ga('send', 'pageview');

    </script>
  ```

이제 같은 폴더 안의 `head.html`에서 호출만 하면 된다. Jekyll은 표준 [Liquid](https://github.com/Shopify/liquid) 문법을 모두 지원원하니 `head.html` 파일의 `<head>` 태그 안 어디나     {% include google_analytics.html %}     블록을 집어넣어주면 작업 끝.

이제 GitHub에 Push하고 며칠 뒤 다시 Google Analytics를 확인해보자. 방문자 통계가 제대로 찍혀 있을 거다. 혹시 Analytics가 제대로 작동하는 것 같지 않다면 [Google Tag Assistant](https://get.google.com/tagassistant/?utm_source=google.com&utm_medium=notif_referral&utm_campaign=TRAFFIC_ANALYSIS_RECOMMENDATION) 를 사용해보자. 간단한 Chrome 에드온인데 내 사이트에서 Analytics가 제대로 작동 중인지 실시간으로 확인할 수 있는 듯 하다.
