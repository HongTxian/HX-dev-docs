# HX 扩展开发文档

写一个 JS 文件 → 导入 HX → 它就能变成影视源、弹幕源、网盘工具或首页推荐。
不需要编译，不需要打包，不需要上架。

- 适用版本：HX **0.30.x**
- 运行时：QuickJS（Android）/ JavaScriptCore（iOS），**单文件拼接执行，没有 ESM / CommonJS**
- 一份扩展 = 一个 `.js` / `.txt` 文件，顶部几行 `//@` 注释声明元数据

---

## 一、扩展能做什么

| 类型 | 你能写出来的东西 | 入口 |
|---|---|---|
| 视频源（101） | 任意站点的搜索 / 分类 / 详情 / 播放地址 | 全局函数 |
| 视频源（100） | 同上，但用「class + 实例」写法 | 固定实例 + `instance` 字段 |
| 弹幕（400） | 按剧名 + 集号搜弹幕 | 全局函数 |
| 网盘工具（300） | 分享链接 → 文件 → 直链 | 固定实例 `uzPanToolsInstance` |
| 首页推荐（200） | 首页 tab、卡片位、筛选 | 固定实例 `uzHomeJs` |

**不能做什么**（写之前先看清楚，别浪费时间）：

- 没有 `fetch` / `XMLHttpRequest`，网络只能走宿主的 `req()`
- 没有 `require` / `import` / `module`，也没有 `window` / `document`
- 不能访问文件、剪贴板、定位、通知等原生能力
- 不能自定义 UI，只能用宿主给的字段（标题、海报、备注…）
- 不能长时间后台运行：每个方法都有超时（默认 45～60 秒），超时即失败

---

## 二、5 分钟上手：写一个弹幕扩展

```js
//@name:我的弹幕
//@version:1
//@type:400

async function searchDanMu(args) {
  // args = { name, episode, videoUrl, line, videoPlatformName }
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

存成 `my_danmu.js`，在 HX 里：

```
设置 → 手动导入的扩展 → 从文件导入（或从剪贴板导入）
  → 列表里点「⋮」→ 设为弹幕源
  → 找个片子播一集，播放页右下角出现「弹幕 · N」
```

---

## 三、怎么装进去

| 方式 | 适合 | 操作 |
|---|---|---|
| 从剪贴板 | 临时试一段代码 | 复制全文 → 设置 → 从剪贴板导入 |
| 从文件 | 自己写的 `.js` | 设置 → 从文件导入（弹系统的文件选择器，不按扩展名过滤） |
| 订阅 | 发一批给别人用 | 把 JSON 传到任意 HTTPS 直链 → 数据管理 → 订阅 → ＋ |

装进来后在扩展卡片的「⋮」里指定用途：设为视频源 / 弹幕源 / 网盘工具 / 推荐源。
**同名多条时**看副标题：`明文 / 已加密 · 体积`，别选错。

---

## 四、文档索引

| 文档 | 内容 |
|---|---|
| [01-扩展类型与入口](01-扩展类型与入口.md) | 五种类型各自要实现哪些函数、什么时候被调用 |
| [02-元数据与订阅](02-元数据与订阅.md) | `//@` 字段、订阅 JSON 格式、安装/更新行为 |
| [03-宿主接口](03-宿主接口.md) | 9 个通道、`req` 完整选项、内置全局库 |
| [04-视频源扩展](04-视频源扩展.md) | type 100/101：7 个方法、剧集格式、嗅探 |
| [05-弹幕扩展](05-弹幕扩展.md) | type 400：`searchDanMu`、线路、HX 的搜索规则 |
| [06-网盘工具扩展](06-网盘工具扩展.md) | type 300：分享链接解析、直链、Cookie |
| [07-推荐扩展](07-推荐扩展.md) | type 200：首页 tab、卡片位、筛选 |
| [08-扩展加密](08-扩展加密.md) | 把自己的扩展加密，只有 HX 能解开 |
| [09-调试与测试](09-调试与测试.md) | 内置调试页、日志标签、Node 侧测试脚本 |
| [10-注意事项](10-注意事项.md) | 踩坑清单：模块、命名冲突、超时、编码 |

---

## 五、类型速查

| type | 类型 | 入口 | 必需方法 |
|---|---|---|---|
| 100 / 101 | 视频源 | 全局函数（100 用实例） | `getClassList` `getVideoList` `getVideoDetail` `getVideoPlayUrl` `searchVideo` |
| 200 | 首页推荐 | 固定实例 `uzHomeJs` | `getHome` `getTab` |
| 300 | 网盘工具 | 固定实例 `uzPanToolsInstance` | `getShareVideos` `getPlayInfo` |
| 400 | 弹幕 | 全局函数 | `searchDanMu` |
| 0 | 采集站 | 不走 JS，`api` 是苹果CMS 接口 | — |
| 10 | 直播源 | 不走 JS，`api` 是 M3U 地址 | — |

> `uzHomeJs` / `uzPanToolsInstance` / `uzTag` 是**接口约定的标识符**，
> 必须原样照写 —— 宿主按这些名字找人、取值，改了就是 undefined。

---

## 六、加密发布

不想让别人看到源码，一条命令即可（解密在本机运行时完成，落盘始终是密文）：

```bash
node tool/hx_encrypt_ext.js my_danmu.js                # 生成 my_danmu.enc.js
node tool/hx_enc_test.js my_danmu.js my_danmu.enc.js   # 验证：逐字节一致 + 篡改被拦
```

详见 [08-扩展加密](08-扩展加密.md)。
