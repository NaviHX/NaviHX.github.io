+++
title = "Range Bitwise-And"
date = "2024-03-09"
tags = ["dev"]
categories = ["dev"]
+++

# 数字范围按位与

给你两个整数 left 和 right ，表示区间 [left, right] ，返回此区间内所有数字 _按位与_ 的结果（包含 left 、right 端点）。

# Solution

注意到按位与的特性：只要参与按位与的所有数中，有一个数在某一位为零，那么结果的这一位肯定为零。

同时我们可以发现在连续出现的数中，第 i 位的数是 0 与 1 交替出现的，交替出现的周期为 2^(i+1) ，也就是说，最长会连续出现 2^i 个 1 。因此我们只需要知道两个端点处第 i 位是否为 1，同时两个端点的距离 dis 是否超过半个周期，就能判断结果中第 i 位是否为一。结果的第 i 位可以通过下面的公式计算：

```haskell
mask i = shiftL 1 i
ans left right i = (left .&. mask i) .&. (right .&. mask i) .&. if (right - left) < mask i then 0xffffffff else 0
```

对应的 Rust 代码是


```rust
pub fn range_bitwise_and(left: i32, right: i32) -> i32 {
    let dis = right - left;

    let mut ans = 0;
    for i in 0..31 {
        let m = 1 << i;
        ans |= left & right & m & if dis < m { !0 } else { 0 }
    }

    ans
}
```

观察第二个式子我们可以发现， (dis\ <\ mask) 是关于 i 单调递增的，即存在一个 a 使得 i >= a, dis >= mask 且 i < a, dis < mask 。那么我们只需要找出最小的满足条件的 i ，就可以知道该位以及更高的位的掩码都是 1 。

现在我们需要找到一个方法求到每一位的掩码组合在一起的掩码，这样就可以同时计算所有位了。方法是很简单的：将 dis 最高的 1 以下的位全部置为 1 ，然后按位取反。这样就可以保证，只保留最低的 1 ，掩码也是大于 dis 的。


```rust
pub fn range_bitwise_and(left: i32, right: i32) -> i32 {
    let dis = right - left;

    let mut m = dis;
    m = m | (m >> 1);
    m = m | (m >> 2);
    m = m | (m >> 4);
    m = m | (m >> 8);
    m = m | (m >> 16);
    m = !m;

    left & right & m
}
```

