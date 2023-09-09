+++
title = "Search in Rotated Sorted Array"
date = "2023-09-09"
tags = ["dev"]
categories = ["dev"]
+++

# Search in Rotated Sorted Array

如果我们需要在一个已经排好序的数组中寻找某个元素，首选的方法肯定是进行二分搜索——因为在排序完成的数组中搜索满足单调性。但是将这个数组进行旋转后，是否依旧能进行二分搜索呢？

```
123456 -> 3456 12
```

这个例子展示了数组旋转的结果：将整个数组旋转了 4 位。数组的旋转定义如下：

```
原数组 N 经过 x (x > 0) 位旋转后得到的新数组 N' 满足：

N'[i] = N[i - x] ，索引值需要对数组长度进行取模。
```

## 单调性

经过旋转后的排序数组已经不再拥有单调性，但是一部分元素内部仍然具有单调性。比如说下面这个数组：

```
3456 | 12
```

被分割开的部分是数组本来的起始位置，在它的两侧，元素分别拥有单调性。这意味着，我们任意选择一个索引把数组分为两侧，至少有一侧的元素可以保持单调：因为数组原本的起始位置只可能落在一侧，另一侧一定是单调的。

根据这个性质，我们可以把数组从某个索引分为两个部分，一个部分拥有单调性，而另一个部分（不一定）没有单调性。在有单调性的一侧，我们只需要将搜索的目标值与起始位置和末尾位置的元素进行比较，就能知道目标值是否有可能落在这个区间，否则目标值只可能在另一侧。

## 实现

```rust
pub fn search(nums: Vec<i32>, target: i32) -> i32 {
    fn bsearch(nums: &[i32], target: i32, offset: usize) -> i32 {
        let len = nums.len();
        println!("{} - {}", offset, offset + len);
        let mid = len >> 1;

        if nums[mid] == target {
            (offset + mid) as i32
        } else {
            if mid + 1 < len && nums[mid + 1] <= nums[len - 1] {
                if target >= nums[mid + 1] && target <= nums[len - 1] {
                    bsearch(&nums[mid+1..], target, offset + mid + 1)
                } else if mid >= 1 {
                    bsearch(&nums[..mid], target, offset)
                } else {
                    -1
                }
            } else if mid >= 1 && nums[mid - 1] >= nums[0] {
                if target >= nums[0] && target <= nums[mid - 1] {
                    bsearch(&nums[..mid], target, offset)
                } else if mid + 1 < len {
                    bsearch(&nums[mid+1..], target, offset + mid + 1)
                } else {
                    -1
                }
            } else {
                -1
            }
        }
    }

    bsearch(&nums, target, 0)
}
```
