# 前言

在面对实际应用时，开发者最常用的两种为系统**行为**建模的方式是：面向状态（State-based）和面向过程（Process-based）。

产业应用最为广泛的面向对象编程（Object-Oriented）是一种混合模型。

在多数情况下，对象（object）是基于状态建模的，成员变量（member variable）是对象的状态，成员方法（method或member function）如果可以改变对象的状态，就是一种状态迁移，这些都符合状态模型的要素。同时绝大多数面向对象语言也支持创建进程（process，指操作系统进程）、线程（thread）、或者协程（fiber/coroutine），这些过程也被当作对象来处理，它们接受输入，产生输出（或者错误），这些符合过程模型的要素。还有一些对象则介于两者之间，例如流对象，它们本身是一个过程，但具有一些状态特征，例如一个写入流是否已经结束写入就是一种状态，写入流也可以抛出错误事件通知使用者其遇到的问题（但是并未自动终止）。

在Node.js中，面向对象不是主力的建模技术，虽然在很多代码细节中离不开它。绝大多OO语言是类型语言，它们为运行时的对象提供静态模板（struct或class），这些模板约定了运行时对象的结构（成员变量）和行为（方法），并以此为基础，提供了更好的代码重用能力（通过interface或inheritance）。

JavaScript ES6提供了class语法支持，但它只是一个语法糖，并没有改变JavaScript对象并不存在静态模板的本质；在JavaScript中所有对象都是运行时构建的，其结构和行为也都是运行时可以改变的，JavaScript是动态类型和duck-type语言。

所以在JavaScript编程中，我们回归到相对原始但是也更为纯粹的建模方式上。

众所周知，JavaScript语言的运行环境（run-time）是事件模型（Event Model），它带来在任何语言中实现事件模型都具有的优点：简单，没有Process间的同步问题，所有的函数都是run-to-completion的。

但这不意味着Javacript里的一切都必须是状态建模的，比如一个最基本的异步函数就是一个过程，绝大多数异步函数甚至没有留给调用者一个可以表示状态的句柄。事实上绝大多数通用编程语言都是支持多种编程范式（Multi-paradigm）的，包括面向过程、面向状态、面向对象，或者函数式编程，JavaScript也不例外。

在一定程度上，Event Model和Thread Model是对立的模型技术；Event Model和State-based可以看作同义词，Thread Model和Process-based也可以看作同义词。但是Event Model和Thread Model这一对术语更强调语言或者库提供的基础原语（primitive），例如是否提供Event Loop, Dispatcher, Listener/Handler等等Event Model需要的要素，或者提供创建Process, Thread, Fiber, Coroutine, Inter-process Communication等Thread Model需要的能力。

在我们的讨论中，并不强调语言的primitive，也不试图构建这样的primitive，我们更多的强调在概念层面的State和Process；在实现层面上，它们可能是function或者object，所以我们尽量使用State-based和Process-based这一组词汇。

如果在某些上下文下使用Event Model或者Thread Model更符合传统上对该问题的表述习惯，我们也会使用这两个术语，但在本文中应该被理解为State-based或Process-based的同义词。



# 模型收益

模型这个术语说起来有点吓人，它听起来象艰涩的数学。但是和绝大多数人的理解相反，数学的目的并不是把一个问题变得更难，相反，它的目的是消除在解决问题时，个体在智力（Intelligence）和直觉（Intuition）两方面的差异。



**好的模型应该是容易理解和运用的**

我们在学习几何的时候知道，使用尺规作图画出一个正十七边形是很困难的，需要高斯那样的异于常人的智商。但是同样的问题用解析几何来解决就不是那么难，解决问题只需要列出所有的方程解出来即可。

好的模型技术就应该象解析几何这样，它具有一套简单和容易理解的规则，也容易让使用者把它运用到需要解决的问题上，最终使用一种通用的方法得出问题的解法，它不需要异于常人的智力或者某种特殊的经验或感觉。



**好的模型应该帮助开发者减少错误的发生**

虽然大多数情况下我们通过直觉来书写代码，同样的在绝大多数情况下我们在心理构建的简单模型可以保证代码的正确性，但是在复杂的情况下，问题本身所需要的模型复杂度可能超过我们的心理模型的演算能力，有时候我们的某种直觉并不可靠，另外一些情况下一些复杂和微妙的场景没能在心理模型中完全展开，看清楚全部可能发生的细节。在这两种情况下，我们都说我们的心理模型是不严格和不完备的。

严格和完备的模型可能是繁琐的，但是作为开发者我们没法回避问题，在这种情况下，模型技术可以帮助我们思考问题，陈列出所有可能的情况，强制我们对所有的程序状态或行为给出定义，做到在设计上是严格和完备的。

同时好的模型应该具有良好的可调试性（Debuggability）和可测试性（Testability），即使我们在设计阶段犯了错误，忽视了某些可能发生的问题，它应该在调试或测试阶段，很容易让开发者看到错误发生时，系统的行为和状态，帮助开发者理解问题的根本原因（root cause），从而得到正确或更加合理的设计。


**好的模型应该易于构建程序**

一个停留在纸面上的模型只对数学家有用，对于开发者来说并没有多大价值。

好的模型应该易于开发者书写代码，它可能略微繁琐但不应该导致代码体积几何级数的膨胀和难以维护；使用模型编码可以对某类问题或者解决复杂问题有额外的收益，但是它不该强制开发者在所有的地方都使用一种模式编码。在开发语言（和库）的进化中，代码的简洁性、可读性和可维护性都是被极致追求的目标，如果一个模型技术会打破这些东西，那么无论它多好都不会被大范围的采纳。

在这里我们还需要特别强调一点，就是应用模型技术书写的代码，需要有良好的可组合性（Composability）。

Divide and Conquer是普适和经受了时间检验的应对复杂问题的工程方法。在传统的面向过程（Procedure-Oriented）的软件开发中，函数作为最基础的代码构件（building block），具有良好的可组合性，基础函数（primitive）可以调用函数，构成组合函数，程序可以从下之上通过组合（Composition）完成，无论在静态代码上还是动态运行上都是如此；同样的，在面向对象的软件开发中，对象（object）作为最基础的构件，也具有良好的可组合性，开发者可以从头书写class构建对象，可以通过对象组合构建更为复杂的对象。

从top-down的角度看，这种可组合性还带来另外一个重要的编程（模型）概念：封装（encapsulation）。在调用一个函数时，使用者并不清楚这个函数是一个基础函数，还是一个函数组合；同样的，当使用new语法构造一个对象时，使用者也不需要知道这个对象内部是否还具有其他的子对象。

如果设计模式能够提供更好的可组合性，这种黑盒封装的能力是非常重要的，它利于开发者灵活组合逻辑单元，重用代码，为软件提供更好的可维护性（maintainability）和可进化能力。



# 状态机

相对而言，大多数开发者更熟悉过程模型，写过多线程或多进程脚本程序的开发者都有朴素的心理模型。这篇文章的最初的几个版本也是从过程开始介绍的，但最终我发现先介绍状态模型更好，因为它可以帮助读者在阅读过程模型部分时，辅以基于状态的思考，这有助于理解和比较两种模型。

状态机模型本身很简单。它使用两个基础要素对对象的行为建模：

1. 一组状态（State），涵盖一个对象可能的全部状态；每个状态通常用成员变量表示，包括有结构的变量（对象）；
2. 一组事件（Event），所有可能触发对象**状态变化**的操作，都理解为Event；

一个对象的方法，如果它能改变对象的状态，需要被理解为Event；在JavaScript中，一个对象内部的异步过程的返回（callback），对这个对象而言，通常也认为是事件，如果他们改变了内部成员的话。

在设计阶段，开发者需要仔细定义一个对象的所有可能的状态组合和所有可能的事件（不是每个事件都适用于所有状态），然后逐一回答在每个状态下，如果一个可能的事件到来，这个对象的状态需要作何种改变，包括停留在这个状态下，或者迁移到新的状态。

如果状态和事件构成的矩阵里，针对每个状态/行为组合，开发者都给出了状态迁移的定义，我们称这个模型是完备的，否则这个对象具有未定义行为，是一种设计缺陷。实际的代码中通常不会没有行为，未定义行为是最常见的bug原因之一。

## State Pattern

大部分软件开发者对状态机的了解来自于那本著名的四人帮（Gang of Four）的设计模式（Design Pattern）。在这本书中，State Pattern作为一种重要的行为型模式提出，书中给出的例子是TCP协议的一个假想实现。

在这里我们不仔细介绍State Pattern，读者可以去看四人帮的书（书中代码是C++/Java）的例子，另外一个例子是我写过的一篇介绍在Node/JavaScript下状态机模式的[介绍文章](https://segmentfault.com/a/1190000011086405)，这个例子里是以JavaScript代码作为示例的。

State Pattern的写法不唯一，在这里我们不强调代码书写的模式，只强调用State/Event建模的方式。

> 有良好英文阅读能力和一点基础的集合论知识的开发者应该阅读wiki上关于finite state machine的文章：https://en.wikipedia.org/wiki/Finite-state_machine ，不需要仔细理解mealy machine和moore machine的区别，通常设计逻辑电路的硬件工程师或者设计控制系统的控制工程师更关心这些内容。其中状态图示和状态迁移表的对应关系，对软件开发者来说应该充分理解。



## State Diagram

在UML图示（Diagram）工具集中，有一个图称为State Diagram。它的原始出处来自David Harel教授在1987年发表的一篇文章：[Statecharts: A Visual Formalism for complex systems](http://www.inf.ed.ac.uk/teaching/courses/seoc/2005_2006/resources/statecharts.pdf)



> 这篇文章虽然是学术文章，但仍然非常值得开发者阅读；它不仅讨论了层级状态机，还讨论了与之对应的其他模型技术，包括Process模型，Petri Nets，等等，这些概念对完全理解领域内的相关知识，能够从各种角度理解本系列文章的内容，都是有帮助的。



Statecharts是针对State Machine的一个很具体的问题提出的。

数学上对State Machine的描述和相关理论称为Automata Theory。在Automata Theroy中定义的经典状态机具有状态集合、事件集合、状态迁移函数、初始状态和最终状态。这样定义的状态机缺乏层级结构和组合的定义，一般称之为Flat State Machine。在描述简单系统，例如一个计时器任务时，状态较少，它可以给出容易理解的定义，且易于实现与之对应的代码（或逻辑电路）。

但是当系统的复杂度增加时，这种Flat State Machine的状态空间会几何级数的增加。状态机的完整定义会成为繁杂和不可能完成的任务。其可理解程度也大大降低，其可维护性也基本完全丧失。

Harel在文章中给出了今天熟悉State Diagram的开发者熟知的一些内容：

1. 可以抽象state之间的共性，把Flat State Machine变换成Hierarchical State Machine；
2. 具有共性的state成为super-state，具有特性的state成为sub-state；
3. sub-state之间可以是互斥的（exclusive OR）；
4. sub-state之间也可以是与（AND）关系，称为正交的；在介绍State Diagram的文章中一般称之为平行的或者并发的。

本质上，Harel的做法并非是在回答State Machine的组合问题，相反，它是在一个庞大复杂的Flat State Machine上进行抽象（Abstraction），其结果是得到一个具有更少的图示元素的图示，更少的State，更少的状态迁移路线。从这个意义上说，我们不把Harel的Hierarchical State Machine看作是一种有效的组合状态机的方式。

在State Diagram中，Harel把具有正交关系的状态称为并发。这个定义和我们一般说的并发过程是有区别的。这个定义本身是可以成立的，比如在一个电子表的显示界面上，既有当前时间的显示，也有一个倒计时时钟的显示，两者在系统中是由不同的逻辑单元维护的，当前时间的状态和倒计时时钟的状态是无关的（正交）。但是Harel的这个例子，它设计的是一个静态（static）的系统，在后面会有更为详细的讨论。用静态系统模型可以描述较小的软件系统，这种并发定义也是可以成立的，但是它无法适应规模庞大的动态软件系统。

在这篇文章里Harel把它的并发定义和Process模型做了对比，他在Hierarchical State Diagram中构建了一种广播机制完成子状态机之间的通讯，例如系统启动时每个模块都收到了Power On事件，开始自己的初始化过程。这种方式和软件系统中常用的Pub/Sub是一回事，它一般被用于一个局部模块的内部，或者整个系统的粗粒度组件上。Pub/Sub机制没有办法Scale是一种共识，我们也不认为它具有良好的组合能力。

这里的结论是Hierarchical State Machine是一种不错的设计方式，它有助于减少状态空间爆炸；但是它仍然是针对单一逻辑单元或者简单逻辑单元的设计手段。它缺乏良好的组合能力，尤其是针对动态和庞大的系统而言。



## State Machine Composition

无论在理论上，还是实践上，并不缺乏对状态机组合的研究和应用。

实现状态机的组合的关键是为每一个状态机定义输入（input）和输出（output）。状态机在接受输入时，它可以把输入理解为事件，并据此定义其状态空间和状态迁移（行为）；但是状态机的输出，与这个状态机自己的状态**无关**。

例如在JavaScript中的一个Event Emitter对象，它可以在某一个事件发生时emit一个事件，但是它是否emit该事件，与它自身的状态无关，它需要emit事件是因为外部可能存在其他组件对这个事件感兴趣。从这个意义上说，我们把一个状态机emit事件理解为它和其他组件**通讯**的一种方式，一个对象emit的事件类型和数据对象，构成了它和外界的**通讯协议**。

所以在状态机的组合方式中，增加的输入输出的概念，被理解为状态机之间的通讯，所以在这里我们看到了第一种状态方法和过程方法的相似性，在实现组合中我们都为之创建了通讯的概念。

使用状态机通讯的方式构建的状态机组合，是设计和实现硬件电路的最重要方式。这和我们用组合函数或者组合对象构建软件系统，在哲学意义上，是一样的方法论。

在这里我们不详细讨论在硬件电路领域实现状态机组合的方式，软件开发者可以简单的看一下以下内容，了解一些基础逻辑，但是它们对软件开发的帮助并不大。

---

这里列举两个推荐阅读的资源：
1. MIT的一个开放课程（OpenCours Ware）[Introduction to Electrical Engineering and Computer Science I](https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-01sc-introduction-to-electrical-engineering-and-computer-science-i-spring-2011/)中，Unit 1中State Machines的讲义：[download](https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-01sc-introduction-to-electrical-engineering-and-computer-science-i-spring-2011/unit-1-software-engineering/state-machines/MIT6_01SCS11_chap04.pdf)，在这份讲义中教师以精炼的语言讲解了状态机和状态机组合，并使用Python给出了代码示例。这个课程难得的地方是它用代码向软件工程师展示了硬件工程师设计电子系统的状态机逻辑，其描述是简练和准确的（不像很多工程课程里讲解数学时引入不必要的类比和无意义的扩大概念范围）。
2. UC Berkeley的教授Edward A. Lee写的[Introduction to Embedded Systems](http://leeseshia.org/)一书的第5章关于State Machine Composition有很详细的介绍，这本书是免费图书，第5章谈到了状态机的并发组合。


在E.A. Lee的书里给出了状态机并发组合的定义，也讨论了同步和异步的组合方式，确定性和不确定性的定义。这本书里对很多概念做了详细的梳理，尤其是在不同的上下文下科学家或者开发者赋予同步概念的不同含义，完整呈现了领域内的相关知识，所以对于想对状态机组合有全面了解的开发者来说，书里的第5章和第6章需要仔细阅读。

> 异步一词在不同的上下文下有不同的定义，但是不管哪一种上下文都是先定义同步的概念。

如果只想了解本文后面述及的在Node.js里的并发编程，不需要理解书中的介绍的全部概念； 但应该知道，在本系列文章中讨论的Node.js的并发编程的基础方法，是源于一批法国的计算科学家提出的针对Reactive System的同步方法（Synchronous Approach），在Lee的书里第6章第2节，有详细的说明。

---



## 同步

Synchronous Execution or Synchronous Communication



## 动态

我们来看一下

我们陈述一下静态和动态编程的概念。

静态编程指的是系统内的组件或者对象，他们的生命周期和系统是一致的；如果一个系统中绝大部分组件或者对象都是静态的，在内存容量允许的情况下，我们就不需要使用`malloc`或者`new`语法创建这些组件或对象；它们可以用全局变量或者全局内存来表示。

在应用编程领域这样的例子不常见，但是在嵌入式系统、单片机中很常见，在硬件逻辑电路设计上很常见。在这些情况下，系统呈现的功能和资源，其数量都是固定的，一个电子表只有一个时钟组件，一个芯片上只有一个加法计算单元，或者芯片只有一个物理的串行通讯单元。在这种情况下，动态编程或者动态内存管理没有显著收益，反而造成了系统牺牲**实时性**（分配和销毁内存造成的），同时有潜在的内存分配或内存管理失败造成系统失败的风险。

> 事实上在最小的嵌入式领域编程中，如果对通讯能力要求不高的话，动态内存管理至今仍然是严格禁止的；但是随着物联网的发展，越来越多的设备需要接入网络，使用动态内存管理实现网络协议栈的系统越来越多，但是即使如此，很多网络协议栈的实现仍然在坚持静态内存的方式，代价是牺牲了网络通讯的质量。

在应用软件领域，完全的静态内存管理显然是一件不可能的事情。

指出这个区别对于理解为什么基于状态机和状态机组合设计硬件或嵌入式系统是广为流行的方式。







但State Machine本身首先是一种模型技术，它使用Event和State来抽象对象的行为（Behavior），它具有完备性。在数学上，自动机（Automata）理论是我们已知且成熟的对机器行为的设计和描述方式。Automata应该被看作State Machine的形式化（Formal）定义，它更为抽象一些，定义了更多的语法规则，而开发者使用的State Machine一般会定义较多的语义，它便于理解和在代码中使用。

State Machine方法在需要高可靠性的系统中，尤其是在嵌入式系统或操作系统内核中，有着极为广泛的应用。对于应用开发者而言，可能大部分开发者的对它的认知是：它通常被用于局部解决很棘手的对象行为设计，直接的收益是能够完备定义状态空间和对象行为，同时在代码层面得到充分分离逻辑的代码块，每个函数各司其职，在文档或者状态机图示的帮助下，这些代码是容易维护、修改、调试、和测试的。



这个做法是相当成功的，它提出的super-state，sub-state，sub-state之间的互斥（exclusive or）和正交（and）关系，可以大大减少Diagram（在Graph意义上的）状态节点（Vertices）和状态迁移（Edges）的数量，使之容易理解和成为代码实现。这也是它为什么能成为UML标准图示的原因之一。

但是Harel在论文里给出的对并发的定义是值得商榷的，他把具有正交关系（AND）的子状态，，定义为并发，这种定义是比较静态（static）的。同时Harel声称Statecharts模型里，子状态机之间是广播方式通讯，例如系统启动时，每一个模块都接收到了Power On事件，开始自己的初始化过程，整个系统的构建类似一个Pub/Sub机制。

我们需要在这里解释一个很细微但是很重要的设计上的概念区别，用消息（message passing）一词来统一抽象表述。

消息可以是单向的，A组件向B组件发送了一个消息，在这里组件可以是一个细粒度的代码单元，也可以是最粗粒度的系统模块，还可以是程序和外部世界的边界。单向通讯的发送者对接受者没有期望，接收者可以是一个、多个、或者没有。这种情况下的消息，我们一般称之为事件（Event）。Event用于描述发送者自己的行为，尤其是状态迁移。

在另外一些情况下我们使用双向的消息来实现功能。A组件向B组件发送一个请求，B组件应答。这个请求具有明显的协议特征，B组件是否会因此改变状态，在一些情况下是的，在另外一些情况下不必如此，但这不是A组件所关心的，A组件关心的是结果。换句话说，这是对意图和结果（而非状态）建模，我们一般用i/o来描述它。至于B组件是一个无态系统，仅仅通过纯计算来完成任务（这种情况下我们称B组件是transformational的），还是B组件是一个有态系统，返回的结果与其状态有关（这种情况下我们称B组件是reactive的），A并不关心。

Event/State和I/O不是完全对立的关系。通常它们是逻辑上的上下层。

例如Harel的Paper里以西铁城的电子表为例子，构建了Event/State系统，但是我们细想一下，它的Event来源有两种，一种是系统内部时钟的tick，另一种是用户操作；这两种情况都可以看作是系统的Input，这时候系统的Output是什么呢？是它向外部世界发送的结果，包括手表可能震动，包括液晶平面上显示的变化，在这种情况下系统并没有通过接口传输的数据作为Output，但是它使用的人机界面实现Output，而这种人机界面的显示状态，恰恰就是系统的State。换句话说，Harel通过Event/State模型来描述、设计、实现的西铁城电子表逻辑，它在更上层看构成了I/O关系。

士兵的例子。

TCP/IP的例子。

自发特性。

静态和动态

Harel的模型很适合静态系统编程。

在这里静态的含义是

但。。。

状态机通讯

基于状态机的组合和通讯能否构建复杂系统呢？答案是可能的。这是在硬件设计领域广泛应用的成熟技术。

Process has its states

State Machine可以组合，使用通讯







过程

这里说的过程（Process）是抽象的概念，实际的实现可能是进程（另一种定义的Process），线程（thread），协程（coroutine/fiber），甚至只是一个函数（function）。



过程（Process）是一种解决问题的方法，或者说是解法域模型：把一个复杂问题拆解成多个简单问题，每个问题用一个过程解决，通过创建和销毁过程，以及过程通讯，来完成整个计算任务。



计算科学家们（尤其在谈并发编程的时候）喜欢举下面的例子来说明这种思维方式：

---

问题是求解整数n之内的所有质数（n > 2）。

解法的关键点是为每一个已知的质数建立一个Process，算法如下：

1. 第一个质数是2，我们建立一个Process，标记为P2；
2. 我们开始把从2到n的数依次发送给P2；
3. P2的逻辑是：
   1. 如果这个数可以被2整除，扔掉它；
   2. 遇到第一个不能被2整除的数，实际上是3，它创建下一个Process，记为P3；
   3. 之后P2把所有不能被2整除的数都扔给P3；
4. P3的逻辑和P2一样，只要把2换成3；

这个过程继续下去，会创建P5, P7, P11...等等。当所有的数都被处理完之后，这些Process本身就是问题的答案。

---

这个例子展示了用Process建立模型解决问题的基本要素：

1. 每个Process是一个计算任务；
2. Process可以创建，称为fork；例子中没有，但当然它也可以销毁，称为join；
3. Process之间可以通讯；



这个例子是不是很优雅不在我们的讨论之列，即使它很优雅，大部分实际的编程问题没有这种优雅特性。



在这里我们只强调一点：用Process建模是用divide and conquer的办法解决问题的一种方式，同样的方式也存在于Component-based, Object-Oriented等多种编程模型技术中。



串行组合过程

我们先看最传统的基于blocking i/o编程的程序，如果把每个function都理解为一个process，这个程序的运行过程如何理解。

在所有的命令式语言中，函数都具有良好的可组合性（Composibility），即可以通过函数调用实现函数的组合；一个函数在使用者看来，不知道它是一个简单的函数还是复杂的函数组合，即函数和函数组合具有自相似性，或者说在语法上具有递归特性。所以我们只需回答针对一次函数组合，如何从Process的角度理解即可。

在一个父函数中调用子函数，父函数本身可以看作一个Process，P1，在它调用子函数时，可以理解为创建（fork）了一个新的Process，P2，然后P1被阻塞（blocked），一直等到P2完成并通过通讯把执行结果返还给P1（join），然后P1继续执行。



如果这样去理解我们可以看到，传统的基于blocking i/o的编程：

1. 程序运行时可以由一组Process的组合来描述；
2. 在任何时刻，只有一个Process在运行，Process之间的组合在运行时只有串行，没有并发；



异步

在上述传统的blocking io编程模式下，整个程序可能被一个i/o访问block住，这不是一个充分利用CPU计算资源的方式，尤其对于网络编程来说，它几乎是不可接受的。



所以操作系统本身提供了基于socket的回调函数机制，或者文件i/o的non-blocking访问，让应用程序可以充分利用处理器资源，减少执行等待时间；代价是开发者需要书写并发过程。



对于Node.js而言，在代码层面上，创建一个过程，是大家熟知的形式，例如：

    fs.readdir('directory path', (err, files) => {

    })

这里的fs.readdir是一个Node API里的文件i/o操作，它也可以是开发者自己封装的过程。



这个函数调用后立即同步返回，在返回时，创建了一个过程，这个过程的结果没有同步获得，从这个意义上说，我们称之为异步函数。



如果连续调用两次这样的函数，在应用内就创建了两个过程，我们称之为并发。



对事件模型的Node.js而言，这样来实现非阻塞和并发编程有两个显而易见的优势：

1. 它不需要额外的语法原语去实现fork，一个函数即可创建一个process，包括构造函数；
2. Node.js是单线程执行的，所以process之间的通讯是同步的；



同步的意思需要这样理解：假如用线程或者进程来实现process，process A只能通过异步通讯通知process B自己发生了某种状态迁移，因为这个通讯是异步的，所以process B不能相信收到的消息是它收到消息那个时刻的process A的真实状态（但是可以相信它是一个历史状态），双方的逻辑也必须健壮到对对方的状态假设是历史状态甚至错误状态之上，就像tcp协议那样。



在Node.js里没有这个烦恼，因为这些process的代码都在同一个线程内运行，无论是process A还是process B遇到事件：

1. 都可以同步读取的对方的真是状态
2. 都可以通过方法调用或者消息机制让自己和另一方实现一次同步和同时的状态迁移
   - 同步的意思是通过同步函数完成（在一个Node event loop的tick内）

- 同时的意思是process A s1 -> s2和process B t1 -> t2同时迁移

同时的特性被称为（shared transition），可以看作两个LTS（Labelled Transition System）的交互（interaction），也可以用Petri Net描述。它可以让过程组合容易具有完备的状态和状态迁移定义。



从上述这个意义上说，在并发模型上，Node领先Go或者任何其他异步过程通讯的语言一大截；但是反过来说，Node的单线程执行模型对于计算而言，没能利用到更多的处理器，是它的显著缺点。但是对于io，Node则完全不输用底层语言编程的多线程程序。



Callback Hell与CPS

这是被谈论最多的话题。



如何用Node.js书写异步与并发，和开发者面对的问题有关。如果在写一个微服务程序，fork过程主要发生在http server对象的事件里，或者express的router里；同时fork出来的每一个过程，我们用串行组合过程的方式完成它，即如果使用callback形式的异步函数嵌套下去，最终会得到一个可以利用到Node的异步并发特性，但是形式上非常难读的代码，这被称为Callback Hell。



在ES7中出现的async/await语法一推出就大受欢迎：

1. 它比较好的解决了Callback Hell的问题，
2. 书写条件流程在形式上也回归了传统的代码形式，
3. 能catch所有的错误
4. 通过Promise.all提供最简单的并发（fork/join）支持



在串行组合过程时，开发者最关心的问题是如何让过程连续下去，所以从这个意义上说callback函数，或者对应的promise，async/await，也被一些开发者称为Continuation Passing Style（CPS）。



这样说在概念上和实践上都没有问题。但是这件事情在整个Node并发编程上是非常微不足道的。因为这样的模型没有考虑到一个重要的问题：过程可以是有态的。



过程的状态

当我们在传统的blocking i/o模式下编程，书写一个表示过程的函数，或者在Node.js里用callback形式或async语法的函数，书写一个表示过程的函数，其状态可以这样表述：

    P: s -> 0



它只有两个状态，在运行（s），或者结束了（0）。结束的意思是这个过程不会对程序逻辑和系统状态产生任何未来影响。



我们优先关心结束，而不是关心它的成功、失败、返回值，因为前者是对任何过程都普适的状态描述，可以理解为语法，后者是针对每个具体过程都不同的语义。



当然不是所有的过程都会自发结束，比如用setInterval创建的周期性fire的时钟，调用了listen方法的http server，或者打开了文件句柄的fs.WriteStream，如果他们没有遇到严重错误导致自发结束，他们需要使用者的进一步触发（trigger）才能结束。



对于setInterval而言这个触发是clearInterval，对于http server而言这个触发是close，对于fs.WriteStream而言，这个触发是end，无论哪种情况，开发者应该从抽象的角度去理解这个问题：

1. 如果从Process模型去理解，它们需要使用者发送message
2. 如果从状态角度去理解，它们需要使用者触发event（形式上可以是method call）



我们举两个例子来说明这种状态更为丰富的过程，即无法用P过程表示的有态过程。



Q过程

第一个例子，我们考虑一个有try/catch逻辑的过程，如果async/await函数形式来写它大体是这样的代码：

    async function MyProcess () {
        try {
            // 1 do some await
        } catch (e) {
            // 2 
          	// 3 do some await
          	throw e
        } 
    }



这个Process开始执行后就进入了s状态，它有可能在try block成功完成任务，即s -> 0。它也可能遇到错误走向catch block，但是错误并不是从2的位置抛出，让使用者可以立刻获知和采取行动，它被推迟到3结束。



这样做的好处是这个过程仍然可以用s -> 0来描述，但是，从并发编程的角度说，使用者的error handler逻辑的执行时间被推迟了。可能很多情况下3的逻辑的时间足够短，并不需要去计较，从practical的角度说，我也会经常这样写代码，因为它容易。



但是从Process状态定义的完备角度说，这是个设计缺陷。



同样的过程我们可以换成Event Emitter的形式来实现，Emitter的实现不强制同时抛出error（或data）和抛出finish这两个事件，这对于callback形式或者async函数是强制的：返回即停止。

这样就给使用者提供了选择：

1. 如果它选择在error/data之后立刻开始后续逻辑，这种情况下我们称之为race。
2. 如果它选择必须在finish之后才开始后续逻辑，这种情况我们称之为settle。

race和settle都是join逻辑。但他们不必是互斥的（exclusive or），使用者也完全可以在error（或data）的时候触发一个后续动作，在settle的时候触发另一个动作。这样的模型才是普适的和充分并发的。



更为重要的，无论你采用什么样的模型去封装一个单元，一个重要的设计原则是，这个封装应该提供机制（Mechanism），而不是策略（Policy），选择策略是使用者的自由，不是实现者的决策。

如果分开两次抛出error和finish，使用者有自由选择race，或者settle，甚至both，这是使用者的Policy。

在这种情况下，我们可以用下述状态描述来表示这个过程：

    Q: s -> 0 | s -> e -> 0



约定：我们用s，或者s1, s2, ...表示可能成功的状态，或者说（意图上）走在走向成功的路上，用e，或者e1, e2, e3...表示明确放弃成功尝试，走在尽可能快结束的路上的状态。0只代表结束，对成功失败未置可否。



在这个Q过程定义中，所有的->状态迁移都是过程的自发迁移，不包含使用者触发的强制迁移。在后面的例子中我们会加入强制迁移逻辑。



在这里我们先列出一个重要观点（没有试图去prove它，所以目前不称之为定理或者结论）：完整的Q过程是无法简化成s -> 0的P过程的，所以它也无法应用到串行组合P过程的编程模式中。



在开发早期对Q过程有充分认知是非常必要的，因为开发者可能从很小的逻辑开始写代码，把他们写成P过程，然后层层封装P过程，等到他们发现某个逻辑需要用Q过程来描述时，整个代码结构可能都坍塌了，需要推倒重来。



这是我为什么说async/await的CPS实现不那么重要的原因。在Node中基于P过程构建整个程序是很简单的，如果设计允许这样做，那么恭喜你。如果设计上不允许这样做，你需要仔细理解这篇文章说的Q过程，和对应的代码实现里需要遵循的设计原则。



Q过程的问题本身不限于Node编程，用fiber，coroutine，thread或者process实现并发一样会遇到这个问题。要实现完整的Q过程逻辑必须用特殊的语法创建过程，例如Go语言的Goroutine，或者fibjs里的Coroutine；使用者和实现者之间需要通过Channel或对等的方式实现通讯。实现者的s/e/0等状态对使用者来说是显式的。



在Node里做这件事情事实上是比其他方式简单的，因为前面说的，inter-process communication是同步的，它比异步通讯要简化得多。



销毁过程

Node/JavaScript社区可以说很不重视从语法角度做抽象设计了。



http.ClientRequest里的取消被称为abort，Node 8.x之后stream对象可以被使用者直接销毁，方法名是destroy，但是 Node 8很新，你不能指望大量的已有代码和第三方库在可见的未来都能遵循一致的设计规则。在Promise的提案里，开发者选择了cancel来表示这个操作。



我在这篇文档里选择了destroy，用这个方法名来表示使用者主动和显式销毁（放弃）一个过程。新设计的引入需要小小的修改一下Q过程的定义：

    Q: s -> 0 | s => e -> 0



=>在这里用于表示它可以是一个自发迁移，也可以是一个被使用者强制的迁移；如果是后者，它必须是一个同步迁移，这在实现上没有困难，能够使用同步的时候我们没必要去使用异步，徒然增加状态数量却没有收益。



在Node生态圈里有不少可以取消的过程对象，例如http.ClientRequest具有abort方法，因此相应的request和superagent等第三方加强版的封装，也都支持了abort方法。



但是绝大多数情况下它们都不符合上述Q过程定义，而是把他们定义成了：

    P: s => 0



即状态s到0的迁移，可能是自发的，可能是被强制的（同步的）。这些库能这么做是因为：它们处理的是网络通讯，Node在底层提供了abort去清理一个socket connection，除此之外没有其他负担，所以在abort函数调用后，即使后续还有一些操作在库里完成，你仍然可以virtually的当作他们都结束了，因为它符合我们前面对结束的定义“不会对程序逻辑和系统状态产生任何未来影响”。



很多库都选择了这样的简化设计，这个设计在使用者比较小心的前提下也能纳入到“用串行Process”来构造逻辑的框架里，因为大部分库都采用了一个callback形式的异步函数同步返回句柄的方式。

    let r = request('something', (err, res) => {

    })

    // elsewhere
    r.abort()



这个写法不是解决destroy问题的银子弹，因为它没办法同时提供race和settle逻辑。而且在r.abort()之后，callback函数是否该被调用，也是一个争议话题，不同的库或者代码可能处理方式不一致，是一个需要注意的坑。



还有很多更为复杂的场景的例子。不一一列举了。我们倒回来回顾一下Node API本身，相当多的重要组件都是采用Event Emitter封装的，包括child，stream，fs.stream等等。他们基本上都可以用Q过程描述，但很难纳入到P过程和P过程串行组合中去。



小结

在这一部分内容里我们提出了用Process Model来分析问题的方法，它是一个概念模型，不仅限于分析Event Model的Node/JavaScript，同样可以用于多进程，多线程，或者协程编程的场景。



基于过程的分析我们看出Node的特点在于可以用函数这样简单的语法形式创建并发过程，也指出了Node的一大优势是过程通讯是保证同步的。



最后我们提出了过程可以是有态的，一个相对通用的Q过程状态定义。这是一个难点，但并不是Node的特色，任何双方有态通讯的编程都困难，例如实现一个tcp协议，但是在并发编程上我们回避不了这个问题。



Node的异步和并发编程可以简单的分为两部分：

1. 如何封装一个有态异步过程（Q过程）
2. 如何写出健壮的过程组合

前者是Node独有的特色，它不难，但是要有一套规则；后者不是Node独有的，而且Node写起来只会比多线程编程更容易，如果在Node里写不好，在其他语言里也一样。



通常的情况是开发者在设计上做一点妥协，牺牲一点并发特性，串行一些逻辑，在可接受的情况下这是Practical的，因为它会大大降低错误处理的难度，但遇到设计需求上不能妥协的场景，开发者可能在错误处理上永远也写不对。



这篇文章的后面部分会讲解第一部分，如何封装异步过程。如何组合这些异步过程，包括等待、排队、调度、fork/join、和如何写出极致的并发和健壮的错误处理，是另外一篇文章的内容。



异步过程

异步过程这个名字不是很恰当，过程本身就是过程，没什么同步异步的，同步或者异步指的是使用者的使用方式。但在没有歧义的情况下我们先用这个称呼。



在Node里一般有两种方式书写一个异步过程：callback形式的异步函数，和Event Emitter。

还有一种不一般的形式是把一个Event Emitter劈成多个callback形式的异步函数，这个做法的一个收益是，和OO里的State Pattern一样，把状态空间劈开到多个代码block之内；很显然它不适合写复杂的状态迁移，会给使用者带来负担，但在只有两三个串行状态的情况下，它可以使用，如果你偏爱function形式的语法，唾弃Class的话。



异步函数

异步函数的状态定义是：

    P: s -> 0



它没有返回值（或句柄），它唯一的问题是要保证异步。初学者常犯的错误是写出这样的代码：

    const doSomething = (args, callback) => {
      if (!isValid(args)) {
        return callback(new Error('invalid args'))
      }
    }



这是个很严重的错误，因为doSomething并不是保证异步的，它是可能异步可能同步的，取决于args是否合法。对于使用者而言，如果象下面这样调用，doElse()和doAnotherThing()谁先执行就不知道了。这样的逻辑在Node里是严格禁止的。

    doSomething((err, data) => {
        doAnotherThing()
    })
    
    doElse()



正确的写法很简单，使用process.nextTick让callback调用发生在下一个tick里。

    const doSomething = (args, callback) => {
      if (!isValid(args)) {
        process.nextTick(() => callback(new Error('invalid args')))
        return
      }
    }



异步函数可以同步返回一个对象句柄或者函数（就像前面说的http.ClientRequest），如果这样做它是一个Event Emitter形式的等价版本。我们先讨论完Emitter形式的异步过程，再回头看这个形式，它事实上是一个退化形式。



Event Emitter

我们直接考虑一个Q过程：

    P: s -> 0 | s => e -> 0



代码大体上是这样一个样子，：

    class QProcess extends EventEmitter {
      constructor() {
        super()
      }
    
      doSomething () {
    
      }
    
      destroy() {
    
      }
      
      end() {
          
      }
    }



这个过程对象可以emit error, data, finish三个event。



用Process Model来理解它，这个class的方法，相当于使用者过程向实现者过程发送消息，而这个class对象emit的事件，相当于实现者向使用者发送消息。从Process模型思考有益于我们梳理使用者和实现者之间的各种约定。



使用者调用new QProcess时创建（fork）了一个新的Process。绝大多数Process不需要有额外的方法，因为大部分参数直接通过constructor提供了。



Builder Pattern在Node和Node的第三方库中很常用，例如superagent，它提供了很多的方法，这些方法都是在构造时使用的，他们都是同步方法，而且没有fork过程，直到最后使用者调用end(callback)之后才fork了过程，我们不仔细讨论这个形式，它很容易和这里的QProcess对应，唯一的区别是end的含义和这里定义的不同。



这里提供了两个特殊方法，destroy和end，前者是销毁过程使用的，它在错误处理时很常见。在串行组合过程的编程方式下，我们没有任何理由去destroy其中的一个正在执行的过程；但是在并发编程下，一个过程即使没有发生任何错误，也可能因为并发进行的另一个过程发生错误而被销毁，从这个角度说，Q过程都应该提供destroy方法。



end在这里是对应stream.Writable的end逻辑；一个写入流（我们同样把他当Process看），如果没有end它永远不会结束；这种过程对象和等价于for-loop的setInterval不同，在绝大多数情况下我们不会希望它是永远不结束的。



doSomething是一个通用的写法，如果QProcess封装的过程需要尽早开始工作，但是它也需要不断的接受数据，doSomething用于完成这个通讯，典型的例子是stream.Writable上的write方法。write方法不同于end方法的地方在于，write方法不会导致QProcess出现对使用者而言显式的状态迁移，但end方法是的。



error事件，当然可以argue说一个QProcess可以在抛出error后继续工作，但是我们不讨论这个场景；我们假定QProcess在抛出error后无法继续走在可能成功的s路线上；如果它可以继续，那么那个error请使用其他event name，当作一种data类型来处理。



Node的Emitter必须提供error handler，否则它会直接抛出错误导致整个应用退出，所以从这个意义上说，error是critical的。符合我们说的限制。



data事件，它类似write，它应该表示QProcess仍然走在可能成功的s路线上。



finish事件，它符合前面我们说过的过程结束的定义。



实现者承诺



1. 异步保证

在任何情况下，QProcess都不允许在使用者调用方法时，同步Emit事件。

在Event Emitter形式下的Q过程，仍然要遵循异步保证。它的出发点是一致的，如果没有这个保证，使用者无法知道它调用方法之后的代码，和event handler里的代码哪一个先执行。如果遇到同步的错误，QProcess仍然需要象异步函数一样，用process.nextTick()来处理。



2. emit时的状态完备性

如果QProcess需要emit事件，它必须保证自己处于一个对使用者而言，显式且完整的状态下。

QProcess内部的实现也会侦听其他过程的事件，这些事件的到来可能会导致QProcess执行一连串的动作。

例子：如果一个QProcess内部包含一个ChildProcess对象，在QProcess处于s状态时，它抛出了error，这时过程已经无法继续，QProcess执行一连串的动作向e状态迁移：

    transition: s -> action 1 -> action 2 -> action 3 -> e

emit error的时间点在进入e状态之后，emit error的含义是通知使用者发生了致命错误，而且QProcess已经迁移至e状态。

在action过程中随意的emit error是严格禁止的。

因为：使用者可能在error handler中调用QProcess的方法，无论是同步还是异步方法。如果严格要求QProcess在有效状态下emit，那么QProcess的实现承诺就是这些方法在有效状态下可用。如果写成在action 1之后也允许emit error，对QProcess的要求就提升到在任何transition action时都可用，这种是无厘头的挑战自我设计，毫无意义。而且即使你成功的挑战了自我，使用者也带来了额外的负担，它的handler里对QProcess的状态假设是什么？是状态迁移中？这明显不合理。



Q过程对使用者是显式有态的，是它的执行逻辑的依据，所以这里应该消除歧义，杜绝错误。这种错误是严重的协议错误。



3. 只emit一次，承诺状态

QProcess在内部的一次event handler中只允许emit一次，而且承诺状态：

1. 如果emit error，表示QProcess处于e状态
2. 如果emit data，表示QProcess处于s状态
3. 如果emit finish，表示QProcess处于0状态

有两种常见的错误情况。



第一个例子：假如QProcess的第一个操作，例如通过fs.createReadStream()创建一个文件输入流，因为文件不存在立刻死亡了。这时它有这样一些选择：

1. 先emit error，然后同步emit finish；
2. 先emit error，然后异步emit finish；
3. 只emit error，不再emit finish；
4. 只emit finish，不emit error；

正确的做法是4。因为QProcess已经结束，是显式状态0，emit finish通知使用者自己发生了状态迁移（s -> 0）是正确的做法。至于错误，推荐的做法是直接在finish里提供，在对象上创建标记让使用者去读取也是可以的。



在这样的设计下，emit error的语义被缩减的了，即如果QProcess emit error，说明它一定处于e状态，不是0状态，这有助于使用者使用（代码路径分开原则，后述）。



在这个例子下如果选择1呢？你怎么考虑如果在emit error之后：

1. 实现者对自己的状态承诺是什么？
2. 使用者如果在error handler里调用QProcess的同步方法，它会被强制状态迁移吗？如果迁移了，那么随后emit finish还能成立吗？
3. 使用者的error handler方法和finish handler方法是可能异步可能同步执行的，使用者要保证在这样的情况下也OK吗？

第二个例子：假如QProcess处于s状态，抛出了data事件，对QProcess而言它不知道这个data是否非法，但是使用者可能有额外的逻辑认定这个data是错误的，这个时候它调用了QProcess的destroy方法，这个方法要求QProcess的强制状态迁移s -> e。



如果遵循这一条设计要求，这种设计就是很安全的。否则连续的emit的第二次的emit对状态的假设就没法确认了，难道使用者还需要在第二次emit之前去检查一下自己的状态吗？



error的处理有另外一种设计路径。在s状态下emit error，然后使用者调用destroy方法强制其进入e状态；逻辑上是对的，也具有数学美感，因为它没区分s和e的处理方式；但我倾向于不要给使用者制造负担，实现者的代码写一次，使用者的代码写很多次，这样设计需要使用者在每一次都要去调用destroy，相对麻烦。当然如果你有足够的理由局部去做这样的设计，可以的。



4. Destroy是同步方法

回到Process模型上来。



如果一个Process的实现是用操作系统进程、线程来实现，同步destroy的可能性是没有的，只能发送一个message或signal，对应的进程或者线程在未来处理这个消息，对于使用者而言它仍然可能在destroy之后获得data之类的message，当然这也不是很麻烦，使用者只要建立一个状态作为guard，表示Process已经被destroy了，忽略除了exit之外的消息即可。



在Node里面，逻辑上是一样的，但是实现者的destroy的代码可以同步执行，它也是同步迁移到e状态的，使用者不需要建立guard变量来记录实现者的状态；按照Node的stream习惯，实现者应该有一个成员变量，destroyed，设置为bool类型，供使用者检查实现者状态。



5. End是同步方法

逻辑上是和Destroy一样的，不同之处在于实现者都处于s状态。



在一些错误处理情况下，使用者可能会根据一个过程对象是否end采取不同的错误处理策略。尚未end的过程一般被抛弃了，通常也无法继续进行；但是已经end的过程可能会等待它完成。即对一组并发的过程可能采用不同的策略。



但是在设计角度说，还是前面那句话，要坚持mechanism和policy分离的原则。实现者应该提供这个机制而不是强制使用者必须用某种策略，策略是使用者的逻辑。它可以全部抛弃尚未完成的Process，也可以只抛弃尚未end的，对于大文件传输来说这可以避免不必要的再次传输，毕竟网络传输已经完成，只是文件尚未完全写入文件系统。



使用者策略

下面来看使用者策略和需要注意的问题，其中一些问题的讨论会给使用者和实现者都增加新规则。

Mutex

Event Handler是一种状态资源。

假如我们在用状态机方法写一个对象（不需要是上述的过程），这个对象的某个状态有一个时钟，它在进入这个状态时需要建立这个时钟，在超时时向另一个状态迁移，但是它也可以在计时状态下收到其他事件从而迁出这个状态，这是它必须清除这个时钟，这是状态机编程下的资源处理原则：在enter时创建，在exit时清理。为什么要清理呢？即使不考虑浪费了一个系统时钟资源，这个时钟挂上了一个callback，我们必须阻止它的执行，否则系统逻辑就错了。



所以从这个意义上说，Event Emitter上的所有Listener，也要被看作是资源。需要做相应的清理，否则可能会导致错误。



例子，启动一个子进程，等待它返回一个消息，作为过程需要的结果。

    let c = child.spawn('some child process')
    c.on('error', err => {})
    c.on('message', message => {})
    c.on('exit', (code, signal) => {
        
    })



这段常见的代码形式最终如何封装取决于我们的设计要求，如果允许省事我们可以先把它封装成s -> 0，即使用最简单的回调函数形式。



我们看一下这三个handler，他们都是我们需要在开始的时候就挂载上去的，包括exit（finish），因为子进程可能没有完成工作就意外死亡了。但是exit有点特殊，如果error先发生了，我们就不关心message和exit了，如果message先发生了，我们也不再关心error和exit了，这意味着我们已经拿到了正确的结果，即使在这个结果之后发生了子程序的意外退出，无所谓了，而如果exit先发生了，那么我们也不用关心error和message了，因为ChildProcess应该承诺这是finish。



所以看起来非常简单的代码，仔细分析一下才会发现三者是互斥的。不管结果如何，first win。所以代码写成这个样子了：

    function doSomething(args, callback) {
      let c = child.spawn('some child process, with args')
      c.on('error', err => {
    	c.removeAllListeners()
        c.on('error', () => {})
        callback(err)
      })
      
      c.on('message', message => {
        c.removeAllListeners()
        c.on('error', () => {})
        callback(null, message)
      })
      
      c.on('exit', (code, signal) => {
      	// if ChildProcess is trusted, no need to remove any listeners
        callback(new Error(`unexpected exit with code ${code} and signal ${signal}`))
      })
    }



在error和message handler里都清除了与之互斥的代码路径。有一个小小的坑是Emitter的error handler必须提供，否则会造成全局错误，这是Node的设计，所以我们塞上一个function mute它。另外这里的child对象是我们自己创建的，这样粗暴的removeAllListeners就可以了。如果是外部传入的，不能这样暴力，而只能清除自己装上去的handler。



这段代码通常的写法是在function内放一个闭包变量，例如let finished = false，然后在所有的event handler里面做guard，如果只有一层逻辑，这样写是OK的，但是闭包变量做guard在逻辑多的时候，尤其是出现等待同步逻辑的时候，很无力。它没有清楚的看到所有的状态空间，容易导致错误。



习惯上我把这个设计原则称为mutex（mutual exclusive），当然有时候不一定是双方互斥，象上面的例子就是多方的。mutex当然在thread model下有其他的含义，但是在Node.js的上下文下没有那个含义的mutex，我们姑且用这个词。



这里顺便给一个拆除pipe的例子，因为这种情况太场景了，写不对的开发者不在少数。



假定在代码中有ws是write stream，rs是read stream，调用过rs.pipe(ws)

    rs.removeAllListeners()
    ws.removeAllListeners()
    rs.on('error', () => {})
    ws.on('error', () => {})
    rs.unpipe()
    rs.destroy()  // node 8.x specific
    ws.destroy()  // node 8.x specific



1. 这里removeAllListeners()是简化的写法，如果stream不是自己创建的，需指定listener。
2. 拆除和重装listeners应该在调用方法之前，因为我们不确定Node stream遵循了写在前面的设计原则，即它会不会同步emit error或其他事件。
3. destroy是node 8.x才有的方法，如果是早期的版本没有这个方法，会报错。



代码分开原则

这个原则表述起来很容易。

1. 空间上分开，例如error是走向错误的代码（e-path），data是走向可能获得结果的代码（s-path）；这一点在Event Emitter的形式上（error和data handler）已经得到保证，在callback或者finish handler上需要自己检查，但不是很大的麻烦；
2. 时间上分开，这个要麻烦一点，它的意思是说最好在每个状态下只给所有的过程对象装上对这个状态向下一个状态迁移的handler代码，在真正迁移时，先移除所有旧的handler，然后装上新的handler。

比如把上面P过程实现的代码进化成过程，即考虑使用者对finish事件是有兴趣的，它可能需要采取settle逻辑，而不仅仅是race。



在这种情况下：

1. s -> 0一步完成且得到message的成功路径是没有的。成功的路径只能是s1 -> s2 -> 0，其中s1没有message，s2有。
2. s -> 0一步完成未获得message的路径是有的。
3. s -> e -> 0的路径是有的，先遇到错误，迁移到e状态，然后迁移到完成。
4. 实际上还可能遇到s1 -> s2 -> e -> 0的情况，即错误发生在获取message之后；与之相反的，也可能遇到先得到error然后得到message的可能，那么这两种情况我们都抛弃了，我们的设计目的是使用者对finish有兴趣，不是为了穷举状态空间。

    class DoSomething extends EventEmitter {

      constructor(args) {
        super()
        this.destroyed = false
        this.finished = false
        
        this.c = child.spawn('some child process, with args')
        c.on('error', err => {
          // enter e state
          this.destroyed = true
          c.removeAllListeners()
          c.on('error', () => {})
          c.on('exit', (code, signal) => {
            this.finished = true
            this.emit('finish')
          })
          this.emit('error', err)
        })
        
        c.on('message', message => {
          // s1 -> s2
          this.message = message
          c.removeAllListeners()
          c.on('error', () => {})
          c.on('exit', (code, signal) => {
            this.finished = true
            this.emit('finish')
          })
          this.emit('data', this.message)
        })
        
        c.on('exit', (code, signal) => {
          // -> 0
          this.finished = true
          this.emit('finish', new Error('unexpected exit'))
        })
      }

      destroy () {
        if (this.finished || this.destroyed) return
        this.destroyed = true
        this.c.removeAllListeners()
        this.c.on('error', () => {})
        this.c.on('exit', () => {
          this.finished = true
          this.emit('finished')
        })
        this.c.kill()
      }
    }



这个例子不算特别的有代表性，但是它展示了和OO编程模式下的State Pattern一样的代码路径分开原则，收益也是一样的。



destroy的实现也很容易。



代码中的这种实现方式大部分开发者都会同意：强制转换状态，且继续emit finish。但我们有另外的设计方式。



更好的Destroy设计

如果考虑error code path和success code path的分开，我推荐另一种设计方式：

destroy在调用后，过程对象不再emit finish事件；如果destroy提供了callback，在原有应该emit finish事件的地方，改为调用该callback；如果destroy没有提供callback，do nothing。换句话说，如果使用者提供了callback，它就选择了settle逻辑，如果它不提供callback，就是race了。



按照这样的设计方式，destroy的代码改为：

      destroy (callback) {
        if (this.finished || this.destroyed) return
        this.destroyed = true
        this.c.removeAllListeners()
        this.c.on('error', () => {})
        if (callback) {
          this.c.on('exit', () => {
            this.finished = true
            callback()
          })
        }
        this.c.kill()
      }



站在使用者角度讲，这样的实现更好。因为使用者可以分开让过程自发运行和强制销毁的finish handler代码。destroy操作如此特殊，它几乎只用于错误处理阶段，让使用者为此独立提供代码块是更方便的，这符合我们严格分开成功和错误处理的代码路径的原则。



退化的场景

下面来给前面用P过程状态定义实现的异步函数加装destroy方法，粗暴的方式是直接返回函数（而不是对象句柄）。

    function doSomething(args, callback) {
      let destroyed = false
      let finished = false
      
      let c = child.spawn('some child process, with args')
      c.on('error', err => {
    	c.removeAllListeners()
        c.on('error', () => {})
        finished = true
        callback(err)
      })
      
      c.on('message', message => {
        c.removeAllListeners()
        c.on('error', () => {})
        finished = true
        callback(null, message)
      })
      
      c.on('exit', (code, signal) => {
      	// if ChildProcess is trusted, no need to remove any listeners
        finished = true
        callback(new Error(`unexpected exit with code ${code} and signal ${signal}`))
      })
      
      return (callback) => {
        if (destroyed || finished) return
        c.removeAllListeners()
        c.on('error', () => {})
        if (callback) {
          c.on('exit', () => callback())
        }
        c.kill()
      }
    }



这个逻辑和之前是一样的。但是这个做法更容易出现设计形式上的争议：因为doSomething在调用时提供的callback没有保证返回。



我们说这个原则在doSomething不提供返回的时候肯定是需要保证的。但是如果象这样写，这个原则可以修改。



反对这样设计的理由是充分的，任何请求或者Listener在提供者销毁时应该得到通知，这是对的；比如在OO编程时我们经常会观察一些对象，或者向某个提供服务的组件请求服务，即使被观察对象或者服务组件销毁也应该提供返回或通知。



但是这里的设计原则和上述场景不一样。在这里我们强调Owner原则，使用者创建实现者是给自己使用的，自己是Owner，自己销毁这个实现者然后还要坚持从原有的callback或者finish handler得到一个特定的Error Code判断销毁原因，这没有必要。



至于上述的Observer Pattern的Listener，或者向Service组件的请求的实现，我们在下一篇会给出代码例子。



在这里我们强调的是，这里的callback或者finish handler，不是上述Observer/Service Request意义上的listener/callback，Observer和Service Requester并非被观察对象或者服务的Owner，他们当然无权修改对象或服务的特性，而且服务质量也必须被保证。



但是在这里，我们是在用callback或者emitter这种代码形式实现process composition所需的inter-process communication。我们有权为了使用便利而设计代码形式。



总结

为复杂Process建立显式状态，是解决问题的根本方法。如果设计要求就是具有这些状态的，你用什么方法写都一样，这不是Node特有的问题。



Node特有的问题在于它写异步有态过程需要遵循的设计原则，很多开发者不熟悉状态机方法，所以很难写的健壮。这个话题也可能有它比较独特的地方，因为Node是强制异步过程和事件模型的混合体，但是这种编程模式在嵌入式系统、内核、以及一些系统程序中，是非常常见的（大多数是C语言的，指针实现callback）。



这篇文章我会长期维护。如果需要更多的代码示例，或者有很具体的问题需要讨论，可以提出。



只要能把基础的异步过程写对，如何组合它们实现并发，并发控制，等待，同步（join），以及健壮的错误处理，是易如反掌的，那是我们下一篇的话题。