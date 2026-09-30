# 第三方来源

插件的样式与标记不是凭空写的。为了和 dsh 自带界面长得一样，下列部分取自 DeepSeek Harness 的包；那些包是 MIT 许可（Copyright (c) 2026 DeepSeek，许可原文随 <https://github.com/deepseek-ai/deepseek-harness> 分发）。

| 本插件的位置 | 来源 | 方式 |
|---|---|---|
| 侧栏身份行 `.dshprf_trigger` / `.dshprf_avatar` / `.dshprf_label` | `@deepseek-ai/dsh-client-ui-settings-account` 的账号启动器 | CSS 声明照搬，类名换前缀 |
| 分页容器与卡片 `.dshprf_section` / `.dshprf_card` | 同包的账号分页 | CSS 声明照搬 |
| 字段行 `.dshprf_field*` | `@deepseek-ai/dsh-client-ui-conversation` 的通用设置行 | CSS 声明照搬 |
| 输入框与按钮 `.dshprf_input` / `.dshprf_button` | `@deepseek-ai/dsh-client-ui-settings-account` 的链接按钮 | 度量与配色参照 |
| 余额的显示口径（两位小数、不足一分显示 `<¥0.01`、千分位） | 同包的 `formatBalance` | 规则照搬，十进制运算自己实现 |

其余部分——状态存储、注册与注销的生命周期、分页内容、余额取数与求和、品牌行的处理——是本插件自己写的。所有抄来的类名都换了 `dshprf_` 前缀，颜色与间距只用 dsh 的主题 token（`--dsw-*`），不复制对方的类名或选择器。
