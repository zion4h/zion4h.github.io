---
title: CSAPP Lab2 Defusing a Binary Bomb
date: 2022-06-02 12:00:00
cover: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img251.jpg
categories: [编程, 编程.Labs, CMU 15-213]
toc: true
---

由于实验需要在x86-64环境下使用，因此我不得不放弃使用m1芯片的mac转而使用win10在虚拟机上跑了个centos，这里用vmware安装好后，再在centos上安装GDB即可。
<!--more-->

实验前务必参考[讲义](http://csapp.cs.cmu.edu/3e/bomblab.pdf)。
## 准备工作

首先，使用objdump反编译bomb，方便后面拆炸弹。

```
[root@MiWiFi-R4A-srv bomb]# objdump -d bomb > bomb.txt
```

然后常用GDB命令查询了解即可，[GDB常用命令 - 大CC - 博客园 (cnblogs.com)](https://www.cnblogs.com/me115/p/3837960.html) 。最后是x86-64架构的寄存器含义，可以参考[【原创】X86_64/X86 GNU汇编、寄存器、内嵌汇编 - 沐多 - 博客园 (cnblogs.com)](https://www.cnblogs.com/wsg1100/p/14290340.html#%E4%BA%94%E5%B8%B8%E8%A7%81%E6%B1%87%E7%BC%96%E7%BB%93%E6%9E%84)，建议反复看。另外，如果想从gdb运行中途出来打印，又不想没头苍蝇一样打断点，可以直接俄ctrl+c出来，然后continue继续。

## Phase1

```c
/* Hmm...  Six phases must be more secure than one phase! */
    input = read_line();             /* Get input                   */
    phase_1(input);                  /* Run the phase               */
    phase_defused();                 /* Drat!  They figured it out!
				      * Let me know how they did it. */
    printf("Phase 1 defused. How about the next one?\n");
```

```
0000000000400ee0 <phase_1>:
  400ee0:	48 83 ec 08          	sub    $0x8,%rsp
  400ee4:	be 00 24 40 00       	mov    $0x402400,%esi
  400ee9:	e8 4a 04 00 00       	callq  401338 <strings_not_equal>
  400eee:	85 c0                	test   %eax,%eax
  400ef0:	74 05                	je     400ef7 <phase_1+0x17>
  400ef2:	e8 43 05 00 00       	callq  40143a <explode_bomb>
  400ef7:	48 83 c4 08          	add    $0x8,%rsp
  400efb:	c3                   	retq
```

从c代码中不难发现，我们输入一行字符串作为参数，进入phase1进行解析。在反汇编得到的文件中找到phase1相关部分，挨个解析，再次提示可以反复查阅[【原创】X86_64/X86 GNU汇编、寄存器、内嵌汇编 - 沐多 - 博客园 (cnblogs.com)](https://www.cnblogs.com/wsg1100/p/14290340.html#%E4%BA%94%E5%B8%B8%E8%A7%81%E6%B1%87%E7%BC%96%E7%BB%93%E6%9E%84)。

首先，esi寄存器保存了0x402400，按照经验这应该是一个地址，然后调用了函数strings_not_equal，其功能不难猜出是比较两个字符串是否相等。注意，esi常用来保存第二个参数，而edi常用保存第一个参数，由于在进入strings_not_equal之前并没有对其修改，所以其接受的第一个参数默认是父函数也就是phase_1接受的第一个（也是唯一一个）参数，即我们所输入的字符串。当我们得到结果后，会经过一段由test和je组合的检测代码，如果eax等于0则跳转，越过explode_bomb。

```
0000000000401338 <strings_not_equal>:
  401338:	41 54                	push   %r12
  40133a:	55                   	push   %rbp
  40133b:	53                   	push   %rbx
  40133c:	48 89 fb             	mov    %rdi,%rbx // 字符串input
  40133f:	48 89 f5             	mov    %rsi,%rbp // 字符串"???"
  401342:	e8 d4 ff ff ff       	callq  40131b <string_length>
  401347:	41 89 c4             	mov    %eax,%r12d
  40134a:	48 89 ef             	mov    %rbp,%rdi
  40134d:	e8 c9 ff ff ff       	callq  40131b <string_length>
  401352:	ba 01 00 00 00       	mov    $0x1,%edx
  401357:	41 39 c4             	cmp    %eax,%r12d
  40135a:	75 3f                	jne    40139b <strings_not_equal+0x63>
  40135c:	0f b6 03             	movzbl (%rbx),%eax
  40135f:	84 c0                	test   %al,%al // 长度为0 jump
  401361:	74 25                	je     401388 <strings_not_equal+0x50>
  401363:	3a 45 00             	cmp    0x0(%rbp),%al
  401366:	74 0a                	je     401372 <strings_not_equal+0x3a>
  401368:	eb 25                	jmp    40138f <strings_not_equal+0x57>
  40136a:	3a 45 00             	cmp    0x0(%rbp),%al
  40136d:	0f 1f 00             	nopl   (%rax)
  401370:	75 24                	jne    401396 <strings_not_equal+0x5e>
  401372:	48 83 c3 01          	add    $0x1,%rbx
  401376:	48 83 c5 01          	add    $0x1,%rbp
  40137a:	0f b6 03             	movzbl (%rbx),%eax
  40137d:	84 c0                	test   %al,%al
  40137f:	75 e9                	jne    40136a <strings_not_equal+0x32>
  401381:	ba 00 00 00 00       	mov    $0x0,%edx
  401386:	eb 13                	jmp    40139b <strings_not_equal+0x63>
  401388:	ba 00 00 00 00       	mov    $0x0,%edx
  40138d:	eb 0c                	jmp    40139b <strings_not_equal+0x63>
  40138f:	ba 01 00 00 00       	mov    $0x1,%edx
  401394:	eb 05                	jmp    40139b <strings_not_equal+0x63>
  401396:	ba 01 00 00 00       	mov    $0x1,%edx
  40139b:	89 d0                	mov    %edx,%eax
  40139d:	5b                   	pop    %rbx
  40139e:	5d                   	pop    %rbp
  40139f:	41 5c                	pop    %r12
  4013a1:	c3                   	retq
```

然后，进入strings_not_equal具体分析，string_length内部就是不断累加长度直到遇到’\0’。如果字符串a和b长度不相等就返回1，长度相等且长度为0则返回0，然后一个字符一个字符比较。

```bash
(gdb) print (char *)0x402400
$1 = 0x402400 "Border relations with Canada have never been better."
```

现在，我们知道0x402400确实是一个字符串起始地址，直接通过GDB找到并输入即可。

## Phase2

```c
/* The second phase is harder.  No one will ever figure out
     * how to defuse this... */
    input = read_line();
    phase_2(input);
    phase_defused();
    printf("That's number 2.  Keep going!\n");
```

```
0000000000400efc <phase_2>:
  400efc:	55                   	push   %rbp
  400efd:	53                   	push   %rbx
  400efe:	48 83 ec 28          	sub    $0x28,%rsp
  400f02:	48 89 e6             	mov    %rsp,%rsi
  400f05:	e8 52 05 00 00       	callq  40145c <read_six_numbers>
  400f0a:	83 3c 24 01          	cmpl   $0x1,(%rsp)
  400f0e:	74 20                	je     400f30 <phase_2+0x34>
  400f10:	e8 25 05 00 00       	callq  40143a <explode_bomb>
  400f15:	eb 19                	jmp    400f30 <phase_2+0x34>
  400f17:	8b 43 fc             	mov    -0x4(%rbx),%eax
  400f1a:	01 c0                	add    %eax,%eax
  400f1c:	39 03                	cmp    %eax,(%rbx)
  400f1e:	74 05                	je     400f25 <phase_2+0x29>
  400f20:	e8 15 05 00 00       	callq  40143a <explode_bomb>
  400f25:	48 83 c3 04          	add    $0x4,%rbx
  400f29:	48 39 eb             	cmp    %rbp,%rbx
  400f2c:	75 e9                	jne    400f17 <phase_2+0x1b>
  400f2e:	eb 0c                	jmp    400f3c <phase_2+0x40>
  400f30:	48 8d 5c 24 04       	lea    0x4(%rsp),%rbx
  400f35:	48 8d 6c 24 18       	lea    0x18(%rsp),%rbp
  400f3a:	eb db                	jmp    400f17 <phase_2+0x1b>
  400f3c:	48 83 c4 28          	add    $0x28,%rsp
  400f40:	5b                   	pop    %rbx
  400f41:	5d                   	pop    %rbp
  400f42:	c3                   	retq
```

进入其汇编码后，发现有个函数read_six_numbers，顾名思义读6个数字。

```
000000000040145c <read_six_numbers>:
  40145c:	48 83 ec 18          	sub    $0x18,%rsp
  401460:	48 89 f2             	mov    %rsi,%rdx // 0->参数3
  401463:	48 8d 4e 04          	lea    0x4(%rsi),%rcx // 4->参数4
  401467:	48 8d 46 14          	lea    0x14(%rsi),%rax 
  40146b:	48 89 44 24 08       	mov    %rax,0x8(%rsp) // 20->参数8
  401470:	48 8d 46 10          	lea    0x10(%rsi),%rax
  401474:	48 89 04 24          	mov    %rax,(%rsp) // 16->参数7
  401478:	4c 8d 4e 0c          	lea    0xc(%rsi),%r9 // 12->参数6
  40147c:	4c 8d 46 08          	lea    0x8(%rsi),%r8 // 8->参数5
  401480:	be c3 25 40 00       	mov    $0x4025c3,%esi
  401485:	b8 00 00 00 00       	mov    $0x0,%eax
  40148a:	e8 61 f7 ff ff       	callq  400bf0 <__isoc99_sscanf@plt>
  40148f:	83 f8 05             	cmp    $0x5,%eax
  401492:	7f 05                	jg     401499 <read_six_numbers+0x3d>
  401494:	e8 a1 ff ff ff       	callq  40143a <explode_bomb>
  401499:	48 83 c4 18          	add    $0x18,%rsp
  40149d:	c3                   	retq
```

实际上其汇编码有点乱，但经过注释，不难发现其实它就是将连续六个数字的位置映射到__isoc99_sscanf@plt函数的参数3~8中，第一个参数rdi虽然没写，但我们知道其实就是input，而第二个参数esi=0x4025c3是这个读写函数的格式，我们用gdb直接调试可以查到：

```bash
(gdb) print (char*)0x4025c3
$3 = 0x4025c3 "%d %d %d %d %d %d"
```

而且，由于第一个数字（我注释0→参数3那行）映射的rsi在父函数phase2中其实是rsp，这下我们终于明白出函数read_six_numbers后，首先做的是将第一个数和1比较，相等则跳过炸弹。然后是一个循环函数，每次让前一个数的两倍和当前数比较，翻译过就是`a[i + 1] = a[i] * 2`，直到6个数都遍历完。

最后，我们输入以1开始的一个等比数列即可“1 2 4 8 16 32”，当然由于跳转语句最后用的jg，我们后面还可以添加数字。另外，[函数__isoc99_sscanf@plt的返回值是其解析匹配成功的数字个数](https://stackoverflow.com/questions/22489434/how-does-call-isoc99-sscanf-work)。

## Phase3

```c
/* I guess this is too easy so far.  Some more complex code will
     * confuse people. */
    input = read_line();
    phase_3(input);
    phase_defused();
    printf("Halfway there!\n");
```

```
0000000000400f43 <phase_3>:
  400f43:	48 83 ec 18          	sub    $0x18,%rsp
  400f47:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx // 参数4
  400f4c:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx // 参数3
  400f51:	be cf 25 40 00       	mov    $0x4025cf,%esi
  400f56:	b8 00 00 00 00       	mov    $0x0,%eax
  400f5b:	e8 90 fc ff ff       	callq  400bf0 <__isoc99_sscanf@plt>
  400f60:	83 f8 01             	cmp    $0x1,%eax
  400f63:	7f 05                	jg     400f6a <phase_3+0x27> // jump greater
  400f65:	e8 d0 04 00 00       	callq  40143a <explode_bomb>
  400f6a:	83 7c 24 08 07       	cmpl   $0x7,0x8(%rsp)
  400f6f:	77 3c                	ja     400fad <phase_3+0x6a> // jump above
  400f71:	8b 44 24 08          	mov    0x8(%rsp),%eax
  400f75:	ff 24 c5 70 24 40 00 	jmpq   *0x402470(,%rax,8)
  400f7c:	b8 cf 00 00 00       	mov    $0xcf,%eax // 0->0xcf=207
  400f81:	eb 3b                	jmp    400fbe <phase_3+0x7b>
  400f83:	b8 c3 02 00 00       	mov    $0x2c3,%eax // 2
  400f88:	eb 34                	jmp    400fbe <phase_3+0x7b>
  400f8a:	b8 00 01 00 00       	mov    $0x100,%eax // 3
  400f8f:	eb 2d                	jmp    400fbe <phase_3+0x7b>
  400f91:	b8 85 01 00 00       	mov    $0x185,%eax // 4
  400f96:	eb 26                	jmp    400fbe <phase_3+0x7b>
  400f98:	b8 ce 00 00 00       	mov    $0xce,%eax // 5
  400f9d:	eb 1f                	jmp    400fbe <phase_3+0x7b>
  400f9f:	b8 aa 02 00 00       	mov    $0x2aa,%eax // 6
  400fa4:	eb 18                	jmp    400fbe <phase_3+0x7b>
  400fa6:	b8 47 01 00 00       	mov    $0x147,%eax // 7
  400fab:	eb 11                	jmp    400fbe <phase_3+0x7b>
  400fad:	e8 88 04 00 00       	callq  40143a <explode_bomb>
  400fb2:	b8 00 00 00 00       	mov    $0x0,%eax
  400fb7:	eb 05                	jmp    400fbe <phase_3+0x7b>
  400fb9:	b8 37 01 00 00       	mov    $0x137,%eax // 1
  400fbe:	3b 44 24 0c          	cmp    0xc(%rsp),%eax
  400fc2:	74 05                	je     400fc9 <phase_3+0x86>
  400fc4:	e8 71 04 00 00       	callq  40143a <explode_bomb>
  400fc9:	48 83 c4 18          	add    $0x18,%rsp
  400fcd:	c3                   	retq
```

在phase3起始位置我们看到了和read_six_numbers似曾相识的代码，不同的是这次读写格式地址变成了0x4025cf，我们使用gdb解析，得到

```bash
(gdb) print (char*)0x4025cf
$4 = 0x4025cf "%d %d"
```

这次只需要解析两个数字，它们分别放在0x8(%rsp)和0xc(%rsp)，理所当然的检查了一下解析数字数目，大于一个才能不爆炸。然后是ja，让第一个数（我们假设为a）不能大于7。然后是一个跳转指令，要跳到0x402470+8xa这个地址。由于8bit为两个字节，所以我们在查看从0~7对应的8组地址时需要16个字节，当然也可以更多就是了，不过我们只需要16个字节，注意用16进制表示。

```bash
(gdb) x/16x 0x402470
0x402470:	0x00400f7c	0x00000000	0x00400fb9	0x00000000
0x402480:	0x00400f83	0x00000000	0x00400f8a	0x00000000
0x402490:	0x00400f91	0x00000000	0x00400f98	0x00000000
0x4024a0:	0x00400f9f	0x00000000	0x00400fa6	0x00000000
```

结合这八个地址和机器码不难发现，这就是一个分支选择，写入下标和其对应数字即可，比如“0 207”。

## Phase4

```c
/* Oh yeah?  Well, how good is your math?  Try on this saucy problem! */
    input = read_line();
    phase_4(input);
    phase_defused();
    printf("So you got that one.  Try this one.\n");
```

```
000000000040100c <phase_4>:
  40100c:	48 83 ec 18          	sub    $0x18,%rsp
  401010:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx
  401015:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx
  40101a:	be cf 25 40 00       	mov    $0x4025cf,%esi
  40101f:	b8 00 00 00 00       	mov    $0x0,%eax
  401024:	e8 c7 fb ff ff       	callq  400bf0 <__isoc99_sscanf@plt>
  401029:	83 f8 02             	cmp    $0x2,%eax
  40102c:	75 07                	jne    401035 <phase_4+0x29>
  40102e:	83 7c 24 08 0e       	cmpl   $0xe,0x8(%rsp)
  401033:	76 05                	jbe    40103a <phase_4+0x2e> // below & equal
  401035:	e8 00 04 00 00       	callq  40143a <explode_bomb>
  40103a:	ba 0e 00 00 00       	mov    $0xe,%edx
  40103f:	be 00 00 00 00       	mov    $0x0,%esi
  401044:	8b 7c 24 08          	mov    0x8(%rsp),%edi
  401048:	e8 81 ff ff ff       	callq  400fce <func4>
  40104d:	85 c0                	test   %eax,%eax
  40104f:	75 07                	jne    401058 <phase_4+0x4c>
  401051:	83 7c 24 0c 00       	cmpl   $0x0,0xc(%rsp)
  401056:	74 05                	je     40105d <phase_4+0x51>
  401058:	e8 dd 03 00 00       	callq  40143a <explode_bomb>
  40105d:	48 83 c4 18          	add    $0x18,%rsp
  401061:	c3                   	retq
```

phase4依然是读入两个数字，jbe就是jump+below+equal的组合，其含义不言自明，第一个数要小于等于14。然后预设了两个参数14和0，和输入的第一个数一起传入func4。后面紧跟test和jne，要求func4结果必须为0，后面的cmpl和je要求第二个数必须为0。一切都很明朗，我们需要输入一个x和一个0。

```
0000000000400fce <func4>:
  400fce:	48 83 ec 08          	sub    $0x8,%rsp
  400fd2:	89 d0                	mov    %edx,%eax 
  400fd4:	29 f0                	sub    %esi,%eax  
  400fd6:	89 c1                	mov    %eax,%ecx
  400fd8:	c1 e9 1f             	shr    $0x1f,%ecx // shift+right
  400fdb:	01 c8                	add    %ecx,%eax
  400fdd:	d1 f8                	sar    %eax
  400fdf:	8d 0c 30             	lea    (%rax,%rsi,1),%ecx // (r - l) / 2 + l
  400fe2:	39 f9                	cmp    %edi,%ecx
  400fe4:	7e 0c                	jle    400ff2 <func4+0x24>
  400fe6:	8d 51 ff             	lea    -0x1(%rcx),%edx // r = mid - 1
  400fe9:	e8 e0 ff ff ff       	callq  400fce <func4>
  400fee:	01 c0                	add    %eax,%eax
  400ff0:	eb 15                	jmp    401007 <func4+0x39>
  400ff2:	b8 00 00 00 00       	mov    $0x0,%eax
  400ff7:	39 f9                	cmp    %edi,%ecx
  400ff9:	7d 0c                	jge    401007 <func4+0x39>
  400ffb:	8d 71 01             	lea    0x1(%rcx),%esi // l = mid + 1
  400ffe:	e8 cb ff ff ff       	callq  400fce <func4>
  401003:	8d 44 00 01          	lea    0x1(%rax,%rax,1),%eax
  401007:	48 83 c4 08          	add    $0x8,%rsp
  40100b:	c3                   	retq
```

由于func4有递归，所以我们可以假设输入的是x，l，r，然后是(r-l)/2+l。这里有一些细节，shr逻辑右移时用0填充，lea等效于mov的源操作数从地址变成了内容（lea (A) B == mov A B）。以及通过逻辑右移获得l和r两个变量差值的符号位，这里需要解释一下，由于ecx是32位形式，所以逻辑右移31位得到符号位，这个操作本质是让r大于等于l。然后，我们不难发现，eax会在从高半区返回后乘以2加1（401003），而从低半区返回后仅仅乘以2。

综上，我们了解到，首先需要r大于等于l，然后每次都从低半区出来，由于判断语句是jle，所以等于情况也算作低半区。又因为之前分析过，x需要小于等于14，所以只有0，1，3，7这四个数字符合，填入即可。

## Phase5

```c
/* Round and 'round in memory we go, where we stop, the bomb blows! */
    input = read_line();
    phase_5(input);
    phase_defused();
    printf("Good work!  On to the next...\n");
```

```
0000000000401062 <phase_5>:
  401062:	53                   	push   %rbx
  401063:	48 83 ec 20          	sub    $0x20,%rsp
  401067:	48 89 fb             	mov    %rdi,%rbx
  40106a:	64 48 8b 04 25 28 00 	mov    %fs:0x28,%rax
  401071:	00 00 
  401073:	48 89 44 24 18       	mov    %rax,0x18(%rsp)
  401078:	31 c0                	xor    %eax,%eax
  40107a:	e8 9c 02 00 00       	callq  40131b <string_length>
  40107f:	83 f8 06             	cmp    $0x6,%eax
  401082:	74 4e                	je     4010d2 <phase_5+0x70>
  401084:	e8 b1 03 00 00       	callq  40143a <explode_bomb>
  401089:	eb 47                	jmp    4010d2 <phase_5+0x70>
  40108b:	0f b6 0c 03          	movzbl (%rbx,%rax,1),%ecx
  40108f:	88 0c 24             	mov    %cl,(%rsp)
  401092:	48 8b 14 24          	mov    (%rsp),%rdx
  401096:	83 e2 0f             	and    $0xf,%edx
  401099:	0f b6 92 b0 24 40 00 	movzbl 0x4024b0(%rdx),%edx
  4010a0:	88 54 04 10          	mov    %dl,0x10(%rsp,%rax,1)
  4010a4:	48 83 c0 01          	add    $0x1,%rax
  4010a8:	48 83 f8 06          	cmp    $0x6,%rax
  4010ac:	75 dd                	jne    40108b <phase_5+0x29>
  4010ae:	c6 44 24 16 00       	movb   $0x0,0x16(%rsp)
  4010b3:	be 5e 24 40 00       	mov    $0x40245e,%esi
  4010b8:	48 8d 7c 24 10       	lea    0x10(%rsp),%rdi
  4010bd:	e8 76 02 00 00       	callq  401338 <strings_not_equal>
  4010c2:	85 c0                	test   %eax,%eax
  4010c4:	74 13                	je     4010d9 <phase_5+0x77>
  4010c6:	e8 6f 03 00 00       	callq  40143a <explode_bomb>
  4010cb:	0f 1f 44 00 00       	nopl   0x0(%rax,%rax,1)
  4010d0:	eb 07                	jmp    4010d9 <phase_5+0x77>
  4010d2:	b8 00 00 00 00       	mov    $0x0,%eax
  4010d7:	eb b2                	jmp    40108b <phase_5+0x29>
  4010d9:	48 8b 44 24 18       	mov    0x18(%rsp),%rax
  4010de:	64 48 33 04 25 28 00 	xor    %fs:0x28,%rax
  4010e5:	00 00 
  4010e7:	74 05                	je     4010ee <phase_5+0x8c>
  4010e9:	e8 42 fa ff ff       	callq  400b30 <__stack_chk_fail@plt>
  4010ee:	48 83 c4 20          	add    $0x20,%rsp
  4010f2:	5b                   	pop    %rbx
  4010f3:	c3                   	retq
```

通过函数string_length函数限制输入字符串长度为6，然后是一段循环，观察代码(%rbx,%rax,1)，由于rbx是起始位置，所以不断增加rax即可达到遍历每个字符的效果。由于cl是rcx的8比特形式（相当于mod255），后面又和0xf相与（相当于去后四位为有效数字），然后将结果同地址0x4024b0加起来得到一个新的地址，这个地址显然保存着一个数，将这个数存到edx。而dl是edx的8比特形式，存到0x10(%rsp+rax * 1)。综合来看，就是将我们输入的六个字符按照特殊方式挨个转换成新的字符，最后同0x40245e处的字符串比较，匹配就不爆炸。

```
(gdb) print (char*)0x4024b0
$6 = 0x4024b0 <array> "maduiersnfotvbylSo you think you can stop the bomb with ctrl-c, do you?"
(gdb) print (char*)0x40245e
$7 = 0x40245e "flyers"
```

这里字典很长，但其实我们只需要前16个有效字母，也就是“maduiersnfotvbyl”，而“flyers”映射到字母序列为“f-9 l-15 y-14 e-5 r-6 s-7”，而恰好a的ASCII码等于0110 0001，当然A对应0100 0001，所以大小写其实都是等效的，我们通过目标字符串解析的对应下标，倒推我们的源码为“ionefg”。

另外，汇编中的401063、40106a和4010d9、4010de呼应，如果二者不匹配会报错。

## Phase6

```c
/* This phase will never be used, since no one will get past the
 * earlier ones.  But just in case, make this one extra hard. */
input = read_line();
phase_6(input);
phase_defused();
```

```
00000000004010f4 <phase_6>:
  4010f4:	41 56                	push   %r14
  4010f6:	41 55                	push   %r13
  4010f8:	41 54                	push   %r12
  4010fa:	55                   	push   %rbp
  4010fb:	53                   	push   %rbx
  4010fc:	48 83 ec 50          	sub    $0x50,%rsp
  401100:	49 89 e5             	mov    %rsp,%r13
  401103:	48 89 e6             	mov    %rsp,%rsi
  401106:	e8 51 03 00 00       	callq  40145c <read_six_numbers>
  40110b:	49 89 e6             	mov    %rsp,%r14
  40110e:	41 bc 00 00 00 00    	mov    $0x0,%r12d
====================================================
  401114:	4c 89 ed             	mov    %r13,%rbp
  401117:	41 8b 45 00          	mov    0x0(%r13),%eax
  40111b:	83 e8 01             	sub    $0x1,%eax
  40111e:	83 f8 05             	cmp    $0x5,%eax
  401121:	76 05                	jbe    401128 <phase_6+0x34>
  401123:	e8 12 03 00 00       	callq  40143a <explode_bomb>
  401128:	41 83 c4 01          	add    $0x1,%r12d
  40112c:	41 83 fc 06          	cmp    $0x6,%r12d
  401130:	74 21                	je     401153 <phase_6+0x5f>
  401132:	44 89 e3             	mov    %r12d,%ebx
  401135:	48 63 c3             	movslq %ebx,%rax
  401138:	8b 04 84             	mov    (%rsp,%rax,4),%eax
  40113b:	39 45 00             	cmp    %eax,0x0(%rbp)
  40113e:	75 05                	jne    401145 <phase_6+0x51>
  401140:	e8 f5 02 00 00       	callq  40143a <explode_bomb>
  401145:	83 c3 01             	add    $0x1,%ebx
  401148:	83 fb 05             	cmp    $0x5,%ebx
  40114b:	7e e8                	jle    401135 <phase_6+0x41>
  40114d:	49 83 c5 04          	add    $0x4,%r13
  401151:	eb c1                	jmp    401114 <phase_6+0x20>
====================================================
  401153:	48 8d 74 24 18       	lea    0x18(%rsp),%rsi
  401158:	4c 89 f0             	mov    %r14,%rax
  40115b:	b9 07 00 00 00       	mov    $0x7,%ecx
  401160:	89 ca                	mov    %ecx,%edx
  401162:	2b 10                	sub    (%rax),%edx
  401164:	89 10                	mov    %edx,(%rax)
  401166:	48 83 c0 04          	add    $0x4,%rax
  40116a:	48 39 f0             	cmp    %rsi,%rax
  40116d:	75 f1                	jne    401160 <phase_6+0x6c>
  40116f:	be 00 00 00 00       	mov    $0x0,%esi
  401174:	eb 21                	jmp    401197 <phase_6+0xa3>
  401176:	48 8b 52 08          	mov    0x8(%rdx),%rdx
  40117a:	83 c0 01             	add    $0x1,%eax
  40117d:	39 c8                	cmp    %ecx,%eax
  40117f:	75 f5                	jne    401176 <phase_6+0x82>
  401181:	eb 05                	jmp    401188 <phase_6+0x94>
  401183:	ba d0 32 60 00       	mov    $0x6032d0,%edx
  401188:	48 89 54 74 20       	mov    %rdx,0x20(%rsp,%rsi,2)
  40118d:	48 83 c6 04          	add    $0x4,%rsi
  401191:	48 83 fe 18          	cmp    $0x18,%rsi
  401195:	74 14                	je     4011ab <phase_6+0xb7>
  401197:	8b 0c 34             	mov    (%rsp,%rsi,1),%ecx
  40119a:	83 f9 01             	cmp    $0x1,%ecx
  40119d:	7e e4                	jle    401183 <phase_6+0x8f>
  40119f:	b8 01 00 00 00       	mov    $0x1,%eax
  4011a4:	ba d0 32 60 00       	mov    $0x6032d0,%edx
  4011a9:	eb cb                	jmp    401176 <phase_6+0x82>
====================================================
  4011ab:	48 8b 5c 24 20       	mov    0x20(%rsp),%rbx
  4011b0:	48 8d 44 24 28       	lea    0x28(%rsp),%rax
  4011b5:	48 8d 74 24 50       	lea    0x50(%rsp),%rsi
  4011ba:	48 89 d9             	mov    %rbx,%rcx
  4011bd:	48 8b 10             	mov    (%rax),%rdx
  4011c0:	48 89 51 08          	mov    %rdx,0x8(%rcx)
  4011c4:	48 83 c0 08          	add    $0x8,%rax
  4011c8:	48 39 f0             	cmp    %rsi,%rax
  4011cb:	74 05                	je     4011d2 <phase_6+0xde>
  4011cd:	48 89 d1             	mov    %rdx,%rcx
  4011d0:	eb eb                	jmp    4011bd <phase_6+0xc9>
  4011d2:	48 c7 42 08 00 00 00 	movq   $0x0,0x8(%rdx)
  4011d9:	00 
  4011da:	bd 05 00 00 00       	mov    $0x5,%ebp
  4011df:	48 8b 43 08          	mov    0x8(%rbx),%rax
  4011e3:	8b 00                	mov    (%rax),%eax
  4011e5:	39 03                	cmp    %eax,(%rbx)
  4011e7:	7d 05                	jge    4011ee <phase_6+0xfa>
  4011e9:	e8 4c 02 00 00       	callq  40143a <explode_bomb>
  4011ee:	48 8b 5b 08          	mov    0x8(%rbx),%rbx
  4011f2:	83 ed 01             	sub    $0x1,%ebp
  4011f5:	75 e8                	jne    4011df <phase_6+0xeb>
  4011f7:	48 83 c4 50          	add    $0x50,%rsp
  4011fb:	5b                   	pop    %rbx
  4011fc:	5d                   	pop    %rbp
  4011fd:	41 5c                	pop    %r12
  4011ff:	41 5d                	pop    %r13
  401201:	41 5e                	pop    %r14
  401203:	c3                   	retq
```

[c++ - Print the whole linked list in gdb? - Stack Overflow](https://stackoverflow.com/questions/16480045/print-the-whole-linked-list-in-gdb)

在分析代码的过程，我发现这段冗长的代码主要可以分为三个部分，一开始是常规的读入我们输入的六个数字，然后在第一部分将输入input根据值转换成新的六个数字（这里我们设其为数组a），然后在第二部分会根据a的值和一个特殊地址0x5032d0二者得到得到六个地址（这里我们设其为数组b，里面每个值都是一个地址），最后在第三部分判断根据b的地址得到的值是否满足降序排列，也就是[b[i] ≥ b[i + 1]]。

下面我们逆向分析，首先通过打印0x5032d0处的数据不难发现，其类型为节点，而通过之前的分析，我们知道一共有6个节点，观察发现每个节点占四个字节：

```
(gdb) x/24w 0x6032d0
0x6032d0 <node1>:	0x0000014c	0x00000001	0x006032e0	0x00000000
0x6032e0 <node2>:	0x000000a8	0x00000002	0x006032f0	0x00000000
0x6032f0 <node3>:	0x0000039c	0x00000003	0x00603300	0x00000000
0x603300 <node4>:	0x000002b3	0x00000004	0x00603310	0x00000000
0x603310 <node5>:	0x000001dd	0x00000005	0x00603320	0x00000000
0x603320 <node6>:	0x000001bb	0x00000006	0x00000000	0x00000000
```

经过解析得到六个节点值为0x14c、0xa8、0x39c、0x2b3、0x1dd、0x1bb，为了让其降序排列，我们需要将其下标交换，得到3、4、5、6、1、2，也就是b[0]存node3的地址，b[1]存node4的地址，以此类推。

再进入第二部分，经过分析，我们发现可以分两种情况：当a[i]≤1时，直接让b[i]=node1；当a[i]>1时，会有一个变量%edx（我们假设它是j）从2到6遍历，直到j和a[i]相等时，让b[i]=nodej。现在我们应该能推断，实际上a[i]的值应是2~6以及一个可能为1或者0的值，而这部分的逻辑就是b[i]=node_a[i]，而为了得到b=[node3，node4，node5，node6，node1，node2]，需要让a=[3, 4, 5, 6, 0 or 1, 2]。

现在再分析第一部分，这部分就相对比较简单了，a[i] = 7 - input[i]，且a[i]两两不相等并且限制了输入数字不能大于6，所以a[4]不能等于0，综上得到input=“4 3 2 1 6 5”

答案：

1."Border relations with Canada have never been better."

2.“1 2 4 8 16 32”

3.“0 207”

4.“0 0”

5.“ionefg”

6.“4 3 2 1 6 5”

参考：

[《深入理解计算机系统》Bomb Lab实验解析 | Yi's Blog (earthaa.github.io)](https://earthaa.github.io/2020/01/12/CSAPP-Bomblab/)

[CMU 15213: Bomb Lab (1) | Wei Bai 白巍 (wordpress.com)](https://baiweiblog.wordpress.com/2018/05/12/cmu-15213-bomb-lab/)
