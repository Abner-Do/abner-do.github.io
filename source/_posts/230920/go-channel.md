---
title: go channel
date: 2023-09-20 00:13:12
tags:
---

Channel 是 Golang 在语言级别提供的 goroutine 之间的通信方式，可以使用 channel 在两个或多个 goroutine 之间传递消息。Channel 是进程内的通信方式，因此通过 channel 传递对象的过程和调用函数时的参数传递行为比较一致，比如也可以传递指针等。使用通道发送和接收所需的共享资源，可以在 goroutine 之间消除竞争条件。

当一个资源需要在 goroutine 之间共享时，channel 在 goroutine 之间架起了一个管道，并提供了确保同步交换数据的机制。Channel 是类型相关的，也就是说，一个 channel 只能传递一种类型的值，这个类型需要在声明 channel 时指定。可以通过 channel 共享内置类型、命名类型、结构类型和引用类型的值或者指针。


基本语法
声明 channel 的语法格式为：
var ChannelName chan ElementType

与一般变量声明的不同之处仅仅是在类型前面添加了一个 chan 关键字。ElementType 则指明这个 channel 能够传递的数据的类型。比如声明一个传递 int 类型的 channel：