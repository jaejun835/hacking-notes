# Lab: Reflected XSS into HTML context with nothing encoded

### [Problem]

This lab contains a simple reflected cross-site scripting vulnerability in the search functionality.

To solve the lab, perform a cross-site scripting attack that calls the alert function.

### [Walkthrough]

The goal is to trigger an alert by exploiting a Reflected XSS vulnerability in the search feature.

Just enter the following into the search box:

```bash
<script>alert(1)</script>
```

![1](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/PortSwigger_XSS_KR/Lab%3A%20Reflected%20XSS%20into%20HTML%20context%20with%20nothing%20encoded/1.Lab%3A%20Reflected%20XSS%20into%20HTML%20context%20with%20nothing%20encoded.png)

When you type the payload into the search bar, the server inserts it directly into the HTML without any encoding, and the browser executes it as JavaScript.

![2](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/PortSwigger_XSS_KR/Lab%3A%20Reflected%20XSS%20into%20HTML%20context%20with%20nothing%20encoded/2.Lab%3A%20Reflected%20XSS%20into%20HTML%20context%20with%20nothing%20encoded.png)

![3](https://raw.githubusercontent.com/jaejun835/hacking-notes/main/Photo/PortSwigger_XSS_KR/Lab%3A%20Reflected%20XSS%20into%20HTML%20context%20with%20nothing%20encoded/3.Lab%3A%20Reflected%20XSS%20into%20HTML%20context%20with%20nothing%20encoded.png)

### [Vulnerability Explanation]

The server inserts user input directly into the HTML response with no encoding. This causes the browser to treat `<script>alert(1)</script>` as executable code rather than data.

### [Mitigation]

Encode user input before rendering it in HTML, or use a Content Security Policy (CSP).

Encoding example (special characters are converted when outputting to HTML):

```bash
<script>alert(1)</script>

<  =  &lt;
>  =  &gt;
"  =  &quot;
'  =  &#x27;

&lt;script&gt;alert(1)&lt;/script&gt;
```

※ CSP: The server tells the browser in advance which scripts are allowed to run on the site, blocking execution of any externally injected scripts.
