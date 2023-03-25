---
date created: 2023-03-24
date modified: 2023-03-24
---
注意点:
要使用 vercel 或 github pages 或 cloudflare pages，不能用 netlify，因为 netlify 会自动将 url 小写，带来 bug。

核心 2 个点，命令参考 dg3 仓库的 deploy. yaml 文件：

	- frontmatter 需要有 title，且 value 值加双引号。我用 linter 实现自动生成 title，但其不支持自动加双引号，目前我是自己用 sed 命令加上双引号，linter 作者说下个版本将支持 title 的 value 也加双引号。
- 目前已支持 [[wikilink]]，但依旧需要指定绝对引用路径，而我更习惯最短路径的引用方式，所以在构建时，我用 mv 命令将 md 文件批量移动至根目录，从而减少改文件链接的麻烦。

Date now: <% tp.file.title %>
Date now with format: <% tp.date.now("Do MMMM YYYY") %>

Last week: <% tp.date.now("dddd Do MMMM YYYY", -7) %>
Today: <% tp.date.now("dddd Do MMMM YYYY, ddd") %>
Next week: <% tp.date.now("dddd Do MMMM YYYY", 7) %>

Last month: <% tp.date.now("YYYY-MM-DD", "P-1M") %>
Next year: <% tp.date.now("YYYY-MM-DD", "P1Y") %>

File's title date + 1 day (tomorrow): <% tp.date.now("YYYY-MM-DD", 1, tp.file.title, "YYYY-MM-DD") %>
File's title date - 1 day (yesterday): <% tp.date.now("YYYY-MM-DD", -1, tp.file.title, "YYYY-MM-DD") %>

Date tomorrow with format: <% tp.date.tomorrow("Do MMMM YYYY") %>    

This week's monday: <% tp.date.weekday("YYYY-MM-DD", 0) %>
Next monday: <% tp.date.weekday("YYYY-MM-DD", 7) %>
File's title monday: <% tp.date.weekday("YYYY-MM-DD", 0, tp.file.title, "YYYY-MM-DD") %>
File's title next monday: <% tp.date.weekday("YYYY-MM-DD", 7, tp.file.title, "YYYY-MM-DD") %>

Date yesterday with format: <% tp.date.yesterday("Do MMMM YYYY") %>

<% tp.user.ls("Hello World!") %>