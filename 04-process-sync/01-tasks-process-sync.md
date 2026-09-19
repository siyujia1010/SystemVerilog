# 任务生命周期 与 进程间同步

## 1. `automatic` vs `static` 任务 —— 并发竞态经典案例

```systemverilog
initial begin
  fork
    #10 run_ID(1, 50);
    #20 run_ID(2, 0);
  join
end

task automatic run_ID (int ID, int t);
  #t;
  $display("%d", ID);
endtask
//automatic : 2, 1
//static    : 2, 2
```

时间线：
- 分支A：时刻10调用 `run_ID(1, 50)` → 内部 `#50` 延迟 → 时刻60执行 `$display`
- 分支B：时刻20调用 `run_ID(2, 0)` → 内部 `#0` 延迟（几乎立即）→ 时刻20执行 `$display`

**如果是 `automatic`**：每次调用都各自拥有独立的一份 `ID`、`t` 存储空间，两次调用互不干扰。按实际完成时间排序：分支B先在时刻20打印`2`，分支A后在时刻60打印`1`。输出顺序：**2, 1**。

**如果是 `static`**（默认情况，且被并发重入调用时）：`ID`、`t` 是全局唯一的一份存储，被所有调用共享。分支A在时刻10把共享的`ID`设为1，进入50时长的等待（`#t`延迟量在语句执行的那一刻求值一次并锁定，不会再变）；但在它还没醒来之前，时刻20分支B把同一个共享的`ID`覆盖成了2。等到时刻60分支A的延迟结束、真正执行`$display("%d", ID)`时，读到的`ID`已经是被分支B改写过的**2**，不是它自己原本的1。所以两次打印都是**2, 2**——这是共享静态局部变量在并发场景下"互相踩踏"的经典bug，也是为什么验证代码里几乎所有task都要显式加`automatic`。

> V0课程未讲这个具体例子，笔记/Mehta补充内容。

## 2. 打印类系统任务

```systemverilog
$display: 打印当前值
$strobe: 打印当前时间step结束时的值（这里的step与`timescale的声明有关）
$monitor: 假如任何值发生更改，则在当前时间步的末尾打印值
          同时$monitor只能调用一次，顺序调用将覆盖前一个
```

- `$display`：打印**当前时刻**的值，语句执行时立即打印。
- `$strobe`：打印当前时间步（time step）**结束时**的值——保证拿到这个时间步里最终稳定的值，而不是中间态。
- `$monitor`：只要监控的信号发生任何变化，就在当前时间步末尾自动打印一次。**全局只能生效一个**，后调用的会覆盖前一个（不是叠加）。

> V0课程未系统讲解这三者对比，笔记/Mehta补充内容。

## 3. 事件阻塞：`@` vs `wait(event.triggered)`

```systemverilog
event e;
-> e;               // 触发事件
@e;                 // 阻塞直到事件被触发（边沿触发型）
wait(e.triggered);  // 阻塞直到事件被触发（电平/状态型）
wait_order(e1, e2); // 要求必须先等到e1触发，再等到e2触发，顺序不对会报错
```

这是SV里一个隐蔽的**竞争(race)问题**：

- `@e` 是**边沿检测**：只关心"触发"这个动作本身。如果触发发生的那一刻，线程还没执行到`@e`这一句（哪怕只差一个仿真步的调度顺序），这次触发就相当于没发生，线程会继续阻塞，得等下一次`->e`。
- `wait(e.triggered)` 是**状态检测**：`e.triggered`是一个在当前时间步内会保持为真的状态标志，不是转瞬即逝的动作。无论检查语句在触发之前还是之后被调度，只要在同一个时间步内，查到的都是"已触发"，会立刻结束阻塞。

如果`->e`和某个线程执行到`@e`/`wait(e.triggered)`恰好被调度在同一个仿真时刻，谁先谁后取决于仿真器内部事件队列调度顺序，用户无法控制。这也是为什么很多验证代码更推荐用`wait(event.triggered)`而不是裸`@event`来做跨线程同步。

> `event`基础语法、`wait_order`在V0课程第3讲《进程间同步和通信》有讲，但`@`与`wait(.triggered)`这个竞态差异细节V0没有展开，笔记/Mehta补充。

## 4. 信号量 semaphore

```systemverilog
semaphore key;
key = new(1);        // 创建一个信号量，初始有1把"钥匙"（可用资源数为1）
key.get(1);           // 阻塞式获取1把钥匙，不够就一直等
key.put(1);           // 归还1把钥匙
key.try_get(1);       // 非阻塞式尝试获取，成功返回1、失败返回0，不会卡住
```

用于验证环境里限制**同一时间只允许N个线程访问某资源**（比如共享总线），`get`/`put`是最常用的一对，`try_get`适合"能拿就拿，拿不到就跳过"的场景。

> V0课程第3讲《进程间同步和通信》"旗语（semaphore）"一致覆盖。

## 5. 路科V0课程对应关系

| 本笔记内容 | V0课程对应位置 | 备注 |
|---|---|---|
| `[task/function] static`等任务/函数生命周期基础语法 | 第2讲《任务和函数》概述部分 | V0只讲基础语法 |
| `automatic`/`static`并发竞态案例 | V0未讲 | 笔记/Mehta补充，本笔记唯一无V0出处的重点 |
| `$display`/`$strobe`/`$monitor`对比 | V0未讲 | 笔记/Mehta补充 |
| `event`基础语法（`->`/`@`/`wait(.triggered)`） | 第3讲《进程间同步和通信》"事件event" | 一致 |
| `@`与`wait(.triggered)`竞态差异 | V0未明确展开 | 笔记/Mehta补充 |
| `wait_order(e1,e2)` | 第3讲《进程间同步和通信》"wait_order()" | 一致 |
| `semaphore`（`new/get/put/try_get`） | 第3讲《进程间同步和通信》"旗语（semaphore）" | 一致 |

## 6. 相关笔记

- **外层**：[fork / join 与进程控制](../02-assertions/04-fork-and-process-control.md) —— 讲怎么**启动和收拢**线程（`join` / `join_any` / `join_none` / `wait fork`），并在 §3 给了 event/semaphore 的速记。
- **本文（内层）**：线程启动之后会遇到的问题 —— task 存储是否共享（`automatic` / `static`）、事件竞态（`@` vs `wait(.triggered)`，对速记里的结论做了展开）、信号量互斥。
