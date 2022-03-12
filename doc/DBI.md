# 十分钟魔法练习：De Bruijn 索引

### By 「玩火」

> 前置技能：Java 基础，ADT，λ 演算

## 符号命名

在 λ 演算的实现介绍中提到过关于符号重名的问题，需要通过重命名来避免规约时因为重名导致符号引用对象出错。如 `λ z. (λ x. (λ z. x)) z` 如果不做重命名处理就会变成 `λ z. (λ z. z)` ，而正确情况应该是 `λ z'. (λ z. z')`。为了保证符号的唯一性，之前采用了生成 UUID 对原符号重命名的策略。

而为了达到「唯一」这一目的，除了随机一个非常大的数以外还有个更好的办法，那就是利用「在 AST 上距离所引用符号跨过的 λ 抽象层数」来生成唯一的符号名。

比如 `λ x. (λ z. x) x` 中内层 λ 中的 `x` 距离所引用的 `λ x.` 跨过了一个 `λ z.` 所以可以命名为 `1` ，而外侧右边的 `x` 向外看最近的一层 λ 抽象就是其引用的符号所以可以命名为 `0`。

那么 `λ x. (λ z. x) x` 用这种方法就表示为 `λ. (λ. 1) 0`。类似的，上面提到的 `λ z. (λ x. (λ z. x)) z` 就可以表示为 `λ. (λ. (λ. 1)) 0`。

不过看上去这个办法有个很显然的困惑，如果向上能数的 λ 抽象层数超过符号值了那意味着什么呢？比如 `λ. 1` 中的 `1`，实际上它是一个未捕获的自由符号，就比较类似于 `λ x. y` 中的 `y` 。

这种表示符号的索引方案就称为「De Bruijn 索引 (De Bruijn index)」。

## 实现细节

这个处理方案和之前的 UUID 方案实现基本相同，唯一要注意的点就是 `apply` 操作时符号的变化。考虑规约 `(λ. X) e` 时把 `e` 应用到 `λ. X` 中，最简单的情况是整个表达式 `(λ. X) e` 中无自由变量，那就只需要将 `X` 中绑定到这一层的符号替换为 `e` 。

当 `X` 中有自由变量时就需要考虑替换后自由变量的引用对象保持不变，注意到替换后少了一层 λ 抽象，所以需要把 `X` 中的自由变量引用的符号层数减一。

而当 `e` 中有自由变量时就要考虑当它所替换的符号层数，每深一层就需要把 `e` 中的自由变量引用的符号层数加一。

说起来好抽象，举个例子就是 `(λ. 0 (λ. 1)) 1` 规约得到的结果是 `1 (λ. 2)`。代码中要注意的点如下：

```java
////
// class Val
public Expr lift(int v) {
    // `e` 中的自由变量
    if (v < this.v) return new Val(this.v + 1);
    return this;
}

public Expr apply(int v, Expr e) {
    // 替换
    if (v == this.v) return e;
    // `X` 中的自由变量
    if (v < this.v) return new Val(this.v - 1);
    return this;
}

////
// class Fun
public Expr apply(int v, Expr e) {
    return new Fun(this.e.apply(v + 1, e.lift(0)));
}
```

## 二进制编码

在 De Bruijn 索引的基础上，可以构造二进制编码来紧凑地存储 λ 表达式：

- `00` 代表 `Fun`，它后面接一个参数
- `01` 代表 `App`，它后面接两个参数
- `111...1` 代表 `Val`，其中第一个 `1` 之后的 `1` 的数量就代表了 `Val` 的索引值

比如 `(λ. 0 (λ. 1)) 1` 的二进制编码就是 `01 00 01 1 00 11 11`。

像这样的前缀编码可以无歧义地解析，状态机也非常简单。再过一遍哈夫曼编码以后文件可以压缩地很小。