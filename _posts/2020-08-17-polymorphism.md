---
layout: post
title: "Typeclass vs Trait vs Interface/聊聊多态"
categories:
  - CS
tags:
  - PL
toc: true
toc_sticky: true
toc_label: 目录
---

*Disclamer：由于本人才疏学浅，并且本文涉及的话题内容过多，肯定有不全面甚至错误。本文既不学术也不实践，仅为个人阶段的学习理解，写的也比较随意。如有错误欢迎指正。*

我的 PL 学习路径是 Python → C++（C with class）→ Coq → Haskell  → Rust → Java。学 Haskell 的时候学到了 Typeclass，觉得太妙了，第一次看见有这么好的东西可以用来减少重复代码。不过随即觉得其实和 C++ 的抽象类挺像。Coq 里也有 Typeclass，但我没怎么用过。学 Rust 发现了 Trait，当时就隐隐觉得和 Typeclass 很像，不过表达力貌似要比 Typeclass 弱一些。稍微研究了一下也没完全搞清楚区别，甚至还发现了 Java 里的 Interface。

那么自然就引出了这个问题：Interface、Trait、Typeclass……这些概念之前都有什么联系和区别？

## 分别介绍

首先我们先简单介绍/复习一下每个语言中的概念的功能。

### Haskell: Typeclass

From [https://www.seas.upenn.edu/~cis194/spring13/lectures/09-functors.html](https://www.seas.upenn.edu/~cis194/spring13/lectures/09-functors.html)

```haskell
class Eq a where
  (==) :: a -> a -> Bool
  (/=) :: a -> a -> Bool
  x /= y = not (x == y)   -- default implementation

-- Let’s look at the type of (==)
(==) :: Eq a => a -> a -> Bool

data Foo = F Int | G Char

instance Eq Foo where
  (F i1) == (F i2) = i1 == i2
  (G c1) == (G c2) = c1 == c2
  _ == _ = False

data Foo' = F' Int | G' Char
  deriving (Eq, Ord, Show)

-- Functor class abstracts over types of kind * -> *.
class Functor f where
  fmap :: (a -> b) -> f a -> f b

instance Functor Maybe where
  fmap _ Nothing  = Nothing
  fmap h (Just a) = Just (h a)

instance Functor [] where
  fmap _ []     = []
  fmap f (x:xs) = f x : fmap f xs
```

### Rust: Trait

From [https://doc.rust-lang.org/rust-by-example/trait.html](https://doc.rust-lang.org/rust-by-example/trait.html)

```rust
struct Sheep { naked: bool, name: &'static str }

trait Animal {
    // Static method signature; `Self` refers to the implementor type.
    fn new(name: &'static str) -> Self;

    // Instance method signatures; these will return a string.
    fn name(&self) -> &'static str;
    fn noise(&self) -> &'static str;

    // Traits can provide default method definitions.
    fn talk(&self) {
        println!("{} says {}", self.name(), self.noise());
    }
}

impl Sheep {
    fn is_naked(&self) -> bool {
        self.naked
    }

    fn shear(&mut self) {
        if self.is_naked() {
            // Implementor methods can use the implementor's trait methods.
            println!("{} is already naked...", self.name());
        } else {
            println!("{} gets a haircut!", self.name);

            self.naked = true;
        }
    }
}

// Implement the `Animal` trait for `Sheep`.
impl Animal for Sheep {
    // `Self` is the implementor type: `Sheep`.
    fn new(name: &'static str) -> Sheep {
        Sheep { name: name, naked: false }
    }

    fn name(&self) -> &'static str {
        self.name
    }

    fn noise(&self) -> &'static str {
        if self.is_naked() {
            "baaaaah?"
        } else {
            "baaaaah!"
        }
    }
    
    // Default trait methods can be overridden.
    fn talk(&self) {
        // For example, we can add some quiet contemplation.
        println!("{} pauses briefly... {}", self.name, self.noise());
    }
}
```

在 Rust 中，`Option<T>`, `Vec<T>`, `Result<T>` 都有自己的 `map`，但却不能更进一步，写出一个更一般的 `map`，因为 Rust 没有 higher kinded `*->*` , 写不出 `Functor`。

不过 Rust 明明可以 `impl<T> Trait for Vec<T>` ，为什么说写不出 `Functor` 呢？这是因为 `fmap :: (a -> b) -> f a -> f b` 这里面的 `f a` , `f b` 这个类型写不出来！当然，可以用山寨的方法，强行多加一个辅助参数来实现效果，比如 `impl<A,B> Functor<A,B,Vec<B>> for Vec<A>` 。我觉得这样做的问题在于，没法保证多加的辅助参数被正确使用，可能会写出奇奇怪怪的东西。

From [https://www.reddit.com/r/rust/comments/2av5tv/why_does_rust_not_have_a_functor_trait/](https://www.reddit.com/r/rust/comments/2av5tv/why_does_rust_not_have_a_functor_trait/)

```rust
// 正宗的 Functor, 这个写不出来。Self 只是 Vec<T>，而不是高阶类型 Vec
trait Functor<A,B> {
    fn fmap<A,B>(&self, f: fn(&A) -> B) -> Self<B>;
}

// 山寨 Functor，多加了辅助参数 C
trait Functor<A,B,C>{
  fn fmap(&self, f : fn(&A) -> B) -> C;
}
// nonsense
impl<B> Functor<i32,B,B> for i32 {
  fn fmap(&self,f : fn(&i32) -> B) -> B {
    f(self)
  }
}
// 正确使用的山寨 Functor
impl<A,B> Functor<A,B,Vec<B>> for Vec<A> {
  fn fmap(&self, f : fn(&A) -> B) -> Vec<B>{
    let mut v = Vec::new();
    for e in self.iter() {
      v.push(f(e));
    }
    v
  }
}

fn main(){
  let i = 10.fmap(|x| x * 2);
  let v = vec![1,2,3,4].fmap(|x| x * 10);
  println!("{}", i); // 20
  println!("{:?}", v); // [10, 20, 30, 40]
}
```

### C++/Java：abstract class/interface

略~

## 多态

现在我们可以暂时放下各个具体语言中的具体特性，从一个一般的角度来看问题。其实这些语言特性都可以回归到一个很大的话题：多态。

首先多态是什么？按照字面理解，就是多种形态。什么东西有多种形态？我的理解是：一个名字/记号（专业的说法可以叫 term）有多种形态/行为。In the context of type systems, polymorphism allows *a single term* to have *several types*.

[维基百科中的 polymorphism 词条](https://en.wikipedia.org/wiki/Polymorphism_(computer_science))中说有三种最被熟知的多态：

- 特设（ad hoc）多态
- 参数化多态
- 子类型多态

### Ad hoc polymorphism

参数化（C++模板/Java 泛型）和子类型（C++虚函数/Java 默认）我们都比较熟悉，那么这个 ad hoc 是什么鬼？看看词典的解释：

> An *ad hoc* activity or organization is done or formed only because a situation has made it *necessary* and is *not planned* in advance.

好像有点懂，但是这怎么和多态联系起来？Wiki 中的定义：

> *Ad hoc polymorphism*: defines a common interface for *an arbitrary set of individually specified types*.

？？？完全不懂在说啥。点进词条再看看更详细的解释吧：

> In programming languages, *ad hoc polymorphism* is a kind of polymorphism in which polymorphic functions can be **applied to arguments of different types**, because a polymorphic function can **denote a number of distinct and potentially heterogeneous implementations depending on the type** of argument(s) to which it is applied.
> 
> The term ad hoc in this context is not intended to be pejorative; it refers simply to the fact that this type of polymorphism is not a fundamental feature of the type system. This is in contrast to parametric polymorphism, in which polymorphic functions are written without mention of any specific type, and can thus apply a single abstract implementation to any number of types in a transparent way.

好家伙，原来就是重载。parametric 是不同的类型不做区分统一对待，而 ad-hoc 就是不同类型表现出不同的行为。一个函数名字不是给某个类型专门留的，而是类型匹配上了哪个就用哪个，这个意义上的 ad-hoc。具体可以想到这两个例子：

- `foo(int)` , `foo(double, int)`，函数重载，不同的函数签名（类型）调了不同的实现可以算 ad-hoc
- 两个类型 A 和 B 都实现了 Interface Ord，`x.cmp(y)` ，根据 x 的类型调 A 或 B 的 `cmp` ，也算 ad-hoc

可以这么理解：先定义一个 `foo: Int → Int` ，再定义一个 `foo: String → Double` ，于是 `foo` 这个名字就变成了一个 Union type `foo: (Int → Int)∪(String→Double)` 。在这个意义上，直接新定义函数重载和实现接口确实没什么区别。

### Bounded polymorphism

但我们都知道，Interface（以及类似物）除了让你多出来一个函数可以用以外，更强大的用法是在函数签名里限制参数必须实现这个接口。`foo<T>(arg: T) where T: Ord` ，这类似于 parametric，不限定于 any specific type，但是对类型又有约束；又类似于 subtyping，使用了一个公共类型 specify 类型需要满足的行为。这就是我们熟悉的设限多态（bounded polymorphism），它可以认为是 an interaction of parametric polymorphism with subtyping。它与 subtyping 的不同之处我觉得就是只关注类的公共行为，面向协议/面向接口，不关注（也不存在）子类型继承关系。众所周知滥用继承（尤其是多重继承）很容易使得类的关系过分复杂，如果我们只定义协议/接口，使用设限多态，那么类的关系就会简化许多，当然相应的也失去了类之间的层级关系，从而可能失去一些重用代码的可能。

### Row polymorphism/duck typing

鸭子类型是我们不用给参数加上任何类型标注，不规定类型的要求，而只看给的类型**实际**满不满足要求（拥有 methods、properties）。Row polymorphism 是静态类型的类似鸭子类型的东西，我不太了解，就不多谈了。

### 静态/动态多态

除了上面说的这些分类，还有另外一种分类方式：静态/动态多态。

这种分类方式属于实现的细节，并不是功能的区别。但是通常 ad hoc 和 parametric 是静态的，例如函数重载选实现、C++模板特化、Java 类型擦除都是编译期做的；subtyping 通常是动态的，例如运行时查虚函数表。但这并非绝对。

### 总结一下

我们说了 ad hoc、parametric、subtyping、bounded、duck typing 这几种多态，那回过头看看，现在能区分 trait、typeclass、interface 了吗？其实我觉得这三类都是 ad hoc + bounded polymorphism……所以还是不能区分。

我们再抛开多态的分类，更一般地想想，减少重复代码有哪些需求：

1. 默认实现
2. 参数化/泛型
3. 给泛型加约束
4. 给既有类型增加功能
5. 高阶类型/类型构造器

Haskell 的 Typeclass 这些都有，Rust 的 Trait 少了 5，Java 的 Interface 少了 5 和 4。

第 5 条高阶类型是比较高级的类型系统的要求了，大部分语言没有也难免。

而 4 给既有类型增加功能通常是 FP 的特性。FP 的类型定义后就固定了，但可以容易地给类型加函数、加入 typeclass，而 OOP 则是类型比较容易扩展（继承），但是想给已有类型加新的功能比较困难。

## 总结

说了半天，其实还是没有把这些东西完全分的很清楚。真要研究/实践的话，还是得具体语言具体分析，深入功能的细节乃至实现才能很好的区分。正如很多人所说，脱离具体语言谈这些毫无意义。但我并不完全赞同这种话，我觉得能对有哪些多态的类别、有哪些功能需求有 general 的了解、区分，对了解 PL 的共性与区别、写出更优雅的代码还是很有帮助的。

## 参考

- [https://en.wikipedia.org/wiki/Polymorphism_(computer_science)](https://en.wikipedia.org/wiki/Polymorphism_(computer_science))
- [Protocol，Interface，Trait，Concept，TypeClass之间的关系和区别？](https://www.zhihu.com/question/314434687)
- [Difference between OOP interfaces and FP type classes](https://stackoverflow.com/questions/8122109/difference-between-oop-interfaces-and-fp-type-classes)