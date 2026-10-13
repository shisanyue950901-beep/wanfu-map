# 万福地图 · 跑山路线规划 (v3)

Motorcycle-club ride planner for Zhengzhou 跑山 trips. React + Zustand + Tailwind, AMap JS API 2.0,
built into ONE inlined HTML file and served by the Cloudflare Worker `wanfu-map`
(https://wanfu-map.shisanyue001.workers.dev — workers.dev only, no custom domain).

## What it does
- Multi-day trips, spot cards with AMap POI photos, add-spot search (AutoComplete + PlaceSearch), delete with undo,
  long-press / right-click to delete a day, restore demo. Owner data autosaves to localStorage (`ride-planner:v1`).
- **Real road routes** per leg via `AMap.Driving` (motorcycle ≈ car). Per-day 「避开高速」 (default ON) → Driving policy **13**
  (不走高速; verified: 0 toll km vs 23.7 toll km with policy 0). Orange polyline with glow + direction arrows; a leg that
  can't be routed falls back to a dashed straight line. Legs are cached in localStorage (`wanfu:routes:v1`) keyed by
  `from>to|policy`, so reloads don't re-query. Some POIs (e.g. 少林寺) fail as a *destination*; fallback chain is
  forward → reverse (path flipped) → nudged destination.
- Cards: 「距上一站 36 km · 约 42 分钟」 + 「导航」 (AMap URI API; phones start from current location, desktop from the previous stop).
  Day header: real km, ride time, stay time; 「全程导航」 lists per-leg links (URI API supports only 1 via point; ≤3 stops get a one-tap link).
- **Weather** via `AMap.Weather` + `AMap.Geocoder` (adcode of the day's farthest stop, cached in `wanfu:adcode:v1`).
  Day has an optional date (date picker): inside the 4-day window → 「预报」, otherwise live → 「实时」. Riding advisory chip
  (雨天路滑 / 大风注意横风 / 能见度低 / 高温 / 低温 / 冰雪 / 适合跑山). Weather mood overlay over the map (CSS tints + a
  30fps DPR-1 canvas for rain/snow/wind; pointer-events none; static under prefers-reduced-motion; paused when hidden).
- **Share to members (read-only)**: owner taps 「分享」 → 「发布分享 / 更新分享」. Status: 未发布 / 已同步 / 有未发布修改.
  Members open `/t/<id>`: same UI, no add/delete/settings/toggles, banner 「万福地图 · 俱乐部路线 · 只读」, real routes come
  pre-packed in the payload (no re-query), weather + 导航 work. Owner sheet has 复制链接, 系统分享 (navigator.share), QR code
  (qrcode-generator, client-side) and 预览车友视图.

## v3: 集合点 · 报名 · 签到 · 出发动画
- **集合点 (meetup)**: every day has `Day.meetup = { name, lng, lat, address, meetTime, departTime, note }`, shown as a pinned
  amber 「集合点」 block above the stops (editable, not deletable) and ONE flag-style marker (created once, moved / relabelled /
  hidden in place). Organizer edits it via AMap AutoComplete search or 「在地图上选点」 (tap the map or drag the flag →
  `AMap.Geocoder` reverse geocode). Routes start at the meetup: leg 0 = meetup → stop 1 (`dayStops()` in `types.ts`).
  Migration: store version 1 → 2 (and v2 share payloads on load): a day whose first spot is tagged 集合 becomes the meetup
  (time parsed from its note, depart = meet + its stay). Member view: meet/depart tiles, 距出发 countdown (when the day has a date,
  China time), 「导航到集合点」 (uri.amap.com/navigation).
- **报名 (no accounts, per published trip)**: member opens `/t/<id>` → 「我要报名」 (昵称, 11-digit 手机号, 车型, 是否带人, 备注).
  Server gives a member token (only its SHA-256 stored); the device keeps it in `localStorage['wanfu:member:v1']` → 「已报名 ✓ / 取消报名」.
  Organizer = the browser holding the trip's edit token (owner view, or `/t/<id>` in that same browser): 名额上限, 报名截止, 开放/关闭,
  签到范围 300/500/1000 m; roster with tel: links, remove, manual check-in / undo, CSV export (UTF-8 BOM, Excel-friendly).
  Members only ever get count + nicknames; phone numbers are returned only with the edit token. Duplicate phone per trip → 409
  (UNIQUE index too); cap enforced with a conditional INSERT; signups rate-limited 30 / IP / 10 min.
- **签到**: 「到达签到」 uses `navigator.geolocation` (WGS-84). The Worker converts to GCJ-02 (the AMap datum; ~500 m offset around
  Zhengzhou) and allows it within the organizer's radius (+ up to 100 m GPS accuracy slack). Denied / failed geolocation →
  「请让组织者手动签到」. Roster shows 已到 X / 报名 Y + departure countdown, polling every 15 s while open (members poll 30 s).
  Check-in is stored once per signup with `checkin_day` (multi-day trips: the latest day's check-in wins).
- **出发 / 行程播放**: 「▶ 出发」 per day. Route cache legs now carry `st` = flat `[endIdx, meters, seconds]` per AMap Driving step
  (each step simplified separately so boundaries survive; legs cached before v3 are re-fetched, old path stays on screen meanwhile).
  `playback.ts` builds a timeline: every step moves at its own real speed (step distance / step time), each stop pauses for its
  stayMin, simulated clock starts at departTime. Rates 60× / 120× (default; 1 real s = 2 sim min, 1 real min = 2 sim h) / 300×.
  HUD: sim clock, segment speed (km/h), km ridden, next stop + ETA, scrubber with stop ticks, play/pause/replay, stop.
  Map: one bike marker (SVG, rotated to the on-screen heading by projecting a 40 m look-ahead point), one bright "traveled" trail
  (+ a 2-point head line updated per frame), remaining legs dimmed in place. Camera: 3D follow at pitch 60, zoom per rate, rotation
  re-aims only on >14° turns and eases slowly (300 m look-ahead), bike kept in the visible part of the map (left of the desktop
  panel / above the phone HUD) by feedback on its projected pixel; dragging the map pauses follow for 5 s. Arrivals: card + marker
  highlight and toast 「抵达 X · 停留 N 分钟」. prefers-reduced-motion: player opens paused, no camera follow.

## Backend (worker.js)
- `GET /` → app. `GET /t/:id` → app with `window.__WF_SHARE__ = {id, data}` injected (+ title/og tags); unknown id → friendly notice.
- `POST /api/trips` `{data}` → `{id, token, updated_at}` (token kept in owner localStorage `wanfu:share-owner:v1`; only its SHA-256 is stored).
- `PUT /api/trips/:id` with `Authorization: Bearer <token>` → update. `GET /api/trips/:id` → JSON.
- Payload ≤ 200 KB, whitelisted/sanitised server-side (https photos only, clamped strings/counts). Writes rate-limited to
  30 per IP per 10 min (D1 table `writes`). `GET /healthz` → `{ok, creds, db}`.
- v3 API (all under `/api/trips/:id`): `GET /reg?day=` (public: settings, count, arrived, nicknames, `me` with `X-Member`;
  `list` with phones only with `Authorization: Bearer <edit token>`), `PUT /reg` (organizer), `POST /signups`, `DELETE /signups/me`
  (member), `POST /checkin` (member, `{day, lat, lng, acc}` WGS-84), `DELETE /signups/:sid`, `POST /signups/:sid/checkin {day, undo}`
  (organizer). Share payload v3 keeps `meetup` per day and `s` (step timing) per leg.
- D1 database `wanfu` (id `aefb7ddb-e08f-4bac-a83a-a0e98bbaa025`) bound as `DB`; schema in `schema.sql`
  (`trips`, `writes`, v3: `signups` (UNIQUE trip_id+phone), `trip_reg` settings). Apply with `APPLY_SCHEMA=1 ./deploy.sh`.
- AMap key + security code come from the Worker **Secret** `AMAP_CREDS` (`KEY|SECURITY_CODE`), injected at request time.
  Never commit or print it; `wrangler deploy` leaves secrets alone (`keep_vars = true` as well).

## Flicker fix (v1 → v2)
v1 rebuilt **every marker and the polyline** each time the runtime POI enrichment snapped a demo spot's coordinates
(4 full rebuilds in the first ~3 s, while the cinematic camera intro was running), wrote the whole trip to the store once per
spot (re-render storm + localStorage writes), stacked an extra `AMap.Buildings` layer on top of the base map's own
buildings (z-fighting when zoomed in), and drew `backdrop-filter` glass over the live WebGL canvas — which iOS Safari
re-samples on every map frame. v2: demo ships with real POI coordinates/photos baked in; markers and route legs are
diffed and updated in place (`setPosition` / `setPath`), never torn down; enrichment runs once per spot and commits one
batched write; no extra Buildings layer; no backdrop-filter on phones/touch devices (near-opaque fills instead); map
init is guarded to run exactly once (StrictMode removed); sheet height ignores visualViewport jitter; root is `position:fixed`.
Probe (`scripts/flicker.py`, WebKit iPhone 13 + Chromium): 1 map, 1 `complete`, 4 marker creations, 0 marker rebuilds over 12 s.
v3 (2026-10-13): 1 map, 1 `complete`, 3 stop markers + 1 flag created once, 0 marker/flag updates, playback adds exactly one bike
marker (`bikeCreates: 1`). Also: the app root is `overflow: clip` — `overflow: hidden` boxes can still be scrolled by
`scrollIntoView()`, which had shifted the whole layout up on phones (sheet stuck mid-screen, map taps landing on the sheet).

## Dev / deploy
```
./deploy.sh                      # tsc + vite build + inline + wrangler deploy (+ healthz)
APPLY_SCHEMA=1 ./deploy.sh       # also (re)apply schema.sql to remote D1 (idempotent)
python3 scripts/e2e.py webkit    # ~37 end-to-end checks incl. v3 (or: chromium); deletes its test trip + signups
python3 scripts/flicker.py       # frame/marker/map-instance probe
python3 scripts/shot.py [DEMO_ID] [DATE]   # shots/v3-*.png; republishes the demo onto DEMO_ID (default u3Nksycq)
```
Wrangler: `npx --prefix /workspace/work/vgrab-cf wrangler` (needs `CLOUDFLARE_API_TOKEN`; account `7137f84a7c341e4a427bc9eb47cbe56b`).
Demo share `/t/u3Nksycq`: its edit token is kept in `.demo-owner.json` (0600, not committed / never printed).
Tests need `pip install playwright`, `playwright install webkit` (+ `sudo playwright install-deps`), and `/usr/bin/chromium`.
Screenshots use Playwright WebKit (Chromium in this sandbox falls back to AMap's non-WebGL raster mode → light tiles).