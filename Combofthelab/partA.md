# Cache Lab - Part A: 缓存模拟器 (csim) 设计文档

本文档概述了 `csim` 程序的设计逻辑与工作流程。`csim` 是一个用于模拟带有 LRU (最近最少使用) 替换策略的缓存系统的程序。

## 🧠 核心逻辑流程 (思维导图)

### Ⅰ. 初始化阶段 (Initialization Phase)

1.  **解析命令行参数 (Parse Command-line Arguments)**
    *   通过遍历 `argv` (或使用 `getopt`) 解析出 `-s`, `-E`, `-b` (缓存配置参数) 和 `-t` (轨迹文件路径)。
    *   将这些值存入本地变量 (`s`, `E`, `b`, `trace_path`)。

2.  **构建缓存数据结构 (Build Cache Data Structure)**
    *   根据 `s` 计算组的数量: `S = 1 << s`。
    *   使用 `malloc` 动态分配内存以创建 `Cache` 结构。
        *   **第一层**: 分配一个包含 `S` 个 `CacheSet` 元素的数组。
        *   **第二层**: 循环 `S` 次，为每个 `CacheSet` 分配一个包含 `E` 个 `CacheLine` 元素的数组。
    *   初始化所有 `CacheLine` 的成员为 0:
        *   `valid = 0`
        *   `tag = 0`
        *   `lru_counter = 0`

3.  **准备模拟 (Prepare for Simulation)**
    *   使用 `fopen` 打开轨迹文件，并处理可能发生的错误。
    *   初始化模拟所需的计数器为 0:
        *   `hit_count = 0` (命中次数)
        *   `miss_count = 0` (缺失次数)
        *   `eviction_count = 0` (替换次数)
    *   为 LRU 策略初始化一个全局时间戳: `time = 0`。

---

### Ⅱ. 模拟循环阶段 (Simulation Loop Phase)

*   使用 `while (fscanf(...) == 3)` 循环从轨迹文件中逐行读取 `operation` (操作), `address` (地址), `size` (大小)。
*   **在循环的每一次迭代中:**

    1.  **时间戳推进与指令过滤 (Timestamp & Filter)**
        *   全局时间戳自增: `time++`。
        *   如果 `operation` 是 `'I'` (指令读取)，则调用 `continue` 跳过本次循环。

    2.  **地址剖析 (Address Decomposition)**
        *   使用位运算 (`>>` 和 `&` 配合掩码) 从 `address` 中提取出 `set_index` (组索引)。
        *   使用位运算 (`>>`) 从 `address` 中提取出 `tag` (标记)。

    3.  **核心访存模拟 (Cache Access Simulation)**
        *   **定位组 (Locate the Set)**: 使用 `set_index` 获取指向 `target_set` (目标组) 的指针。
        *   **查找命中 (Search for a Hit)**:
            *   初始化一个标志位: `is_hit = 0`。
            *   通过 `for` 循环遍历 `target_set` 中的所有 `E` 个 `Line` (行)。
            *   **如果找到一个 `Line` 满足 `valid == 1` 并且 `tag` 匹配**:
                *   **判定为命中 (HIT)!**
                *   `hit_count` 自增。
                *   更新该 `Line` 的新鲜度: `line->lru_counter = time`。
                *   设置标志位: `is_hit = 1`。
                *   使用 `break` 跳出查找循环。

    4.  **处理缺失 (Handle a Miss, 如果 `is_hit` 仍为 `0`)**:
        *   **判定为缺失 (MISS)!**
        *   `miss_count` 自增。
        *   **寻找空闲行**:
            *   初始化标志位: `found_empty_line = 0`。
            *   再次 `for` 循环遍历 `target_set` 中的所有 `Line`。
            *   **如果找到一个 `Line` 满足 `valid == 0`**:
                *   将新数据放入该行 (更新 `valid = 1`, `tag` 和 `lru_counter = time`)。
                *   设置标志位: `found_empty_line = 1`。
                *   使用 `break` 跳出此循环。
        *   **处理替换 (Handle an Eviction, 如果 `found_empty_line` 仍为 `0`)**:
            *   **判定为替换 (EVICTION)!**
            *   `eviction_count` 自增。
            *   **寻找牺牲者**: 第三次 `for` 循环遍历 `target_set`，找到 `lru_counter` 值**最小**的那个 `Line`。
            *   **替换牺牲者**: 用新数据的 `tag` 和 `lru_counter = time` 覆盖牺牲者 `Line` 的相应成员。

    5.  **处理修改操作 ('M' - Modify)**:
        *   'M' 操作等同于一次 `Load` (加载) + 一次 `Store` (存储)。
        *   上述逻辑已经完整模拟了 `Load` 的过程。
        *   后续的 `Store` 操作必然是**命中**。
        *   **因此，如果 `operation` 是 `'M'`，则无条件地将 `hit_count` 再加一。**

---

### Ⅲ. 收尾阶段 (Finalization Phase)

1.  **清理 (Cleanup)**
    *   `while` 循环结束后，使用 `fclose` 关闭轨迹文件。
    *   (推荐) 按照与 `malloc` 相反的顺序，使用 `free` 释放所有动态分配的内存，以防止内存泄漏。

2.  **报告结果 (Report Results)**
    *   调用 `printSummary(hit_count, miss_count, eviction_count)` 输出最终的计分结果。
    <br>
  <br>
  <br> 
  
  ---

# Cache Lab - Part A: Cache Simulator (csim) Design

This document outlines the logic and workflow of the `csim` program, a simulator for a cache memory system with an LRU (Least Recently Used) replacement policy.

## 🧠 Core Logic Flow (Mind Map)

### Ⅰ. Initialization Phase

1.  **Parse Command-line Arguments**
    *   Read `-s`, `-E`, `-b` (cache parameters) and `-t` (trace file path) from `argv`.
    *   Store them in local variables (`s`, `E`, `b`, `trace_path`).

2.  **Build Cache Data Structure**
    *   Calculate the number of sets: `S = 1 << s`.
    *   Dynamically allocate memory for the `Cache` structure using `malloc`.
        *   **Level 1**: Allocate an array of `S` `CacheSet`s.
        *   **Level 2**: Loop `S` times, and for each `CacheSet`, allocate an array of `E` `CacheLine`s.
    *   Initialize all `CacheLine` members to zero:
        *   `valid = 0`
        *   `tag = 0`
        *   `lru_counter = 0`

3.  **Prepare for Simulation**
    *   Open the trace file using `fopen` and handle potential errors.
    *   Initialize simulation counters to zero:
        *   `hit_count = 0`
        *   `miss_count = 0`
        *   `eviction_count = 0`
    *   Initialize a global timestamp for LRU: `time = 0`.

---

### Ⅱ. Simulation Loop Phase

*   Use a `while (fscanf(...) == 3)` loop to read each line (`operation`, `address`, `size`) from the trace file.
*   **For each line processed:**

    1.  **Timestamp & Filter**
        *   Increment the global timestamp: `time++`.
        *   If `operation` is `'I'` (Instruction access), `continue` to the next iteration.

    2.  **Address Decomposition**
        *   Calculate `set_index` from the `address` using bitwise operations (`>>` and `&` with a mask).
        *   Calculate `tag` from the `address` using a bitwise right shift (`>>`).

    3.  **Cache Access Simulation**
        *   **Locate the Set**: Use `set_index` to get a pointer to the `target_set`.
        *   **Search for a Hit**:
            *   Initialize a flag: `is_hit = 0`.
            *   Loop through all `E` lines in the `target_set`.
            *   **If a `line` is found where `valid == 1` and `tag` matches**:
                *   **It's a HIT!**
                *   Increment `hit_count`.
                *   Update the line's freshness: `line->lru_counter = time`.
                *   Set flag: `is_hit = 1`.
                *   `break` out of the search loop.

    4.  **Handle a Miss (if `is_hit` is still `0`)**:
        *   **It's a MISS!**
        *   Increment `miss_count`.
        *   **Look for an empty line**:
            *   Initialize a flag: `found_empty_line = 0`.
            *   Loop through all `E` lines in the `target_set` again.
            *   **If a `line` is found where `valid == 0`**:
                *   Place new data: update `valid = 1`, `tag`, and `lru_counter = time`.
                *   Set flag: `found_empty_line = 1`.
                *   `break` out of this loop.
        *   **Handle an Eviction (if `found_empty_line` is still `0`)**:
            *   **It's an EVICTION!**
            *   Increment `eviction_count`.
            *   **Find the victim**: Loop through all `E` lines a third time to find the one with the minimum `lru_counter`.
            *   **Replace the victim**: Overwrite the victim line's `tag` and `lru_counter = time`.

    5.  **Handle Modify Operation ('M')**:
        *   The 'M' operation is a `Load` followed by a `Store`.
        *   The `Load` part is simulated by the logic above.
        *   The subsequent `Store` is always a **hit**.
        *   **Therefore, if `operation` is `'M'`, unconditionally increment `hit_count` one more time.**

---

### Ⅲ. Finalization Phase

1.  **Cleanup**
    *   After the `while` loop finishes, close the trace file using `fclose`.
    *   (Recommended) Free all dynamically allocated memory in the reverse order of allocation to prevent memory leaks.

2.  **Report Results**
    *   Call `printSummary(hit_count, miss_count, eviction_count)` to output the final score.
