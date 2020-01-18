# Substrate 入门 - Runtime的wasm与native -（九）

上一篇文章已经介绍了Substrate Runtime的设计概要，结合之前的文章，在此设计的基础上我们必须明白以下几点：

1. Runtime 是一个隔离的环境，其通过api与外界通信，通信时需要加载上指定块高以对应加载的状态
2. Substrate的Runtime是同一份代码编译出两种执行文件，rust的native与能在wasm虚拟机下运行的wasm，通过启动节点时制定的参与及硬编码进入执行文件中的Runtime版本信息觉得执行哪一份文件

因此本篇文章就来具体介绍Runtime编译成wasm所需要的条件。

**本篇只会介绍创建或引入一个包进入Runtime的wasm体系所需要注意的点，至于其需要这么做的原理暂不在本篇中介绍。**

## Runtime 的WASM

这里首先就放出结论：

**Substrate的Runtime的WASM并非标准WASM，而是是一个有条件限制的wasm，并非所有代码都可以编译成Runtime的WASM**

而这个“有条件”即是本篇文章需要指明的内容。

因此很多使用Substrate写Runtime部分的程序员，经常遇到的一个问题就是：

“为什么我把一个库引入Runtime后就编译不过了”

所以到这里，开发者应该明白Substrate的Runtime WASM是要受到条件制约的，因为对于链而言，并不是什么东西都可以放在Runtime环境中的，例如在之前的文章中提到，**由于共识的限制，需要每个节点执行相同代码得到相同的结果**，因此在Runtime中能够执行的应该是“没有副作用”逻辑，如：

1. 系统调用（操作系统函数）
2. 网络、磁盘访问（因此被限制了io）
3. **全局单例变量**（由于在执行每个块的每个交易中wasm环境都是重新创建的，因此在Substrate中都是存到某个存储中，在finalize的时候删除）
4. ... 等等其他

因此在Substrate的Runtime以上的依赖中，需要严格区分`std`与`no_std`，也就是对应着native与Runtime wasm（后文简称wasm）。而且这里的`no_std`是Rust能编译成wasm的库中的一个**子集**（当前剔除了一些类型，详情见：`primitives/std/without_std.rs`中提供的类型）

## Runtime 的依赖

> sp-std  sp-io
>
> |               |
>
> |           frame-support  frame-system
>
> |            /
>
> frame-assets frame-balances ...
>
> |      /
>
> node-runtime

例如当前对于node节点项目而言，依赖的简化版本如上。这里列出这个依赖只是想说明如下：

**对于Runtime而言，node-runtime作为整个依赖树末端的叶，以其为根基往上的所有依赖都要满足上一节提到的“条件”，直到依赖的根**

因此只要编写了会进入`node-runtime`依赖树的crate，那么即便其可能被在非Runtime的部分中被引用到，那么其也必须满足这一些条件。否则**如果这个库不满足这条件，那么只能以`optional`的形式引入，使其只能在native下编译，不能在wasm下编译**

## 能编译成WASM的条件

以下内容若不清楚原理，那么**直接照抄**即可。若希望自己探究原理，请记住在Substrate Runtime设计中，区分WASM和Native编译过程中引入什么库是通过**“条件编译”**控制的。今后写Substrate进阶或深入文章的时候笔者再来自己剖析其中原理。

### 1. Runtime依赖树中的crate的Cargo.toml 的编写

一个能在Runtime里面被引用的库，其`Cargo.toml`必须按照如下方式编写：

1. 在`dependencies`一栏中，一个库的引入要么指定了`default-features = false`，要不指定为`optional = true`。
   1. 若指定为`default-features = false`，那么进入这个被依赖的包，其`Cargo.toml`在`features`这一栏中必须指定过`default = ["std"]`。（非严禁说法，若不知道为什么，照做，若知道，根据自己需求变更）
   2. 若指定为`optional = true`，那么这个被依赖包只可以被native编译，不可以被wasm编译，在当前包中引用它时，需要加上`#[cfg(feature = "std")]`的条件编译
2. 在`[features]`一栏中，一定要按照如下方式写
   1. `default = ["std"]` ，将当前的default feature **指定为`std`，一定不能少**
   2. `std = ["serde", "codec/std", ... ]` ，在`dependencies`中出现过的库名，一定要出现在`std`feature对应的**列表**中，其中在`dependencies`中（以下两点皆为非严禁说法，若不知道为什么，照做，若知道，根据自己需求变更）：
      1. 以`default-features = false`出现的，一定要指定为`xxx/std`
      2. 以`optional = true`出现的，一定指代为当前包名

需要注意的点：

**由于cargo.toml 对编写出错没有显著提示**，因此若出现不明原因错误，检查cargo.toml时一定要注意以下几点：

1. `default-features`的`feature`是带`s`的
2. 若填写的`default-features=false`，那么在`std`指定的列表里必须要留意有没有也添加了`/std`，一定不能忘记在`std`中也出现对应的包，否则如果遗漏，编译到wasm时，可能会有奇怪的错误。
3. `[dev-dependencies]`是用于test的，因此直接正常引入即可

总体来说一定要保证`dependencies`和`features`没有编写出错

### 2. Runtime依赖树中的crate项目内容的编写

对于这个crate项目，需要注意的有以下几点：

1. lib.rs文件中的第一行**必须有**`#![cfg_attr(not(feature = "std"), no_std)]`
2. 引入标准库的类型，如`Vec`, `Result`，`BTreeMap`等等，必须通过`sp-std`这个库引入。
3. 在Runtime的依赖库中，不能出现没有在`sp-std`导入的原本std拥有的标准库类型及宏，例如`String`，宏`println!`，若一定要出现，那么需要通过条件编译`#[cfg(feature = "std")]`包起来，那么被条件编译包起来的部分，显然在编译wasm的时候不会被编译进去，**那么就必须得保证即使wasm没有编译这部分逻辑，那么native与wasm的执行结果也必须保持一致** （例如只有println在里面的话，只会产生的在native下打印的效果，不会影响执行的结果。但是若是有操作逻辑改变了变量状态在条件编译中，那么是一定要禁止的，否则就会导致节点运行过程中产生不同的结果）

举例：

我们来观察Substrate的frame提供的包`frame/assets`，我们修改assets下的内容，然后**在substrate根目录下执行`cargo build`进行编译**

```rust
#![cfg_attr(not(feature = "std"), no_std)]  // 第一行即出现这个条件编译控制语句
// 接下来若编写
// use std::vec::Vec;  // 在wasm下编译不通过
use sp_std::vec::Vec; // 编译通过
// use std::string::String; // 在wasm下编译不通过

decl_module! {
	pub struct Module<T: Trait> for enum Call where origin: T::Origin {
		type Error = Error<T>;
        fn issue(origin, #[compact] total: T::Balance) {
			let origin = ensure_signed(origin)?;
			// println("test"); // wasm下编译不通过
            #[cfg(feature = "std")] {
                println("test"); // 能编译通过，但是在wasm执行中不会打印出来，只有在native执行中才会打印
            }
			//...
		}
    }
}
```

其他情况下例如需要出现能支持serde序列化的结构体，这里列举一个例子`substrate/frame/evm/src/lib.rs:L36`：

```rust
#[derive(Clone, Eq, PartialEq, Encode, Decode, Default)]
#[cfg_attr(feature = "std", derive(Debug, Serialize, Deserialize))]  // 请注意这里支持serde序列化的部分是在 `std` 条件编译下的
/// External input from the transaction.
pub struct Vicinity {
	/// Current transaction gas price.
	pub gas_price: U256,
	/// Origin of the transaction.
	pub origin: H160,
}
```

对于结构体，尤其需要留意`Debug`的trait，这里在`#[derive()]`自动推导的地方，只能使用`RuntimeDebug`而不能使用std下的`Debug`，例如balances模块下的例子`frame/balances/src/lib.rs:L310`：

```rust
/// Struct to encode the vesting schedule of an individual account.
#[derive(Encode, Decode, Copy, Clone, PartialEq, Eq, RuntimeDebug)]
pub struct VestingSchedule<Balance, BlockNumber> {
	/// Locked amount at genesis.
	pub locked: Balance,
	/// Amount that gets unlocked every block after `starting_block`.
	pub per_block: Balance,
	/// Starting block for unlocking(vesting).
	pub starting_block: BlockNumber,
}
```

### 3. 引入一个第三方库兼容Runtime wasm的编译环境

对于以上自己编写Runtime引用到的crate时，若不明白原理，还可以直接照抄即可。但是要引入第三方库的时候就比较麻烦了，需要比较深入了解后可能才知道怎么引入编译。

由于上文提到的关于wasm的标准库的导入问题，因此**可能会出现一个第三方包虽然在自己描述中介绍了支持Rust wasm编译，但是不一定支持Runtime wasm的编译。**

那么这里先说明一个原则：

**请分清引入这个库的作用，确保这份代码的执行必须在Runtime内部，若确定只能在Runtime内部，那么只能尝试将其改成能满足前面说的条件的情况，并且其一系列依赖也要满足条件，若不确定只在Runtime内部运行，那么只把定义抽离出来，将实现通过`runtime_interface`导出到native执行**

若这个库只需要在native下执行（如serde），那么使用Optional引入，只在std下编译。

因此若一个第三方库一定要引入Runtime的编译依赖中，**请再三思量是否是必须要引入**的，因为这并非一件简单的事情。一方面引入新的库，编译会造成wasm文件庞大（因为会引入很多依赖一同编译），一方面将一个库改造成能在Runtime wasm下编译需要很多工作量。

因此例如：

1. 若只是需要一些小的工具函数，那么直接拷贝进入runtime为妙。

2. 若是**需要一些密码库**，那么请参考Substrate实现ed25519，escda等密码学函数的方法，抽离定义，将实现通过`runtime_interface`放在native下实现。这块内容在进阶部分笔者再进行讲解

## 总结

本文简要介绍了Substrate 的Runtime wasm的实现要点，指明Runtime的wasm实际上是rust wasm的一个子集，其使用过程中受到Runtime WASM的限制。因此在编写自己的crate进入Runtime依赖树中时，请按照本文指明的方式进行编写。若采用第三方库引入时，请再三权衡，并根据情况做出判定。**否则可能会在解决如何编译的过程中花费大量时间。**

