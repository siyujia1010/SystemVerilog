# 随机约束（randomize / constraint）

## 1. 随机数生成函数

```systemverilog
$random(seed)%n;        // 产生随机数范围为 [-(n-1) : (n-1)]（有符号）
{$random(seed)}%n;      // 加花括号强转无符号，范围变成 [0 : (n-1)]，n 为大于0的整数
$random()%(b-a+1)+a;    // 表示 a~b 之间的一个随机整数
$urandom_range(10, 50); // 功能同上，更规范，推荐使用
bit result = std::randomize(a) with {a >= 1};  // 局部变量随机化，result为1成功/0失败
addr = {$urandom, $urandom};  // 拼两个32位随机数凑成64位
number = $urandom & 15;       // 按位与截取低4位，得到[0:15]的4位随机数
object.randomize() with {};   // 类成员随机化，with{}里可临时追加约束（只在本次调用生效）
```

> `$random%n` 加不加花括号，结果的符号范围不一样，容易踩坑。

## 2. inline约束的名字解析（`local::`）—— 易错点

```systemverilog
class C1;
  rand integer x;
endclass

class C2;
  integer x;
  integer y;
  task doit(C1 f, integer x, integer z);
    int result;
    result = f.randomize() with {x < y + z;};
    // x -> class C1 / y -> class C2 / z -> local argument
    result = f.randomize() with {local::x < y + z;};
    // x -> local argument
  endtask
endclass
```

**规则**：`with{}` 里的裸标识符，优先绑定到被随机化对象自身（这里是 `f`，即 `C1` 类型）的成员变量，而不是外层局部变量。第一行的 `x` 指的是 `C1::x`；`y` 因为 `C1` 里没有，往外层作用域找到 `C2::y`；`z` 是普通局部形参。

如果确实想引用当前作用域（task的局部形参）里的 `x`，必须显式加 `local::` 前缀。**这是唯一区分"约束对象自身成员"和"外部同名变量"的方式**，是面试高频坑点。

## 3. `rand_mode` / `constraint_mode` / `pre_randomize` / `post_randomize`

```systemverilog
bus.cstr.constraint_mode(0);   // 关闭 bus 对象里名为 cstr 的这一条约束块
bus.constraint_mode(0);        // 关闭 bus 对象上所有的约束块
bus.rand_mode(0);              // bus这个对象的所有rand变量都不再被随机化，保持原值
bus.data.rand_mode(0);         // 只关闭 bus.data 这一个具体变量的随机化
constraint bus::cstr2 { addr > 5; }  // extern constraint写法，类外部扩展约束
pre_randomize();   // randomize()调用前自动执行的钩子函数
post_randomize();  // randomize()调用后自动执行的钩子函数
```

**一句话区分**：`rand_mode` 管的是"变量还随不随机化"，`constraint_mode` 管的是"某条约束还参不参与求解"，两者互不影响、可独立开关，也都能随时用 `(1)` 重新打开。注释"these will change randc behavior"提醒：关闭/打开 `rand_mode` 会影响 `randc` 变量的遍历状态（相当于重置了洗牌记录）。

## 4. soft 约束

```systemverilog
constraint cstr {
  soft data == 1;
  data == 2 -> soft data == 1;
}
```

soft 约束只是一个可被覆盖的默认建议值：只要存在与之冲突的硬约束（或优先级更高/更晚声明的 soft 约束），硬约束一定优先满足，soft 约束会被静默丢弃，**不会**导致 `randomize()` 失败。这是它和普通硬约束最大的区别——硬约束冲突会直接导致 `randomize()` 返回 0。

## 5. dist 权重分布：`:=` vs `:/`

```systemverilog
data dist{0 := 20, [1:3] := 50};
// := 表示区间内每一个值单独获得同样的权重
// 0:1:2:3 = 20 : 50 : 50 : 50

data dist{0 :/ 20, [1:3] :/ 50};
// :/ 表示50是整个区间[1:3]的总权重，会被平均分给区间里的3个值
// 0:1:2:3 = 20 : 50/3 : 50/3 : 50/3
```

## 6. 数组归约约束

```systemverilog
array.sum with (int'(item)) == 50;
array.product with (int'(item)) == 50;
array.and with (int'(item)) == 50;
array.or with (int'(item)) == 50;
array.xor with (int'(item)) == 50;
```

`item` 是 `with()` 子句里系统隐式提供的迭代变量，代表遍历过程中的当前数组元素（这里先做 `int'` 类型转换再求和），不需要用户自己声明，类似 `foreach(array[i])` 的隐式循环变量。整体约束的意思是：数组所有元素（转换为int后）求和必须等于50。

（V0课程里对应的基础写法是 `A.sum() < 1000`，不带 `with()` 的条件表达式；`with(int'(item))` 这种更灵活的写法是笔记/Mehta书补充的进阶内容。）

## 7. `solve...before` 与概率分布偏置 —— 经典面试坑点

条件：`a` 只能取 `{0,1}`，`b` 只能取 `{0,1,2,3}`，唯一约束：

```systemverilog
constraint c { (a == 0) -> (b == 1); }
```

**不加 `solve...before` 时**：8种组合里有5种满足约束（`(0,1)(1,0)(1,1)(1,2)(1,3)`），求解器把整个解空间当作整体，对这5个解做**均匀采样**，每个解概率都是 `1/5`。

**加上 `solve a before b` 后**：求解器分两步走——

1. 先按 `a` 的定义域均匀选 `a`（各 `1/2`），不考虑约束
2. 再在 `a` 已确定的前提下，对 `b` 在约束限定范围内均匀选：
   - `a=0` 时：`b` 被约束死只能取 `1`，概率 `1`，所以 `P(a=0,b=1) = 1/2 × 1 = 1/2`
   - `a=1` 时：约束不生效，`b` 可任意取，4个值各 `1/2 × 1/4 = 1/8`

**对比**：不加 `solve` 时 `(0,1)` 的概率是 `1/5`，加了 `solve a before b` 之后变成 `1/2`，差了2.5倍。

![solve...before 概率分布对比：同一张 (a,b) 网格，左为整体均匀采样，右为先选 a 再选 b](assets/solve-before-probability.svg)

**结论**：`solve...before` 不只是决定求解顺序、提高求解效率，它会**实实在在改变结果的概率分布**，使其不再是全局均匀。这是DV面试里考察"约束求解器不是黑箱"的高频题。

> 补充：V0课程里没有讲到这个知识点，是笔记/Mehta书的独有内容。

## 8. 路科V0课程对应关系

| 本笔记内容 | V0课程对应位置 | 备注 |
|---|---|---|
| `$random`/`$urandom`/`$urandom_range` | 第3讲《随机约束》"如何简单地产生一个随机数？" | 一致 |
| `randomize() with{}` 内联约束 | 第3讲《随机约束》"内嵌约束" | 一致 |
| inline约束名字解析 | 第3讲《随机约束》"内嵌约束（指向模糊）" | 一致 |
| `local::` 域指向 | 第3讲《随机约束》"local域指向" | 一致，例子不同规则相同 |
| `rand_mode` | 第3讲《随机约束》"随机控制" | 一致 |
| `constraint_mode` | 第3讲《随机约束》"约束控制" | 一致 |
| `randomize(x)`带参数 | 第3讲《随机约束》"内嵌变量控制" | 一致 |
| `pre_randomize()`/`post_randomize()` | V0未讲 | 笔记/Mehta补充 |
| extern约束`constraint bus::cstr2{}` | V0未讲 | 笔记/Mehta补充 |
| `dist` 权重分布 | 第3讲《随机约束》"约束块（权重分布）" | 一致 |
| 数组归约约束 `.sum()` | 第3讲《随机约束》"约束块（迭代约束）" | V0只讲基础`.sum()`，不含`with(int'(item))`进阶写法 |
| `solve...before`概率偏置 | V0未讲 | 笔记/Mehta补充，V0全文搜索无此内容 |
| soft约束 | 第3讲《随机约束》"约束块（软约束）" | 一致 |

## 9. 相关笔记

- **依托的语言基础**：`rand` / `randc` 成员和约束块都定义在 class 里（对应后续"类的封装与继承"一章）；§6 的数组归约约束用到 [数组](../01-data-types/04-arrays.md) 的内置方法。
- **下游**：随机约束负责"产生激励"，[功能覆盖率](../04-functional-coverage/README.md) 负责衡量"随机有没有覆盖到该覆盖的场景"，两者合起来就是 coverage-driven verification。
