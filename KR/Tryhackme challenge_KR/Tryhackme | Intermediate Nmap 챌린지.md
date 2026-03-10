# Tryhackme | Intermediate Nmap 챌린지

tryhackme 링크 → https://tryhackme.com/room/intermediatenmap

TryHackMe의 Intermediate Nmap 룸을 풀어보았다

Nmap으로 리콘을 수행하고 커스텀 서비스에서 힌트를 찾아 SSH로 접근한 뒤 플래그를 획득하는 흐름으로 진행하였다

먼저 정보 수집의 가장 기본이 되는 포트 스캔을 실시하였다

```
sudo nmap -sS --min-rate 5000 -p- <IP>
```

![1](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80/1.Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

3개의 포트가 모두 열려있는 것을 확인했으니 이제 각 포트에서 어떤 서비스가 동작하고 있는지 간단하게 설명하겠다

가장 먼저 31337은 비표준 포트이다

CTF나 보안 실습에서는 이런 비표준 포트에 힌트나 특별한 정보를 제공하는 커스텀 서비스가 동작하는 경우가 종종 있다

그래서 가장 큰 포트 번호부터 확인하는 것이 효율적이라고 판단하였다

```
nc <IP> 31337
```

![2](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80/2.Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

netcat을 사용해서 연결했다

nc(netcat)는 TCP/UDP 연결을 만들고 데이터를 주고받을 수 있는 네트워킹 도구이며 웹 브라우저 없이도 서비스와 직접 통신할 수 있어서 원시 데이터를 확인하기에 적합하다

확인해보니 커스텀 서비스가 평문으로 사용자:비밀번호 형태의 자격 증명을 출력하고 있었다

이것이 SSH 접속에 필요한 인증 정보라는 것을 직감했다

CTF 실습에서는 이처럼 의도적으로 힌트를 제공하는 경우가 많다

다음으로는 SSH포트를 분석하였다

SSH 포트가 22번과 2222번 두 개가 열려있어서 어느 포트로 접속할 수 있는지 파악해야 했다

가장 먼저 각 포트의 인증 방식을 확인해봤습니다

22번 포트는 표준 SSH 포트로 비밀번호 인증이 가능했다.

![3](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80/3.Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

![4](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80/4.Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

반면 2222번 포트는 대체 SSH 포트이긴 하지만 공개키 인증만 허용하고 있어서 비밀번호 인증은 불가능했다

31337번 포트에서 얻은 자격 증명이 비밀번호 형태였기 때문에 22번 포트로 접속을 시도해야 했다

![5](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80/5.Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

22번 포트로 먼저 접속하였다

```
ssh user@<IP> -p 22
```

![6](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80/6.Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

31337번 포트에서 얻은 비밀번호를 입력했고 성공적으로 로그인되었다

로그인 후 현재 위치를 확인하니 /home/ubuntu에 있었고 상위 디렉토리로 이동해서 파일을 탐색했다

![7](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80/7.Tryhackme%C2%A0%7C%20Intermediate%20Nmap%20%EC%B1%8C%EB%A6%B0%EC%A7%80.jpg)

```bash
pwd
cd ..
ls
```

ubuntu와 user 두 디렉토리가 있었다

먼저 user 디렉토리로 이동해서 파일을 확인했다

```bash
cd user
ls
```

flag.txt 파일을 발견했다

```css
cat flag.txt
```
