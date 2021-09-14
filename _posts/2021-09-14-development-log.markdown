---
layout: post
title:  "9월 14일, 현재 까지 개발로그"
categories: dev-log
---

## Github 블로그
깃헙 블로그는 처음 써보는지라 사람들이 하라는대로 [jekyll][jekyll]을 써서 블로그를 만들었다.  
마크다운 형식은 좀 써봤지만 아직 덜 익숙한 것 같다. 그래도 확실히 익숙해지면 작업 속도는 빠를듯.

## SKKU Notification bot
[공지사항 봇][skku_noti_bot]의 DAG는 대충 틀이 잡혔다.  
기본적으로 slack이나 discord에 메시지 쏴주는 것 까지 기능 구현이 다 되었다. 이 과정에서 Discord bot operator를 새로 만들었다.  
이제 세부사항을 좀 더 디벨롭 시켜야한다.

예를 들어.. 지금 DAG의 주기는 1일인데 이걸 1시간 이내로 줄일 계획이다. 아무래도 공지사항 알람을 받는 사람들은 봇이 실시간으로 공지사항이 올라올 때 마다 알람을 쏴주길 바랄테니까..  
문제는 이 기능을 위해서 DB와의 연동은 피할 수 없을 것 같다. 주기가 1일일때는 날짜가 다르니까 알람으로 전송하지 않은 새 공지사항을 쉽게 구별할 수 있었는데, 주기가 더 짧아지면 날짜는 같은데 어느 공지는 전송하고 어느 공지는 전송하지 않았고를 구분해야 되기 때문이다.  
Airflow 때문에 Postgres는 띄워놓은 상태이니 그걸 쓰면 되지 않을까 생각한다.

그 외에도 여러가지를 고려중이다. 지금은 공지의 제목만 보내주고 있는데 내용도 미리보기 형식으로 보여줄까 생각중이고, 메시지 스타일이야 당연히 지금은 너무 투박하다. 세련된 방식으로 포맷팅해야 할 것 같다.


## Airflow Discord bot operator
내가 만들고 있는 [성대 공지사항 봇][skku_noti_bot]은 기본적으로 Airflow DAG를 기반으로 한다. 문제는, 기존의 Airflow에서 지원하는 Discord 오퍼레이터는 채널의 webhook을 사용하는데, 내가 원한 것은 봇을 사용해서 메시지를 전송하는거 였기 때문이다. 기존의 방식을 그대로 차용할 경우 관리자한테 Webhook 설정등을 요청해야 하지만, 봇 오퍼레이터를 새로 만들 경우 그럴 필요 없이 관리자가 봇을 초대하기만 하면 되고, 봇의 설정은 내가 다 따로 할 수 있다. 그리고 나중에 메시지를 전송하는 기능 뿐만 아니라 다른 기능까지 집어넣으려면 그게 더 확장성 있는것 같기도 하고 말이다. 그래서 아예 오퍼레이터를 새로 만들기로 했다.

기본적으로 기존 Discord webhook operator와 매우 유사한 구조를 가진다. HTTP 훅을 사용해 메시지를 전달하는 형식인데, 기존의 코드가 채널의 Webhook을 사용해 REST Api로 메시지를 전송했다면, 이 경우에는 채널의 id만 따와서 Bot으로 전송하는 방식이다. 두 가지 방식의 큰 차이점은 인증 방식인데, 기존 코드의 경우 Webhook 엔드포인트에서 인증 토큰을 정하는 반면, 봇의 경우 엔드포인트는 channel의 정보만 담고 있고 인증은 헤더에서 `Authorization: Bot $TOKEN`을 포함하는 방식이다.

포스트를 쓰면서 깨달은 건데 지금 포함하고 있는 기능에서는 딱히 새 Bot operator가 필요하지 않다. 그래도 여전히 기존 코드는 발전이 필요한게, 기존 코드의 경우 그냥 일반 Text만 전송할 수 있기 때문이다. 
```python
    # discord_webhook.py
    def _build_discord_payload(self) -> str:
        """
        Construct the Discord JSON payload. All relevant parameters are combined here
        to a valid Discord JSON payload.

        :return: Discord payload (str) to send
        """
        payload: Dict[str, Any] = {}

        if self.username:
            payload['username'] = self.username
        if self.avatar_url:
            payload['avatar_url'] = self.avatar_url

        payload['tts'] = self.tts

        if len(self.message) <= 2000:
            payload['content'] = self.message # Only content!
        else:
            raise AirflowException('Discord message length must be 2000 or fewer characters.')

        return json.dumps(payload)
```
Discord에서 지원하는 좀 더 Fancy한 기능들인 Embed object나 기타 등등을 사용하려면 아예 Json 형태로 사용자측에서 구성하게 하는것도 더 확장성 있지 않나 하는 생각을 한다. 문제는 Airflow측에서 이것에 관심이 있나 하는건데.. 뭐 그건 나중에 이슈를 새로 열어서 물어보던가 해야겠다.


[jekyll]: https://jekyllrb.com/
[skku_noti_bot]: https://github.com/Sonins/SKKU-Notification-Bot-Dag