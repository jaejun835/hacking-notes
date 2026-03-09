
tryhackme 링크 → https://tryhackme.com/room/attacktivedirectory

오늘은 TryHackMe의 Attacktive Directory 룸을 풀어보았다

### [What tool will allow us to enumerate port 139/445?]

139는 보통 NetBIOS, 445는 보통 SMB 서비스를 사용하는데 쓴다

여기서 NetBIOS는 쉽게 말해 SMB 서비스의 과거 버젼이라 생각하면 된다

이 두가지 포트를 열거하기 위해선 enum4linux를 사용하면 된다

```
enum4linux -a <IP>
```

정답: enum4linux

### [What is the NetBIOS-Domain Name of the machine?]

enum4linux 스캔 결과 도메인 이름이 THM-AD 인 것을 알 수 있었다

여기서 NetBIOS-Domain는 Active Directory의 구조를 파악할 때 매우 큰 역할을 한다

왜냐하면 아래의 결과로 도메인 컨트롤러(최상)를 찾을 수 있기 때문이다

![AD-1.png](attachment:b05fa338-e405-43fa-ac8b-29e89588f404:AD-1.png)

정답: THM-AD

### [What invalid TLD do people commonly use for their Active Directory Domain?]

도메인은 쉽게 말해 한 회사/조직에 속한 모든 컴퓨터와 사람들을 묶어서 관리하는 시스템이다

흔히 사람들이 Active Directory 도메인에 사용하는 유효하지 않은 TLD는 .local이다

정답: .local

### [What command within Kerbrute will allow us to enumerate valid usernames?]

사용자 이름을 열거하기 위해선 kerbrute에 userenum 옵션을 추가해 주면 된다

```
kerbrute userenum --dc <IP> -d spookysec.local user.txt
```

정답: userenum

### [What notable account is discovered? (These should jump out at you)]

kerbrute를 통해 총 16개의 계정을 찾았다

하지만 AD 자체가 대소문자 비구분 시스템이기 때문에 출력 결과를 자세히 보게 되면 중복된 이메일들을 볼 수 있다

결과적으로는 계정 자체는 8개가 유효하며 그 중 svc-admin을 활용하면 다음 단계를 진행 할 수 있다

![AD-2.png](attachment:9b70d275-36ce-4e45-8e49-dca529f0579b:AD-2.png)

### [What is the other notable account is discovered? (These should jump out at you)]

kerbrute로 찾은 계정들 중 backup 계정 또한 유용하게 사용 가능하다

(백업 작업 특성상 스크립트나 스케줄러로 자동 실행되어야 해서 패스워드가 어딘가에 평문으로 저장되거나 Kerberos 사전인증이 꺼져있거나 아니면 DCSync 권한 같은 강력한 권한을 가지는 경우가 많기 때문..)

### [We have two user accounts that we could potentially query a ticket from. Which user account can you query a ticket from with no password?]

이 질문은 AS-REP Roasting 공격 기법을 의미한다

AS-REP Roasting은 쉽게 말해 비밀번호 없이 티켓을 받아서 오프라인으로 크래킹하는 공격이 생각하면 된다

```
impacket-GetNPUsers spookysec.local/ -usersfile user.txt -dc-ip <IP> -format hashcat -request
```

![AD-3.png](attachment:5db773d5-8a1a-4b01-b5d9-ab6a714fbffa:AD-3.png)

로스팅 결과 svc-admin 가 취약한 계정이였다는 것을 알 수 있다

정답: svc-admin

### [What mode is the hash?]

로스팅 결과, 해쉬를 자세히 보면 $krb5asrep$23$이기 때문에 18200 타입일거다

정답: 18200

### [Now crack the hash with the modified password list provided, what is the user accounts password?]

해쉬캣을 사용하여 크랙해주면 바로 자격증명을 얻을 수 있다

```
hashcat -m 18200 ./hash.txt ./password.txt
```

![AD-4.png](attachment:a029d9c9-ad07-419a-bd3b-fab59302a428:AD-4.png)

정답: management2005

### [What utility can we use to map remote SMB shares?]

원격 SMB 공유에 매핑할려면 간단하게 smbclient를 써주면 된다

정답: smbclient

### [Which option will list shares?]

List의 약자인 -L 옵션을 사용하면 공유폴더 리스트를 볼 수 있다

정답: -L

### [How many remote shares is the server listing?]

smbclient를 사용하여 공유 리스트를 열거 해보면 총 6개의 리스트를 확인 할 수 있다

```
smbclient -L //10.48.185.32 -U svc-admin
```

![AD-5.png](attachment:cbc05475-cbe7-4d77-83be-95a64a905e11:AD-5.png)

### [There is one particular share that we have access to that contains a text file. Which share is it?]

backup파일에 text 파일이 있는 것을 확인 할 수 있다

![AD-6.png](attachment:d77bca47-0f97-4d5c-9aac-ce2f70d1fc11:AD-6.png)

정답: backup

### [What is the content of the file?]

backup 결과에서 확인한 것처럼 파일의 내용은 YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw이다

정답: YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw

### [Decoding the contents of the file, what is the full contents?]

backup 코드의 해쉬는 base64 타입으로 디코딩 하면 된다

```
cat hash_1.txt | base64 -d
```

![AD-7.png](attachment:ed821718-d57f-42d9-b79c-ba62a6852ae3:AD-7.png)

정답: backup@spookysec.local:backup2517860

### [What method allowed us to dump NTDS.DIT?]

여기서 NTDS.DIT 는 쉽게 말해 도메인 컨트롤러의 핵심 데이터베이스 파일이다

DCSync는 도메인 컨트롤러끼리 데이터를 동기화할 때 쓰는 정상적인 기능이지만 backup 계정이 DCSync 권한을 가지고 있어서 DC인 척 하고 모든 해시를 요청할 수 있게 된다

![AD-8.png](attachment:5e5c433b-c9bd-45b4-87b0-8dec35358dea:AD-8.png)

![AD-9.png](attachment:42733fce-db23-4ace-8200-e73918b8927c:AD-9.png)

※번외로 krbtgt를 이용해 golden ticket 공격도 가능

출력 결과 NTLM 해시 덤프가 성공한 것을 확인 할 수 있으며 ([*] Using the DRSUAPI method to get NTDS.DIT secrets) DRSUAPI 방법을 사용한 것을 알 수 있다

정답: DRSUAPI

### [What is the Administrators NTLM hash?]

Administrator 해쉬중 NTLM 해쉬는 0e0363213e37b94221497260b0bcb4fc이다

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:0e0363213e37b94221497260b0bcb4fc:::
              │   └─ LM hash                             └─ NTLM hash (중요)
              └─ RID
```

정답: 0e0363213e37b94221497260b0bcb4fc

### [What method of attack could allow us to authenticate as the user without the password?]

패스워드 없이 유저로 인증할 수 있게 해주는 공격 방법은 Pass the Hash 이다

정답: Pass the Hash

### [Using a tool called Evil-WinRM what option will allow us to use a hash?]

Evil-WinRM에서 해시를 사용할 수 있게 해주는 옵션은 -H이다

정답: -H
