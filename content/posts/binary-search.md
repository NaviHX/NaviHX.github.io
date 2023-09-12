+++
title = "Binary Search"
date = "2023-09-12"
+++

# Binary Search

最常见的算法之一。因为每次写出来的都不一样 XD ，所以在这里留一个笔记。

## 算法输出与循环不变量

这里介绍的二分搜索算法可以寻找目标元素的最小索引（假设数组已经从小到大排序），当无法找到时，则返回目标元素可以插入的位置（也就是比多少个元素大）。

已知已排序的数组 elem ，定义两个变量 left & right ，满足

- 0 <= left < right <= elem.len()
- target 大于 elem[..left] 中的所有元素
- target 小于等于 elem[right..] 中的所有元素

每次循环都要缩小 left - right 的范围来找到目标元素。因为需要对数组进行排序和比较大小，所以数组元素 / 数组元素对应的 key 必须是全序的。

缩小范围的步骤为

- 比较中间元素的与目标元素的大小
- 如果目标元素 >= 中间元素， right := mid
- 如果目标元素 < 中间元素， left := mid + 1

缩小范围的过程一直保持后两个不变量，所以跳出循环的条件应该是第一个不变量不成立。此时满足 target > elem[..left] and target <= elem[right..] （注意这里 ..left 不包括 left ），由于不满足第一个不变量，此时 left >= right 。由于缩小范围的过程中，每次缩小时都满足 right' = mid or left' = mid + 1 ，而 left <= mid < right ，所以最后一次缩小后，left == right 。考虑到后两个不变量成立，此时 left 就是目标元素的最小索引。

## Code

```rust
fn binary_search<T>(elem: &[T], target: T) -> Result<usize, usize>
where
    T: Ord
{
    use std::cmp::Ordering;

    let len = elem.len();
    let mut left = 0;
    let mut right = len;

    while left < right {
        let mid = left + ((right - left) >> 1);
        match target.cmp(&elem[mid]) {
            Ordering::Less | Ordering::Equal => {
                right = mid;
            }
            Ordering::Greater => {
                left = mid + 1;
            }
        }
    }

    if elem[left] == mid { Ok(left) } else { Err(left) }
}
```

