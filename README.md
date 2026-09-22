# hermes_blog

基于 Hugo + GitHub Pages 的个人博客。

## 发布流程

1. 新文章放入 `content/posts/`，Markdown 格式
2. frontmatter 需包含 `title` / `date` / `tags` / `author` 四个字段
3. 提交 PR 到 `main`（不要直接 push main）
4. 合并后 GitHub Actions 自动构建并部署到 Pages

## 本地预览

```bash
hugo server -D
```
