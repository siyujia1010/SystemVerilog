# enum 枚举类型

## 原始笔记

```systemverilog
typedef enum{red,blue} light;
int c = red + 1; light l = light'(c)
l.first()/last()/next()/prev/num()
light l = l.first();
do begin
  l = l.next();
end
while(l != l.first())
```

## 讲解

### 声明与整数互转

```systemverilog
typedef enum {red, blue} light;   // 默认 red=0, blue=1（可以显式指定数值）
```

枚举变量本质上还是整数（默认 `int`），所以可以直接参与整数运算，但运算结果的类型是整数，不会自动变回枚举类型，要转回来必须显式做**类型转换（cast）**：

```systemverilog
int   c = red + 1;      // c 是普通 int，值为 1
light l = light'(c);    // 用 light'(...) 把整数 1 转型回枚举类型 light，也就是 blue
```

如果不加 `light'(...)` 这个转型，直接写 `light l = c;` 是不合法的（整数不能隐式赋给枚举类型）。

### 内置方法：遍历所有取值

| 方法 | 作用 |
|---|---|
| `.first()` | 返回枚举类型的第一个值 |
| `.last()` | 返回最后一个值 |
| `.next()` | 返回当前值的下一个；如果当前已经是最后一个，`.next()` 会**绕回到第一个** |
| `.prev()` | 返回当前值的上一个；同理，第一个值的 `.prev()` 会绕回到最后一个 |
| `.num()` | 返回这个枚举类型总共有多少个取值 |

正因为 `.next()`/`.prev()` 会自动绕回，才能写出下面这种"遍历一整圈"的写法：

```systemverilog
light l = l.first();
do begin
  l = l.next();
end while (l != l.first());
```

这是一个 `do...while` 循环：先从第一个值开始，每次取下一个值，直到 `.next()` 转了一整圈、又绕回到 `first()` 为止。用 `do...while` 而不是普通 `while`，是因为要保证循环体至少执行一次（哪怕枚举类型只有一个值）。
