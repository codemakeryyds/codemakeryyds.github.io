# 理塘丁真

个人技术博客，使用 [Hugo](https://gohugo.io/) 和 [Hugo Theme Stack](https://github.com/CaiJimmy/hugo-theme-stack) 构建。

## 本地预览

先安装 Hugo Extended `0.158.0` 或更高版本：

```powershell
winget install Hugo.Hugo.Extended
```

```powershell
hugo server -D
```

然后访问 `http://localhost:1313/`。

## 构建

```powershell
hugo --gc --minify --baseURL "https://www.ccyun.cloud/"
```

## 部署

仓库推送到 `codemakeryyds/codemakeryyds.github.io` 后，GitHub Actions 会自动构建并发布到 GitHub Pages。

GitHub Pages 需要设置：

- Repository: `codemakeryyds/codemakeryyds.github.io`
- Pages Source: `GitHub Actions`
- Custom domain: `www.ccyun.cloud`

DNS 需要设置：

- `www.ccyun.cloud` CNAME 指向 `codemakeryyds.github.io`

## 评论

评论系统已按 Giscus 预配置，但默认关闭。启用前需要：

1. 在 GitHub 仓库开启 Discussions。
2. 安装 Giscus App。
3. 到 `https://giscus.app/zh-CN` 生成 `repoID` 和 `categoryID`。
4. 填入 `config/_default/params.toml`，再把 `comments.enabled` 改为 `true`。
