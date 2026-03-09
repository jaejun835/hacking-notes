# Lab: Reflected XSS into HTML context with nothing encoded

### [문제]

This lab contains a simple reflected cross-site scripting vulnerability in the search functionality.

To solve the lab, perform a cross-site scripting attack that calls the alert function.

### [풀이과정]

이 랩은 검색 기능의 Reflected XSS 취약점을 이용하여 alert 함수를 실행하는 것이 목표이다

이 문제는 간단하게 아래의 명령어만 검색 창에 입력해 주면 끝이다

```bash
<script>alert(1)</script>
```

![1](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/PortSwigger_XSS_KR/Lab%3A%20Reflected%20XSS%20into%20HTML%20context%20with%20nothing%20encoded/1.Lab%3A%20Reflected%20XSS%20into%20HTML%20context%20with%20nothing%20encoded.png)

검색창에 명령어를 입력하면 서버가 입력값을 인코딩 없이 HTML에 그대로 삽입하고 브라우저가 이를 JavaScript로 실행한다

![2](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/PortSwigger_XSS_KR/Lab%3A%20Reflected%20XSS%20into%20HTML%20context%20with%20nothing%20encoded/2.Lab%3A%20Reflected%20XSS%20into%20HTML%20context%20with%20nothing%20encoded.png)

![3](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/PortSwigger_XSS_KR/Lab%3A%20Reflected%20XSS%20into%20HTML%20context%20with%20nothing%20encoded/3.Lab%3A%20Reflected%20XSS%20into%20HTML%20context%20with%20nothing%20encoded.png)

### [취약점 설명]

서버가 사용자의 입력값을 인코딩 처리없이 HTML에 그대로 삽입하기 때문에 브라우져가 입력값(<script>alert(1)</script>)을 데이터가 아닌 코드로 인식하여 JavaScript를 실행한다

### [대응 방안]

입력값이 코드로 해석되지 않도록 인코딩을 하거나 CSP를 이용한다

인코딩 예시(HTML에 출력할 때 특수문자를 변환한다):

```bash
<script>alert(1)</script>

<  =  &lt;
>  =  &gt;
"  =  &quot;
'  =  &#x27;

&lt;script&gt;alert(1)&lt;/script&gt;
```

※ CSP: 서버가 브라우져에게 사이트에서 실행 가능한 스크립트를 미리 지정하여 외부에서 주입된 스크립트 실행을 막는다
