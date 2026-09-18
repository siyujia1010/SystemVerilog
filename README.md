# SV 功能覆盖率与断言（SVA）学习笔记

> 基于《Introduction to SystemVerilog》(Ashok B. Mehta) 第 14-15 章、《Cracking Digital VLSI Verification Interview》相关章节，以及个人手写笔记（dv笔记）整理。
> 学习方式：先讲透概念 → 用例题/案例检验理解 → 记录到这里，方便复习和面试前突击。

## 目录

| 文件 | 内容 |
|---|---|
| [01-functional-coverage.md](01-functional-coverage.md) | 功能覆盖率：covergroup / coverpoint / bins / cross |
| [02-assertions-basics.md](02-assertions-basics.md) | 断言基础：immediate vs concurrent、property/sequence、`|->` vs `|=>`、采样值函数 `$rose/$fell/$stable/$past` |
| [03-assertion-operators-cheatsheet.md](03-assertion-operators-cheatsheet.md) | SVA 操作符速查表（and/or/intersect/not/throughout/within/first_match/重复操作符等） |
| [04-req-ack-case-study.md](04-req-ack-case-study.md) | 综合案例：req/ack 握手协议断言（含 intersect、影子寄存器技巧、local variable ID 匹配进阶版） |

## 学习进度

- [x] 第一课：功能覆盖率（covergroup/coverpoint/bins 数组 vs 单 bin/cross）—— 已完成，自测题全部答对
- [x] 第二课：断言基础（14.9 并发断言基础、14.14 重叠 vs 非重叠蕴含、14.17 采样值函数）—— 已完成
- [x] 综合案例：req/ack 握手协议断言（intersect / not / 影子寄存器技巧 + 带 ID 匹配的进阶版）—— 已完成
- [ ] 后续：功能覆盖率与断言的联合追踪（coverage-driven verification 收尾）

## 参考资料

- *Introduction to SystemVerilog*, Ashok B. Mehta —— 第 14 章 SystemVerilog Assertions，第 15 章 Functional Coverage
- *Cracking Digital VLSI Verification Interview*, Ramdas Mozhikunnath & Robin Garg —— 覆盖率与断言相关面试题
- 个人手写笔记 (dv笔记)
