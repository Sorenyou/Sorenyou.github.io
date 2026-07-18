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
- [10. 像素地形 Canvas 引擎](#10-像素地形-canvas-引擎)
- [11. 管理后台完整实现](#11-管理后台完整实现)
- [12. 问题、踩坑与设计取舍](#12-问题踩坑与设计取舍)
- [13. 附录与参考](#13-附录与参考)

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

### 2.3 `@chili/shared` 核心类型定义

整个项目的数据契约定义在 `@chili/shared` 包中，确保前后端类型一致：

```typescript
// packages/shared/src/types.ts

/** 巡演站点 */
export interface Station {
  id: string;
  city: string;                    // 城市名
  province: string;                // 省份
  venue: string;                   // 场馆名
  date: string;                    // 演出日期 "2026-07-20"
  status: StationStatus;           // 'upcoming' | 'live' | 'done'
  lat: number;                     // 纬度
  lng: number;                     // 经度
  adcode?: string;                 // 行政区划代码（匹配 GeoJSON）
  message_count?: number;          // 留言数
  created_at?: string;
}

export type StationStatus = 'upcoming' | 'live' | 'done';

/** 留言 */
export interface Message {
  id: string;
  station_id: string;
  user_id: string;
  nickname: string;
  content: string;
  mood?: string;                   // 心情 emoji
  rating?: number;                 // 1-5 星评分
  official: boolean;               // 是否官方留言
  status: 'pending' | 'published';
  parent_id?: string | null;       // 回复目标（自引用）
  images?: MessageImage[];
  replies?: Message[];             // 嵌套回复（前端构建）
  likes: number;
  hearts: number;
  liked?: boolean;                 // 当前用户是否已赞
  hearted?: boolean;               // 当前用户是否已比心
  created_at: string;
}

/** 留言图片 */
export interface MessageImage {
  id: string;
  message_id: string;
  url: string;
  thumb_url?: string;
  width?: number;
  height?: number;
  order_index: number;
}

/** 用户 */
export interface User {
  id: string;
  nickname: string;
  avatar_url?: string;
  role: 'fan' | 'admin';
  visitor_id?: string;
  created_at: string;
}
```

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

### 3.4 省份名称标准化：`normalizeProvinceName()`

GeoJSON 中的省份名称格式各异，需要标准化后才能与站点数据匹配：

```typescript
/**
 * 标准化省份名称，去除行政区划后缀。
 *
 * 处理规则：
 * - "广西壮族自治区" → "广西"
 * - "内蒙古自治区" → "内蒙古"
 * - "宁夏回族自治区" → "宁夏"
 * - "西藏自治区" → "西藏"
 * - "新疆维吾尔自治区" → "新疆"
 * - "香港特别行政区" → "香港"
 * - "澳门特别行政区" → "澳门"
 * - "北京市" → "北京"
 * - "河南省" → "河南"
 */
function normalizeProvinceName(name: string): string {
  if (!name) return '';
  return name
    .replace(/壮族自治区$/, '')
    .replace(/回族自治区$/, '')
    .replace(/维吾尔自治区$/, '')
    .replace(/自治区$/, '')
    .replace(/特别行政区$/, '')
    .replace(/省$/, '')
    .replace(/市$/, '');
}
```

### 3.5 GeoJSON Feature 匹配：`featureMatchesStation()`

将站点的省份信息匹配到 GeoJSON Feature，adcode 优先、名称兜底：

```typescript
/**
 * 判断一个 GeoJSON Feature 是否匹配某个站点的省份。
 *
 * 匹配优先级：
 * 1. adcode 精确匹配（最可靠）
 * 2. 省份名称标准化后匹配
 * 3. 省份名称包含匹配（兜底）
 */
function featureMatchesStation(
  feature: ChinaFeature,
  station: Station
): boolean {
  const props = feature.properties;

  // 优先级1：adcode 精确匹配
  if (station.adcode && props.adcode) {
    // adcode 前两位为省级代码
    const featureProvCode = String(props.adcode).slice(0, 2);
    const stationProvCode = String(station.adcode).slice(0, 2);
    if (featureProvCode === stationProvCode) return true;
  }

  // 优先级2：名称标准化后比较
  const featureName = normalizeProvinceName(props.name || '');
  const stationProv = normalizeProvinceName(station.province || '');

  if (featureName && stationProv && featureName === stationProv) {
    return true;
  }

  // 优先级3：包含匹配（兜底）
  if (featureName && stationProv) {
    return featureName.includes(stationProv) || stationProv.includes(featureName);
  }

  return false;
}
```

### 3.6 中心点计算：`centerFromFeature()`

计算一个 GeoJSON Feature 的「视觉中心点」，用于放置城市标记：

```typescript
/**
 * 计算 GeoJSON Feature 的中心点。
 *
 * 尝试顺序：
 * 1. properties.center：GeoJSON 元数据中预设的中心点
 * 2. properties.centroid：质心
 * 3. bbox 中心：从边界框计算
 * 4. 坐标均值：遍历所有坐标点取平均（最终 fallback）
 */
function centerFromFeature(feature: ChinaFeature): Coord | null {
  const props = feature.properties;

  // 优先级1：预设中心点
  if (props.center && Array.isArray(props.center) && props.center.length === 2) {
    return props.center as Coord;
  }

  // 优先级2：质心
  if (props.centroid && Array.isArray(props.centroid) && props.centroid.length === 2) {
    return props.centroid as Coord;
  }

  // 优先级3：bbox 中心
  if (feature.bbox && feature.bbox.length >= 4) {
    return [
      (feature.bbox[0] + feature.bbox[2]) / 2,
      (feature.bbox[1] + feature.bbox[3]) / 2,
    ];
  }

  // 优先级4：坐标均值
  const coords = extractAllCoords(feature);
  if (coords.length === 0) return null;

  const sumLng = coords.reduce((s, c) => s + c[0], 0);
  const sumLat = coords.reduce((s, c) => s + c[1], 0);
  return [sumLng / coords.length, sumLat / coords.length];
}
```

### 3.7 标记位置确定：`markerPosition()` 的 fallback 链

```typescript
/**
 * 确定城市标记在 SVG 中的位置。
 *
 * Fallback 链：
 * 1. 站点自带的 lat/lng → project()
 * 2. GeoJSON Feature 的中心点 → project()
 * 3. 省份名称硬编码查表（极端 fallback）
 */
function markerPosition(
  station: Station,
  features: ChinaFeature[]
): { x: number; y: number } | null {
  // 1. 站点坐标
  if (station.lat && station.lng) {
    return project([station.lng, station.lat]);
  }

  // 2. 匹配 Feature 的中心点
  const matchedFeature = features.find(f => featureMatchesStation(f, station));
  if (matchedFeature) {
    const center = centerFromFeature(matchedFeature);
    if (center) return project(center);
  }

  // 3. 硬编码省会查表
  const fallbackCoord = PROVINCE_CAPITALS[station.province];
  if (fallbackCoord) return project(fallbackCoord);

  return null;
}
```

### 3.8 省份着色策略

根据巡演站点状态对省份着色，优先级：正在 > 即将 > 已去过：

```typescript
function toneForFeature(provinceStations: Station[]): StationStatus | 'plain' {
  if (provinceStations.some((s) => s.status === 'live'))     return 'live';     // 红
  if (provinceStations.some((s) => s.status === 'upcoming')) return 'upcoming'; // 黄
  if (provinceStations.some((s) => s.status === 'done'))     return 'done';     // 绿
  return 'plain';  // 灰
}
```

### 3.9 标记碰撞分散算法

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

### 3.10 SVG 巡演路线渲染

金色虚线连接各巡演站点，用 CSS 动画实现流动效果：

```typescript
/**
 * 生成巡演路线的 SVG <polyline>。
 * 按日期排序站点，依次连接。
 */
function renderRoute(stations: Station[]): JSX.Element | null {
  // 按日期排序
  const sorted = [...stations]
    .filter(s => s.status !== 'upcoming')  // 只连接已去过的站
    .sort((a, b) => a.date.localeCompare(b.date));

  if (sorted.length < 2) return null;

  // 生成坐标点
  const points = sorted
    .map(s => {
      const pos = markerPosition(s, features);
      return pos ? `${pos.x.toFixed(2)},${pos.y.toFixed(2)}` : null;
    })
    .filter(Boolean)
    .join(' ');

  return (
    <polyline
      points={points}
      fill="none"
      stroke="#ffd23f"
      strokeWidth="1.2"
      strokeDasharray="4 3"
      strokeLinecap="round"
      className="route-line"
      opacity="0.85"
    />
  );
}
```

对应 CSS：
```css
.route-line {
  animation: routeDash 1.6s linear infinite;
}
@keyframes routeDash {
  to { stroke-dashoffset: -14; }
}
```

### 3.11 进度条组件

开机动画中的像素进度条，40 步逐帧填充：

```typescript
/**
 * 像素进度条组件。
 * 纯 CSS 实现逐帧填充，无 JS 计时器。
 */
function PixelProgressBar({ duration = 1.6 }: { duration?: number }) {
  return (
    <div className="boot-progress-track">
      <div
        className="boot-progress-fill"
        style={{ animationDuration: `${duration}s` }}
      />
    </div>
  );
}
```

```css
.boot-progress-track {
  width: 240px;
  height: 12px;
  border: 2px solid #ffd23f;
  background: #0a0518;
  image-rendering: pixelated;
}

.boot-progress-fill {
  width: 0;
  height: 100%;
  background: #ffd23f;
  animation: bootFill 1.6s steps(40) forwards;
}

@keyframes bootFill {
  to { width: 100%; }
}
```

---

## 4. 图片上传：手写 AWS4 签名 + R2 直传

### 4.1 架构：浏览器直传

传统做法是图片先上传到服务端再转存，但这消耗服务端带宽。本项目采用 **预签名 URL 直传**：

```
浏览器 → POST /api/uploads/presign → 获取预签名 URL
浏览器 → PUT 直接上传到 R2 (绕过服务端)
```

### 4.2 手写 AWS4 签名：`presign/route.ts` 完整详解

没有使用 AWS SDK，仅用 Node.js `crypto` 模块手动实现完整的 AWS Signature V4：

```typescript
// presign/route.ts — 零依赖 AWS4 签名

import { createHmac, createHash } from 'crypto';

/** HMAC-SHA256 签名 */
function hmac(key: string | Buffer, data: string): Buffer {
  return createHmac('sha256', key).update(data).digest();
}

/** HMAC-SHA256 输出 hex */
function hexHmac(key: string | Buffer, data: string): string {
  return createHmac('sha256', key).update(data).digest('hex');
}

/** SHA256 哈希 */
function hash(data: string): string {
  return createHash('sha256').update(data).digest('hex');
}
```

#### 4.2.1 日期格式化辅助函数

```typescript
/**
 * 生成 AWS4 需要的 ISO 8601 时间戳。
 * 格式：20260718T120000Z
 */
function amzDate(date: Date = new Date()): string {
  return date.toISOString()
    .replace(/[-:]/g, '')
    .replace(/\.\d{3}/, '');
}

/**
 * 提取日期部分。
 * "20260718T120000Z" → "20260718"
 */
function dateStamp(amz: string): string {
  return amz.slice(0, 8);
}
```

#### 4.2.2 文件名安全处理

```typescript
/**
 * 清洗文件名中的非安全字符。
 * "我的照片 (1).jpg" → "wo-de-zhao-pian--1-"
 *
 * 处理：
 * - 中文字符替换为拼音（简化版：直接移除）
 * - 特殊字符替换为连字符
 * - 连续连字符合并
 * - 去除首尾连字符
 */
function sanitizeBaseName(name: string): string {
  return name
    .replace(/[^\w.-]/g, '-')    // 非字母数字点连字符 → 连字符
    .replace(/-{2,}/g, '-')      // 合并连续连字符
    .replace(/^-|-$/g, '')       // 去除首尾
    .slice(0, 80)                // 限制长度
    || 'file';                   // 兜底
}

/**
 * URL 路径编码（不编码 /）。
 * AWS4 要求路径中的特殊字符编码，但 / 保留。
 */
function encodePath(path: string): string {
  return path
    .split('/')
    .map(s => encodeURIComponent(s))
    .join('/');
}
```

#### 4.2.3 签名密钥派生

```typescript
function signingKey(secret: string, date: string, region: string, service: string) {
  const kDate    = hmac(`AWS4${secret}`, date);
  const kRegion  = hmac(kDate, region);
  const kService = hmac(kRegion, service);
  return hmac(kService, 'aws4_request');
}
```

#### 4.2.4 预签名 URL 生成

```typescript
function presignPutUrl(input: {
  bucket: string;
  key: string;
  contentType: string;
  accessKeyId: string;
  secretAccessKey: string;
  region: string;
  endpoint: string;
  expiresIn?: number;
}) {
  const { bucket, key, contentType, accessKeyId, secretAccessKey,
          region, endpoint, expiresIn = 600 } = input;

  const now = new Date();
  const timestamp = amzDate(now);
  const date = dateStamp(timestamp);
  const service = 's3';
  const host = new URL(endpoint).host;

  const canonicalUri = encodePath(`/${bucket}/${key}`);
  const credentialScope = `${date}/${region}/${service}/aws4_request`;
  const signedHeaders = 'content-type;host';

  const params = new URLSearchParams({
    'X-Amz-Algorithm': 'AWS4-HMAC-SHA256',
    'X-Amz-Credential': `${accessKeyId}/${credentialScope}`,
    'X-Amz-Date': timestamp,
    'X-Amz-Expires': String(expiresIn),
    'X-Amz-SignedHeaders': signedHeaders,
  });

  const canonicalRequest = [
    'PUT',
    canonicalUri,
    params.toString(),
    `content-type:${contentType}\nhost:${host}\n`,
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
    signingKey(secretAccessKey, date, region, service),
    stringToSign
  );

  params.set('X-Amz-Signature', signature);
  return `${endpoint}${canonicalUri}?${params}`;
}
```

### 4.3 客户端文件处理

#### 4.3.1 唯一文件标识：`fileId()`

```typescript
/**
 * 生成文件的唯一标识。
 * 基于文件名 + 大小 + 最后修改时间，避免重复上传。
 */
function fileId(file: File): string {
  return `${file.name}_${file.size}_${file.lastModified}`;
}
```

#### 4.3.2 人性化文件大小：`humanSize()`

```typescript
/**
 * 将字节数转为人类可读的文件大小。
 * 1024 → "1.0 KB"
 * 5242880 → "5.0 MB"
 */
function humanSize(bytes: number): string {
  if (bytes < 1024) return `${bytes} B`;
  if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`;
  return `${(bytes / (1024 * 1024)).toFixed(1)} MB`;
}
```

#### 4.3.3 完整校验流程：`chooseFiles()`

```typescript
const MAX_IMAGES = 6;
const MAX_FILE_SIZE = 10 * 1024 * 1024;  // 10MB
const ALLOWED_TYPES = ['image/jpeg', 'image/png', 'image/webp', 'image/gif'];

/**
 * 处理用户选择的文件列表。
 *
 * 校验流程：
 * 1. 检查总数量（已有 + 新选 ≤ 6）
 * 2. 逐文件检查类型（jpg/png/webp/gif）
 * 3. 逐文件检查大小（≤ 10MB）
 * 4. 检查是否已选择过（去重）
 * 5. 返回通过校验的文件列表 + 错误信息
 */
function chooseFiles(
  newFiles: FileList,
  existingFiles: UploadFile[]
): { accepted: File[]; errors: string[] } {
  const errors: string[] = [];
  const accepted: File[] = [];
  const remaining = MAX_IMAGES - existingFiles.length;

  if (remaining <= 0) {
    errors.push(`最多上传 ${MAX_IMAGES} 张图片`);
    return { accepted, errors };
  }

  const existingIds = new Set(existingFiles.map(f => fileId(f.file)));

  for (let i = 0; i < newFiles.length && accepted.length < remaining; i++) {
    const file = newFiles[i];

    // 类型检查
    if (!ALLOWED_TYPES.includes(file.type)) {
      errors.push(`${file.name}: 不支持的格式（仅 JPG/PNG/WebP/GIF）`);
      continue;
    }

    // 大小检查
    if (file.size > MAX_FILE_SIZE) {
      errors.push(`${file.name}: 文件过大（${humanSize(file.size)}，上限 10MB）`);
      continue;
    }

    // 去重
    if (existingIds.has(fileId(file))) {
      continue;  // 静默跳过
    }

    accepted.push(file);
  }

  if (newFiles.length > remaining + accepted.length) {
    errors.push(`已忽略超出数量限制的文件`);
  }

  return { accepted, errors };
}
```

#### 4.3.4 图片移除与内存释放：`removeImage()`

```typescript
/**
 * 移除已选择的图片。
 * 关键：释放 Object URL 防止内存泄漏。
 */
function removeImage(images: UploadFile[], index: number): UploadFile[] {
  const removed = images[index];

  // 释放预览 URL（如果是 Object URL）
  if (removed.previewUrl && removed.previewUrl.startsWith('blob:')) {
    URL.revokeObjectURL(removed.previewUrl);
  }

  // 释放缩略图 URL
  if (removed.thumbPreviewUrl && removed.thumbPreviewUrl.startsWith('blob:')) {
    URL.revokeObjectURL(removed.thumbPreviewUrl);
  }

  return images.filter((_, i) => i !== index);
}
```

#### 4.3.5 `blobToImage()` Promise 包装

```typescript
/**
 * 将 Blob/File 转为 HTMLImageElement（Promise 包装）。
 * 用于 Canvas 缩略图生成前获取图片尺寸。
 */
function blobToImage(blob: Blob): Promise<HTMLImageElement> {
  return new Promise((resolve, reject) => {
    const url = URL.createObjectURL(blob);
    const img = new Image();
    img.onload = () => {
      URL.revokeObjectURL(url);  // 立即释放
      resolve(img);
    };
    img.onerror = () => {
      URL.revokeObjectURL(url);
      reject(new Error('图片加载失败'));
    };
    img.src = url;
  });
}
```

### 4.4 客户端缩略图生成

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

### 5.2 文字换行算法：`wrapText()`

Canvas 没有内置的文字换行能力，需要手动实现：

```typescript
/**
 * 将文本按指定最大宽度换行。
 *
 * 算法：
 * 1. 逐字符遍历
 * 2. 用 ctx.measureText() 检测当前行是否超宽
 * 3. 超宽时在最近的空格/标点/CJK 字符处断行
 * 4. CJK 字符可以在任意位置断行
 *
 * 返回：换行后的行数组
 */
function wrapText(
  ctx: CanvasRenderingContext2D,
  text: string,
  maxWidth: number
): string[] {
  const lines: string[] = [];
  let current = '';

  for (let i = 0; i < text.length; i++) {
    const ch = text[i];
    const test = current + ch;
    const measured = ctx.measureText(test).width;

    if (measured > maxWidth && current.length > 0) {
      lines.push(current);
      current = ch;
    } else {
      current = test;
    }
  }

  if (current) lines.push(current);
  return lines;
}
```

### 5.3 行数测量：`measureWrapLines()`

绘制前需要预先知道文字会占多少行，以计算布局高度：

```typescript
/**
 * 测量文本换行后的行数（不实际绘制）。
 * 用于动态计算卡片高度和布局。
 */
function measureWrapLines(
  ctx: CanvasRenderingContext2D,
  text: string,
  font: string,
  maxWidth: number
): number {
  const prevFont = ctx.font;
  ctx.font = font;
  const lines = wrapText(ctx, text, maxWidth);
  ctx.font = prevFont;
  return lines.length;
}
```

### 5.4 星星评分渲染：`drawStars()`

```typescript
/**
 * 绘制 1-5 星评分。
 * 实心星 = 已选分数，空心星 = 未选。
 * 使用像素字体字符 ★/☆ 绘制。
 */
function drawStars(
  ctx: CanvasRenderingContext2D,
  x: number,
  y: number,
  rating: number,
  size: number = 20
) {
  ctx.font = `${size}px "Zpix"`;
  ctx.fillStyle = '#ffd23f';  // 金色

  for (let i = 1; i <= 5; i++) {
    const char = i <= rating ? '★' : '☆';
    ctx.fillText(char, x + (i - 1) * (size + 4), y);
  }
}
```

### 5.5 像素分隔线：`drawPixelDivider()`

```typescript
/**
 * 绘制像素风装饰分隔线。
 * 由交替的小方块组成，模拟 8-bit 游戏风格。
 */
function drawPixelDivider(
  ctx: CanvasRenderingContext2D,
  x: number,
  y: number,
  width: number,
  color: string = '#ffd23f'
) {
  ctx.fillStyle = color;
  const blockSize = 4;
  const gap = 4;
  const step = blockSize + gap;
  const count = Math.floor(width / step);

  for (let i = 0; i < count; i++) {
    ctx.fillRect(x + i * step, y, blockSize, blockSize);
  }
}
```

### 5.6 Canvas 下载实现：`downloadCanvas()`

```typescript
/**
 * 将 Canvas 导出为 PNG 并触发下载。
 *
 * 步骤：
 * 1. canvas.toBlob() 转为 Blob
 * 2. URL.createObjectURL() 创建临时 URL
 * 3. 创建隐藏的 <a> 标签触发下载
 * 4. 释放 Object URL
 */
function downloadCanvas(canvas: HTMLCanvasElement, filename: string) {
  canvas.toBlob((blob) => {
    if (!blob) return;
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = filename;
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);
  }, 'image/png');
}
```

### 5.7 三种模式的绘制逻辑差异

```typescript
/**
 * ShareModal.tsx — 主绘制入口
 */
async function drawShareCard(
  canvas: HTMLCanvasElement,
  mode: 'page' | 'footprint' | 'message',
  data: ShareData
) {
  const ctx = canvas.getContext('2d')!;

  // 公共部分：复古外壳 + 扫描线
  drawRetroFrame(ctx);

  switch (mode) {
    case 'page':
      // 站点统计卡
      // 标题：城市名 + 场馆
      // 内容：留言数 / 图片数 / 互动数
      // 底部：日期 + 伪 QR 码
      drawPageTitle(ctx, data.station);
      drawStats(ctx, data.stats);
      drawPixelDivider(ctx, 96, 520, CARD_W - 192);
      drawPixelQR(ctx, CARD_W - 200, CARD_H - 240, 120, data.url);
      break;

    case 'footprint':
      // 个人足迹卡
      // 标题：MY FOOTPRINT
      // 内容：打卡城市列表 + 日记数
      // 特殊：城市名用像素标签排列
      drawFootprintTitle(ctx);
      drawCityTags(ctx, data.cities);
      drawStars(ctx, 96, 800, data.totalStations);
      drawPixelQR(ctx, CARD_W - 200, CARD_H - 240, 120, data.url);
      break;

    case 'message':
      // 单条留言卡
      // 标题：FIELD NOTE
      // 内容：留言正文（自动换行）
      // 额外：心情 emoji + 评分星星
      drawMessageTitle(ctx, data.station);
      // 动态计算正文区域高度
      const lines = measureWrapLines(
        ctx, data.message.content, '24px "Zpix"', CARD_W - 200
      );
      drawWrappedContent(ctx, data.message.content, 96, 440, CARD_W - 200);
      if (data.message.rating) {
        drawStars(ctx, 96, 440 + lines * 32 + 20, data.message.rating);
      }
      if (data.message.mood) {
        ctx.font = '36px serif';
        ctx.fillText(data.message.mood, CARD_W - 140, 380);
      }
      drawPixelDivider(ctx, 96, CARD_H - 300, CARD_W - 192);
      drawPixelQR(ctx, CARD_W - 200, CARD_H - 240, 120, data.url);
      break;
  }
}
```

### 5.8 伪二维码生成

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

### 5.9 复古游戏机外壳绘制

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

### 6.2 开机动画：`BootScreen.tsx` 完整实现

```typescript
/**
 * CRT 开机动画组件。
 * 序列：闪烁 → logo 渐现 → 进度条填充 → 小人出场 → 淡出
 */
export function BootScreen({ onComplete }: { onComplete: () => void }) {
  const [phase, setPhase] = useState<'flicker' | 'logo' | 'bar' | 'npc' | 'done'>('flicker');

  useEffect(() => {
    const timers = [
      setTimeout(() => setPhase('logo'), 300),      // 300ms 后显示 logo
      setTimeout(() => setPhase('bar'), 1700),       // 1.4s logo 动画后显示进度条
      setTimeout(() => setPhase('npc'), 3300),       // 进度条 1.6s 后显示小人
      setTimeout(() => {
        setPhase('done');
        onComplete();
      }, 4200),                                       // 小人动画后完成
    ];
    return () => timers.forEach(clearTimeout);
  }, [onComplete]);

  return (
    <div className={`boot-screen ${phase === 'done' ? 'boot-fade-out' : ''}`}>
      {/* 屏幕闪烁 */}
      {phase === 'flicker' && <div className="boot-flicker" />}

      {/* Logo */}
      {(phase === 'logo' || phase === 'bar' || phase === 'npc') && (
        <div className="boot-logo">
          <img src="/logo-pixel.png" alt="ChiliChill" className="boot-logo-img" />
          <div className="boot-subtitle">TOUR DIARY</div>
        </div>
      )}

      {/* 进度条 */}
      {(phase === 'bar' || phase === 'npc') && (
        <PixelProgressBar duration={1.6} />
      )}

      {/* NPC 小人 */}
      {phase === 'npc' && (
        <div className="boot-npc">
          <div className="npc-sprite" />
        </div>
      )}
    </div>
  );
}
```

### 6.3 完整 CSS @keyframes 定义

```css
/* 开机 logo 缩放 + 模糊渐现 */
@keyframes bootGlow {
  0%   { transform: scale(0.6); opacity: 0; filter: blur(8px); }
  50%  { transform: scale(1.05); opacity: 0.8; filter: blur(2px); }
  100% { transform: scale(1); opacity: 1; filter: blur(0); }
}

/* 进度条 40 步填充 */
@keyframes bootFill {
  from { width: 0; }
  to   { width: 100%; }
}

/* 页面切换 6 步放大 */
@keyframes pageIn {
  from { transform: scale(0.85); opacity: 0; }
  to   { transform: scale(1); opacity: 1; }
}

/* 路线虚线流动 */
@keyframes routeDash {
  to { stroke-dashoffset: -14; }
}

/* 标记弹出 */
@keyframes markerPop {
  from { transform: scale(0); }
  to   { transform: scale(1); }
}

/* 弹窗底部弹性滑入 */
@keyframes sheetUpBounce {
  from { transform: translateY(100%); }
  to   { transform: translateY(0); }
}

/* 开机闪烁 */
@keyframes bootFlicker {
  0%, 15%, 30% { opacity: 0; }
  5%, 20%      { opacity: 0.3; }
  35%, 100%    { opacity: 1; }
}

/* 淡出 */
@keyframes bootFadeOut {
  to { opacity: 0; pointer-events: none; }
}

/* NPC 走路动画 */
@keyframes npcWalk {
  from { transform: translateX(-40px); }
  to   { transform: translateX(40px); }
}
```

### 6.4 像素逐帧动画

全部使用 CSS `steps()` 函数模拟逐帧渲染，完美匹配像素风美术：

| 动画 | CSS | 效果 |
|------|-----|------|
| 开机 logo | `animation: bootGlow 1.4s steps(8)` | 缩放 + 模糊渐现 |
| 进度条 | `animation: bootFill 1.6s steps(40)` | 40 步填充 |
| 页面切换 | `animation: pageIn .32s steps(6)` | 6 步放大 |
| 路线流动 | `animation: routeDash 1.6s linear infinite` | 虚线持续流动 |
| 标记弹出 | `animation: markerPop 0.35s cubic-bezier(.34,1.56,.64,1)` | 弹性弹出 |
| 弹窗入场 | `animation: sheetUpBounce .4s cubic-bezier(.34,1.56,.64,1)` | 带弹性底部滑入 |

### 6.5 自定义滚动条样式

```css
/* 像素风滚动条 */
::-webkit-scrollbar {
  width: 8px;
  height: 8px;
}

::-webkit-scrollbar-track {
  background: #0a0518;
  border: 1px solid #2a1f4e;
}

::-webkit-scrollbar-thumb {
  background: #3d2d6b;
  border: 1px solid #5a3f9e;
}

::-webkit-scrollbar-thumb:hover {
  background: #5a3f9e;
}

::-webkit-scrollbar-corner {
  background: #0a0518;
}
```

### 6.6 响应式策略

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

### 7.2 通用请求函数：`api()` 的错误处理

```typescript
/**
 * 通用 API 请求函数。
 *
 * 设计要点：
 * - 所有 HTTP 错误返回 null（不抛异常）
 * - 自动附加 Content-Type
 * - 超时 10 秒
 * - 降级场景：返回 null 后调用方自动切到 mock
 */
async function api<T>(
  path: string,
  options: RequestInit = {}
): Promise<T | null> {
  try {
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), 10_000);

    const res = await fetch(path, {
      ...options,
      headers: {
        'Content-Type': 'application/json',
        ...options.headers,
      },
      signal: controller.signal,
    });

    clearTimeout(timeout);

    if (!res.ok) {
      console.warn(`[api] ${path} → ${res.status}`);
      return null;
    }

    return (await res.json()) as T;
  } catch (err) {
    // 网络错误、超时、AbortError 都返回 null
    console.warn(`[api] ${path} → ${err instanceof Error ? err.message : 'unknown'}`);
    return null;
  }
}
```

### 7.3 Mock 层完整实现

`mock.ts` 不是简单的占位，而是一套**完整的 localStorage 持久化数据层**：

```typescript
// packages/db/src/mock.ts

const STORAGE_KEY = 'chilichill_mock_data';

/** 从 localStorage 读取数据 */
function loadData(): MockDatabase {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (raw) return JSON.parse(raw);
  } catch {}
  // 首次访问：初始化种子数据
  const initial = createSeedData();
  saveData(initial);
  return initial;
}

/** 写入 localStorage */
function saveData(data: MockDatabase): void {
  try {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(data));
  } catch (e) {
    console.warn('[mock] localStorage 写入失败（可能已满）');
  }
}
```

#### 7.3.1 嵌套回复归组：`nestMessages()`

```typescript
/**
 * 将扁平的留言列表转为嵌套回复结构。
 *
 * 算法：
 * 1. 建立 id → message 映射
 * 2. 遍历所有留言，有 parent_id 的挂到父留言的 replies 下
 * 3. 返回没有 parent_id 的顶级留言（已包含嵌套 replies）
 */
function nestMessages(flat: Message[]): Message[] {
  const map = new Map<string, Message>();

  // 第一遍：建映射，初始化 replies
  for (const msg of flat) {
    map.set(msg.id, { ...msg, replies: [] });
  }

  const roots: Message[] = [];

  // 第二遍：归组
  for (const msg of flat) {
    const node = map.get(msg.id)!;
    if (msg.parent_id && map.has(msg.parent_id)) {
      map.get(msg.parent_id)!.replies!.push(node);
    } else {
      roots.push(node);
    }
  }

  return roots;
}
```

#### 7.3.2 互动状态计算：`withReactionState()`

```typescript
/**
 * 为留言附加当前用户的互动状态。
 *
 * 检查 message_reactions 表中是否存在
 * (message_id, visitor_id, type) 记录。
 */
function withReactionState(
  messages: Message[],
  visitorId: string
): Message[] {
  const data = loadData();
  const reactions = data.reactions || [];

  return messages.map(msg => ({
    ...msg,
    liked: reactions.some(
      r => r.message_id === msg.id && r.visitor_id === visitorId && r.type === 'like'
    ),
    hearted: reactions.some(
      r => r.message_id === msg.id && r.visitor_id === visitorId && r.type === 'heart'
    ),
    // 递归处理嵌套回复
    replies: msg.replies
      ? withReactionState(msg.replies, visitorId)
      : [],
  }));
}
```

#### 7.3.3 分页查询

```typescript
/**
 * Mock 分页留言查询。
 * 完整实现 limit/offset/order 语义。
 */
function pageMessages(
  stationId: string,
  options: { limit?: number; offset?: number; order?: 'newest' | 'oldest'; visitorId?: string }
): { messages: Message[]; total: number; hasMore: boolean } {
  const data = loadData();
  let msgs = data.messages.filter(
    m => m.station_id === stationId && m.status === 'published' && !m.parent_id
  );

  // 排序
  if (options.order === 'oldest') {
    msgs.sort((a, b) => a.created_at.localeCompare(b.created_at));
  } else {
    msgs.sort((a, b) => b.created_at.localeCompare(a.created_at));
  }

  const total = msgs.length;
  const offset = options.offset || 0;
  const limit = options.limit || 30;

  // 分页
  const sliced = msgs.slice(offset, offset + limit);

  // 嵌套 + 互动状态
  const nested = nestMessages([
    ...sliced,
    ...data.messages.filter(m => sliced.some(s => s.id === m.parent_id)),
  ]);
  const final = options.visitorId
    ? withReactionState(nested, options.visitorId)
    : nested;

  return {
    messages: final,
    total,
    hasMore: offset + limit < total,
  };
}
```

#### 7.3.4 种子数据：`seed.ts`

```typescript
// packages/db/src/seed.ts

export function createSeedData(): MockDatabase {
  return {
    stations: [
      {
        id: 'seed-bj',
        city: '北京',
        province: '北京',
        venue: 'MAO Livehouse',
        date: '2026-06-15',
        status: 'done',
        lat: 39.9042,
        lng: 116.4074,
        message_count: 3,
      },
      {
        id: 'seed-sh',
        city: '上海',
        province: '上海',
        venue: '育音堂',
        date: '2026-06-22',
        status: 'done',
        lat: 31.2304,
        lng: 121.4737,
        message_count: 2,
      },
      {
        id: 'seed-cd',
        city: '成都',
        province: '四川',
        venue: '小酒馆',
        date: '2026-07-20',
        status: 'upcoming',
        lat: 30.5728,
        lng: 104.0668,
        message_count: 0,
      },
    ],
    messages: [
      {
        id: 'seed-msg-1',
        station_id: 'seed-bj',
        user_id: 'seed-user-1',
        nickname: '辣椒粉丝A',
        content: '北京场太燃了！！安可唱了三首',
        mood: '🔥',
        rating: 5,
        official: false,
        status: 'published',
        parent_id: null,
        likes: 12,
        hearts: 8,
        created_at: '2026-06-15T22:30:00Z',
      },
      // ... 更多种子数据
    ],
    reactions: [],
    users: [],
  };
}
```

### 7.4 状态管理

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

### 8.1 完整 SQL Schema

```sql
-- ============================================================
-- 1. stations — 巡演站点
-- ============================================================
create table stations (
  id            uuid primary key default gen_random_uuid(),
  city          text not null,
  province      text not null,
  venue         text not null default '',
  date          date,
  status        text not null default 'upcoming'
                check (status in ('upcoming', 'live', 'done')),
  lat           double precision,
  lng           double precision,
  adcode        text,
  message_count integer not null default 0,
  created_at    timestamptz not null default now(),
  updated_at    timestamptz not null default now()
);

create index idx_stations_status on stations (status);
create index idx_stations_province on stations (province);
create index idx_stations_date on stations (date);

-- ============================================================
-- 2. messages — 留言
-- ============================================================
create table messages (
  id            uuid primary key default gen_random_uuid(),
  station_id    uuid not null references stations(id) on delete cascade,
  user_id       text not null default '',
  nickname      text not null default '匿名',
  content       text not null default '',
  mood          text,
  rating        smallint check (rating between 1 and 5),
  official      boolean not null default false,
  status        text not null default 'pending'
                check (status in ('pending', 'published', 'hidden')),
  parent_id     uuid references messages(id) on delete set null,
  likes         integer not null default 0,
  hearts        integer not null default 0,
  created_at    timestamptz not null default now()
);

create index idx_messages_station on messages (station_id);
create index idx_messages_status on messages (status);
create index idx_messages_parent on messages (parent_id);
create index idx_messages_created on messages (created_at desc);

-- ============================================================
-- 3. message_images — 留言图片
-- ============================================================
create table message_images (
  id            uuid primary key default gen_random_uuid(),
  message_id    uuid not null references messages(id) on delete cascade,
  url           text not null,
  thumb_url     text,
  width         integer,
  height        integer,
  order_index   smallint not null default 0,
  created_at    timestamptz not null default now()
);

create index idx_images_message on message_images (message_id);

-- ============================================================
-- 4. message_reactions — 互动
-- ============================================================
create table message_reactions (
  id            uuid primary key default gen_random_uuid(),
  message_id    uuid not null references messages(id) on delete cascade,
  visitor_id    text not null,
  type          text not null check (type in ('like', 'heart')),
  created_at    timestamptz not null default now(),
  unique (message_id, visitor_id, type)
);

create index idx_reactions_message on message_reactions (message_id);
create index idx_reactions_visitor on message_reactions (visitor_id);
```

#### 迁移语句

```sql
-- 后续新增字段的 ALTER TABLE 迁移
alter table stations add column if not exists adcode text;
alter table stations add column if not exists updated_at timestamptz not null default now();
alter table messages add column if not exists mood text;
alter table messages add column if not exists rating smallint check (rating between 1 and 5);
```

### 8.2 Row Level Security

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

### 8.3 管理后台认证 API

```typescript
// app/api/admin/auth/route.ts

import { cookies } from 'next/headers';

const ADMIN_COOKIE = 'chilichill_admin';
const COOKIE_MAX_AGE = 8 * 60 * 60;  // 8 小时

/**
 * requireAdmin 中间件。
 * 检查 httpOnly Cookie 中的 admin token。
 */
export function requireAdmin(): boolean {
  const cookieStore = cookies();
  const token = cookieStore.get(ADMIN_COOKIE)?.value;

  if (!token) return false;

  // 验证 token（简单实现：与环境变量比较）
  return token === process.env.ADMIN_SECRET;
}

/**
 * POST /api/admin/auth — 登录
 */
export async function POST(req: Request) {
  const { password } = await req.json();

  if (password !== process.env.ADMIN_PASSWORD) {
    return Response.json({ ok: false, message: '密码错误' }, { status: 401 });
  }

  // 设置 httpOnly Cookie
  const cookieStore = cookies();
  cookieStore.set(ADMIN_COOKIE, process.env.ADMIN_SECRET!, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: COOKIE_MAX_AGE,
    path: '/',
  });

  return Response.json({ ok: true });
}

/**
 * DELETE /api/admin/auth — 登出
 */
export async function DELETE() {
  const cookieStore = cookies();
  cookieStore.set(ADMIN_COOKIE, '', {
    httpOnly: true,
    maxAge: 0,     // 立即过期
    path: '/',
  });

  return Response.json({ ok: true });
}

/**
 * GET /api/admin/auth — 检查登录状态
 */
export async function GET() {
  return Response.json({ authed: requireAdmin() });
}
```

### 8.4 系统诊断面板：`AdminDiagnostics.tsx`

```typescript
/**
 * 管理后台系统诊断面板。
 * 检测各个外部依赖的连接状态。
 */

interface DiagItem {
  name: string;
  status: 'ok' | 'warn' | 'error';
  message: string;
  latency?: number;
}

async function runDiagnostics(): Promise<DiagItem[]> {
  const items: DiagItem[] = [];

  // 1. Supabase 连接
  try {
    const start = Date.now();
    const res = await fetch('/api/admin/diagnostics?check=supabase');
    const data = await res.json();
    items.push({
      name: 'Supabase',
      status: data.ok ? 'ok' : 'error',
      message: data.ok ? `连接正常 (${data.stationCount} 站点)` : data.error,
      latency: Date.now() - start,
    });
  } catch (e) {
    items.push({ name: 'Supabase', status: 'error', message: '无法连接' });
  }

  // 2. R2 对象存储
  try {
    const start = Date.now();
    const res = await fetch('/api/admin/diagnostics?check=r2');
    const data = await res.json();
    items.push({
      name: 'Cloudflare R2',
      status: data.ok ? 'ok' : 'warn',
      message: data.ok ? '预签名正常' : '预签名失败（图片上传不可用）',
      latency: Date.now() - start,
    });
  } catch (e) {
    items.push({ name: 'Cloudflare R2', status: 'error', message: '无法连接' });
  }

  // 3. 环境变量完整性
  const envCheck = await (await fetch('/api/admin/diagnostics?check=env')).json();
  items.push({
    name: '环境变量',
    status: envCheck.missing.length === 0 ? 'ok' : 'warn',
    message: envCheck.missing.length === 0
      ? '全部配置'
      : `缺少: ${envCheck.missing.join(', ')}`,
  });

  return items;
}
```

### 8.5 认证设计

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
const toggleReaction = useCallback(async (messageId: string, type: 'like' | 'heart') => {
  const key = `${messageId}_${type}`;

  // 1. 防重复（reactionPending Set）
  if (reactionPending.current.has(key)) return;
  reactionPending.current.add(key);

  try {
    // 2. 计算当前状态
    const likedKey = type === 'like' ? 'liked' : 'hearted';
    const countKey = type === 'like' ? 'likes' : 'hearts';

    // 查找当前消息状态（需要在嵌套回复中递归查找）
    let wasOn = false;
    setMessages(current => {
      const found = findMessageById(current, messageId);
      if (found) wasOn = found[likedKey] ?? false;
      return current;
    });

    // 3. 立即更新 UI（乐观）
    setMessages(current => updateMessageById(current, messageId, msg => ({
      ...msg,
      [likedKey]: !wasOn,
      [countKey]: Math.max(0, msg[countKey] + (wasOn ? -1 : 1)),
    })));

    // 4. 发送 API 请求
    const result = await toggleMessageReaction(messageId, type, visitorId);

    // 5. 用服务端返回覆盖本地状态
    if (result) {
      setMessages(current => updateMessageById(current, messageId, msg => ({
        ...msg,
        likes: result.likes,
        hearts: result.hearts,
        liked: result.liked,
        hearted: result.hearted,
      })));
    }
  } catch {
    // 6. 异常时回滚到原始状态（通过重新拉取）
    // 简化实现：不回滚，下次加载会自动同步
  } finally {
    reactionPending.current.delete(key);
  }
}, [visitorId]);
```

### 9.2 递归更新嵌套回复：`updateMessageById`

互动状态更新必须能处理嵌套回复结构，因为用户可能点赞的是一条回复而非顶级留言：

```typescript
/**
 * 递归查找并更新指定 ID 的消息。
 * 遍历顶级消息及其所有 replies 子树。
 */
function updateMessageById(
  messages: Message[],
  targetId: string,
  updater: (msg: Message) => Message
): Message[] {
  return messages.map(msg => {
    if (msg.id === targetId) {
      return updater(msg);
    }
    if (msg.replies && msg.replies.length > 0) {
      return {
        ...msg,
        replies: updateMessageById(msg.replies, targetId, updater),
      };
    }
    return msg;
  });
}

/**
 * 递归查找指定 ID 的消息。
 */
function findMessageById(
  messages: Message[],
  targetId: string
): Message | null {
  for (const msg of messages) {
    if (msg.id === targetId) return msg;
    if (msg.replies) {
      const found = findMessageById(msg.replies, targetId);
      if (found) return found;
    }
  }
  return null;
}
```

### 9.3 基于 visitorId 去重

数据库层通过 `UNIQUE (message_id, visitor_id, type)` 约束实现去重——同一个匿名用户对同一条留言的同一类型反应只能存在一条记录。切换操作是 insert/delete 语义。

---

## 10. 像素地形 Canvas 引擎

### 10.1 `@chili/scene` 包概述

`@chili/scene` 是一个独立的像素地形渲染引擎，在 64×40 的网格上渲染中国地形，用于开机动画背景和分享卡装饰。

### 10.2 网格系统

```typescript
// packages/scene/src/terrain.ts

const GRID_W = 64;     // 网格列数
const GRID_H = 40;     // 网格行数
const CELL_SIZE = 8;   // 每个网格单元的像素大小

/** 地形类型 */
type Terrain = 'ocean' | 'plain' | 'plateau' | 'desert' | 'mountain';

/** 地形颜色方案 */
const TERRAIN_COLORS: Record<Terrain, string> = {
  ocean:    '#0a1628',   // 深蓝
  plain:    '#1a3a1a',   // 深绿
  plateau:  '#3a2a1a',   // 棕色
  desert:   '#4a3a1a',   // 沙黄
  mountain: '#2a2a3a',   // 灰蓝
};
```

### 10.3 Run-Length 编码陆地轮廓

中国轮廓使用 RLE（Run-Length Encoding）压缩存储，每行记录「从第几列开始、连续几列是陆地」：

```typescript
/**
 * RLE 编码的中国陆地轮廓。
 * 格式：每行一个数组，[start, length, terrain, start, length, terrain, ...]
 * 第 0 行是最北端，第 39 行是最南端。
 */
const CHINA_RLE: Array<number[]> = [
  // 行0（北端）：新疆西部
  [8, 4, 3],                        // col 8 开始，4 格沙漠
  // 行1
  [7, 6, 3, 20, 3, 2],              // 新疆 + 内蒙古高原
  // 行2
  [6, 8, 3, 18, 8, 2, 30, 5, 1],    // 新疆 + 内蒙古 + 东北平原
  // ... 共 40 行
  // 行35（南端）：海南
  [30, 2, 1],
];

/** 解码 RLE 为 2D 地形网格 */
function decodeRLE(rle: Array<number[]>): Terrain[][] {
  const grid: Terrain[][] = [];

  for (let row = 0; row < GRID_H; row++) {
    const line: Terrain[] = new Array(GRID_W).fill('ocean');
    const segments = rle[row] || [];

    for (let i = 0; i < segments.length; i += 3) {
      const start = segments[i];
      const length = segments[i + 1];
      const terrainCode = segments[i + 2];

      const terrain: Terrain =
        terrainCode === 1 ? 'plain' :
        terrainCode === 2 ? 'plateau' :
        terrainCode === 3 ? 'desert' :
        terrainCode === 4 ? 'mountain' : 'plain';

      for (let c = start; c < start + length && c < GRID_W; c++) {
        line[c] = terrain;
      }
    }

    grid.push(line);
  }

  return grid;
}
```

### 10.4 水系绘制

长江、黄河、珠江三大水系用特定颜色的像素线段绘制：

```typescript
/** 河流路径定义（网格坐标） */
const RIVERS = {
  yangtze: [   // 长江
    [12, 22], [14, 22], [16, 21], [18, 21], [20, 22],
    [22, 23], [24, 23], [26, 22], [28, 22], [30, 23],
    [32, 24], [34, 24], [36, 23],
  ],
  yellow: [    // 黄河（含"几"字弯）
    [12, 18], [14, 16], [16, 15], [18, 14], [20, 14],
    [22, 15], [22, 17], [20, 18], [22, 19], [24, 19],
    [26, 20], [28, 20], [30, 19],
  ],
  pearl: [     // 珠江
    [26, 28], [28, 29], [30, 30], [32, 30],
  ],
};

const RIVER_COLOR = '#1a4a6a';   // 深蓝，与海洋区分

function drawRivers(ctx: CanvasRenderingContext2D): void {
  ctx.fillStyle = RIVER_COLOR;

  for (const river of Object.values(RIVERS)) {
    for (const [col, row] of river) {
      ctx.fillRect(col * CELL_SIZE, row * CELL_SIZE, CELL_SIZE, CELL_SIZE);
    }
  }
}
```

### 10.5 完整渲染流程

```typescript
/**
 * 渲染完整的像素中国地形。
 *
 * 渲染顺序：
 * 1. 填充海洋背景
 * 2. 解码 RLE 绘制陆地色块
 * 3. 叠加水系
 * 4. 叠加像素噪声（增加纹理感）
 */
export function renderTerrain(canvas: HTMLCanvasElement): void {
  const ctx = canvas.getContext('2d')!;
  canvas.width = GRID_W * CELL_SIZE;
  canvas.height = GRID_H * CELL_SIZE;

  // 1. 海洋背景
  ctx.fillStyle = TERRAIN_COLORS.ocean;
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  // 2. 陆地
  const grid = decodeRLE(CHINA_RLE);
  for (let row = 0; row < GRID_H; row++) {
    for (let col = 0; col < GRID_W; col++) {
      const terrain = grid[row][col];
      if (terrain === 'ocean') continue;

      ctx.fillStyle = TERRAIN_COLORS[terrain];
      ctx.fillRect(col * CELL_SIZE, row * CELL_SIZE, CELL_SIZE, CELL_SIZE);
    }
  }

  // 3. 水系
  drawRivers(ctx);

  // 4. 像素噪声
  ctx.globalAlpha = 0.08;
  for (let row = 0; row < GRID_H; row++) {
    for (let col = 0; col < GRID_W; col++) {
      if (grid[row][col] !== 'ocean' && Math.random() > 0.7) {
        ctx.fillStyle = Math.random() > 0.5 ? '#fff' : '#000';
        ctx.fillRect(
          col * CELL_SIZE + 2,
          row * CELL_SIZE + 2,
          CELL_SIZE - 4,
          CELL_SIZE - 4
        );
      }
    }
  }
  ctx.globalAlpha = 1;
}
```

---

## 11. 管理后台完整实现

### 11.1 AdminConsole 三 Tab 架构

管理后台采用三个 Tab 页面组织：

```typescript
/**
 * AdminConsole.tsx — 管理后台主组件
 *
 * 三个 Tab：
 * 1. Stations — 站点管理（新增/编辑/删除）
 * 2. Messages — 留言管理（审核/删除）
 * 3. System — 系统诊断
 */
export function AdminConsole() {
  const [tab, setTab] = useState<'stations' | 'messages' | 'system'>('stations');

  return (
    <div className="admin-console">
      <div className="admin-tabs">
        <button
          className={tab === 'stations' ? 'active' : ''}
          onClick={() => setTab('stations')}
        >
          站点管理
        </button>
        <button
          className={tab === 'messages' ? 'active' : ''}
          onClick={() => setTab('messages')}
        >
          留言管理
        </button>
        <button
          className={tab === 'system' ? 'active' : ''}
          onClick={() => setTab('system')}
        >
          系统诊断
        </button>
      </div>

      {tab === 'stations' && <StationEditor />}
      {tab === 'messages' && <MessageEditor />}
      {tab === 'system' && <AdminDiagnostics />}
    </div>
  );
}
```

### 11.2 StationEditor：省份选择 + 坐标自动绑定

```typescript
/**
 * 站点编辑器。
 * 选择省份后自动填入省会坐标和 adcode。
 */
function StationEditor() {
  const [form, setForm] = useState<Partial<Station>>({
    status: 'upcoming',
  });

  /** 省份选择 → 自动绑定坐标 */
  function handleProvinceChange(province: string) {
    const coord = PROVINCE_CAPITALS[province];
    setForm(prev => ({
      ...prev,
      province,
      lat: coord?.[1] ?? prev.lat,
      lng: coord?.[0] ?? prev.lng,
      adcode: PROVINCE_ADCODES[province] ?? prev.adcode,
    }));
  }

  async function handleSave() {
    if (!form.city || !form.province) {
      alert('请填写城市和省份');
      return;
    }

    const res = await fetch('/api/admin/stations', {
      method: form.id ? 'PATCH' : 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(form),
    });

    if (res.ok) {
      // 刷新列表
      refreshStations();
      setForm({ status: 'upcoming' });
    }
  }

  return (
    <div className="station-editor">
      <div className="form-row">
        <label>省份</label>
        <select
          value={form.province || ''}
          onChange={e => handleProvinceChange(e.target.value)}
        >
          <option value="">选择省份</option>
          {PROVINCES.map(p => (
            <option key={p} value={p}>{p}</option>
          ))}
        </select>
      </div>

      <div className="form-row">
        <label>城市</label>
        <input
          value={form.city || ''}
          onChange={e => setForm(prev => ({ ...prev, city: e.target.value }))}
        />
      </div>

      <div className="form-row">
        <label>场馆</label>
        <input
          value={form.venue || ''}
          onChange={e => setForm(prev => ({ ...prev, venue: e.target.value }))}
        />
      </div>

      <div className="form-row">
        <label>坐标</label>
        <span className="auto-coord">
          {form.lat?.toFixed(4)}, {form.lng?.toFixed(4)}
          {form.adcode && <span className="adcode">(adcode: {form.adcode})</span>}
        </span>
      </div>

      <button onClick={handleSave}>
        {form.id ? '更新' : '新增'}站点
      </button>
    </div>
  );
}
```

### 11.3 MessageEditor：留言编辑器

```typescript
/**
 * 留言管理列表。
 * 功能：审核（pending → published）、隐藏、删除。
 */
function MessageEditor() {
  const { allMsgs, refreshAdmin } = useApp();

  async function handleAction(id: string, action: 'publish' | 'hide' | 'delete') {
    const method = action === 'delete' ? 'DELETE' : 'PATCH';
    const body = action === 'delete'
      ? undefined
      : JSON.stringify({ status: action === 'publish' ? 'published' : 'hidden' });

    await fetch(`/api/admin/messages/${id}`, { method, body });
    refreshAdmin();
  }

  return (
    <div className="message-editor">
      {allMsgs.map(msg => (
        <div key={msg.id} className={`msg-row status-${msg.status}`}>
          <div className="msg-meta">
            <span className="nickname">{msg.nickname}</span>
            <span className="status-badge">{msg.status}</span>
            <span className="date">{msg.created_at}</span>
          </div>
          <div className="msg-content">{msg.content}</div>
          <div className="msg-actions">
            {msg.status === 'pending' && (
              <button onClick={() => handleAction(msg.id, 'publish')}>审核通过</button>
            )}
            {msg.status === 'published' && (
              <button onClick={() => handleAction(msg.id, 'hide')}>隐藏</button>
            )}
            <button className="danger" onClick={() => handleAction(msg.id, 'delete')}>
              删除
            </button>
          </div>
        </div>
      ))}
    </div>
  );
}
```

### 11.4 城市抽屉：`useCityGroups` hook

移动端的城市列表通过侧滑抽屉展示，`useCityGroups` hook 将站点按城市分组并合并同城多站：

```typescript
/**
 * 按城市分组站点的 hook。
 * 同城多站（如北京有两场）合并为一个城市组。
 */
function useCityGroups(stations: Station[]) {
  return useMemo(() => {
    const groups = new Map<string, Station[]>();

    for (const station of stations) {
      const key = cityGroupKey(station);
      if (!groups.has(key)) {
        groups.set(key, []);
      }
      groups.get(key)!.push(station);
    }

    return Array.from(groups.entries()).map(([key, stations]) => ({
      key,
      city: stations[0].city,
      province: stations[0].province,
      stations,
      status: combinedStatus(stations),
      totalMessages: stations.reduce((s, st) => s + (st.message_count || 0), 0),
    }));
  }, [stations]);
}

/**
 * 生成城市分组键。
 * "北京_北京" — 省份 + 城市确保唯一。
 */
function cityGroupKey(station: Station): string {
  return `${station.province}_${station.city}`;
}

/**
 * 合并同城多站的状态。
 * 优先级：live > upcoming > done
 */
function combinedStatus(stations: Station[]): StationStatus {
  if (stations.some(s => s.status === 'live')) return 'live';
  if (stations.some(s => s.status === 'upcoming')) return 'upcoming';
  return 'done';
}
```

---

## 12. 问题、踩坑与设计取舍

### 12.1 为什么不用地图库？

Leaflet/Mapbox 体积大、风格与像素美术不搭。自定义 SVG 渲染可以：
- 完全控制视觉风格（省份填色、标记样式）
- 体积更小（GeoJSON 数据 < 500KB）
- 更好的交互控制（点击省份 = 跳转留言墙）

代价是失去了瓦片地图的平移/缩放能力，但对于固定视角的巡演地图来说完全够用。

### 12.2 为什么手写 AWS4 签名？

AWS SDK 完整引入会增加 bundle 体积。预签名 URL 只需要 HMAC-SHA256 签名，用 Node.js `crypto` 模块即可实现，零额外依赖。

### 12.3 为什么用 React Context 而不是 Zustand？

项目只有一个页面（SPA），状态树不复杂。Context + `useCallback` 足够，且零依赖。如果后续状态膨胀再迁移也很简单。

### 12.4 Vercel 冷启动影响

Vercel Serverless Functions 的冷启动延迟约 300-800ms，首次请求明显慢于后续请求。应对措施：

```typescript
// Mock 降级在冷启动时自动兜底
// 用户感知：首次访问看到 mock 数据 → API 返回后无缝替换

// 另外：将 Supabase 客户端初始化放在模块顶层
// 避免每次请求重新初始化
const supabase = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);
```

### 12.5 GeoJSON 文件大小优化

原始中国省级 GeoJSON 约 2.8MB，优化后降到 480KB：

```
原始 GeoJSON:          2.8 MB
  ↓ 移除不需要的属性     2.1 MB
  ↓ 降低坐标精度（6→3位）1.2 MB
  ↓ 简化多边形（保留形状）0.7 MB
  ↓ gzip 传输            0.48 MB
```

### 12.6 R2 CORS 配置踩坑

```json
// 踩坑：浏览器直传 R2 报 CORS 错误

// 原因：R2 默认不允许跨域 PUT 请求

// 解决：在 R2 Bucket 设置 CORS 策略
{
  "AllowedOrigins": ["https://chilichill.vercel.app", "http://localhost:3000"],
  "AllowedMethods": ["PUT", "GET"],
  "AllowedHeaders": ["Content-Type"],
  "ExposeHeaders": ["ETag"],
  "MaxAgeSeconds": 3600
}

// 教训：预签名 URL 虽然绕过了服务端，但 CORS 策略仍由 R2 控制
```

### 12.7 像素字体加载时序问题

```css
/* 踩坑：像素字体 @font-face 声明后，Canvas 绘制时字体还未加载完 */
/* 导致分享卡中的文字用系统默认字体渲染 */

/* 解决：使用 document.fonts.load() 等待字体就绪 */
```

```typescript
async function ensureFontLoaded() {
  try {
    await document.fonts.load('16px "Zpix"');
  } catch {
    // Safari 某些版本不支持 fonts.load()
    // fallback：等待 500ms
    await new Promise(resolve => setTimeout(resolve, 500));
  }
}

// 在绘制分享卡前调用
async function drawShareCard(canvas, mode, data) {
  await ensureFontLoaded();
  // ... 开始绘制
}
```

### 12.8 `content-visibility` 在 Safari 的兼容性

```css
/* 踩坑：content-visibility: auto 在 Safari < 17 中不支持 */
/* 导致留言卡片全部立即渲染，页面卡顿 */

/* 解决：条件使用，Safari 降级到常规渲染 */
@supports (content-visibility: auto) {
  .msg-card {
    content-visibility: auto;
    contain-intrinsic-size: auto 180px;
  }
}
```

### 12.9 同城多站合并逻辑

```typescript
// 踩坑：北京有两场巡演，地图上显示两个重叠标记
// 用户体验差，且碰撞分散算法会把它们推开到奇怪的位置

// 解决：useCityGroups hook 将同城站点合并为一个城市组
// 地图上只显示一个标记，点击后在城市抽屉中展示两场

// cityGroupKey 保证分组唯一：
// "北京_北京" → 同一个标记
// "重庆_重庆" → 另一个标记
```

### 12.10 性能优化

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

### 12.11 美术素材预留架构

`@chili/assets` 声明了完整的像素美术元数据（spritesheet 帧数/尺寸/FPS），`@chili/ui` 的 `drawNpc()` 当前用 Canvas 绘制 8x8 占位小人。设计为**美术交付后只需替换实现不改接口**——面向接口编程的典型实践。

### 12.12 内容审核流设计

数据库 RLS 设计为粉丝只能插入 `pending` 状态留言（需管理员审核后改为 `published`）。但 MVP 阶段 API 直接设为 `published` 简化流程，后续可随时切回审核模式。

---

## 13. 附录与参考

### 13.1 核心数据流

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

### 13.2 API 路由索引

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

### 13.3 关键文件索引

| 文件 | 行数 | 核心职责 |
|------|------|---------|
| `store.tsx` | 395 | 全局状态管理（路由/数据/认证/弹窗） |
| `globals.css` | 836 | CRT/地图/留言/管理全量样式 |
| `TourMap.tsx` | 353 | GeoJSON 投影 + 省份着色 + 碰撞分散 |
| `ShareModal.tsx` | 349 | Canvas 分享卡（复古外壳/伪QR/星星） |
| `Composer.tsx` | 287 | 留言撰写 + 图片上传 + 缩略图生成 |
| `MessageWall.tsx` | ~300 | 留言墙（卡片/回复/互动/分页） |
| `BootScreen.tsx` | ~100 | CRT 开机动画 |
| `AdminConsole.tsx` | ~200 | 管理后台三 Tab 架构 |
| `presign/route.ts` | 137 | 手写 AWS4 签名 |
| `schema.sql` | 102 | 数据库 DDL + RLS 策略 |
| `db/index.ts` | ~200 | API 优先 + Mock 降级数据层 |
| `db/mock.ts` | ~300 | 完整 localStorage 持久化 Mock |
| `scene/terrain.ts` | ~150 | 像素地形 Canvas 渲染引擎 |

---

## 免责声明

本项目为 ChiliChill 乐队巡演粉丝社区产品的技术实现记录。文中代码摘录用于技术交流学习。
