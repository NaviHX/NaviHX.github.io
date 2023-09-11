+++
title = "Lowest Common Ancestor"
date = "2023-09-11"
tags = ["dev"]
categories = ["dev"]
+++

# Lowest Common Ancestor

给定一棵树与树中的两个节点，如何找出这两个节点的最近公共祖先。最近公共祖先的定义为，对于一个有根树 T 与其中的两个节点 p ， q ，p 与 q 的最近公共祖先为一个节点 x ，满足以 x 为根的子树同时包含了节点 p 与 q ，且 x 的深度尽可能的深。

## 朴素做法： DFS

```rust
type RefNode = RefCell<TreeNode>;
type RcNode = Rc<RefNode>;
type PtrNode = *const RefNode;
type MaybeNode = Option<RcNode>;

enum Res {
    Both(MaybeRcNode), // LCA
    Only1,
    Only2,
    Neither,
}

fn dfs(root: MaybeRcNode, node1: RcNode, node2: RcNode) -> Res {
    if let Some(root) = root {
        let left = root.borrow_mut().left.take();
        let right = root.borrow_mut().right.take();

        match (root == node1, root == node2, dfs(left, node1, node2), dfs(right, node1, node2)) {
            (_, _, Both(ans), _) | (_, _, _, Both(ans)) => Both(ans),
            (true, true, _, _) => Both(Some(root)),
            (_, _, Only1, Only2) | (_, _, Only2, Only1) => Both(Some(root)),
            (true, _, Only2, _) | (true, _, _, Only2) => Both(Some(root)),
            (_, true, Only1, _) | (_, true, _, Only1) => Both(Some(root)),
            (true, _, Only1, _) | (true, _, _, Only1) => Only1,
            (_, true, Only2, _) | (_, true, _, Only2) => Only2,
            (_, _, Only1, Only1) => Only1,
            (_, _, Only2, Only2) => Only2,
            _ => Neither,
        }
    } else {
        Res::Neither
    }
}
```

## 多次查询

上面的做法对于单次查询比较有效，只需要将树中的所有节点遍历一遍就可以找到最近公共祖先。但是如果我们将要对同一棵树进行多次的 LCA 查询，这种做法的效率就无法接受了。我们可不可以只遍历一次树，利用树的信息来寻找 LCA 呢？

通过一次遍历，从树中我们可以得到任意一个节点的深度与父亲节点信息。因此我们可以通过下面这个步骤来寻找 LCA ：

1. 将两个节点指针中，深度较大的指针向上移动至同一个深度。
2. 比较两个节点指针，如果相同，则这个节点就是 LCA ，否则跳至第三步。
3. 将两个节点指针分别移动到他们的父节点。

```rust
let mut depth: HashMap<PtrNode, usize> = HashMap::new();
let mut parent: HashMap<PtrNode, RcNode> = HashMap::new();

build_info(root.clone(), None, &mut depth, &mut parent); // fn build_info(root, parent_node: MaybeNode, depth, parent);

let depth1 = depth[&(node1.as_ptr() as *const _)];
let depth2 = depth[&(node2.as_ptr() as *const _)];

// Make sure depth1 <= depth2
if depth2 < depth1 {
    std::mem::swap(&mut depth1, &mut depth2);
    std::mem::swap(&mut node1, &mut node2);
}

// Raise node1
let d = depth2 - depth1;
for _ in 0..d {
    node2 = parent[&(node2.as_ptr() as *const _)];
}

// Raise both until equal
while node1 != node2 {
    node1 = parent[&(node1.as_ptr() as *const _)].clone();
    node2 = parent[&(node2.as_ptr() as *const _)].clone();
}

// LCA
return node1;
```

## 倍增

在上面这个过程中，我们发现后续的 Raise 过程的时间复杂度与深度有关。当二叉树退化为链表时，时间复杂度退化为整个树的节点数目，不算十分理想。我们是否可以加速 Raise 过程呢？

我们不妨修改一下 parent 的定义，将其通过索引节点指针得到的结果变成一个列表（而不是直接父节点），第 i 个位置的元素表示这个节点的第 2^0 个父节点。这样的定义满足如下的性质：

1. parent(parent(node, i-1), i-1) == parent(node, i)
2. 交换律 parent(parent(node, i), j) == parent(parent(node, j), i)

第一条性质我们可以使用在构建 parent 表中。对于任意一个节点的 parent 列表，我们可以通过下面的过程计算。

1. 将这个节点的直接父节点作为它的 parent 列表的第 0 元素
2. let i = 1
3. 使用性质一， `parent[node][i] = parent[parent[node][i-1]][i-1]`  ，如果此处的索引超出界限则跳出循环
4. i += 1
5. 跳转到第三步

在构建好 parent 表之后，可以通过这张表将节点指针上升任意 2 的整数次幂高度。我们接下来考虑，如何通过这个表，将节点指针上升任意高度 d 。首先，对于任意一个正整数 d ，我们可以将其转换为一个关于 2 的多项式。

```
d = (a_n)2^n + (a_(n-1))2^(n-1) + ... + (c_0)2^0
c_i = 0 | 1
```

这意味着我们可以通过 n 次跳跃就可以将节点指针提升任意高度。同时由于性质二交换律，我们可以任意组织跳跃的顺序。

```rust
// raise height d
let i = 0;
while d > 0 {
    node = parent[&(node.as_ptr() as *const)][i]; // Assume element exists
    i += 1;
    d >>= 1;
}
```

这一段代码可以代替将较低的节点指针提升到相同高度的部分。

剩下的将两个节点提升到 LCA 的过程也可以通过新的 parent 表进行优化——我们不再需要将节点指针逐次提升一个高度，而是跳转到 parent 表中一个恰好没有使节点指针相同的高度。当没有一个高度可以使两个指针不同时，则父节点就是 LCA 。

## 完整代码

```rust
type RefNode = RefCell<TreeNode>;
type RcNode = Rc<RefNode>;
type PtrNode = *const RefNode;
type MaybeNode = Option<RcNode>;

pub fn lowest_common_ancestor(root: Option<Rc<RefCell<TreeNode>>>, p: Option<Rc<RefCell<TreeNode>>>, q: Option<Rc<RefCell<TreeNode>>>) -> Option<Rc<RefCell<TreeNode>>> {
    use std::collections::HashMap;

    fn build_parent(root: MaybeNode, depth: usize, parent: MaybeNode, parents: &mut HashMap<PtrNode, Vec<RcNode>>, depths: &mut HashMap<PtrNode, usize>) {
        if let Some(root) = root {
            depths.insert(&(root.as_ptr() as *const _), depth);
            if let Some(parent) = parent {
                let mut ps: Vec<RcNode> = vec![parent.clone()];
                let mut i = 0;
                loop {
                    if let Some(p) = parents
                        .get(&(ps[i].as_ptr() as *const _))
                        .and_then(|pv| pv.get(i))
                    {
                        ps.push(p.clone());
                        i += 1;
                    } else {
                        break;
                    }
                }

                parents.insert(root.as_ptr() as *const _, ps);
            }

            let left = root.borrow_mut().left.take();
            let right = root.borrow_mut().right.take();
            
            let (pdl, qdl) = build_parent(left, depth + 1, Some(root.clone()), parents, depths);
            let (pdr, qdr) = build_parent(right, depth + 1, Some(root.clone()), parents, depths);
        }
    }

    if let (Some(mut p), Some(mut q)) = (p, q) {
        let mut parents = HashMap::new();
        let mut depths = HashMap::new();
        build_parent(root.clone(), 0, None, &mut parents, &mut depths);
        if let (Some(mut pd), Some(mut qd)) = (depths.get(&(p.as_ptr() as *const _)), depths.get(&(q.as_ptr() as *const _))) {
            // Make sure that pd <= qd
            if pd > qd {
                std::mem::swap(&mut p, &mut q);
                std::mem::swap(&mut pd, &mut qd);
            }

            // Jump to the same level
            let mut i = 0;
            let mut d = qd - pd;
            while d > 0 {
                if d & 1 == 1 {
                    q = parents[&(q.as_ptr() as *const _)][i].clone();
                    println!("+{i}");
                }

                i += 1;
                d >>= 1;
            }

            // Jump as far as they can, until find the LCA
            while p != q {
                // p_depth == q_depth, so p_len == q_len
                let len = parents[&(p.as_ptr() as *const _)].len();
                for i in (0..len).rev() {
                    let pp = parents[&(p.as_ptr() as *const _)][i].clone();
                    let qp = parents[&(q.as_ptr() as *const _)][i].clone();

                    if pp != qp || i == 0 {
                        p = pp;
                        q = qp;
                    } 
                }
            }

            // LCA
            Some(p)
        } else {
            None
        }
    } else {
        None
    }
}
```
