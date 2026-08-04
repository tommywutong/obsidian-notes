---
title: "TommyWu's Lab 使用指南"
published: 2026-05-04
description: "这篇文章记录 TommyWu's Lab 的日常写作、同步、预览、构建和发布流程。"
tags: ["Blog", "Astro", "Fuwari", "Obsidian"]
category: "工程"
pinned: true
draft: false
---

# TommyWu's Lab 使用指南

这篇指南记录这个博客的日常使用方式。它适合当成自己的操作手册：以后忘了怎么新增文章、怎么同步、怎么预览、怎么排查问题，就回来翻这一篇。

当前博客由三部分组成：

- 博客项目：`/Users/tommywu/tommywu-lab`
- Obsidian 公开写作目录：`/Users/tommywu/Obsidian/TommyWu Lab`
- 静态站点框架：Fuwari / Astro

只有放在 `TommyWu Lab` 这个 Obsidian 文件夹里的内容会被同步到网站。其他私人笔记不会被博客读取。

## 推荐发布流程

现在可以用一条命令完成同步、检查、构建、提交和推送：

```bash
cd /Users/tommywu/tommywu-lab
pnpm publish "Update blog"
```

这条命令会依次执行同步文章、类型检查、生产构建、提交和推送。如果没有任何改动，它会提示 `No changes to publish.`，不会创建空提交。

Cloudflare Pages 会在推送到 GitHub 后自动部署。

## 新建文章命令

可以用命令直接在 Obsidian 公开写作目录里创建文章：

```bash
cd /Users/tommywu/tommywu-lab
pnpm new-post ios-runtime-message-forwarding "Runtime 消息转发流程"
```

它会创建：

```text
/Users/tommywu/Obsidian/TommyWu Lab/ios-runtime-message-forwarding.md
```

默认会设置 `draft: true`，写完准备发布时改成：

```md
draft: false
```

## Obsidian 模板

我也放了一份 Obsidian 模板：

```text
/Users/tommywu/Obsidian/_Templates/TommyWu Lab 技术博客模板.md
```

模板不会被同步到网站，因为它不在 `TommyWu Lab` 公开目录里。

## 日常写作流程

最常用的流程是：

```bash
cd /Users/tommywu/tommywu-lab
pnpm dev
```

然后打开：

```text
http://127.0.0.1:4321/
```

接着在 Obsidian 里编辑：

```text
/Users/tommywu/Obsidian/TommyWu Lab
```

每次启动 `pnpm dev` 时，博客会先自动执行同步，把 Obsidian 公开目录里的 Markdown 文件同步到博客项目的 `src/content/posts`。

如果开发服务器已经开着，新增或修改文章后，可以手动同步一次：

```bash
pnpm sync-posts
```

## 新增一篇文章

在 Obsidian 的 `TommyWu Lab` 文件夹中新建一个 `.md` 文件，例如：

```text
my-first-technical-note.md
```

文件顶部需要写 frontmatter：

```md
---
title: "我的第一篇技术笔记"
published: 2026-05-04
description: "这是一篇测试文章。"
tags: ["Astro", "Obsidian"]
category: "工程"
draft: false
---
```

然后在下面写正文：

```md
# 我的第一篇技术笔记

这里是正文内容。
```

写完后运行：

```bash
cd /Users/tommywu/tommywu-lab
pnpm sync-posts
```

如果本地服务器开着，页面会自动刷新或在刷新浏览器后看到新文章。

## frontmatter 字段说明

每篇文章建议都包含这些字段：

```md
---
title: "文章标题"
published: 2026-05-04
description: "文章摘要，会用于 SEO、首页卡片和搜索结果。"
tags: ["Tag1", "Tag2"]
category: "分类"
draft: false
---
```

字段含义：

- `title`：文章标题，必填。
- `published`：发布日期，必填，格式必须是 `YYYY-MM-DD`。
- `updated`：最后更新时间，选填，格式同样是 `YYYY-MM-DD`。
- `description`：文章摘要，建议填写。
- `tags`：标签，可以有多个。
- `category`：分类，通常放一个大方向，比如 `工程`、`AI`、`iOS`、`读书`。
- `series`：系列名，选填，例如 `iOS Runtime 系列`。
- `seriesSlug`：系列地址，选填，例如 `ios-runtime`，会生成 `/series/ios-runtime/`。
- `seriesOrder`：系列内排序，选填，数字越小越靠前。
- `pinned`：是否置顶，选填，`true` 表示首页优先展示。
- `draft`：是否草稿。`false` 表示发布，`true` 表示隐藏。

同步脚本会检查 `title` 和 `published`。如果缺少这两个字段，同步会失败，这样可以避免不完整的文章被发布。

## 系列文章

如果一组文章属于同一个专题，可以加系列字段：

```md
---
title: "Category 的加载流程"
published: 2026-05-05
updated: 2026-05-05
description: "整理 Objective-C Category 在 Runtime 中的加载过程。"
tags: ["iOS", "Objective-C", "Runtime"]
category: "iOS"
series: "iOS Runtime 系列"
seriesSlug: "ios-runtime"
seriesOrder: 1
draft: false
---
```

这样文章页会显示系列导航，同时网站会生成：

```text
/series/
/series/ios-runtime/
```

建议 `seriesSlug` 使用英文、小写、短横线，不要随意改。改了以后旧链接会变化。

## 文章置顶

想让某篇文章在首页靠前，可以加：

```md
pinned: true
```

适合置顶的文章：

- 使用指南
- 长期维护的索引文章
- 正在更新的系列第一篇
- 项目介绍页

不要给太多文章都加置顶，否则置顶就失去意义。

## 相关文章推荐

文章页底部会自动根据 `category` 和 `tags` 推荐相关文章。

想让推荐更准确，可以保持同一主题的标签一致，例如 iOS Runtime 相关文章都使用：

```md
tags: ["iOS", "Objective-C", "Runtime"]
category: "iOS"
```

## 代码块增强

当前代码块已经支持复制按钮、行号、文件名和高亮行。

写文件名：

````md
```ts title="src/config.ts"
export const siteConfig = {
  title: "TommyWu's Lab",
}
```
````

高亮指定行：

````md
```ts {2,4-6}
const name = "TommyWu"
console.log(name)
```
````

也可以同时写文件名和高亮：

````md
```swift title="ViewController.swift" {3}
final class ViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
    }
}
```
````

## GitHub 仓库卡片

文章里可以插入 GitHub 仓库卡片：

```md
::github{repo="tommywutong/tommywu-lab"}
```

页面会自动请求 GitHub API 展示仓库描述、语言、star、fork 和 license。

## LeetCode / 项目卡片

当前已经有 `/projects/` 项目页，适合放项目、专题、LeetCode 练习归档。

如果一篇文章想链接到项目页，可以直接写：

```md
[查看项目页](/projects/)
```

后续如果想把 LeetCode 题解做得更像题库，可以继续扩展成独立数据文件，比如按题号、难度、语言和标签生成列表。

现在已经有独立 LeetCode 题库页：

```text
/leetcode/
```

题库数据在：

```text
/Users/tommywu/tommywu-lab/src/data/leetcode.ts
```

以后新增题目时，在 `leetcodeProblems` 里追加一项即可。

## 隐藏文章

如果文章还没写完，但你想保留在公开写作目录里，可以设置：

```md
draft: true
```

Fuwari 会把它当成草稿，不展示在网站上。

等要发布时再改回：

```md
draft: false
```

## 删除文章

删除文章只需要两步：

1. 从 Obsidian 的 `TommyWu Lab` 文件夹里删除对应 `.md` 文件。
2. 在博客项目中运行同步：

```bash
cd /Users/tommywu/tommywu-lab
pnpm sync-posts
```

同步脚本会清空并重建博客项目里的文章目录，所以网站内容会和 Obsidian 公开目录保持一致。

## 修改文章文件名

文章 URL 和文件名有关。

例如：

```text
hello-tommywu-lab.md
```

会生成类似这样的地址：

```text
/posts/hello-tommywu-lab/
```

如果你修改文件名，文章地址也会变化。已经发出去的旧链接可能会失效，所以正式发布后的文章文件名尽量少改。

## 图片使用

最稳妥的方式是把图片放在文章旁边。

例如：

```text
TommyWu Lab/
  my-post.md
  my-image.png
```

在 Obsidian 里可以写：

```md
![[my-image.png]]
```

同步脚本会把它转换成标准 Markdown：

```md
![my-image.png](./my-image.png)
```

也可以手动写标准 Markdown 图片：

```md
![图片说明](./my-image.png)
```

支持同步的图片格式包括：

- `.png`
- `.jpg`
- `.jpeg`
- `.gif`
- `.webp`
- `.avif`
- `.svg`

## Obsidian 双链

同步脚本会简单处理 Obsidian 双链：

```md
[[hello-tommywu-lab]]
```

会转换成：

```md
[hello-tommywu-lab](/posts/hello-tommywu-lab/)
```

如果你写：

```md
[[hello-tommywu-lab|这篇测试文章]]
```

会转换成：

```md
[这篇测试文章](/posts/hello-tommywu-lab/)
```

注意：双链转换是按文件名推断 URL 的，所以文件名最好使用英文、小写和短横线。

## 推荐文件命名

建议使用英文小写和短横线：

```text
build-my-blog-with-astro.md
notes-on-rag.md
ios-memory-debugging.md
```

不太建议使用：

```text
我的第一篇博客.md
My First Blog.md
2026/05/04/文章.md
```

中文标题可以放在 `title` 里，文件名保持稳定、简短、URL 友好。

## 本地预览

启动开发服务器：

```bash
cd /Users/tommywu/tommywu-lab
pnpm dev
```

访问：

```text
http://127.0.0.1:4321/
```

常看的页面：

- 首页：`http://127.0.0.1:4321/`
- 归档：`http://127.0.0.1:4321/archive/`
- 系列：`http://127.0.0.1:4321/series/`
- 项目：`http://127.0.0.1:4321/projects/`
- LeetCode：`http://127.0.0.1:4321/leetcode/`
- 时间线：`http://127.0.0.1:4321/timeline/`
- 标签云：`http://127.0.0.1:4321/tags/`
- 分类：`http://127.0.0.1:4321/categories/`
- 友链：`http://127.0.0.1:4321/friends/`
- Now：`http://127.0.0.1:4321/now/`
- Uses：`http://127.0.0.1:4321/uses/`
- Stack：`http://127.0.0.1:4321/stack/`
- 关于：`http://127.0.0.1:4321/about/`
- RSS：`http://127.0.0.1:4321/rss.xml`

## 构建检查

正式发布前建议运行：

```bash
cd /Users/tommywu/tommywu-lab
pnpm check
pnpm build
```

`pnpm check` 用来检查 Astro、TypeScript 和内容结构。

`pnpm build` 会生成生产环境静态网站，并生成搜索索引、RSS、sitemap。

当前 `pnpm build` 已经配置为先自动同步 Obsidian 文章，所以不用额外先跑 `pnpm sync-posts`。

## 常用命令速查

```bash
cd /Users/tommywu/tommywu-lab
```

同步文章：

```bash
pnpm sync-posts
```

本地预览：

```bash
pnpm dev
```

类型和内容检查：

```bash
pnpm check
```

生产构建：

```bash
pnpm build
```

预览生产构建结果：

```bash
pnpm preview
```

## 修改网站外观和资料

可以改，而且这个博客本质上就是一个普通的 Astro/Fuwari 项目。常见信息主要在：

```text
/Users/tommywu/tommywu-lab/src/config.ts
```

常用修改项：

- 网站标题：`siteConfig.title`
- 网站副标题：`siteConfig.subtitle`
- 首页 Hero：`src/components/HomeHero.astro`
- 最近在学/最近在做：`src/data/profile.ts`
- 项目页数据：`src/data/projects.ts`
- 友链数据：`src/data/friends.ts`
- LeetCode 题库数据：`src/data/leetcode.ts`
- 右侧个人卡片名字：`profileConfig.name`
- 右侧个人卡片简介：`profileConfig.bio`
- 头像：`profileConfig.avatar`
- banner：`siteConfig.banner`
- favicon：`siteConfig.favicon`
- 主题色：`siteConfig.themeColor`
- 导航栏链接：`navBarConfig.links`
- GitHub 等社交链接：`profileConfig.links`

### 修改头像

当前头像配置在：

```ts
avatar: "assets/images/tommy-avatar.jpg"
```

最简单的替换方式：

1. 把新头像放到：

```text
/Users/tommywu/tommywu-lab/src/assets/images/
```

2. 例如文件名叫：

```text
tommy-avatar.png
```

3. 修改 `/Users/tommywu/tommywu-lab/src/config.ts`：

```ts
avatar: "assets/images/tommy-avatar.png"
```

4. 本地检查：

```bash
pnpm build
```

5. 提交并推送后，Cloudflare Pages 会重新部署。

建议头像使用正方形图片，比如 `512x512` 或 `1024x1024`，格式用 `.png`、`.jpg`、`.jpeg` 或 `.webp` 都可以。

### 修改关于页

关于页内容在：

```text
/Users/tommywu/tommywu-lab/src/content/spec/about.md
```

这里适合写站点介绍、个人介绍、联系方式、这个博客会记录什么内容。

### 修改首页 Hero

首页 Hero 在：

```text
/Users/tommywu/tommywu-lab/src/components/HomeHero.astro
```

这里可以修改：

- 首页标题下方的标签。
- 首页按钮。
- GitHub 卡片文案。
- 技术栈图标墙。
- 最近在学。
- 最近在做。

改完后运行：

```bash
pnpm build
```

### 修改 banner

站点 banner 配置在 `/Users/tommywu/tommywu-lab/src/config.ts`：

```ts
banner: {
  enable: false,
  src: "assets/images/demo-banner.png",
  position: "center",
}
```

想启用 banner：

```ts
banner: {
  enable: true,
  src: "assets/images/my-banner.jpg",
  position: "center",
}
```

然后把图片放到：

```text
/Users/tommywu/tommywu-lab/src/assets/images/my-banner.jpg
```

### 修改默认文章封面

如果文章没有写 `image`，网站会按分类或标签给它一个默认封面。

默认封面在：

```text
/Users/tommywu/tommywu-lab/public/covers/
```

映射规则在：

```text
/Users/tommywu/tommywu-lab/src/utils/cover-utils.ts
```

比如想让 `AI` 分类默认使用新封面，可以修改：

```ts
AI: "/covers/ai.svg"
```

### 开启评论

评论组件已经预留好了，使用 Giscus。配置在：

```text
/Users/tommywu/tommywu-lab/src/config.ts
```

找到 `commentsConfig`，去 [Giscus](https://giscus.app/) 生成 `repoId` 和 `categoryId`，填好后把：

当前已经使用独立公开评论仓库：

```text
tommywutong/tommywu-lab-comments
```

Giscus 官方要求：

- 评论仓库必须是 public。
- GitHub Discussions 必须开启。
- Giscus App 必须安装到这个仓库。

当前配置已经开启：

```ts
enable: true
```

如果线上评论区提示 Giscus App 没有安装，去这个地址安装，并选择 `tommywutong/tommywu-lab-comments`：

```text
https://github.com/apps/giscus/installations/new
```

### 开启访问统计

访问统计也已经预留配置，支持 Cloudflare Web Analytics 和 Umami。

配置在：

```text
/Users/tommywu/tommywu-lab/src/config.ts
```

找到 `analyticsConfig`。如果使用 Cloudflare Web Analytics，把 token 填进去并开启：

```ts
cloudflare: {
  enable: true,
  token: "你的 token",
}
```

如果使用 Umami，填入 `websiteId` 和 `scriptUrl`。

### 修改 Now / Uses / Stack 页面

这三个页面的数据在：

```text
/Users/tommywu/tommywu-lab/src/data/profile.ts
```

对应页面：

```text
/now/
/uses/
/stack/
```

### 修改 favicon

favicon 文件在：

```text
/Users/tommywu/tommywu-lab/public/favicon/
```

如果只是想快速替换，可以保留原文件名，直接替换对应尺寸的图片。更系统的做法是在 `/Users/tommywu/tommywu-lab/src/config.ts` 里的 `siteConfig.favicon` 配置新路径。

### 修改主题色

主题色在 `/Users/tommywu/tommywu-lab/src/config.ts`：

```ts
themeColor: {
  hue: 210,
  fixed: false,
}
```

`hue` 是色相，范围是 `0` 到 `360`。例如：

- `0`：偏红
- `120`：偏绿
- `210`：偏蓝
- `280`：偏紫

`fixed: false` 表示访客可以自己调整主题色；改成 `true` 就会固定站点主题色。

## 发布前检查清单

发布文章前可以扫一遍：

- `title` 是否准确。
- `published` 是否是正确日期。
- `description` 是否能概括文章内容。
- `tags` 是否不要太散。
- `category` 是否和已有分类一致。
- `draft` 是否已经改成 `false`。
- 图片是否放在公开目录里。
- 文章里是否有不想公开的私人信息。
- 本地 `pnpm build` 是否通过。

## 常见问题

### 同步失败：缺少 title

说明某篇文章没有写：

```md
title: "文章标题"
```

补上后重新运行：

```bash
pnpm sync-posts
```

### 同步失败：缺少 published

发布日期必须是：

```md
published: 2026-05-04
```

不要写成：

```md
published: 2026/05/04
published: May 4, 2026
```

### 文章没有显示

优先检查：

- 文件是否在 `/Users/tommywu/Obsidian/TommyWu Lab`。
- 文件扩展名是否是 `.md` 或 `.mdx`。
- `draft` 是否是 `true`。
- 是否运行过 `pnpm sync-posts` 或重新启动过 `pnpm dev`。

### 图片没有显示

优先检查：

- 图片是否也在 `TommyWu Lab` 目录里。
- 图片路径是否写对。
- 图片文件名是否和 Markdown 里的引用一致。
- 图片格式是否是支持同步的格式。

### 搜索搜不到最新文章

搜索索引是生产构建时生成的。本地开发环境里搜索可能不是最终效果。

要测试搜索，运行：

```bash
pnpm build
pnpm preview
```

然后在预览页面里测试搜索。

## 后续可以优化的地方

现在这套流程已经能稳定写作和发布。之后还可以继续优化：

- 配置真实域名到 `astro.config.mjs` 的 `site`。
- 接入 Cloudflare Pages 自动部署。
- 给 Obsidian 增加文章模板。
- 给常用分类和标签定一套命名规则。
- 增加自动提交和发布脚本。

先把写作流程跑顺，比一开始就追求完美更重要。
