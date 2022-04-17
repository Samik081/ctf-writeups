## Foro Romano (884 points)

### Description
I was reading Fasti by Ovid and something about the Forum Romanum set off a lighbulb in my head.

Author : CSN3RD#0923

### source (handout.py)
```python
randseed = '66a48631d401c5e6b5e18'
randpos = 7

dictionarium = ['gravida', 'tristique', 'nunc', 'ornare', 'luctus', 'velit', 'ullamcorper', 'quam', 'mi', 'aliquam', 'ac', 'eleifend', 'porttitor', 'cursus', 'nisl', 'vivamus', 'faucibus', 'nibh', 'blandit', 'venenatis', 'tortor', 'egestas', 'enim', 'orci', 'sit', 'dignissim', 'ipsum', 'urna', 'id', 'semper', 'quisque', 'maecenas', 'in', 'morbi', 'suspendisse', 'posuere', 'nam', 'nec', 'eget', 'sagittis', 'est', 'auctor', 'dictum', 'nullam', 'amet', 'arcu', 'consequat', 'pulvinar', 'ligula', 'lacus', 'justo', 'elementum', 'pharetra', 'viverra', 'neque', 'sed']
hexed_dict = ["".join("{:02x}".format(ord(c)) for c in word) for word in dictionarium]
hexed_dict[randpos] = randseed
int_dict = [int(c, 16) for c in hexed_dict]

passwd = input("Welcome to Foro Romano. You must enter the password to enter: ")
bined_passwd = "".join("{0:b}".format(ord(c)).zfill(7) for c in passwd)

key_str = 's3cr3t_k3y'
hexed_key = "".join("{:02x}".format(ord(c)) for c in key_str)
int_key = int(hexed_key,16)

xor = 0

assert(len(bined_passwd) == len(int_dict))

for i in range(len(bined_passwd)):
	if bined_passwd[i] == '1':
		xor = xor ^ int_dict[i]

if xor == int_key:
	FLAG = 'crew{{{}}}'.format(passwd)
	print("Flag:", FLAG)
else:
	print("Wrong password!")
```

## Solution
### Approach
We are given dictionary of 56 strings and a secret key. Secret key gets converted to it's number representation 
The strings are being XORed based on the user input (password):
1. password (each character) is converted to its binary representation
2. the padding of binary representation of each character is set to 7 (so, for example, 'a' is `0x1100001`, not `0x01100001`) **(IMPORTANT)**
3. the strings are concatenated into a "binary password"
4. `xor = 0`
5. then, for each `n` of the "binary password" bit (0, 1), if it's 1, then `xor = xor ^ dictionary_string[n_index]`
6. if int representation of secret key `int_key == xor`, we get the flag

This means, that we choose which dictionary strings are going to be XORed by changing the **input password**.
Our job is now to :
1. find the correct strings, that XORed together will give the key
2. get the "binary password" - created from the correct dictionary string indexes (`1` if the string should be XORed) 
3. convert obtained "binary password" to the **input password** format:
   1. split the binary string to the groups of 7
   2. (optional, for nice output ^^) add 0 at the left to get all 8 bits
   3. convert each group of binary number to the string (`chr()`)
   4. check if character is printable - we can get multiple candidates, but the password is going to be the flag, so it has to be printable
   5. concatenate character to the password
4. read the password
5. get the flag

### Choosing correct dictionary strings
To find the correct strings to be XORed, we should notice few things:
- dictionary strings differ in length.
- longest string is 11 characters long, which gives us 88 bits (with leading zeros to pad to the whole byte) when presented as binary
- when XORing strings, we actually XOR character by character - byte by byte - bit by bit
- this means, that the very beginning of the XOR result can be "manipulated" only with the longest strings and,
- that each character in the secret key `s3cr3t_k3y` can be affected by the dictionary string only if the string is long enough to cover this character position

Having this knowledge, we can craft our password starting with the longest strings in such fashion:
1. Group dictionary strings by their length and assign the groups to the bytes (`[byte_0 => ['X', 'Y', 'Z'], byte_1 => ['A', 'B', 'C']]`)
2. `n = 0`, 
3. find all permutations of `0` and `1` based on how many dictionary strings correspond to the byte `n`
4. for each permutation (for example `['0', '1', '1]`), set the value of permutation element in the "binary password", at the position corresponding to index of dictionary string
5. calculate XOR for the chosen strings 
6. check if character representation of byte `XOR_result[n]`is the same as `secret_key[n]`
7. if yes, go to `3.` (else return, those values don't provide us the needed character)
8. if `n == 11` (length of the longest string and the final byte), 
9. convert "binary password" to the readable string and **get the flag**

### Implementation in python
```python
import itertools

# from sourcefile -->

randseed = '66a48631d401c5e6b5e18'
randpos = 7

key_str = 's3cr3t_k3y'
hexed_key = "".join("{:02x}".format(ord(c)) for c in key_str)
int_key = int(hexed_key, 16)

dictionarium = ['gravida', 'tristique', 'nunc', 'ornare', 'luctus', 'velit', 'ullamcorper', 'quam', 'mi', 'aliquam',
                'ac', 'eleifend', 'porttitor', 'cursus', 'nisl', 'vivamus', 'faucibus', 'nibh', 'blandit', 'venenatis',
                'tortor', 'egestas', 'enim', 'orci', 'sit', 'dignissim', 'ipsum', 'urna', 'id', 'semper', 'quisque',
                'maecenas', 'in', 'morbi', 'suspendisse', 'posuere', 'nam', 'nec', 'eget', 'sagittis', 'est', 'auctor',
                'dictum', 'nullam', 'amet', 'arcu', 'consequat', 'pulvinar', 'ligula', 'lacus', 'justo', 'elementum',
                'pharetra', 'viverra', 'neque', 'sed']

hexed_dict = ["".join("{:02x}".format(ord(c)) for c in word) for word in dictionarium]
hexed_dict[randpos] = randseed
int_dict = [int(c, 16) for c in hexed_dict]


# extracted this to function
def do_xor(passwd):
    xor = 0
    for k in range(len(passwd)):
        if passwd[k] == '1':
            xor = xor ^ int_dict[k]
    return xor


# <-- from sourcefile


# helper function to convert ints to strings
def number_to_string(number):
    hx = hex(number)[2:]
    if len(hx) % 2:  # padding for appended randseed
        hx = '0' + hx
    return bytearray.fromhex(hx).decode('ascii')


# groups dictionary strings by its lengths
#
# list index is the length of a string
# value contains the list of dictionary indexes
#
# Value for the challenge: [[], [], [8, 10, 28, 32], [24, 36, 37, 40, 55], [2, 14, 17, 22, 23, 27, 38, 44, 45], [5,
# 26, 33, 49, 50, 54], [3, 4, 13, 20, 29, 41, 42, 43, 48], [0, 9, 15, 18, 21, 30, 35, 53], [11, 16, 31, 39, 47, 52],
# [1, 12, 19, 25, 46, 51], [], [6, 7, 34]]
def group_dict_by_length():
    lengths = [[], [], [], [], [], [], [], [], [], [], [], []]
    for idx, string in enumerate(int_dict):
        lengths[len(number_to_string(string))].append(idx)
    return lengths


# traverse through the possible permutations to find candidates for a given byte
def traverse(byte_no, base, dict_indexes_grouped_by_length):
    if byte_no == 11:  # we reached the final byte
        #  because of the zfill(7) while "binning" the password, we split binary by 7 and add missing padding
        output = '0' + " 0".join([base[i:i + 7] for i in range(0, len(base), 7)])

        password = ''
        for char in output.split(' '):
            if not chr(int(char, 2)).isprintable():
                return
            password += chr(int(char, 2))

        print('output with missing paddings: ', output)
        print('binary without paddings: ', base)

        print(f'flag: crew{{{password}}}')
        return

    # get indexes of dictionary strings that might affect given byte
    dict_indexes = dict_indexes_grouped_by_length[len(dict_indexes_grouped_by_length) - byte_no - 1]
    # calculate all the possible permutations of adding those strings to the xor chain
    perms = ["".join(seq) for seq in itertools.product("01", repeat=len(dict_indexes))]
    for perm in perms:
        binary_password = list(base)
        for perm_idx, str_idx in enumerate(dict_indexes):
            binary_password[str_idx] = perm[perm_idx]  # substitute permutation values at given indexes
        binary_password = "".join(binary_password)

        assert (len(binary_password) == len(int_dict))

        xor = do_xor(binary_password)
        xor_byte = number_to_string(xor).rjust(11, '\x00')[byte_no]  # fill to 11 so we can compare bytes
        key_byte = number_to_string(int_key).rjust(11, '\x00')[byte_no]
        if xor_byte == key_byte:  # byte in this xor matches the byte in the key
            traverse(byte_no + 1, binary_password, dict_indexes_grouped_by_length)  # go deeper!
            
    return


lengths = group_dict_by_length()
traverse(0, '0' * 56, lengths)

```

### Result
```
output with missing paddings:  01001101 01001001 01010100 01001101 01011111 01000100 01010000 00100001
binary without paddings:  10011011001001101010010011011011111100010010100000100001
flag: crew{MITM_DP!}
```
