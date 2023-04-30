---
title: CSAPP Lab7 Writing a Dynamic Storage Allocator
date: 2022-06-07 12:00:00
cover: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img268.jpg
categories: [编程, Labs, CMU 15-213]
toc: true
---

## 准备

首先，在 linux 环境下解压，由于在 make 过程中发现缺少 glibc 的库，所以还要安装相关包：

```c
[root@MiWiFi-R4A-srv malloclab-handout]# tar xvf malloclab-handout.tar
[root@MiWiFi-R4A-srv malloclab-handout]# cd malloclab-handout/
[root@MiWiFi-R4A-srv malloclab-handout]# yum install glibc-devel.i686
[root@MiWiFi-R4A-srv malloclab-handout]# yum install libstdc++-devel.i686
```
<!--more-->

lab 缺少 trace 文件，可自行下载 [CSAPP-Lab/initial_labs/08_Malloc Lab/traces at master · Deconx/CSAPP-Lab (github.com)](https://github.com/Deconx/CSAPP-Lab/tree/master/initial_labs/08_Malloc%20Lab/traces)。然后，详读 [malloclab.dvi (cmu.edu)](http://csapp.cs.cmu.edu/3e/malloclab.pdf) 和书上 9.9 节。

## 思路和过程

众所周知，由于通常是等程序实际运行时，我们才知道数据结构的实际大小，所以需要动态内存来专门维护进程的虚拟内存的。动态内存的分配方式主要有两种：类似于 C 中 malloc 包这种显式分配和 Java 中垃圾收集器这种隐式分配，当然在这个 lab 中我们主要考虑前者。

为了设计好这个分配器，我们需要阅读和掌握 malloc、sbrk、free 等函数，这里我主要参考书上介绍内容和 [Memory Allocation and C (The GNU C Library)](https://www.gnu.org/software/libc/manual/html_node/Memory-Allocation-and-C.html)。简单来说，malloc 返回一个 size 大小的可用块，sbrk 直接扩展收缩堆并返回旧 brk（这里可以参考 memlib.c），而 free 则是对指针有要求。

我们使用的隐式空闲链表中的每个块主要由四个部分组成，头部、有效载荷、填充字段（可选）和尾部，其中头部尾部是 knuth 提出以便于边界判定从而快速追溯和合并连续空闲块。在采用双字对齐时，块的大小是 8 的倍数（默认一个字 4 字节，也就是 32bit，双字就是 64bit）。因此块头虽然表示块大小，但后 3 位实际上是无用的（因为有效大小应是 8 的倍数），所以直接用它来表示附加信息，比如用最后一位表示块是否空闲，在 PPT 中也有其他用法。

```c
/* 从一个字中读出块长和分配位，和 PACK 对应 */
#define GET_SIZE(p)  (GET(p) & ~0x7)
#define GET_ALLOC(p) (GET(p) & 0x1)

/* 通过块指针得到该块头部和脚部地址 */
#define HDRP(bp) ((char *)(bp) - WSIZE)
#define FTRP(bp) ((char *)(bp) + GET_SIZE(HDRP(bp)) - DSIZE)
```

当一个块被释放时，就会涉及空闲块合并，可分为 4 种情况 afa、aff、ffa 和 fff。在编码过程中应注意，块指针指向的是第一个有效载荷字节地址，当然如果有效载荷为空则指向尾部。当堆被初始化时或者当分配函数无法找到一个合适的匹配块时，就会拓展堆。mem_sbrk 每次调用都返回一个块，由于 bp 指向的是结尾块（就是那个 0/1 块），因此结尾块成了新的头部，并设置为空闲。注意啊，这个 sbrk 其实只返回了一个双字对齐的内存片段，它一个部分成了新块的载荷和尾部，还有一部分成了新的结尾块。再之后，拓展完新块会遇到最后一个分配块为空闲的情况，所以拓展堆后还需要将二者合并。

堆在初始化时需要四个字，其中第二个字和第三个字构成起始块（载荷为 0），第四个字构成结尾块。之后让头指针就指向起始块，后面我们采用邻近适配法时新指针也指向这个块。

测试前，我们需要在 config.h 中为自己 tracefiles 文件夹的路径设置 TRACEDIR，然后 make 执行即可：

```bash
[root@localhost malloclab-handout]# make && ./mdriver -V
make: “mdriver”已是最新。
Team Name:H.Z.N.
Member 1 :Zion Huang:zion@abc.com
Using default tracefiles in /mnt/hgfs/csapp/malloclab-handout/tracefiles/
Measuring performance with gettimeofday().

Testing mm malloc
Reading tracefile: amptjp-bal.rep
Checking mm_malloc for correctness, efficiency, and performance.
Reading tracefile: cccp-bal.rep
Checking mm_malloc for correctness, efficiency, and performance.
Reading tracefile: cp-decl-bal.rep
Checking mm_malloc for correctness, efficiency, and performance.
Reading tracefile: expr-bal.rep
Checking mm_malloc for correctness, efficiency, and performance.
Reading tracefile: coalescing-bal.rep
Checking mm_malloc for correctness, efficiency, and performance.
Reading tracefile: random-bal.rep
Checking mm_malloc for correctness, efficiency, and performance.
Reading tracefile: random2-bal.rep
Checking mm_malloc for correctness, efficiency, and performance.
Reading tracefile: binary-bal.rep
Checking mm_malloc for correctness, efficiency, and performance.
Reading tracefile: binary2-bal.rep
Checking mm_malloc for correctness, efficiency, and performance.
Reading tracefile: realloc-bal.rep
Checking mm_malloc for correctness, efficiency, and performance.
Reading tracefile: realloc2-bal.rep
Checking mm_malloc for correctness, efficiency, and performance.

Results for mm malloc:
trace  valid  util     ops      secs  Kops
 0       yes   99%    5694  0.006877   828
 1       yes   99%    5848  0.006181   946
 2       yes   99%    6648  0.010611   626
 3       yes  100%    5380  0.007164   751
 4       yes   66%   14400  0.000089161435
 5       yes   92%    4800  0.006173   778
 6       yes   92%    4800  0.005336   900
 7       yes   55%   12000  0.128997    93
 8       yes   51%   24000  0.233592   103
 9       yes   27%   14401  0.043192   333
10       yes   34%   14401  0.001473  9780
Total          74%  112372  0.449685   250

Perf index = 44 (util) + 17 (thru) = 61/100
```

动态内存分配也有多种方式，包括首次适配、最佳适配、最差适配和下次适配，我们仿照课本代码实现的是一个典型的首次适配算法，用下次适配法，可以得到：

```bash
[root@localhost malloclab-handout]# make && ./mdriver -V
gcc -Wall -O2 -m32   -c -o mm.o mm.c
gcc -Wall -O2 -m32 -o mdriver mdriver.o mm.o memlib.o fsecs.o fcyc.o clock.o ftimer.o
Team Name:H.Z.N.
Member 1 :Zion Huang:zion@abc.com
Using default tracefiles in /mnt/hgfs/csapp/malloclab-handout/tracefiles/
Measuring performance with gettimeofday().

Testing mm malloc
Reading tracefile: amptjp-bal.rep
Checking mm_malloc for correctness, efficiency, and performance.
Reading tracefile: cccp-bal.rep
Checking mm_malloc for correctness, efficiency, and performance.
Reading tracefile: cp-decl-bal.rep
Checking mm_malloc for correctness, efficiency, and performance.
Reading tracefile: expr-bal.rep
Checking mm_malloc for correctness, efficiency, and performance.
Reading tracefile: coalescing-bal.rep
Checking mm_malloc for correctness, efficiency, and performance.
Reading tracefile: random-bal.rep
Checking mm_malloc for correctness, efficiency, and performance.
Reading tracefile: random2-bal.rep
Checking mm_malloc for correctness, efficiency, and performance.
Reading tracefile: binary-bal.rep
Checking mm_malloc for correctness, efficiency, and performance.
Reading tracefile: binary2-bal.rep
Checking mm_malloc for correctness, efficiency, and performance.
Reading tracefile: realloc-bal.rep
Checking mm_malloc for correctness, efficiency, and performance.
Reading tracefile: realloc2-bal.rep
Checking mm_malloc for correctness, efficiency, and performance.

Results for mm malloc:
trace  valid  util     ops      secs  Kops
 0       yes   86%    5694  0.000073 78430
 1       yes   90%    5848  0.000069 84265
 2       yes   94%    6648  0.000082 80582
 3       yes   95%    5380  0.000072 74412
 4       yes   66%   14400  0.000081178882
 5       yes   84%    4800  0.000667  7197
 6       yes   82%    4800  0.001048  4582
 7       yes   55%   12000  0.000088135747
 8       yes   51%   24000  0.000146163934
 9       yes   26%   14401  0.045631   316
10       yes   34%   14401  0.001445  9966
Total          69%  112372  0.049403  2275

Perf index = 42 (util) + 40 (thru) = 82/100
```

这里是代码：

```c
/*
 * mm-naive.c - The fastest, least memory-efficient malloc package.
 *
 * In this naive approach, a block is allocated by simply incrementing
 * the brk pointer.  A block is pure payload. There are no headers or
 * footers.  Blocks are never coalesced or reused. Realloc is
 * implemented directly using mm_malloc and mm_free.
 *
 * NOTE TO STUDENTS: Replace this header comment with your own header
 * comment that gives a high level description of your solution.
 */
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>
#include <unistd.h>
#include <string.h>

#include "mm.h"
#include "memlib.h"

/* 操作空闲链表的基本定义和宏 */
#define WSIZE     4 // 字长
#define DSIZE     8 // 双字长
#define CHUNKSIZE (1 << 12) // 初始堆默认为 4kB，拓展堆也基于此

#define MAX(x, y) ((x) > (y) ? (x) : (y))

/* 打包块长和分配位到一个字，主要用在头部和脚部 */
#define PACK(size, alloc) ((size) | (alloc))

#define GET(p)      (*(unsigned int *)(p))
#define PUT(p, val) (*(unsigned int *)(p) = (val))

/* 从一个字中读出块长和分配位，和 PACK 对应 */
#define GET_SIZE(p)  (GET(p) & ~0x7)
#define GET_ALLOC(p) (GET(p) & 0x1)

/* 通过块指针得到该块头部和脚部地址 */
#define HDRP(bp) ((char *)(bp) - WSIZE)
#define FTRP(bp) ((char *)(bp) + GET_SIZE(HDRP(bp)) - DSIZE)

/* 通过块指针得到前一块和后一块的块指针 */
#define NEXT_BLKP(bp) ((char *)(bp) + GET_SIZE(((char *)(bp) - WSIZE)))
#define PREV_BLKP(bp) ((char *)(bp) - GET_SIZE(((char *)(bp) - DSIZE)))

static char *heap_listp;
static char *heap_bp;

static void *coalesce(void *bp) {
    size_t prev_alloc = GET_ALLOC(FTRP(PREV_BLKP(bp)));
    size_t next_alloc = GET_ALLOC(HDRP(NEXT_BLKP(bp)));
    size_t size = GET_SIZE(HDRP(bp));

    if (prev_alloc && next_alloc) { // case 1
        heap_bp = bp;
        return bp;
    } else if (prev_alloc && !next_alloc) {
        size += GET_SIZE(HDRP(NEXT_BLKP(bp)));
        PUT(HDRP(bp), PACK(size, 0));
        PUT(FTRP(bp), PACK(size, 0));
    } else if (!prev_alloc && next_alloc) {
        size += GET_SIZE(HDRP(PREV_BLKP(bp)));
        PUT(FTRP(bp), PACK(size, 0));
        PUT(HDRP(PREV_BLKP(bp)), PACK(size, 0));
        bp = PREV_BLKP(bp);
    } else {
        size += GET_SIZE(FTRP(NEXT_BLKP(bp))) + GET_SIZE(HDRP(PREV_BLKP(bp)));
        PUT(HDRP(PREV_BLKP(bp)), PACK(size, 0));
        PUT(FTRP(NEXT_BLKP(bp)), PACK(size, 0));
        bp = PREV_BLKP(bp);
    }

    heap_bp = bp;
    return bp;
}

static void *extend_heap(size_t words) {
    char *bp;
    size_t size;

    // 维持双字对齐
    size = (words % 2) ? (words + 1) * WSIZE : words * WSIZE;
    if ((long) (bp = mem_sbrk(size)) == -1) {
        return NULL;
    }

    PUT(HDRP(bp), PACK(size, 0));
    PUT(FTRP(bp), PACK(size, 0));
    PUT(HDRP(NEXT_BLKP(bp)), PACK(0, 1));

    return coalesce(bp);
}

static void *find_fit(size_t asize) {
    char *bp;
    for (bp = heap_bp; GET_SIZE(HDRP(bp)) != 0; bp = NEXT_BLKP(bp)) {
        if (GET_ALLOC(HDRP(bp)) == 0 && GET_SIZE(HDRP(bp)) >= asize) {
            heap_bp = bp;
            return bp;
        }
    }

    for (bp = heap_bp; bp < heap_bp; bp = NEXT_BLKP(bp)) {
        if (GET_ALLOC(HDRP(bp)) == 0 && GET_SIZE(HDRP(bp)) >= asize) {
            heap_bp = bp;
            return bp;
        }
    }
    return NULL;
}

static void place(void *bp, size_t asize) {
    // 剩余部分必须要超过 DSIZE 才切割
    size_t origin = GET_SIZE(HDRP(bp));
    size_t rest = origin - asize;
    if (rest >= (2*DSIZE)) {
        PUT(HDRP(bp),PACK(asize,1));
        PUT(FTRP(bp),PACK(asize,1));
        PUT(HDRP(NEXT_BLKP(bp)),PACK(rest,0));
        PUT(FTRP(NEXT_BLKP(bp)),PACK(rest,0));
    } else {
        PUT(HDRP(bp),PACK(origin,1));
        PUT(FTRP(bp),PACK(origin,1));
    }
    heap_bp = bp;
}

/*********************************************************
 * NOTE TO STUDENTS: Before you do anything else, please
 * provide your team information in the following struct.
 ********************************************************/
team_t team = {
        /* Team name */
        "H.Z.N.",
        /* First member's full name */
        "Zion Huang",
        /* First member's email address */
        "zion@abc.com",
        /* Second member's full name (leave blank if none) */
        "",
        /* Second member's email address (leave blank if none) */
        ""
};

/* single word (4) or double word (8) alignment */
#define ALIGNMENT 8

/* rounds up to the nearest multiple of ALIGNMENT */
#define ALIGN(size) (((size) + (ALIGNMENT-1)) & ~0x7)

#define SIZE_T_SIZE (ALIGN(sizeof(size_t)))

/*
 * mm_init - initialize the malloc package.
 */
int mm_init(void) {
    if ((heap_listp = mem_sbrk(4 * WSIZE)) == (void *) -1)
        return -1;
    PUT(heap_listp, 0);
    PUT(heap_listp + (1 * WSIZE), PACK(DSIZE, 1));
    PUT(heap_listp + (2 * WSIZE), PACK(DSIZE, 1));
    PUT(heap_listp + (3 * WSIZE), PACK(0, 1));
    heap_listp += (2 * WSIZE);
    heap_bp = heap_listp;

    if (extend_heap(CHUNKSIZE / WSIZE) == NULL) {
        return -1;
    }
    return 0;
}

/*
 * mm_malloc - Allocate a block by incrementing the brk pointer.
 *     Always allocate a block whose size is a multiple of the alignment.
 */
void *mm_malloc(size_t size) {
    size_t asize; // 真正的块大小需要包括头部脚部（1 DSIZE）和双字对齐的载荷
    size_t extendsize; // 拓展堆大小（如果没匹配成功）
    char *bp;

    // 第一次申请 malloc
    if (heap_listp == 0) {
        mm_init();
    }

    if (size == 0)
        return NULL;

    if (size <= DSIZE)
        asize = DSIZE * 2;
    else
        asize = DSIZE * ((DSIZE + size + (DSIZE - 1)) / DSIZE);

    if ((bp = find_fit(asize)) != NULL) {
        place(bp, asize);
        return bp;
    }

    extendsize = MAX(asize, CHUNKSIZE);
    if ((bp = extend_heap(extendsize / WSIZE)) == NULL)
        return NULL;

    place(bp, asize);
    return bp;
}

/*
 * mm_free - Freeing a block does nothing.
 */
void mm_free(void *ptr) {
    size_t size = GET_SIZE(HDRP(ptr));

    PUT(HDRP(ptr), PACK(size, 0));
    PUT(FTRP(ptr), PACK(size, 0));
    coalesce(ptr);
}

/*
 * mm_realloc - Implemented simply in terms of mm_malloc and mm_free
 */
void *mm_realloc(void *ptr, size_t size) {
    size_t oldsize;
    void *newptr;

    if (ptr == NULL) {
        return mm_malloc(size);
    }

    if (size == 0) {
        mm_free(ptr);
        return NULL;
    }

    newptr = mm_malloc(size);

    /* If realloc() fails the original block is left untouched  */
    if(!newptr) {
        return NULL;
    }

    /* Copy the old data. */
    oldsize = GET_SIZE(HDRP(ptr));
    // 改进
    size = GET_SIZE(HDRP(newptr));
    if(oldsize < size) {
        size = oldsize;
    }
    memcpy(newptr, ptr, size - DSIZE);

    /* Free the old block. */
    mm_free(ptr);

    return newptr;
}
```

`TODO`，参考书和资料只能得到 82 分左右，但如果我们采用伙伴系统 buddy system 应该会得到更高的分数吧，时间紧迫，暂时搁置了。
