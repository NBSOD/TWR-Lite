# Changelog

本项目的所有重要改动都会记录在这里。

格式参考 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)，
版本号遵循 [语义化版本](https://semver.org/lang/zh-CN/)。

## [1.0.0] - 2026-11-14

首个公开版本。基线是上游 X-Flow / TikTok Mode Media Ranking 6.3.4（作者 Chris_C，
Apache-2.0），本版本在它之上做了两件事：修预览图跨域，以及把"一直在后台下载视频"
改成"点开才加载"。

> 开发过程中曾用过 `1.0.2` / `1.0.4` 等版本号，那些只是本地迭代，从未公开发布，
> 因此不单独列入本文件；它们的内容已并入本次 1.0.0。

### Added

- **省流量模式**（默认开启），解决"用上脚本后下载速度一路飙升"的问题：
  - 视频懒加载：只有正在观看的那一条会真正下载；划走的视频立即卸载 `src` 并 `load()`，
    当场断开后台缓冲，同时解除 `data-index` 绑定，避免异步回填已经离开的那一条。
  - 预览图懒加载：改用 `IntersectionObserver`（视口下方 800px）触发，滚到附近才下载。
  - 播放器内对前后条只贴缩略图（`loadThumbOnly` / `applyThumb`），不再触达视频。
  - 菜单新增 4 个开关，配置持久化到 `GM_setValue("xflow_lazy_cfg_v1")`：
    「视频点开才加载」「悬停自动播放预览」「首页第 1 名自动播放」「预览图懒加载」。
- **预览图跨域（CSP）修复**：`pbs.twimg.com` 的图在 Firefox 下会被页面的 CSP 拦掉，
  改为用 `GM_xmlhttpRequest` 取图并回填 `blob:` URL；拯救器同时监听 `src` / `poster`
  的属性变化（虚拟列表复用卡片时是"改 src"而不是"插入元素"）。
- 仓库工程化：`README.md`、`CHANGELOG.md`、`LICENSE`(Apache-2.0)、`.gitignore`、
  GitHub Actions 语法与元数据检查。
- README 新增「与上游 6.3.4 的差异（实测对比）」：逐项列出预览图加载方式、懒加载、预加载点、
  菜单开关、`@grant` 等差异，并给出可复验的 `grep` 命令。
- 脚本头部增加「修改声明」块（Apache-2.0 §4(b)：告知本文件相对上游被修改过）。

### Changed

- 项目更名为 **TWR-Lite**，仓库 `TWR-Lite`；`@namespace` 从上游的 `xflow.loadingi.local` 改为
  `https://github.com/NBSOD/TWR-Lite`，避免与上游脚本被识别为同一个（同时装会跑两套 DOM 观察器）。
- 启用 `@homepageURL` / `@supportURL` / `@updateURL` / `@downloadURL`，安装与更新统一走
  `raw.githubusercontent.com`，脚本管理器可自动检查更新。
- `@author` 记为 `NBSOD, DeepSeek (upstream: X-Flow / Chris_C)`。
- 预取数量在省流量模式下大幅收敛：播放器内 `startPrefetching` 5 → 1，首页 8 → 1；
  `schedulePreload()` / `preloadNode()` 在总开关开启时直接短路。
- 卡片悬停预览、长按预览、首页第 1 名自动播放：改为受开关控制，默认关闭。
- 同一元素换了新图时覆盖排队中的 URL，避免贴上过期的图。
- 省流量模式下不再额外铺一层 CSS 背景图（同一张图会被下载两遍），改用 `object-fit: cover` 铺满。
- 没有缩略图时不再写入空 `src`（空 `src` 会让浏览器去请求当前页面地址）。

### Fixed

- 异步 `loadNode` 在快速连续切条时可能把视频写回已经复用的节点。
- `resolvedItem.url` 为空时把 `undefined` 赋给 `video.src` 的边缘情况。

### Removed

- 头部 9 个从未使用的 `@grant`（`GM_addValueChangeListener`、`GM_deleteValue`、`GM_download`、
  `GM_getResourceText`、`GM_info`、`GM_listValues`、`GM_removeValueChangeListener`、`GM_setValues`、
  `unsafeWindow`）；实际只用到 5 个。
- 与 `@name` 内容完全相同的 `@name:zh-CN` / `@name:zh-TW` / `@name:en` 三行。
- 头部冗长的版本说明注释压缩为一行（细节移到 README）。

[1.0.0]: https://github.com/NBSOD/TWR-Lite/releases/tag/v1.0.0