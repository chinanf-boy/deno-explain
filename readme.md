# deno [![explain]][source] [![translate-svg]][translate-list]

[explain]: http://llever.com/explain.svg
[source]: https://github.com/chinanf-boy/Source-Explain
[translate-svg]: http://llever.com/translate.svg
[translate-list]: https://github.com/chinanf-boy/deno-zh

「 基于V8构建的安全TypeScript运行时 」

[关于deno markdown文件 中文][translate-list] | [english](https://github.com/denoland/deno)

---

## explain 🀄️
<!-- doc-templite START generated -->
<!-- docTempliteId = 'github' -->
<!-- time = '2018 9.10' -->
<!-- name = 'denoland' -->
<!-- repo = 'deno' -->
<!-- commit = 'c2663e1d82521e9b68a7e2e96197030a4ee00c30' -->
版本 | 与日期 | 最新更新 | 更多
---|---|---|---
[commit] | ⏰ 2018 9.10 | ![version] | [源码解释][source]

[commit]: https://github.com/denoland/deno/tree/c2663e1d82521e9b68a7e2e96197030a4ee00c30
[version]: https://img.shields.io/github/tag/denoland/deno.svg

<!-- doc-templite END generated -->

## 生活

[help me live , live need money 💰](https://github.com/chinanf-boy/live-need-money)

---

### 目录

<!-- START doctoc -->
<!-- END doctoc -->

> deno 的 构成一开始是 Go语言的版本, 现在 作者开始 用 Rust 来 重构 - 2018 9.10

### deno的语言构成

> 算然说， rust是核心, 但repo却不仅仅有rust, 还有 python, ts, c，... 

- [x] [python](#python) 
    - [ ] deno-gn
- [ ] [rust](#rust)
- [ ] [ts](#ts)
- [ ] //...

### deno构建的介绍

> [官方的构建所需与过程](https://github.com/denoland/deno#build-instructions)

为了确保可重现的构建,Deno在git子模块中具有大部分依赖项目. 然而, 下面是需要您单独安装:

- Rust
- Node
- Python 2. 不是 3.
- ccache (可选但,有助于加速V8的重建. ) .

``` bash
    # Fetch deps.
    git clone --recurse-submodules https://github.com/denoland/deno.git
    cd deno
    ./tools/setup.py

    # Build.
    ./tools/build.py

    # Run.
    ./out/debug/deno tests/002_hello.ts

    # Test.
    ./tools/test.py

    # Format code.
    ./tools/format.py
```

[更多](https://github.com/chinanf-boy/deno-zh#%E6%9E%84%E5%BB%BA%E8%AF%B4%E6%98%8E)

> 若自身构建失败, 可以看看,[二进制包](https://github.com/denoland/deno/releases) / 下载 deno 先行版(for windows) [deno.js.cn](http://deno.js.cn)

### python

#### 1. Setup

> explain start, 庆幸有个伟大项目,而它现在还是个孩子

我们从`./tools/setup.py`开始, 漫漫长路, 始于足下

- [x] [安装/初始化第三方-脚本](./setup.py.md)

#### 2. Build

`./tools/build.py`

- [x] [构建-脚本](./build.py.md)


- [ ] [deno .gn的编写](deno-gn.md)

> 构建脚本, 最重要的且唯一做得事情就是: 启动`v8-js引擎`的构建工具[ninja]

对不熟悉[ninja]的同学, 提及一下:

1. [ninja]为`v8项目`搭建一个使用`gcc`之类编译工具的工作流程再加点平台特性, 以此构建不同平台的二进制/发布版本 的功能
2. `gn` 是为`ninja`服务的, 提供[ninja]专用的`***.ninja`文件
3. 为什么需要另找`gn`提供ninja文件? 可能是因为,更快速或者扩展配置,也可能`***.ninja`语法对用户并不友好
4. `gn`本身也具有自己的`**.gn`文件语法, 也就是可以根据不同平台生成对应`***.ninja`配置与工具链
5. 总结(顺序): 编写`**.gn`文件 -> gn 根据这些文件生成 `**.ninja` -> [ninja] 再根据这些文件运行搭载配置工具链

> 这种专有工具使用的专用后缀文件中的 特定语法 - 可以称为[DSL](http://www.yinwang.org/blog-cn/2017/05/25/dsl)

> `gn` 由 depot_tools 提供 - [一点解释](http://gclxry.com/use-depot_tools-to-manage-chromium-source/)

[ninja]: https://ninja-build.org/

#### 3. Test.

`./tools/test.py`

- [x] [测试-脚本](./test.py.md)

#### 4. Format

`./tools/format.py`

- [x] [格式代码-脚本](./format.py.md)

### rust

### ts