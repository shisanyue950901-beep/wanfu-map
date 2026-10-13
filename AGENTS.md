# 万福地图 · 交接说明 (for Codex / other agents)

郑州机车俱乐部跑山行程规划 + 会员只读分享。地图为主体的深色 3D 高德地图页面，手机和电脑都要顺滑，第一眼要精致。

## 线上
- 主站: https://wanfu-map.shisanyue001.workers.dev （Cloudflare Worker `wanfu-map`）
- 会员示例: https://wanfu-map.shisanyue001.workers.dev/t/u3Nksycq
- 健康检查: `/healthz` → `{ok, creds, db}`
- 已知问题: `*.workers.dev` 在中国大陆经常被屏蔽/不稳定（2026-10-13 用户大陆网络白屏）。下一步最优先: 绑定自定义域名（Cloudflare Workers Custom Domain / Route）。

## 技术栈
Vite + React + TypeScript + Tailwind + Zustand(persist, localStorage key `ride-planner:v1`)；高德 JS API 2.0。
构建产物是**单文件 HTML**，由 `scripts/inline.mjs` 内嵌进 `html.js`，再被 `worker.js` 直接返回（不用 Pages / 静态资源上传）。
后端全部在 `worker.js`：分享行程、报名、签到 API；D1 数据库 `wanfu`（binding `DB`，表结构见 `schema.sql`，幂等）。

## 密钥（不在仓库里）
- `AMAP_CREDS`: Worker Secret，格式 `KEY|安全密钥`（worker 按 `| ｜ , 空格` 切分），通过 `/amap-config` 之类注入前端。用 `wrangler secret put AMAP_CREDS` 设置。
- 部署需要 `CLOUDFLARE_API_TOKEN` 环境变量。
- 分享编辑 token 只存哈希；组织者编辑权在其浏览器 localStorage（`wanfu:share-owner:v1`）。

## 开发 / 部署
```bash
npm install
npm run build        # tsc --noEmit && vite build && inline → html.js
npx wrangler deploy  # 或 ./deploy.sh（脚本里 wrangler 路径是原沙箱的，改成 npx wrangler 即可）
APPLY_SCHEMA=1 ./deploy.sh   # 同时执行 schema.sql
```

## 功能现状 (v3, 2026-10-13)
- 按天行程、景点卡片（照片）、添加/删除/撤销、地图联动；手机底部三档抽屉、电脑右侧面板。
- 真实道路路线 `AMap.Driving`，默认避开高速（策略 13），缓存含每段 step 的时长/距离；导航/全程导航用高德 URI（最多一个途经点）。
- 天气：高德天气 + 逆地理，4 日内预报，雨雪风雾地图特效，尊重 prefers-reduced-motion。
- 每天独立**集合点**（搜索 / 地图点选 / 拖动旗标，集合时间、出发时间、备注）；旧数据中 tag=集合 的首个景点自动迁移。
- 只读分享 `/t/<id>`、复制/系统分享/二维码，200KB 限制、IP 限流。
- **报名**（无账号，按行程）：昵称、11 位手机号、车型、带人、备注；去重、名额、截止；手机号只返回给组织者；CSV 导出。
- **签到**：GPS（服务端 WGS84→GCJ-02 后判断 300/500/1000 m 半径）或组织者手动；组织者面板 15s 轮询。
- **▶ 出发回放**：摩托沿真实路线按每段真实速度运动，模拟时钟、时速、里程、到站停留，60×/120×/300×。

## 必须遵守（屏闪教训）
- 地图只初始化一次；标记/路线/旗标**原地更新**，绝不销毁重建。
- 触摸设备上不要在地图上方使用 `backdrop-filter`。
- 不要额外加 Buildings 图层。
- 每次改动后跑 `scripts/e2e.py`（Playwright，WebKit iPhone 13 + Chromium）和 `scripts/flicker.py`，看截图再交付。

## 已知缺口 / 下一步
1. 自定义域名（大陆可达性）。
2. 报名按整趟行程，多天行程每人只有一条签到记录。
3. 手机上回放 HUD 约 210px，地图区域偏小。
4. 未在真机 iPhone / 大陆网络测试；GPS 可被伪造。
5. 无取消发布/删除分享接口；清浏览器存储会失去分享编辑权。
6. 生产包里有只读调试钩子 `window.__wfUI`。
7. 待做：拖动排序、跨天移动；沿途加油站/补给图层；费用 AA；实时位置 + SOS（需要会员登录与实时后端）。

## 目录
- `src/` 前端（`components/` UI，`store.ts` 状态与迁移，`routes.ts` 路线，`playback.ts` 回放，`reg.ts` 报名签到客户端，`share.ts` 分享，`weather.ts`，`amap.ts` 加载器，`demo.ts` 示例数据）
- `worker.js` Worker 后端；`schema.sql` D1 表；`wrangler.toml`
- `scripts/` 构建内联、e2e、截图、防屏闪探针、GCJ 换算
- `shots/` v2/v3 截图