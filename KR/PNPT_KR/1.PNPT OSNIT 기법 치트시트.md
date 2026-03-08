이 글은 TCM Security의 PNPT(Practical Network Penetration Tester) 자격증 취득을 위한 OSINT(Open Source Intelligence) 기법 치트시트 이다

실제 시험과 현업 침투 테스트에서 사용되는 흐름 그대로 단계별로 정리하였다

### [OSINT 공격 흐름]

OSINT는 아무런 접근 없이 공개된 정보만으로 대상을 파악하는 단계이다

```bash
[대상 도메인/조직 정보만 존재]

1단계. 이메일 수집 및 검증
  대상 조직의 이메일 주소와 패턴 파악
  (hunter.io, phonebook.cz, voilanorbert ...)

[이메일 목록 + 이메일 패턴 확보]

2단계. 서브도메인 열거
  공격 가능한 외부 인프라 전체 파악
  (sublist3r, crt.sh, amass, httprobe, gowitness ...)

[외부 공격 표면 지도 완성]

3단계. 유출 데이터 검색
  과거 유출된 자격증명 확보
  (dehashed, breach-parse, haveibeenpwned ...)

[이메일:패스워드 쌍 확보 → 비밀번호 스프레이 준비]

4단계. Google Dorking
  구글 인덱스에서 민감한 정보 탐색
  (filetype:, inurl:, site:, intitle: ...)

[로그인 포털, 설정 파일, 내부 문서 발견]

5단계. 소셜 미디어 OSINT
  직원 정보와 조직 구조 파악
  (linkedin2username, sherlock, twitter ...)

[직원 이름 → 유저명 목록 생성]

6단계. DNS 정찰
  네트워크 인프라 구조 매핑
  (dig, nslookup, dnsrecon, whois ...)

[네임서버, 메일서버, IP 대역 파악]

7단계. 웹사이트 기술 식별 & 문서 메타데이터 분석
  기술 스택과 내부 정보 수집
  (whatweb, builtwith, exiftool, FOCA ...)

[소프트웨어 버전, 내부 경로, 작성자 이름 획득]

[OSINT 완료 → 외부 침투 단계로 이동]
```

OSINT는 대상 서버에 직접 접근하지 않으므로 흔적이 남지 않는다

이 단계에서 발견한 자격증명과 이메일 목록들이 이후의 외부 침투로 이어진다

### [1단계 : 이메일 수집 및 검증]

이메일 수집 단계에선 대상 조직의 이메일 주소와 네이밍 패턴을 파악하는게 목표이다

(이메일 패턴(예: first.last@company.com)을 파악하면 LinkedIn(5-1에서 자세한 설명)에서 수집한 직원 이름으로 전체 이메일 목록(패턴 기반 워드리스트)을 생성할 수 있기 때문이다)

### [1-1 : hunter.io]

hunter.io는 공개적으로 인덱싱된 이메일 주소를 도메인 기반으로 검색하는 플랫폼이다 = (도메인 기반 이메일 스크래핑)

무료 계정으로 월 25회 검색 + 50회 검증이 가능하다

**사용 조건:**

```css
1.hunter.io 계정 생성 (무료)
2.대상 도메인 보유 (예: tesla.com)
```

**공격 흐름 + 툴 & 명령어:**

```perl
(공격 흐름)

1.hunter.io Domain Search에 대상 도메인 입력
2.이메일 주소 목록 + 이메일 패턴 확인 (예: {first}.{last}@domain.com)
3.각 이메일의 출처 URL과 신뢰도 확인
4.Email Finder로 이름 + 도메인 입력 → 특정 인물 이메일 확인
5.Email Verifier로 수집한 이메일 유효성 검증

------------------------------------------------------------------

(API 명령어)

# 도메인 검색
curl "https://api.hunter.io/v2/domain-search?domain=target.com&api_key=YOUR_KEY"

# 특정 인물 이메일 찾기
curl "https://api.hunter.io/v2/email-finder?domain=target.com&first_name=John&last_name=Doe&api_key=YOUR_KEY"

# 이메일 유효성 검증
curl "https://api.hunter.io/v2/email-verifier?email=john@target.com&api_key=YOUR_KEY"
```

### 

### [1-2 : phonebook.cz]

Intelligence X가 운영하는 phonebook.cz는 2,680억 건 이상의 레코드에서 도메인 관련 이메일, 서브도메인, URL을 검색한다

hunter.io와 다른 데이터 소스(유출 데이터 포함)를 사용하므로 상호 보완적으로 사용해야 한다

**사용 조건:**

```less
1.대상 도메인 보유
2.별도 계정 없이 사용 가능 (횟수 제한 있음)
```

**공격 흐름:**

```scss
(공격 흐름)

1.phonebook.cz 접속
2.검색 유형 선택 (Domains / Email Addresses / URLs)
3.대상 도메인 입력
4.결과에서 이메일 주소, 서브도메인 수집
5.hunter.io 결과와 합산하여 최종 이메일 목록 생성
```

### 

### [1-3 : VoilaNorbert]

VoilaNorbert는 인물의 전체 이름 + 회사 도메인을 입력하면 검증된 이메일을 반환한다

가입 시 50회 무료 크레딧을 제공하며, 확신도 점수(녹색=높음, 주황=미확인)로 결과의 신뢰도를 표시해준다

**사용 조건:**

```
1.voilanorbert.com 계정 생성
2.대상 인물의 이름 + 소속 도메인 보유
```

**공격 흐름:**

```scss
(공격 흐름)

1.voilanorbert.com 접속 후 이름 + 도메인 입력
2.확신도 점수와 함께 이메일 반환
3.Chrome 확장 프로그램으로 LinkedIn 프로필 브라우징 중 자동 탐색 가능
4.수집된 이메일 목록에 추가
```

### 

### [1-4 : 이메일 유효성 검증]

마지막으로 앞서 수집한 이메일의 유효성을 반드시 검증해야 한다

존재하지 않는 이메일로 피싱을 보내면 보안 팀에 경고가 발생하기 때문이다

**주요 도구:**

```perl
# Email Hippo
https://tools.verifyemailaddress.io
→ Valid / Invalid / Trashmail(일회용) 판정

# email-checker.net
https://email-checker.net/validate
→ MX 레코드, SMTP 응답, 메일박스 존재 여부 확인

# hunter.io Email Verifier (API)
curl "https://api.hunter.io/v2/email-verifier?email=john@target.com&api_key=YOUR_KEY"
```

### [2단계 : 서브도메인 열거]

서브도메인 열거 단계에선 대상의 공격 표면을 극대화하는 것이 목표이다

이를 잘 활용 할 경우 잊혀진 개발 환경, 관리자 패널, 취약한 레거시 서비스 등을 발견할 수 있다

※서브 도메인 = 같은 조직이 운영하는 여하위 주소들 = 메인 주소들에 비해 관리가 허술한 경우가 많음

### [2-1 : Sublist3r]

Sublist3r는 Google, Bing, Yahoo, Baidu 등 다중 검색엔진에서 서브도메인을 자동으로 수집하는 도구이다

**사용 조건:**

```
1.Python 환경
2.대상 도메인 보유
```

**공격 흐름 + 툴 & 명령어:**

```bash
(공격 흐름)

1.대상 도메인으로 Sublist3r 실행
2.Google, Bing 등 다중 소스에서 서브도메인 수집
3.브루트포스 옵션(-b)으로 추가 서브도메인 탐색
4.결과를 파일로 저장하여 다음 단계 입력으로 사용

--------------------------------------------------------------

(설치)

pip install sublist3r
# 또는
git clone https://github.com/aboul3la/Sublist3r.git
cd Sublist3r && pip install -r requirements.txt

--------------------------------------------------------------

(명령어)

# 기본 실행
sublist3r -d target.com

# 옵션 설명
# -d DOMAIN   : 대상 도메인 (필수)
# -b          : subbrute 브루트포스 모듈 활성화
# -t THREADS  : 스레드 수 (브루트포스용)
# -e ENGINES  : 검색엔진 지정 (google,bing,yahoo,baidu,dnsdumpster...)
# -o OUTPUT   : 결과 파일 저장
# -p PORTS    : 발견된 서브도메인에 TCP 포트 스캔
# -v          : 실시간 결과 표시

# 전체 옵션 활용
sublist3r -d target.com -b -t 100 -e google,bing,yahoo -o sublist3r.txt -v
```

### 

### [2-2 : crt.sh]

crt.sh는 SSL/TLS 인증서 투명성(CT) 로그를 검색하여 서브도메인을 발견하는 서비스이다

조직이 staging.target.com에 대한 인증서를 발급받으면 그 정보가 공개 CT 로그에 기록되므로, DNS 브루트포스로 발견할 수 없는 서브도메인까지 노출된다

※인증서는 도메인 단위로 발급 됨 + 발급할 때마다 CT 로그에 기록 됨 = crt.sh에서 검색할 시 도메인 열거 가능

**사용 조건:**

```
1.대상 도메인 보유
2.별도 설치 없이 웹 또는 curl로 사용 가능
```

**공격 흐름 + 명령어:**

```perl
(공격 흐름)

1.crt.sh에서 %.target.com 검색
2.발급된 인증서 목록에서 서브도메인 추출
3.와일드카드(*.target.com) 인증서도 개별 서브도메인 노출 가능
4.결과를 Sublist3r 결과와 합산

--------------------------------------------------------------

(명령어)

# 웹 인터페이스
# https://crt.sh 에서 %.target.com 입력

# 커맨드라인 (JSON 파싱)
curl -s "https://crt.sh/?q=%25.target.com&output=json" | jq -r '.[].name_value' | sed 's/\*\.//g' | sort -u > crt.txt
```

### 

### [2-3 : Amass]

Amass는 OWASP에서 만든 가장 강력한 공격 표면 매핑 도구(조직의 모든 진입점을 지도처럼 그려주는 도구)이다

DNS 검증, 역방향 WHOIS, 인증서 수집, ASN 탐색까지 광범위한 정찰이 가능하다

※ASN 파악 = 조직이 소유한 전체 IP 대역 파악

**사용 조건:**

```
1.Go 환경 또는 snap으로 설치
2.API 키 설정 시 더 많은 결과 수집 가능
```

**공격 흐름 + 툴 & 명령어:**

```csharp
(공격 흐름)

1.passive 모드로 대상 흔적 없이 서브도메인 수집
2.active 모드로 DNS 검증 + 영역 전송 시도
3.브루트포스 옵션으로 wordlist 기반 서브도메인 탐색
4.결과를 다른 도구 결과와 합산

--------------------------------------------------------------

(설치)

go install github.com/owasp-amass/amass/v4/...@master
# 또는
sudo snap install amass

--------------------------------------------------------------

(명령어)

# 수동 열거 (스텔스, 흔적 없음)
amass enum -passive -d target.com -o amass_passive.txt

# 능동 열거 (DNS 검증, 영역 전송 시도, 인증서 수집)
amass enum -d target.com -active -o amass_active.txt

# 브루트포스 포함
amass enum -d target.com -brute -w /usr/share/wordlists/dnsmap.txt

# 데이터 소스 표시 + IP 포함
amass enum -d target.com -src -ip

# ASN으로 인텔리전스 수집
amass intel -asn 12345

# 역방향 WHOIS
amass intel -d target.com -whois
```

### 

### [2-4 : httprobe]

httprobe는 서브도메인 목록에서 실제 동작하는 웹 서비스만 걸러내는 도구이다

(HTTP/HTTPS 응답이 있는 호스트만 필터링하여 유요한 서비스만 걸러냄)

**사용 조건:**

```
1.Go 환경
2.서브도메인 텍스트 파일 보유
```

**공격 흐름 + 명령어:**

```coffeescript
(공격 흐름)

1.수집된 서브도메인 목록을 httprobe에 파이프
2.HTTP/HTTPS 응답이 있는 호스트만 필터링
3.커스텀 포트(8080, 8443 등) 추가 탐색
4.결과를 GoWitness에 입력

--------------------------------------------------------------

(설치 + 명령어)

go install github.com/tomnomnom/httprobe@latest

# 기본 사용
cat subdomains.txt | httprobe

# 커스텀 포트 + 높은 동시성
cat subdomains.txt | httprobe -p http:8080 -p https:8443 -c 50

# 옵션 설명
# -c         : 동시 연결 수 (기본 20)
# -t         : 타임아웃 ms (기본 10000)
# -p         : 추가 프로토콜:포트
# --prefer-https : HTTPS 우선
```

### [2-5 : GoWitness]

GoWitness는 라이브(유요한) URL 목록을 입력받아 각 페이지의 스크린샷을 자동으로 찍어주는 도구이다

로그인 포털, 관리자 패널, API 엔드포인트(서버의 특정 기능을 호출할 수 있는 주소)를 눈으로 확인하여 공격 대상을 선정할 수 있다

**사용 조건:**

```
1.Go 환경
2.httprobe 결과 파일 보유
```

**공격 흐름 + 명령어:**

```perl
(공격 흐름)

1.httprobe 결과 파일을 GoWitness에 입력
2.각 URL 스크린샷 자동 캡처 + 데이터베이스 저장
3.웹 뷰어에서 스크린샷 분류하며 공격 대상 선정
4.로그인 포털, 관리자 패널 → 자격증명 스프레이 대상으로 분류

--------------------------------------------------------------

(설치 + 명령어)

go install github.com/sensepost/gowitness@latest

# 파일에서 스크린샷
gowitness scan file -f alive.txt --write-db

# 단일 URL
gowitness scan single --url "https://target.com" --write-db

# 결과 웹 뷰어
gowitness report server
# → http://localhost:7171 접속
```

### [2-6 : 전체 서브도메인 열거 워크플로우]

TCM 과정에서 가르치는 서브도메인 → 라이브 확인 → 스크린샷의 전체 흐름이다

```perl
# Step 1: 다중 도구로 서브도메인 수집
sublist3r -d target.com -o sublist3r.txt
curl -s "https://crt.sh/?q=%25.target.com&output=json" | jq -r '.[].name_value' | sort -u > crt.txt
amass enum -passive -d target.com -o amass.txt

# Step 2: 중복 제거 후 통합
cat sublist3r.txt crt.txt amass.txt | sort -u > all_subdomains.txt

# Step 3: 라이브 호스트 확인
cat all_subdomains.txt | httprobe > alive.txt

# Step 4: 스크린샷 + 분석
gowitness scan file -f alive.txt --write-db
gowitness report server
```

### [3단계 : 유출 데이터 검색]

유출 데이터 검색 단계에선 과거 데이터 유출 사건에서 흘러나온 자격증명을 확보하는것이 목표이다

발견된 자격증명으로 VPN, OWA, O365 포털에 비밀번호 스프레이 공격을 수행할 수 있다

### [3-1 : DeHashed]

DeHashed는 133억 건 이상의 유출 레코드를 보유한 가장 방대한 유출 데이터 검색 플랫폼이다

이메일, 사용자명, IP, 이름, 주소, 전화번호, 비밀번호, 해시, 도메인 등 다양한 필드로 검색 가능하다

**사용 조건:**

```less
1.dehashed.com 유료 계정 (API 사용 시 필요)
2.또는 Heath Adams의 DeHashed API Tool 사용
```

**공격 흐름 + 툴 & 명령어:**

```perl
(공격 흐름)

1.dehashed.com에서 대상 도메인 검색
2.이메일:패스워드 쌍 확보
3.평문 패스워드가 없으면 해시만 존재 → 크랙 시도 or 패턴 분석
4.패스워드 패턴 파악 (Password1 → Password2024! 변형 등)
5.수집된 자격증명으로 비밀번호 스프레이 준비

--------------------------------------------------------------

(Heath Adams DeHashed API Tool 설치 + 명령어)

pipx install git+https://github.com/hmaverickadams/DeHashed-API-Tool

# API 키 저장
dat --key  --store-key

# 도메인으로 비밀번호만 추출
dehashapitool -d target.com --only-passwords

# 이메일로 검색, CSV 출력
dehashapitool -e user@target.com --output results.csv

# 사일런트 CSV 출력
dehashapitool -d target.com --only-passwords -oS results.csv

# 검색 플래그
# -e : 이메일   -u : 사용자명   -i : IP
# -n : 이름     -a : 주소       -d : 도메인
# -p : 비밀번호 -H : 해시
```

### [3-2 : breach-parse]

breach-parse는 Heath Adams가 직접 만든 bash 스크립트로 BreachCompilation 데이터셋에서 대상 도메인의 자격증명을 추출한다

**사용 조건:**

```jsx
1.BreachCompilation 데이터셋 다운로드 (BitTorrent, 약 87GB)
2.기본 경로: /opt/breach-parse/BreachCompilation/data
```

**공격 흐름 + 툴 & 명령어:**

```lua
(공격 흐름)

1.BreachCompilation 데이터셋 로컬에 보유
2.breach-parse로 대상 도메인 검색
3.output-master.txt에서 email:password 쌍 확인
4.output-passwords.txt를 비밀번호 스프레이 wordlist로 활용

--------------------------------------------------------------

(설치 + 명령어)

git clone https://github.com/hmaverickadams/breach-parse.git
cd breach-parse && sudo ./install.sh

# 기본 사용
breach-parse @target.com output.txt

# 커스텀 데이터 경로
breach-parse @target.com output.txt "~/BreachCompilation/data"
```

**출력 파일:**

```lua
output-master.txt    → email:password 쌍 전체
output-users.txt     → 이메일/사용자명만
output-passwords.txt → 비밀번호만 (스프레이용)
```

### [3-3 : Have I Been Pwned (HIBP)]

HIBP(haveibeenpwned.com)는 이메일이 어떤 유출 사건에 포함되었는지 확인하는 서비스이다

실제 비밀번호는 공개하지 않지만, 유출된 서비스명과 노출된 데이터 유형을 알려준다

**사용 조건:**

```less
1.별도 설치 없이 웹에서 사용 가능
2.API 사용 시 키 필요 (유료)
```

**공격 흐름:**

```csharp
(공격 흐름)

1.haveibeenpwned.com에서 대상 이메일 검색
2.어떤 유출 사건에 포함되었는지 확인 (Adobe, LinkedIn, RockYou 등)
3.어떤 데이터가 노출되었는지 확인 (이메일, 비밀번호, 전화번호 등)
4.유출 사건 이름 → dehashed에서 해당 데이터 검색

--------------------------------------------------------------

(비밀번호 직접 확인)

# k-anonymity 방식으로 안전하게 비밀번호 유출 여부 확인
https://haveibeenpwned.com/Passwords

# SHA-1 해시 앞 5자리로 API 쿼리
curl https://api.pwnedpasswords.com/range/21BD1
```

### [4단계 : Google Dorking]

Google Dorking 단계에선 구글 검색 연산자를 활용해 인덱싱(검색 가능하게 수집·정리된 상태)된 민감한 정보를 탐색하는것이 목표이다

대상 서버에 직접 접근하지 않으므로 완전히 수동적인 정찰이라고 볼 수 있다

### [4-1 : 핵심 연산자]

Google Dorking을 사용하려면 Google 검색 연산자와 파라미터 사이에 공백 없이 작성하면 된다

※ex) inurl:admin O, inurl: admin X

```groovy
site:        → 특정 도메인으로 결과 제한     (site:target.com)
filetype:    → 파일 형식 지정               (filetype:pdf)
ext:         → 확장자 지정 (filetype과 동일)  (ext:sql)
inurl:       → URL에 키워드 포함            (inurl:admin)
intitle:     → 페이지 제목에 키워드         (intitle:"index of")
intext:      → 본문에 키워드               (intext:password)
cache:       → Google 캐시 버전            (cache:target.com)
""           → 정확한 구문 매칭            ("confidential report")
-            → 결과 제외                   (site:target.com -www)
OR / |       → OR 연산                    (filetype:pdf OR filetype:doc)
before:      → 날짜 이전                   (before:2024-01-01)
after:       → 날짜 이후                   (after:2024-01-01)
```

### [4-2 : PNPT 유용 Dork 모음]

**문서 탐색:**

```groovy
site:target.com filetype:pdf
site:target.com filetype:xlsx
site:target.com filetype:docx "confidential"
site:target.com filetype:csv intext:email
site:target.com filetype:pdf "internal use only"
```

**로그인 페이지 발견:**

```groovy
site:target.com inurl:admin
site:target.com inurl:login
site:target.com inurl:portal
site:target.com intitle:"login" | intitle:"sign in"
site:target.com inurl:wp-login.php
site:target.com inurl:webmail
site:target.com inurl:owa    (→ Outlook Web Access)
```

**민감한 디렉토리:**

```groovy
site:target.com intitle:"index of"
site:target.com intitle:"index of /" +passwd
site:target.com inurl:wp-admin
site:target.com inurl:phpmyadmin
```

**설정 파일 노출:**

```visual-basic
site:target.com ext:xml | ext:conf | ext:cnf | ext:env
site:target.com filetype:env "DB_PASSWORD"
site:target.com filetype:env "API_KEY" | "MAIL_PASSWORD"
site:target.com filetype:ini
```

**데이터베이스 파일:**

```sql
site:target.com ext:sql | ext:db | ext:mdb
site:target.com filetype:sql "INSERT INTO"
site:target.com ext:bak | ext:old | ext:backup
```

**비밀번호/자격증명:**

```visual-basic
site:target.com intext:password filetype:log
site:target.com intext:"username" intext:"password" filetype:txt
site:target.com "password" ext:txt | ext:log | ext:cfg
```

**.git / .env 파일:**

```groovy
site:target.com inurl:".git"
site:target.com intitle:"index of" ".git"
site:target.com intitle:"index of" ".env"
```

**정보 노출:**

```groovy
site:target.com intitle:"phpinfo()"
site:target.com inurl:"server-status"
site:target.com "SQL syntax" | "mysql_fetch"
```

※Google Hacking Database ([https://www.exploit-db.com/google-hacking-database)](https://www.exploit-db.com/google-hacking-database)) 에서 카테고리별 수천 개의 추가 도크를 확인할 수 있다

### [5단계 : 소셜 미디어 OSINT]

소셜 미디어 OSINT 단계에선 직원 이름, 직책, 조직 구조, 기술 스택을 파악하여 유저명 목록을 생성하는 것이 목표이다

### [5-1 : LinkedIn — linkedin2username / CrossLinked]

LinkedIn은 직원 이름과 직책을 공개적으로 노출하는 SNS이다

이를 이용하면 직원 이름과 소속 회사를 수집할 수 있으며 이메일 패턴과 결합하면 1-1에서 언급한 패턴 기반 워드리스트를 생성할 수 있다.

**사용 조건:**

```less
1.LinkedIn 계정 보유 (linkedin2username)
2.또는 계정 없이 Google/Bing 스크래핑 (CrossLinked)
```

**공격 흐름 + 툴 & 명령어:**

```sql
(공격 흐름)

1.LinkedIn에서 대상 회사 직원 목록 수집
2.이름 패턴으로 유저명/이메일 생성
3.생성된 목록 → Kerbrute 유저명 열거 or 패스워드 스프레이

--------------------------------------------------------------

(linkedin2username)

git clone https://github.com/initstring/linkedin2username
cd linkedin2username && pip3 install -r requirements.txt

# 기본 사용
linkedin2username -c company-name

# 이메일 형식으로 생성
linkedin2username -c company-name -n target.com

# 키워드 필터 (IT, engineering 등)
linkedin2username -c company-name -k "engineering,IT"

# 1000명 제한 우회
linkedin2username -c company-name -g

--------------------------------------------------------------

(CrossLinked - 계정 없이 사용 가능)

pip3 install crosslinked

# 이메일 형식 생성
crosslinked -f '{first}.{last}@target.com' "Target Company"

# Windows 도메인 형식
crosslinked -f 'DOMAIN\{f}{last}' "Target Company"
```

### [5-2 : Twitter/X 고급 검색]

Twitter 고급 검색으로 대상 조직 관련 기술 정보, 직원 발언, 내부 정보를 수집할 수 있다

**주요 검색 연산자:**

```makefile
from:username        → 특정 사용자의 트윗
to:username          → 특정 사용자에게 보낸 트윗
since:YYYY-MM-DD     → 날짜 이후
until:YYYY-MM-DD     → 날짜 이전
near:"city"          → 위치 근처
filter:images        → 이미지 포함 트윗
geocode:lat,long,radius → 지리적 범위 내

# 고급 검색 페이지
https://twitter.com/search-advanced
```

### [5-3 : Sherlock — 사용자명 크로스 플랫폼 검색]

Sherlock는 입력한 사용자명을 400개 이상의 소셜 미디어 플랫폼에서 동시에 검색하는 도구이다

**사용 조건:**

```
1.Python 환경
2.대상 인물의 사용자명 보유
```

**공격 흐름 + 명령어:**

```
(공격 흐름)

1.LinkedIn/이메일 앞부분 등에서 사용자명 추측
2.Sherlock로 400+ 플랫폼 동시 검색
3.사용 중인 플랫폼에서 추가 정보 수집 (사진, 직책, 위치)
4.수집한 정보로 소셜 엔지니어링 공격 기반 구축

--------------------------------------------------------------

(설치 + 명령어)

pipx install sherlock-project

# 단일 검색
sherlock username

# 다중 검색
sherlock user1 user2 user3

# 파일 저장
sherlock --output results.txt username

# 특정 사이트만
sherlock --site github username

# CSV 출력
sherlock --csv username

# Tor 사용
sherlock --tor username
```

### [6단계 : DNS 정찰]

DNS 정찰 단계에선 대상 조직의 네임서버, 메일 서버, IP 주소, 네트워크 구조를 파악하는것이 목표이다

### [6-1 : nslookup]

nslookup은 Windows/Linux 기본 탑재 DNS 조회 도구이다

IP주소나 메일 서버 주소, 네임 서버 주소 등 다양한 서버 주소들을 레코드 단위로 조회 가능하다

※DNS 조회 → DNS의 정의 자체는 "도메인 이름을 IP 주소로 변환해주는 시스템" 이다 하지만 인터넷이 동작하려면 반드시 필요한 정보들이 있기 때문에 DNS를 조회 할 경우 IP 주소, 메일 서버, 네임서버, 서브도메인 등 조직의 네트워크 구조가 노출된다

```bash
nslookup target.com                   # 기본 A 레코드
nslookup -type=MX target.com         # 메일 서버
nslookup -type=NS target.com         # 네임서버
nslookup -type=TXT target.com        # TXT (SPF, DKIM 등)
nslookup -type=AAAA target.com       # IPv6
nslookup -type=SOA target.com        # SOA 레코드
nslookup -type=CNAME target.com      # 별칭
nslookup target.com 8.8.8.8         # 특정 DNS 서버 사용

# 대화형 모드
nslookup
> server 8.8.8.8
> set type=MX
> target.com
```

### [6-2 : dig]

dig는 보안 전문가들이 가장 많이 사용하는 DNS 조회 도구이다

nslookup보다 출력이 상세하고 스크립팅에 유리하다

```csharp
dig target.com                             # 기본 조회
dig target.com +short                      # 간결한 출력
dig target.com MX +short                   # MX 레코드
dig target.com NS                          # 네임서버
dig target.com TXT                         # TXT 레코드
dig target.com ANY                         # 모든 레코드
dig @8.8.8.8 target.com                   # 특정 DNS 서버
dig -x 8.8.8.8                            # 역방향 DNS
dig +trace target.com                      # 전체 해석 경로 추적
dig +noall +answer target.com             # 응답 섹션만
dig -f domains.txt +short                  # 배치 모드

# 영역 전송 시도 (핵심 기술!)
dig +short ns target.com                   # Step 1: 네임서버 확인
dig axfr @ns1.target.com target.com       # Step 2: AXFR 시도

# 연습 사이트 (실제로 영역 전송 가능)
dig axfr zonetransfer.me @nsztm1.digi.ninja.
```

※영역 전송(AXFR)은 모든 DNS 레코드를 한 번에 노출하는 중대한 정보 유출이다. 대부분 거부하지만 허용되면 모든 중요한 정보들이 노출됨

### [6-3 : dnsrecon]

dnsrecon는 DNS 열거를 자동화하는 도구로 Kali Linux에 기본 탑재되어 있다

브루트포스, 영역 전송, SRV 레코드, DNSSEC 영역 워크까지 한번에 가능하다

※ SRV 레코드 = 서비스 관련 레코드 조회

※ DNSSEC 영역 워크 = DNSSEC 허점으로 서브도메인 열거

**사용 조건:**

```less
1.Kali Linux (기본 탑재) 또는 pip install dnsrecon
```

**명령어:**

```
# 기본 표준 열거 (SOA, NS, A, MX + AXFR 시도)
dnsrecon -d target.com -t std

# 영역 전송 테스트
dnsrecon -d target.com -t axfr

# 브루트포스 (wordlist 기반)
dnsrecon -d target.com -t brt -D /usr/share/wordlists/dnsmap.txt

# 역방향 조회 (IP 대역)
dnsrecon -r 192.168.1.0/24 -t rvl

# DNSSEC 영역 워크
dnsrecon -d target.com -t zonewalk

# SRV 레코드
dnsrecon -d target.com -t srv

# 출력 저장
dnsrecon -d target.com -t std --xml output.xml
dnsrecon -d target.com -t std -j output.json
dnsrecon -d target.com -t std -c output.csv

# 열거 유형(-t) 전체 목록
# std        : 표준 (SOA, NS, A, AAAA, MX, SRV, 취약한 서버 AXFR)
# axfr       : 영역 전송 테스트
# brt        : 브루트포스
# rvl        : 역방향 조회
# srv        : SRV 레코드
# crt        : crt.sh 활용
# zonewalk   : DNSSEC 영역 워크
```

### [6-4 : WHOIS]

WHOIS는 도메인 등록 정보를 조회하여 등록자 이름, 조직, 이메일, 네임서버를 파악해준다

```
whois target.com          # 도메인 WHOIS
whois 8.8.8.8            # IP WHOIS

# 과거 WHOIS 조회 (프라이버시 적용 전 데이터)
# whoxy.com
# DomainTools (domaintools.com)
```

※GDPR(개인정보 보호법) 이후 많은 정보가 "Redacted for Privacy"로 숨겨지지만 과거 WHOIS 조회를 통해 프라이버시 적용 전 데이터를 찾을 수 있다

### [7단계 : 웹사이트 기술 식별 & 문서 메타데이터 분석]

7단계에선 대상 웹사이트의 기술 스택과 공개 문서에 숨겨진 내부 정보를 수집하는것이 목표이다

※기술 스택 = 웹사이트를 만들 때 사용한 언어나 프레임워크 등등 = 취약점 탐색 가능

### [7-1 : BuiltWith & Wappalyzer]

BuiltWith와 Wappalyzer는 대상 웹사이트가 사용하는 기술 스택을 식별하는 도구이다

※앞서 말한 것 처럼 기술 스택 파악은 해당 CMS/프레임워크의 알려진 취약점 탐색으로 이어질 수 있다

```less
# BuiltWith
https://builtwith.com → 도메인 입력
→ CMS, 프레임워크, CDN, 분석 도구, JS 라이브러리, 이메일 서비스 등 2500+ 카테고리

# Wappalyzer
Chrome/Firefox 확장 프로그램으로 설치
→ 방문 중인 사이트 기술 스택 즉시 표시 (1000+ 기술 감지)
→ CMS(WordPress, Drupal), JS 프레임워크(React, Angular), 웹 서버(Apache, Nginx)
```

### [7-2 : WhatWeb]

WhatWeb은 Kali Linux에 사전 설치된 CLI 기반 웹 핑거프린터(웹사이트의 기술 스택을 식별하는 것)이다

900+ 플러그인으로 CMS, 웹 서버, 프레임워크, 에러 페이지, 쿠키 등을 자동 탐지한다

**명령어:**

```perl
whatweb target.com                          # 기본 (스텔스, 레벨 1)
whatweb -a 3 target.com                    # 공격적 (추가 요청)
whatweb -a 4 target.com                    # 헤비 (모든 플러그인)
whatweb -v target.com                      # 상세 출력
whatweb -i targets.txt                     # 다중 대상 (파일)
whatweb --log-json=output.json target.com  # JSON 출력
whatweb --log-xml=output.xml target.com    # XML 출력
whatweb -t 50 target.com                   # 50 스레드

# 공격 레벨(-a)
# 1 : 스텔스 (HTTP 요청 1회)
# 3 : 공격적 (추가 요청 발생)
# 4 : 헤비 (모든 플러그인 URL 시도)
```

### [7-3 : ExifTool — 문서 메타데이터 추출]

공개 문서에서 작성자 이름(=사용자명), 소프트웨어 버전, 내부 파일 경로, GPS 좌표를 추출한다

저자 이름은 도메인 유저명으로 활용 가능하고 내부 경로는 서버 구조 파악에 유용하게 사용 가능하다

**명령어:**

```coffeescript
# 기본 메타데이터 추출
exiftool document.pdf

# 모든 메타데이터 (중복, 미지 태그 포함, 그룹별)
exiftool -a -u -g1 file.pdf

# 특정 필드만
exiftool -Author -Creator -Producer -CreateDate file.pdf

# 배치 처리 (디렉토리 전체)
exiftool -r /path/to/directory/
exiftool -r -ext pdf /path/to/directory/

# GPS 추출 (이미지)
exiftool -GPSLatitude -GPSLongitude image.jpg

# CSV 출력
exiftool -csv -r *.pdf > output.csv

# 메타데이터 제거 (내 파일 OPSEC)
exiftool -all= file.pdf
```

※Author 필드의 사용자명은 비밀번호 스프레이 대상으로 사용 가능하다

※내부 경로(예: \\SERVER01\shared\) 발견 시 서버 구조 파악에 활용한다

### [7-4 : FOCA — 자동화 문서 메타데이터 수집]

FOCA(Fingerprinting Organizations with Collected Archives)는 Windows 도구로 대상 웹사이트의 모든 문서를 자동으로 찾고 메타데이터를 추출한다

**사용 조건:**

```less
1.Windows 환경
2.FOCA 설치 (https://github.com/ElevenPaths/FOCA)
```

**공격 흐름:**

```sql
(공격 흐름)

1.프로젝트 생성 → 도메인 입력
2.문서 유형 선택 (PDF, DOC, XLS, PPT 등)
3.Search All → Google/Bing에서 자동으로 문서 탐색
4.Download All → 발견된 문서 자동 다운로드
5.Extract All Metadata → 전체 메타데이터 자동 추출
6.카테고리별 결과 확인:
   Users      → 사용자명 목록
   Software   → 소프트웨어 버전
   Paths      → 내부 경로
   OS         → 운영체제
   Emails     → 이메일 주소
```
