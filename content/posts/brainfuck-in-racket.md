+++
title = "Brainf*cking with Racket"
date = "2023-02-08"
tags = ["dev","programming-language"]
categories = ["dev"]
[extra]
toc = true
+++
# Intro

Racket 等 Lisp 家族的语言拥有强大的宏, 可以让我们很方便地对语言进行扩展,
但是使用宏却带来了两点限制:

- 宏不能限制它的上下文和改变上下文的含义.
  宏的设计者无法阻止使用者在任何地方使用宏, 这可能会造成预想不到的后果.
- 宏使用的词法必须遵守原语言的词法规则.

使用宏对语言进行的改造是十分有限的. 如果需要对语言进行深层次的定制,
仅仅使用宏就有点力不从心了. 本文将从 brainfuck 出发, 使用 racket
创造出一门基于 racket 的语言, 同时也是我最近这段时间学习 racket 的总结.

目标是成功运行以下代码:

    #lang bf
    ++++++[>++++++++++++<-]>.
    >++++++++++[>++++++++++<-]>+.
    +++++++..+++.>++++[>+++++++++++<-]>.
    <+++[>----<-]>.<<<<<+++[>+++++<-]>.
    >>.+++.------.--------.>>+.

# Racket 关于语法的基础设施

为了定制我们的 brainfuck , 我们需要了解 Racket 本身提供的设施.
在我们编写 racket 源文件时, 总会像这样写:

``` racket
#lang racket
;; our code begins ...
```

开头的 `#lang racket` 指明了这个源文件使用的语言, 这种特性被称为 module
language. 这个其实是 racket module 的一个 shorthand. 展开为完整形式如下:

``` racket
(module <file_name> racket
  ;; our code begins ...)
```

`module` 中的 `racket` 其实就是最开始导入的包名(即所谓 `init module` ).
这提供了一种控制语法的方法. 比如说可以为我们定制的语言提供一个
`function` 关键字:

``` racket
> (module raquet racket
    (provide (except-out (all-from-out racket) lambda) ;; 导出racket中除了lambda外的内容
       (rename-out [lambda function]))) ;; 将lambda作为function导出
> (module src-file 'raquet ;; 可以理解为 #lang raquet
    (map (function (points) (case points
                             [(0) "love"] [(1) "fifteen"]
                             [(2) "thirty"] [(3) "forty"]))
         '(0 2)
> (require 'src-file)
'("love" "thirty")
```

这样, 我们就可以用 `function` 创建一个函数, 而不是使用 `lambda` .
作为module language导入的包与普通的包又略有不同,
它还需要提供一些额外的项目:

- `#%module-begin`, 包裹了模块代码的隐式语句. 后面会用到.
- `#%app`, 调用一个过程, `(func arg1 arg2)` 就是
  `(#%app func arg1 arg2)`.
- `#%top`, `(#%top . id)`, 返回顶层的变量, 如果有本地绑定则报错.
- `#%datum`, 创建一个字面量.

这些项目都可以在代码中显式地使用, 但它们最主要的用处还是为 module
language 提供了修改这些隐式调用的能力.
比如说在一门定制的lambda运算语言中限制函数只能有一个参数:

``` racket
> (module lambda-calculus racket
  (provide (rename-out [1-arg-lambda lambda]
                       [1-arg-app #%app]
                       [1-form-module-begin #%module-begin]
                       [no-literals #%datum]
                       [unbound-as-quoted #%top]))
  (define-syntax-rule (1-arg-lambda (x) expr)
    (lambda (x) expr))
  (define-syntax-rule (1-arg-app e1 e2)
    (#%app e1 e2))
  (define-syntax-rule (1-form-module-begin e)
    (#%module-begin e))
  (define-syntax (no-literals stx)
    (raise-syntax-error #f "no" stx))
  (define-syntax-rule (unbound-as-quoted . id)
    'id))
> (module ok 'lambda-calculus
    ((lambda (x) (x z))
     (lambda (y) y)))
> (require 'ok)

'z
> (module not-ok 'lambda-calculus
    (lambda (x y) x))

eval:4:0: lambda: use does not match pattern: (lambda (x)

expr)

  in: (lambda (x y) x)
> (module not-ok 'lambda-calculus
    (lambda (x) x)
    (lambda (y) (y y)))

eval:5:0: #%module-begin: use does not match pattern:

(#%module-begin e)

  in: (#%module-begin (lambda (x) x) (lambda (y) (y y)))
> (module not-ok 'lambda-calculus
    (lambda (x) (x x x)))

eval:6:0: #%app: use does not match pattern: (#%app e1 e2)

  in: (#%app x x x)
> (module not-ok 'lambda-calculus
    10)

eval:7:0: #%datum: no

in: (#%datum . 10)
```

上面的例子都重复导出了 Racket 的内容, 或者直接使用了 Racket,
并没有直接使用 `#lang xxx`. 一门完整的 module language, 还需要提供
reader 来解析源文件. 一份 Racket 源文件(或者其他由 Racket
定制而来的语言)的处理过程如下:

    surface syntax --|reader|-> AST --|macro expansion|--> coreforms

另外, module language 还可以用 `read` , `read-syntax` 自定义解析器逻辑.
看上去完全实现一门 module language 是有些困难的. Racket 提供了 `s-exp`
元语言来简化这些流程. 可以像其他 module language 一样使用:
`#lang s-exp module-path` . 它读入后面的 s-expression 并用 *module path*
展开.

这些 Racket 的基础设施为我们的语言定制提供了帮手.
上面的内容如果没有完全理解也不用着急. 后文中遇到相关内容时会做详细解释.

# brainfuck 语法

之所以选择 brainfuck 作为示例, 是因为它的语法足够简单,
我们不用操心解析器的复杂度, 也能完全实现整个流程. 完整的 brainfuck
语法只使用了8种符号用来操作 brainfuck 机器:

- `>` , 向后移动指针
- `<` , 向前移动指针
- `+` , 指针所指字节 + 1
- `-` , 指针所指字节 - 1
- `,` , 从标准输入读入一个字节并写入指针
- `.` , 向标准输出写入指针所指的一个字节
- `[ ... ]` , 循环, 当运行到 `]` 时若指针指向的字节为0则跳出

brainfuck 机器由一条长度为30000 byte 的数据与一个指针构成.

# 实现 brainfuck 语义

在这个章节我们需要实现 brainfuck 各种符号的语义. 在此之前, 我们需要实现
brainfuck 机器的定义:

``` racket
(define-struct state (data ptr) #:mutable)

(define (new-state)
  (state (make-vector 30000 0) 0))
```

`define-struct` 宏为 `state` 生成了默认的构造函数 `state`
和访问成员的函数 `state-*` . 我们需要修改虚拟机的状态, 所以需要添加
`#:mutable` 关键字, 让宏为我们生成 `set-state-*!` 函数.
最后我们定义了一个无参函数 `new-state` 用于创建 brainfuck 虚拟机.

有了虚拟机状态定义, 实现除了循环外的语义是十分简单的工作.

``` racket
(define (increment-ptr a-state)
  (set-state-ptr! a-state (add1 (state-ptr a-state))))

(define (decrement-ptr a-state)
  (set-state-ptr! a-state (sub1 (state-ptr a-state))))

(define (increment-byte a-state)
  (let ([v (state-data a-state)] [i (state-ptr a-state)])
    (vector-set! v i (add1 (vector-ref v i)))))

(define (decrement-byte a-state)
  (let ([v (state-data a-state)] [i (state-ptr a-state)])
    (vector-set! v i (sub1 (vector-ref v i)))))

(define (write-byte-to-stdout a-state)
  (let ([v (state-data a-state)] [i (state-ptr a-state)])
    (write-byte (vector-ref v i) (current-output-port))))

(define (read-byte-from-stdin a-state)
  (let ([v (state-data a-state)] [i (state-ptr a-state)])
    (vector-set! v i (read-byte (current-input-port)))))
```

`current-*-port` 是一个 Racket 中的 parameter,
我们需要使用函数调用才能获取端口的值. Parameter 的值可以通过
`parameterize` 指定某个 scope 中的 parameter 值. (后面也会用到
parameter)

与其他语法符号不同, 我们需要使用宏来实现循环. 至于原因,
我们先看循环的实现:

``` racket
(define-syntax-rule (loop a-state body ...)
  (local [(define (loop)
            (unless (= (vector-ref (state-data a-state) (state-ptr a-state)) 0)
              body ...
              (loop)))]
    (loop)))
```

当符合条件时, 执行 body 并进行一次尾递归. 如果我们将这个宏定义为函数,
含义就发生了变化: 因为 Racket 是严格求值(strict)的, 在 "展开" 函数之前,
body 的值已经计算出来了, 实际上循环的 body 只执行了一遍.

严格求值的反面就是非严格求值. 其中的典型就是 Haskel,
执行函数并不一定需要计算出参数的值, 计算是 lazy 的. 比如像下面这个函数:

``` haskel
-- non_strict :: Num b => a -> b
> non_strict a = 1

> bot = bot

> non_strict bot
1
```

`bot` 是一个无法计算的值, 但是我们仍可以计算出 `non_strict bot` 的值,
因为返回值实际上是与参数无关的, 如果是一门严格求值的语言,
这里已经陷入死循环了.

似乎我们已经离题了. 在完成这个章节前, 我们需要在源文件最前面加上 `#lang`
和导出其他模块需要的定义.

``` racket
#lang racket

(provide (all-defined-out))

;; out code begins
...
```

# Expand level

在这个章节我们会将 brainfuck 的 s-expression 形式转换为真正的 Racket
代码来执行. 我们在 Intro 中提到的 brainfuck 代码可以用如下的 s-exp
来表示:

``` racket
(plus)(plus)(plus)(plus)(plus) (plus)(plus)(plus)(plus)(plus)
(brackets
 (greater-than) (plus)(plus)(plus)(plus)(plus) (plus)(plus)
 (greater-than) (plus)(plus)(plus)(plus)(plus) (plus)(plus)
 (plus)(plus)(plus) (greater-than) (plus)(plus)(plus)
 (greater-than) (plus) (less-than)(less-than)(less-than)
 (less-than) (minus))
(greater-than) (plus)(plus) (period)
(greater-than) (plus) (period)
(plus)(plus)(plus)(plus)(plus) (plus)(plus) (period)
(period) (plus)(plus)(plus) (period)
(greater-than) (plus)(plus) (period)
(less-than)(less-than) (plus)(plus)(plus)(plus)(plus)
(plus)(plus)(plus)(plus)(plus) (plus)(plus)(plus)(plus)(plus)
(period) (greater-than) (period)
(plus)(plus)(plus) (period)
(minus)(minus)(minus)(minus)(minus)(minus)(period)
(minus)(minus)(minus)(minus)(minus)(minus)(minus)(minus)
(period)(greater-than) (plus) (period) (greater-than) (period)
```

需要做的就是将这些节点展开为真实的代码, 直接定义宏就行了.

``` racket
(define current-state (make-parameter (new-state)))

(define-syntax-rule (greater-than)
                    (increment-ptr (current-state)))

(define-syntax-rule (less-than)
                    (decrement-ptr (current-state)))

(define-syntax-rule (plus)
                    (increment-byte (current-state)))

(define-syntax-rule (minus)
                    (decrement-byte (current-state)))

(define-syntax-rule (period)
                    (write-byte-to-stdout (current-state)))

(define-syntax-rule (comma)
                    (read-byte-from-stdin (current-state)))

(define-syntax-rule (brackets body ...)
                    (loop (current-state) body ...))
```

这里的 `current-state` 就是一个 `parameter`. 如果我们运行了多个
brainfuck 代码, 我们是不希望多个代码间互相干扰的. 也就是说, 每一个
brainfuck module 需要有一个自己的 state.
显然我们的实现会让所有的代码使用相同的 `current-state` .
为了解决这个问题, 我们可以使用前文提到的 `#%module-begin` 和
`parameterize`, 在每一个 brainfuck module 隐式插入 `parameterize` ,
保证每一个 module 的 state 相互隔离.

``` racket
(define-syntax-rule (bf-module-begin body ...)
  (#%plain-module-begin
    (parameterize ([current-state (new-state)])
      body ...)))
```

这样, 一个模块的代码被完全包裹在了 `parameterize` 的 scope 中,
无需再担心多个模块互相干扰了.

我们并不需要定制函数调用等等, 所以就不用管其他三个了.
不要忘了导入之前的定义和导出.

``` racket
#lang racket
(require "semantics.rkt") ;; 我们之前写的语义部分
(provide greater-than
         less-than
         plus
         minus
         period
         comma
         brackets
         (rename-out [bf-module-begin #%module-begin]))

;; our code begins
...
```

我们使用 Racket 提供的元语言 `#lang s-exp "language.rkt"` 并用
language.rkt 进行展开就可以运行这些 s-expressions 了.

# Reader

## 实现 Parser

前面我们已经可以执行生成的 s-exp 了, 我们现在来考虑如何将 brainfuck
代码转换为 s-exp.

Racket 中使用 [syntax
object](https://docs.racket-lang.org/guide/stx-obj.html#%28tech._identifier._syntax._object%29)
来表示语法对象. 其实 syntax object 就是 s-exp ,
不过附加了对象在文件中的位置信息, 以及其他的信息. 我们可以使用
`datum->syntax` 来将 s-exp 转换为 syntax object.

Racket 本身提供了一个递归下降 (recursive descent) 的解析器, 可以通过
`readtable` 进行扩展. 但是 brainfuck 不需要太多的语法解析,
所以我们自己手写一个也问题不大.

``` racket
#lang racket

(provide parse-expr)

(define (parse-expr src in)
  ;; Find out where we are now
  (define-values (line column position) (port-next-location in))
  ;; Read the next char
  (define next-char (read-char in))

  ;; Wrap raw s-exp as syntax object
  (define (decorate sexp span)
    (datum->syntax #f sexp (list src line column position span)))

  (cond
    [(eof-object? next-char) eof]
    [else
      (case next-char
        ;; Generate s-expression
        [(#\>) (decorate '(greater-than) 1)]
        [(#\<) (decorate '(less-than) 1)]
        [(#\+) (decorate '(plus) 1)]
        [(#\-) (decorate '(minus) 1)]
        [(#\,) (decorate '(comma) 1)]
        [(#\.) (decorate '(period) 1)]

        ;; Recursive descent
        ;; Find out all s-exps in brackets
        [(#\[)
         (define elements (parse-exprs src in))
         (define-values (l c tail-position) (port-next-location in))
         (decorate `(brackets ,@elements)
                   (- tail-position position))]

        [else (parse-expr src in)])]))

;; parse [ ... ]
(define (parse-exprs source-name in)
  (define peeked-char (peek-char in))
  (cond
    [(eof-object? peeked-char)
     (error 'parse-exprs "Expected ], but read eof")]
    [(char=? peeked-char #\])
     (read-char in)
     empty]
    [(member peeked-char (list #\< #\> #\+ #\- #\, #\. #\[))
     (cons (parse-expr source-name in)
           (parse-exprs source-name in))]
    [else
      (read-char in)
      (parse-exprs source-name in)]))
```

用于解析 `[ ... ]` 的 `parse-exprs` 使用了尾递归. `cons`
的作用就是将当前的解析到的符号和之后尾递归解析到的符号连接在一起,
作为一个整体返回. 可以当作这样的伪代码:

    while xxx {
      // 计算出当前的语法树节点
      ast.push_back(node)
    }

    return ast

## 在 brainfuck 中加入解析器

将我们实现的 parser 作为 module language 的解析器, 然后再用之前的 expand
level 代码就可以运行了 brainfuck 代码了. 幸运的是, Racket 为我们提供了
`#lang s-exp syntax/module-reader` , 我们可以在这个文件中指定用于 expand
level 的代码和使用的 `read` 和 `read-syntax` .

``` racket
#lang s-exp syntax/module-reader
bf/language

#:read my-read
#:read-syntax my-read-syntax

(require "../parser.rkt")

(define (my-read in)
  (syntax->datum (my-read-syntax #f in)))

(define (my-read-syntax src in)
  (parse-expr src in))
```

第二行指出了 expand level 的模块. 因为语法限制, 这里只能使用字母, 括号,
下划线等等, 并不能使用双引号来引用文件路径,
所以我们必须安装我们的这个包, 使用
`$ raco pkg install --link path/to/bf` 安装本地包, 这里的 `language`
即指我们之前写的 expand 代码. 然后我们 `require` 这个文件,
就可以调用我们自己的parser 和 expander 来执行代码了.

## Hello world!

为了能够在源文件开头使用 `#lang bf` 指定语言, 我们需要将这个文件放到
`path/to/bf/reader.rkt` 处, 这样 Racket 就可以找到我们的语言了.
现在我们可以成功运行 \*Intro 的代码了. Hello World!

# 结语

虽然 Racket 是一门 JIT 语言, 但是我们在这里实现的 brainfuck
的效率还有很大的提升空间(如何优化可以参考 brainfudge,
本文主要的参考来源). 之前学习了 Scheme 和 Racket , 总想写点什么. 在读到
Racket Guide 第十七章后便决定写一个基于 Racket 的语言, 最后决定为
brainfuck. 参照教程磕磕碰碰写完后, 这让我再一次领略到了 Lisp 的魅力
(第一次是完全尝试用递归实现了算法题). 之后准备看看 *Hackers & Painters*
.

# 参考

[The Racket Guide :: Creating
Languages](https://docs.racket-lang.org/guide/languages.html)

[brainfudge](https://github.com/dyoo/brainfudge)

学习 Racket 可以参考

[The Racket Guide](https://docs.racket-lang.org/guide/index.html)

[Teach Yourself Scheme in Fixnum Days
中译](http://songjinghe.github.io/TYS-zh-translation/)
