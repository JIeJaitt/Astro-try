---
title: 'Go Struct Pointer Usage'
description: 'Go Struct Pointer Usage.'
pubDate: 'Jul 01 2024'
tags:
  - Go
badge: 还没写好
---

## Go语言一般把一个结构体的指针作为结构体的成员，这种一般是做什么呢，很多人都分不清楚

在 Go 语言中，将一个结构体的指针作为另一个结构体的成员，通常有几个目的：

1. **减少内存使用**：使用指针可以避免在结构体中直接存储另一个结构体的完整副本，这样可以节省内存，特别是当被引用的结构体较大时。

2. **避免复制**：当结构体通过值传递时，Go 会复制整个结构体。如果结构体中包含其他大型结构体，则复制操作可能会很昂贵。使用指针可以避免这种复制，因为只复制指针本身，而不是它指向的数据。

3. **允许修改原始数据**：当你想在不同的地方修改同一个实例的数据时，使用指针是很有用的。如果结构体的成员是指针，那么对这个成员的修改会反映到所有引用这个指针的地方。

4. **动态数据结构**：在实现像链表、树等动态数据结构时，节点间的链接通常通过指针来实现。这允许动态地添加、删除或修改结构中的节点。

5. **实现关联和共享**：通过使用指针，可以在多个结构体实例或组件间建立和维护复杂的关系，例如，在父子、观察者等模式中。

举个简单的例子，假设你有一个代表图书的结构体和一个代表作者的结构体。每本书都有一个作者，但多本书可能是由同一作者编写的。这种情况下，使用指针作为图书结构体中的成员可以有效地引用作者结构体，不需要在每本书中都复制作者的全部信息。这样做还可以在作者的信息更新时，让所有引用这位作者的书籍都能访问到更新后的信息。

## 结构体指针类型可以访问结构体的结构体方法和指针方法，但是结构体类型只能访问结构体方法 这样对吗？其实这样理解是大概是对的，但是

是的，你的理解是正确的。在 Go 语言中，对于结构体和其方法的调用规则如下：

1. **结构体类型（值接收者）**：如果一个方法的接收者是结构体类型（即使用结构体本身作为接收者），那么这个方法可以被结构体实例（值类型）和指向该结构体的指针调用。

2. **结构体指针类型（指针接收者）**：如果一个方法的接收者是结构体指针（即使用结构体的指针作为接收者），那么这个方法只能被结构体的指针调用。不过，由于 Go 语言的便利性设计，即使你有一个结构体的实例，当你尝试调用一个指针接收者的方法时，Go 会自动取该实例的地址来调用该方法。

简而言之：
- 结构体类型（值）可以调用所有的方法（无论方法是定义为值接收者还是指针接收者）。
- 结构体指针可以调用所有的方法，包括那些专门为结构体指针定义的方法。

这种设计允许更灵活的方法定义，同时在使用上提供了便利。这样，开发者可以根据需要选择使用值接收者还是指针接收者，通常基于是否需要在方法中修改接收者的数据或考虑到性能（如传递大型结构体时使用指针接收者以避免数据复制）。

## 而且slice，map啥的都是引用类型吗？

对的，slice（切片）和 map（映射）在 Go 语言中都是引用类型。这意味着当你将它们作为参数传递给函数或方法时，传递的是对底层数组（对于切片）或对底层数据结构（对于映射）的引用，而不是整个数据结构的副本。

这里是关于这两种类型的一些详细解释：

1. **Slice（切片）**：切片是一个动态数组的引用。它本身并不存储任何数据，只是对底层数组的一个视图或窗口。因此，当你修改切片中的元素时，实际上是在修改底层数组中的相应元素。同样，如果你将切片传递给函数，函数内部对切片的修改也会影响原始切片。

2. **Map（映射）**：映射是一种键值对集合，它也是引用类型。映射内部的数据结构可以动态增长，并且可以方便地通过键来存取数据。当你将映射传递给一个函数时，如同切片一样，你传递的是对这个映射的引用。因此，对映射的任何修改都会反映到原始映射上。

由于 slice 和 map 都是引用类型，使用它们时需要考虑如何处理并发访问和修改的问题，尤其是在多协程的环境中，可能需要使用锁或其他同步机制来避免竞争条件。