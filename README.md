# zxgangandy 技术博客

网址：https://zxgangandy.github.io/

## 开发

```
npm install
npm run watch
``` 

## 部署

只要把 code push 之后就会自动通过 GitHub actions 部署到 GitHub Pages。

## 产生脸书预览图

请执行以下指令，第一个参数带你的作者 id，第二个带文章名

```
npm run og-image -- "huli" "how-i-hacked-glints-and-your-resume"
```

英文请在最后面加上 "en"
```
npm run og-image -- "huli" "how-i-hacked-glints-and-your-resume" "en"
```

跑完之后，可以在 `og-image-generator/cover.png` 找到你的图片

## 该如何新增作者？

每一个作者都会有个 unique 的 key 來识別，这里假设 key 是 peter。

1. 把个人大头贴放到 `img/authors` 里面
2. 打开 `_data/metadata.json`，在 `authors` 列表里面新增一个 object，格式可参考其他，key 是 `peter`
3. 在 `posts/` 文件夹底下新增 `peter` 文件夹，并复制其它文件夹的 `index.njk`，內容会是作者的个人页面，可自由定制化
4. 在 `img/posts` 文件夹底下新增 `peter` 文件夹，文章的图片可以放到这里面

## 该如何发文？

一样假设作者的 key 是 `peter`

1. 把 repo clone 下來
2. 在 `posts/peter` 里面新增 markdown 文件，里面 frontmatter 格式请參考下面
3. 完成之后 commit + push 就会触发部署流程，大约五分钟后可以在 production 上看到改动

## 文章 frontmatter 格式

```
title: 用 Paged.js 做出适合印成 PDF 的 HTML 网页 // 标题
date: 2018-09-30 // 发文日期
tags: [Front-end, JavaScript] // 标签
author: zxgagnandy // 作者 key
layout: zh-cn/layouts/post.njk // 这固定不变
image: /img/posts/huli/test/cover.png // 选填，会当着预览图
description: 这是一篇关于 Paged.js 的文章 // 选填
```

## 摘要功能
使用一对 `<!-- summary -->` 可以选择將一部分內容显示在摘要区中。

```
<!-- summary -->
应该有許多人都跟小明一樣，有过类似的疑惑。把旧密码寄给我不是很好吗，干嘛强迫我换密码？

这一个看似简单的问题，背后其实藏了许多资讯安全相关的概念，就让我们慢慢尋寻找问题的答案，順便学习一些基本的安全知识吧！
<!-- summary -->

这边不会出现在摘要里面
```

如果想要显示的摘要的內容不在文章里面，可以使用 comment 指定：

```
<!-- summary -->
<!-- 我是会吸引人贴进文章，但沒有整段出現在文章里的摘要 -->
<!-- summary -->
```

使用 comment 指定的摘要支援 HTML，例如`<code>`等。 结尾的 `-->` 目前不可省略。

另外，`<!-- summary -->` 和 comment 标签中的半形空白是必須的。

## 多语言支持

目前支持两个语言：中文跟英文

默认语言为中文，英文的文档都放置在 `en` 资料夾以及 `_includes/en` 里面，要修改的時候必須两份一起修改

英文文章请发表在 `en/posts/` 里面，发表文章与中文相同


## 如何定制化？

### 模板

`_includes` 里面都是 layout 相关的東西，请注意可能会牵一发动全身，里面主要会是各个頁面的 template。

### 样式

css/main.css 所有的样式都在里面，有新增的都放在最下面

## 参考资源

1. [Eleventy Documentation](https://www.11ty.dev/docs/collections/)
2. [Nunjucks 文件](https://mozilla.github.io/nunjucks/templating.html)

此文根据 [error-baker-blog](https://github.com/Lidemy/error-baker-blog) 以及 [eleventy-high-performance-blog](https://github.com/google/eleventy-high-performance-blog) 改写而來

