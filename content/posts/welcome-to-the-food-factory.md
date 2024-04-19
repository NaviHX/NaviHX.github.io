+++
title = "Welcome to the Food Factory"
date = "2024-04-19"
tags = ["dev"]
categories = ["dev"]
+++

# Welcome to the Food Factory!

这是一篇介绍 [工厂模式](https://zh.wikipedia.org/wiki/%E5%B7%A5%E5%8E%82%E6%96%B9%E6%B3%95) 的短文。

## 原始机器

假如你是一位机械工程师，你开发了一款机器，它可以自动使用原材料生产食物。

```rust
fn original_machine(input: Ingredients) -> Dish {
    let water = boil(100);
    let dish = make_dish_with_water(input, water);
    dish
}
```

你觉得这样很不错，因为你可以使用这一台机器构造流水线，完成每天的工厂生产任务。

```rust
let mut products = vec![];
for i in 0..1000 {
    let input = get_ingredients();
    let new_product = original_machine(input);
    products.push(new_product);
}
```

## 订单需求改变

你突然接到了一个新订单，它不再需要用水煮的食物，而是用火烤的。你的第一反应是改进原本的机器。

```rust
fn original_machine(input: Ingredients) -> Dish {
    let dish = roast(input);
    dish
}

let mut products = vec![];
for i in 0..1000 {
    let input = get_ingredients();
    let new_product = original_machine(input);
    products.push(new_product);
}
```

实际上，这是一个很不明智的做法：如果你还需要生产原始的订单，你还需要把机器再改回去。所以，一个更好的方法是，在保留原有的机器基础上，重新设计一台新机器。

```rust
fn original_machine(input: Ingredients) -> Dish {
    let water = boil(100);
    let dish = make_dish_with_water(input, water);
    dish
}

fn roast_machine(input: Ingredients) -> Dish {
    let dish = roast(input);
    dish
}
```

这样的话，你就能根据订单类型来决定生产哪种食品了。

```rust
enum Order{
    Original,
    Roast,
}

fn assembly_line(order: Order, amount: usize) -> Vec<Dish> {
    let mut products = vec![];
    for i in 0..amount {
        use Order::*;
        let input = get_ingredients();

        let new_product = match order {
            Original => original_machine(input),
            Roast => roast_machine(input),
        };
        products.push(new_product);
    }

    products
}
```

然而，随着你业务的增长，你的流水线里很可能有非常多的机器种类。

```rust
fn assembly_line(order: Order, amount: usize) -> Vec<Dish> {
    // ...

    let new_product = match order {
        Original => // ...
        Roast => // ...
        // Oh my god. Too many machines!
        Fry => // ...
    }

    ///...
}
```

这个时候，你的流水线构造（或者说你的代码）将会变得十分冗余，每次添加新的订单种类，总是需要设计新的机器。有没有办法作出改变呢？

## 做什么，你来决定

在之前的设计中，流水线只能生产已经设计好的产品（已经设计了机器的商品）。而实际上，客户总是能提出新的订单需求，作为工程师，每次设计一个新的机器是十分费神的。为了节约精力，我们需要让客户自己决定生产的流程。

```rust
trait Order {
    fn produce(input: Ingredients) -> Dish;
}
```

客户需要通过写明制作流程，告诉我们应该如何制作。我们只需要制造新的满足流程的机器就好了。

```rust
struct OriginalOrder;
impl Order for OriginalOrder {
    fn produce(input: Ingredients) -> Dish {
        original_machine(input) // 客户说，使用本来的设计就行了。
    }
}

struct RoastOrder;
imple Order for RoastOrder {
    fn produce(input: Ingredients) -> Dish {
        roast_machine(input) // 这个订单需要火烤。
    }
}

struct FryOrder;
imple Order for FryOrder {
    fn produce(input: Ingredients) -> Dish {
        fry_order(input) // 油炸也是可以完成的。
    }
}
```

当需要生产时，客户把订单交给我们，我们按订单上写明的流程生产就行了。

```rust
fn assembly_line(order: impl Order, amount: usize) -> Vec<Dish> {
    let mut products = vec![];
    for i in 0..amount {
        let input = get_ingredients();
        let new_product = order.produce(input);

        products.push(new_product);
    }

    products
}
```

## 现在你拥有了工厂模式

如果我们把这层食品工厂的皮扒掉，工厂其实是一个这样的东西。

```
Factory :: Input -> Output
```

它获取一个输入，返回一个输出。这里的输出既可以是某一个具体的类型，也可以是满足一个特征的类型集合，我们根据输入的不同产生不同的输出。

工厂提供了创建一个值的流程的抽象，将创建流程的控制反转给了输入。

```
Produce :: input -> Factory -> output
Produce input factory = factory input -- 根据不同的工厂决定如何输出
```
