---
date created: 2023-03-24
date modified: 2023-03-24
---
注意点:
要使用 vercel 或 github pages 或 cloudflare pages，不能用 netlify，因为 netlify 会自动将 url 小写，带来 bug。

核心 2 个点，命令参考 dg3 仓库的 deploy. yaml 文件：

	- frontmatter 需要有 title，且 value 值加双引号。我用 linter 实现自动生成 title，但其不支持自动加双引号，目前我是自己用 sed 命令加上双引号，linter 作者说下个版本将支持 title 的 value 也加双引号。
- 目前已支持 [[wikilink]]，但依旧需要指定绝对引用路径，而我更习惯最短路径的引用方式，所以在构建时，我用 mv 命令将 md 文件批量移动至根目录，从而减少改文件链接的麻烦。