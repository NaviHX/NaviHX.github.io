+++
title = "Single Number"
date = "2024-09-19"
tags = ["dev"]
categories = ["dev"]
+++

# 只出现一次的数字

给你一个非空整数数组 nums ，除了某个元素只出现一次以外，其余每个元素均出现两次。找出那个只出现了一次的元素。

必须使用线性时间，常量空间的算法。

## Solution

根据异或的性质，将所有元素异或后的结果为只出现一次的元素，因为

```
x ^ x == 0
```

且异或满足交换律与结合律，只需要将相同的元素移动到相同位置即可证明。

```rust
fn single_number(nums: &[i32]) -> i32 {
    assert!(!nums.is_empty());
    nums.into_iter().fold(0, |a, b| a ^ b)
}
```

# 出现两次的数字

下面来考虑上面问题的一个变式。

给你一个非空整数数组 nums 其长度为 n + 1 ，只会出现 0..n 的数字，除了某个元素出现两次以外，其余每个元素均只出现一次。找出那个只出现了两次的元素。

## Solution

这个问题拥有更强的条件：只会出现 0..n 的数字。

因为我们知道数组中出现的元素范围，我们只需要将 nums 所有元素的异或再与 0..n 的所有数异或一遍。这样可以使得原本出现两次的元素现在出现了三次（异或结果为本身），原本出现一次的元素出现了两次（异或结果为 0 ）。

```rust
fn double_number(nums: &[i32]) -> i32 {
    let n = nums.len - 1;
    nums.into_iter().copied().chain(0..n as i32).fold(0, |a, b| a ^ b)
}
```

# Sneaky Numbers

[原题目](https://leetcode.cn/problems/the-two-sneaky-numbers-of-digitville/description/)

一个数字列表 nums，其中包含从 0 到 n - 1 的整数。每个数字本应只出现一次，然而，有两个顽皮的数字额外多出现了一次，使得列表变得比正常情况下更长。请你找出这两个顽皮的数字。

## Solution

相比于前一个题目，这一个题目中出现了两次的元素有两个。也就是说假设出现两次的元素分别为 a, b ，参考上一个题目的方法，所有元素的异或结果应为 a ^ b 。

因为这 `a != b` ，所以 `a ^ b` 的结果中必定至少有一个 `1` bit ，假设只保留其中最低位的 `1` bit 的数为 lowbit 。我们可以将列表中的所有元素分为两类：lowbit 处为 1 的数以及 lowbit 处为 0 的数。分别将两类数异或的结果就是所求的元素。

```rust
fn sneaky_numbers(nums: &[i32]) -> Vec<i32> {
    let n = nums.len() - 2;
    let sum = nums.into_iter().copied().chain(0..n as i32).fold(0, |a, b| a ^ b);
    let lowbit = sum & -sum;

    let mut ans = vec![0; 2];
    for num in nums.into_iter().copied().chain(0..n as i32) {
        let id = if num & lowbit == 0 { 0 } else { 1 };
        ans[id] ^= num;
    }

    ans
}
```
