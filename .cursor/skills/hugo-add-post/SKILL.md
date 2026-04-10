---
name: hugo-add-post
description: >-
  Adds a Markdown article into a Hugo site with TOML front matter, verifies the
  build, commits, and pushes to origin main. Use when the user asks to add an
  article or file to Hugo, the blog, or the project (e.g. 将某文章加进 hugo /
  博客 / 项目), or to publish a post with git push.
---

# Hugo：新文章接入并提交推送

## When this applies

User intent matches any of:

- 把 / 将 **某篇文章、某个 `.md` 文件** **加进 / 加入** **Hugo / 博客 / 站点 / 项目**
- English: add this post to Hugo, wire up markdown to the blog, publish with git

## Workflow (execute in order)

1. **定位正文**
   - 默认：将文件放在仓库的 `content/posts/` 下（若 `hugo.toml` 未改 `contentDir`，即为 Hugo 常规约定）。
   - 若用户给出的是其他路径下的草稿，先**移动或复制**到 `content/posts/`，文件名建议 `kebab-case.md`，与 URL slug 一致。

2. **补全 TOML front matter**
   - 若文件开头没有 `+++` … `+++` 块，则**在全文最前**插入。
   - **对齐本站**：读 `content/posts/` 里一篇已发布文章（或 `archetypes/default.md`）的字段风格；本仓库常见字段：
     - `date`：ISO8601，时区与既有文章一致（如 `+08:00`）；日期缺省时用对话里给出的「Today's date」或用户指定日。
     - `draft`：默认 `false`；仅当用户明确要求草稿时为 `true`。
     - `title`：与正文第一个 `#` 标题一致，或从文件名合理推断后让用户可改。
     - `tags`：与 `hugo.toml` 里 `menu.main` 标签页一致（本站常有 `web3`、`cpp`、`rust` 等），按文章主题选。
   - 插入后保留正文原有 `#` 标题（与现有数学文一致）。

3. **验证 Hugo**
   - 在仓库根目录执行 `hugo --minify`（或项目一贯使用的 build 命令）。
   - 失败则根据报错修 front matter 或路径，直到通过。

4. **Git：提交并推送到 `origin main`**
   - `git status`：只 stage **与本文相关的**文件（新文章、必要时 `hugo.toml` / 主题配置等），避免无关改动混入。
   - `git commit -m "..."`：说明添加了哪篇文章或接入 Hugo（完整句子，与用户规则一致）。
   - `git push origin main`。
   - **注意**：当前分支若不是 `main`，先切换到 `main` 或按用户指示操作；勿 `force push`。推送失败（权限、网络）时说明原因与已完成的本地步骤。

## Checklist

```
- [ ] 文件在 content/posts/（或项目约定的 posts 路径）
- [ ] TOML 头完整且与站内既有文章风格一致
- [ ] hugo 构建通过
- [ ] git 仅包含相关改动，commit 信息清楚
- [ ] 已 push origin main（或已说明无法推送的原因）
```

## Optional

用户要预览时，可运行 `hugo server -D` 并说明本地 URL（由终端输出为准）。
