이글은 THM의 Linux PrivEsc Arena 룸에 대한 write-up 글이다

Linux PrivEsc Arena 룸은 총 19개의 Task로 이루어져 있으며  including kernel exploits, sudo attacks, SUID attacks 등 리눅스 권한 상승의 실전 연습을 목표로 구성되었다

룸에 대한 자세한 정보는 아래의 링크에서 찾아 볼 수 있다

[→ https://tryhackme.com/room/linuxprivescarena](https://tryhackme.com/room/linuxprivescarena)

[Linux PrivEsc Arena
Students will learn how to escalate privileges using a very vulnerable Linux VM. SSH is open. Your credentials are TCM:Hacker123
tryhackme.com](https://tryhackme.com/room/linuxprivescarena)

### [ Task2 - Deploy the vulnerable machine ]

※Task1은 Openvpn을 이용하여 THM의 네트워크에 연결하는 단계이므로 넘어가도록 하겠다

이번 룸에선 리눅스 권한 상승에 초점이 맞춰진 룸이다 보니 SSH 크리덴셜 확보 과정은 넘어가는 듯 하다

그러니 간단하게 주어진 자격증명(TCM:Hacker123)으로 SSH에 접속만 해주면 된다

**명령어:**

```
ssh -p 22 TCM@<IP>
```

**결과:**

![](https://blog.kakaocdn.net/dna/cy9eJ7/dJMcabJ95Cp/AAAAAAAAAAAAAAAAAAAAAEFqsX6gHaryMSh0pymfc2E0M_fmklOhdfcKP-w0gJYt/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1774969199&allow_ip=&allow_referer=&signature=cyIczcXqC7MTBmb5czZ4DaBMyJc%3D)

만약 표준 연결 명령어를 사용하였을 때 SSH 클라이언트와 서버간의 버전 차이 때문에 연결이 안될수도 있다

※최신 OpenSSH 클라이언트는 보안상 취약한 ssh-rsa 알고리즘을 기본적으로 비활성화하기 때문에 구버전 서버와의 호스트키 알고리즘 협상이 실패하는 현상이 일어남

그럴경우 아래와 같이 HostKey Algorithms 옵션을 사용하여 레거시(구형) 알고리즘인 ssh-rsa를 허용하도록 강제한다

**명령어:**

```
ssh -o "HostKeyAlgorithms=+ssh-rsa" TCM@<IP>
```

**결과:**

![](https://blog.kakaocdn.net/dna/L5R0l/dJMcagq8aXR/AAAAAAAAAAAAAAAAAAAAAOawBboWufijmdfd1DwSEx_OECdzL51qEkjWgQfBzldf/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1774969199&allow_ip=&allow_referer=&signature=mTtD7SPRHK4tO%2Fd%2FHSJ0O7ajgKU%3D)

명령어를 사용하면 사진과 같이 ssh 접속이 성공한 것을 확인 할 수 있다

### [ Task3 - Privilege Escalation - Kernel Exploits ]

앞서 Task2에서 획득한 ssh 셸로 미리 설치된 linux-exploit-suggester.sh를 실행하면 현재 시스템의 커널 버전을 분석하여 사용 가능한 익스플로잇 목력을 확인할 수 있다

※실전 상황에선 익스플로잇 도구가 타겟 머신에 존재하지 않으므로 wget,scp 또는 python으로 웹 서버 등을 이용해 직접 업로드 해야 한다

**명령어:**

```
/home/user/tools/linux-exploit-suggester/linux-exploit-suggester.sh
```

**결과:**

![](https://blog.kakaocdn.net/dna/b7PZa2/dJMcafeJw3y/AAAAAAAAAAAAAAAAAAAAAJ2dZU-HrKMW_RHlxFV47oASsJ8Lxu1g-27rtRauNhJ9/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1774969199&allow_ip=&allow_referer=&signature=QPrnMYoFqNooJOzSrlZm9xUKF7A%3D)

![](https://blog.kakaocdn.net/dna/cQiAG7/dJMcaiCsWbz/AAAAAAAAAAAAAAAAAAAAAKa_jEs8m8tX22yX6KmCUE9HZJgWApc7r1jhzC30nm4r/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1774969199&allow_ip=&allow_referer=&signature=DZarEQJeYeEQtZOk8TneZSHQBAY%3D)

![](https://blog.kakaocdn.net/dna/mIUzD/dJMcaiJdzmh/AAAAAAAAAAAAAAAAAAAAAEW7EZNWPFHzAAZVVN8DGMaqTFZrRzzzVSDo2BYzMdDG/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1774969199&allow_ip=&allow_referer=&signature=1DgnZMJHFQn%2BWyycajTkB10uvNc%3D)

![](https://blog.kakaocdn.net/dna/onD9k/dJMcaaYKrmj/AAAAAAAAAAAAAAAAAAAAANPXzutcOcwi7q9rdEQmQgOs21meD51QxKpdIg6ccFTs/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1774969199&allow_ip=&allow_referer=&signature=jof3Lx01070kWPFzKAsYtgcmebU%3D)

![](https://blog.kakaocdn.net/dna/ZreSv/dJMcaadoWzN/AAAAAAAAAAAAAAAAAAAAAOLjgAtDdTWRoiQ53xYAmnH60RhppsfRhuFMl6uj0_n9/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1774969199&allow_ip=&allow_referer=&signature=FEKSHpaDZjLO6OGLJ2y5gO5v3aI%3D)

출력 결과 다수의 익스플로잇이 발견되었으며 이 중 [CVE-2016-5195] dirtycow 를 이용해 권한 상승을 진행하겠다

※목록에서 Dirtycow를 선택한 이유는 현재 시스템(debian)이 취약 범위에 포함되며 신뢰성이 높고 검증된 익스플로잇이기 때문이다

Dirtycow 취약점을 이용하기 위해선 /usr/bin/passwd 바이너리를 root 쉘로 바꿔치기 해서 일반 유저가 root권한을 얻을 수 있도록 만들어주면 된다

※Dirtycow는 Race Condition 취약점이다

**Dirtycow 취약점 원리:**

```
Race Condition:
두 개의 스레드가 동시에 같은 자원에 접근할 때 타이밍 차이로 인해 의도치 않은 결과가 나오는 것

DirtyCow 원리:
리눅스 커널(데비안 포함)에는 Copy-on-Write(CoW)라는 메커니즘이 있다

패치 전 CoW 원리:
1.읽기 전용 파일을 수정하려 하면 커널이 복사본을 만들어서 수정하게 함
2.원본은 건드리지 못하도록 보호

Dirtycow 취약점 원리:
1.스레드1: 복사본 만드는 중
2.스레드2: 그 찰나의 순간에 원본에 접근
3.타이밍이 맞으면 읽기 전용인 원본 파일을 수정 가능

이걸 이용해서 root 소유의 /usr/bin/passwd를 일반 유저가 덮어쓸 수 있게 된다
```

**명령어:**

```
(1 - 익스플로잇 코드 컴파일 + 명령어 기능)

# gcc -pthread /home/user/tools/dirtycow/c0w.c -o c0w
1.c0w.c 소스코드는 그냥 텍스트 파일이라 실행이 불가능하기 때문에 gcc로 컴파일해준다
2.-pthread 옵션은 Dirtycow가 Race Condition 기반 취약점이기 때문에 멀티스레드를 사용할 수 있도록 해준다

(2 - 익스플로잇 실행 + 명령어 기능)

# ./c0w
1.Dirtycow 취약점을 이용해 /usr/bin/passwd 바이너리를 root 셸로 교체한다

(3 - root 권한 획득 + 명령어 기능)

# passwd
1./usr/bin/passwd가 root 셸로 교체된 상태이므로 실행하면 root 셸이 실행된다

(4 - 권한 확인 + 명령어 기능)

# id
1.uid=0(root)가 출력되면 권한 상승 성공이다

(5 - 복구 + 명령어 기능)

# cp /tmp/bak /usr/bin/passwd
1.2단계에서 백업해둔 원본 passwd를 되돌려놓아 머신을 원래 상태로 복구
2.작업 중 실행하면 root 권한을 잃기 때문에 작업 완료 후 실행해야한다
```

**결과:**

![](https://blog.kakaocdn.net/dna/LwI1A/dJMcadA6Mz2/AAAAAAAAAAAAAAAAAAAAANCuv34p_BjXWXEo72DTNs3mpoKH5W2hEnVoQ1EVBi-t/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1774969199&allow_ip=&allow_referer=&signature=yfQgLdBS%2FXZOsZzzPgZtvnk67kQ%3D)

![](https://blog.kakaocdn.net/dna/cbIBXp/dJMcahKmUNh/AAAAAAAAAAAAAAAAAAAAAB7WXAc9PwBkSaSTeXQWVZjb76aFuaBhT4gmw4VZ4QKb/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1774969199&allow_ip=&allow_referer=&signature=yVH1jh1gAKi18UvyzAzHwsXaZXg%3D)

실행 결과 root 권한을 획득할 수 있었다
