# Pagine v2

Pagine 是一个充分利用多核设施条件的高性能网站内容构建器，
页面构建任务可以在很短时间内完成。

## 特性

- 并行层次化处理和单元任务执行，从头到尾都如此
- 层次化元数据传播使元数据管理简化
- 借助 Git 管理模板和素材，所有模板都可以被分发使用而无需改动
- 模板内建函数
- 在模板中与引擎交互
- 以 HTTP 服务器运行时，在文件发生变动时更新内容

已支持的富文本格式:

- [Markdown](https://markdownguide.org) 并带有 [MathJax](https://www.mathjax.org) 支持
- [Asciidoc](https://asciidoc.org)

## 安装

### 已构建的二进制档

在 [releases](https://github.com/webpagine/pagine/v2/releases) 中选择匹配你操作系统和处理器架构的二进制档。

### 从源码构建

```shell
$ go install github.com/webpagine/pagine/v2/cmd/pagine@v2.1.3
```

> [!TIP]
> 在非 amd64 平台上建议借助 Go mod 安装 Pagine。 [在此](https://go.dev/dl)选择适用你平台的 Go 工具链。

## CLI 用法

Usage of pagine:
- `-public` string
  - Location of public directory. (default `/tmp/$(basename $PWD).public`)
- `-root` string
  - Site root. (default `$PWD`)
- `-serve` string
  - Specify the port to listen and serve as HTTP.


### 生成

```shell
$ cd ~/web/my_site
$ pagine
Generation complete.
```

### 以 HTTP 服务器运行

```shell
$ cd ~/web/my_site
$ pagine --serve :12450
```

文件变动被 `inotify` 捕获后，Pagine 会自动执行生成。

> [!NOTE]
> 增量生成尚未实现。<br/>
> 建议将 `--public` 设在 `/tmp` 下以减少硬盘写入。

自版本 v2.1.0，服务器提供一个 WebSocket 位于 `/ws` 的接口以提供事件监听，例如页面更新。

> [!CAUTION]
> 在公共网络暴露你的 Pagine 服务器可能是有风险的，你应该将生成的页面结果部署到静态页面服务。

## 构成

### 模板

模板是一组页面框架（Go 模板文件）和素材（比如 SVG，样式表和脚本）。

一个木板的自述文件看起来应该像:
```toml
[manifest]
canonical = "com.symboltics.pagine.genesis" # Canonical name
patterns  = [ "/*html" ]                    # Matched files will be added as template file.

[[templates]]
name   = "page"      # Export as name `page`
export = "page.html" # Export `page.html`

[[templates]]
name   = "post"      # Export as name `post`
export = "post.html" # Export `post.html`
```

至于 Go 模板语法，见 [text/template](https://pkg.go.dev/text/template)。

样例:  `page.html`
```html
<html lang="{{ .lang }}">
<head>
  <title>{{ .title }}</title>
  <link rel="stylesheet" href="{{ (getAttr).templateBase }}/css/base.css" />
</head>
<body>
{{ template "header.html" .header }}
<main>{{ render .content }}</main>
</body>
{{ template "footer.html" .footer }}
</html>
```

### 环境

环境是对任务全过程的细节配置。

```toml
ignore = [ "/.git*" ] # Pattern matching. Matched files will not be **copied** to the public/destination.

[use]
genesis = "/templates/genesis"
another = "/templates/something_else" # Load and set alias for the template.
```

建议通过 Git submodule 安装现有模板，例如：

```shell
$ git submodule add https://github.com/webpagine/genesis templates/genesis
```

### 层级

每一“层”拥有元数据和一组将要执行的单元任务。

对于目录，元数据被以表的形式储存在 `metadata.toml`，单元任务则被储存在 `unit.toml`。

每一个模板都有其在“环境”中定义的别名作为命名空间。

层级可以覆盖父层级传播的元数据。

样例: `/metadata.toml`
```toml
[genesis]
title = "Pagine"

[genesis.head]
icon = "/favicon.ico"

[[genesis.header.nav.items]]
name = "Documentation"
link = "/docs/"
```

### 单元

样例: `/unit.toml`
```toml
[[unit]]
template = "genesis:page"       # Which template to use.
output   = "/index.html"        # Where to save the result.
define   = { title = "Pagine" } # Unit-specified metadata.

[[unit]]
template = "genesis:page"
output   = "/404.html"
define   = { title = "Page not found" }
```

## 内建函数

### 算术

| Func  | Args      | Result |
|-------|-----------|--------|
| `add` | a, b: Int | Int    |
| `sub` | a, b: Int | Int    |
| `mul` | a, b: Int | Int    |
| `div` | a, b: Int | Int    |
| `mod` | a, b: Int | Int    |

### 引擎 API

| Func      | Description       |
|-----------|-------------------|
| `getEnv`  | 获取环境信息，调试目的。      |
| `getAttr` | 获取有关单元、层次和模板的元信息。 |

| Environment | Description                |
|-------------|----------------------------|
| `isServing` | 在 Pagine 以服务器运行时返回 `true`。 |

| Attribution    | Description  |
|----------------|--------------|
| `unitBase`     | 单元所在层级的目录路径。 |
| `templateBase` | 记录着模板被放在哪里了。 |

| Func          | Description  |
|---------------|--------------|
| `getMetadata` | 返回模板元数据的根节点。 |

> [!TIP]
> `(getEnv).isServing` 可以被用来启用一些模板中的调试代码，例如页面实时更新。

### 数据处理

| Func         | Args                        | Result |
|--------------|-----------------------------|--------|
| `hasPrefix`  | str: String, prefix: String | Bool   |
| `trimPrefix` | str: String, prefix: String | String |

| Func             | Args                                            | Result                                           | Description                         |
|------------------|-------------------------------------------------|--------------------------------------------------|-------------------------------------|
| `divideSliceByN` | slice: []Any, n: Int                            | [][]Any                                          | 将切片分为 *len(slice) / N* 个切片。         |
| `mapAsSlice`     | map: map[String]Any, **key**, **value**: String | []map[String]{ **key**: String, **value**: Any } | 将表分为以 **key** 为键名，**value** 为值名的切片。 |

### 内容

路径以单元所在路径为基。

| Func             | Args                           | Description             |
|------------------|--------------------------------|-------------------------|
| `apply`          | path: String, data: Any        | 调用一个模板。                 |
| `applyFromEnv`   | templateKey: String, data: Any | 调用一个在 `env.toml` 定义的模板。 |
| `embed`          | path: String                   | 嵌入文件原始内容。               |
| `render`         | path: String                   | 根据扩展名调用渲染器。             |
| `renderAsciidoc` | path: String                   | 渲染并嵌入 Acsiidoc 内容。      |
| `renderMarkdown` | path: String                   | 渲染并嵌入 Markdown 内容。      |

| Format   | File Extension Name |
|----------|---------------------|
| Markdown | `md`                |
| Asciidoc | `adoc`              |

## 部署

### 手动

```shell
$ pagine --public ../public
```

上传 `public` 到你的服务器。

### 通过 GitHub Actions 部署到 GitHub Pages（推荐）

GitHub Actions 工作流配置可以在 [Get Started](https://github.com/webpagine/get-started) 仓库找到。

## 问答

### 为什么要搓另一个生成器？Hugo 不够用吗？

Pagine **不是** Hugo，也不旨在替代 Hugo。

明显的一点是：Pagine 不会在内容中嵌入页面配置（元数据），他们被分开而且理应分开。

Pagine 不止关注 Markdown，我希望能够支持多种来源。

### 我是否能够用 Pagine 来构建复杂的 Web 应用？

它只能帮你摆脱关于静态内容的重复工作。

如果 Pagine 与外部工具集成得好，模板则还可以继续提高生产力。

所以，**这要视情况而定**。

### 与外部工具互操作，比如 npx?

这是可能的，但这一步应该对外部工具透明。

也许需要在 Pagine 完成生成后运行 `npx`。

### Logo 和命名的来源是?

它**不是**浏览器引擎，**也不是**渲染引擎。

Page Gen × Engine ⇒ Pagine. 与 "pagen" 发音相似。

Logo 是一个带书签的打开的书本。

### 以其他编程语言重写?

*我料到会有人这么说*

除非这样做有明显优势，这不会被采纳。

因此：不。目前不在计划内。