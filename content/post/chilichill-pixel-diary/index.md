---
title: "ChiliChill 巡演像素日记：零依赖像素风全栈 Web 应用技术报告"
date: 2026-07-18T12:00:00+08:00
draft: false
tags: ["Next.js", "React", "TypeScript", "Supabase", "Cloudflare R2", "Canvas", "像素风", "全栈"]
categories: ["技术"]
summary: "为独立乐队 ChiliChill 全国巡演打造的像素复古游戏风粉丝留言产品。涵盖 GeoJSON 自定义地图渲染、标记碰撞分散算法、手写 AWS4 签名直传 R2、Canvas 分享卡生成、CRT 扫描线特效、API 优先 + Mock 降级双模架构等技术细节。"
---

# ChiliChill 巡演像素日记：零依赖像素风全栈 Web 应用技术报告

> **文档版本**: v1.0
> **撰写日期**: 2026-07-18
> **作者**: kris
> **技术栈**: Next.js 15 / React 19 / TypeScript / Supabase / Cloudflare R2
> **设计风格**: 90年代复古游戏机 + CRT 显像管

---

## 目录

- [1. 项目概览与设计目标](#1-项目概览与设计目标)
- [2. 技术栈与架构设计](#2-技术栈与架构设计)
- [3. GeoJSON 中国地图渲染引擎](#3-geojson-中国地图渲染引擎)
- [4. 图片上传：手写 AWS4 签名 + R2 直传](#4-图片上传手写-aws4-签名--r2-直传)
- [5. Canvas 像素风分享卡生成](#5-canvas-像素风分享卡生成)
- [6. CRT 扫描线与像素动画系统](#6-crt-扫描线与像素动画系统)
- [7. 数据层：API 优先 + Mock 降级](#7-数据层api-优先--mock-降级)
- [8. 数据库设计与 RLS 安全策略](#8-数据库设计与-rls-安全策略)
- [9. 互动系统：乐观更新与去重](#9-互动系统乐观更新与去重)
- [10. 问题、踩坑与设计取舍](#10-问题踩坑与设计取舍)
- [11. 附录与参考](#11-附录与参考)

---

## 1. 项目概览与设计目标

### 1.1 背景

**ChiliChill** 是一支中国独立乐队，正在进行全国巡演。项目的目标是为粉丝打造一个像素复古游戏风的互动留言产品——粉丝在一张像素风中国地图上点击巡演城市，打开该站的「日记本」，查看和发布现场回忆。

**核心设计理念**：
- **90 年代复古游戏机质感**：CRT 扫描线、像素字体、`steps()` 动画
- **零第三方 UI 库**：没有 Tailwind、shadcn、Ant Design，全部手写 CSS
- **离线可用**：无 Supabase 时自动降级到 localStorage 完整运行
- **移动优先**：单栏布局 + 侧滑城市抽屉，桌面端双栏并排

### 1.2 功能矩阵

| 功能模块 | 说明 | 技术亮点 |
|----------|------|---------|
| CRT 开机动画 | 像素 logo + 进度条 + 小人出场 | CSS `steps()` 逐帧动画 |
| 中国行政区地图 | 真实 GeoJSON 省份高亮 + 城市标记 | 自定义 SVG 投影 + 碰撞分散 |
| 巡演路线 | 金色虚线连接各站，流动效果 | SVG polyline + CSS 动画 |
| 留言墙 | 分页加载 / 排序切换 / 嵌套回复 | 乐观更新 + 分页 |
| 图片上传 | 最多 6 张 + 客户端缩略图 | 手写 AWS4 签名 + R2 直传 |
| 点赞/比心 | 双反应 + visitorId 去重 | 乐观更新 + 回滚 |
| 分享卡 | Canvas 绘制像素风 PNG | 伪 QR 码 + 复古游戏机外壳 |
| 管理后台 | 站点 CRUD / 留言审核 / 系统诊断 | 隐藏路径 + httpOnly Cookie |

### 1.3 运行环境

| 项目 | 值 | 备注 |
|------|-----|------|
| 框架 | Next.js 15 (App Router) + React 19 | |
| 数据库 | Supabase (PostgreSQL + RLS) | |
| 对象存储 | Cloudflare R2 (S3 兼容) | |
| 部署 | Vercel | |
| Monorepo | pnpm workspace + Turborepo | |
| CSS | 纯手写 (836 行全局 CSS) | 零 UI 框架 |

---

## 2. 技术栈与架构设计

### 2.1 Monorepo 结构

```
chilichill-diary/
├── apps/
│   └── web/                    # Next.js 15 Web 应用
│       ├── app/
│       │   ├── store.tsx              # 全局状态管理 (395行)
│       │   ├── globals.css            # 全量样式 (836行)
│       │   ├── components/            # 12 个组件
│       │   └── api/                   # 12 个 API 路由
│       └── public/
│           └── data/china-*.json      # 中国省级 GeoJSON
│
├── packages/
│   ├── shared/     # @chili/shared — 类型 + 常量
│   ├── db/         # @chili/db — 数据访问层 (API优先 + Mock降级)
│   ├── ui/         # @chili/ui — UI 工具函数
│   ├── scene/      # @chili/scene — 像素地形 Canvas 引擎
│   └── assets/     # @chili/assets — 美术素材元数据
│
└── docs/
    └── supabase/schema.sql     # 数据库 DDL + RLS 策略
```

### 2.2 零第三方 UI 依赖

整个项目没有使用任何 UI 框架。全部视觉效果由 836 行手写 CSS 实现，包括：
- CRT 扫描线 + 暗角
- 像素风按钮/卡片/弹窗
- 响应式双栏布局
- 侧滑抽屉动画
- 自定义滚动条
- 留言卡片虚拟渲染（`content-visibility: auto`）

---

## 3. GeoJSON 中国地图渲染引擎

### 3.1 设计思路

没有使用任何地图库（Leaflet、Mapbox 等），完全自定义实现。加载中国省级 GeoJSON，用线性投影将经纬度转为 SVG 坐标，渲染为可交互的行政区划地图。

### 3.2 经纬度 → SVG 投影

```typescript
// TourMap.tsx — 中国版图边界
const CHINA_BOUNDS = {
  minLng: 73.5,   maxLng: 135.1,
  minLat: 18,     maxLat: 53.6,
};

// 经纬度 → SVG 坐标（线性映射）
function project([lng, lat]: Coord) {
  const x = MAP_PAD_X
    + ((lng - CHINA_BOUNDS.minLng) / (CHINA_BOUNDS.maxLng - CHINA_BOUNDS.minLng)) * MAP_W;
  const y = MAP_PAD_Y
    + ((CHINA_BOUNDS.maxLat - lat) / (CHINA_BOUNDS.maxLat - CHINA_BOUNDS.minLat)) * MAP_H;
  return { x, y };
}
```

### 3.3 GeoJSON → SVG Path

将 Polygon/MultiPolygon 几何体转为 SVG `<path>` 的 `d` 属性：

```typescript
function pathFromRing(ring: Ring) {
  return ring
    .filter(([, lat]) => lat >= 17.5)  // 过滤南海诸岛极南端
    .map((coord, index) => {
      const point = project(coord);
      return `${index === 0 ? 'M' : 'L'}${point.x.toFixed(2)},${point.y.toFixed(2)}`;
    })
    .join(' ');
}

function pathFromFeature(feature: ChinaFeature) {
  const polygons = feature.geometry.type === 'Polygon'
    ? [feature.geometry.coordinates]
    : feature.geometry.coordinates;
  return polygons
    .map((polygon) => polygon.map(pathFromRing).filter(Boolean).join(' Z '))
    .filter(Boolean)
    .join(' Z ')
    .concat(' Z');
}
```

### 3.4 省份着色策略

根据巡演站点状态对省份着色，优先级：正在 > 即将 > 已去过：

```typescript
function toneForFeature(provinceStations: Station[]): StationStatus | 'plain' {
  if (provinceStations.some((s) => s.status === 'live'))     return 'live';     // 红
  if (provinceStations.some((s) => s.status === 'upcoming')) return 'upcoming'; // 黄
  if (provinceStations.some((s) => s.status === 'done'))     return 'done';     // 绿
  return 'plain';  // 灰
}
```

### 3.5 标记碰撞分散算法

当多个城市标记位置重叠时，通过 10 轮迭代将它们推开，同时限制最大偏移距离：

```typescript
// 10 轮物理模拟
for (let pass = 0; pass < 10; pass++) {
  let moved = false;
  for (const g of visibleCityGroups) {
    let cx = spreadOffsets[s.id].x;
    let cy = spreadOffsets[s.id].y;
    for (const og of visibleCityGroups) {
      if (og === g) continue;
      const dx = cx - ox, dy = cy - oy;
      const dist = Math.sqrt(dx * dx + dy * dy);
      if (dist > 0 && dist < 2.8) {
        // 距离 < 2.8 时施加排斥力
        const push = (2.8 - dist) * 0.4;
        const angle = Math.atan2(dy, dx);
        cx += Math.cos(angle) * push;
        cy += Math.sin(angle) * push;
        moved = true;
      }
    }
    // 限制偏移不超过原始位置 3.5 单位
    const sd = Math.sqrt(sdx * sdx + sdy * sdy);
    if (sd > 3.5) {
      const scale = 3.5 / sd;
      cx = base.x + sdx * scale;
      cy = base.y + sdy * scale;
    }
  }
  if (!moved) break;  // 收敛则提前退出
}
```

**参数解读**：
- **最小间距 2.8**：标记间至少保持 2.8 SVG 单位距离
- **排斥系数 0.4**：每轮推动力度为间距差的 40%
- **最大偏移 3.5**：标记不会偏离原始位置超过 3.5 单位
- **最多 10 轮**：防止死循环，实测通常 3-5 轮收敛

---

## 4. 图片上传：手写 AWS4 签名 + R2 直传

### 4.1 架构：浏览器直传

传统做法是图片先上传到服务端再转存，但这消耗服务端带宽。本项目采用 **预签名 URL 直传**：

```
浏览器 → POST /api/uploads/presign → 获取预签名 URL
浏览器 → PUT 直接上传到 R2 (绕过服务端)
```

### 4.2 手写 AWS4 签名

没有使用 AWS SDK，仅用 Node.js `crypto` 模块手动实现完整的 AWS Signature V4：

```typescript
// presign/route.ts — 零依赖 AWS4 签名

function signingKey(secret: string, date: string, region: string, service: string) {
  const kDate    = hmac(`AWS4${secret}`, date);
  const kRegion  = hmac(kDate, region);
  const kService = hmac(kRegion, service);
  return hmac(kService, 'aws4_request');
}

function presignPutUrl(input) {
  const canonicalRequest = [
    'PUT',
    canonicalUri,
    params.toString(),
    `content-type:${input.contentType}\nhost:${host}\n`,
    signedHeaders,
    'UNSIGNED-PAYLOAD',
  ].join('\n');

  const stringToSign = [
    'AWS4-HMAC-SHA256',
    timestamp,
    credentialScope,
    hash(canonicalRequest),
  ].join('\n');

  const signature = hexHmac(
    signingKey(input.secretAccessKey, date, region, service),
    stringToSign
  );

  return `https://${host}${canonicalUri}?${params}`;
}
```

### 4.3 客户端缩略图生成

浏览器端使用 Canvas API 生成 WebP 缩略图，减少列表加载流量：

```typescript
// Composer.tsx
const THUMB_MAX_EDGE = 480;
const THUMB_QUALITY = 0.72;

async function createThumbnail(file: File): Promise<File | null> {
  if (file.type === 'image/gif') return null;   // GIF 不生成缩略图
  const image = await blobToImage(file);

  // 按最长边等比缩放到 480px
  const scale = Math.min(1, THUMB_MAX_EDGE / Math.max(image.naturalWidth, image.naturalHeight));
  const width  = Math.round(image.naturalWidth * scale);
  const height = Math.round(image.naturalHeight * scale);

  const canvas = document.createElement('canvas');
  canvas.width = width;
  canvas.height = height;
  canvas.getContext('2d')!.drawImage(image, 0, 0, width, height);

  const blob = await canvasToBlob(canvas);  // WebP, 72% 质量
  return new File([blob], `${baseName}-thumb.webp`, { type: 'image/webp' });
}
```

**完整上传流程**：
```
1. 校验类型 (jpg/png/webp/gif) + 大小 (≤10MB) + 数量 (≤6张)
2. POST /api/uploads/presign → 获取预签名 URL
3. PUT 原图到 R2
4. Canvas 生成 480px WebP 缩略图
5. PUT 缩略图到 R2
6. 返回 { url, thumbUrl } 保存到 message_images 表
```

---

## 5. Canvas 像素风分享卡生成

### 5.1 三种分享模式

| 模式 | 内容 | 副标题 |
|------|------|--------|
| `page` | 当前站/全站统计 | TOUR DIARY |
| `footprint` | 个人打卡城市 + 日记数 | MY FOOTPRINT |
| `message` | 单条留言完整展示 | FIELD NOTE |

### 5.2 伪二维码生成

使用基于种子的伪随机数生成器 + QR 码 finder pattern 模拟二维码视觉：

```typescript
function drawPixelQR(ctx, x, y, size, seed) {
  const cells = 25;
  const cell = size / cells;

  // 伪随机数生成器（线性同余法）
  let hash = 0;
  for (let i = 0; i < seed.length; i++)
    hash = ((hash << 5) - hash + seed.charCodeAt(i)) | 0;
  const rng = () => {
    hash = ((hash * 16807) % 2147483647);
    return (hash & 0x7fffffff) / 0x7fffffff;
  };

  // 随机填充像素
  for (let i = 0; i < cells; i++)
    for (let j = 0; j < cells; j++)
      if (rng() > 0.48) ctx.fillRect(x + i * cell, y + j * cell, cell, cell);

  // 三个角的 finder pattern（真实 QR 码标志）
  const drawFinder = (ox, oy) => {
    ctx.fillStyle = '#0a0518';
    ctx.fillRect(x + ox * cell, y + oy * cell, 7 * cell, 7 * cell);
    ctx.fillStyle = '#f4e7d3';
    ctx.fillRect(x + (ox+1) * cell, y + (oy+1) * cell, 5 * cell, 5 * cell);
    ctx.fillStyle = '#0a0518';
    ctx.fillRect(x + (ox+2) * cell, y + (oy+2) * cell, 3 * cell, 3 * cell);
  };
  drawFinder(0, 0);
  drawFinder(cells - 7, 0);
  drawFinder(0, cells - 7);
}
```

### 5.3 复古游戏机外壳绘制

```typescript
function drawRetroFrame(ctx) {
  const pad = 48;
  // 外壳
  ctx.fillStyle = '#0e081c';
  ctx.fillRect(pad, pad, CARD_W - pad*2, CARD_H - pad*2);
  // 内框
  ctx.fillStyle = '#160f2d';
  ctx.fillRect(pad + 12, pad + 12, ...);
  // 四角金色装饰
  ctx.fillStyle = '#ffd23f';
  [[pad+8, pad+8], [pad+w-20, pad+8], ...].forEach(([cx, cy]) =>
    ctx.fillRect(cx, cy, 12, 12)
  );
  // 扫描线叠加
  ctx.globalAlpha = 0.06;
  for (let y = pad; y < pad + h; y += 4) {
    ctx.fillStyle = '#000';
    ctx.fillRect(pad, y, w, 1);
  }
  ctx.globalAlpha = 1;
}
```

分享卡还包含：文字自动换行 + 行数测量、星星评分渲染、心情表情绘制等。最终生成 1080x1440 PNG 供下载/分享。

---

## 6. CRT 扫描线与像素动画系统

### 6.1 CRT 显像管效果

纯 CSS 实现 CRT 电视质感，零 JS：

```css
/* 扫描线 */
.crt::before {
  content: '';
  position: fixed;
  inset: 0;
  background: repeating-linear-gradient(
    0deg,
    rgba(0,0,0,.08) 0,
    rgba(0,0,0,.08) 1px,
    transparent 2px,
    transparent 3px
  );
  pointer-events: none;
  z-index: 9999;
}

/* 屏幕暗角 */
.crt::after {
  content: '';
  position: fixed;
  inset: 0;
  background: radial-gradient(
    ellipse at center,
    transparent 55%,
    rgba(0,0,0,.55) 100%
  );
  pointer-events: none;
  z-index: 9998;
}
```

### 6.2 像素逐帧动画

全部使用 CSS `steps()` 函数模拟逐帧渲染，完美匹配像素风美术：

| 动画 | CSS | 效果 |
|------|-----|------|
| 开机 logo | `animation: bootGlow 1.4s steps(8)` | 缩放 + 模糊渐现 |
| 进度条 | `animation: bootFill 1.6s steps(40)` | 40 步填充 |
| 页面切换 | `animation: pageIn .32s steps(6)` | 6 步放大 |
| 路线流动 | `animation: routeDash 1.6s linear infinite` | 虚线持续流动 |
| 标记弹出 | `animation: markerPop 0.35s cubic-bezier(.34,1.56,.64,1)` | 弹性弹出 |
| 弹窗入场 | `animation: sheetUpBounce .4s cubic-bezier(.34,1.56,.64,1)` | 带弹性底部滑入 |

### 6.3 响应式策略

```css
/* 移动端（<520px）：单栏 + 侧滑抽屉 */
@media (max-width: 520px) { ... }

/* 桌面端（≥980px）：双栏并排 */
@media (min-width: 980px) {
  .crt-shell {
    display: grid;
    grid-template-columns: 46% 54%;
  }
}
```

---

## 7. 数据层：API 优先 + Mock 降级

### 7.1 双模架构

这是整个项目最重要的设计模式。`@chili/db` 包的每个数据函数先尝试调用 Next.js API，失败时自动降级到 localStorage 完整 mock：

```typescript
// packages/db/src/index.ts
export async function listStations() {
  return (await api<...>('/api/stations')) ?? mock.listStations();
}

export async function listMessages(stationId, options) {
  return (await api<...>('/api/messages?...')) ?? mock.pageMessages(stationId, options);
}
```

### 7.2 Mock 层的完整实现

`mock.ts` 不是简单的占位，而是一套**完整的 localStorage 持久化数据层**，实现了：

- 分页查询（`limit` / `offset` / `order`）
- 嵌套回复归组（`nestMessages`）
- 互动状态计算（`withReactionState`）
- CRUD 全套操作
- 种子数据自动初始化

这意味着**没有任何后端服务也能完整运行**——开发者克隆项目后立即可用，零配置。

### 7.3 状态管理

使用 React Context + `useCallback` 实现全局状态，没有引入任何第三方状态管理库（Redux / Zustand / Jotai）：

```typescript
// store.tsx (395行)
const AppContext = createContext<AppContextValue | null>(null);

// 70+ 个状态字段和方法，包括：
// - 路由（screen / wallMode）
// - 数据（stations / messages）
// - 分页（messagesHasMore / loadMoreMessages）
// - 互动（toggleReaction）
// - 认证（user / login / logout）
// - 弹窗（lightbox / composer / share）
// - 管理（allMsgs / users / refreshAdmin）
```

---

## 8. 数据库设计与 RLS 安全策略

### 8.1 四表结构

```sql
stations          -- 巡演站点（城市、场馆、日期、地图坐标、状态）
messages          -- 留言（文字、心情、评分、自引用回复）
message_images    -- 留言图片（多图支持、缩略图 URL）
message_reactions -- 互动（like/heart，visitor_id 去重）
```

### 8.2 Row Level Security

通过 PostgreSQL RLS 实现数据层安全，无需应用层校验：

```sql
-- 站点：公开读取
create policy "Public station read"
  on stations for select using (true);

-- 留言：仅展示已发布
create policy "Public published message read"
  on messages for select using (status = 'published');

-- 留言插入：只能插入 pending 状态 + 非官方
create policy "Public pending message insert"
  on messages for insert
  with check (status = 'pending' and official = false);

-- 图片/互动：关联留言必须是 published
create policy "Public published message image read"
  on message_images for select using (
    exists (select 1 from messages
            where messages.id = message_images.message_id
              and messages.status = 'published')
  );
```

管理操作通过 Next.js API 路由 + Supabase `service_role_key` 执行，绕过 RLS。

### 8.3 认证设计

极简但有效：

| 角色 | 认证方式 | 会话 |
|------|---------|------|
| 粉丝 | 仅输入昵称 | localStorage |
| 管理员 | 昵称 `admin` + 服务端密码校验 | httpOnly Cookie (8h) |
| 匿名用户 | `visitorId`（自动生成） | localStorage |

```typescript
// 匿名用户 ID 生成
function getVisitorId() {
  const existing = localStorage.getItem('chilichill_visitor_id');
  if (existing) return existing;
  const next = `v_${Date.now().toString(36)}_${Math.random().toString(36).slice(2, 12)}`;
  localStorage.setItem('chilichill_visitor_id', next);
  return next;
}
```

---

## 9. 互动系统：乐观更新与去重

### 9.1 乐观更新模式

点赞/比心使用**乐观更新**——先更新 UI 再发请求，失败时回滚：

```typescript
const toggleReaction = useCallback(async (messageId, type) => {
  // 1. 防重复（reactionPending Set）
  if (reactionPending.current.has(key)) return;
  reactionPending.current.add(key);

  // 2. 立即更新 UI（乐观）
  setMessages(current => updateMessageById(current, messageId, msg => ({
    ...msg,
    [likedKey]: !wasOn,
    [countKey]: Math.max(0, msg[countKey] + (wasOn ? -1 : 1)),
  })));

  // 3. 发送 API 请求
  const result = await toggleMessageReaction(messageId, type, visitorId);

  // 4. 用服务端返回覆盖本地状态
  setMessages(current => updateMessageById(current, messageId, msg => ({
    ...msg,
    likes: result.likes,
    hearts: result.hearts,
    liked: result.liked,
    hearted: result.hearted,
  })));

  // 5. 异常时回滚到原始状态
}, []);
```

### 9.2 基于 visitorId 去重

数据库层通过 `UNIQUE (message_id, visitor_id, type)` 约束实现去重——同一个匿名用户对同一条留言的同一类型反应只能存在一条记录。切换操作是 insert/delete 语义。

---

## 10. 问题、踩坑与设计取舍

### 10.1 为什么不用地图库？

Leaflet/Mapbox 体积大、风格与像素美术不搭。自定义 SVG 渲染可以：
- 完全控制视觉风格（省份填色、标记样式）
- 体积更小（GeoJSON 数据 < 500KB）
- 更好的交互控制（点击省份 = 跳转留言墙）

代价是失去了瓦片地图的平移/缩放能力，但对于固定视角的巡演地图来说完全够用。

### 10.2 为什么手写 AWS4 签名？

AWS SDK 完整引入会增加 bundle 体积。预签名 URL 只需要 HMAC-SHA256 签名，用 Node.js `crypto` 模块即可实现，零额外依赖。

### 10.3 为什么用 React Context 而不是 Zustand？

项目只有一个页面（SPA），状态树不复杂。Context + `useCallback` 足够，且零依赖。如果后续状态膨胀再迁移也很简单。

### 10.4 性能优化

```css
/* 留言卡片虚拟渲染 —— 视口外的卡片不渲染内容 */
.msg-card {
  content-visibility: auto;
  contain-intrinsic-size: auto 180px;
}
```

```typescript
// MessageCard 使用 memo() 避免不必要重渲染
const MessageCard = memo(function MessageCard({ message, ... }) { ... });
```

图片使用 `loading="lazy"` + `decoding="async"` 延迟加载。

### 10.5 美术素材预留架构

`@chili/assets` 声明了完整的像素美术元数据（spritesheet 帧数/尺寸/FPS），`@chili/ui` 的 `drawNpc()` 当前用 Canvas 绘制 8x8 占位小人。设计为**美术交付后只需替换实现不改接口**——面向接口编程的典型实践。

### 10.6 内容审核流设计

数据库 RLS 设计为粉丝只能插入 `pending` 状态留言（需管理员审核后改为 `published`）。但 MVP 阶段 API 直接设为 `published` 简化流程，后续可随时切回审核模式。

---

## 11. 附录与参考

### 11.1 核心数据流

```
用户点击城市标记
  ↓
openCityWall(station, cityStations)
  ↓
setScreen('wall') + setCurStation(station)
  ↓
listMessages(stationId, { limit: 30, offset: 0, visitorId })
  ├─ API /api/messages → Supabase 查询 (带 RLS)
  └─ 降级 → mock.pageMessages() (localStorage)
  ↓
messages 渲染到 MessageWall
  ↓
用户点击 ❤️
  ↓
toggleReaction(messageId, 'heart')
  ├─ 乐观更新 UI (立即)
  ├─ POST /api/messages/:id/reactions
  ├─ 服务端返回 → 覆盖本地状态
  └─ 失败 → 回滚到原始状态
```

### 11.2 API 路由索引

| 方法 | 路径 | 功能 |
|------|------|------|
| GET | `/api/stations` | 全部站点 |
| GET/POST | `/api/messages` | 留言查询/发布 |
| POST | `/api/messages/:id/reactions` | 切换互动 |
| POST | `/api/uploads/presign` | R2 预签名 URL |
| GET/POST/DELETE | `/api/admin/auth` | 管理员认证 |
| GET | `/api/admin/diagnostics` | 系统诊断 |
| POST | `/api/admin/stations` | 新增站点 |
| PATCH/DELETE | `/api/admin/stations/:id` | 站点管理 |
| GET | `/api/admin/messages` | 全部留言 |
| PATCH/DELETE | `/api/admin/messages/:id` | 留言管理 |

### 11.3 关键文件索引

| 文件 | 行数 | 核心职责 |
|------|------|---------|
| `store.tsx` | 395 | 全局状态管理（路由/数据/认证/弹窗） |
| `globals.css` | 836 | CRT/地图/留言/管理全量样式 |
| `TourMap.tsx` | 353 | GeoJSON 投影 + 省份着色 + 碰撞分散 |
| `ShareModal.tsx` | 349 | Canvas 分享卡（复古外壳/伪QR/星星） |
| `Composer.tsx` | 287 | 留言撰写 + 图片上传 + 缩略图生成 |
| `MessageWall.tsx` | ~300 | 留言墙（卡片/回复/互动/分页） |
| `presign/route.ts` | 137 | 手写 AWS4 签名 |
| `schema.sql` | 102 | 数据库 DDL + RLS 策略 |
| `db/index.ts` | ~200 | API 优先 + Mock 降级数据层 |
| `db/mock.ts` | ~300 | 完整 localStorage 持久化 Mock |

---

## 免责声明

本项目为 ChiliChill 乐队巡演粉丝社区产品的技术实现记录。文中代码摘录用于技术交流学习。
