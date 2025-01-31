# 前言

这系列文章介绍用Rust语言从零开始实现一个Lua解释器。

Rust语言个性鲜明，也[广受欢迎](https://survey.stackoverflow.co/2022/?utm_source=so-owned&utm_medium=announcement-banner&utm_campaign=dev-survey-2022&utm_content=results#section-most-loved-dreaded-and-wanted-programming-scripting-and-markup-languages)，然而学习曲线陡峭。我在读完[《Rust程序设计语言》](https://kaisery.github.io/trpl-zh-cn/)并写了些练习代码后，深感必须通过一个较大的项目实践才能理解和掌握。

[实现一个Lua解释器](http://lua-users.org/wiki/LuaImplementations)就很适合作为这个练习项目。因为其规模适中，足够涉及Rust的大部分基础特性而又不至于难以企及；[目标明确](https://www.lua.org/manual/5.4/)，无需花费精力讨论需求；另外Lua语言本身也是一门设计优秀且应用广泛的语言，实现一个Lua解释器不仅可以实践Rust语言技能，还能深入了解Lua语言。

这个系列的文章记录了在这个项目中的学习和探索过程。与其他从零开始[Build your own X](https://build-your-own-x.vercel.app/)的项目类似，这个项目也有明确的大目标、未知的探索过程以及持续的成就感，但也有一些不同之处：

- 其他项目的作者大都在相关领域浸淫多年，而我的工作并不是编程语言或编译原理方向，对于实现一个解释器而言并没有完整的理论知识，纯粹摸着石头过河。不过凡事要往好处想，这也提供了一个真正的初学者视角。

- 其他项目大都是以学习或教学为目的，去繁从简，实现一个只具备最基本功能的原型。而我的目标是实现一个生产级别的Lua解释器，追求稳定、完整、和性能。

另外，由于项目的初衷是学习Rust语言，所以文章中也会有一些Rust语言的学习笔记和使用心得。

## 内容

内容安排如下。第1章实现一个最小的、只能解析 `print "hello, world!"` 语句的解释器。虽然简单，但是包括了解释器的完整流程，构建了基本框架。后续章节就以Lua语言特性为出发点，在这个最小解释器上逐渐地增加功能。

第2章介绍编程语言中最基本的类型和变量的概念。第3章以完善字符串类型为目标，介绍Rust语言的几个特性。第4章实现Lua中的表结构，并引入语法分析中关键的ExpDesc概念。第5章是繁琐的数值计算。

第6章控制结构，事情开始变得有趣起来，根据判断条件在字节码间跳来跳去。第7章介绍逻辑运算和关系运算，通过特定的优化跟上一章的控制结构结合起来。

第8章介绍函数，函数的基本概念和实现是比较简单的，但可变参数和多返回值要求对栈做精细管理。第9章介绍的闭包是Lua语言中的一个强大特性，这其中的关键是Upvalue及其逃逸。

每个功能都是按需设计，而不是先知般的一步到位。比如关于条件跳转指令，最开始为了支持[`if`语句](./ch06-01.if.md)，添加`Test(u8, u16)`字节码，其语义是如果第1个关联参数的值为**false**则**向前**跳转第2个关联参数代表的距离；然后为了支持[`while`语句](./ch06-03.while_break.md)，需要**向后**跳转，则把第2个关联参数从`u16`改成`i16`类型，用负数代表向后跳转；再然后为了支持[逻辑运算](./ch07-01.logical_in_condition.md)，判断条件为**true**或**false**都可能需要跳转，则用`TestAndJump`和`TestOrJump`两个字节码代替`Test`字节码。由此，按照我们自己的学习和发展路径，也产生了一套跟Lua官方实现版本略微不同的字节码。

每一章都从Lua功能特性出发，先讨论如何设计，再介绍具体实现。不仅要讲清楚“怎么做”，更重要的是讲清楚“为什么这么做”，尽量避免写成代码阅读笔记。不过为了实现完整的Lua特性，肯定会有部分文章很无聊，尤其是前面几章。读者可以先浏览相对有趣的[字符串类型的定义](./ch03-01.string_type.md)和[Upvalue的逃逸](./ch09-02.escape_and_closure.md)这两节，以判断本系列文章是否符合口味。

每一章都有完整的[可运行代码](https://github.com/WuBingzheng/build-lua-in-rust/tree/main/listing)，并且每一章的代码都是基于前一章的最终代码，保证整个项目是连续的。每个小节对应的代码改动都集中在两三个commit中，可以通过git来查看变更历史。开头的章节在介绍完设计原理后，基本还会逐行解释代码；到后面只会解释关键部分的代码；最后两章基本上就不讲代码了。

目前这些章节只是完成了Lua解释器最核心的部分，而距离一个完整的解释器还差很多。[未完待续](./TO_BE_CONTINUED.md)一节罗列了部分未完成的功能列表。

文章中并不会介绍Lua和Rust的基本语法，我们预期读者对这两门语言都有基本的了解。其中对Lua语言自然是越熟悉越好。对Rust语言则无高要求，只要读过《Rust程序设计语言》，了解基本语法即可，毕竟这个项目的初衷就是为了学习Rust。另外，一提到实现一门语言的解释器就会让人想起艰深的编译原理，但实际上由于Lua语言很简单，并且有Lua官方实现的解释器代码作为参考，这个项目不需要太多理论知识，主要是工程实践为主。

由于我在编译原理、Lua语言、Rust语言等各方面技术能力所限，项目和文章中必定会有很多错误；另外我的语言表达能力很差，文章中也会有很多词不达意或语句不通之处，都欢迎读者来项目的[github主页](https://github.com/WuBingzheng/build-lua-in-rust)提issue反馈。