# SV 学习笔记

> 基于《Introduction to SystemVerilog》(Ashok B. Mehta)、《Cracking Digital VLSI Verification Interview》、UVM 相关资料，以及个人手写笔记（dv笔记）整理。
> 学习方式：先讲透概念 → 用例题/案例检验理解 → 记录到这里，方便复习和面试前突击。
> 按主题分章节存放，每个章节一个文件夹，方便后续持续添加新内容。

## 目录

### 第一章：数据类型（string / enum / struct / 数组）

| 文件 | 内容 |
|---|---|
| [01-data-types/01-string.md](01-data-types/01-string.md) | string 类型：拼接/重复、len/putc/getc/atoi/substr |
| [01-data-types/02-enum.md](01-data-types/02-enum.md) | enum 枚举：整数互转、first/last/next/prev/num 遍历 |
| [01-data-types/03-struct.md](01-data-types/03-struct.md) | struct 结构体：packed vs unpacked、赋值模式 `'{...}` 与 default |
| [01-data-types/04-arrays.md](01-data-types/04-arrays.md) | packed / unpacked / dynamic / associative / queue 数组，及流操作符 `{>>{}}` / `{<<{}}` |
| [01-data-types/05-streaming-operator.md](01-data-types/05-streaming-operator.md) | 流操作符深入：`{<<n{...}}` 切片粒度、nibble swap / byte swap（含配图，04-arrays §5 的深入版） |

### 第二章：进程与同步（task 生命周期 / 事件 / 信号量）

> 与第五章 [fork / join 与进程控制](05-assertions/04-fork-and-process-control.md) 是外层/内层关系：fork/join 管"线程怎么启动和收拢"，本章管"线程之间共享什么、怎么同步"。

| 文件 | 内容 |
|---|---|
| [02-process-sync/01-tasks-process-sync.md](02-process-sync/01-tasks-process-sync.md) | `automatic` vs `static` 并发竞态、`$display/$strobe/$monitor`、`@` vs `wait(.triggered)` 竞态、semaphore |

### 第三章：随机约束（Constrained Random）

| 文件 | 内容 |
|---|---|
| [03-constrained-random/01-randomize-and-constraints.md](03-constrained-random/01-randomize-and-constraints.md) | `$urandom_range`、`randomize() with` 与 `local::`、`rand_mode` / `constraint_mode`、soft、`dist`（`:=` vs `:/`）、数组归约约束、`solve...before` 概率偏置（含概率分布图） |

### 第四章：功能覆盖率（Functional Coverage）

- [04-functional-coverage/README.md](04-functional-coverage/README.md) —— covergroup / coverpoint / bins / cross

### 第五章：断言（SystemVerilog Assertions）

| 文件 | 内容 |
|---|---|
| [05-assertions/01-basics.md](05-assertions/01-basics.md) | 断言基础：immediate vs concurrent、property/sequence、`\|->` vs `\|=>`、采样值函数 `$rose/$fell/$stable/$past` |
| [05-assertions/02-operators-cheatsheet.md](05-assertions/02-operators-cheatsheet.md) | SVA 操作符速查表（and/or/intersect/not/throughout/within/first_match/重复操作符等） |
| [05-assertions/03-req-ack-case-study.md](05-assertions/03-req-ack-case-study.md) | 综合案例：req/ack 握手协议断言（7 条规格、时序图、past_req 影子寄存器、local variable ID 匹配进阶版） |
| [05-assertions/04-fork-and-process-control.md](05-assertions/04-fork-and-process-control.md) | fork / join / join_any / join_none / wait fork 进程控制，及 event/semaphore/mailbox 速记 |

### 后续章节（占位，陆续补充）

- 第六章：类的封装与继承

## 学习进度

- [x] 第一章：数据类型（string / enum / struct / 数组）—— 已完成
- [x] 第一章：流操作符深入（nibble swap / byte swap）—— 已完成
- [x] 第二章：进程与同步（automatic/static 竞态、@ vs wait、semaphore）—— 已完成
- [x] 第三章：随机约束（inline 约束名字解析、soft、dist、solve...before）—— 已完成
- [x] 第四章：功能覆盖率（covergroup/coverpoint/bins 数组 vs 单 bin/cross）—— 已完成，自测题全部答对
- [x] 第五章：断言基础（14.9 并发断言基础、14.14 重叠 vs 非重叠蕴含、14.17 采样值函数）—— 已完成
- [x] 第五章：req/ack 握手协议综合案例（7 条规格 + past_req 影子寄存器 + local variable ID 匹配）—— 已完成
- [x] 第五章：fork/join 进程控制综合例题（join/join_any/join_none/wait fork）—— 已完成
- [ ] 后续：功能覆盖率与断言的联合追踪（coverage-driven verification 收尾）
- [ ] 第六章：类的封装与继承

## 参考资料

- *Introduction to SystemVerilog*, Ashok B. Mehta —— 第 14 章 SystemVerilog Assertions，第 15 章 Functional Coverage
- *Cracking Digital VLSI Verification Interview*, Ramdas Mozhikunnath & Robin Garg —— 覆盖率与断言相关面试题
- 路科验证 V0 课程 —— 第 2 讲《任务和函数》、第 3 讲《随机约束》《进程间同步和通信》（第二、三章笔记末尾附有与课程的对应关系表）
- 个人手写笔记 (dv笔记)
