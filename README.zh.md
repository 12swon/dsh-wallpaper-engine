# dsh-wallpaper-engine

[English](README.md) | [中文](README.zh.md)

一个 DSH bundle，把你电脑上的 **Wallpaper Engine** 壁纸变成 **DSH 网页界面（`dsh web`）的背景**。

它会自动发现你本机的 Wallpaper Engine 安装，列出你的壁纸，并把其中*可移植*的类型（Video `.mp4` 和 Web/HTML）渲染到 DSH 对话界面的后方，配以 **iOS 风格液态玻璃**效果。你可以在设置里挑选壁纸、用四个滑动条微调，也能随时暂停或关闭。

## 为什么只支持 Video 和 Web 壁纸？

Wallpaper Engine 的壁纸分四种类型：

| 类型 | 由谁渲染 | 能否搬到 DSH |
|---|---|---|
| **Scene（场景）** | Wallpaper Engine 自带的 3D 引擎 | ❌ 不能 — 原生 3D（`.obj`/着色器），只有 WE 能渲染 |
| **Video（视频）** | 就是一个 `.mp4` 文件 | ✅ 能 — 在 `<video>` 标签里播放 |
| **Web（网页）** | WE 内置的 Chromium 壳（`webwallpaper64.exe`）承载 HTML | ✅ 能 — 在 `<iframe>` 里加载 |
| **Application（应用）** | 注入的外部窗口 | ❌ 不能 |

这是 mineradio 以及所有第三方 Wallpaper Engine 集成方案都无法回避的同一限制：只有 *Video* 和 *Web* 两种壁纸可移植。Scene 壁纸仍会列在选择器里（标为 `[不可播放]`），让你知道自己有什么，但没办法拿来做动态背景。

## 工作原理

- **Host 端**（`lib/index.js`）：一个 Cordis 插件，负责
  1. 通过读取 Steam 的 `libraryfolders.vdf` 定位 Wallpaper Engine 安装位置（所以 Steam 装在非默认盘也能用）；
  2. 从 `projects/defaultprojects`、`projects/myprojects` 以及 `steamapps/workshop/content/431960/*` 枚举壁纸；
  3. 在 DSH webserver 上注册同源 HTTP 路由，让浏览器端直接获取数据和流式加载媒体：
     - `GET /wallpaper-engine/inventory` → 壁纸 JSON 列表
     - `GET /wallpaper-engine/media/<token>` → 视频 / HTML（支持 Range）
     - `GET /wallpaper-engine/preview/<token>` → 预览图
- **Client 端**（`lib/client.js`）：一个浏览器模块，拉取壁纸列表，把选中壁纸渲染到应用三列**后方**的固定图层，并在「设置 → General」里加一个「Wallpaper Engine」行（含选择器）。

## 安装

**发布后（推荐）**——包上线到 npm 后：

```sh
dsh plugin --profile web add dsh-wallpaper-engine
```

**本地开发**——从本仓库 checkout，用 `link:` 指向本地目录（这个绝对路径就是**包含 `package.json` 的那个文件夹**）：

```sh
dsh plugin --profile web add link:D:\path\to\dsh-wallpaper-engine
```

> **这里说的"绝对路径"到底指什么？**
> 它指的是你**插件 checkout 所在的完整文件夹路径**——也就是**文件夹里带着 `package.json` 的那个目录**，*不是* `package.json` 文件本身的路径，也不是它内部某个文件的路径。你可以直接把这个路径理解为「打开该文件夹时，资源管理器地址栏显示的那一串路径」。
>
> 换句话说：**绝对路径 = 这个插件的代码文件夹在哪**，把那一整串路径填进去即可。
>
> 举例：如果仓库在 `D:\dev\dsh-wallpaper-engine`，就运行：
> `dsh plugin --profile web add link:D:\dev\dsh-wallpaper-engine`。
>
> Linux/macOS 也一样：`dsh plugin --profile web add link:/home/你/dsh-wallpaper-engine`，如果你已经 `cd` 到它的上级目录，也可以用相对路径 `link:./dsh-wallpaper-engine`。
>
> `dsh` 会把参数原样转发给 `pnpm`，所以路径必须指向**仓库目录本身**（也就是 `package.json` 里声明了 `dsh.bundle` 的那个目录）。

> 为什么用 `link:` 而不是 `file:`？`link:` 会创建指向**源目录**的 junction，改完 `src/client.js` 并 `npm run build` 后无需重新安装即可生效；`file:` 则打包成一份快照，每次改动都要重新 add。首次安装两者都可以。

然后重启 `dsh web`。host 端会成为 bundle 层，client 端会自动加载（`dsh.client.immediately: true`）。

如果 Steam 装在非标准位置，host 会通过 `libraryfolders.vdf` 自动探测，无需额外配置。

## 使用

1. 打开 `dsh web`，进入 DSH 界面。
2. 打开 **设置 → General**，找到 **Wallpaper Engine** 行。
3. 在下拉框里选一个 Video 或 Web 壁纸，它会出现在界面后方。
4. 用 **暂停/播放** 暂停视频壁纸，用 **关闭** 清除壁纸。
   选择会保存在浏览器的 `localStorage`（键 `dsh-wallpaper-engine:selection`）中。

### 四个滑动条

壁纸激活后，四个滑动条可以微调它与界面的融合效果：

| 滑动条 | 作用 | 范围 | 默认 |
|---|---|---|---|
| **壁纸模糊** | 模糊壁纸本身 | 0–60 px | 0 |
| **暗化** | 加深壁纸与文字之间的遮罩 | 0–90 % | 25 % |
| **边框** | 提高边框 / 分割线的对比度 | 0–90 % | 35 % |
| **玻璃** | 玻璃面板（输入栏、气泡）的模糊半径 | 0–40 px | 24 |

> **浅色 / 深色模式的适配提醒** — 每张壁纸的色系和明暗差异很大，**没有哪一种模式能适配所有壁纸**。请在 DSH 的「浅色 / 深色」主题之间来回切换，找到适合当前壁纸的那一种。如果在偏亮或花纹复杂的壁纸上 **文字或分割线看不清**，就把 **暗化**、**边框** 两个滑动条调高（必要时再稍微加一点 **壁纸模糊**），直到看着舒服为止。四个滑动条都是即时生效的，**无需刷新页面**。

## 配置

本插件不会向模型暴露任何工具或提示文本，对 agent 零 token 开销。所有状态都是进程内 / 浏览器内的，不会写入任何持久化 DSH 设置。

## 已知限制

- Scene（原生 3D）和 Application 壁纸无法内嵌，选择器里会显示为 `[不可播放]`；它们的动态渲染仍是 Wallpaper Engine 在桌面上的工作。
- 浏览器需能自动播放静音 `<video>`（DSH 跑在 loopback，现代浏览器允许静音自动播放）。
- 媒体从你本机的 Wallpaper Engine 安装路径提供；host 只提供它已枚举过的文件，不会暴露任意文件系统。
- 选择器文案为中英混合（本 bundle 尚未接入 DSH 的 locale 命名空间）。

## 开发 / 重建

host 端（`lib/index.js`）是纯 ESM，无需构建。client 端（`lib/client.js`）是**编译产物**，由规范源文件 `src/client.js` 经 `scripts/build-client.mjs` 生成，输出 DSH 模块加载器要求的 `window.__ModuleLoader__.load({ id, factory })` 外壳（与盒内 client 包 `tsdown` 产出的形态一致）。

```sh
npm run build      # 从 src/client.js 重新生成 lib/client.js
npm run verify     # 物化生成的 bundle 并断言其导出
```

编辑 `src/client.js` 后运行 `npm run build`，不要手改 `lib/client.js`。`npm install`/`pnpm install` 会自动触发 `prepare` → `build`，因此全新 checkout 总是带最新的 `lib/client.js`。

host↔browser 的契约是同源 HTTP，两端可独立开发：改 host 后重启 `dsh web` 生效，改 client 则先 `npm run build` 再重启 `dsh web`。
