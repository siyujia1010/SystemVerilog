# string 类型

## 原始笔记

```systemverilog
string ans = "" = {"a", "b"} = {2{"a"}}
str.len()/putc(3, "d")/getc(3)/atoi()/substr(i, j)
```

## 讲解

`string` 是 SystemVerilog 里的内置动态类型，专门用来存变长字符串（不用像 Verilog 那样拿 `reg [8*N-1:0]` 硬凑）。上面这行笔记其实是把几种不同的初始化/赋值方式压缩写在一起了，拆开看：

```systemverilog
string ans;
ans = "";           // 空字符串
ans = {"a", "b"};    // 拼接：结果是 "ab"
ans = {2{"a"}};      // 复制/重复拼接：结果是 "aa"
```

`string` 类型支持用 `{}` 做**拼接（concatenation）**和**重复（replication）**，这一点和普通的位矢量拼接语法是一致的，只是操作数换成了字符串：

- `{s1, s2}`：把 s1、s2 首尾相连
- `{n{s}}`：把 s 重复 n 遍

### 常用内置方法

| 方法 | 作用 |
|---|---|
| `str.len()` | 返回字符串长度 |
| `str.putc(i, c)` | 把第 `i` 个字符替换成 `c` |
| `str.getc(i)` | 取第 `i` 个字符（返回的是 byte，即该字符的 ASCII 码） |
| `str.atoi()` | 把字符串转成整数（笔记原文写的是 `atio()`，是 `atoi()` 的笔误；同系列还有 `atohex()`/`atooct()`/`atobin()`/`atoreal()`，以及反过来的 `itoa()`） |
| `str.substr(i, j)` | 取子串，从下标 `i` 到 `j`（闭区间） |

例：
```systemverilog
string s = "hello";
$display(s.len());        // 5
s.putc(0, "H");            // s 变成 "Hello"
byte c = s.getc(1);        // c = "e" 的 ASCII 码
int  n = "123".atoi();     // n = 123
string sub = s.substr(1,3); // "ell"
```

> Mehta：2.11 String Data Type（2.11.1 String Operators、2.11.2 String Methods）。
