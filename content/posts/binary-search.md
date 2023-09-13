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

## 例题：寻找两个正序数组的中位数

[题目描述](https://leetcode.cn/problems/median-of-two-sorted-arrays/)

本题的关键是确定如何进行二分搜索。我们假设最终的中位数大于等于 nums1 ， nums2 的前 id1 ， id2 个元素。

- 第一点，因为我们有两个正序数组，在没有将它们合并的情况下，如何进行二分搜索？这里的关键是，我们确定了在其中一个数组的索引，我们能不能确定在另一个数组的索引。首先我们假设两个数组 nums1 和 nums2 ，其中，nums1.len() <= nums2.len() ，如果我们确定了在 nums1 中的索引，有没有办法确定 nums2 的索引。注意到中位数的性质，它一定比一半的数组元素要大，这意味着 id1 + id2 = (len1 + len2) / 2 ，那么另一个数组的索引可以通过 id2 = (len1 + len2) / 2 - id1 来计算。

- 第二点，二分搜索的键值是什么？这也需要利用中位数的性质。中位数比前一半的数大，比后一半的数小，转换到两个数组的情况下可以这样表示：nums1[id1], nums2[id2] >= nums1[..id1] U nums2[..id2] ，且对于 id < id1 ， nums1[id] < nums2[id2] ，对 nums2 同理。这个性质中体现了单调性：只有 id >= id1 ，才有 nums1[id] >= nums2[id2 - 1] ，而 id2 与 id1 有关。接下来的问题就是利用二分搜索查找 id1 了。

```rust
pub fn find_median_sorted_arrays(mut nums1: Vec<i32>, mut nums2: Vec<i32>) -> f64 {
    fn binary_search_by_key(elems: &[usize], target: bool, key_fn: impl Fn(usize) -> bool) -> Result<usize, usize> {
        use std::cmp::Ordering;

        let len = elems.len();
        let (mut left, mut right) = (0, len);

        while left < right {
            println!("{left} - {right}");
            let mid = left + ((right - left) >> 1);
            let mid_key = key_fn(elems[mid]);

            match target.cmp(&mid_key) {
                Ordering::Less | Ordering::Equal => {
                    right = mid;
                }
                Ordering::Greater => {
                    left = mid + 1;
                }
            }
        }

        let left_key = key_fn(elems[left]);
        if left_key == target { Ok(left) } else { Err(left) }
    }

    let (mut len1, mut len2) = (nums1.len(), nums2.len());
    let len = len1 + len2;
    let mid = len >> 1;

    if len1 > len2 {
        std::mem::swap(&mut len1, &mut len2);
        std::mem::swap(&mut nums1, &mut nums2);
    }

    let elems: Vec<usize> = (0..=len1).collect();
    let search_res = binary_search_by_key(&elems[..], true, |id| {
        let id1 = id;
        let id2 = mid - id1;

        if id1 == len1 {
            true
        } else {
            nums1[id1] >= nums2[id2 - 1]
        }
    }).unwrap();
    let id1 = search_res;
    let id2 = mid - id1;

    if len & 1 == 1 {
        *(nums1.get(id1).unwrap_or(&i32::MAX).min(nums2.get(id2).unwrap_or(&i32::MAX))) as f64
    } else {
        let left = *(nums1.get(id1 - 1).unwrap_or(&i32::MIN).max(nums2.get(id2 - 1).unwrap_or(&i32::MIN)));
        let right = *(nums1.get(id1).unwrap_or(&i32::MAX).min(nums2.get(id2).unwrap_or(&i32::MAX)));
        (left as f64 + right as f64) / 2.
    }
}
```

