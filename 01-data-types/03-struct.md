# struct 结构体

## 原始笔记

```systemverilog
//default : unsigned & unpacked
//unpacked signed structure is illegal, defalut format is unpacked
//packed structure cannot have rand variables
typedef struct packed signed{
  int coins;
  real dollars;
  byte data[4];
} money;

//default 中的内容可以被后面的覆盖
money array = '{coins:100，default: 0, data:0};
```

## 讲解

### packed vs unpacked

- **默认情况**：结构体不加 `packed` 关键字时是 **unpacked**（成员各自占自己的存储空间，不保证连续排布），且默认是 **unsigned**。
- **`unpacked` + `signed` 不合法**：`signed`/`unsigned` 这个属性只对 **packed** 结构体有意义（因为只有 packed 结构体才被当成一个整体的位矢量来看待，才谈得上"整体是有符号还是无符号"）；对 unpacked 结构体写 `signed` 是不合法的。
- **packed 结构体不能有 `rand` 成员**：`packed struct` 的每个成员必须是可以"打包"进连续位域的类型（整型、其它 packed 类型等），`rand` 声明的随机化属性和 packed 的位域打包机制是冲突的，所以 packed 结构体里不能直接放 `rand` 变量。

> ⚠️ 笔记里这个 `money` 例子本身有一处需要留意：`packed` 结构体要求所有成员都是可打包的（整型/packed 类型），而 `real dollars;` 是浮点类型，并不满足这个条件。这行大概率是手写笔记速记时的简化/笔误，实际写 packed 结构体时不能塞 `real` 成员。复习到这里时建议对照原始资料再确认一遍这个例子的准确写法，这里先如实保留原文，避免我自己瞎改动了原始记录。

### 结构体字面量赋值：`'{...}` 与 `default`

```systemverilog
money array = '{coins:100, default: 0, data:0};
```

`'{...}` 是**赋值模式（assignment pattern）**语法，可以按成员名指定初始值：

- `coins: 100`：把 `coins` 成员设成 100
- `default: 0`：**没有被显式点名的成员**，一律用 0 填充
- `data: 0`：显式把 `data` 也设成 0（相当于覆盖了 `default` 对 `data` 本来也会生效的填充值，只是这里写的值恰好一样）

关键规则：**写在后面的具体成员赋值，会覆盖 `default` 对该成员的默认填充**——`default` 更像是"兜底"，具体点名的成员优先级更高。这也是为什么原始笔记特意加了一句注释"default 中的内容可以被后面的覆盖"。
