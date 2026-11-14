# TWR-Lite

X-Flow / TikTok Mode Media Ranking（SleazyFork [#562631](https://sleazyfork.org/zh-CN/scripts/562631)）的省流修复版：
修好预览图跨域加载失败，并让视频与预览图都只在需要时才加载。

> 非官方版本，基于上游 6.3.4（Apache-2.0）修改。请勿向上游作者报告本版本的问题。

## 安装

先卸掉已装的 X-Flow 或旧 `TwRanking-Fix`（`@namespace` 不同会被当成两个脚本同时跑），然后点这个链接：

```
https://raw.githubusercontent.com/NBSOD/TWR-Lite/main/TWR-Lite.user.js
```

Firefox 需在扩展设置里允许脚本访问这些站点。

## 它做了什么

| | 上游 6.3.4 | 本版 |
|---|---|---|
| 预览图 | `<img>` 直连 `pbs.twimg.com`，被站点 CSP 拦掉（Firefox 表现为整片空白） | `GM_xmlhttpRequest` 取图 → `blob:` 回填，不再发必失败的请求 |
| 预览图何时加载 | 元素一出现就发请求 | 滚到视口下方 800px 才拉 |
| 视频 | 当前条 + 前后条 + 悬停 + 长按，共 4 处 `preload="auto"`，都会下整段 | 只有正在看的那条下载；其余只贴缩略图 |
| 划走的那条 | 只 `pause()`，缓冲继续跑 | `pause()` + 卸载 `src` + `load()`，当场断开 |
| 预取 | 一次解析后面 5～8 条 | 只解析下一条（1 次 HTML + 1 次 HEAD，不碰视频字节） |
| 菜单 | 无 | 4 个开关 |

上游没有任何懒加载、也没有任何 CSP 绕过，这两件事都是本版补的。

## 菜单开关

Tampermonkey 图标 → 本脚本，`✅` = 已开启：

- **省流量：视频点开才加载**（默认 ✅）— 总开关，关掉即恢复上游的预加载行为
- **悬停自动播放预览**（默认 ⬜）— 每次悬停都会下一段视频
- **首页第 1 名自动播放**（默认 ⬜）
- **预览图懒加载**（默认 ✅）— 切换后需刷新页面

## 更新

不用管。脚本头部 `@updateURL` / `@downloadURL` 指向本仓库 raw 文件，脚本管理器大约每天检查一次 `@version`，有新版会提示；也可以在脚本菜单里点「检查更新」。

**Release / tag 都不是安装或更新的必要条件**，raw 链接 + 递增的 `@version` 就够了。

发新版：改 `TWR-Lite.js` → 同步拷成 `TWR-Lite.user.js` → 改 `@version` → 提交。CI 会跑 `node --check` 和头部字段检查。

## 已知限制

- 切条时多一次详情页请求，首次看到视频比旧版慢几百毫秒 —— 省流量就省在这。
- 依赖各站点 DOM 结构，站点改版后可能失效。
- 只在 Chrome / Firefox + Tampermonkey 上验过。

## 授权与致谢

- 上游：X-Flow / TikTok Mode Media Ranking 6.3.4，作者 **Chris_C**，[SleazyFork #562631](https://sleazyfork.org/zh-CN/scripts/562631)。站点适配、虚拟列表、播放器等主体逻辑均来自上游。
- 本仓库只改了预览图加载方式和按需加载策略（在与 AI 协作下完成），同样以 `Apache-2.0` 发布，见 [LICENSE](LICENSE)。
- 仓库名刻意不使用上游的 “X-Flow”（Apache-2.0 §6 不授予商标权），与上游无隶属关系。

---

## English

A bandwidth-friendly patch of **X-Flow / TikTok Mode Media Ranking** ([SleazyFork #562631](https://sleazyfork.org/zh-CN/scripts/562631)) by Chris_C, Apache-2.0.

- **Blank thumbnails fixed**: the site CSP blocks direct `pbs.twimg.com` requests, so images are fetched via
  `GM_xmlhttpRequest` and swapped in as `blob:` URLs.
- **Lazy loading**: only the video you open is downloaded; scrolled-away videos are unloaded immediately, and
  thumbnails load 800px before entering the viewport. Eager prefetching, hover preview and the auto-playing
  #1 card are off by default (four toggles in the userscript menu).

Install (remove any installed X-Flow / `TwRanking-Fix` first — different `@namespace` means two scripts would run):

```
https://raw.githubusercontent.com/NBSOD/TWR-Lite/main/TWR-Lite.user.js
```

Updates are picked up from the same raw file through `@updateURL`; no releases or tags required. Upstream has no
lazy loading and no CSP workaround at all — both are added here. Not affiliated with upstream.