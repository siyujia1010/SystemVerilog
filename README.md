# SV 学习笔记

> 基于《Introduction to SystemVerilog》(Ashok B. Mehta)、《Cracking Digital VLSI Verification Interview》、UVM 相关资料，以及个人手写笔记（dv笔记）整理。
> 学习方式：先讲透概念 → 用例题/案例检验理解 → 记录到这里，方便复习和面试前突击。
> 按主题分章节存放，每个章节一个文件夹，方便后续持续添加新内容；所有图片统一放在 [assets/](assets/)。
> 各章知识点下方用 `> Mehta：x.y 小节名` 标注了对应的《Introduction to SystemVerilog》小节号，方便回书查阅。

## 目录

### 第一章：数据类型（string / enum / struct / 数组）

| 文件 | 内容 |
|---|---|
| [01-data-types/01-string-enum-struct.md](01-data-types/01-string-enum-struct.md) | string（拼接/重复、len/putc/getc/atoi/substr）；enum（整数互转、first/last/next/prev/num 遍历）；struct（packed vs unpacked、赋值模式 `'{...}` 与 default） |
| [01-data-types/02.arrays_streaming.md](01-data-types/02.arrays_streaming.md) | packed / unpacked / dynamic / associative / queue 数组；流操作符 `{>>{}}` / `{<<{}}`（数组打包/解包基础，及 32 位 nibble swap / byte swap 深入，含配图） |

### 第二章：随机约束（Constrained Random）

| 文件 | 内容 |
|---|---|
| [02-constrained-random/01-randomize-and-constraints.md](02-constrained-random/01-randomize-and-constraints.md) | `$urandom_range`、`randomize() with` 与 `local::`（含变量归属图）、`rand_mode` / `constraint_mode`、soft、`dist`（`:=` vs `:/`）、数组归约约束、`solve...before` 概率偏置（含概率分布图） |

### 第三章：进程控制与同步（fork/join / task 生命周期 / 事件 / 信号量）

| 文件 | 内容 |
|---|---|
| [03-process-sync/01-process-control-and-sync.md](03-process-sync/01-process-control-and-sync.md) | fork / join / join_any / join_none / wait fork 及综合例题；`automatic` vs `static` 并发竞态（含示意图）；`$display/$strobe/$monitor`（含时间步示意图）；`@` vs `wait(.triggered)` 竞态；semaphore；mailbox |

### 第四章：功能覆盖率（Functional Coverage）

- [04-functional-coverage/README.md](04-functional-coverage/README.md) —— covergroup / coverpoint / bins / cross

### 第五章：断言（SystemVerilog Assertions）

| 文件 | 内容 |
|---|---|
| [05-assertions/01-basics.md](05-assertions/01-basics.md) | 断言基础：immediate vs concurrent、property/sequence、`\|->` vs `\|=>`、采样值函数 `$rose/$fell/$stable/$past` |
| [05-assertions/02-operators-cheatsheet.md](05-assertions/02-operators-cheatsheet.md) | SVA 操作符速查表（and/or/intersect/not/throughout/within/first_match/重复操作符等） |
| [05-assertions/03-req-ack-case-study.md](05-assertions/03-req-ack-case-study.md) | 综合案例：req/ack 握手协议断言（7 条规格、时序图、past_req 影子寄存器、local variable ID 匹配进阶版） |

### 后续章节（占位，陆续补充）

- 第六章：类的封装与继承

## 学习进度

- [x] 第一章：数据类型（string / enum / struct / 数组）—— 已完成
- [x] 第一章：流操作符深入（nibble swap / byte swap）—— 已完成
- [x] 第二章：随机约束（inline 约束名字解析、soft、dist、solve...before）—— 已完成
- [x] 第三章：进程控制与同步（fork/join 综合例题、automatic/static 竞态、@ vs wait、semaphore/mailbox）—— 已完成
- [x] 第四章：功能覆盖率（covergroup/coverpoint/bins 数组 vs 单 bin/cross）—— 已完成，自测题全部答对
- [x] 第五章：断言基础（14.9 并发断言基础、14.9.1 重叠 vs 非重叠蕴含、14.17 采样值函数）—— 已完成
- [x] 第五章：req/ack 握手协议综合案例（7 条规格 + past_req 影子寄存器 + local variable ID 匹配）—— 已完成
- [ ] 后续：功能覆盖率与断言的联合追踪（coverage-driven verification 收尾）
- [ ] 第六章：类的封装与继承

## 参考资料

- *Introduction to SystemVerilog*, Ashok B. Mehta —— 第 2/3/4/5 章 数据类型、数组、队列与结构体，第 12.13 节 流操作符，第 13 章 Constrained Random，第 14 章 SystemVerilog Assertions，第 15 章 Functional Coverage，第 16 章 SystemVerilog Processes，第 18 章 Semaphores and Mailboxes（第二、三章笔记末尾另附有完整的小节对应表）
- *Cracking Digital VLSI Verification Interview*, Ramdas Mozhikunnath & Robin Garg —— 覆盖率、断言与随机约束相关面试题（`solve...before` 第 238 题、`std::randomize` 第 261 题）
- 个人手写笔记 (dv笔记)
