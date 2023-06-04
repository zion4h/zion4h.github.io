---
title: 数位 DP
date: 2023-06-04 12:37:44
cover: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img016.jpg
thumbnail: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img016.jpg
categories: [IT]
tags:
    - 算法
    - DP
toc: true
---
算法常见题之**数位 DP**，从题面来看数据范围很大（比如 10 的 20 次方，**无法暴力**），喜欢问在 `[L, R]` 中有多少符合的数。
<!--more-->
数位 DP 板子如下：

```python
from functools import cache
number = 18890101
s = str(number)

@cache
def f(i, mask, is_limit, is_number):
    """
    :param i: 数字字符串 s 的下标
    :param mask: 记录已填数字（用于构造特殊整数）
    :param is_limit: 前面填的数字是否都对应 number 上面的
    :param is_number: 前面是否填入了数字
    :return: 符合规则的整数个数
    """
    if i is len(s):
        return int(is_number)
    ret = 0
    if not is_number:
        ret += f(i + 1, mask, False, False)
    up = int(s[i]) if is_limit else 9
    for d in range(1 - int(is_number), up + 1):
        if mask >> d & 1 == 0:
            ret += f(i + 1, mask | (1 << d), is_limit and d is up, True)
    return ret

ans = f(0, 0, True, False)
```

【1】[灵茶山艾府 - 数位 DP 通用模板](https://www.bilibili.com/video/BV1rS4y1s721/?spm_id_from=333.337.search-card.all.click&vd_source=e36f47f043068554931919060ccd92ef)
