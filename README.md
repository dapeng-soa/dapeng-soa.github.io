# dapeng-soa.github.io
dapeng-soa website

## start 

```
gem install jekyll

jekyll serve

# => Now browse to http://localhost:4000
```
### 添加blog文章

添加 `.md` 文件至`_post`目录，文件名的命名非常重要。要求一篇文章的文件名遵循下面的格式：

```
年-月-日-标题.md  #如：2018-11-08-start-dapeng.md
```

在 `.md` 头信息中添加
```
---
layout: post
title:  "我是标题"
date:   2018-11-08 09:12:38
author: struy
categories: dapeng
---
```
关于博客的摘要，默认使用文章正文的第一段话为摘要，如下实例：
```
Dapeng是一个支持SOA架构的开发框架，提供微服务的定义、开发、部署、监控、文档、测试、运维的一个完整工具链。

我是后面的文档内容...
```
> 其中`我是后面的文档内容...` 之前的则为摘要

### 添加docs文档

添加 `.md` 文件至`docs`目录，在 `.md` 头信息中添加
```
---
layout: doc
title: 谢谢我是标题
---
```

将新加的doc添加到文档目录中，在`_data/doc_toc.yml`中添加类似链接：

```
- title:      其它
    en_title:   Others
    children:
      - title:  我是之前的文档
        url:    /docs/zh/old
      - title:  我是新加的文档
        url:    /docs/zh/new  # 新加的文档路径，不要文件后缀
```


### 关于添加流程图/时序图/甘特图/
> 使用 `mermaid.js` 实现，与md语法稍有不同

语法参考: https://mermaidjs.github.io

以时序图为例：

1.md语法（代码块包裹）
```
···
sequenceDiagram
A->>B: How are you?
B->>A: Great!
···
```
> 这里使用 ··· 代表反引号

2.mermaid 语法，
> 使用div替换了反引号
```
<div class="mermaid">
sequenceDiagram
A->>B: How are you?
B->>A: Great!
</div>
```
效果：

![](http://www.struy.top/18-11-23/8527709.jpg)

