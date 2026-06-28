# _config.yml 分析报告

> 比对来源：
> - **当前仓库**: `/Users/joy/Git/joysnow.github.io/_config.yml`
> - **参考示例**: [Shiina18](https://raw.githubusercontent.com/Shiina18/shiina18.github.io/master/_config.yml)（NexT 主题成熟使
用案例）
> - **原始模板**: [Simpleyyt/jekyll-theme-next](https://github.com/Simpleyyt/jekyll-theme-next/blob/master/_config.yml)（配置源
头）

---

## 1. ✅ 已正确配置，无需修改

| 配置项 | 当前值 | 说明 |
|--------|--------|------|
| `title` / `subtitle` / `description` / `author` | JoySnow / Joy's personal blog | 基本信息完整 |
| `language` | `en` | 与参考一致 |
| `url` / `baseurl` / `permalink` | `https://joysnow.github.io` / `/:title/` | 永久链接格式清晰 |
| `paginate` | `10` | 与参考一致 |
| `markdown` / `kramdown` | GFM + rouge | 标准配置 |
| `scheme` | Mist | 与 Shiina18 相同 |
| `sidebar.position` / `display` / `offset` | left / always / 12 | 良好 |
| `toc` | enabled + numbering | 完整 |
| `avatar` | GitHub 头像 | 已设置 |
| `post_meta` | item_text, created_at, categories | 完整 |
| `plugins` | jemoji, sitemap, feed | 标准栈 |
| `defaults` | layout post, comments true | 完整 |
| `font` | Lato enable | 与参考一致 |
| `canonical` | true | SEO 基础 |
| `copyright` / `since` | true / 2018 | 完整 |
| `exclude` | Gemfile, vendor 等 | 干净 |
| `use_motion` | `true` | 动画开启 |

---

## 2. 🟡 缺失但推荐添加（无冲突，纯增益）

### 2.1 基础 SEO / 信息完善

| 配置 | 来源 | 说明 | 推荐值 | 验证方式 |
|------|------|------|--------|---------|
| `favicon` | 两者皆有 | 当前未配置 favicon，浏览器标签页缺图标 | `/assets/favicon.ico`（需准备图标文件） | 浏览器标签页显示图标；`curl -sI https://joysnow.github.io/assets/favicon.ico` 返回 200 |
| `keywords` | 两者皆有 | 站点关键词，利于搜索引擎收录 | `"Jekyll, NexT, blog, JoySnow"` | 页面源码中存在 `<meta name="keywords" content="...">` |
| `seo` | Shiina18: `true` / 模板: `false` | 当前为 `false`；设为 `true` 改善标题层级 SEO | `true` | 文章页 `<title>` 格式为 `post.title | site.title` |
| `index_with_subtitle` | 两者皆有 | 控制首页副标题是否参与 SEO，不设也没副作用 | `false` | 首页 `<title>` 不包含副标题内容 |
| `rss` | 模板有 | 当前为空字段；可显式设空或指向 feed 链接 | 留空（自动使用 feed 链接） | `curl https://joysnow.github.io/atom.xml` 返回有效 RSS XML |
| `feed.path` | 模板有 | 显式指定 atom.xml 路径 | `atom.xml` | `/atom.xml` 路径可访问（rss 验证已覆盖） |

### 2.2 文章体验改善

| 配置 | 来源 | 说明 | 推荐值 | 验证方式 |
|------|------|------|--------|---------|
| `scroll_to_more` | 两者都有 | 点击「阅读更多」后自动滚动到 <!-- more --> 处 | `true` | 首页点击摘要「阅读更多」链接，页面自动滚动到正文对应位置 |
| `save_scroll` | Shiina18: `true` | Cookie 保存阅读位置，返回时自动恢复 | `true` | 滚动后刷新页面，自动恢复到上次阅读位置 |
| `post_wordcount` | 两者都有 | 文章字数 / 阅读时间显示 | `item_text: true`, `wordcount: true`, `min2read: true` | 文章页顶部或底部显示「字数 / 阅读时间」信息 |
| `post_copyright` | 模板有 | 文章底部版权声明块 | `enable: true`（配合 CC 协议） | 文章底部出现版权声明块，含作者和原文链接 |
| `creative_commons` | 两者皆有 | 声明博客采用的知识共享协议 | `by-nc-sa`（最常用） | 版权声明中显示 CC BY-NC-SA 图标/文字 |
| `b2t` (sidebar) | 模板有 | 回到顶部按钮 | `true` | 页面右下角出现回到顶部按钮（滚动后可见） |
| `scrollpercent` (sidebar) | 模板有 | 回到顶部按钮上显示滚动百分比 | `true` | 回到顶部按钮上显示当前滚动百分比数字 |

### 2.3 菜单项补充

| 配置 | 来源 | 说明 |
|------|------|------|
| `about: /about/` | Shiina18 | 已存在 `about.md` 页面，但菜单未启用。可考虑添加 |

### 2.4 第三方服务（可选，默认关闭）

| 配置 | 来源 | 说明 |
|------|------|------|
| `mathjax` | 两者皆有 | LaTeX 数学公式渲染，`per_page: true` 按需加载，不拖慢速度 |
| `gitalk` | Shiina18 使用中 | GitHub Issue 驱动的评论系统，作为 Disqus 的备选 |
| `algolia_search` | 模板有 | 全文搜索替代方案，比 local_search 更高效（需注册 Algolia） |
| `busuanzi_count` | 模板有 | 不蒜子页面访问统计，简单轻量无需注册 |
| `leancloud_visitors` | 模板有 | LeanCloud 文章阅读次数统计（需注册） |
| `rating` | 模板有 | 文章星级评分（需 widgetpack 账号） |
| `exturl` | 模板有 | 外部链接 Base64 加密，防泄露 |
| `google_site_verification` | Shiina18 有 | Google Search Console 验证 |
| `baidu_site_verification` | Shiina18 有 | 百度站长验证（中文 SEO 可选） |
| `baidu_push` | Shiina18: `true` | 自动推送 URL 到百度（可选） |

### 2.5 视觉效果

| 配置 | 来源 | 说明 |
|------|------|------|
| `pace` / `pace_theme` | Shiina18 + 模板 | 页面顶部加载进度条。Shiina18 启用 `minimal` 主题 |
| `canvas_nest` | 两者皆有 | 背景粒子网络动效，关闭不影响性能 |
| `three_waves` | 模板有 | Three.js 波浪动效 |
| `canvas_lines` | 模板有 | 线条背景动效 |
| `canvas_sphere` | 模板有 | 球体背景动效 |
| `canvas_ribbon` | 模板有 |  ribbon 彩带动效（仅 Pisces 方案） |
| `authoricon` | 模板有 | 页脚年份和作者之间的图标（`heart`） |

### 2.6 供应商 CDN 配置（性能优化）

模板和 Shiina18 的 `vendors` 段比当前仓库完整得多。当前仓库只有 `_internal: assets/lib`。可以添加 CDN 覆盖项以加速加载：

```yaml
vendors:
  _internal: assets/lib
  jquery: https://cdnjs.cloudflare.com/ajax/libs/jquery/2.1.3/jquery.min.js
  fancybox:
  fancybox_css:
  fastclick:
  lazyload:
  velocity:
  velocity_ui:
  ua_parser:
  fontawesome:
  pace:
  pace_css:
  canvas_nest:
  # ... 其他按需添加
```

所有值为空时走本地资源，填 URL 则走 CDN，无副作用。

---

## 3. 🔴 当前与参考的关键差异（需注意）

### 3.1 `auto_excerpt` —— 差异项

| | 当前仓库 | Shiina18 |
|--|---------|----------|
| `enable` | `true` | `false` |
| `length` | `150` | （无） |

Shiina18 和模板都建议手动使用 `<!-- more -->` 精确控制摘要，而不是自动截取。当前仓库已启用自动摘要，这可能导致：
- 摘要截断位置不理想
- 与 `excerpt_description: true` 配合时，某些页面行为不确定

**建议**：保持当前设置（不影响功能），但日后考虑逐步切换为手动 `<!-- more -->`。

### 3.2 `fancybox` —— 差异项

| | 当前仓库 | Shiina18 |
|--|---------|----------|
| fancybox | `false` | `true` |

当前未启用 fancybox（图片灯箱）。Shiina18 启用了。如果博客有图片展示需求，开启 fancybox 可以提升体验。无冲突。

### 3.3 `post_meta.updated_at`

| | 当前仓库 | Shiina18 |
|--|---------|----------|
| updated_at | `false` | `true` |

Shiina18 显示文章更新日期。如果文章经常更新，建议开启。

**验证方式**：打开任意文章页，在文章元信息区域（发表日期旁）应显示 `Updated on YYYY-MM-DD`；DOM 中存在 `.post-meta .updated-at` 元素。

### 3.4 `toc.number`

| | 当前仓库 | Shiina18 |
|--|---------|----------|
| number | `true` | `false` |

主观偏好，当前选择无问题。

---

## 4. 📊 总评

### 当前配置成熟度：⭐⭐⭐⭐（4/5）

当前配置已经覆盖了绝大多数核心功能。主要缺失集中在：

1. **SEO 微调**（`seo: true`, `keywords`, `favicon`）—— 零成本提升
2. **阅读体验**（`scroll_to_more`, `save_scroll`, `wordcount`, `post_copyright`）—— 极低成本提升
3. **可选第三方服务**（`mathjax`, `busuanzi`, `gitalk` 等）—— 按需开启，关闭无开销
4. **视觉点缀**（`pace` 加载进度条, `authoricon`）—— 小趣味
5. **CDN 供应商配置** —— 性能优化潜在空间

### 强烈推荐的前 10 项（无冲突、高性价比）

| 优先级 | 配置项 | 理由 |
|--------|--------|------|
| 1 | `favicon` | 基本站点标识，缺了不专业 |
| 2 | `keywords` | 简单 SEO 提升 |
| 3 | `seo: true` | 改善标题层级 SEO |
| 4 | `scroll_to_more: true` | 增强阅读连贯性 |
| 5 | `save_scroll: true` | 提升回访体验 |
| 6 | `post_wordcount` | 文章信息丰富度 |
| 7 | `post_copyright` + `creative_commons` | 版权保护 |
| 8 | `b2t: true` + `scrollpercent: true` | 长文导航便利 |
| 9 | `pace` 进度条 | 用户体验细节 |
| 10 | 补充 `vendors` CDN 项 | 性能提升基础 |
