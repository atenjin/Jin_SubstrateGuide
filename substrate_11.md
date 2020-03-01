# Substrate 入门 - 学习Runtime必备的技能 -（十一）

上一篇文章介绍了Runtime的构成方式。但是在介绍过程中我们可以看到，其比较核心的组件大多都是用rust的宏编写。熟悉编程语言的人应该知道，宏本质上是创建了一种DSL，使用者必须按创作者的方式来编写才可编译通过，因此宏更像是黑盒，在中间做了许多表面上看不到的事情。

Rust使用了卫生宏系统，在编译器可以安全的解决许多问题，而Substrate的开发者对于宏似乎有一些迷恋，在Runtime中诸多核心组件都采用了宏，并且通过宏自动化做了相当多的事情并生成了许多额外变量和类型。笔者个人觉得Substrate的框架在宏的使用上有一些滥用，其在一定程度上阻碍了使用者能够轻松理解Substrate的这套系统。但客观来说，Substrate Runtime中的宏做了许多重复性与自动化的工具，隐藏了许多细节不需要开发人员需要操心的细节，因此如果能**正确理解了创作者创建这个宏所表达的意图**，那么确实可以节省很多无用的工作。

所以关键问题在于如何理解这个宏背后所做的工作，因为只有正确理解了才能明白例如在上一篇文章中介绍的Module类型的生成等情况。

## 展开宏

要理解宏背后做的工作，最直接的方式当然就是看宏自身是怎么写的。但是平心而论，Substrate编写这块的作者虽然有一些滥用宏，但是他的技巧是十分高超，生成宏这部分的代码量都十分庞大。若不是对宏系统十分熟悉（因为如果只是写简单的宏理解起来不困难，但是若不常写，只是看宏的话那些`$`替换符会很别扭，思维也不容易把这些联系起来），那么硬生生去读宏的写法会相当困难。

因此若只是为了理解宏最后干了什么事情的话，使用宏展开比去理解宏的写法好得多。

因此本文介绍在Runtime中宏展开及一些相应技巧。

首先先要明确一个前提，由于之前的介绍，我们应该能理解对于Runtime而言，native和wasm应该在大部分情况下是同一份代码。因此我们展开宏一般情况下只针对native展开。很特殊及稀少的情况下才可能需要wasm展开。那么在展开wasm的时候请依据之前的文章添加上相应的feature开关。

展开宏笔者在这里介绍是对于`crate`维度，不针对`xxx.rs`维度。因为只编译`xxx.rs`难度很大，而且在很多情况下反而不太方便。

例如如果想要node项目中的runtime的宏，那么首先切换到相应的`crate`目录下：

```bash
cd bin/node/runtime/
```

然后使用`carge`的宏展开命令：

```bash
cargo rustc -- -Z unstable-options --pretty=expanded > runtime.rs
```

由于目前Substrate已经支持stable的rust了，所以这里展开没必要用nightly。如果需要nightly，那么在`cargo`后面加上`+nightly`

另一方面，由于当前runtime的特性，我们首先要看到在`bin/node/runtime/src/lib.rs:L74`行，有：

```rust
// Make the WASM binary available.
#[cfg(feature = "std")]
include!(concat!(env!("OUT_DIR"), "/wasm_binary.rs"));
```

因此这里需要明白在`bin/node/runtime/`展开宏的时候，事实上把编译好的wasm代码也包含了进来。而对于当前的substrate来说，wasm即使在release模式下也已经达到了1.7M（见文件`target/debug/wbuild/node-runtime/node_runtime.compact.wasm`），若wasm以debug编译有10几兆大小。因此在上面`cargo rustc`中将输出重定向到的`runtime.rs`一定大于这数。

```bash
$-> ll -h
-rw-r--r-- 1 name name  955 2月  26 11:08 build.rs
-rw-r--r-- 1 name name 7.7K 3月   1 10:14 Cargo.toml
-rw-r--r-- 1 name name  11M 3月   1 20:32 runtime.rs
drwxr-xr-x 2 name name 4.0K 1月  13 20:49 src
```

此时若使用ide的读者，不要急着直接点开这个文件，而且先经过以下操作再打开。

由于wasm被包含进入了`runtime.rs`，而实际上我们并不需要看懂编译出来的wasm的字节串，因此我们将其直接删除即可：

```bash
vim runtime.rs
```

打开后搜索`WASM_BINARY`，找到后删除这一行及下一行字节乱码串(就是编译的wasm)

再搜索`WASM_BINARY_BLOATY`，同样删除这一行及下一行

然后保存退出

```bash
$-> ll -h
-rw-r--r-- 1 name name  955 2月  26 11:08 build.rs
-rw-r--r-- 1 name name 7.7K 3月   1 10:14 Cargo.toml
-rw-r--r-- 1 name name  836k 3月   1 20:32 runtime.rs  # 请注意runtime.rs的体积已经缩小了很多
drwxr-xr-x 2 name name 4.0K 1月  13 20:49 src
```

此时再打开runtime.rs文件就不会受到wasm的干扰了，之后可以格式化一下，这样查看会好一些。

## 展开后的`runtime.rs`

我们通过以上方式可以得到这个展开的文件，那么我们可以查看一下上一章节提到的一些类型：

例如`AllModules`

```rust
type AllModules
=
((Vesting,
  (Recovery,
   (Society,
    (Identity,
     (RandomnessCollectiveFlip,
      (Offences,
       (AuthorityDiscovery,
        (ImOnline,
         (Sudo,
          (Contracts,
           (Treasury,
            (Grandpa,
             (FinalityTracker,
              (TechnicalMembership,
               (Elections,
                (TechnicalCommittee,
                 (Council,
                  (Democracy,
                   (Session,
                    (Staking,
                     (TransactionPayment,
                      (Balances,
                       (Indices,
                        (Authorship,
                         (Timestamp,
                          (Babe, (Utility, ))))))))))))))))))))))))))));
```

我们可以看到`AllModule`实际上是一个将所有模块集合在一起的嵌套元组，对应`OnInitialize`的定义`primitives/runtime/src/traits.rs:L343`

```rust
#[impl_for_tuples(30)]  // 注意这个impl_for_tuples
pub trait OnInitialize<BlockNumber> {
	/// The block is being initialized. Implement to have something happen.
	fn on_initialize(_n: BlockNumber) {}
}
```

再跟随一下执行器对于`on_initialize`的实现`frame/executive/src/lib.rs:L186`：

```rust
<AllModules as OnInitialize<System::BlockNumber>>::on_initialize(*block_number);
```

即可**理解`AllModules`为什么是使用嵌套元素的形式定义，而`on_initialize`的调用顺序即是`construct_runtime!`中模块定义的顺序**

例如`Runtime`

我们在原本的`bin/node/runtime/lib.rs`中可以看到，每个runtime module导出的trait都实现给了`Runtime`类型，但是我们却不知道`Runtime`在哪定义了。

那么在展开文件中，我们可以搜索到

```rust
pub struct Runtime;
```

因此，`Runtime`这个类型是由宏展开生成的，并且结合`lib.rs`，应该知道实际上所有的runtime module 中定义的trait都实现给了这个`Runtime`，因此这个`Runtime`是所有module的trait的实现体。而`Runtime`自身不持有任何成员，因此实际上持有Runtime的意义在于将所有module的trait中的关联属性集合到一个对象上。

例如`Balances`

可以看到`Balances`的定义为：

```rust
pub type Balances = pallet_balances::Module<Runtime>;
```

因此`Balances`结构体即是在`pallet_balances`这个crate下的Module，传入了Runtime的类型，而Runtime是所有trait的实现体。而我们在`frame/balances`这个crate下却不能发现`Module`的定义，而是在函数中会出现`<Module<T>>::xxx`这样的调用。因此我们可以知道两个实事：

1. Module也是通过宏生成的，那么为了知道Module是啥，可以参照生成`runtime.rs`的方式去在`balances`这个`crate`下展开宏。
2. 在最后的编译结果中，每个模块中的`<Module<T>>`里的`<T>`即是在runtime中生成的`Runtime`。

其他类型同理。

## 总结

从以上介绍可得，只要展开了宏，我们便可以看到宏后的世界，可以发现Substrate实际上帮开发者做了相当多的事情。因此若想要理解Substrate的Runtime，展开宏是必不可少的技能。