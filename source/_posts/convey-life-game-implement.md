---
title: 复现康威的生命游戏
date: 2024-07-04 19:13:07
cover: https://i.imgur.com/QGol8ne.png
thumbnail: https://i.imgur.com/QGol8ne.png
categories: ['编程', '扫盲']
tags:
    - intro
toc: true
excerpt: 康威生命游戏（英语：Conway's Game of Life），又称康威生命棋，是英国数学家约翰·何顿·康威在1970年发明的细胞自动机。它最初于1970年10月在《科学美国人》杂志上马丁·葛登能的“数学游戏”专栏出现。
---

## 康威提出该游戏的原因

康威生命游戏（英语：_Conway's Game of Life_），又称康威生命棋，是英国数学家**约翰・何顿・康威**在 1970 年发明的**细胞自动机**。它最初于 1970 年 10 月在《**科学美国人**》杂志上马丁・葛登能的 “数学游戏” 专栏出现。

康威生命游戏是**图灵完备**的，简单来说就是它能做到**图灵机**的一切。图灵机的一个显著特点是能够产生和读取无限个 “0” 和 “1”，且支持条件分支并拥有无限大的内存。

图灵机是对人用纸笔计算的一种行为抽象，每一步计算都需要阅读纸上的数学符号，再经过大脑思考后得到结果。为了实现图灵机，需要有这样几个东西，**TAPE**、**HEAD**、**TABLE** 还有一个**状态寄存器**。考虑有一条无限长的纸带 **TAPE**，里面写着连续的数学符号（空白的地方可以视作一个特殊符号），一个读写头 **HEAD** 能够读取数学符号和左右移动，以及一套有限的规则表 **TABLE**。这个 **TABLE** 能够根据当前状态寄存器和 **HEAD** 指向 **TAPE** 的当前数学符号来确定下一步的动作，具体来说是改变 **TAPE** 上的数学符号，移动 **HEAD**，改变当前状态。

## 康威的生命游戏规则

在康威的生命游戏中，整个世界是建立在一片无限细胞格子棋盘中的，它们只有**死亡**和**存活**两种状态。每个周期结束时都会对所有细胞进行存活判定，该判定取决于周围 8 个细胞的存活状态：

1. 周围活细胞低于 2 个时，该细胞死亡（过于稀疏）。
2. 周围活细胞为 2 个或者 3 个时，该细胞保持不变。
3. 周围活细胞超过 3 个时，该细胞死亡（过于拥挤）。
4. 周围活细胞刚好为 3 个时，死细胞复活（模拟繁殖）。

## 部分代码展示

游戏引擎：

```cpp
// Game.cpp
#include "Game.h"
using namespace sf;

Game::Game(std::vector<std::pair<int, int>> initValue) {
    int width = initValue[0].first, height = initValue[0].second;
    map = BoardMap(width, height);
    for (size_t i = 1; i < initValue.size(); i++) {
        int x = initValue[i].first, y = initValue[i].second;
        map.setBoxValue(x, y, true);
    }
}

void Game::next() {
    if (!isRun()) {
        return;
    }
    map = map.gotoNext();
}

bool Game::isRun() {
    return map.isEmpty();
}

void Game::draw(RenderWindow& window) {
    int rows = map.getRows();
    int cols = map.getCols();
    window.clear(Color(234, 255, 208));
    for (int i = 0; i < rows; ++i) {
        for (int j = 0; j < cols; ++j) {
            // 绘制格子
            RectangleShape rectangle(Vector2f(CELL_SIZE, CELL_SIZE));
            rectangle.setPosition(j * CELL_SIZE, i * CELL_SIZE);
            if (map.getBoxValue(i, j)) {
                rectangle.setFillColor(Color::White); // 白色格子
            } else {
                rectangle.setFillColor(Color::Black); // 黑色格子
            }
            // 绘制边框
            RectangleShape border(Vector2f(CELL_SIZE, CELL_SIZE));
            border.setPosition(j * CELL_SIZE, i * CELL_SIZE);
            border.setFillColor(Color::Transparent); // 边框内部透明
            border.setOutlineThickness(1); // 设置边框厚度
            border.setOutlineColor(Color(128, 128, 128)); // 灰色边框
            window.draw(border);
            window.draw(rectangle);
        }
    }
    window.display();
}

void Game::run() {
    int rows = map.getRows();
    int cols = map.getCols();
    // 初始界面
    RenderWindow window(VideoMode(1000, 1000), "Board Map");
    draw(window);
    int week = 3;
    while (window.isOpen()) {
        Event event;
        while (window.pollEvent(event)) {
            if (event.type == Event::Closed)
                window.close();
            // 处理键盘事件
            if (event.type == Event::KeyPressed) {
                int rows = map.getRows();
                int cols = map.getCols();
                if (event.key.code == Keyboard::Space) {
                    map = map.gotoNext();
                }
            }
            circles ++;
            if (circles % week == 0) {
                // 检测上下左右 week 排如果有格子，则增加 map 大小
                bool needExpand = false;
                int rows = map.getRows();
                int cols = map.getCols();
                for (int i = 0; !needExpand && i < week; i++) {
                    for (int j = 0; j < cols; j++) {
                        if (map.getBoxValue(i, j)) {
                            needExpand = true;
                            std::cout << "(" << i << "," << j << ") boo! " << std::endl;
                        }
                    }
                }
                for (int i = rows - week; !needExpand && i < rows; i++) {
                    for (int j = 0; j < cols; j++) {
                        if (map.getBoxValue(i, j)) {
                            needExpand = true;
                            std::cout << "(" << i << "," << j << ") boo! " << std::endl;
                        }
                    }
                }
                for (int j = 0; !needExpand && j < week; j++) {
                    for (int i = 0; i < rows; i++) {
                        if (map.getBoxValue(i, j)) {
                            needExpand = true;
                            std::cout << "(" << i << "," << j << ") boo! " << std::endl;
                        }
                    }
                }
                for (int j = cols - week; !needExpand && j < cols; j++) {
                    for (int i = 0; i < rows; i++) {
                        if (map.getBoxValue(i, j)) {
                            needExpand = true;
                        }
                    }
                }
                if (needExpand) {
                    std::cout << "get a bigger size." << std::endl;
//                    std::cout << "==== BEFORE EXPAND ====" << std::endl;
//                    for (int i = 0; i < rows; ++i) {
//                        for (int j = 0; j < cols; ++j) {
//                            std::cout << map.getBoxValue(i, j) << " ";
//                        }
//                        std::cout << std::endl;
//                    }
                    map = map.expand(week);
//                    std::cout << "==== AFTER  EXPAND ====" << std::endl;
                    rows = map.getRows();
                    cols = map.getCols();
                    // 重新调整窗口大小
                    window.create(VideoMode(cols * CELL_SIZE, rows * CELL_SIZE), "Board Map");
                    window.setSize(Vector2u(cols * CELL_SIZE, rows * CELL_SIZE));
                    // 打印更新后的地图状态（示例）
//                    for (int i = 0; i < rows; ++i) {
//                        for (int j = 0; cols; ++j) {
//                            std::cout << map.getBoxValue(i, j) << " ";
//                        }
//                        std::cout << std::endl;
//                    }
//                    std::cout << "==== END EXPAND ====" << std::endl;
                }
            }
        }
        draw(window);  // 调用绘制函数
    }
}
```

核心代码：

```cpp
BoardMap gotoNext() {
    if (isEmpty()) {
        return *this;
    }
    BoardMap newMap = BoardMap(rows, cols);
    for (int i = 1; i < rows - 1; i++) {
        for (int j = 1; j < cols - 1; j++) {
            int aliveNeighbor = countAliveNeighbor(*this, i, j);
            bool isAlive = getBoxValue(i, j);
            // 核心规则
            if (aliveNeighbor > 0) {
//                    std::cout << "("<<i <<","<<j<<") owns " << aliveNeighbor << " alive neighbors."<< std::endl;
            }
            if (isAlive && (aliveNeighbor < 2 || aliveNeighbor > 3)) {
//                    std::cout << "("<<i <<","<<j<<") become death." << std::endl;
                isAlive = false;
            } else if (!isAlive && aliveNeighbor == 3) {
//                    std::cout << "("<<i <<","<<j<<") become alive." << std::endl;
                isAlive = true;
            } else if (isAlive) {
//                    std::cout << "("<<i <<","<<j<<") keep alive." << std::endl;
            }
            newMap.setBoxValue(i, j, isAlive);
        }
    }
    return newMap;
}
```

## 六种结构体分类说明

1. **静止生命**
   不随时间变化的模式称为静止模式或静物模式。
   例如：蜂窝、灯笼、船只和浴缸。

2. **振荡器**
   振荡器是随着时间循环的模式。
   例如：闪光灯和齿轮。

3. **飞船**
   飞船是自我复制并移动的模式。
   例如：滑翔机和轻型飞船。

4. **枪**
   枪是能不断产生新的物体的模式。
   例如：Gosper 滑翔枪。

5. **不属于上述 1\~4 的全部**
   在经过有限个周期后从无序对象化作 1\~4 中的任意对象
   例如：模糊泡沫和伪泡沫。

6. **无规则图案**
   一些图案不属于上述任何一种类别。

   ![Gosper Glider Gun](https://i.imgur.com/fN7NR1R.png)

## 总结

康威的生命游戏以其简单的规则和复杂的行为而闻名。它展示了简单系统如何通过局部交互产生复杂的动态模式。本文提供了一些代码示例，并说明了康威生命游戏中的六种常见结构体。希望这篇文章能让你对康威生命游戏有更深入的了解，并激发你对细胞自动机和复杂系统的兴趣。
