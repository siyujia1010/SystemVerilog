# SystemVerilog Assertions（SVA）基础

## 1. Immediate Assertion vs Concurrent Assertion

| | Immediate Assertion | Concurrent Assertion |
|---|---|---|
| 语法 | `assert (expr);` | `assert property (prop);` |
| 求值方式 | 像一条过程语句，立即（组合逻辑式）求值一次 | 基于时钟采样，可以跨越多个时钟周期，描述"时序关系" |
| 位置 | 只能出现在**过程块**内（initial/always/task/function 等） | 可以出现在 module/interface/program/checker/generate block（**不能在 class 里**，但 property/sequence 本身能在 class 外的这些地方声明） |
| 典型问题 | `module test(...); assert_1: assert(a && b); endmodule` —— 这是**错的**，immediate assertion 不能直接写在 module 顶层，必须放在过程块里 | 天然支持"多线程"求值（见下） |

> Mehta：14.8 Immediate Assertions；14.9 Concurrent Assertions: Basics。

## 2. property 与 sequence 的区别

- **sequence**：只是描述信号之间的"时序/组合关系"，本身不含"如果...那么..."的蕴含逻辑，**不能用 `|->` / `|=>`**。可以有形参，可以被实例化、引用。可以声明在 module/interface/program/clocking block/package/checker/generate block（不能在 class 里）。
- **property**：在 sequence 基础上加上蕴含关系（antecedent |-> consequent），本身也不会自动触发，必须被 `assert` / `cover` / `assume`。同样不能声明在 class 里；formal argument 甚至可以是"property 类型"（把一个 property 当参数传给另一个 property）。

```systemverilog
sequence seq2(a, b, c);
  a ##1 b ##1 c;
endsequence

property name;
  @(posedge clk) disable iff (condition) A |-> ##0 B;
endproperty

assert property (name);
```

> Mehta：14.16 Difference Between "sequence" and "property"。

## 3. `|->` vs `|=>`：重叠 vs 非重叠蕴含

蕴含操作符左边叫 **antecedent（前提）**，右边叫 **consequent（结论）**：前提匹配成功才会去检查结论。

- **重叠蕴含 `|->`**：前提序列匹配成功的**那一拍**，就是开始检查结论的那一拍（同一时钟沿）。
  ```systemverilog
  assert property (@(posedge clk) (a==1) |-> b ##1 c);
  // a==1 命中的当拍 b 就要为真，再下一拍 c 为真，才算通过
  ```
- **非重叠蕴含 `|=>`**：前提匹配成功后，**下一拍**才开始检查结论。
  ```systemverilog
  assert property (@(posedge clk) (a==1) |=> b ##1 c);
  // a==1 命中后，下一拍 b 为真，再下一拍 c 为真
  ```

**等价关系**：在非重叠蕴含前面显式加一拍延迟，就能等价于重叠蕴含往后挪一拍：

```systemverilog
1) req |=> ##2 $rose(ack);
2) req |-> ##3 $rose(ack);
// 1) 和 2) 是等价的：|=> 天然带 1 拍延迟，再加 ##2，正好等于 |-> 后面接 ##3
```

**嵌套蕴含（nested implication）** 也是合法的：
```systemverilog
a |=> b |=> c;
// a 为真 -> 下一拍看 b，b 为真 -> 再下一拍看 c，c 为真才算 pass
```

> 记忆技巧：`->` 一根箭头 = "当拍就看"，`=>` 两道杠 = "隔一拍再看"。

> Mehta：14.9.1 Implication Operator；嵌套蕴含见 14.28 Nested Implications。

## 4. 采样值函数：`$rose` / `$fell` / `$stable` / `$past`

这几个函数都是**边沿相关**的，比较的是"当前采样时刻"和"上一采样时刻"的值，**不是** Verilog 里的 `posedge`（`$rose` 不会在信号变化的瞬间触发，只在下一次时钟采样点才会被判定为真）。

- **`$rose(expr)`**：表达式最低位在上一个时钟沿采样值为 0，当前时钟沿采样值为 1。
- **`$fell(expr)`**：相反，上一拍是 1，当前拍是 0。
- **`$stable(expr)`**：当前拍采样值与上一拍采样值相同即为真。
- **`$past(expr [, N])`**：取表达式 N 拍之前的值（默认 N=1）。仿真开始时如果往前数的时钟数不够，会用该变量**声明时的初始值**（不是 initial block 里赋的值）。

### 为什么优先用边沿相关函数而不是电平？

如果写成：
```systemverilog
property checkiack;
  @(posedge clk) intr |=> iack;   // 电平触发：只要 intr 为高，每拍都会开一个新线程
endproperty
```
只要 `intr` 一直是高电平，每一拍都会 fork 一个新的检查线程，**严重影响仿真性能**。更好的写法是用边沿函数只在跳变那一刻触发一次：
```systemverilog
property checkiack;
  @(posedge clk) $rose(intr) |=> $rose(iack);
endproperty
```

> Mehta：14.17 Sampled Value Functions（14.17.1 `$rose`、14.17.2 `$fell`、14.17.4 `$stable`、14.17.5 `$past`）；为什么用边沿函数见 14.17.3、多线程见 14.11 Concurrent Assertions Are Multi-threaded。

## 5. 常用系统函数

```systemverilog
$isunknown(sig)      // 是否含有 X/Z，可用来检查"信号不该是 X"
$isonehot(state)     // 是否恰好一位为 1（独热码检查）
$onehot0(state)      // 最多一位为 1（含全 0 合法）
$countones(sig)      // 统计 1 的个数
```

例：
```systemverilog
assert property (@(posedge clk) !$isunknown(mysignal));
assert property (@(posedge clk) $isonehot(state));
assert property (@(posedge clk) $countones(grant[5:0]) == 1);   // 5 位 grant 任意时刻只能有 1 位为 1
```

请求-授权时序例子：
```systemverilog
property p_req_grant;
  @(posedge clk) $rose(req) |-> ##[2:5] $rose(gnt);
endproperty
// req 上升沿之后 2~5 拍内 gnt 必须出现上升沿
```

> Mehta：14.19 System Functions and Tasks（14.19.1 `$onehot`/`$onehot0`、14.19.2 `$isunknown`、14.19.3 `$countones`）。

## 6. `disable iff`：复位期间关闭断言

```systemverilog
assert property (@(posedge clk) disable iff (reset) a |=> b);
```
复位有效期间整条断言直接跳过求值，避免复位过程中的瞬态值触发误报。

> Mehta：14.13 Disable (Property) Operator: disable iff。

## 7. `accept_on` / `reject_on`：带中止条件的属性

```systemverilog
property reqack;
  @(posedge clk) accept_on(cycle_end) req |-> ##5 ack;
endproperty
assert property (reqack);
```
- `req` 为真后开始等 `ack`（最多等 5 拍）。
- 如果在等待期间 `cycle_end` 先出现 —— `accept_on` 直接让本次求值**提前通过（pass）**。
- 如果 `cycle_end` 一直没出现 —— 按正常规则判断 `ack` 是否按时到达。
- 如果 `cycle_end` 和 `ack` 同一拍出现 —— **accept_on 优先**，判定为 pass。
- `reject_on` 逻辑相反：中止条件出现时直接判为 **fail**。
- 多个中止算子嵌套时的词法顺序（从左到右）固定为：`accept_on`、`reject_on`、`sync_accept_on`、`sync_reject_on`。

> Mehta：14.25 Abort Properties: reject_on, accept_on, sync_reject_on, sync_accept_on。

## 8. 自测要点

1. `req |=> ##2 $rose(ack)` 和 `req |-> ##3 $rose(ack)` 是否等价？为什么？
2. 为什么用电平触发（如直接写 `intr |=> iack`）的断言在仿真性能上是个隐患？应该怎么改？
3. `$past(b)` 在仿真刚开始、还没有足够历史时钟时，取的是什么值？
4. `disable iff` 和 `accept_on` 的本质区别是什么（一个是完全跳过，一个是提前判定 pass/fail）？
