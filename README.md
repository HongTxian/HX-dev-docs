# HX 扩展开发文档

面向第三方开发者：写一个 JS 文件，放进 HX 就能跑起来。

- 适用版本：HX `0.26.x`
- 运行时：QuickJS（和 uz 同一套 jsbox 规则：拼接 + 剥壳，没有 ESM / CommonJS）
- 一份扩展 = **一个 `.js` 或 `.txt` 文件**，顶部写几行 `//@` 注释声明元数据

---

## 5 分钟上手：写一个弹幕扩展

```js
//@name:我的弹幕
//@version:1
//@type:400

async function searchDanMu(args) {
  // args: { name, episode, videoUrl, line, videoPlatformName }
  const kw = `${args.name} ${args.episode}`.trim();
  const r = await req(`https://example.com/api/search?q=${encodeURIComponent(kw)}`);
  if (r.code !== 200) return { data: [], error: '接口返回 ' + r.code };

  return {
    data: JSON.parse(r.data).list.map((e) => ({
      content: e.text,
      time: e.time,        // 秒，支持小数
      color: e.color || '', // '#ff5252' 或 16777215，可空
    })),
    error: '',
  };
}
```

把这段存成 `my_danmu.js`，在 HX 里：

```
设置 → 手动导入的扩展 → 从文件导入（或从剪贴板导入）
  → 列表里点「⋮」→ 设为弹幕源
  → 随便播一集，播放页右下角会出现「弹幕 · N」
```

---

## 文档索引

| 文档 | 内容 |
|---|---|
| [01-扩展类型与入口](01-扩展类型与入口.md) | 五种类型分别要实现哪些函数、什么时候被调用 |
| [02-元数据与订阅](02-元数据与订阅.md) | `//@` 字段、`env` 声明、订阅 JSON 格式 |
| [03-宿主接口](03-宿主接口.md) | `sendMessage` 的 9 个通道、`req` 的完整选项 |
| [04-弹幕扩展](04-弹幕扩展.md) | type 400：`searchDanMu`、线路、HX 的搜索规则 |
| [05-网盘工具扩展](05-网盘工具扩展.md) | type 300：分享链接解析、播放地址、cookie |
| [06-推荐扩展](06-推荐扩展.md) | type 200：首页 tab、推荐位、筛选 |
| [07-注意事项](07-注意事项.md) | 踩坑清单：ESM、ignore 块、命名冲突、错误处理… |
| [08-调试与测试](08-调试与测试.md) | 日志标签、导入方式、常见报错对照表 |

---

## 类型速查

| type | 类型 | 入口 | 必需方法 |
|---|---|---|---|
| 100 / 101 | 视频源 | 全局函数（100 可带 `instance`） | `getClassList` `getVideoList` `getVideoDetail` `getVideoPlayUrl` `searchVideo` |
| 200 | 首页推荐 | 固定实例 `uzHomeJs` | `getHome` `getTab`（`getFilterList` `getSearchSuggest` 可选） |
| 300 | 网盘工具 | 固定实例 `uzPanToolsInstance` | `getShareVideos` `getPlayInfo`（`getPanTypes` `getSupportMountPan` 可选） |
| 400 | 弹幕 | 全局函数 | `searchDanMu`（`getLines` 可选） |
| 0 | 采集站 | 不走 JS，`api` 直接是苹果CMS 接口 | — |
| 10 | 直播源 | 不走 JS，`api` 是 M3U 地址 | — |

> 加密扩展（文件里带 `@codeID:` 且正文是一整段 base64）**当前版本解不开**，请提交明文版本，见 [07-注意事项](07-注意事项.md#加密扩展codeid)。
