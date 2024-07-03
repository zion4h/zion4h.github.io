---
title: 如何实现一个内存分配器？
date: 2024-07-03 10:58:58
cover: https://i.imgur.com/9tcpIkI.jpeg
thumbnail: https://i.imgur.com/9tcpIkI.jpeg
categories: ['编程', '扫盲']
tags:
    - intro
toc: true
excerpt: 这篇文章会用C写一个内存分配器，本质是对malloc(), calloc(), realloc() 和 free()的复现。
---

## 使用 C 语言实现内存分配器

在这篇文章中，我们将使用 C 语言编写一个简单的内存分配器。实际上，这个内存分配器是对 `malloc()`, `calloc()`, `realloc()` 和 `free()`函数的复现。

### 程序的虚拟地址空间结构

一个程序的虚拟地址空间结构包括以下部分：代码段 (text)，数据段 (data)，未初始化数据段 (BSS)，堆 (Heap) 和栈 (Stack)。其中，堆是用于动态内存分配的区域，内存块的申请和释放操作都在这里进行。`brk` 指向堆的末尾，我们可以通过`sbrk()`函数控制 `brk` 位置从而分配和释放内存。

![虚拟地址空间结构](https://i.imgur.com/9tcpIkI.jpeg)

### 内存分配的问题

在申请分配内存时，主要面临两个问题：

1. 假设我们申请了内存块 A、B、C，然后想要释放 B，但由于 C 还在使用中，实际上我们不能直接释放 B 的空间。
2. 当我们想释放 B 时，我们无法确定要释放多少字节。

#### 解决方案

- 对于问题 1，我们可以在释放 B 时将 B 标记为空闲状态，但保留 B 所占的内存区域。
- 对于问题 2，我们需要在申请内存时记录每个内存块的大小。

我们将创建一个名为 `header_block` 的结构体来保存这些信息。具体来说，当我们想要释放 B 时，我们会将 B 的状态设置为空闲状态。当申请新的内存块时，我们可以检查被标记为空闲的内存块，如果申请的空间小于或等于该内存块的大小，我们就直接使用这个空闲内存块，并将其状态修改为非空闲。为了方便管理，我们采用链表结构将申请的内存块链接起来。

![块链图示](https://i.imgur.com/BQqas9C.png)

### 线程安全

为了避免多个线程同时访问内存，我们需要给内存分配器加锁。我们将使用`pthread_mutex_t`全局锁。

```c
pthread_mutex_t global_malloc_lock;
pthread_mutex_lock(&global_malloc_lock);
pthread_mutex_unlock(&global_malloc_lock);
```

### 代码实现

在代码实现阶段，我们的目标是重写内存分配器，也就是重新实现 Linux 中的`malloc()`, `free()`, `calloc()`, 和 `realloc()`四个函数。您可以参考 Linux 手册的`malloc(3)`来查看具体的输入输出参数。

#### `malloc()`函数

- 接收一个调用块的大小，并返回一个指向分配内存的指针。需要注意的是，这里的内存是未初始化的。如果 size 为 0，那么返回 NULL。
- 由于我们的问题 1 和问题 2，我们需要给这个内存块定义一个头部 `header` 去记录内存块大小和空闲状态，因此实际向系统申请的内存是 `header` 和 `block` 的总和。

#### `free()`函数

- 接收一个内存块指针，并释放这个指针对应的内存。如果指针为空，那么不做任何操作。
- 在具体的实现中，我们首先需要得到这个 `block` 对应的 `header` 指针，然后检查当前内存块是否位于程序堆栈的顶部，如果是，那么释放它。

#### 实践

根据我们之前的研究和讨论，我们首先需要为内存块定义一个 `header`。这个 `header` 包含三个信息：内存块的大小、内存块是否可用以及指向下一个内存块的指针。

在这里，我们需要对齐 16 字节。这是因为在 64 位处理器上，16 字节是一个常用的对齐边界。我们使用 `size_t` 表示内存大小和数组索引的无符号整数类型。这通常对应于机器字大小。最终我们得到以下的代码：

```c
typedef char ALIGN[16]; // 对齐 16 字节
union header {
    struct {
        size_t size;
        unsigned is_free;
        union header *next;
    } h;
    ALIGN stub;
};

typedef union header header_block;
header_block *head, *tail;
void *malloc(size_t size) {
    if (size == 0) {
        return NULL;
    }
    void *block;
    block = sbrk(size + sizeof (header_block));
    if (block == (void *) -1) {
        return NULL;
    }
    header_block *hb = block;
    hb->h.size = size;
    hb->h.is_free = 0;
    hb->h.next = NULL;
    tail->h.next = block;
    tail = block;
    return block;
}
```

在这段代码中，我们首先检查请求的内存大小。如果大小为 0，我们直接返回 NULL。然后，我们使用 `sbrk()` 函数来分配内存。如果分配失败，我们返回 NULL。接着，我们创建一个 `header_block`，并设置它的大小、空闲状态和下一个块的指针。最后，我们更新链表的尾部，并返回新分配的内存块。

接下来，我们需要修改这个函数，使其在申请新内存块之前先查找空闲的内存块。如果找到一个足够大的空闲块，我们就直接分配这个块，否则我们就申请一个新的内存块。同时，我们还需要考虑到线程同步的问题。以下是修改后的代码：

```c
typedef union header header_block;
header_block *head, *tail;
pthread_mutex_t global_malloc_lock;
header_block *find_free_block(size_t size);
void *malloc(size_t size);
void free(void *_Nullable ptr);
void *calloc(size_t nmemb, size_t size);
void *realloc(void *_Nullable ptr, size_t size);
void *malloc(size_t size) {
    if (size == 0) {
        return NULL;
    }
    header_block *hb;
    void *block;
    pthread_mutex_lock(&global_malloc_lock);
    hb = find_free_block(size);
    if (hb) {
        hb->h.is_free = 0;
        pthread_mutex_unlock(&global_malloc_lock);
        return (void *) (hb + 1);
    }
    block = sbrk(size + sizeof(header_block));
    if (block == (void *) -1) {
        pthread_mutex_unlock(&global_malloc_lock);
        return NULL;
    }
    hb = block;
    hb->h.size = size;
    hb->h.is_free = 0;
    hb->h.next = NULL;
    if (!head) {
        head = hb;
    }
    if (tail) {
        tail->h.next = hb;
    }
    tail = hb;
    pthread_mutex_unlock(&global_malloc_lock);
    return (void *) (hb + 1);
}
header_block *find_free_block(size_t size) {
    header_block *curr = head;
    while (curr) {
        if (curr->h.size >= size) {
            return curr;
        }
        curr = curr->h.next;
    }
    return NULL;
}
```

在这段代码中，我们首先获取全局锁。然后，我们调用 `find_free_block()` 函数来查找一个足够大的空闲块。如果找到了，我们就将这个块标记为非空闲，释放锁，然后返回这个块。如果没有找到，我们就像以前一样使用 `sbrk()` 函数来分配新的内存。最后，我们更新链表的头部和尾部，释放锁，并返回新分配的内存。

`find_free_block()` 函数的实现很简单。它遍历链表，查找一个足够大的空闲块。如果找到了，它就返回这个块，否则返回 NULL。

### 内存释放

在分配内存之后，我们需要释放内存。释放内存的逻辑如下：首先我们将内存块标记为空闲，然后如果这个内存块是链表的最后一个块，我们就释放它。

```c
void free(void *_Nullable ptr) {
    if (!ptr || ptr == NULL) {
        return;
    }
    pthread_mutex_lock(&global_malloc_lock);
    header_block *hb = (header_block *) ptr - 1;
    hb->h.is_free = 1;
    if ((char *) ptr + hb->h.size == sbrk(0)) {
        if (head == tail) {
            head = tail = NULL;
        } else {
            header_block *tmp = head;
            while (tmp) {
                if (tmp->h.next == tail) {
                    tmp->h.next = NULL;
                    tail = tmp;
                }
                tmp = tmp->h.next;
            }
            tail = tmp;
        }
        sbrk(-hb->h.size - sizeof(header_block));
        pthread_mutex_unlock(&global_malloc_lock);
        return;
    }
    pthread_mutex_unlock(&global_malloc_lock);
}
```

在这段代码中，我们首先检查指针是否为空。如果为空，我们就直接返回。然后，我们获取全局锁，并找到这个指针对应的 `header_block`。我们将这个块标记为空闲，然后检查这个块是否是链表的最后一个块。如果是，我们就释放这个块。最后，我们释放锁。

### 内存初始化和重新分配

`calloc()`函数用于为数组分配内存块，而`realloc()`函数用于改变内存块的大小。如果`ptr`为 NULL 或者`size`为 0，那么它们的行为就等同于`malloc()`和`free()`。

```c
void *calloc(size_t nmemb, size_t size) {
    if (!nmemb || !size) {
        return NULL;
    }
    int total_size = nmemb * size;
    if (size != total_size / nmemb) {
        return NULL;
    }
    void *block = malloc(total_size);
    memset(block, 0, total_size);
    return block;
}
void *realloc(void *_Nullable ptr, size_t size) {
    if (ptr == NULL) {
        return malloc(size);
    }
    if (size == 0) {
        free(ptr);
        return NULL;
    }
    header_block *hb;
    hb = (header_block *) ptr - 1;
    if (hb->h.size >= size) {
        return ptr;
    }
    void *block = malloc(size);
    if (block) {
        memcpy(block, ptr, hb->h.size);
        free(ptr);
    }
    return block;
}
```

在这段代码中，`calloc()`函数首先检查输入参数。如果`nmemb`或`size`为 0，那么它就返回 NULL。然后，它计算总的内存大小，并检查是否发生溢出。如果没有发生溢出，它就使用`malloc()`函数来分配内存，并使用`memset()`函数来初始化内存。

`realloc()`函数首先检查输入参数。如果`ptr`为 NULL，那么它就调用`malloc()`函数。如果`size`为 0，那么它就调用`free()`函数并返回 NULL。然后，它检查当前的内存块是否足够大。如果足够大，那么它就直接返回这个块。否则，它就分配一个新的内存块，复制旧的内存块的内容，释放旧的内存块，然后返回新的内存块。

![运行测试结果](https://i.imgur.com/Aas0fDU.png)

### 完整代码

以下是我们的完整代码：

```c
#include <unistd.h>
#include <string.h>
#include <pthread.h>
#include <stdio.h>

typedef char ALIGN[16]; // 对齐16字节

union header {
    struct {
        size_t size;
        unsigned is_free;
        union header *next;
    } h;
    ALIGN stub;
};
typedef union header header_block;
header_block *head = NULL, *tail = NULL;
pthread_mutex_t global_malloc_lock;

header_block *find_free_block(size_t size);

void *malloc(size_t size);

void free(void *ptr);

void *calloc(size_t nmemb, size_t size);

void *realloc(void *ptr, size_t size);
void print_mem_list();
void *malloc(size_t size) {
    if (size == 0) {
        return NULL;
    }
    header_block *hb;
    void *block;
    pthread_mutex_lock(&global_malloc_lock);
    hb = find_free_block(size);
    if (hb) {
        hb->h.is_free = 0;
        pthread_mutex_unlock(&global_malloc_lock);
        return (void *) (hb + 1);
    }
    block = sbrk(size + sizeof(header_block));
    if (block == (void *) -1) {
        pthread_mutex_unlock(&global_malloc_lock);
        return NULL;
    }
    hb = block;
    hb->h.size = size;
    hb->h.is_free = 0;
    hb->h.next = NULL;
    if (!head) {
        head = hb;
    }
    if (tail) {
        tail->h.next = hb;
    }
    tail = hb;
    pthread_mutex_unlock(&global_malloc_lock);
    return (void *) (hb + 1);
}

header_block *find_free_block(size_t size) {
    header_block *curr = head;
    while (curr) {
        if (curr->h.is_free && curr->h.size >= size) {
            return curr;
        }
        curr = curr->h.next;
    }
    return NULL;
}


void free(void *ptr) {
    if (!ptr) {
        return;
    }
    pthread_mutex_lock(&global_malloc_lock);
    header_block *hb = (header_block *) ptr - 1;
    hb->h.is_free = 1;
    if ((char *) ptr + hb->h.size == sbrk(0)) {
        if (head == tail) {
            head = tail = NULL;
        } else {
            header_block *tmp = head;
            while (tmp) {
                if (tmp->h.next == tail) {
                    tmp->h.next = NULL;
                    tail = tmp;
                }
                tmp = tmp->h.next;
            }
        }
        sbrk(0 - hb->h.size - sizeof(header_block));
        pthread_mutex_unlock(&global_malloc_lock);
        return;
    }
    pthread_mutex_unlock(&global_malloc_lock);
    print_mem_list();
}

void *calloc(size_t nmemb, size_t size) {
    print_mem_list();
    if (!nmemb || !size) {
        return NULL;
    }
    size_t total_size = nmemb * size;
    if (size != total_size / nmemb) {
        return NULL;
    }
    void *block = malloc(total_size);
    if (!block) {
        return NULL;
    }
    memset(block, 0, total_size);
    print_mem_list();
    return block;
}

void *realloc(void *ptr, size_t size) {
    print_mem_list();
    if (!ptr || !size) {
        return malloc(size);
    }
    header_block *hb = (header_block *) ptr - 1;
    if (hb->h.size >= size) {
        return ptr;
    }
    void *block = malloc(size);
    if (block) {
        memcpy(block, ptr, hb->h.size);
        free(ptr);
    }
    print_mem_list();
    return block;
}

void print_mem_list()
{
    header_block *curr = head;
    printf("\nhead = %p, tail = %p \n", head, tail);
    while(curr) {
        printf("addr = %p, size = %zu, is_free=%u, next=%p\n",
               (void*)curr, curr->h.size, curr->h.is_free, (void*)curr->h.next);
        curr = curr->h.next;
    }
}
```

在这段代码中，我们首先定义了一些全局变量和函数原型。然后，我们实现了`malloc()`, `free()`, `calloc()`, 和 `realloc()`函数。最后，我们提供了一个`print_mem_list()`函数，用于打印当前的内存块列表。
