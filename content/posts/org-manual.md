---
title: Org 速查手册
date: 2023-01-27
tags: [org,blogging]
categories: [blogging]
---
# 文档结构

Org是基于Outline模式实现的.

## 标题

标题定义了大纲树的结构, 以从行首开始的若干个星号标识.

## 列表

可以使用有序列表, 无序列表, 描述列表.

无序列表  
以 - , + 或 \* 开头

有序列表  
以 1. 或 1) 开头

描述列表  
用::将项和描述分开

## 脚注

脚注就是以脚注定义符号开头的一段话,
脚注定义符号是将脚注名称放在一个方括号里形成的, 要求放在第0列,
不能有缩进. 而引用就是在正文中将脚注名称用方括号括起来.

    The Org homepage.[fn:1]

    [fn:1] The link is: http://orgmode.org

# 表格

任何以 \| 为首个非空字符的行都会被认为是表格的一部分. \| 也是列分隔符.

    | Name  | Phone | Age |
    |-------+------+-----|
    | Peter | 1234 | 17  |
    | Anna  | 4321 | 25  |

| Name  | Phone | Age |
|-------|-------|-----|
| Peter | 1234  | 17  |
| Anna  | 4321  | 25  |

# 链接

    [[link][description]]

    or

    [[link]]

# 待办事项

    * STATE [#PRIORITY] ITEM :TAG_A:TAG_B:...
    :PROPERTIES:
    :p: ...
    ...
    :END:

当纯文本的项以 `[]` 开头时, 就会变成一个复选框.

    * TODO Organize party [1/3]
    - [-] call people [1/2]
      - [ ] Peter
      - [X] Sarah
    - [X] order food
    - [ ] think about what music to play

# 字体样式

    *粗体*
    /斜体/
    +删除线+
    _下划线_
    下标： H_2 O
    上标： E=mc^2
    等宽字：  =git=  或者 ~git~

**粗体**

*斜体*

~~删除线~~

<u>下划线</u>

等宽字： `git`

# 参考

[Orgmode官方手册](https://orgmode.org/manuals.html)

[Orgmode for
Neovim](https://github.com/nvim-orgmode/orgmode/blob/master/DOCS.md)
