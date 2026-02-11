---
layout: post
title: "YAKVDB gets brand new CLI"
date: 2025-04-25 02:18:41 +0100
categories: yakvdb db
---

I was calmly sleeping in my bed the other night, everything was fine, and then...

![Brain when Sleeping]({{ site.url }}/assets/2025-04-25-yakvdb-cli/brain-sleep.png)

(source: [imgflip](https://imgflip.com/i/ajtcod))

OK, time to handle overflow pages, finally address the 4+ y.o. [issue](https://github.com/sergey-melnychuk/yakvdb/issues/3).

It has became painfully obvious how to represent overflow. Currently `Slot` can be either `vlen > 0` for leaf values or `page > 0` for node entries. Why can't it be both for overflow? This is what the brain was thinking about at 2am, fixing an issue no one carese about in a 5+ year old educational project that... well, no one cares about but me.

(UPDATE from the future: overflow pages are implemented, [see PR](https://github.com/sergey-melnychuk/yakvdb/pull/6)).

But this post is not about overflow pages implementation. The main hero of this post is a brand new CLI that [YAKVDB](https://github.com/sergey-melnychuk/yakvdb) recently got. Let's see it in action with [seed-gen](https://github.com/sergey-melnychuk/seed-gen). The `seed-gen` is a very simple tool to generate ranbom seed phrases and store them into YAKVDB file. For this example we'll use 10k random seeds.

```
$ yak 10k.yak
YAK CLI started. Type 'help' for commands. Type 'exit' to quit.
Page size: 4096 bytes. Total pages: 501.
> help
Commands:
  lookup <key>       - Get value for key
  insert <key> <val> - Set value for key
  remove <key>       - Delete key
  min                - Show minimum key
  max                - Show maximum key
  len                - Count total entries (alias: size, count)
  from <key> <num>   - Show <num> entries starting after <key> (exclusive)
  till <key> <num>   - Show last <num> entries before <key> (exclusive)
  above <key>        - Show key above the given key
  below <key>        - Show key below the given key
  mode <str|hex>     - Set value display mode (str: utf8, hex: 0x...)
  tree               - Show B-tree structure (pages, types, fill ratios)
  page <id>          - Show detailed page information
  root               - Show root page details
  exit/quit          - Exit CLI
  help/?             - Show this help

Key/value arguments can be:
  - 0x...            - hex bytes (e.g. 0x68656c6c6f for 'hello')
  - "..." or '...'   - quoted string literal
  - plain            - plain string (utf8 bytes)
>
```

The help is very self-explanatory I think, just plain minimum of necessary tools, just like YAKVDB itself.

```
> min
0x000169928da76bc979ce77bb3b624e9d532692af
> max
0xfff54f5164549f5ed2e3ff6be2c24940dbad53a6
> lookup 0x000169928da76bc979ce77bb3b624e9d532692af
0x756e7665696c20666c75736820686561727420776869702074756e6120736f6d656f6e6520736164206c75636b792070696c6c2072656f70656e2064697a7a79206c6576656c
> mode str
Mode set to 'str'.
> lookup 0x000169928da76bc979ce77bb3b624e9d532692af
"unveil flush heart whip tuna someone sad lucky pill reopen dizzy level"
```

Navigating tree structure happens with tree/node/root commands, pretty self-explanatory as well:

```
> tree
B-Tree Structure:
=================
ID     TYPE     FILL%  KEYS                
---------------------------------------------
1      NODE     7      0x210ebace43d1b...  
2      LEAF     62     0x000169928da76...  
3      LEAF     63     0x88ce82f3beda6...  
4      LEAF     48     0x39a32c1624e97...  
5      LEAF     57     0xc7f7d8975976a...  
6      LEAF     66     0x577f1563fb9d1...  
7      LEAF     75     0x15aaabe2f7784...  
8      LEAF     49     0xa52c00fafe503...  
9      LEAF     57     0xdeb99bf333ac4...  
10     LEAF     65     0x6c4a3e636956c...  
11     LEAF     52     0x2ff2098a037fb...  
12     LEAF     46     0xbcaf5e6b7d6a1...  
13     LEAF     76     0x7d355d54aa71f...  
14     LEAF     59     0x4a9104d17d64f...  
15     LEAF     43     0x9b429784a60e8...  
16     LEAF     75     0xee845bd75613d... 
<scroll down, press Q to stop>
```

The root node has a little bit of special treatment:

```
> root
Root Page (ID: 1):
==================
Type: NODE
Fill: 7%
Capacity: 4096 bytes
Free: 3792 bytes
Entries: 8

Entries:
IDX  KEY                  VALUE               
--------------------------------------------
0    0x210ebace43d1b...   [PAGE: 94]          
1    0x42ddc9c946c51...   [PAGE: 356]         
2    0x60f290ee4e9c8...   [PAGE: 187]         
3    0x805caa2b39716...   [PAGE: 374]         
4    0xa0022039d958a...   [PAGE: 95]          
5    0xc1b020a0afd87...   [PAGE: 372]         
6    0xe1da6163053d7...   [PAGE: 195]         
7    0xfff54f5164549...   [PAGE: 401] 
```

Just a random page - not so much :)

```
> page 62
Page Details:
=============
ID: 62
Type: LEAF
Fill: 63%
Capacity: 4096 bytes
Free: 1499 bytes
Entries: 23

Entries:
IDX  KEY                  VALUE               
--------------------------------------------
0    0xc080a0ac2affd...   0x706f7461746f2...  
1    0xc081550a832d1...   0x6c61776e206d6...  
2    0xc0873f7a827ef...   0x77617272696f7...  
3    0xc09d008d9e2d2...   0x7368792062656...  
4    0xc09dc41470ae7...   0x6d65737361676...  
5    0xc0b3d0e8bd518...   0x72616e646f6d2...  
6    0xc0b8684aefe99...   0x6164647265737...  
7    0xc0b9d4e2c6963...   0x6665766572206...  
8    0xc0c9096e5f6fb...   0x6d6f6e69746f7...  
9    0xc0ccde3f7e33a...   0x7475726e20647...  
10   0xc0cf3b3cc50aa...   0x6f7264696e617...  
11   0xc0d5604b8b787...   0x7265616479207...  
12   0xc0dc3bbe975ba...   0x7669736974206...  
13   0xc0de605a9ccfc...   0x746f646179206...  
14   0xc0ed16b715e95...   0x72656379636c6...  
15   0xc0eddfb543055...   0x736f6d656f6e6...  
16   0xc0ff2a1e1b7f4...   0x6c616e6775616...  
17   0xc10013dc57571...   0x736d696c65207...  
18   0xc1010f89c1486...   0x7265677265742...  
19   0xc1020f2d5ba4e...   0x7374617920746...  
20   0xc10736434e0f5...   0x626f737320656...  
21   0xc1097771a32c1...   0x626c696e64206...  
22   0xc10c778b6a47b...   0x6c61796572206... 
```

When doing prefix lookup, use above/below:

```
> above 0xc10c778b6a47b0
0xc10c778b6a47b86b3cd1869789df9301239476c7
> lookup 0xc10c778b6a47b86b3cd1869789df9301239476c7
"layer museum another forward anger theme tool spread cost gesture chapter impact"
```

But remember that hex string has to have even number of characters ;)

```
> above 0xc10c778b6a47b
Invalid key format
```

That's all folks!
