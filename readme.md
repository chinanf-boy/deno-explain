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

- [ ] [python](#python) 
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

#### 3. Test.

`./tools/test.py`

- [x] [测试-脚本](./test.py.md)

#### 4. Format

`./tools/format.py`

- [x] [格式代码-脚本](./format.py.md)

### ts
### rust