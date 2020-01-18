# Substrate 入门 - 环境配置与编译-  （一）

substrate目前已经趋近成熟，因此可以比较系统的对Substrate进行介绍。

本文首先介绍substrate的依赖与编译过程，以此管中窥豹了解substrate的概况。

因笔者的开发环境是ubuntu/deepin/debain，因此后文命令皆基于这个环境下。

## 一 前言

截止目前为止（2019年12月1日，提交`33476f08b3400a07fd7c69cd5bf4ad8f47f11373`），substrate的README已经出现比较大的改动，所有文档集中到了substrate的官方文档中 [substrate.dev](https://substrate.dev/docs/en/getting-started/)。而本身对于环境的配置（linux/mac os）过程已经迁移到了脚本`https://getsubstrate.io`中，即该[链接](https://substrate.dev/docs/en/getting-started/installing-substrate#unix-based-operating-systems)。在这个链接中的介绍为：

```bash
curl https://getsubstrate.io -sSf | bash -s -- --fast
```

但是现在的这种配置方式已经把很多细节隐藏了，且后续执行的命令`--fast`实际上把substrate下载到了一个temp目录中进行编译并安装到了cargo命令目录下，实际上并不方便想要探究substrate的开发者，因此本文将跳开脚本，把本身配置环境与编译的过程重新介绍。

## 二 环境配置

以下过程实际和脚本 `https://getsubstrate.io/index.html`相同，因此若有疑问可参考脚本本身的内容。

环境配置分为两个部分：

1. 依赖库
2. rust编译链

其他linux发行版，mac os可以对应进行参考

### 1. 依赖库

```bash
 apt install -y cmake pkg-config libssl-dev git gcc build-essential git clang libclang-dev
```

在依赖库中大部分都是正常的编译工具，这里重点介绍以下`libssl-dev`。这个库实际上就是提供ssl的支持，在substrate中用于 Libp2p 与 websocket 这两个库的编译依赖，提供ssl保护。因此对应与其他发行版（如centos，redhat等）就是提供类似`openssl`的编译依赖。

在同一发行版的不同版本上编译后需要在另一个版本上运行时（例如在ubuntu 17.04上编译，把执行文件拷贝到ubuntu16.04上），尤其需要注意这个库（ssl）的动态链接依赖。例如在ubuntu 17.04默认的openssl库使用的是 `openssl 1.1` 以上的版本，而 ubuntu 16.04 使用的时 openssl1.0 版本。因此把ubuntu17.04编译的版本放到16.04上会出现动态链接库不匹配的问题。

若需要在ubuntu 17.04及以上版本编译出能在16.04版本上运行的执行文件，可以在编译环境下装好执行环境对应的openssl版本，然后使用

```bash
export OPENSSL_LIB_DIR="<you path>"
export OPENSSL_INCLUDE_DIR="<you path>"
```

导出这两个环境变量，再进行编译。

ps：在脚本中对于arch系指定的openssl版本时 1.0 的，因为笔者不用arch系的linux，尚不可知为何parity团队要指定这个版本。若需要探究的话可参考到 rust-libp2p的相应需求。

### rust 编译链

rust的安装简单得多，首先按照rust官网装好rust环境，确保有`rustup`和`cargo`命令后：

```bash
rustup update nightly  # 安装 nightly 编译链
rustup target add wasm32-unknown-unknown --toolchain nightly  # 对 nightly 编译链添加 wasm 编译target
```

这里说明：

实际上自rust stable 1.38之后 wasm 的这个target就已经可以提供给stable了，但是由于substrate改变了其编译wasm的方式，是在substrate的代码中指定了使用nightly编译wasm，因此这里如果只给stable添加`wasm32-unknown-unknown`是没有用的，必须先提供nightly编译链，再对nightly添加wasm的target。

在该版本的substrate中已经不需要在添加`wasm-gc`做wasm的压缩。（windows的还是看到了这个命令，笔者怀疑是substrate的文档还没有改）

## 三 编译

首先讲substrate项目拉下来

```bash
git clone https://github.com/paritytech/substrate.git
```

参照之前笔者的文章，这里建议导出一个环境变量：

```bash
export WASM_BUILD_TYPE=release
```

在这个环境变量下编译出的wasm才是以release模式编译，否则可能影响出块。

因此进入substrate目录后

```bash
cd substrate
```

编译可以采用

```bash
cargo build
# 若刚才没有导出 WASM_BUILD_TYPE 可以在这里执行以下命令强制设定
#  WASM_BUILD_TYPE=release cargo build
```

这里首先说明，在cargo下：

```bash
cargo run -- <参数>  # 注意这里有两个横杆 --
# 等于
cargo build && ./target/debug/<项目执行文件> <参数>
# 而 release编译
cargo run --release -- <参数>
# 等于
cargo build --release && ./target/release/<执行文件> <参数>
```

对于substrate而言，执行

```bash
cargo run -- --dev
```

等于

```bash
cargo build && ./target/debug/substrate --dev
```

由于substrate项目编译后（`cargo build`）当前会在target目录下生成多个可执行文件，因此请依据自己当前的需求执行。

### 注意的点：

1. wasm是否以release编译

   查看 `target/<debug/release>/wbuild/node-runtime/node_runtime.compact.wasm`的大小，若其大小是1.3M左右，可确定是以release编译，若是8M以上，则一定是以debug编译，此时建议设置好相应环境变量重新编译。

2. 编译后的执行文件

   编译后产生的执行文件有4个

   * substrate  # 即 substrate的node项目执行文件，源码位于`bin/node`，研究substrate的基础入口，cargo run 执行的文件即为该文件
   * node-rpc-client # 与substrate node进行交互的rpc执行文件，源码位于`bin/node/rpc-client`下，注意这个和node交互，不是与node-template交互
   * node-template  # 精简版的node，源码位于`bin/node-template`，与node相比去除了大部分runtime模块，可最为最精简链运行
   * subkey # 用于生成一些公私钥的工具，源码位于`bin/substrate`

对于substrate的研究，只需要关心node项目即可，不擅长js的，可用辅助使用node-rpc-client用来发交易。

## 其他

在substrate的官方脚本中，后续操作还安装了`https://github.com/paritytech/substrate-up`这个项目，该项目是用来生成依托于substrate框架的链的一个模板生成工具。由于substrate更新很频繁，这个工具早已更不上变化，因此不建议安装。

## 总结

由于substrate官方的安装脚本隐藏了很多细节，且没给予用户一些选择的权利，因此本文不建议使用脚本安装，而是梳理了当前substrate的环境配置与编译，并解释了各个步骤的含义。因此用户可根据本文的解释结合自己的环境情况进行substrate的编译环境配置。



