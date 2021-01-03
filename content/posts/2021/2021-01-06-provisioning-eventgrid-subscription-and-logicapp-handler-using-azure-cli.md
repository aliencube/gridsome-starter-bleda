---
title: "애저 CLI를 이용해 애저 이벤트그리드 구독 및 로직앱 이벤트 핸들러 프로비저닝하기"
slug: provisioning-eventgrid-subscription-and-logicapp-handler-using-azure-cli
description: "이 포스트는 애저 이벤트그리드 구독과 로직앱 이벤트 핸들러를 애저 CLI를 이용해서 별다른 외부 입력값 없이 자동으로 프로비저닝하는 방법에 대해 알아봅니다."
date: "2021-01-06"
author: Justin-Yoo
tags:
- azure-cli
- azure-logic-apps
- azure-eventgrid
- github-actions
cover: https://sa0blogs.blob.core.windows.net/aliencube/2021/01/provisioning-eventgrid-subscription-and-logicapp-handler-using-azure-cli-00.png
fullscreen: true
---

[애저 이벤트그리드][az evtgrd]라는 서비스를 이용하면 다양한 형태의 이벤트 기반 아키텍처를 구성할 수 있다. 특히 다양한 형태로 기존에 존재하는 애플리케이션간 이벤트 메시지를 전송하는데 있어서 꽤 유용하게 쓰인다.

![Azure Event Grid][image-01]


## 이벤트 기반 아키텍처의 세 요소 ##

이벤트 기반 아키텍처에는 중요한 역할을 하는 세 가지 요소가 있다.


### 이벤트 생성자/퍼블리셔 ###

이벤트 생성자/퍼블리셔는 이벤트가 발생하는 근원이다. 이벤트만 발생시킬 수 있다면 무엇이든 이벤트 생성자가 될 수 있다. 아래 그림은 지난 2018년 [오픈 인프라 데이즈][oid]에서 발표했던 [클라우드이벤트 CloudEvents][oid ce] 발표 내용의 일부를 캡쳐한 것이다. 아래 그림에서는 라즈베리파이 장치에서 이벤트를 생성하는 것에 대해 묘사하고 있다.

![Event Publisher][image-02]


### 이벤트 구독자/섭스크라이버/처리자/핸들러 ###

이벤트 구독자와 처리자는 엄밀히 말하면 다른 것이긴 하지만, 보통은 구독하면서 동시에 해당 이벤트를 처리하기 때문에 같은 것드로 봐도 큰 무리는 없다. 아래 그림에서는 이벤트 생성자가 보낸 이벤트를 받아 시각화 처리를 하는 형태를 묘사하고 있다.

![Event Subscriber][image-03]


### 이벤트 중개자/브로커 ###

일반적으로 이 이벤트 생성자와 구독자 사이에서는 직접적인 연결을 하지 않고 비동기식으로 진행하는데, 이 때 필요한 것이 바로 [애저 이벤트그리드][az evtgrd]와 같은 이벤트 중개자가 있다. 아래 그림은 바로 [애저 이벤트그리드][az evtgrd]가 이벤트 생성자와 구독자 사이에서 어떤 역할을 하는지 보여준다.

![Event Broker][image-04]

> [클라우드이벤트][ce]에 대해 좀 더 알고 싶다면, [발표 영상][oid yt]과 [발표 자료][oid ss]를 확인해 보자.


## 애저 이벤트그리드 구독자 생성하기 ##

일반적인 CI/CD 환경에서 애저 이벤트그리드 토픽을 생성할 때에는 [ARM 템플릿][az evtgrd arm topic]을 이용하면 쉽다. 하지만, 이벤트그리드 구독을 생성하는 [ARM 템플릿][az evtgrd arm sub]은 내가 생성한 이벤트그리드 토픽을 지정할 수 없기 때문에 여기서는 사용할 수 없다. 대신, [애저 CLI][az cli]를 사용해서 내가 지정한 토픽에 대해 구독을 생성하면 된다.


## 애저 CLI 사전 준비사항 ##

[애저 CLI][az cli]로 이벤트그리드 구독 리소스를 프로비저닝하기 위해서는 아래 [확장기능][az cli extensions]을 먼저 설치해야 한다. 여기서 [Logic][az cli extensions logic]은 [애저 로직앱][az logapp]을 위한 것인데, 이 포스트에서는 이벤트 처리자로서 로직앱을 사용하기 때문이다.

* [EventGrid][az cli extensions eventgrid]
* [Logic][az cli extensions logic]

> 이 두 확장기능은 이 글을 쓰는 현재 프리뷰 기능이며 언제든 구현이 바뀔 수 있다.


## 애저 CLI 명령어 ##

먼저 [애저 로직앱][az logapp]을 이벤트 처리자로 사용하기 위해서는 로직앱의 엔드포인트 URL을 알아내야 한다. 로직앱은 특성상 엔드포인트 뒤에 항상 [SAS 토큰][az logapp sas]이 붙어오기 때문에 이를 함께 불러와야 한다. 먼저 엔드포인트를 찾고자 하는 로직앱의 리소스 ID를 아래와 같이 `az logic workflow show` 명령어를 통해 알아낸다.

https://gist.github.com/justinyoo/8865213b31baeda9f7c1ad258351a039?file=01-az-logic-workflow-show.sh

그 다음에는 `az rest` 명령어를 통해 SAS 토큰이 붙어 있는 엔드포인트 값을 알아낸다.

https://gist.github.com/justinyoo/8865213b31baeda9f7c1ad258351a039?file=02-az-rest.sh

이제 로직앱을 이벤트 처리자로 사용하기 위한 준비는 끝났다. 다음 순서는 이벤트그리드 토픽의 리소스 ID를 아래와 같이 `az eventgrid topic show` 명령어를 통해 알아낸다.

https://gist.github.com/justinyoo/8865213b31baeda9f7c1ad258351a039?file=03-az-eventgrid-topic-show.sh

이벤트그리드 구독자를 생성하기 위한 모든 준비가 끝났다. 아래 `az eventgrid event-subscription create` 명령어를 통해 이벤트그리드 구독자를 생성한다. 여기서는 [CNCF][cncf]의 인큐베이팅 프로젝트인 [CloudEvents 스키마][ce]를 사용하는 것으로 한다.

https://gist.github.com/justinyoo/8865213b31baeda9f7c1ad258351a039?file=04-az-eventgrid-event-subscription-create.sh

위와 같은 방법으로 하면 곧바로 애저 이벤트그리드 구독자 리소스를 프로비저닝할 수 있다. 위 명령어를 한줄로 합치면 아래와 같다.

https://gist.github.com/justinyoo/8865213b31baeda9f7c1ad258351a039?file=05-one-liner.sh

---

지금까지 이벤트그리드 구독자를 프로비저닝하고 로직앱을 그 처리자로 설정하기 위해 애저 CLI를 이용하는 방법에 대해 알아 보았다. 이를 조금 더 활용하면 [깃헙 액션][gh actions]과 같은 CI/CD 파이프라인에서도 손쉽게 통합시킬 수 있을 것이다.


[image-01]: https://docs.microsoft.com/ko-kr/azure/event-grid/media/overview/functional-model.png?WT.mc_id=devops-12244-juyoo
[image-02]: https://sa0blogs.blob.core.windows.net/aliencube/2021/01/provisioning-eventgrid-subscription-and-logicapp-handler-using-azure-cli-02.png
[image-03]: https://sa0blogs.blob.core.windows.net/aliencube/2021/01/provisioning-eventgrid-subscription-and-logicapp-handler-using-azure-cli-03.png
[image-04]: https://sa0blogs.blob.core.windows.net/aliencube/2021/01/provisioning-eventgrid-subscription-and-logicapp-handler-using-azure-cli-04.png

[az cli]: https://docs.microsoft.com/ko-kr/cli/azure/what-is-azure-cli?WT.mc_id=devops-12244-juyoo
[az cli extensions]: https://docs.microsoft.com/ko-kr/cli/azure/azure-cli-extensions-list?WT.mc_id=devops-12244-juyoo
[az cli extensions eventgrid]: https://github.com/Azure/azure-cli-extensions/tree/master/src/eventgrid
[az cli extensions logic]: https://github.com/Azure/azure-cli-extensions/tree/master/src/logic

[az logapp]: https://docs.microsoft.com/ko-kr/azure/logic-apps/logic-apps-overview?WT.mc_id=devops-12244-juyoo
[az logapp sas]: https://docs.microsoft.com/ko-kr/azure/logic-apps/logic-apps-securing-a-logic-app?WT.mc_id=devops-12244-juyoo#generate-shared-access-signatures-sas

[az evtgrd]: https://docs.microsoft.com/ko-kr/azure/event-grid/overview?WT.mc_id=devops-12244-juyoo
[az evtgrd arm topic]: https://docs.microsoft.com/ko-kr/azure/templates/microsoft.eventgrid/topics?WT.mc_id=devops-12244-juyoo
[az evtgrd arm sub]: https://docs.microsoft.com/ko-kr/azure/templates/microsoft.eventgrid/eventsubscriptions?WT.mc_id=devops-12244-juyoo

[oid]: https://event.openinfradays.kr/2018/about/
[oid ce]: https://event.openinfradays.kr/2018/session1/track_4_0
[oid yt]: https://youtu.be/h2_ZNTXwlVc
[oid ss]: https://www.slideshare.net/openstack_kr/openinfra-days-korea-2018-track-4-cloudevents

[cncf]: https://www.cncf.io/
[ce]: https://cloudevents.io/

[gh actions]: https://docs.github.com/en/free-pro-team@latest/actions
