# 综合案例：req / ack 握手协议断言

这一课把前面学的 `|->`/`|=>`、`intersect`、`not`、采样值函数、local variable 全部串起来，用一个真实的 req/ack 握手协议作为综合练习。

## 1. 基础版：6 条断言（来自手写笔记原文）

下面是原始记录下来的 6 条 property 表达式（未来复习时可以对照教材 14.14 / 14.17 / 14.18.20 节，把每一条的用途重新推导一遍，加深理解）：

```systemverilog
// (1) 没有 req 的时候不该有 ack（"未经请求不得响应"，防止虚假 ack）
!req |-> !ack;

// (2) 核心的 req->ack 超时检查：(!req||ack) ##1 req 是"干净的 req 上升沿"
//     （前一拍没有正在进行的握手），命中后允许 req && !ack 最多持续 100 拍，
//     之后 ack 必须出现。(!req||ack) ##1 req 就是笔记里说的"影子寄存器"技巧的
//     简化版：用一个组合条件捕捉"这是一次全新的握手开始"，避免把同一次握手的
//     中间状态误判成新的 req
(!req || ack) ##1 req |-> (req && !ack)[*0:100] ## ack;

// (3) ack 一旦出现（$rose(ack)），最多保持 100 拍后必须回落（不能一直悬空拉高）
$rose(ack) |-> ack[*0:100] ##1 (!ack);

// (4) ack 到达且 req 仍为高的那一拍，下一拍 req 必须撤销
//     —— 即握手完成后 req 要及时收回，不能赖着不走
$rose(ack) && req |=> !req;

// (5) 健康性检查：req 不能出现未知态 X
!(isunknown(req));

// (6) 数值稳定性/回绕（wrap-around）检查：用 $stable + $past 判断计数值 value
//     与上一拍的差是否符合预期（含跨越最大值时的回绕情况），常用于给带
//     ID/计数域的握手信号做数据完整性校验，为下面的"ID 匹配进阶版"做铺垫
$stable(value) |-> (val < $past(val) ?
                      ($past(val) - val == value) :
                      ($past(val) + ({width{1'b1}} - value + 1 == value)));
```

> 复习建议：拿这 6 条去对照"7 条文字规格"重新逐条复述一遍中文规格，直到能不看代码就把每条断言口头描述出来（大致方向如上面注释所示，复习时建议自己重新对一遍教材原文推导出更严谨的表述）。

## 2. 进阶版：用 local variable 做 ID 匹配

基础版的问题：如果两次 `req` 挨得很近（还没等到第一次 `ack`，第二次 `req` 又来了），只看"有没有 ack 在超时窗口内出现"是不够的——**无法保证每一次 req 都对应到了正确的那一次 ack**（可能一次 ack 被"错误地"当成满足了两次 req 的检查）。

教材 14.21 节用一个几乎同构的例子（`rdy` / `rdyAck`）说明了这个陷阱以及正确解法，这正是"进阶版 ID 匹配"要解决的问题：

### 2.1 有问题的写法（无法保证一一对应）

```systemverilog
sequence rdyAckCheck;
  (1'b1, $display($stime,,,"ENTER SEQUENCE rdy ARRIVES"))
  ##[1:5]
  ((rdyAck), $display($stime,,,"rdyAck ARRIVES"));
endsequence

gcheck: assert property (@(posedge clk) rdy |-> rdyAckCheck)
  begin $display($stime,,,"PASS"); end
  else begin $display($stime,,,"FAIL"); end
```
如果两个 `rdy` 挨着来，中间只有一个 `rdyAck`，这个属性依然会 PASS 两次——因为它只检查"5 拍内有没有 rdyAck"，不检查"是不是专门对应这一次 rdy 的"。

### 2.2 正确写法：local variable 逐次打标签

```systemverilog
sequence rdyAckCheck;
  byte localData;                       // local variable：每次进入 sequence 都会创建一个新实例，
                                         // 相当于给每一次 rdy 单独开一个线程去追踪，不用手动维护流水线状态

  (1'b1, localData = rdyNum,
   $display($stime,,,"rdy ARRIVES: ",,,"LOCAL rdyNum=", localData))
  ##[1:5]
  ((rdyAck && rdyAckNum == localData),
   $display($stime,,,"rdyAck ARRIVES ",,,"LOCAL",,,
            "rdyNum=", localData,,, "rdyAck=", rdyAckNum));
endsequence

gcheck: assert property (@(posedge clk) (rdy) |-> rdyAckCheck)
  begin $display($stime,,,"PASS"); end
  else begin $display($stime,,,"FAIL",,,"rdyNum=", rdyNum,,,"rdyAckNum=", rdyAckNum); end
```

关键技巧拆解：
- **local variable 是"动态"的**：sequence 每被触发一次（每次 `rdy` 命中），都会 fork 出一份独立的 `localData`，不会和上一次的实例互相覆盖——不需要你手动写队列/计数器去维护"流水线中有几个 rdy 还没等到 ack"。
- **逗号采样动作（`, localData = rdyNum`）**：在序列匹配的某个节点上顺便执行一个赋值/打印动作，把"当前这次 rdy 对应的编号"记下来。
- **匹配条件里再比对**：`rdyAck && rdyAckNum == localData` —— 不仅要求 `rdyAck` 出现，还要求它带的编号跟这个线程当初记下来的编号**完全一致**，才算这次 rdy 真正等到了属于自己的 ack。
- 时间窗口：本例用的是 `##[1:5]`；在带 ID 的 req/ack 握手（笔记里提到的 `p_req_ack_id`）中，同样的技巧配合更长的窗口（例如 `##[100:250]`）就能在真实设计里做"每个 req 必须在 100~250 拍内等到带正确 ID 的 ack"这种更贴近实际协议的校验。

### 2.3 与基础版的关系

| | 基础版（第 1 节） | 进阶版（本节） |
|---|---|---|
| 能否处理背靠背（back-to-back）请求 | 不行，容易"张冠李戴" | 可以，每个请求独立追踪 |
| 关键机制 | 影子条件 `(!req\|\|ack) ##1 req` 捕捉"干净的新请求" | `local variable` 给每次请求单独开线程 + 显式比对 ID |
| 复杂度 | 低，适合协议简单、请求稀疏的场景 | 略高，但能覆盖请求密集、允许多个未完成事务并行的场景（更贴近真实总线协议） |

## 3. 自测要点

1. 为什么"两个 req 挨着来，只看超时窗口内有没有 ack"这种检查方式不够严谨？
2. local variable 在 sequence 里是"静态"还是"动态"的？这意味着什么？
3. `localData = rdyNum` 这种写法叫什么？它在序列匹配的哪个阶段执行？
4. 如果要把本节的技巧套到"任意个未完成事务并发"的场景（而不仅仅是两个），思路上需要改变什么吗？（提示：local variable 本身已经是"per-thread"的，不需要额外改动）
