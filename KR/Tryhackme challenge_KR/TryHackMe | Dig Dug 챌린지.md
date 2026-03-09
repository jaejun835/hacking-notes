챌린지 링크 : https://tryhackme.com/room/digdug

이 과제에서는 플래그가 DNS의 TXT 레코드 안에 들어 있다는 것을 알 수 있다

보통이라면 아래처럼 둘 중 하나의 명령어를 사용하여 TXT 레코드를 조회하면 된다

```xml
dig txt <MACHINE IP>
```

또는

```xml
dig txt <domain>
```

하지만 이번 문제에서 서버는 givemetheflag.com 도메인에 대해서만 **특수한 방식의 요청**에 응답한다고 나온다

그래서 단순히 IP만 치는 게 아니라 도메인과 함께 DNS 서버(즉 머신 IP)를 명시해서 요청해야 한다

정확한 명령은 다음과 같다

```xml
dig txt <domain> @<MACHINE IP>
```

예를 들어

```xml
dig txt givemetheflag.com @<MACHINE_IP>
```

이렇게 실행하면 응답에 포함된 TXT 레코드에서 플래그를 확인할 수 있다

![dig-1.png](https://github.dev/jaejun835/hacking-notes/blob/main/KR/PortSwigger_API%20testing_KR/Lab%3A%20Exploiting%20an%20API%20endpoint%20using%20documentation.md)
