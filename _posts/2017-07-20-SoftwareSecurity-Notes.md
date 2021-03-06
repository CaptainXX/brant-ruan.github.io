---
title: Coursera | Software Security Notes
category: CS
---

## Coursera | Software Security Notes

### Overview and Preparation

Low-level Vulnerabilities:

- Programs written in C and C++
    - Buffer Overflows
        + On the stack
        + On the heap
        + Due to integer overflow
        + Over-writing and over-reading
    - Format string mismatches
    - Dangling pointer dereferences

So, Attacks:

- Stack smashing
- Format string attack
- Stale memory access
- Return-oriented programming (ROP)

Ensuring Memory Safety:

- Use a memory-safe programming language
- For C/C++, use automated defenses
    + Stack canaries
    + Non-executable data (W+X or DEP)
    + Address space layout randomization (ASLR)
    + Memory-safety enforcement (e.g. SoftBound)
    + Control-flow Integrity (CFI)

Secure Software development:

- Apply coding rules
- Apply automated code review techniques
    + Static analysis and symbolic execution (whitebox fuzz testing)
- Apply penetration testing

**Notes come from both videos and my learning from websites in references. And reference marked with √ is the one I have read and learnt.**

### Week 1 | Memory-based Attacks

> Analyzing security requires a whole-systems view.

##### Memory Layout

0x00000000 ~ 0xffffffff

Image from the lesson:

![]({{ site.url }}/resources/pictures/SoftwareSecurityNotes-0.jpg)

##### Buffer Overflows

```c
void func(char *arg1)
{
    char buffer[4];
    strcpy(buffer, arg1);
    ...
}
int main()
{
    char *mystr = "AuthMe!";
    func(mystr);
    ...
}
```

##### Code Injection

- Load code into memory
    + it can not contain any all-zero bytes
    + can not use loader (must self-contained)
    + goal: general-purpose shell
    + code to launch a shell is called shellcode

```c
#include <stdio.h>

int main()
{
    char *name[2];
    name[0] = "/bin/sh";
    name[1] = NULL;
    execve(name[0], name, NULL);
}
```

```asm
xor eax, eax
push eax
push 0x68732f2f
push 0x6e69622f
mov ebx, esp
push ebx
...
```

- Getting injected code to run
    + Overwrite eip

- Finding the return address
    + Without address randomization
        * Stack always starts from the same fixed address
        * Stack will grow, but usually does not grow very deeply
        * Improving chances: nop sleds

![]({{ site.url }}/resources/pictures/SoftwareSecurityNotes-1.jpg)

##### Dive Into Stack Overflow

##### Integer Overflow

**Example 1**

```c
/* width1.c - exploiting a trivial widthness bug */
#include <stdio.h>
#include <string.h>

int main(int argc, char *argv[]){
    unsigned short s;
    int i;
    char buf[80];
    if(argc < 3)
        return -1;
    i = atoi(argv[1]);
    s = i;
    if(s >= 80){            /* [w1] */
        printf("Oh no you don't!\n");
        return -1;
    }
    printf("s = %d\n", s);
    memcpy(buf, argv[2], i);
    buf[i] = '\0';
    printf("%s\n", buf);
    return 0;
}
```

If `i=65536`, what will happen? And stack overflow will follow.

**Example 2**

```c
int copy_something(char *buf, int len){
    char kbuf[800];

    if(len > sizeof(kbuf)){         /* [1] */
        return -1;
    }

    return memcpy(kbuf, buf, len);  /* [2] */
}
```

- `len > sizeof(kbuf)` uses `signed` comparation
- `memcpy(kbuf, buf, len)` takes `len` as `unsigned`

Integer overflows can be extremely dangerous, partly because it is impossible to detect them after they have happened.

##### Other Memory Exploits

The code injection attack we have just considered is call **stack smashing** attack.

Now let's see some other types of attack:

###### Heap Overflow

```c
typedef struct _vulnerable_struct{
    char buff[MAX_LEN];
    int (*cmp)(char *, char *);
} vulnerable;

int foo(vulnerable *s, char *one, char *two)
{
    strcpy(s->buff, one);
    strcat(s->buff, two);
    return s->cmp(s->buff, "file://foobar");
}
```

Variants:

- Overflow into C++ object vtable
    + C++ objects are represented using a vtable, containing pointers to the objects's methods
    + vtable is analogous to s->cmp in the previous example
- Overflow into adjacent objects
- Overflow heap metadata
    + Hidden header just before the pointer returned by malloc
    + Flow into that header to corrupt the heap itself

###### Integer Overflow

```c
void vulnerable()
{
    char *response;
    int nresp = packet_get_int();
    if(nresp > 0){
        response = malloc(nresp * sizeof(char *));
        for(i = 0; i < nresp; i++)
            response[i] = packet_get_string(NULL);
    }
}
```

If we set `nresp` to `1073741824` and `sizeof(char *)` is `4`, then `nresp * sizeof(char *)` overflows to become 0 

###### Read Overflow

```c
int main()
{
    char buf[100], *p;
    int i, len;
    while(1){
        // input an integer as length
        p = fgets(buf, sizeof(buf), stdin);
        if(p == NULL)
            return 0;
        len = atoi(p);
        // input message
        p = fgets(buf, sizeof(buf), stdin);
        if(p == NULL)
            return 0;
        // echo message
        for(i = 0; i < len; i++){
            if(!iscntrl(buf[i]))
                putchar(buf[i]);
            else
                putchar('.');
        }
        printf("\n");
    }
}
```

If `len > sizeof(buf)` then read overflow occurs.

Heartbleed is just a read overflow.

###### Stale Memory

A dangling pointer bug occurs when a pointer is freed, but the program continues to use it.

An attacker can arrange for the freed memory to be reallocated and under his control.

When dangling pointer is dereferenced, it will access attacker-controlled data

```c
struct foo{
    int (*cmp)(char *, char *);
};

struct foo *p = malloc(...);
free(p);

... // time goes by

q = malloc(...); // reuses memory

*q = 0xdeadbeef; // if attacker controls

...

p->cmp("hello", "hello"); // reuses dangling ptr
```

##### Format String Vulnerabilities

```c
void vulnerable()
{
    char buf[80];
    if(fgets(buf, sizeof(buf), stdin) == NULL)
       return;
    printf(buf); 
}
```

- printf("%d"); // four bytes above stored eip
- printf("%s");
- print("100%no way!"); // writes `3` to address pointed to by stack entry

##### Project 1

Tasks are easy. But the `runbin.sh` is useful even when you do actual exploits, which allows you inputing hex value directly:

```sh
#!/bin/bash

while read -r line; do echo -e $line; done | ./wisdom-alt
```

##### Open File Safely

Opening safely should be simple, but is not.

Strategies for safe open:

- Verify path is trusted
    + Do not use if not trusted
- Safely open an untrusted file
    + Prevents common security problems with
        * symbolic links
        * misuse of API leading to weak permissions
    + Detect attacks of the path

###### Untrusted File

Case: Program assumes only trusted user can modify the file while attacker can also do that.

###### Untrusted Directory

Case: Program assumes only trusted user can modify the directory entry (e.g. delete the trusted file) while attacker can also do that.

###### Symbolic Link Attack

Checking whether a files exists or not before creating it is good, but cracker may create a file between your check and the moment you actually use the file (race condition). It sounds like mutex issue.

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

#define MY_TMP_FILE "/tmp/file.tmp"

int main(int argc, char **argv)
{
    FILE *tmpFile;
    if(!access(MY_TMP_FILE, F_OK)){
        printf("File exists!\n");
        return EXIT_FAILURE;
    }
    /* At this point the attacker creates a symlink 
       from /tmp/file.tmp to /etc/passwd
     */
    tmpFile = fopen(MY_TMP_FILE, "w");
    if(tmpFile == NULL){
        return EXIT_FAILURE;
    }
    fputs("Some text...\n", tmpFile);
    fclose(tmpFile);
    // now you overwrite /etc/passwd
    return EXIT_SUCCESS;
}
``` 

Mitigation:

```c
#include <unistd.h>
#include <stdio.h>
#include <fcntl.h>
#include <stdlib.h>
 
#define MY_TMP_FILE "/tmp/file.tmp"
 
enum { FILE_MODE = 0600 };
 
int main(int argc, char* argv[])
{
    int fd;
    FILE* f;
 
    /* Remove possible symlinks */
    unlink(MY_TMP_FILE);
    /* Open, but fail if someone raced us and restored the symlink (secure version of fopen(path, "w") */
    /* if successfully, then the file must be created by this program with its privilege (e.g. root). So attacker can not overwrite or delete this file unless he is with equal or greater than that privilege. If he is, there's no need for him to do this attack. */ 
    fd = open(MY_TMP_FILE, O_WRONLY | O_CREAT | O_EXCL, FILE_MODE);
    if (fd == -1) {
        perror("Failed to open the file");
        return EXIT_FAILURE;
    }
    /* Get a FILE*, as they are easier and more efficient than plan file descriptors */
    f = fdopen(fd, "w");
    if (f == NULL) {
        perror("Failed to associate file descriptor with a stream");
        return EXIT_FAILURE;
    }
    fprintf(f, "Hello, world\n");
    fclose(f);
    /* fd is already closed by fclose()!!! */
    return EXIT_SUCCESS;
}
```

###### Hard Link Attack

I do not understand that case. On RedHat 7 I fail to create a hard link to the file to which I am inaccessible.

##### Learn More

- [Common vulnerabilities guide for C programmers √](https://security.web.cern.ch/security/recommendations/en/codetools/c.shtml)
- [How to Open a File and Not Get Hacked √](http://research.cs.wisc.edu/mist/presentations/kupsch_miller_secse08.pdf)
- [Memory Layout of C Programs √](http://www.geeksforgeeks.org/memory-layout-of-c-program/)
- [How security flaws work: The buffer overflow √](https://arstechnica.com/security/2015/08/how-security-flaws-work-the-buffer-overflow/)
- [Smashing the Stack for Fun and Profit ](http://insecure.org/stf/smashstack.html)
- [Exploiting Format String Vulnerabilities](https://crypto.stanford.edu/cs155/papers/formatstring-1.2.pdf)
- [Basic Integer Overflows √](http://phrack.org/issues/60/10.html)

### Week 2 | Defenses Against Low-level Attacks

##### Canaries

- Terminator canaries (CR/LF/NUL)
- Random canaries
- Random XOR canaries

##### N^X

But ret2libc can bypass it.

##### ASLR

Available on Linux in 2004.  
May not apply to program code.

##### ROP

> by Hovav Shacham in 2007

- find gadgets
- string them together

pop ret

https://github.com/0vercl0k/rp

##### Blind ROP

Attack ASLR

If server restarts on a crash, but does not re-randomize:

1. Read the stack to leak canaries and a return address
2. Find gadgets (at run-time) to effect call to write
3. Dump binary to find gadgets for shellcode

The exploit was carried out on a 64-bit executable with full stack canaries and randomization.

##### Control Flow Integrity

Idea: observe the program's behavior; is it doing what we expect it to? If not, it might be compromised

- Define "expected behavior"

Control flow graph (CFG)

- Detect deviations from expection efficiently

In-line reference monitor (IRM)

- Avoid compromise of the detector

Sufficient randomness, immutability

Monitor only indirect calls: jmp, call, ret

##### Cat and Mourse

```
Defense: N^X
Attack: ret2libc
Defense: ASLR
Attack: Brute force search (for 32-bit)
Attack: Information leak (format string vulnerability)
Defense: Avoid using libc code entirely
Attack: ROP
```

##### Learn More

- [What is memory safety?](http://www.pl-enthusiast.net/2014/07/21/memory-safety/)
- [What is type safety?](www.pl-enthusiast.net/2014/08/05/type-safety/)
- [On the Effectiveness of Address-Space Randomization](http://cseweb.ucsd.edu/~hovav/papers/sppgmb04.html)
- [Smashing the Stack in 2011](https://paulmakowski.wordpress.com/2011/01/25/smashing-the-stack-in-2011/)
- [Low-Level Software Security by Example](https://courses.cs.washington.edu/courses/cse484/14au/reading/low-level-security-by-example.pdf)
- [Geometry of Innocent Flesh on the Bone: Return to libc without Function Calls (on the x86)](https://cseweb.ucsd.edu/~hovav/dist/geometry.pdf)
- [Exploit Hardening Made Easy](https://www.usenix.org/legacy/event/sec11/tech/full_papers/Schwartz.pdf)
- [Blind ROP](http://www.scs.stanford.edu/brop/)
- [Control-Flow Integrity](https://www.microsoft.com/en-us/research/publication/control-flow-integrity/?from=http%3A%2F%2Fresearch.microsoft.com%2Fpubs%2F64250%2Fccs05.pdf#)
- [Enforcing Forward-Edge Control Flow Integrity](https://www.usenix.org/conference/usenixsecurity14/technical-sessions/presentation/tice)
- [MoCFI](www.cse.lehigh.edu/~gtan/paper/mcfi.pdf)
- [Secure Programming HOWTO](https://www.dwheeler.com/secure-programs/Secure-Programs-HOWTO/internals.html)
- [Robust Programming](http://nob.cs.ucdavis.edu/bishop/secprog/robust.html)
- [CERT C coding standard](https://www.securecoding.cert.org/confluence/display/c/SEI+CERT+C+Coding+Standard)
- [DieHard project](http://plasma.cs.umass.edu/emery/diehard.html)

### Week 3 | Web Security

##### Learn More

- [OWASP's guide to SQL injection](https://www.owasp.org/index.php/SQL_Injection)
- [SQL injection cheat sheet](http://ferruh.mavituna.com/sql-injection-cheatsheet-oku/)
- [OWASP's guide to session hijacking](https://www.owasp.org/index.php/Session_hijacking_attack)
- [CWE/SANS top 25 most dangerous software errors](https://cwe.mitre.org/top25/)

### Week 5 | Static Analysis and Symbolic Execution for Security

##### Learn More

- [What is noninteference, and how do we enforce it?](http://www.pl-enthusiast.net/2015/03/03/noninterference/)
- [A Few Billion Lines of Code Later: Using Static Analysis to Find Bugs in the Real World](cacm.acm.org/magazines/2010/2/69354-a-few-billion-lines-of-code-later/fulltext)
- [All You Ever Wanted to Know About Dynamic Taint Analysis and Forward Symbolic Execution (but might have been afraid to ask)](https://edmcman.github.io/papers/oakland10.pdf)