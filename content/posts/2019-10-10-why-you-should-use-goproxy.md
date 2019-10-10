---
title: (译)为什么要使用 Go 模块代理
date: 2019-10-10T21:51:01+08:00
---

引入Go模块之后，我以为这就是最终解决方案了。我很快意识到事实并非如此。最近，大家开始提倡使用Go模块代理。在研究了利弊之后，我得出结论，这是近年来**最重要的变化**之一。何处此言？是什么使Go模块代理如此特别？

在Go模块中，如果你添加了新的依赖项或者在没有缓存过的新机器上构建Go模块，则它将（`go get`）下载`go.mod`中的所有依赖项，并将其缓存以用于进一步的操作。可以通过使用`vendor/`文件夹并编译时携带`-mod=vendor`参数来绕过缓存（以及下载依赖项）。

但是这两种方法都不甚完美，我们有更好的方案。

<!--more-->

![img](https://d33wubrfki0l68.cloudfront.net/0b770b8c51b14b5b1eb4c73bbb523ea7cd0690d0/ddbec/images/why-you-should-use-a-go-module-proxy-1.jpg)



## （不）使用vendor/文件夹的问题

如果使用`vendor/`文件夹，有以下缺点：

- `vendor/`默认情况下，该`go`命令不再使用该文件夹（在[*模块感知*](https://golang.org/cmd/go/#hdr-Modules_and_vendoring)模式下）。如果你不使用`-mod=vendor`参数，它将不会发挥作用。这烦人的问题驱使其他hacky解决方案来支持Go的旧版本（请参阅：[在Travis CI中使用 go modules 并启用 vendor支持](https://arslan.io/2018/08/26/using-go-modules-with-vendor-support-on-travis-ci/)）
- `vendor/`文件夹，特别是对于大型项目，会占用大量空间。这增加了clone仓库所花费的时间。即使您认为clone仅执行一次，但大多数情况下并非如此。CI / CD系统通常会为每个触发器（例如“pull request”）clone仓库。因此，从长远来看，这将导致更长的build时间并影响团队中的每个人。
- 添加新的依赖项通常会导致改变代码评审的*困难*程度。在大多数情况下，您必须将依赖项与实际的业务逻辑捆绑在一起，这使得难以进行改动。

可能你想说，那我**跳过vendor/**文件夹不就没事了？这也无济于事，你必须解决以下问题：

- `go`会尝试从原先的仓库下载依赖项。但是任何依赖关系都可能在哪天就消失了（[*例如 left-pad包被删惨案*](https://qz.com/646467/how-one-programmer-broke-the-internet-by-deleting-a-tiny-piece-of-code/)）。

- VCS可能已关闭（例如github.com）。在这种情况下，你的项目跑不起来了。

- 有些公司不希望内部网络外部有任何传出连接。因此，删除 `vendor/`文件夹对他们来说是不行的。

- 假设一个依赖项被发布为`v1.3.0`并且你`go get`拉取到本地缓存下来。然而，一旦依赖项的所有者通过推送具有同样 tag的恶意内容来破坏仓库，假如你的在一天没有拉取过缓存的赶紧的计算机上重新build的，那你这次就拉到了这个有恶意内容的版本。为了防止这种情况，您需要将`go.sum`文件与文件一起`go.mod`存储。

- 有些依赖使用的是`git`之外的VCS，例如`hg`（Mercurial），`bzr`（Bazaar）或`svn`（Subversion）。并非所有这些工具都安装在你的电脑（或Dockerfile）上，这常常导致失败。

- `go get`需要获取`go.mod`列出的每个依赖项的源代码以解决递归依赖项（这需要每个依赖项自己的`go.mod`文件）。这极大地减慢了整个构建过程的速度，因为这意味着它必须下载（例如`git clone`）每个存储库以[获取单个文件](https://about.sourcegraph.com/go/gophercon-2019-go-module-proxy-life-of-a-query)。

我们如何改善这种情况？

## 使用Go模块代理的优点

![img](https://d33wubrfki0l68.cloudfront.net/5d86b7737a874531d6c6cb9088ba1f21d4f3d5b8/b4313/images/why-you-should-use-a-go-module-proxy-2.jpg)

默认情况下，`go`命令直接从VCS下载模块。`GOPROXY`环境变量允许在下载源的进一步控制。设置这个环境变量，就会开启`go`**Go模块代理**。

通过将`GOPROXY`环境变量设置为Go模块代理，可以克服上面列出的所有缺点：

- 默认情况下，Go模块代理是缓存和永久存储所有依赖项（在不可变存储中）。这意味着您不再需要使用任何`vendor/`文件夹。
- 摆脱`vendor/`文件夹意味着您的项目将不会在存储库中占用空间。
- 因为依赖项存储在不可变的存储中，所以即使依赖项从Internet上消失了，也可以免受影响。
- 一旦将Go模块存储在Go代理中，就无法覆盖或删除它。这可以让我们的代码免受有人在相同版本上注入恶意代码的攻击。
- 不再需要任何VSC工具来下载依赖项，因为依赖项是通过HTTP提供的（Go proxy在后台使用HTTP）。
- 下载和构建Go模块的速度明显更快，因为Go代理通过HTTP单独提供了源代码（`.zip`存档）`go.mod`。与从VCS进行提取相比，这导致下载花费更少的时间和更快的时间（由于更少的开销）。解决依赖关系的速度也更快，因为`go.mod`可以独立获取（而之前必须获取整个存储库）。Go团队对其进行了测试，[他们发现](https://twitter.com/sajma/status/1155006281263923201?s=21)在带宽高的网络环境下速度提高了3倍，带宽差的网络环境下速度提高了6倍！
- 可以轻松地运行自己的Go代理，这可以让我们更好地控制构建pipeline的稳定性，因为不依赖版本库，可以防止VCS停机这种罕见情况。

如你所见，使用Go模块代理简直完美。但是我们如何使用它呢？如果你不想维护自己的Go模块代理怎么办？让我们研究许多替代选择。

## 如何使用Go模块代理

要开始使用Go模块代理，我们需要将`GOPROXY`环境变量设置为兼容的[Go module proxy](https://golang.org/cmd/go/#hdr-Module_proxy_protocol)。有多种方法：

**1。）**如果`GOPROXY`没有设置，空或设置为`direct`，`go get`会直接从VCS（例如github.com）的下载依赖：

```bash
GOPROXY=""
GOPROXY=direct
```

也可以将其设置为`off`，这表示不访问任何的网络。

```bash
GOPROXY=off
```

**2.）**您可以开始使用公共Go代理。您的选择之一是使用Go小组（*由Google维护*）中的Go代理。可以在这里找到更多信息：[https](https://proxy.golang.org/) : [//proxy.golang.org/](https://proxy.golang.org/)

要开始使用它，您只需设置环境变量：

```bash
GOPROXY=https://proxy.golang.org
```

其他公共代理有：

```bash
GOPROXY=https://goproxy.io
GOPROXY=https://goproxy.cn # proxy.golang.org 被墙了, 可以使用这个代替
```

**3.）**也可以运行多个开源实现并自己托管。例如：

- `Athens`：[https](https://github.com/gomods/athens) : [//github.com/gomods/athens](https://github.com/gomods/athens)
- `goproxy`：[https](https://github.com/goproxy/goproxy)：[//github.com/goproxy/goproxy](https://github.com/goproxy/goproxy)
- `THUMBAI`：[https](https://thumbai.app/) : [//thumbai.app/](https://thumbai.app/)

这需要自己来维护。但可自己决定这个代理是通过公共互联网还是只能内部网络访问。

**4.）**可以购买商业产品：

- `Artifactory`：[https](https://jfrog.com/artifactory/)：[//jfrog.com/artifactory/](https://jfrog.com/artifactory/)

**5.）**甚至可以传递一个`file:///`URL。由于Go模块代理是响应GET请求（没有查询参数）的Web服务器，因此任何文件系统中的文件夹也可以用作Go模块代理。

## 即将进行的Go v1.13更改(已经发布)

Go v1.13版本中的Go proxy会有一些变化，我认为应该强调：

1. 在`GOPROXY`环境变量现在可以设置为逗号分隔的列表。在回退到下一个路径之前，它将尝试第一个代理。

2. 默认值`GOPROXY`会`https://proxy.golang.org,direct`。`direct` token后的所有内容都会被忽略。这也意味着`go get`现在将默认使用`GOPROXY`。如果根本不想使用Go代理，则需要将其设置为`off`。

3. 引入了一个新的环境变量`GOPRIVATE`，是一个逗号分隔的支持glob匹配规则的列表。这可用于绕过`GOPROXY`某些路径的代理，尤其是公司中的私有库（例如：）`GOPRIVATE=*.internal.company.com`。

（白话翻译：GOPROXY设置例如`GOPROXY=https://goproxy.cn,direct`，意思是，首先尝试从`https://goproxy.cn`进行下载，如果没有，启用direct，即从仓库的的源地址下载，比如 githu.com/spf13/viper 这个路径，会使用https://githu.com/spf13/viper 这个路径进行下载。`GOPRIVATE`变量相当于同时设置了`GONOPROXY`和`GONOSUMDB`，即对某个指定的私有仓库，直接从私有仓库拉取而不走github，而且不对模块进行校验）

所有这些更改表明Go模块代理是Go模块的核心和重要部分。

## 结语

同时使用`GOPROXY` 公共和专用的网络有很大的优势。这是一个很棒的功能，可以与`go`命令无缝配合。考虑到它具有许多优点（安全，快速，存储高效），明智的做法是在项目或组织中用起来。在**Go v1.13中**，会默认开启，这是个受欢迎的进步，它改进了Go中依赖管理的状态。

------

[原文链接](https://arslan.io/2019/08/02/why-you-should-use-a-go-module-proxy/)

[延伸阅读](https://segmentfault.com/a/1190000020293616)