## Slot Machine (453 points)

### Description

We have onboard entertainment! Try your luck on our newly installed slot machine.

```sh
ncat --ssl slot-machine.chal.uiuc.tf 1337
```

### Source

```python
from hashlib import sha256  
  
hex_alpha = "0123456789abcdef"  
  
print("== Welcome to the onboard slot machine! ==")  
print("If all the slots match, you win!")  
print("We believe in skill over luck, so you can choose the number of slots.")  
print("We'll even let you pick your lucky number (little endian)!")  
  
lucky_number = input("What's your lucky (hex) number: ").lower().strip()  
lucky_number = lucky_number.rjust(len(lucky_number) + len(lucky_number) % 2, "0")  
if not all(c in hex_alpha for c in lucky_number):  
    print("Hey! That's not a hex number! -999 luck!")  
    exit(1)  
hash = sha256(bytes.fromhex(lucky_number)[::-1]).digest()[::-1].hex()  
  
length = min(32, int(input("How lucky are you feeling today? Enter the number of slots: ")))  
print("=" * (length * 4 + 1))  
print("|", end="")  
for c in hash[:length]:  
    print(f" {hex_alpha[hex_alpha.index(c) - 1]} |", end="")  
print("\n|", end="")  
for c in hash[:length]:  
    print(f" {c} |", end="")  
print("\n|", end="")  
for c in hash[:length]:  
    print(f" {hex_alpha[hex_alpha.index(c) - 15]} |", end="")  
print("\n" + "=" * (length * 4 + 1))  
  
if len(set(hash[:length])) == 1:  
    print("Congratulations! You've won:")  
    flag = open("flag.txt").read()  
    print(flag[:min(len(flag), length)])  
else:  
    print("Better luck next time!")
```

### Challenge analysis

This *looks* pretty simple, let's analyze the code:

1. We input `lucky_number` string which must be a valid hex number.
2. We input the `slots` number.
3. The `lucky_number` gets hashed in the following way:
  - It gets padded with `0` if it's of odd length.
  - It gets reversed.
  - `sha256` is calculated from it.
  - It gets reversed again.

If the hash has X (X = `slots`) repeating characters in the beginning, we get the flag.

In short, to get the flag we must find the number in hex format whose `sha256` hash results in a string ending (because it gets reversed in the end) with X repeating characters (where X is the flag length or a higher value), and then reverse it.

### Finding the solution

#### Do I actually understand Python?

As I'm not super confident with Python (I am not a Python dev on daily basis), I spent some time trying to understand if it's possible to reveal the flag partially using some funky `slots` values like negative numbers, etc., but did not succeed.

#### Bruteforce attempt

Being out of ideas, I also tried to find such a number myself by simply bruteforcing it but managed to only find a number which after hashing gave me a maximum of 7 repeating characters in the beginning of a string. Definitely not enough for a flag.

#### Bitcoin to the rescue?!

Then, I started searching the web to know more about `sha256` nature and to see if it's possible to find such hashes (with repeating characters in the beginning) in some existing lists. After a few hours (and a few coffee breaks), I finally found something that shed more light on this problem: a question and an answer on Bitcoin Stack Exchange: [Is it always possible to find a number whose hash starts with a certain number of..](https://bitcoin.stackexchange.com/questions/121920/is-it-always-possible-to-find-a-number-whose-hash-starts-with-a-certain-number-o) and specifically this comment:

> "Due to these characteristics, it is statistically almost certain that a valid input exists which will generate a hash with a specific number of leading zeros. It is just a matter of trying enough inputs (or "nonces" in the case of ₿ mining) until you find one. This is what ₿ miners do in the proof-of-work system - they try billions of different inputs every second until they find one that generates a hash with the required number of leading zeros."

Looks like finding the solution to this challenge is what Bitcoin miners actually do! Well, they actually try to find hex strings whose `sha256` hash (as a number) is lower value than X (this value depends on the difficulty of the blockchain), but that results in numbers with a lot of leading 0s in the beginning. I was not very familiar with Bitcoin Proof of Work algorithm, so I wasn't aware of that 🤯

### Bitcoin PoW and lowest hash

Let's find the lowest Bitcoin block hash ever mined and try to recreate the algorithm and find a solution.

I've found this answer about the lowest hash on the same Stack Exchange: https://bitcoin.stackexchange.com/questions/65478/which-is-the-smallest-hash-that-has-ever-been-hashed. Looks like block `756951` had the lowest (as of Jan 22, 2023) hash value with 24 leading zeros. I hope it will do!

I've also found a nonce verifying algorithm written in Python here: https://stackoverflow.com/questions/66944273/bitcoin-verify-a-single-block-in-python and modified it a little for my needs, so it outputs my "lucky number":

```python
header = (struct.pack("<L", lowest_bitcoin_block_hash["version"]) +  
          bytes.fromhex(lowest_bitcoin_block_hash["previousblockhash"])[::-1] +  
          bytes.fromhex(lowest_bitcoin_block_hash["merkle_root"])[::-1] +  
          struct.pack("<LLL", lowest_bitcoin_block_hash["timestamp"],  
                      lowest_bitcoin_block_hash["bits"],  
                      lowest_bitcoin_block_hash["nonce"]))  
lucky_number = hashlib.sha256(header).digest()[::-1].hex()  
print("lucky number: " + lucky_number)  
leading_zeros = str(len(re.match(r"^0*", lowest_bitcoin_block_hash['id']).group()))  
print("slots: " + leading_zeros)
```

I've fetched the values for `lowest_bitcoin_block_hash` from Blockstream Explorer API: https://blockstream.info/api/block/0000000000000000000000005d6f06154c8685146aa7bc3dc9843876c9cefd0f

```json
{  
    "id": "0000000000000000000000005d6f06154c8685146aa7bc3dc9843876c9cefd0f",  
    "height": 756951,  
    "version": 541065216,  
    "timestamp": 1664846893,  
    "tx_count": 2905,  
    "size": 1464226,  
    "weight": 3993487,  
    "merkle_root": "62c46f1efadf6e39b7463e5362bb552cba98f74a80a58378ff5194c7b058005a",  
    "previousblockhash": "000000000000000000050da0da9451c2e1306db4ddb5acc965fc1016678d9154",  
    "mediantime": 1664843725,  
    "nonce": 3240300428,  
    "bits": 386464174,  
    "difficulty": 31360548173144.85  
}
```

And got the flag!

```python
# ncat --ssl slot-machine.chal.uiuc.tf 1337  
r = pwn.remote('slot-machine.chal.uiuc.tf', 1337, ssl=True)  
r.sendline(lucky_number.encode('utf-8'))  
r.sendline(leading_zeros.encode('utf-8'))  
response = r.recvall().decode()  
r.close()  
flag = re.search(r"Congratulations! You've won:\n(.+)", response).group(1)  
print(flag)
```

```plaintext
lucky number: 93ccd10e30712e566e0bc0189c791e609b11fc17190b00eb50d6fa8b4909b2f5
slots: 24
uiuctf{keep_going!_3cyd}
```

### Credits

- Jake (*could not find him in the UIUCT Dev Team, so no link :(*), for creating this challenge.
- deyw, Bitcoin Stack Exchange user (https://bitcoin.stackexchange.com/users/87158/deyw), for the answer leading me to the solution https://bitcoin.stackexchange.com/a/121922.
- snapo, Stack Overflow user (https://stackoverflow.com/users/4520017/snapo), for an answer with a simplified version of Python Bitcoin PoW code https://stackoverflow.com/a/67140338.
- razvancazacu, GitHub user (https://github.com/razvancazacu), for the original Python Bitcoin mining code https://github.com/razvancazacu/bitcoin-mining-crypto.
