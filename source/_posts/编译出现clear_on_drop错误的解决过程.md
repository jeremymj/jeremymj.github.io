---
title: 编译出现clear_on_drop错误的解决过程
tags:
  - substrate
  - 智能合约
abbrlink: 1c5692ea
date: 2020-02-18 21:57:18
---

　　由于远程开发需要部署一台测试服务器，在服务器上运行substrate实例后，接着需要部署测试合约，供同事调试使用。<!--more-->
## 问题描述
　　从github上取回节前提交的代码，开始使用`cargo contract build`编译。在编译的过程中出现问题依赖文件不存在的的错误，出于以前调试substrate合约的经验，估计是依赖源码升级造成的。习惯性的先升级合约编译工具cargo-contract,从0.1.1升级到0.3.0，升级后继续编译，继续提示依赖的源码在依赖库不存在，这下反应过来了，他们最近又是大升级，变化比较大。刚开始还想把依赖项改成最新版的，经过一番尝试后，发现短时间内改不好，只有放弃。最后才想到在依赖库中指定提交的版本（增加rev+提交版本号），同时把使用的cargo-contract编译工具也还原到0.1.1版本，接着再编译修改后的合约项目，出现如下错误提示：

```sh
error: failed to run custom build command for `clear_on_drop v0.2.3`

Caused by:
  process didn't exit successfully: `/home/ming/work/temp/diamond/target/release/build/clear_on_drop-b2b3a807da34adf1/build-script-build` (exit code: 1)
--- stdout
TARGET = Some("wasm32-unknown-unknown")
OPT_LEVEL = Some("z")
HOST = Some("x86_64-unknown-linux-gnu")
CC_wasm32-unknown-unknown = None
CC_wasm32_unknown_unknown = None
TARGET_CC = None
CC = None
CFLAGS_wasm32-unknown-unknown = None
CFLAGS_wasm32_unknown_unknown = None
TARGET_CFLAGS = None
CFLAGS = None
CRATE_CC_NO_DEFAULTS = None
DEBUG = Some("false")
running: "clang" "-Oz" "-ffunction-sections" "-fdata-sections" "-fPIC" "--target=wasm32-unknown-unknown" "-Wall" "-Wextra" "-o" "/home/ming/work/temp/diamond/target/wasm32-unknown-unknown/release/build/clear_on_drop-c7f1c48f1d7a25a0/out/src/hide.o" "-c" "src/hide.c"
cargo:warning=error: unable to create target: 'No available targets are compatible with this triple.'
cargo:warning=1 error generated.
exit code: 1

--- stderr


error occurred: Command "clang" "-Oz" "-ffunction-sections" "-fdata-sections" "-fPIC" "--target=wasm32-unknown-unknown" "-Wall" "-Wextra" "-o" "/home/ming/work/temp/diamond/target/wasm32-unknown-unknown/release/build/clear_on_drop-c7f1c48f1d7a25a0/out/src/hide.o" "-c" "src/hide.c" with args "clang" did not execute successfully (status code exit code: 1).



warning: build failed, waiting for other jobs to finish...
error: build failed
error: Build failed

```
看到这个错误，印象中是跟依赖库中`sp-core`、`sp-io` 是否使用 `std` features 相关，于是把这两个依赖项中的features取消掉，去掉后编译，出现错误：

```rust
error[E0152]: found duplicate lang item `panic_impl`
   --> /home/ming/.cargo/git/checkouts/substrate-7e08433d4c370a21/b443dda/primitives/io/src/lib.rs:864:1
    |
864 | / pub fn panic(info: &core::panic::PanicInfo) -> ! {
865 | |     unsafe {
866 | |         let message = sp_std::alloc::format!("{}", info);
867 | |         logging::log(LogLevel::Error, "runtime", message.as_bytes());
868 | |         core::intrinsics::abort()
869 | |     }
870 | | }
    | |_^
    |
    = note: the lang item is first defined in crate `std` (which `parity_scale_codec` depends on)

error[E0152]: found duplicate lang item `oom`
   --> /home/ming/.cargo/git/checkouts/substrate-7e08433d4c370a21/b443dda/primitives/io/src/lib.rs:875:1
    |
875 | / pub fn oom(_: core::alloc::Layout) -> ! {
876 | |     unsafe {
877 | |         logging::log(LogLevel::Error, "runtime", b"Runtime memory exhausted. Aborting");
878 | |         core::intrinsics::abort();
879 | |     }
880 | | }
    | |_^
    |
    = note: the lang item is first defined in crate `std` (which `parity_scale_codec` depends on)

error: aborting due to 2 previous errors

For more information about this error, try `rustc --explain E0152`.
error: could not compile `sp-io`.

Caused by:
  process didn't exit successfully: `rustc --crate-name sp_io --edition=2018 /home/ming/.cargo/git/checkouts/substrate-7e08433d4c370a21/b443dda/primitives/io/src/lib.rs --error-format=json --json=diagnostic-rendered-ansi --crate-type lib --emit=dep-info,metadata,link -C opt-level=z -C panic=abort -C overflow-checks=on -C metadata=1603ab4b0f5657bd -C extra-filename=-1603ab4b0f5657bd --out-dir /home/ming/work/temp/envtest/diamond/contract/diamond/target/wasm32-unknown-unknown/release/deps --target wasm32-unknown-unknown -L dependency=/home/ming/work/temp/envtest/diamond/contract/diamond/target/wasm32-unknown-unknown/release/deps -L dependency=/home/ming/work/temp/envtest/diamond/contract/diamond/target/release/deps --extern hash_db=/home/ming/work/temp/envtest/diamond/contract/diamond/target/wasm32-unknown-unknown/release/deps/libhash_db-2a1247b66dab3f2e.rmeta --extern codec=/home/ming/work/temp/envtest/diamond/contract/diamond/target/wasm32-unknown-unknown/release/deps/libparity_scale_codec-97d1d02b4b85917a.rmeta --extern sp_core=/home/ming/work/temp/envtest/diamond/contract/diamond/target/wasm32-unknown-unknown/release/deps/libsp_core-ab9564e391d6b38c.rmeta --extern sp_runtime_interface=/home/ming/work/temp/envtest/diamond/contract/diamond/target/wasm32-unknown-unknown/release/deps/libsp_runtime_interface-2c02a210bd05a2c8.rmeta --extern sp_std=/home/ming/work/temp/envtest/diamond/contract/diamond/target/wasm32-unknown-unknown/release/deps/libsp_std-e4cf72917021ed6f.rmeta --cap-lints allow -C 'link-args=-z stack-size=65536 --import-memory'` (exit code: 1)
warning: build failed, waiting for other jobs to finish...
error: build failed
error: Build failed

```
看到这个错误，知道不是依赖项features的原因，毕竟以前是调试通过的，接着又到github上查看clear_on_drop项目相关issue,发现有人问了类似的问题，从回复
```
According to the error message, the issue is that your C compiler (in this case, clang) does not know the wasm32-unknown-unknown target. That is not something which could be fixed by this crate.
```
里面提到可能是clang的原因，一直到这里我都还没有反应过来问题出在哪里。在继续折腾一番后，鉴于到了该洗漱睡觉的时间了，就想到明天再弄吧。在洗漱的过程中想起以前在公司调试的时候也是出现过类似的问题，也花了很长时间都没有排查出来原因，因为在同事的电脑上能够正常编译，所以排除了代码出错，猜测大概率是编译环境的原因，同事使用的是ubuntu 19.10,我使用的是18.04，最后是直接重新安装系统解决这个编译问题（之前就对当前的系统不满意，算是一次性全部把这些问题解决吧）。接着查看家里电脑的系统版本号，居然还是ubuntu 18.04(印象中重装过一次系统)，看到这个版本号后，心里面顿时轻松了些，知道这个问题可以怎么解决了。给我的选择是重装系统（肯定会解决问题）还是尝试先升级`llvm`、`clang`版本(不一定能够解决这个问题)。最后还是想尝试升级`clang`版本号，毕竟家里面的电脑各种开发环境、工具折腾起来需要费不少精力。
## 升级clang
　　在网上查看一下`llvm`相关介绍后，了解这个属于编译系统比较底层的工具库，里面包含很多系统编译需要的工具，使用命令`clang --version`查看了当前的clang版本为6.0，跟github上llvm 的10.0 bate版本差距已经比较大了。编译llvm的过程并不顺利，在生成Makefile过程中没有添加参数，造成后续编译过程很慢，耗时很长，最后也没有编译完全（没有耐心等下去了）。在短暂的放弃后，最后按照[帮助文档](https://clang.llvm.org/get_started.html)设置编译条件`cmake -DLLVM_ENABLE_PROJECTS=clang -G "Unix Makefiles" ../llvm`，在Makefile生成完后使用`make -j8`很快的就编译结束，最后使用`sudo make install`,使用`clang --version` 查看了最新版本为11.0.0,完成了clang升级。
## 验证
　　在clang 升级后，使用`cargo clean`将合约项目所有的以前编译的结果全部清除掉，使用`cargo contract build`重新编译合约，在经过几分钟的紧张时间后，看到wasm文件正确的编译出来了，心里面顿时轻松了，最后使用`cargo contract generate-metadata`编译合约ABI文件(metadata.json)，算是把折腾一天的问题给解决了。同时算是把系统编译环境做了一次更新，以后在编译中不会出现类似错误。

## 总结
- 在以后的开发中，若存在需要直接引用github上的项目源码，需要加上版本号，使用`branch`、`tag`、`rev`等字段；
- 在安装三方库的时候，查看官方文档时要仔细，把里面提到的注意事项读完。