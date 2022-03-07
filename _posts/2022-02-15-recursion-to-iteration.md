---
layout: post
title: "递归改写成迭代的通用方法"
categories:
  - CS
toc: true
toc_sticky: true
toc_label: 目录
---

> 递归是像呼吸一般自然的事情。
>  
> ——罗宸[《谈递归（一）：递归的五种定式》](https://zhuanlan.zhihu.com/p/84452538)

喜欢刷 LeetCode 或者刚学数据结构的同学一定很熟悉二叉树的各种遍历，包括递归和迭代写法。显然递归写起来更简洁自然，迭代则需要一些 trick，尤其是后序遍历，入栈出栈，连续往左往右什么的，刚学难免有点挠头。总有时候，我们被迫要用非递归的方式实现，比如面试官强力要求，或者要实现迭代器（如 [LeetCode 173. 二叉搜索树迭代器（中序遍历）](https://leetcode-cn.com/problems/binary-search-tree-iterator/)）。这可咋办呢？

有一个通用的方法，可以非常容易地把任意的递归代码重写成迭代的代码，这就好比在呼吸受限的情况下，还可以戴上氧气面罩。这个方法就是：手动模拟递归执行，或者说手动模拟函数调用栈。

## 函数调用栈

为了理解这个方法，首先需要知道一点计算机的运行原理。其实很好懂，这里简单介绍一下。由于有各种控制语句，还有函数调用，代码的执行不是顺序的，那 CPU 怎么知道下一条指令是什么？答案是维护了一个寄存器 PC (program counter)，指向下一条要执行的指令。对于控制语句，我跳走了就没事了不需要回来了，所以只需要修改 PC，但是函数调用完了还需要回来，这可怎么办呢？答案是每当要进行函数调用时我就保存现场，把 PC 存起来就行了。这就是函数调用栈的作用。

```
函数调用栈（这个图是瞎画的，仅示意）
-----------------------
(many frames ...)
-----------------------
(last frame, caller)
parameters
pc (我最后执行到哪儿了，同时也是下一个函数的返回地址)
-----------------------
(current frame, callee)
parameters
-----------------------
```

## 手动模拟函数调用栈

接下来就进入正题。直接上代码说明问题。

一个一般的递归函数。

```python
def foo(param):   # pc = 0
    foo(param_1)  # pc = 1
    foo(param_2)  # pc = 2
    # ...
    foo(param_i)  # pc = i
    do_some_thing(param)
    # ...
    foo(param_n)
```

转化成迭代的代码。

```python
class StackFrame:
    def __init__(self, param, pc):
        self.param = param
        self.pc = pc

def foo_iter(param):
    stack = []
    stack.append(StackFrame(param, 0))

    # 模拟 CPU 执行
    while len(stack) != 0:
        frame = stack[-1]

        if frame.pc == 0:
            # 下一条指令是
            # foo(param_1) # pc = 1
            frame.pc = 1
            stack.append(StackFrame(param_1, 0))
        elif frame.pc == 1:
            # 下一条指令是
            # foo(param_2) # pc = 2
            frame.pc = 2
            stack.append(StackFrame(param_2, 0))
        # ...
        elif frame.pc == i:
            # 下两条指令是
            # do_some_thing(param)
            # foo(param_i+1)
            do_some_thing(param)
            frame.pc = i + 1
            stack.append(StackFrame(param_i+1, 0))
        # ...
        else:
            # 这个函数执行完了，要返回了，销毁它的栈帧
            # 也可以在上面的最后一个分支直接就 pop 了
            stack.pop()
```

## 例题：二叉搜索树迭代器（中序遍历）

Ok，那我们上一道例题，[LeetCode 173. 二叉搜索树迭代器（中序遍历）](https://leetcode-cn.com/problems/binary-search-tree-iterator/)。

首先中序遍历的递归写法

```python
def inorder_traversal(root):
    if root is None:
        return
    if root.left is not None:
        inorder_traversal(root.left)
    print(root.val) 
    if root.right is not None:
        inorder_traversal(root.right)
```

重写成迭代

```python
def inorder_traversal(root):
    stack = []
    stack.append(StackFrame(param, 0))

    while True:
        frame = stack[-1]
        node = frame.param

        if frame.pc == 0:
            frame.pc = 1
            if node.left != None:
                stack.append(StackFrame(node.left, 0))
        elif frame.pc == 1:
            stack.pop()
            print(node.val)
            if node.right != None:
                stack.append(StackFrame(node.right, 0))
```

再重写成迭代器，其实就是把上面迭代的过程中的**状态**（这里就只有 stack）放进 Iterator 类里面就行了。

```python
class BSTIterator:

    def __init__(self, root: TreeNode):
        self.stack = []
        if root != None:
            self.stack.append(StackFrame(root, 0))

    def next(self) -> int:
        while True:
            frame = self.stack[-1]
            node = frame.param

            if frame.pc == 0:
                frame.pc = 1
                if node.left != None:
                    self.stack.append(StackFrame(node.left, 0))
            elif frame.pc == 1:
                self.stack.pop()
                if node.right != None:
                    self.stack.append(StackFrame(node.right, 0))
                return frame.param.val

    def hasNext(self) -> bool:
        return len(self.stack) != 0
```

就是这样（又水了一篇），如果有不懂的地方欢迎评论提问🥰
