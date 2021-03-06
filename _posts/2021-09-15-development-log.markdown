---
title:  "9월 15일 개발로그"
categories: dev-log SKKU Discord-bot Airflow ETC
---

## SKKU Notification bot
오늘 성대 공식 디스코드 채널에 봇 초대 URL을 올리고 봇을 초대해 달라고 했다. 3일간의 테스트를 거친 후에 공식적으로 채널에 배포될 것 같은데, 그동안 해야할 일이 많다.

우선 첫 번째로, 라즈베리파이 서버를 세팅해서 거기에 이식해야한다. 서버라고는 했지만 일단 외부 노출까지는 딱히 필요없으므로 포트 포워드 세팅 같은건 당장은 필요 없을 것 같다. 문제는 라즈베리파이 세팅이다. 임베디드 수업 때 전 굽던거 생각하면 치가 떨리는데 custom linux 같은 변태 같은걸 썼었으니 그렇게 헤매던것도 이번엔 공식 OS 같은걸 사용하면 좀 세팅이 원활하지 않을까 싶다.

두 번째로, 관리자가 봇의 설정을 만질 수 있도록 명령어 통로를 열어놔야 할 것 같다. 이 부분은 전혀 생각하지 못했던 건데, 생각해보니까 채널이 바뀔 때마다 디스코드 채널 관리자가 나한테 연락해야 된다면, 그건 잘못 만들어진 봇이다. 지금 생각은 봇에 명령어로 (!set_channel 같은 느낌으로) 관리자가 설정해 줄 수 있도록 만드는 건데, 그렇게 되면 명령어가 입력됐을 때 트리거할 이벤트 기반 로직을 짜야한다. 그러나 안타깝게도, Airflow에서는 이벤트 기반으로 DAG를 트리거시키는 기능을 갖고 있지 않는 것 같다. 따라서, 선택지는 다음과 같다.
1. 봇으로 정기적으로 알람을 전송해주는 Airflow dag와, discord.py등의 라이브러리를 사용한 이벤트 기반 로직 구현 코드를 병행해서 구현
2. 지금 있는거 다 밀고 discord.py 기반으로 새로 다 구현
3. DAG External trigger?? (실험 기능인 것 같다.)

지금 생각하고 있는건 1번이 제일 유력한데, 어쨌건 새로 라이브러리를 배우는 거라 좀 부담스럽긴 하다. 또한, discord.py로 작성한 코드도 docker화 해서 컨테이너로 띄워놓는 것을 목표로 하고 있기 때문에, 거기 까지 하려면 해야할 일이 좀 많이 있다.

세 번째로, 지금 봇이 보낼 채널 설정이나, 실행 주기를 줄여 보냈던 공지사항 포스트의 ID를 저장하는 것 둘 다 Stateful하게 저장해야한다는 거다. 전자의 경우 .cfg 파일 같은걸로 설정해줘도 될 것 같긴 한데, 후자의 경우는 어쩔수 없이 DB를 써야한다. 지금 현재 배포되어 있는 Postgres에 연동하는 것이 최선일 것 같은데, 시간을 많이 잡아먹을 것 같다.

## ETC
디스코드 봇을 만드는 건 좋은데 어쨌건 본업에 충실해야하긴 할 것 같다. 자율차 대회도 얼마 안남았고, 수업도 많이 밀렸다.