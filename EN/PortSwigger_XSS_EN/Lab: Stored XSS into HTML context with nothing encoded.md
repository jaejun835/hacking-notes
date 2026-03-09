# Lab: Stored XSS into HTML context with nothing encoded

### [Problem]

This lab contains a simple stored cross-site scripting vulnerability in the comment functionality.

To solve the lab, submit a comment that calls the alert function when the blog post is viewed.

### [Walkthrough]

The goal is to trigger an alert by exploiting a Stored XSS vulnerability in the comment section.

Just enter the following into the comment box:

```
<script>alert(1)</script>
```

![1](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/PortSwigger_XSS_KR/Lab%3A%20Stored%20XSS%20into%20HTML%20context%20with%20nothing%20encoded/1.Lab%3A%20Stored%20XSS%20into%20HTML%20context%20with%20nothing%20encoded.png)

When the payload is submitted, the server stores it in the database without encoding. The next time anyone views the page, the browser renders and executes it as JavaScript.

![2](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/PortSwigger_XSS_KR/Lab%3A%20Stored%20XSS%20into%20HTML%20context%20with%20nothing%20encoded/2.Lab%3A%20Stored%20XSS%20into%20HTML%20context%20with%20nothing%20encoded.png)

![3](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/PortSwigger_XSS_KR/Lab%3A%20Stored%20XSS%20into%20HTML%20context%20with%20nothing%20encoded/3.Lab%3A%20Stored%20XSS%20into%20HTML%20context%20with%20nothing%20encoded.png)

### [Vulnerability Explanation]

The server stores user input without encoding and injects it directly into HTML. The browser treats the stored value as executable code rather than plain text, running the JavaScript.

### [Mitigation]

Encode user input before rendering it in HTML, or use a Content Security Policy (CSP).

Encoding example (special characters are converted when outputting to HTML):

```
<script>alert(1)</script>

<  =  &lt;
>  =  &gt;
"  =  &quot;
'  =  &#x27;

&lt;script&gt;alert(1)&lt;/script&gt;
```

※ CSP: The server tells the browser in advance which scripts are allowed to run on the site, blocking execution of any externally injected scripts.
