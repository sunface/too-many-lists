# too-many-lists

> 其它语言：兄弟，语言学了吗？来写一个链表证明你基本掌握了语法。
> 
> Rust 语言: 兄弟，语言精通了吗？来写一个链表证明你已经精通了 Rust！


上面的对话非常真实，我们在之前的章节也讲过[初学者学习 Rust 应该避免的坑](https://course.rs/sth-you-should-not-do.html#千万别从链表或图开始练手)，其中最重要的就是 - 不要写链表或者类似的数据结构！

而本章，你就将见识到何为真正的深坑，看完后，就知道没有提早跳进去是一个多么幸运的事。总之，在专题中，你将学会如何使用 Rust 来实现链表。

> 本书由 [RustTT 翻译小组](https://rusttt.com) 进行翻译，并对内容进行了一些调整，更便于国内读者阅读(原书虽然非常棒，但是在一些内容组织和文字细节上我觉得还是可以优化下的 ：D)。
> 
> [Learning Rust With Entirely Too Many Linked Lists](https://rust-unofficial.github.io/too-many-lists/)


## 目录
- [不太优秀的单向链表：栈](bad-stack/intro.md)
    - [数据布局](bad-stack/layout.md)
    - [基本操作](bad-stack/basic-operations.md)
    - [最后实现](bad-stack/final-code.md)

## 我们到底需不需要链表
经常有读者询问该如何实现一个链表，怎么说呢，这个答案主要取决于你的需求，因此并不是很好回答。鉴于此，我决定通过这本书来详尽的介绍该如何实现一个链表，大家应该都能从这本书中找到答案。

书中我们将通过实现 6 种链表来学习基本和进阶 Rust 编程知识，在此过程中，你能学到：

- 指针类型: `&`, `&mut`, `Box`, `Rc`, `Arc`, `*const`, `*mut`, `NonNull`
- 所有权、借用、继承可变性、内部可变性、Copy
- 所有的关键字：struct、enum、fn、pub、impl、use, ...
- 模式匹配、泛型、解构
- 测试、安装新的工具链、使用 `miri`
- Unsafe: 裸指针、别名、栈借用、`UnsafeCell`、变体 variance

是的，链表就是这么可怕，只有将这些知识融会贯通后，你才能掌握 :( 


> 事实上这本书中关于 Rust 语言的绝大部分知识都在 `Rust语言圣经`中有讲，因此除非特殊情况，我们将直接提供链接供大家学习，重点还是放在链表实现上

#### 创建一个项目
在开始前，先来创建一个项目专门用于链表学习：
```shell
> cargo new --lib lists
> cd lists
```

之后，我们会将每个一个链表放入单独的文件中，需要注意的是我们会尽量模拟真实的 Rust 开发场景：你写了一段代码，然后编译器开始跳出试图教你做事，只有这样才能真正学会 Rust，温室环境是无法培养出强大的 Rustacean 的。

#### 义务告知
首先，本书不是保姆式教程，而且我个人认为编程应该是快乐，这种快乐往往需要你自己发现而不是别人的事无巨细的讲解。

其次，我讨厌链表。链表真的是一种糟糕的数据结构，尽管它在部分场景下确实很有用：

- 对列表进行大量的分割和合并操作
- 无锁并发
- 要实现内核或嵌入式的服务
- 你在使用一个纯函数式语言，由于受限的语法和缺少可变性，因此你需要使用链表来解决这些问题

但是实事求是的说，这些场景对于几乎任何 Rust 开发都是很少遇到的，99% 的场景你可以使用 `Vec` 来替代，然后 1% 中的 99% 可以使用 `VecDeque`。 由于它们具有更少的内存分配次数、更低的内存占用、随机访问和缓存亲和特性，因此能够适用于绝大多数工作场景。总之，类似于 `trie` 树，链表也是一种非常小众的数据结构，特别是对于 Rust 开发而言。

> 本书只是为了学习链表该如何实现，如果大家只是为了使用链表，强烈推荐直接使用标准库或者社区提供的现成实现，例如 [std::collections::LinkedList](https://doc.rust-lang.org/std/collections/struct.LinkedList.html)

#### 链表有 O(1) 的分割、合并、插入、移除性能
是的，但是你首先要考虑的是，这些代码被调用的频率是怎么样的？是否在热点路径？ 答案如果是否定的，那么还是强烈建议使用 `Vec` 等传统数据结构，况且整个数组的拷贝也是相当快的！

况且，`Vec` 上的 `push` 和 `pop` 操作是 `O(1)` 的，它们比链表提供的 `push` 和 `pop` 要更快！我们只需要通过一个指针 + 内存偏移就可以访问了。

> 关于是否使用链表这个问题，Bjarne Stroustrup 有过非常深入的[讲解](https://www.youtube.com/watch?v=YQs6IC-vgmo)

但是如果你的整体项目确实因为某一段分割、合并的代码导致了性能低下，那么就放心大胆的使用链表吧。


#### 我无法接受内存重新分配的代价
是的，`Vec` 当 [`capacity`](https://practice.rs/collections/vector.html#capacity) 不够时，会重新分配一块内存，然后将之前的 `Vec` 全部拷贝过去，但是对于绝大多数使用场景，要么 `Vec` 不在热点路径中，要么 `Vec` 的容量可以提前预测。

对于前者，那性能如何自然无关紧要。而对于后者，我们只需要使用 `Vec::with_capacity` 提前分配足够的空间即可，同时，Rust 中所有的迭代器还提供了 `size_hint` 也可以解决这种问题。


当然，如果这段代码在热点路径，且你无法提前预测所需的容量，那么链表确实会更节省性能。

#### 链表更节省内存空间
首先，这个问题较为复杂。一个标准的数组调整策略是：增加或减少数组的长度使数组最多有一半为空，例如 capacity 增长是翻倍的策略。这确实会导致内存空间的浪费，特别是在 Rust 中，我们不会自动收缩集合类型。

但是上面说的是最坏的情况，如果是最好的情况，那整个数组其实只有 3 个指针大小(指针在 Rust 中占用一个 word 的空间，例如 64 位机器就是 8 个字节的大小)的内存浪费，或者说，没有浪费。

而且链表实际上也有内存浪费，例如链表中的每个元素都会占用额外的内存：单向链表浪费一个指针，双向链表浪费两个指针。当然，如果你的链表中每个元素都很大，那相对来说，这种浪费也微不足道，但是如果链表的元素较小且数量很多呢？那浪费的空间就相当可观了！

当然，这个也和使用的内存分配器有关( allocator )：对链表节点的分配和回收会经常发生，这样就不会浪费内存。

总之，如果链表的元素较大，你也无法预测数组的空间，同时还有一个不错的内存分配器，那链表确实可以节省空间！

#### 我在函数语言中一直使用链表
对于函数语言而言，链表确实非常棒，因为你可以解决可变性问题，还能递归地去使用，当然，可能还有一定的图方便的因素，因为链表不用操心长度等问题。

但彼之蜜糖不等于吾之蜜糖，函数语言的一些使用习惯不应该带入到其它语言中，例如 Rust。

- 函数语言往往将链表用于迭代，但是 Rust 中最适合迭代的数据结构是迭代器 `Iterator`
- 函数式语言的不可变对于 Rust 也不是问题
- Rust 还支持对数组进行切片以获取其中一部分连续的元素，而在函数语言中你可能得通过链表的 `head/tail` 分割来完成


其实，在函数语言中，我们也应该选择合适的数据结构来解决适合的场景，而不是*一根链表挂腰间，潇潇洒洒走天下*。


#### 链表适合构建并发数据结构
是这样的，如果有这样的需求，那么链表会非常合适！但是只有在你确实需要并发数据结构，且没有其它办法时，再考虑链表！

#### 链表非常适合教学目的
额... 这么说也没错，毕竟所有的编程语言课程都以链表来作为最常见的练手项目，包括本书也是服务于这个目的的。



