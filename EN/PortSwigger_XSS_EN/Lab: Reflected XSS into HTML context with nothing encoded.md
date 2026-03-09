### [Problem]

This lab contains a simple reflected cross-site scripting vulnerability in the search functionality.

To solve the lab, perform a cross-site scripting attack that calls the alert function.

### [Walkthrough]

The goal is to trigger an alert by exploiting a Reflected XSS vulnerability in the search feature.

Just enter the following into the search box:

```bash
<script>alert(1)</script>
```

![image.png](attachment:6d2b321f-6297-48d4-958c-efe2ebe801e8:image.png)

When you type the payload into the search bar, the server inserts it directly into the HTML without any encoding, and the browser executes it as JavaScript.

![image.png](attachment:9c3a2484-5605-4717-b806-25ae481bada8:image.png)

![image.png](attachment:fbcb2142-77b3-4a45-b9a2-dad6b6375a5c:image.png)

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
