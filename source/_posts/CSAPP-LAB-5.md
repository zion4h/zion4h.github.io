---
title: CSAPP Lab5 Understanding Cache Memories
date: 2022-06-05 12:00:00
cover: /img/cover/img253.jpg
categories: [编程, 编程.Labs, CMU 15-213]
toc: true
---

## 环境准备

参考指导文档[cachelab.dvi (cmu.edu)](http://csapp.cs.cmu.edu/3e/cachelab.pdf)，我们需要在linux环境下解压文档。
<!--more-->

```bash
[root@MiWiFi-R4A-srv csapp]# cd cachelab-handout/
[root@MiWiFi-R4A-srv cachelab-handout]# ls
cachelab.c  csim.c    driver.py  README     test-trans.c  traces
cachelab.h  csim-ref  Makefile   test-csim  tracegen.c    trans.c
[root@MiWiFi-R4A-srv cachelab-handout]# make clean
rm -rf *.o
rm -f *.tar
rm -f csim
rm -f test-trans tracegen
rm -f trace.all trace.f*
rm -f .csim_results .marker
[root@MiWiFi-R4A-srv cachelab-handout]# make
gcc -g -Wall -Werror -std=c99 -m64 -o csim csim.c cachelab.c -lm 
gcc -g -Wall -Werror -std=c99 -m64 -O0 -c trans.c
gcc -g -Wall -Werror -std=c99 -m64 -o test-trans test-trans.c cachelab.c trans.o 
gcc -g -Wall -Werror -std=c99 -m64 -O0 -o tracegen tracegen.c trans.o cachelab.c
# Generate a handin tar file each time you compile
tar -cvf zion-handin.tar  csim.c trans.c 
csim.c
trans.
```

这里提一句，由于我使用的是win10+VMware+Centos，每次都是通过Clion在共享文件夹中编辑文档，再在虚拟机中运行，因此在这个实验PartB中遇到了由共享文件夹造成的超时问题。解决思路也很简单，复制项目到非共享文件目录再运行即可。以及，每次都需要通过以下指令将共享目录可见，

```
vmware-hgfsclient # 查看当前共享目录有哪些
vmhgfs-fuse .host:/  /mnt/hgfs -o subtype=vmhgfs-fuse,allow_other -o nonempty # 把共享目录放到/mnt/hgfs中
```

然后我们需要安装valgrind，它是一个内存调试检测工具，项目中traces目录下的文档就是该工具生成的样例，我们也可以试着运行它去检测指令（比如论文示范的“ls -l”）。

```c
[root@MiWiFi-R4A-srv zion]# yum install valgrind
上次元数据过期检查：0:02:55 前，执行于 2022年07月10日 星期日 12时34分57秒。
软件包 valgrind-1:3.19.0-3.el9.x86_64 已安装。
依赖关系解决。
无需任何处理。
完毕！
```

在实验过程中，主要参考指导文档[cachelab.dvi (cmu.edu)](http://csapp.cs.cmu.edu/3e/cachelab.pdf)和教学PPT[rec07.pdf (cmu.edu)](http://www.cs.cmu.edu/afs/cs/academic/class/15213-f15/www/recitations/rec07.pdf)，尤其后者有清晰配图，容易理解。

## PartA

在PartA中，我们工作就是编写一个类似valgrind的内存监测工具，当然并非完全复刻，而是只实现其中的路径追踪即可。

lab本身提供了一个模拟器csim-ref，我们要做的就是编写实现相同功能的c程序，实现的重点是对内存存储方式的理解。实际上我们并没有对缓存做任何操作，我们会读入trace文件，根据不同的s、E、b构成的不同缓存环境，判断miss、hit和eviction的次数。也就是说我们输入一个按照`[space]operation address,size`方式描述trace的文本文档，而c程序会根据其文本描述解析每条指令L、M或S的访存情况，最后统计并反馈给`printSummary`。

```bash
[root@MiWiFi-R4A-srv cachelab-handout]# ./csim-ref -s 4 -E 1 -b 4 -t traces/yi.trace -v
L 10,1 miss 
M 20,1 miss hit 
L 22,1 hit 
S 18,1 hit 
L 110,1 miss eviction 
L 210,1 miss eviction 
M 12,1 miss eviction hit 
hits:4 misses:5 evictions:3
```

主存到cache有三种映射方式：直接映射、全相联映射、组相联映射。根据参数说明，缓存本身有2^s组，每组由E块组成，当然每个块有2^b字节。比如文中示例的“-s 4 -E 1 -b 4”，说明当前缓存环境为有16组，每组1块，每个块16字节。再根据PPT，我们知道访存地址可以拆分为三个部分，tag、组号和块偏移。

接着，我们一条一条指令分析。“L 10，1”说明访问地址0x10，访问1个字节，访问方式为Load。因此，而每块16字节，因此地址0x10表示访问第1组，并且tag为0x0的块，发现没有，于是从主存中复制过来，记一次miss。同理，“M 20，1”访问第2组，发现没有找到tag为0x0的块，于是同样记一次miss。这里注意，由于M等于“L+S”，所以虽然L部分算做miss，但S部分却hit了，因此还有记一次hit。“L 22，1”访问第2组且tag为0x0的块，发现有，于是记一次hit，同理“S 18，1”访问第1组且tag为0x0的块，同样找到了，记一次hit。再之后，“L 110，1”访问组1且tag为0x1的块，没找到，且组1已经满了（由于E=1表示每组容量上限为1个块），所以还要腾出一块，记一次miss和一次eviction。后面的不再赘述，经过一条条指令的分析，虽然一条代码都还没有写，但可以说我们已经搞清楚PartA是怎么一回事儿了。

一些细节，PPT已经帮我们处理好了，比如接收并处理指令参数的optget，也可看[Example of Getopt (The GNU C Library)](http://www.gnu.org/software/libc/manual/html_node/Example-of-Getopt.html)：

```c
		while (-1 != (opt = getopt(argc, argv, "hvs:E:b:t:"))) {
        switch (opt) {
            case 'h':
                print_usage_info();
                return -1;
            case 'v':
                v = 1;
                break;
            case 's':
                s = atoi(optarg);
                break;
            case 'E':
                E = atoi(optarg);
                break;
            case 'b':
                b = atoi(optarg);
                break;
            case 't':
                t = optarg;
                break;
            default:
                printf("wrong argument\n");
                print_usage_info();
                return -1;
        }
    }
```

还有读取文件，我们需要读t参数指定的trace文件：

```c
		FILE *pFile;
    pFile = fopen(t, "r");
    char identifier;
    unsigned long address;
    int size;

    while (fscanf(pFile, " %c %lx,%d", &identifier, &address, &size) > 0) {
        int hitsOld = hits, missesOld = misses, evictionsOld = evictions;
        switch (identifier) {
            case 'I':
                continue;
                break;
						...
				}
		}
```

接着，我讲讲我的思路，首先通过参数s、E、b建立输入命令的缓存环境。对于每一组缓存块，我们用一个LRU队列去维持。当访问块时，会遇到三种情况：1）在指定缓存队列中找到块，记hit；2）在指定缓存队列中没找到块，但队列空间有剩余（也就是小于E），记miss并在队首插入当前块；3）没找到块，且队列已满，将队尾缓存块释放，再在队首插入当前块。由于是LRU，我们每次都从队首开始遍历查找，按经验可以减少换出频率。

```c
typedef struct block{
    unsigned long tag;
    struct block *next;
}Block;
```

然后是每个缓存块，我们需要设置两个参数，一个是tag，用来判断是否是同一块；一个是next，指向队列下一个缓存块。简单来说，我们将每个地址尾部拆分成组下标和块偏移两个部分，同一组的组下标都是一样的，但起始tag不一样，比如“L 210，1”里的210实际上是0x1000010000，其中0000是块偏移，0001是组下标，0x10才是tag，所以我们的每个块结构都要记录tag来判断是否是同一块。

```bash
[root@MiWiFi-R4A-srv cachelab-handout]# gcc csim.c cachelab.c -o csimbak
[root@MiWiFi-R4A-srv cachelab-handout]# ./csimbak -s 2 -E 2 -b 3 -t traces/trans.trace
v:0 s:2 E:2 b:3 t:traces/trans.trace
hits:201 misses:37 evictions:29
```

除了make，我们也可以单独编译成csimbak（随便取的名哈）去测试执行指令，由于之前有用到math库的pow函数所以还加了参数-ml，但后来都优化成了位运算，就不再需要了。第一行是我的中间打印结果，后一行是需要保留的。最后，再进一步优化代码细节，具体参考项目文件csim.cTODO。

值得一提的是，在写代码过程中，突然忘记LRU要将访问节点交换到最开始位置了，导致debug了半小时，其实之前整理思路时已经规划了要去做这个的。虽然最找到bug，但这依然是一顿教训，以后千万不能无脑debug了，多分析多规划。

## PartB

lab已经预设了缓存环境参数为s=5，E=1，b=5，也就是32组，每组1块，每块32字节，由于int占4字节，因此1个块可以装8个整数。矩阵转置操作之所以要优化访问，是因为由于每组只有1块，当碰到A[0][0]翻转到B[0][0]时，二者对应到缓存中都是第0组，因此需要存取两次。而当访问到下一个元素A[0][1]时，又要将旧块放回去，由于miss次数多导致频繁的换入换出，虚拟内存的实际性能也将大幅降低。因此，我们的任务表面上是降低矩阵转置的miss次数，实际上是在探究什么样的访存才是高性能的。文中还藏了一些我不知道算不算彩蛋的，比如“column-wise scan”，“zig-zag access”，显然这二者都和原方法一样低效，因为换扫描方式是无法解决该问题的。

PPT中提到的一种解决思路，就是分块方法。我们以32x32为例，将其分为16个8x8的正方形，为什么选8呢，因为题设环境一个块最多装8个整数。简单思考一下，A的一个8x8正方形经过转置之后，这64个数字都恰好会放入B的某个8x8正方形中。

在最好情况下，比如左下角和右上角，我们可以同时装入A的8个块来载入其64个数字，和B的8个块，这16个块的组号都不一样，因此总共miss次数为16。这意味只需要加载一次，理想情况下甚至只需要256次miss即可完成整个32x32的矩阵转置，另外该部分满分需要降低miss次数到300次以下。

但最坏情况下，比如同为对角线上的8x8，假设我们在将A[0][0]传值给B[0][0]后，再去处理A[0][1]和B[1][0]的交换，会导致原本装A[0][0]的块反复换入换出。由于lab允许我们使用12个变量，因此我们可以考虑用变量将A[0][0]~A[0][7]记录下来，再传给B，从而减少miss次数。当然，这样做依然无法做到最优，因为B的块仍然在被反复换入换出，不过比之前好多了，最终测试也的确小于300次。

```c
void transpose_32_32(int M, int N, int A[N][M], int B[M][N]) {
    int v1, v2, v3, v4, v5, v6, v7, v8;
    for (int i = 0; i < N; i += 8) {
        for (int j = 0; j < M; j += 8) {
            for (int k = i; k < i + 8 && k < N; k++) {
                v1 = A[k][j];
                v2 = A[k][j + 1];
                v3 = A[k][j + 2];
                v4 = A[k][j + 3];
                v5 = A[k][j + 4];
                v6 = A[k][j + 5];
                v7 = A[k][j + 6];
                v8 = A[k][j + 7];
                B[j][k] = v1;
                B[j + 1][k] = v2;
                B[j + 2][k] = v3;
                B[j + 3][k] = v4;
                B[j + 4][k] = v5;
                B[j + 5][k] = v6;
                B[j + 6][k] = v7;
                B[j + 7][k] = v8;
            }
        }
    }
}
```

再考虑64x64，如果我们依然按照8x8分块，那么会有一个问题，之前我们能成功是因为缓存有32组，具体到32x32，每8行的元素行与行之间都不会发生组号冲突。但如果改成64x64，不会发生组号冲突的行范围就缩小到了4行了。但如果我们按照4x4分块，那么又会浪费一半的块空间，比如读A[0][0]时，实际上A[0][4]也被换入了，但我们并没有使用它。因此，我们仍然采用8x8，但是这个8x8实际上是由两个4x8拼凑而成的。

为了更好的理解，我将这个8x8块按照左上右上左下右下分为abcd四个区域。类似的，我们知道A的abcd四个区域翻转过后到B的abcd四个区域。更具体的说，A转置到B的过程中，a区域会翻转到a区域，b区域会翻转到c，而c会翻转到b，d区域翻转到d区域。这样，我们先读A的a和b区域（因为所属同一块），再放到B的a区域和b区域（暂存），然后再读A的c和d区域，再放到B的c和d区域。最后，将B的b区域和c区域交换。但是在交换b和c时必然遇到大量miss，因此，思考无果参考了网上其他人的答案。

```c
void transpose_64_64(int M, int N, int A[N][M], int B[M][N]) {
    int v1, v2, v3, v4, v5, v6, v7, v8;
    for (int i = 0; i < N; i++) {
        for (int j = 0; j < M; j++) {
            // 处理a区域和b区域
            for (int k = i; k < i + 4; i++) {
                v1 = A[k][j];
                v2 = A[k][j + 1];
                v3 = A[k][j + 2];
                v4 = A[k][j + 3];
                v5 = A[k][j + 4];
                v6 = A[k][j + 5];
                v7 = A[k][j + 6];
                v8 = A[k][j + 7];
                B[j][k] = v1;
                B[j + 1][k] = v2;
                B[j + 2][k] = v3;
                B[j + 3][k] = v4;
                B[j][k + 4] = v5;
                B[j + 1][k + 4] = v6;
                B[j + 2][k + 4] = v7;
                B[j + 3][k + 4] = v8;
            }

            // 处理c区域和d区域
            for (int k = i + 4; k < i + 8; i++) {
                v1 = A[k][j];
                v2 = A[k][j + 1];
                v3 = A[k][j + 2];
                v4 = A[k][j + 3];
                v5 = A[k][j + 4];
                v6 = A[k][j + 5];
                v7 = A[k][j + 6];
                v8 = A[k][j + 7];
                B[j + 4][k] = v1;
                B[j + 5][k] = v2;
                B[j + 6][k] = v3;
                B[j + 7][k] = v4;
                B[j + 4][k + 4] = v5;
                B[j + 5][k + 4] = v6;
                B[j + 6][k + 4] = v7;
                B[j + 7][k + 4] = v8;
            }

            // 交换B的b区域和c区域
            for (int k = 0; k < 4; k++) {
                for (int l = 0; l < 4; l++) {
                    v1 = B[j + k + 4][i + l];
                    B[j + k + 4][i + l] = B[j + k][i + l + 4];
                    B[j + k][i + l + 4] = v1;
                }
            }
        }
    }
```

最后考虑61x67，事实上我并没什么思路，考虑到之前因为对角线优化32x32，这次直接裸上一个8x8方格，事实证明不行。然后又测试了16x16，32x32，发现朴素的16x16也能直接降到2000以下。但别的优化手段，确实是没有了。

```c
void transpose_61_67(int M, int N, int A[N][M], int B[M][N]) {
    for (int i = 0; i < N; i += 16) {
        for (int j = 0; j < M; j += 16) {
            for (int k = i; k < i + 16 && k < N; k++) {
                for (int l = j; l < j + 16 && l < M; l++) {
                    B[l][k] = A[k][l];
                }
            }
        }
    }
}
```

在做partB的时候，可以用test-trans测试：

```bash
[root@MiWiFi-R4A-srv cachelab-handout]# ./test-trans -M 32 -N 32

Function 0 (5 total)
Step 1: Validating and generating memory traces
Step 2: Evaluating performance (s=5, E=1, b=5)
func 0 (Transpose submission): hits:1765, misses:288, evictions:256

Function 1 (5 total)
Step 1: Validating and generating memory traces
Step 2: Evaluating performance (s=5, E=1, b=5)
func 1 (Simple row-wise scan transpose): hits:869, misses:1184, evictions:1152
```

最后，利用driver.py测试评分：

```bash
[root@MiWiFi-R4A-srv cachelab-handout]# /usr/local/bin/python2.7 ./driver.py
Part A: Testing cache simulator
Running ./test-csim
                        Your simulator     Reference simulator
Points (s,E,b)    Hits  Misses  Evicts    Hits  Misses  Evicts
     3 (1,1,1)       9       8       6       9       8       6  traces/yi2.trace
     3 (4,2,4)       4       5       2       4       5       2  traces/yi.trace
     3 (2,1,4)       2       3       1       2       3       1  traces/dave.trace
     3 (2,1,3)     167      71      67     167      71      67  traces/trans.trace
     3 (2,2,3)     201      37      29     201      37      29  traces/trans.trace
     3 (2,4,3)     212      26      10     212      26      10  traces/trans.trace
     3 (5,1,5)     231       7       0     231       7       0  traces/trans.trace
     6 (5,1,5)  265189   21775   21743  265189   21775   21743  traces/long.trace
    27

Part B: Testing transpose function
Running ./test-trans -M 32 -N 32
Running ./test-trans -M 64 -N 64
Running ./test-trans -M 61 -N 67

Cache Lab summary:
                        Points   Max pts      Misses
Csim correctness          27.0        27
Trans perf 32x32           8.0         8         288
Trans perf 64x64           8.0         8        1180
Trans perf 61x67          10.0        10        1993
          Total points    53.0        53
```

注意，这里要用python2。现在python3.11都出来了，而且centos默认安装python3，所以要想使用python2必须专门去下载安装包了，下载安装过程我主要参考了[How To Install Python 2.7.16 on CentOS/RHEL 7/6 and Fedora 30-25 (tecadmin.net)](https://tecadmin.net/install-python-2-7-on-centos-rhel/)，之后就可以用了：

```bash
[root@MiWiFi-R4A-srv cachelab-handout]# /usr/local/bin/python2.7 ./driver.py
```

参考：

[1]:[CSAPP实验之cache lab - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/79058089)

[2]:[CSAPP:Lab4-Cache Lab - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/142942823)