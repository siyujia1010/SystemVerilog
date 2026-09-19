# SVA 操作符速查表

> 汇总自个人手写笔记 + 教材，按"多条件组合 / 序列长度 / 序列关系"三类整理，方便写断言时直接查。

## 1. 序列内的多条件组合（信号级）

在同一个时刻判断多个信号：`&&`、`||`、`!`

```systemverilog
@(posedge clk) A |-> ##[1:3] B;      // A 之后 1~3 拍内 B 为真
@(posedge clk) A[*1:2] |-> B;        // A 连续出现 1~2 次之后 B 为真
```

> Mehta：14.18.1 ##m: Clock Delay；14.18.2 ##[m:n]: Clock Delay Range。

## 2. 序列之间的组合（时序级）：`and` / `or` / `intersect` / `not`

在**序列（sequence）之间**做组合时，用的是 `and`、`or`、`intersect`、`not`（不是信号级的 `&&`/`||`）：

```systemverilog
@(posedge clk) A |-> B and C;
// B、C 同时开始；以两者中"较晚结束"的那个作为整体的结束点
// 如果 B、C 各自会重复触发多次，最终结果也会触发多次

@(posedge clk) A |-> B intersect C;
// B、C 必须同时开始、同时结束，才算匹配

@(posedge clk) A |-> B or C;
// B、C 同时开始，任意一个先结束就算匹配（谁先完成听谁的）

@(posedge clk) A |-> not(B);
// B 匹配成功则整个属性失败；B 不匹配则属性通过（否定逻辑，容易搞反，务必想清楚）
```

`not` 的经典应用——"两次 req 之间必须先出现一次 ack"（strictly one ack）：
```systemverilog
property strictlyOneAck;
  @(posedge clk) $rose(req) |-> not(!ack[*0:$] ##1 $rose(req));
endproperty
```
逻辑拆解：内层序列 `!ack[*0:$] ##1 $rose(req)` 表示"ack 一直不来，直到下一次 req 又来了"——这是我们**不希望**发生的情况，所以外面套一层 `not`，命中这种坏情况就 fail，没命中（即 ack 确实在下一次 req 之前来了）就 pass。

> Mehta：14.18.13 Seq1 and Seq2、14.18.15 Seq1 or Seq2、14.18.16 Seq1 "intersect" Seq2、14.18.20 not Operator。

## 3. `first_match`：只取第一次匹配

```systemverilog
@(posedge clk) first_match(A |-> B) |-> C;
// 只能用在蕴含操作符左边（antecedent 位置），取多个可能匹配里的第一个
```

> Mehta：14.18.18 first_match（14.18.19 Application）。

## 4. `throughout` / `within`：谁包住谁

```systemverilog
@(posedge clk) $rose(A) |=> (B[*5] within !C[->1]);
// B 连续 5 拍必须发生在 "!C 第一次出现之前" 这段区间内 —— C 比 B 更"长"（外层）

@(posedge clk) $rose(A) |=> (B throughout C[*5]);
// B 必须在 C 连续出现 5 次的整个过程中持续为真 —— B 比 C "更长"，且 B 只能是信号/表达式，不能是 sequence
```

一句话区分：**`within`** 是"我这段发生在你那段区间里面"；**`throughout`** 是"我从头到尾一直成立，贯穿你那一串"。

> Mehta：14.18.10 Sig1 throughout Seq1；14.18.11 Seq1 within Seq2。

## 5. `ended()`：引用另一个序列的结束点

```systemverilog
@(posedge clk) $rose(C) |-> sequence_inst(param1, param2).ended();
// 使用 .ended() 时，前面不能再接表达式操作符
```

> Mehta：14.22 End Point of a Sequence (.triggered)（14.22.1 .matched）。

## 6. 重复操作符（Repetition Operators）

| 写法 | 含义 |
|---|---|
| `##m` | 固定延迟 m 拍 |
| `##[m:n]` | 延迟范围 m~n 拍（谁先满足谁触发） |
| `[*m]` | 连续重复 m 次（consecutive） |
| `[*m:n]` | 连续重复 m~n 次 |
| `[=m]` | 非连续重复 m 次（不要求紧挨着，也不要求以它结尾） |
| `[=m:n]` | 非连续重复 m~n 次 |
| `[->m]` | 非连续 **GoTo** 重复 m 次（必须以第 m 次命中作为序列结束点） |
| `[->m:n]` | 非连续 GoTo 重复 m~n 次 |

`[=m:n]` 与 `[->m:n]` 的区别：两者都允许中间夹杂其他值（非连续），但 **GoTo（`->`）版本要求最后一次命中就是整个匹配的结束点**，普通非连续版本（`=`）在命中最多次数后，下一拍还需要额外满足别的条件才算真正结束。

```systemverilog
@(posedge clk) A |=> B[->1:2] |=> C;
// A 之后：B 非连续出现 1~2 次（以最后一次 B 为结束点）|=> 下一拍看 C

@(posedge clk) A |=> B[=1:2] |=> C;
// A 之后：B 非连续出现 1~2 次，但结束点不是 B 本身，其后还要再等一拍才看 C
```

> Mehta：14.18.3 `[*m]`、14.18.4 `[*m:n]`、14.18.5 `[=m]`、14.18.6 `[=m:n]`、14.18.7 `[->]`；`[=m:n]` 与 `[->m:n]` 的区别见 14.18.8。

## 7. 手写笔记里的原始速记（保留原文，便于对照）

```systemverilog
assert property(p_name); else $display("fail");

sequence seq2(a, b, c);
  a ##1 b ##1 c;
endsequence

property name;
  @(posedge clk) disable iff(condition) A |-> ##0 B;
  @(posedge clk) A |=> B;
  //多条件时候使用: &, |, ! ；多个序列之间使用: and, or, not
  @(posedge clk) A |-> ##[1:3] B;
  @(posedge clk) A[*1:2] |-> B;
  @(posedge clk) A |=> B[->1:2] |=> C;   // A #1 -> ... -> B ... -> B -> C
  @(posedge clk) A |=> B[=1:2] |=> C;    // A #1 -> ... -> B ... -> B -> ... -> C
endproperty

@(posedge clk) $rose(A)/$fell(A) |-> B;
$onehot(A);
$onehot0(A);
$countones(A);
$isunknown({A, B});
```

## 8. 自测要点

1. `B and C` 与 `B intersect C` 的区别是什么？（结束点是否要求对齐）
2. `throughout` 和 `within` 分别要求哪个序列在"外层/内层"？
3. `[=m:n]` 和 `[->m:n]` 的核心区别是什么（结束点定义）？
4. 为什么 `not(...)` 类断言写起来容易把逻辑搞反？举一个例子说明。
