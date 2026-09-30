# @local/dsh-personalization

侧栏底部那行身份的替代品，加上设置面板里属于它自己的一页。dsh 自身不提供自定义头像/昵称，这里以客户端插件的形式挂在插槽上：接管侧栏的账号行，编辑界面作为独立分页注册进 Settings 的导航。

这一页是**我们对客户端做界面调整的落点**：目前只有侧栏身份行（头像、昵称、余额），以后的界面改动都加在这里，而不是散到系统自带的分页里去。

## 用法

安装后：

1. 侧栏底部的头像行显示你的头像、昵称，右侧是该账户的余额（点头像打开设置面板）。
2. **设置 → 个性化**：面板左侧导航里的一页，排在「Agent 预设」之后。页面按功能分两张卡片，每张以它自己的开关开头：
   - **身份行**：自定义身份（开关）→ 头像（选择图片 / 恢复默认）→ 昵称。
   - **界面元素**：隐藏顶部品牌标识（开关）。

头像在本地缩放到 128px 方图后存进 localStorage；昵称即时生效。侧栏折叠成图标栏时不显示昵称与余额（那一行只剩头像）。

## 挂在哪

| 位置 | 插槽 | 说明 |
|---|---|---|
| 侧栏底部 | `settings.launcher`（single） | `priority: -1` 压过系统自带的账号行；关掉开关即注销注册，原样恢复 |
| 侧栏顶部品牌 | `sidebar.brand.mark`、`sidebar.brand.name`（single） | 隐藏开关打开时以 `priority: -1` 顶上、渲染空；关掉即让回 |
| 品牌行的高度 | 注入的样式，只在开关打开时挂上 | 见下 |
| 设置面板导航 | `settings.section`（list，id `personalization`, order `25`） | 一页一条导航项；`label` 是随语言变化的 thunk，壳在 ledger 变动时重投影 |

壳把品牌行的行高写成死值（60px，且没有走 token），所以只清空两个座位会留下一块空白。插件为此额外注入一条规则：

```css
[class*="_logoRow"]:not(:has(button)){height:0;min-height:0;margin:0;padding:0;overflow:hidden}
```

这是整个插件唯一一处伸到自己组件之外的地方。它按壳的 CSS-modules 局部名片段匹配，并且拒绝含按钮的行——Windows 上折叠按钮就在那一行里，这条守卫让那边保持原样（只空出内容，不收行高）。壳若改了局部名，规则自动失效，结果是退回"空出内容但留白"，不会坏。样式元素随开关挂上或摘掉，开关关掉后文档里不留痕迹。

`settings.section` 的导航与图标由壳拥有：图标按 id 固定映射（`account`/`models`/`agent-presets`/`plugins`/`archived-sessions`），未知 id 落到默认齿轮——所以本项与「通用设置」同图标，只能靠文字区分。注册项本身也拿不到图标字段。

壳体不提供「直接打开某一分页」的接口（`openSection` 只发给 onboarding 登记项），因此侧栏头像行只能打开设置面板，落在哪个分页由壳自己的选中状态决定；分页选择是壳的常驻状态，去过一次「个性化」之后一般会停在那里。

## 余额

侧栏那一行右侧的数字来自账户 Remote，与官方账号页同一条路径：

```
ctx.remote.account.getBalance({ version, locale, timezoneOffsetSeconds })
```

`version` 是客户端构建版本（`x-client-version` 头）。壳把它在建构建时内联进各自的 bundle，运行时没有可读的全局值，所以 `client.js` 里的 `CLIENT_VERSION` 常量要跟着 dsh 版本手工改。代价可控：版本被接口拒绝时余额只是不显示，不会显示错数。

- 口径：充值余额 + 赠金余额，同一币种求和，沿用官方账号页的两位小数与不足一分显示规则（`<¥0.01`）。
- 刷新：账号状态流的每一帧（登录、登出、状态变化都会推动一次）、插件激活时、你在侧栏点开设置面板那一下，另有页面可见时每 3 分钟一次的兜底。首帧到达之前凭证尚不可用，所以读取跟在状态流之后，而不是抢在启动时。
- 读不到时不显示数字：侧栏那一格直接不渲染。插件内部仍区分 `loading` / `signed-out` / `failed` / `unavailable`（后者指激活 10 秒内始终没拿到账户 Remote），但只有 `ready` 会落到界面上——余额只出现在头像行；官方设置里本来就有余额，本插件的页面不重复。

## 数据

`localStorage['dsh.personalization.v1']`（改名前的 `dsh.profile-identity.v1` 会被读一次，你的头像和昵称不会丢；旧键留着不删）：

```json
{ "enabled": true, "nickname": "Mgeeeeee", "avatar": "data:image/png;base64,...", "hideBrand": false }
```

`enabled` 管自定义身份，`hideBrand` 管顶部品牌标识，两者互不影响。

纯客户端状态，不进 session log，也不上云；换浏览器或清缓存即丢失。余额不落盘。

## 取舍

系统自带的账号行除了头像和手机号，还挂着一个下拉菜单（设置 / 意见反馈 / 退出登录）。`settings.launcher` 是单占位插槽，接管它就等于接管整个菜单，而退出登录的实现在系统插件内部（`ctx.remote.account.signOut(client)`，与余额同一个客户端元数据），本插件没有复制它。所以**自定义身份开着的时候，「退出登录」没有入口**，需要时到本页把「自定义身份」关掉，原账号行与菜单立刻回来。官方设置本身不受影响，侧栏底部的「设置」行照旧。

## 卸载

```
plugin_manager  remove_bundle  @local/dsh-personalization
```

卸载后侧栏恢复系统账号行，设置导航里的一页随之消失。localStorage 里的那条记录不会被清除；想彻底清干净，删掉那个键即可。

`remove_bundle` 会清掉 profile 的依赖与 bundle 列表，但**不删 `~/.dsh/profiles/<profile>/node_modules/@local/<name>` 这个软链**，它会留在原地变成悬空链。卸载后顺手看一眼那个目录，删掉指向已删包的链接。

## 开发

宿主半边 `index.js` 是空的，全部逻辑在 `client.js`（模块名与包名一致，`window.__ModuleLoader__.load` 注册）。改 `client.js` 后刷新页面；如果新代码没生效，重启 dsh。

样式抄的是系统自带组件：头像行来自账号启动器，分页容器来自账号分页（`.dshprf_section`），卡片与字段行来自账号设置卡。类名一律换成 `dshprf_` 前缀，声明只用 `--dsw-*` token。
