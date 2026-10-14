# 极简主题（short: minimal）

monitor 状态页的极简单文件主题。主界面为**卡片网格**，每张卡一眼看全：
**发行版图标 / 在线状态 / CPU·内存·硬盘 / 3 网延迟 / 累计流量 / 到期·价格**；
点任意卡片进入**节点详情页**（`/node/{id}`），含**可选时间范围的性能图表**。

## 文件
- `dist/index.html`  —— 主题本体（零构建、单文件、内联 CSS/JS + 内联品牌 SVG，约 55KB，无外链）
- `theme.json`       —— 六字段 + config（公告）
- `preview.png`      —— 面板预览图（浅色）
- `theme.tar.gz`     —— 安装包（`gzip -t` 通过；顶层 `dist/ theme.json preview.png`，无外层套目录）

## 主界面：卡片网格（紧凑）
`grid` 自适应列（≥292px/卡，桌面 4 列、窄屏 1 列）。顶栏右侧有**深浅色切换按钮**（存 localStorage，
未设置时跟随系统）。每张卡：
- 卡头：**发行版图标** + 节点名（国家 chip + 备注）+ 在线状态（离线带时长）
- 三条资源进度条：CPU / 内存 / 硬盘（百分比 + 高占用变色：>70 橙、>90 红）
- 卡底：**延迟热力色带**（3 条探测线路各一条色带，按延迟分 5 档着色：<50 绿 / <90 黄绿 / <130 黄 / <180 橙 / ≥180 红，
  无数据浅灰，悬停显示时刻+延迟+丢包）。每行线路名用**彩色圆点**（蓝/紫/粉，与详情页 3 网配色一致）标识，
  色带/进度条/数值**逐列严格对齐**（统一 42px 标签 / 1fr 内容 / 46px 数值 / 8px 间距网格）、**流量**（累计 ↓RX ↑TX）、**到期**（天数 + 价格·周期）

### 发行版图标（真实品牌 logo，来自 Simple Icons）
内联 SVG，零外链。覆盖：Debian / Ubuntu / Red Hat·RHEL / Rocky Linux / AlmaLinux / CentOS /
Fedora / Arch Linux / Alpine / openSUSE / Gentoo / Amazon Linux / Oracle Linux / Windows /
FreeBSD / macOS / Linux(通用)，认不出走 Linux。
用官方品牌色渲染；品牌色过暗的（AlmaLinux / CentOS / macOS）在深色模式下自动转为前景浅色，保证可见。

## 详情页（客户端路由 `/node/{id}`）
- 4 张指标卡：CPU(·核数) / 内存 / 流量速率 / 到期
- **3 网延迟**卡（含丢包率）
- **性能统计**：时间范围切换 **1 小时 / 6 小时 / 12 小时 / 24 小时 / 7 天**，
  5 张图表（CPU 使用率 / 内存占用 / 硬盘占用 / 网络速率 RX·TX / **网络延迟（按线路，3 条折线 + 图例）**），
  内联 SVG 折线，零依赖，每图带当前值与时间轴（≤24h 显示时:分，7 天显示月/日）
- 系统信息网格：OS / 内核 / 架构 / 虚拟化 / CPU / 核数 / 内存·硬盘·交换总量 / Agent 版本 / 价格 / 累计流量
> 路由前缀 `/node/{id}`。若前面有按路径放行的反代/WAF，需把该前缀加进白名单。

## 对接的接口（文档五接口，同源）
- `GET /api/me`          站点名 / 公开页开关
- `GET /api/nodes`       节点列表 + 实时指标 + 累计流量（主数据）
- `GET /api/nodes/{id}/metrics?hours=N&series=ping`     3 网延迟
- `GET /api/nodes/{id}/metrics?hours=N&points=180&series=metrics`  详情页图表（N 随范围切换）
- `GET /api/ws?gzip`     每 2s 推快照；断线 / 满额 503 时退回 5s 轮询 `/api/nodes`
- `GET /api/themes/minimal/config`  读站点级设置（公告）；未保存 = `{}` 按默认渲染

## 关键实现（按文档规范）
- 到期：用 `expires_in`（0=今天、负=已过期），旧版 hub 没有才回退 `expires_at` 自算
- 离线时长：用 `last_seen_ago`（秒），不在浏览器里减时钟
- 三态：`metrics:null` 判「不可用」；`online:true + metrics:null` 显示「指标未上报」
- gzip 双帧：文本 / Blob（DecompressionStream）都能收，逐帧串行解码
- `visibilitychange`：后台关 WS 停轮询，回前台立即刷一次 + 重连
- 货币走 `Intl.NumberFormat(zh-CN)`，付款周期 `monthly/.../<n>m/once` 中文化
- 所有节点自报字段（os/kernel/remark…）按纯文本转义，不用 innerHTML

## 本地预览（无 hub）
`dev-mock.cjs` 是零依赖 mock（12 节点、覆盖多发行版，含 `last_seen_ago/expires_in` 假数据 + 真 WS 握手）：
```
env -u NODE_OPTIONS node dev-mock.cjs minimal/dist 9932
# Edge 无头截图（必须 --no-proxy-server）
"$B" --headless --no-sandbox --no-proxy-server --window-size=1440,900 \
  --virtual-time-budget=9000 --screenshot=/tmp/x.png "http://127.0.0.1:9932/"
# 详情页： http://127.0.0.1:9932/node/1
# ?dark=1 深色 / ?frame=390 窄屏 / ?probe=1 横向溢出探针 / ?click=button[data-h="168"] 切范围
```
已验：12 卡片 / 12 真实图标 / 36 进度条、无 NaN/undefined、无横向溢出（VW=DOCW）、
卡片同行等高、时间范围切换重拉数据、归档解包后一致。

## 安装
把 `theme.tar.gz` 上传到 hub 面板，或解包放进 hub 的 `--themes` 目录（目录名须 = `minimal`）。

## 版本
- 1.0.0  首版：紧凑表格
- 1.0.1  上下行合并、3 网延迟、内存/硬盘/总流量分列、删顶部汇总卡片
- 1.1.0  彩色自绘图标、列对齐修复、节点详情页
- 1.2.0  **主界面改卡片网格**、**图标换成 Simple Icons 真实品牌 logo**、
         **详情页性能图表 + 1h/6h/12h/24h/7d 时间范围切换**
- 1.2.1  主界面加**深浅色切换按钮**、卡片更紧凑（4 列 / 卡高 ~208px）、去掉返回按钮下划线
- 1.2.2  主界面延迟改**色带热力图**（3 线路 × 5 档色阶）、详情页新增**网络延迟折线图**（按线路，随范围切换）
- 1.2.3  美化重排：主界面卡片**进度条 / 延迟色带 / 流量 / 到期统一网格逐列对齐**，
         延迟行线路名改**彩色圆点**标识；详情页**图标统一线框风格**（返回 / 章节 / 指标卡 / 图例），修复图标超大渲染

## 说明
本主题本地开发中，**未推送 GitHub、未发 release**（按交付纪律，审核通过再动远端）。
