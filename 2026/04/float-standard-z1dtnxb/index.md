# IEEE 754 浮点数标准（以 FP32 为例）






这篇文章主要记录一下 **IEEE 754 的 float 标准**，以最常见的 **FP32（single precision）**  为例。

FP32 一共由 **32 位**组成：

- **1 位符号位（sign）**
- **8 位指数位（exponent）**
- **23 位尾数位（fraction）**

可以表示为：

```text
[ sign | exponent(8) | fraction(23) ]
```

![image](https://pic.bynshard.top/image-20260412174005-qmogslb.png)

---

# 一、正规数（Normal）

对于绝大多数浮点数，IEEE 754 使用的是 **正规数（normal number）** 。

它的数学形式为：  
​$x = (-1)$<sup>$s \times (1.f) \times 2$</sup>$e$  

其中：

- ​`s`：符号位
- ​`1.f`：有效数字（significand）
- ​`e`：真实指数（biased exponent - 127）

---

## 为什么尾数默认是 `1.f`

二进制科学计数法要求规格化后一定写成：  
​$1.xxxxx \times 2^e$  

因为小数点左边永远是 `1`​，所以这个 `1`​ **根本不用存**。

于是 IEEE 754 直接白嫖这一位：

> 默认最高位恒为 `1`

这就是经典的 **hidden bit（隐含前导 1）** 。

所以 FP32 虽然只存了 **23 位 fraction**，但实际精度是：

> **24 位有效精度**

这也是 float 能提供约 **7 位十进制有效数字** 的核心原因。 :contentReference[oaicite:4]{index=4}

---

# 二、为什么需要非正规数（Subnormal）

hidden bit 很优雅，但它有一个问题：

> 最小正规数不能继续缩小了

FP32 最小正规正数是： $1.0 \times 2^{-126}$

约等于：$1.17549435\times10^{-38}$

如果没有额外设计，再小就会直接变成 `0`。

这会造成：

> **sudden underflow（骤然下溢）**

也就是最小正规数和 `0` 中间出现巨大的表示空洞。 :contentReference[oaicite:5]{index=5}

---

## 渐进下溢（Gradual Underflow）

IEEE 754 的解决方案是：

> 引入 **非正规数（subnormal）**

规则是：

- 指数位固定全 0
- 不再使用 hidden bit
- 尾数变成：$0.f$

数学形式变成：$x = (-1)$​<sup>$s \times (0.f) \times 2$</sup>​${-126}$

这样就可以利用 **23 位 fraction**  
把 `(0, min_normal)` 之间的空间均匀填满。

最小 subnormal 为：$2^{-149}$ ，约等于：$1.40129846\times10^{-45}$

这就实现了：

> **平滑地衰减到 0**  
> 而不是突然掉成 0。 :contentReference[oaicite:6]{index=6}

---

# 三、特殊值：Inf 和 NaN

当指数位全为 `1` 时，表示特殊值。

---

## Infinity

如果：

- exponent = `11111111`
- fraction = `000...000`

则表示：$+\infty / -\infty$

例如：

- overflow
- ​`1.0 / 0.0`

都会产生 Infinity。 :contentReference[oaicite:7]{index=7}

---

## NaN

如果：

- exponent = `11111111`
- fraction ≠ `0`

则表示：

> **NaN（Not a Number）**

例如：

- ​`0.0 / 0.0`
- ​`sqrt(-1)`

---

# 四、一个非常优雅的 float 排序技巧

FP32 的 bit pattern 可以通过一个映射，直接转成：

> **可按整数比较大小的 key**

代码如下：

```cpp
uint32_t to_key(float f) {
    uint32_t bits;
    memcpy(&bits, &f, 4);
    return (bits >> 31) ? ~bits : (bits | 0x80000000u);
}
```

---

## 原理

---

### 1）正数：最高位强制设为 1

正数原始 bit：

```text
0 exponent fraction
```

映射后：

```text
1 exponent fraction
```

因为正规浮点在正数区间天然满足：

> 指数越大，数越大  
> 指数相同，尾数越大，数越大

所以可以直接按整数排序。

---

### 2）负数：全部取反

负数原始 bit：

```text
1 exponent fraction
```

直接：

```cpp
~bits
```

效果非常巧妙：

### 符号位

最高位变成 `0`，保证所有负数都排在正数前面。

### 数值顺序

负数里：

> 绝对值越大，实际值越小

而按位取反本质是：  $\sim x = (2^N-1)-x$

所以：

> 原来 bit 越大  
> 映射后反而越小

顺序刚好被纠正。

---

# 五、为什么“尾数不可能大过指数”

这是 IEEE 754 最优雅的设计之一。

正规数恒满足： $1 \le M < 2$

其中： $M = 1.f$

所以浮点数总写成：$x = M \times 2^e$

---

## 为什么指数优先天然正确

假设：

$x_1=M_1\times2^{e_1}$  
$x_2=M_2\times2^{e_2}$

如果：  
​$e_1=e_2+1$  

那么：  
​$x_1 \ge 1\times2$<sup>${e_2+1}=2\times2$</sup>${e_2}$  
  

而：  
$x_2 < 2\times2^{e_2}$

因此一定有：  
​$x_1>x_2$  
  

---

## 本质理解：binade 区间

每个指数对应一个数值区间： $[2$​<sup>$e,\ 2$</sup>​${e+1})$ 尾数只是在这个区间里移动。所以：

> **指数决定区间**  
> **尾数决定区间内位置**

尾数永远不可能跨区间反超。

这就是为什么：

> **先比较指数，再比较尾数**  
> 天然等价于数值排序

‍

