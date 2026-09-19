# 功能覆盖率（Functional Coverage）

## 1. 为什么需要功能覆盖率？

- **代码覆盖率（Code Coverage）** 是 EDA 工具自动统计的：哪些 RTL 语句、分支、状态跳转被执行过。它不知道哪些跳转是"有意义的"，会把所有没覆盖到的都列出来，需要人工筛掉 don't-care 的部分。
- **功能覆盖率（Functional Coverage）** 回答的是另一个问题：**是否覆盖了设计规格（design spec）想要验证的场景**？
  - 100% 代码覆盖率 ≠ 100% 功能覆盖率。代码覆盖率只保证"结构"被执行过，不保证"设计意图"被验证到。
  - 功能覆盖率是**人工/手动**设计的（根据 spec 提炼出关键场景），无法自动生成。
  - 三种典型覆盖角度：
    - **Control-oriented**：是否测试过所有协议组合（比如 burst / non-burst 都测过吗）？
    - **Transition coverage**：状态/数值的先后跳变有没有测到（比如先访问一个字节，再访问一个四字节）？
    - **Cross coverage**：多个事件是否同时发生过（比如 tag error 和 data error 同时注入）？

功能覆盖率方法学有两大语言机制：
1. `cover property`（属于 SVA 的一部分，见断言笔记）——本质是 `assert` 的"影子"，同一个 property 既能拿来断言，也能拿来统计覆盖率（时序/组合域覆盖）。只能用在 module/program/interface，不能用在 class 里。
2. `covergroup` / `coverpoint` / `bins`（本文重点）——可以用在 class 里，专门用来统计**变量/表达式的取值分布**。

## 2. covergroup 基本结构

```systemverilog
bit [2:0] a;
bit [3:0] b;

covergroup cg @(posedge clk);      // 采样事件：每个 posedge clk 采样一次
  cp_a: coverpoint a {             // 用户自定义 bins
    bins values_a = {0, 1, 3, 5, 7};
  }
  cp_b: coverpoint b;              // 不写 bins，自动为每个取值建一个 bin
endgroup

cg cg_inst = new();                // 像 class 一样，需要 new() 实例化
```

要点：
- `covergroup` 可以定义在 package / module / program / interface / checker / class 里。
- 定义了 covergroup 之后必须 `new()` 出实例才会真正采样。
- 采样可以用时钟事件触发（如上），也可以手动调用 `cg_inst.sample()`。

## 3. coverpoint 与 bins

`coverpoint` 是"要覆盖的表达式"，`bins` 是这个表达式落在哪个"桶"里才算覆盖到。

### 3.1 自动 bins vs 用户自定义 bins

```systemverilog
bit [3:0] var_a;
covergroup test_cg @(posedge clk);
  cp_a: coverpoint var_a {
    bins low_bins[]  = {[0:3]};   // "[]" => 数组形式：0,1,2,3 各占一个独立 bin，共 4 个
    bins med_bins    = {[4:12]};  // 不加 []：4~12 所有取值合并算一个 bin
  }
endgroup
```

**这道题的关键结论**：`low_bins[]` 因为写了 `[]`，会把 `[0:3]` 拆成 4 个独立 bin（0、1、2、3 各一个，命中任意一个都单独计数）；`med_bins` 没写 `[]`，`[4:12]` 这 9 个值合起来只算 1 个 bin，只要命中其中任意一个数就算这个 bin 被覆盖。所以本例总共 **5 个 bin**（4 + 1）。

一句话记忆：**`bins name[] = {...}` → 数组形式，一个值一个格子；`bins name = {...}` → 一个 bin，多个值共享一个格子。**

### 3.2 ignore_bins / illegal_bins

```systemverilog
coverpoint a {
  ignore_bins ignore_vals = {7, 8};   // 明确排除，不会被统计也不会报错
}

covergroup cg3;
  coverpoint b {
    illegal_bins bad_vals = {1, 2, 3}; // 一旦采样到这些值，运行时直接报错
  }
endgroup
```

区别：`ignore_bins` 只是"不算数"，安静地跳过；`illegal_bins` 是"这个值根本不该出现"，采样到就是运行时错误（优先级高于其他 bins，即使同时属于别的合法 bin 也会报错）。

### 3.3 wildcard bins

```systemverilog
coverpoint a[3:0] {
  wildcard bins bin_12_to_15 = {4'b11??};
}
```
`?` 在 wildcard bin 里代表"0 或 1 都行"（don't care），所以 `4'b11??` 匹配 1100/1101/1110/1111 这 4 个值，命中任意一个都算这个 bin 覆盖到。

### 3.4 transition bins（跳变覆盖）

```systemverilog
coverpoint v_a {
  bins sa = (4 => 5 => 6);   // 连续三拍分别是 4,5,6 才算命中
}

coverpoint my_variable {
  bins trans_bin[] = (a, b, c => x, y);
  // 展开为 6 种跳变：a=>x, a=>y, b=>x, b=>y, c=>x, c=>y，每种一个独立 bin
}

coverpoint var_a {
  bin hit_bin = {3[*4]};    // 连续 4 拍都采样到值 3，才算命中
}
```

### 3.5 default bins 的坑

```systemverilog
int var_a;
covergroup test_cg @(posedge clk);
  cp_a: coverpoint var_a {
    bins low = {0, 1};
    bins other[] = default;   // 危险！
  }
endgroup
```
`default` 会把"除已列出之外的所有取值"各建一个独立 bin。对一个 32 位 `int` 来说，这意味着可能产生 2^32 - 2 个 bin，**极易让仿真器崩溃或内存爆炸**，慎用。

## 4. cross coverage（交叉覆盖）

用来验证"多个变量/coverpoint 是否同时取到过某些组合"，本质是多个 coverpoint 的 bins 做笛卡尔积。

```systemverilog
bit [31:0] a_var;
bit [3:0]  b_var;

covergroup cov3 @(posedge clk);
  cp_a: coverpoint a_var { bins yy[] = {[0:9]}; }   // 10 个 bin
  cp_b: coverpoint b_var;                            // 4 位变量，16 个自动 bin
  cc_a_b: cross cp_b, cp_a;                           // 16 * 10 = 160 个交叉 bin
endgroup
```

也可以直接对"变量"做 cross（不用先手动声明 coverpoint），语言会隐式为它建一个 coverpoint：

```systemverilog
bit [1:0] cmd;
bit [3:0] sub_cmd;

covergroup abc_cg @(posedge clk);
  a_cp: coverpoint cmd;              // 2 位 => 4 个自动 bin
  cmd_x_sub: cross cmd, sub_cmd;     // sub_cmd 隐式生成 16 个 bin，交叉 4*16=64 个 bin
endgroup
```

**注意**：cross coverage 只能在**同一个 covergroup 内**的 coverpoint 之间做。

## 5. 小结 / 自测要点

- covergroup 是干什么的？—— 统计变量/表达式实际采样到的取值分布，反映"设计意图"是否被验证覆盖。
- `bins x[] = {...}` 和 `bins x = {...}` 的区别？—— 前者数组形式一值一 bin，后者合并成一个 bin。
- ignore_bins vs illegal_bins？—— 前者悄悄排除，后者采样到就报错，且优先级最高。
- cross coverage 的 bin 数怎么算？—— 各 coverpoint bin 数的乘积（笛卡尔积）。
