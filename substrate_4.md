# Substrate 入门 - 项目结构 -（四）

在开始讲解Substrate的内容之前，还需要补充一个**当前**Substrate的项目结构。之所以这里强调“当前”是因为Substrate从之前的结构到现在已经发生了巨大的变化，如果不重新再介绍以下，后续的文章介绍起来就会有一些麻烦。

笔者在一年前已经写过一个Substrate的项目结构[《Substrate 设计总览 （二）》](https://zhuanlan.zhihu.com/p/56414647)的介绍，但是截止目前项目的结构几乎已经全部变了。因此重新进行介绍。（在看本文前最好先了解了Substrate的设计总览：[《Substrate 设计总览》](https://zhuanlan.zhihu.com/p/56383616)）

本文所基于的Substrate提交为：`a98625501be68cc3084e666497c16b111741dded`，即2019年12月21日的提交。

## `core`->`primitives` + `client`

以前的`core`目录包含了所有的链的功能模块的部分，也就是所谓的Substrate的框架主体内容。在现在的版本中，拆成了`primitives` + `client`两个部分。

其中：

* `primitives`：元语，赋予了新的含义，用于**定义**一条链中很多**基础设置**的模块。实际上这个归类也是比较笼统的，但是主要还是用来定义用的，举例来说：
  * `primitives/core`中定义了密码学类型如`ed25519`，`sr25519`等，定了`hash`，定义了`H160`，`H256`，`H512`等长度类型，
  * `primitives/runtime`定义了“Runtime”中需要用到的基础类型，如区块头 `Header`，区块`Block`，交易体`extrinisic`，区块头附带的信息“digest”，“Runtime”编写过程中用到的一些基础工具，如错误处理`DispatchError`，兼容多种类型签名的`MultiSignature`，Runtime中的随机数，交易合法验证，offchain的一些类型定义等等
  * `primitives/consensus`Substrate提出的新共识的核心`aura`，"babe"，以及“pow”的一些类型定义，注意这里只是一些定义以及对于Runtime的api接口，实现在`client/consensus`对应的模块下。
  * `primitives/api`runtime的api的工具宏定义
  * `primitives/trie` 对 MPT 的包装，MPT见上文介绍，实现在`trie-db`这个库中
  * `primitives/std` 和 `primitives/io` 原来的`sr-std` 和`sr-io`，用于提供Runtime中的对于wasm编写的支持以及runtime访问trie的接口。
  * 等等内容

而对于另一个模块`client`而言：

* `client`是对于很多模块的**功能实现**以及集成功能模块组件
  * `client/api`就是对于runtime api 调用包装的实现
  * `client/consensus`是对共识模块的实现。
  * `client/network`是对网络p2p的实现，底层使用Libp2p
  * `client/state-db`是对每个块提交到MPT的过程的管理
  * `client/service`是对许多功能模块的集成，例如网络模块，交易池，rpc等等，service相当于启动了这些模块并持有这些模块的引用
  * `client/cli`是对命令行的解析并根据相应的参数配置service
  * 等等... 由于本系列主要是为了介绍使用substrate，而不是讲解substrate怎么实现，所有就不必说的很细致了。

所以在今后的文章中介绍`primitives`/`client` 即代表着这些是Substrate框架内的代码，若要修改，需要进行fork

ps:（这是不严谨的说法，实际上也可以引用`primitives`，自己实现`client`，只所以这么说是因为`client`可以看做`Substrate`定义的基础结构的一种实现，当然可以独立进行另一种client的实现。但由于当前的client里面是一种实现，因此里面的模块都是互相关联的，很难只修改其中某个组件，如果需要沿用Substrate的实现但是需要修改其中的某些实现，那么只能fork修改了。）

## `srml` -> `frame`

参见笔者之前的《Substrate 设计总览》与《Substrate 设计总览 （二）》，应该知道了Runtime的概念，与Substrate中实现Runtime的模块srml（Substrate Runtime Module Library）。在新版中，`srml`命名为了`frame`。

对于`frame`而言，模块之间没有很大的迁移，基本沿袭了原来的命名与结构，只不过新增了很多的runtime module 模块。

如理解了这两篇文章的内容，那么应该可以理解`frame`中除了“system”之外，都是可以由开发者自由实现的，实际上使用Substrate链编写的应用链的核心部分就是开发自己的Runtime Module。编写自己的Runtime Module不会写的时候，就需要去模仿`frame`模块中这些模块的编写技巧。其中提供了很多最佳实现这类的东西。

## `node` + `node-template` + `subkey` + 其他 -> `bin`

**对于上文的 `primitive`，`client`或者再加上`frame`中的`system`共同构成了Substrate的框架**，而对于新版本中的`bin`则是对Substrate框架的**使用案例**（一种实现），因此**`bin`中的内容就是Substrate的框架外**的部分。

在新版本中把原来的node，node-template等全部移动到了bin中，代表这些模块是可以进行“可执行文件”的入口。因此调试也好，研究数据流流转也好，都是从这个入口开始。

所以在今后的文章中，介绍`bin`中的代码，即代表着这是使用者可以自由进行定制的部分，可参考bin中的一些组件进行修改。而bin中的案例即是如何把runtime和service组合在一起的案例。

对于本系列而言，只用参考node的实现就够了，因为它的实现是最全面的。