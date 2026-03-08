### 1단계 (개별 머신, 쉬움)

| Blue | EternalBlue, Meterpreter | tryhackme.com/room/blue |
| --- | --- | --- |
| Ice | Windows 서비스 익스플로잇 | tryhackme.com/room/ice |
| Kenobi | SMB, PrivEsc | tryhackme.com/room/kenobi |
| Basic Pentesting | 열거 → PrivEsc 전체 흐름 | tryhackme.com/room/basicpentestingjt |
| Lazy Admin | Linux 초기 침투 + PrivEsc | tryhackme.com/room/lazyadmin |

### 2단계 (중급 개별 머신 (Boot2Root))

| Alfred | Windows, Jenkins RCE, 토큰 탈취 | tryhackme.com/room/alfred |
| --- | --- | --- |
| HackPark | Windows, Hydra, PrivEsc | tryhackme.com/room/hackpark |
| GameZone | SQLi, SSH 터널링 | tryhackme.com/room/gamezone |
| Skynet | SMB, RFI, Cronjob PrivEsc | tryhackme.com/room/skynet |
| Anonymous | FTP, Linux PrivEsc | tryhackme.com/room/anonymous |
| Mr. Robot CTF | 웹 열거, Linux PrivEsc | tryhackme.com/room/mrrobot |
| Relevant | SMB, Windows PrivEsc | tryhackme.com/room/relevant |
| Overpass | Linux 열거, PrivEsc | tryhackme.com/room/overpass |
| Overpass 2 | 포렌식 + 역방향 분석 | tryhackme.com/room/overpass2hacked |
| Overpass 3 | NFS 악용, PrivEsc | tryhackme.com/room/overpass3hosting |
| Startup | Linux, 이상한 서비스 분석 | tryhackme.com/room/startup |
| Dogcat | Docker 탈출, PHP LFI | tryhackme.com/room/dogcat |

### 3단계 (PrivEsc 아레나 (CTF 스타일))

| Linux PrivEsc Arena | SUID/cron/passwd 등 실전 | tryhackme.com/room/linuxprivescarena |
| --- | --- | --- |
| Windows PrivEsc Arena | 서비스/레지스트리/DLL 등 실전 | tryhackme.com/room/windowsprivescarena |
| Steel Mountain | Tib3rius 제작, Windows PrivEsc | tryhackme.com/room/steelmountain |

### 4단계 (AD + 네트워크 실전)

| Attacktive Directory | Kerberoasting, ASREPRoast, DC 장악 | 단독 DC | tryhackme.com/room/attacktivedirectory |
| --- | --- | --- | --- |
| Throwback | AD 네트워크, 피벗팅, DC 장악 | 멀티 머신 | tryhackme.com/room/throwback |
| **Wreath** | 외부→피벗→내부 DC 장악, PNPT 최적 | 3대 네트워크 | tryhackme.com/room/wreath |
| **Holo** | AV 우회 포함, 대형 AD 네트워크 | 대형 네트워크 | tryhackme.com/room/hololive |

### 권장 순서

```less
[1단계] Blue → Kenobi → Basic Pentesting
         ↓
[2단계] Alfred → HackPark → Skynet → Relevant → Mr. Robot
         ↓
[3단계] Linux PrivEsc Arena → Windows PrivEsc Arena → Steel Mountain
         ↓
[4단계] Attacktive Directory  ← 여기서 보고서 1번 써보기
         ↓
        Throwback
         ↓
        Wreath              ← 여기서 보고서 2번 써보기
         ↓
        Holo (여유 있으면)
```

1단계~3단계는 각 머신 끝날 때마다 **스크린샷 + 메모 습관** 들이는 용도로 활용하고 4단계부터 보고서 작성
