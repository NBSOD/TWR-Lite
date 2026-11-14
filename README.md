# TWR-Lite

[中文](#中文) | [English](#english)

X-Flow / TikTok Mode Media Ranking（SleazyFork [#562631](https://sleazyfork.org/zh-CN/scripts/562631)）的
**省流修复版**油猴脚本：修复预览图跨域加载失败，并新增默认开启的**省流量模式**（视频与预览图都只在需要时才加载）。

> 非上游官方版本，基于上游 6.3.4（Apache-2.0）修改，改动清单见下。请勿向上游作者报告本版本的问题。

---

## 中文

### 这个版本修了什么

| 问题 | 原因 | 处理方式 |
|---|---|---|
| 预览图一片空白，控制台刷 CSP / 跨域报错 | 站点 CSP 拦截 `pbs.twimg.com` 的图片直连 | 用 `GM_xmlhttpRequest` 以特权拉取并转成 blob URL 回填，同时抹掉原始 `src`，避免浏览器发出必失败的请求 |
| 打开页面 / 划视频时下载速度飙升 | 脚本有 6 处主动预加载（见下） | 新增「省流量模式」：视频只在被点开后才真正下载，预览图滚到附近才下载 |

原先会持续消耗带宽的地方：

1. `PoolManager.startPrefetching` —— 一次性解析后面 5～8 条的详情；
2. `preloadNode()` —— 2 秒后把前后两条视频设成 `preload="auto"` 并赋 `src`，直接下载整段；
3. 播放器 `openModal` / `navigate` 的 1.5 秒定时器 —— 每切一条就把前后两条 `src` 填上；
4. 首页卡片悬停 / 长按预览 —— 每次新建 `<video preload="auto">`，下载整段；
5. 首页第 1 名自动播放并 `loop`；
6. 图片拯救器在元素插入瞬间就拉原图，首页几十张并发。

### 与上游 6.3.4 的差异（实测对比）

上游脚本 6.3.4（`@namespace xflow.loadingi.local`，作者 Chris_C）与本文这个版本都是约 30 万字符的单文件，
下表是逐段比对 + 关键字计数得到的结果（`@match` 28 条 / 14 个站点两边一致，未新增站点）。

| 项目 | 上游 6.3.4 | 本版 1.0.0 |
|---|---|---|
| 预览图怎么加载 | `<img loading="lazy" referrerpolicy="no-referrer">` 直连 `pbs.twimg.com`；**blob 绕过 0 处**，站点 CSP 下必然失败（Firefox 表现为整片空白 + 控制台刷报错） | 独立「CSP 拯救器」：`GM_xmlhttpRequest`（带站点 `Referrer`）取图 → `createObjectURL` → 回填 blob，并先把原 `src` 抹掉，浏览器不会发出必失败的请求 |
| 预览图何时加载 | 元素一插入就发请求（`loading="lazy"` 挡不住 CSP 拦截，脚本侧也没有拦截逻辑） | `IntersectionObserver`，滚到视口下方 800px 才拉；同时监听 `src`/`poster` 的**属性变化**（虚拟列表复用卡片是改 `src`，不触发子节点插入） |
| 正在看的视频 | `preload="auto"`，全脚本共 4 处 auto：当前条、相邻条、鼠标悬停、长按预览 | 只有当前条是 `auto`，其余先不拿 `src` |
| 相邻条目 | `preloadNode()` 2 秒后 `auto` + 赋 `src`，直接下整段 | 不解析、不下视频，只把缩略图贴上去 |
| 划走的那一条 | 只 `pause()`，缓冲继续下几 MB | `pause()` + `removeAttribute("src")` + `load()`，并解绑 `data-index`，防止慢网络下被异步回填 |
| 详情预取 | `startPrefetching` 一次解析后面 5～8 条；播放器 `openModal` / `navigate` 还有 1.5 秒定时器补前后两条 | 只解析下一条（1 次 HTML + 1 次 HEAD，不碰视频字节） |
| 首页第 1 名 | 自动播放并 `loop` | 默认不播（菜单可开） |
| 悬停 / 长按预览 | 每次悬停都新建 `<video preload="auto">` 并下载整段 | 默认不播（菜单可开） |
| 菜单开关 | 无（`GM_registerMenuCommand` 0 处，只能用扩展面板改） | 4 项开关，脚本菜单里直接切 |
| 头部权限 | `@grant` 4 个：`GM_getValue`、`GM_setValue`、`GM_openInTab`、`GM_xmlhttpRequest` | 5 个：多了 `GM_registerMenuCommand` / `GM_unregisterMenuCommand`，少了用不到的 `GM_openInTab` |
| 版本号 | 6.3.4（沿用上游版本序列） | 另起 `1.0.x`，避免与上游版本号混淆 |

一句话：**上游没有任何懒加载，也没有任何 CSP 绕过**；本版把「跨域修复」和「按需加载」两件事都补上了。

想自己复验，可以在这两个文件里数这几个关键字：

```bash
grep -c 'preload = "auto"'  TWR-Lite.user.js   # 主动预加载视频的点
grep -c 'createObjectURL'   TWR-Lite.user.js   # blob 绕过（上游为 0）
grep -c 'IntersectionObserver' TWR-Lite.user.js # 懒加载（上游为 0）
```

### 省流量模式的行为

- **视频**：只有 `currentIndex` 对应的节点会拿到 `src` 与 `preload="auto"`；划走的视频立即 `removeAttribute("src") + load()` 断开缓冲（仅 `pause()` 的话浏览器还会继续下几 MB），并解除 `data-index` 绑定，防止慢网络下已经划走的视频又被异步回填。
- **预览图**：`IntersectionObserver`（视口下方 800px 触发）命中后才用 GM 特权拉 blob。
- **播放器里对前后条只贴缩略图**，不再触达视频。
- 每次切条仍会解析「下一条」的详情（一次 HTML + 一次 HEAD，不下视频字节），用于展示标题 / 作者。

### 安装

1. 安装 [Tampermonkey](https://www.tampermonkey.net/) 或 Violentmonkey；
2. 打开下面的链接直接安装（脚本管理器会接管安装，不要手动复制源码）：

   ```
   https://raw.githubusercontent.com/NBSOD/TWR-Lite/main/TWR-Lite.user.js
   ```

3. Firefox 需要在扩展设置里允许访问对应站点（用户脚本 + 跨域请求权限）。
4. 如果装过上游 X-Flow 或旧的 `TwRanking-Fix`，**先卸掉**：`@namespace` 不同会被识别成两个脚本，同时跑会出现两套 DOM 观察器。

### 更新

脚本头部已启用 `@updateURL` / `@downloadURL`，指向本仓库 `main` 分支的 raw 地址：

- **自动**：Tampermonkey 默认每 24 小时检查一次（也受扩展的「更新」设置影响），有新版本会提示；
- **手动**：脚本菜单里点「检查更新」，或重新打开上面的安装链接覆盖安装。

发新版时的流程：改 `TWR-Lite.js` → 同步 `TWR-Lite.user.js` → 改 `@version` → 打 `v1.0.x` tag（这样 `@updateURL` 拿到的内容才和 tag 对得上）。

### 菜单开关

脚本菜单（Tampermonkey 图标 → 本脚本）里有 4 个开关，`✅` 表示已开启，点一下即切换：

| 菜单项 | 默认 | 作用 |
|---|---|---|
| 省流量：视频点开才加载 | ✅ | 总开关。关闭 = 恢复原来的预加载行为 |
| 悬停自动播放预览（费流量） | ⬜ | 鼠标悬停卡片时播放预览；每次悬停都会下载一段视频 |
| 首页第 1 名自动播放（费流量） | ⬜ | 进入首页自动播放排名第一的视频并循环 |
| 预览图懒加载 | ✅ | 预览图滚到视口附近才下载 |

- 前三项即时生效；「预览图懒加载」在脚本启动时读取，切换后**刷新页面**完全生效。
- 配置保存在 `GM_setValue("xflow_lazy_cfg_v1")`，与脚本原有设置互不影响。
- 想完全回到旧版本：把四项都切成 ⬜。

### 支持的站点

`pektino.com`、`x-ero-anime.com`、`truvaze.com`、`twihub.net`、`twikeep.com`、`twiidol.com`、
`twiigle.com`、`monsnode.com`、`twivideo.net`、`xiaohuangniao.me`、`twidouga.net`、`javtwi.com`、
`xhotvideo.com`、`uraaka-times.com`（均含 `*.` 子域）。

### 已知限制

- 切条时会多一次详情页请求，首次看到视频比旧版慢几百毫秒（这也是它能省流量的原因）。
- 依赖各站点的 DOM 结构，站点改版后可能需要适配（`AdapterManager` 下的各 `*Adapter`）。
- 仅在 Chrome / Firefox + Tampermonkey 上验证。

### 开发

单文件脚本，无构建、无依赖。改动后做语法检查即可：

```bash
node --check TWR-Lite.user.js
```

仓库已配置 GitHub Actions（`.github/workflows/check.yml`），push / PR 时自动跑语法与元数据检查。

### 发布清单（上传 GitHub 前）

- [x] 本地文件名已是 `TWR-Lite.user.js`（`.user.js` 后缀才能从 raw 链接一键安装）
- [x] 头部 `@namespace` / `@homepageURL` / `@updateURL` / `@downloadURL` 已填 `NBSOD/TWR-Lite` 并取消注释
- [x] 头部 `@author`：`NBSOD, DeepSeek (upstream: X-Flow / Chris_C)`
- [x] 上游授权协议：6.3.4 头部声明 `@license Apache-2.0`，与本仓库 `LICENSE` 一致
- [ ] 更新 `CHANGELOG.md` 的版本号与日期（发下个版本时）
- [ ] 上传后打首个 tag `v1.0.0`，Release 正文可直接用 `CHANGELOG.md` 里 `[1.0.0]` 那段

### 致谢与授权

- 上游脚本：X-Flow / TikTok Mode Media Ranking 6.3.4（作者 Chris_C），
  SleazyFork [#562631](https://sleazyfork.org/zh-CN/scripts/562631)。
  本仓库是它的**衍生版本**，站点适配、虚拟列表、播放器等主体逻辑均来自上游。
- 本仓库的修改（预览图跨域修复 + 省流量模式）在 AI（DeepSeek）协助下完成，
  同样以 `Apache-2.0` 发布，见 [LICENSE](LICENSE)；脚本头部的「修改声明」块满足 §4(b) 的告知要求。
- 上游 6.3.4 头部声明的是 `@license Apache-2.0`，与本仓库使用的协议一致，无需换协议；分发时保留上游署名即可。
- 本仓库名称刻意不使用上游的 “X-Flow”（Apache-2.0 §6 不授予商标使用权），与上游无隶属关系。

### 目录结构

```
TWR-Lite.js                # 开发用的源文件
TWR-Lite.user.js           # 同上的副本，安装/更新走这个（两者内容必须一致）
README.md
CHANGELOG.md
LICENSE                    # Apache-2.0
.gitignore
.github/workflows/check.yml
```

---

## English

A patched build of **X-Flow / TikTok Mode Media Ranking** (SleazyFork [#562631](https://sleazyfork.org/zh-CN/scripts/562631)).

- **Fixes blank thumbnails**: the site's CSP blocks direct `pbs.twimg.com` image loads. Thumbnails are now
  fetched through `GM_xmlhttpRequest` (privileged, with a proper `Referrer`) and swapped in as blob URLs,
  so the console stays clean.
- **Adds a bandwidth-saving mode (on by default)**: a video is downloaded only after you open it, and
  thumbnails load when they scroll near the viewport. Hover autoplay and the auto-playing #1 card are off.
- Four toggles in the userscript menu; turn all of them off to get the original eager-loading behaviour.

**Compared to upstream 6.3.4** (verified by section-by-section diff + keyword counts):

| Area | Upstream 6.3.4 | This fork 1.0.0 |
|---|---|---|
| Thumbnails | `<img loading="lazy" referrerpolicy="no-referrer">` straight to `pbs.twimg.com`; **no blob bypass at all**, so the site CSP blocks them | Privileged fetch (`GM_xmlhttpRequest` + `Referrer`) → `createObjectURL` → blob `src`; the original `src` is removed first |
| Thumbnail timing | Requested as soon as the element exists | `IntersectionObserver` (800px ahead), plus observation of `src`/`poster` attribute changes (the virtual list reuses nodes) |
| Playing video | `preload="auto"` — 4 places: current, neighbours, hover, long-press | Only the current one; others get no `src` |
| Neighbours | `preloadNode()` sets `auto` + `src` after 2s | Thumbnail only, no detail fetch, no video bytes |
| Scrolled-away video | `pause()` only — the download keeps going | `pause()` + `removeAttribute("src")` + `load()`, and `data-index` is unbound so it cannot be re-filled late |
| Detail prefetch | 5–8 items ahead, plus 1.5s timers in `openModal`/`navigate` | Next item only (1 HTML + 1 HEAD) |
| Auto-playing #1 card, hover preview | On | Off by default (menu toggles) |
| Userscript menu | None (`GM_registerMenuCommand` absent) | 4 toggles |
| `@grant` | 4 | 5 (`GM_registerMenuCommand`, `GM_unregisterMenuCommand` instead of the unused `GM_openInTab`) |

Upstream has **no lazy loading and no CSP workaround** whatsoever; both are added here. `@match` scope is unchanged (14 sites / 28 lines).

Requirements: Tampermonkey / Violentmonkey. No build step — a single file, checked with `node --check`.

Install / update (one click, the manager takes over):

```
https://raw.githubusercontent.com/NBSOD/TWR-Lite/main/TWR-Lite.user.js
```

`@updateURL` / `@downloadURL` point at the same raw file, so userscript managers pick up new versions
automatically (Tampermonkey checks about once a day). Remove any previously installed upstream X-Flow or
old `TwRanking-Fix` first — a different `@namespace` makes them two separate scripts that would both run.

License: `Apache-2.0` — upstream 6.3.4 declares the same, so `LICENSE` matches. The name deliberately
avoids upstream's "X-Flow" (Apache-2.0 §6 grants no trademark rights); this fork is not affiliated with upstream.