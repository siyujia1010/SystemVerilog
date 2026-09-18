# SV 学习笔记

> 基于《Introduction to SystemVerilog》(Ashok B. Mehta)、《Cracking Digital VLSI Verification Interview》、UVM 相关资料，以及个人手写笔记（dv笔记）整理。
> 学习方式：先讲透概念 → 用例题/案例检验理解 → 记录到这里，方便复习和面试前突击。
> 按主题分章节存放，每个章节一个文件夹，方便后续持续添加新内容。

## 目录

### 第一章：功能覆盖率（Functional Coverage）

- [01-functional-coverage/README.md](01-functional-coverage/README.md) —— covergroup / coverpoint / bins / cross

### 第二章：断言（SystemVerilog Assertions）

| 文件 | 内容 |
|---|---|
| [02-assertions/01-basics.md](02-assertions/01-basics.md) | 断言基础：immediate vs concurrent、property/sequence、`\|->` vs `\|=>`、采样值函数 `$rose/$fell/$stable/$past` |
| [02-assertions/02-operators-cheatsheet.md](02-assertions/02-operators-cheatsheet.md) | SVA 操作符速查表（and/or/intersect/not/throughout/within/first_match/重复操作符等） |
| [02-assertions/03-req-ack-case-study.md](02-assertions/03-req-ack-case-study.md) | 综合案例：req/ack 握手协议断言（7 条规格、时序图、past_req 影子寄存器、local variable ID 匹配进阶版） |
| [02-assertions/04-fork-and-process-control.md](02-assertions/04-fork-and-process-control.md) | fork / join / join_any / join_none / wait fork 进程控制，及 event/semaphore/mailbox 速记 |

### 后续章节（占位，陆续补充）

- 第三章：类的封装与继承
- 第四章：数组（packed / unpacked / dynamic / associative / queue）

## 学习进度

- [x] 第一章：功能覆盖率（covergroup/coverpoint/bins 数组 vs 单 bin/cross）—— 已完成，自测题全部答对
- [x] 第二章：断言基础（14.9 并发断言基础、14.14 重叠 vs 非重叠蕴含、14.17 采样值函数）—— 已完成
- [x] 第二章：req/ack 握手协议综合案例（7 条规格 + past_req 影子寄存器 + local variable ID 匹配）—— 已完成
- [x] 第二章：fork/join 进程控制综合例题（join/join_any/join_none/wait fork）—— 已完成
- [ ] 后续：功能覆盖率与断言的联合追踪（coverage-driven verification 收尾）
- [ ] 第三章：类的封装与继承
- [ ] 第四章：数组

## 参考资料

- *Introduction to SystemVerilog*, Ashok B. Mehta —— 第 14 章 SystemVerilog Assertions，第 15 章 Functional Coverage
- *Cracking Digital VLSI Verification Interview*, Ramdas Mozhikunnath & Robin Garg —— 覆盖率与断言相关面试题
- 个人手写笔记 (dv笔记)
