+++
title = "Longest Palindrome Substring"
date = "2023-09-08"
tags = ["dev"]
categories = ["dev"]
+++

# How to Find the Longest Palindrome Substring

[题目链接](https://leetcode.cn/problems/longest-palindromic-substring/)

曾经会的算法，因为太久没做算法题，已经忘了。在这里留一下笔记。

## 朴素的做法

对于一个回文字符串中心 S[c]，如果它的半径是 R ，必须满足

```
S[c - i] == S[c + i], 0 <= i <= R
```

所以如果我们需要知道以字符串中的任何一个位置作为中心的回文串的最大长度，只需要利用扩散的方法，找到第一个不相等的位置，就能得到回文串的半径。如果要求一个字符串的最长回文子串，只需要求出每个位置为中心的回文串的半径，选取最大者即可。

不过，这种做法的时间复杂度为 O(n^2) 。

## 利用已经计算的回文半径

如果我们能够利用之前已经获得的回文半径，是否可以加速计算的过程。在回答这个问题前，我们先来观察一下回文串的性质。

对于一个回文串中心 S[c] ，如果它的回文半径为 R （ R > 0 ）

```
......A......
```

比如上图所示的字符串，它的中心位置 c = 6 处的字符为 A ，回文半径为 6 。假设在中心的左侧子串中包含了一个更小的回文串

```
.ABA..A...B..
```

左侧的子串 ABA 是一个回文串，中心在 c = 2 处。由于回文串的性质可以得知，在原字符串的索引 10 处也应该有一个 B 。如果我们此时已经知道 c = 6 处的回文半径，现在需要计算 c = 10 处的回文半径，我们是否需要从回文半径 R = 0 开始进行扩散，逐个匹配字符？

注意到在第一节中提到的性质，距离回文中心相同距离的两侧的字符应该是相等的。这意味着在上面提到的例子中，回文中心 c = 10 处的回文半径至少是 1 —— 根据回文串的性质，这个字符串应该为 `.ABA..A..ABA.` ；同时回文半径最多只能为 1 —— 因为 c = 2 处的回文半径为 1 ，已经不能再扩散了。因此，当一个较大的回文串「完全」包含了当前的回文子串时，它的回文半径与它的关于回文中心的位置的回文半径相同。

## 中心在两个字符之间

只要在原字符串的每两个字符之间插入一个相同的无关字符即可。

## 特殊情况

现在来考虑这样一种情况：根据如上的算法，如果现在正在计算的回文中心的对称点的回文串左侧的索引与大回文串的左侧索引相同，像下面这种情况：

```
ABA.C.ABA
```

这时我们还能保证以右侧 B 为中心的回文串的回文半径只能为 1 吗？显然不能。此时索引为 9 的位置已经超出了大回文串的范围，已经失去了之前的信息，我们只能确定回文半径的最小值为 1 ，而不能确定最大值，必须进行扩散来获取准确值。

那么如果对称点的回文半径超出了大回文串的范围呢？由于失去了大回文串的限制，我们不能保证超出范围的部分依旧满足回文的条件，只能截取在大回文串中的部分。

```
A[BA.C.AB]?
```

比如上面这个例子，左侧 B 的回文半径为 1 ，但是不能保证右侧 ? 位置的字符为 A ，所以也只能从包含在大回文串的部分，也就是回文半径为 0 开始扩散。

## 示例代码

```rust
pub fn longest_palindrome(s: String) -> String {
    fn expand(s: &str, center: usize, skip: usize) -> usize {
        assert!(center >= skip);

        let (mut left, mut right) = (center - skip, center + skip);
        let s = s.as_bytes();
        let len = s.len();
        loop {
            if left > 0 && right < len - 1 && s[left - 1] == s[right + 1] {
                left -= 1;
                right += 1;
            } else {
                break (right - left) >> 1;
            }
        }
    }

    let ns: String = "#"
        .chars()
        .chain(s.chars().flat_map(|c| [c, '#']))
        .collect();
    let len = ns.len();
    let mut radius = vec![0; len];
    let mut center = 0; // center with max right bound
    let mut right = 0; // max right bound = center + radius[center]
    let (mut ans_pos, mut ans_radius) = (0, 0);

    for i in 1..len {
        let arm = if i > right {
            expand(&ns, i, 0)
        } else {
            let i_sym = center - (i - center);
            let skip = radius[i_sym].min(right - i);
            expand(&ns, i, skip)
        };
        radius[i] = arm;

        if right < i + arm {
            center = i;
            right = i + arm;
        }

        if arm > ans_radius {
            ans_pos = i;
            ans_radius = arm;
        }
    }

    ns[(ans_pos - ans_radius)..(ans_pos + ans_radius + 1)]
        .chars()
        .filter(|&c| c != '#')
        .collect()
}
```

