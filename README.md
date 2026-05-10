# PersonalDreamIT.github.io

这是从现有 GitHub Pages 静态产物中重建的 Hexo 源码工程。

## 目录说明

- `source/_posts/`：从现有页面迁移回来的 5 篇文章 Markdown 源文件。
- `source/about/`、`source/categories/`、`source/tags/`、`source/404/`：从现有页面迁移回来的独立页面源文件。
- `source/images/`：保留图片迁移说明和文本 SVG；二进制图片不在源码目录重复提交，构建后由脚本从仓库根目录现有 `images/`、`uploads/` 复制到 `public/`。
- `_config.yml`：Hexo 站点配置。
- `_config.next.yml`：NexT 主题配置。
- 根目录现有的 `index.html`、`archives/`、`tags/`、`categories/` 等文件仍是当前线上静态页面产物。

## 本地开发

```bash
npm install
npm run server
```

## 生成静态文件

```bash
npm run clean
npm run build
```

生成结果会输出到 `public/`。如果需要继续使用仓库根目录作为 GitHub Pages 发布目录，请在确认 `public/` 内容无误后，再将生成结果同步到发布分支或根目录。

## 迁移说明

源码丢失后，本次重建尽量保留了现有站点中的文章标题、发布日期、更新时间、分类、标签、关于页内容、资源引用、Hexo/NexT 版本与本地搜索配置。为避免在源码迁移提交中重复添加二进制文件，构建脚本会复用仓库根目录现有的 `images/` 和 `uploads/` 资源。原静态页面中的 Gitalk OAuth 字段未写入源码配置，避免在新源码中继续固化敏感配置；需要启用评论时，请在 `_config.next.yml` 中重新填写专用 OAuth App 配置。
