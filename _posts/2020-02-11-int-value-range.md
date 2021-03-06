---
layout: post
title: "int取值范围引发的有趣思考"
subtitle: '计算机是如何存储负数的？'
author: "Tukeping"
header-style: text
tags:
  - Java
  - 计算机原理
---

#### 引言

​    在国内面试的时候，有可能会被问到：在Java语言里int类型的取值范围是多少？这个问题也容易回答：`-2147483648 ~ 2147483647`。int占用4字节。每个字节有8bit。所以总共能表达出 2^32个数字。如果带上有符号的话，需要将第一个bit位表示成符号位，0表示正数，1表示负数。那范围就在 -2^31 ~ (2^31) - 1 之间 (2^31 = 2147483648)。正因为自然数0这个特殊的存在所以能表达的正数需要少一个数字。

#### 存储二进制的方式

​    你知道计算机是如何存储`8`的吗？用二进制表达出来就是`0000 1000`1个字节就可以。那你知道`-8`是怎么在计算机里存储的吗？按照上面的逻辑，我们把第一个bit位当作符号位，那`1000 1000`这样就可以表示出是负的8。是不是真的是这样呢？带着问题到java里试一下。

#### 找寻真相

​    直接在java里执行`Integer.toBinaryString(-8)`你会惊奇的发现，诶，居然打印出来的是`11111111111111111111111111111000`这个是什么鬼？可以去google一下或者找下CS的教科书，上面会写到`负数在计算机里会用补码的形式来存储`，而补码就是先反码加1。ok，什么鬼，为什么是反码，为什么还要加1？先小小tips一下反码就是二进制每一位取反，也就是每一位从0变成1或者从1变成0。 好了，我们来试想一个问题，`16 + (-8)`在二进制里进行加法结果会是如何？
```math
​    0001 0000
 +  1000 1000
—————————————
    1001 1000
```
这里相加后再转成十进制后的答案是 `-24`。这个答案明显是错的。也就是说这里不能把`-8`当作一个整体来加上去，而是要单独做减法。这样显然是让计算机的处理更加复杂了。那么如果我们用补码的方式来存储负数，再做一次上面的加法呢？
```math
​    0001 0000
 +  1111 1000
—————————————
   10000 1000
```
由于我们这里设定的是这个数据结构只会用到一个字节也就是8bit，所以第9个bit中的1，就会溢出进而被舍去。所以结果就是 `0000 1000`也就是十进制中的 `8`。好了，这下 `16 + (-8) = 8`答案是正确的了。计算机也可以使用一套加法的处理逻辑来处理所有关于加和减的逻辑了。ok，很统一，很完美。但是好像有什么被忽略了。就是还是`没有解释通为什么要把负数存储成补码形式？`

#### 负数存储补码的本质

`-8`可以表示为`0-8`。由于0的二进制没法让8来减，那么我们可以再向上借一位来减去8。
```math
   10000 0000
 -  0000 1000
—————————————
​    1111 1000
```
而这个`1111 1000`就是`-8`的二进制补码了。我们再继续观察会发现 `1 0000 0000 = 1111 1111 + 1`也就是说 我们可以让 `1111 1111` - `1000` + `1`。
```math
​     1111 1111
 -   0000 1000
——————————————
​     1111 0111
 +   0000 0001
——————————————
​     1111 1000
```
这个就是补码是从反码+1，而负数在计算机中存储的二进制方式就是补码的由来。
结论：`本质就是数学中的向高位借位`。

#### 用数学的方式来证明：正数(包括负数) 加法 适用于补码方式？

实际上，我们要证明的是，X-Y或X+(-Y)可以用X加上Y的补码完成。Y的补码等于`(11111111-Y) + 1`。所以，X加上Y的补码，就等于：`X + (11111111-Y) + 1`。我们假定这个算式的结果等于Z，即 `Z = X + (11111111-Y) + 1`。接下来，分成两种情况讨论：

第一种情况：如果X小于Y，那么Z是一个负数。这时，我们就对Z采用补码的逆运算，求出它对应的正数绝对值，再在前面加上负号就行了。所以

`Z = -[11111111 - (Z-1)] = -[11111111 - (X + (11111111 - Y) + 1 - 1)] = X - Y`

第二种情况：如果X大于Y，这意味着Z肯定大于`11111111`，但是我们规定了这个数据结构只有8位，最高的第9位是溢出位，必须被舍去，这相当于减去`100000000`。所以

`Z = Z - 100000000 = X + (11111111-Y) + 1 - 100000000 = X - Y`

这就证明了，在正常的加法规则下，可以利用补码得到正数与负数相加的正确结果。换言之，计算机只要部署加法电路和补码电路，就可以完成所有整数的加法。

#### 额外

自己动手实现了一下Integer.toBinaryString。虽然实现不是代码最优美的，但是毕竟是自己动手写的，也对于二进制处理中补码，反码，以及二进制进位更进一步深入了解了。

```java
import org.junit.Test;

import java.util.Arrays;

import static org.hamcrest.core.Is.is;
import static org.junit.Assert.assertThat;

/**
 * @author tukeping
 * @date 2020/2/11
 **/
public class IntegerToBinary {

    public String toBinaryString(int i) {
        char[] buf = new char[32];
        Arrays.fill(buf, '0');

        boolean isNegative = i < 0;

        if (isNegative) {
            buf[31] = '1';
            i = -i;
        }

        boolean isOdd = (i % 2 == 1);
        if (isOdd) {
            buf[0] = '1';
            i--;
        }

        int n = i / 2; // 偶数一定能被2整除

        for (int k = 0; k < n; k++) {
            // 进位
            fireCarry(buf, 1);
        }

        if (isNegative) {
            // 先反码 再加1
            inverseBytes(buf);
            if (buf[0] == '0') {
                buf[0] = '1';
            } else {
                fireCarry(buf, 0);
            }
        }

        int lastOneIndex = lastOne(buf);

        String str = new String(buf);
        String outputStr = str.substring(0, lastOneIndex + 1);

        return new StringBuilder(outputStr).reverse().toString();
    }

    private int lastOne(char[] buf) {
        for (int i = buf.length - 1; i >= 0; i--) {
            if (buf[i] == '1') return i;
        }
        return 0;
    }

    private void fireCarry(char[] buf, int start) {
        for (int x = start; x <= 30; x++) {
            if (buf[x] == '0') {
                buf[x] = '1';
                break;
            } else {
                buf[x] = '0';
            }
        }
    }

    private void inverseBytes(char[] buf) {
        for (int i = 0; i <= 30; i++) {
            if (buf[i] == '0') {
                buf[i] = '1';
            } else if (buf[i] == '1') {
                buf[i] = '0';
            }
        }
    }

    @Test
    public void test1() {
        assertThat(toBinaryString(0), is("0"));
        assertThat(toBinaryString(1), is("1"));
        assertThat(toBinaryString(2), is("10"));
        assertThat(toBinaryString(3), is("11"));
        assertThat(toBinaryString(4), is("100"));
        assertThat(toBinaryString(5), is("101"));
        assertThat(toBinaryString(6), is("110"));
        assertThat(toBinaryString(7), is("111"));
        assertThat(toBinaryString(8), is("1000"));
        assertThat(toBinaryString(9), is("1001"));
        assertThat(toBinaryString(-8), is("11111111111111111111111111111000"));
    }
}
```