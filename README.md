# 极简主题（short: minimal）

monitor 状态页的极简黑白单文件主题，主界面一张紧凑表格，一眼看全每个节点的：
在线状态 / CPU·内存 / 延迟 / 上下行流量 / 到期·价格 / 离线时长，外加顶部汇总条
（在线率、全站上下行、本月流量）。

## 文件
- `dist/index.html`  —— 主题本体（零构建、单文件、内联 CSS/JS，约 17KB，无依赖）
- `theme.json`       —— 六字段 + config（公告）
- `preview.png`      —— 面板预览图（浅色）
- `theme.tar.gz`     —— 安装包（`gzip -t` 通过；顶层 `dist/ theme.json preview.png`，无外层套目录）

## 对接的接口（文档五接口，同源）
- `GET /api/me`          站点名 / 公开页开关
- `GET /api/nodes`       节点列表 + 实时指标 + 累计流量（主数据）
- `GET /api/nodes/{id}/metrics?hours=1&series=ping`  延迟（对在线节点分批 ≤20 补拉；若 hub 在 nodes/WS 里带 ping 则首屏直接出）
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
`dev-mock.cjs` 是零依赖 mock（带 `last_seen_ago/expires_in` 假数据 + 真 WS 握手）：
```
env -u NODE_OPTIONS node dev-mock.cjs minimal/dist 9932
# Edge 无头截图（必须 --no-proxy-server）
"$B" --headless --no-sandbox --no-proxy-server --window-size=1440,980 \
  --virtual-time-budget=8000 --screenshot=/tmp/x.png "http://127.0.0.1:9932/"
# ?dark=1 深色 / ?frame=390 窄屏 / ?probe=1 横向溢出探针 / ?rects=... 列几何
```
已验：横向无溢出（VW=DOCW）、CPU 列 left 逐行一致、无 NaN/undefined、归档解包后渲染一致。

## 安装
把 `theme.tar.gz` 上传到 hub 面板，或解包放进 hub 的 `--themes` 目录（目录名须 = `minimal`）。

## 说明
本主题为本地交付，**未推送 GitHub、未发 release**（按交付纪律，审核通过再动远端）。
