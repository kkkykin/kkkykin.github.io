# blog ✧

基于 [Zola](https://www.getzola.org/) + [Even](https://github.com/getzola/even) 主题，部署在 GitHub Pages。

## 写作

在 `content/` 目录下新建 `.md` 文件，格式：

```markdown
+++
title = "文章标题"
date = 2026-07-22
description = "简介"

[taxonomies]
tags = ["tag1", "tag2"]
categories = ["分类"]
+++

正文 Markdown...
```

### 本地预览

```bash
zola serve
```

### 发布

推送到 `main` 分支，GitHub Actions 自动构建部署。
