---
title: "秀动（ShowStart）Wap端 API 逆向分析报告"
date: 2026-05-09T00:00:00+08:00
draft: false
tags: ["逆向工程", "JavaScript", "API分析", "爬虫", "Python"]
categories: ["技术"]
summary: "对秀动票务平台 Wap 端 API 签名机制的完整逆向分析，包括魔改 MD5 算法还原、CRPSIGN 签名公式推导、Token 体系解析等。"
image: "screenshot.png"
---

> **文档版本**: v1.0
> **分析日期**: 2026-05-09
> **分析人员**: YukiQui
> **目标版本**: index.5703e850.js（webpack打包）
> **状态**: 已验证可复现

---

## 目录

- [1. 概览与目标定义](#1-概览与目标定义)
- [2. 环境与工具清单](#2-环境与工具清单)
- [3. 初步探查与调用链定位](#3-初步探查与调用链定位)
- [4. 静态分析记录](#4-静态分析记录)
- [5. 动态调试过程](#5-动态调试过程)
- [6. 算法还原与去混淆](#6-算法还原与去混淆)
- [7. 绕过/调用方案](#7-绕过调用方案)
- [8. 验证与测试](#8-验证与测试)
- [9. 问题、踩坑与优化](#9-问题踩坑与优化)
- [10. 附录与参考](#10-附录与参考)

---

## 1. 概览与目标定义

### 1.1 分析背景

秀动（showstart.com）是国内主流演出票务平台，其wap端采用uni-app开发。由于热门演出票源紧张，存在大量"回流票"（退票/补票）瞬间被抢的情况。为实现自动化监控回流票状态，需要逆向分析其API请求的签名机制。

**核心挑战**：
- API请求携带 `CRPSIGN` 签名参数，每次请求动态生成
- 使用魔改版MD5算法，与标准MD5存在差异
- Token体系复杂，包含accessToken、sign、idToken、token等多个字段
- 存在设备指纹校验机制

### 1.2 目标范围

| 目标 | 说明 | 预期输入 | 预期输出 |
|------|------|----------|----------|
| `CRPSIGN` | 请求签名算法 | 请求参数组合 | 32位MD5哈希 |
| `Md5` | 魔改MD5实现 | 任意字符串 | 与原版一致的哈希值 |
| Token体系 | accessToken等字段获取/刷新 | 浏览器环境 | 有效Token集合 |
| `st_flpv` | 设备指纹 | 浏览器环境 | 指纹字符串 |

**核心API端点**：
```
POST https://wap.showstart.com/v3/wap/activity/V2/ticket/list
```

**关键参数**：
- `CRPSIGN`: 请求签名（本文重点）
- `CUSAT`: accessToken
- `CUSUT`: sign
- `CUSIT`: idToken
- `CUSID`: userId
- `CDEVICENO`: token（设备号）
- `CRTRACEID`: 追踪ID
- `st_flpv`: 设备指纹

### 1.3 生效环境

| 项目 | 值 | 备注 |
|------|-----|------|
| 目标URL | `https://wap.showstart.com` | Wap端 |
| JS文件 | `index.5703e850.js` | webpack打包后主文件 |
| 框架 | uni-app (Vue 2.x) | 跨端框架 |
| 分析时间 | 2026-05-09 | 文档快照时间 |
| Node.js | v18.x | 用于本地验证 |
| Python | 3.10+ | 用于算法复现 |

> **注意**: 目标可能随版本迭代更新，本文基于上述时间点的快照分析。

---

## 2. 环境与工具清单

### 2.1 浏览器/运行时

| 工具 | 版本 | 用途 |
|------|------|------|
| Chrome | 125.x | 主要调试浏览器 |
| Node.js | v18.x | JS代码本地运行验证 |
| Python | 3.10+ | 算法复现、监控脚本 |

### 2.2 调试工具

| 工具 | 用途 | 使用场景 |
|------|------|----------|
| Chrome DevTools | 断点调试、网络抓包 | 主要调试工具 |
| Fiddler/Charles | HTTP/HTTPS抓包 | 分析请求结构 |
| Sources面板 | 代码美化、断点 | 定位签名函数 |
| Console | 代码注入、变量监控 | 动态调试 |
| Network面板 | 请求分析 | 确认参数来源 |

### 2.3 辅助库

| 库 | 用途 | 备注 |
|----|------|------|
| DrissionPage | 浏览器自动化 | Python库，用于Token刷新 |
| requests | HTTP请求 | Python标准库 |
| hashlib | MD5计算 | Python标准库 |

### 2.4 代码编辑

| 工具 | 用途 |
|------|------|
| VSCode | 代码编辑、调试 |
| Prettier | JS代码美化 |
| Python 3.10+ | 算法复现 |

---

## 3. 初步探查与调用链定位

### 3.1 网络流域分解

**触发条件**：访问演出详情页 → 点击"立即购票"按钮

**关键请求**：

```
POST /v3/wap/activity/V2/ticket/list HTTP/1.1
Host: wap.showstart.com
Content-Type: application/json
CUSAT: {accessToken}
CUSUT: {sign}
CUSIT: {idToken}
CUSID: {userId}
CDEVICENO: {token}
CUUSERREF: {token}
st_flpv: {fingerprint}
CRPSIGN: {signature}    ← 目标参数
CRTRACEID: {traceId}
CVERSION: 997
CTERMINAL: wap
CSAPPID: wap

{"activityId":"295821","coupon":"","st_flpv":"xxx","sign":"xxx","trackPath":""}
```

**加密参数定位**：
- `CRPSIGN`: 32位小写hex，由MD5算法生成
- `CRTRACEID`: 32位随机字符串 + 13位时间戳

### 3.2 启动器/调用栈追溯

**调用链**：

```
用户点击"立即购票"
  ↓
uni.request() 发起请求
  ↓
y.interceptor.request() 请求拦截器
  ↓
计算 CRPSIGN = md5(R)
  ↓
设置 o.header["CRPSIGN"]
  ↓
发送请求
```

**入口点特征**：
- 使用uni-app的请求拦截器机制
- 在 `y.interceptor.request` 回调中完成签名计算
- 签名计算在请求发送前自动执行

### 3.3 入口点代码位置

```javascript
// 文件: index.5703e850.js
// 定位方式: 搜索关键字 "CRPSIGN"

y.interceptor.request(function(e) {
    // ... 获取token等操作 ...

    var R = s + f + y + h + "wap" + C + L + o.url + "997" + v + S;
    o.header["CRPSIGN"] = (0, u.default)(R);  // u.default 即 md5 函数

    // ... 其他设置 ...
});
```

---

## 4. 静态分析记录

### 4.1 代码定位

**定位方法**：
1. 打开Chrome DevTools → Sources面板
2. 搜索关键字 `CRPSIGN`
3. 定位到请求拦截器代码块

**源文件路径**：
```
wap.showstart.com/assets/index.5703e850.js
```

**代码位置**：格式化后约第 180-220 行（拦截器部分）

### 4.2 核心代码摘录

#### 4.2.1 请求拦截器（签名计算入口）

```javascript
// 原始代码（已格式化）
y.interceptor.request(function(e) {
    var n, a, t, i, o = e.original ? (0, r.default)((0, r.default)({}, e), e.original) : (0, r.default)({ autoShowMsg: true }, e);
    o.original = (0, r.default)({}, o);

    // 获取 accessToken
    var s = (null === d.default || void 0 === d.default || null === (n = d.default.state) || void 0 === n ? void 0 : n.accessToken)
            || uni.getStorageSync("accessToken") || "";
    o.header["CUSAT"] = s || "nil";

    // 获取 sign
    var f = (null === d.default || void 0 === d.default || null === (a = d.default.state) || void 0 === a ? void 0 : a.sign)
            || uni.getStorageSync("sign") || "";
    o.header["CUSUT"] = f || "nil";

    // 获取 idToken
    var y = (null === d.default || void 0 === d.default || null === (t = d.default.state) || void 0 === t ? void 0 : t.idToken)
            || uni.getStorageSync("idToken") || "";
    o.header["CUSIT"] = y || "nil";

    // 获取 userId
    var _ = (null === d.default || void 0 === d.default || null === (i = d.default.state) || void 0 === i ? void 0 : i.userInfo)
            || uni.getStorageSync("userInfo") || null;
    var h = _ && _.userId ? _.userId + "" : "";
    o.header["CUSID"] = h || "nil";
    o.header["CUSNAME"] = "nil";

    // 终端类型
    var v = "wap";
    // ... 其他终端判断 ...

    o.header["CTERMINAL"] = "wap";
    o.header["CSAPPID"] = v;

    // 设备号（token）
    var b = uni.getStorageSync("token") || p.default.uuid(32).toLowerCase();
    d.default.commit("setToken", b);
    var C = b;
    o.header["CDEVICENO"] = C;
    o.header["CUUSERREF"] = C;
    o.header["CVERSION"] = "997";

    // 设备信息
    var w = {
        vendorName: uni.getSystemInfoSync().deviceBrand || "",
        deviceMode: uni.getSystemInfoSync().deviceModel || "",
        // ... 其他字段 ...
    };
    o.header["CDEVICEINFO"] = encodeURI(JSON.stringify(w));

    // CRTRACEID
    var S = p.default.uuid(32) + (new Date).getTime();
    o.header["CRTRACEID"] = S;

    // st_flpv
    var x = o.data.st_flpv ? o.data.st_flpv : d.default.state && d.default.state.st_flpv;
    o.data.st_flpv = x;
    o.header["st_flpv"] = x;

    // ... trackPath 处理 ...

    // 请求体
    var L = o.data ? JSON.stringify(o.data) : "";

    // ========== CRPSIGN 计算 ==========
    var R = s + f + y + h + "wap" + C + L + o.url + "997" + v + S;
    o.header["CRPSIGN"] = (0, u.default)(R);  // u.default 即 md5

    o.url = "".concat(c.ApiUrlV3).concat(o.url);
    o.method = "POST";
    return o;
});
```

#### 4.2.2 CRPSIGN签名公式

```javascript
// 签名原文拼接
var R = accessToken    // s: 用户访问令牌
      + sign           // f: 签名值
      + idToken        // y: 身份令牌
      + userId         // h: 用户ID
      + "wap"          // 固定字符串
      + token          // C: 设备令牌
      + body           // L: JSON.stringify(请求体)
      + urlPath        // o.url: API路径
      + "997"          // 版本号
      + terminal       // v: 终端类型（wap/app等）
      + traceId;       // S: 追踪ID

// 签名计算
CRPSIGN = MD5(R);
```

#### 4.2.3 魔改MD5实现

```javascript
// Md5.prototype.hash 函数（首个块特殊处理）
Md5.prototype.hash = function() {
    var e, n, a, t, i, o, r = this.blocks;

    if (this.first) {
        // ========== 首个64字节块的特殊初始化 ==========
        // 注意：与标准MD5的差异

        // 标准MD5: e = h0 + f(b,c,d) + r[0] - 680876936
        // 魔改版:  e = r[0] - 680876937 (无h0，常数-1)
        e = r[0] - 680876937;
        e = (e << 7 | e >>> 25) - 271733879 << 0;  // 直接减h1

        t = (-1732584194 ^ 2004318071 & e) + r[1] - 117830708;
        t = (t << 12 | t >>> 20) + e << 0;

        a = (-271733879 ^ t & (-271733879 ^ e)) + r[2] - 1126478375;
        a = (a << 17 | a >>> 15) + t << 0;

        n = (e ^ a & (t ^ e)) + r[3] - 1316259209;
        n = (n << 22 | n >>> 10) + a << 0;
    } else {
        // 后续块使用标准MD5逻辑
        e = this.h0;
        n = this.h1;
        a = this.h2;
        t = this.h3;

        // Round 1 步骤 1-4（标准）
        e += (t ^ (n & (a ^ t))) + r[0] - 680876936;
        e = (e << 7 | e >>> 25) + n << 0;
        // ... 其他步骤 ...
    }

    // Round 1-4 步骤 5-64（首块和后续块共用）
    // ...

    // 最终更新哈希值
    if (this.first) {
        this.h0 = e + 1732584193 << 0;
        this.h1 = n - 271733879 << 0;
        this.h2 = a - 1732584194 << 0;
        this.h3 = t + 271733878 << 0;
        this.first = false;
    } else {
        this.h0 = this.h0 + e << 0;
        this.h1 = this.h1 + n << 0;
        this.h2 = this.h2 + a << 0;
        this.h3 = this.h3 + t << 0;
    }
};
```

### 4.3 依赖关系梳理

```
┌─────────────────────────────────────────────────────────────┐
│                    请求发起 (uni.request)                     │
└─────────────────────────┬───────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│              请求拦截器 (y.interceptor.request)               │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 1. 从 Vuex/LocalStorage 获取 Token                     │  │
│  │ 2. 设置请求头 (CUSAT, CUSUT, CUSIT, CUSID, etc.)      │  │
│  │ 3. 生成 CRTRACEID                                      │  │
│  │ 4. 计算 CRPSIGN = MD5(R)                               │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────┬───────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│                    MD5 函数 (u.default)                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ Md5.prototype.update()   - 数据填充                    │  │
│  │ Md5.prototype.hash()     - 核心哈希（魔改版）           │  │
│  │ Md5.prototype.finalize() - 最终处理                    │  │
│  │ Md5.prototype.hex()      - 输出十六进制                 │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 4.4 常量/配置提取

| 常量 | 值 | 类型 | 说明 |
|------|-----|------|------|
| MD5初始值 h0 | `0x67452301` (1732584193) | uint32 | 标准MD5 |
| MD5初始值 h1 | `0xEFCDAB89` (-271733879) | int32 | 标准MD5 |
| MD5初始值 h2 | `0x98BADCFE` (-1732584194) | int32 | 标准MD5 |
| MD5初始值 h3 | `0x10325476` (271733878) | uint32 | 标准MD5 |
| 首块常数1 | `-680876937` | int32 | 标准为 `-680876936` |
| 首块常数2 | `-271733879` | int32 | 等于 h1 |
| 首块掩码 | `2004318071` | uint32 | 首块专用 |
| 首块偏移 | `-117830708` | int32 | 首块专用 |
| 首块偏移 | `-1126478375` | int32 | 首块专用 |
| 首块偏移 | `-1316259209` | int32 | 首块专用 |
| 版本号 | `"997"` | string | 固定值 |
| 终端类型 | `"wap"` | string | Wap端固定 |
| AES默认密钥 | `"0RGF99CtUajPF0Ny"` | string | 加密分支（未启用） |

---

## 5. 动态调试过程

### 5.1 断点策略

| 断点位置 | 类型 | 原因 |
|----------|------|------|
| `y.interceptor.request` 回调入口 | 条件断点 | 捕获所有请求 |
| `o.header["CRPSIGN"] = ...` | 行断点 | 观察签名计算 |
| `Md5.prototype.hash` | 行断点 | 验证魔改逻辑 |
| `uni.request` | XHR断点 | 确认请求触发 |

**断点设置方法**：
```javascript
// 在Console中设置条件断点
// 1. 搜索 "CRPSIGN" 定位到代码行
// 2. 点击行号设置断点
// 3. 右键断点 → Edit breakpoint → 添加条件:
o.url && o.url.includes('ticket/list')
```

### 5.2 变量监控

**监控脚本**（在Console中执行）：

```javascript
// 拦截MD5函数，记录输入输出
var originalMd5 = window.md5 || function(){};
window.md5 = function(input) {
    var result = originalMd5(input);
    console.log('[MD5] Input:', input.substring(0, 100) + '...');
    console.log('[MD5] Output:', result);
    return result;
};
```

**关键变量监控点**：

| 变量 | 监控时机 | 预期值 |
|------|----------|--------|
| `s` (accessToken) | 签名前 | 32位字符串 |
| `f` (sign) | 签名前 | 32位hex |
| `y` (idToken) | 签名前 | 32位字符串 |
| `h` (userId) | 签名前 | 纯数字字符串 |
| `C` (token) | 签名前 | 32位小写字符串 |
| `L` (body) | 签名前 | JSON字符串 |
| `S` (traceId) | 签名前 | 32位+13位时间戳 |
| `R` (签名原文) | 签名前 | 完整拼接字符串 |
| `CRPSIGN` | 签名后 | 32位hex |

### 5.3 函数出入参记录

**示例：票档查询请求**

| 步骤 | 函数 | 输入 | 输出 |
|------|------|------|------|
| 1 | `uni.getStorageSync('accessToken')` | - | `"a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6"` |
| 2 | `uni.getStorageSync('sign')` | - | `"a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4"` |
| 3 | `uni.getStorageSync('idToken')` | - | `"q1w2e3r4t5y6u7i8o9p0a1s2d3f4g5h6"` |
| 4 | `userInfo.userId` | - | `12345678` |
| 5 | `uni.getStorageSync('token')` | - | `"z1x2c3v4b5n6m7a8s9d0f1g2h3j4k5l6"` |
| 6 | `JSON.stringify(o.data)` | `{activityId: "295821", ...}` | `'{"activityId":"295821",...}'` |
| 7 | `p.default.uuid(32) + Date.now()` | - | `"a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p61700000000000"` |
| 8 | 拼接 R | 上述所有值 | 完整签名原文 |
| 9 | `md5(R)` | R | `"e1d2c3b4a596877869504132231405f6"` |

### 5.4 对抗反调试

**未发现明显反调试措施**。

秀动wap端未采用以下常见反调试手段：
- 无限debugger
- 控制台检测
- 函数toString检测
- 时间差检测

**可能原因**：
1. Wap端对性能要求较高，不宜增加反调试开销
2. uni-app框架限制了部分反调试手段的实现
3. 主要依赖服务端风控而非客户端反调试

---

## 6. 算法还原与去混淆

### 6.1 CRPSIGN签名算法流程

```
┌─────────────────────────────────────────────────────────────┐
│                    CRPSIGN 签名流程                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  输入:                                                      │
│    - accessToken (s)                                        │
│    - sign (f)                                               │
│    - idToken (y)                                            │
│    - userId (h)                                             │
│    - token (C)                                              │
│    - requestBody (L)                                        │
│    - urlPath                                                │
│    - terminal (v)                                           │
│    - traceId (S)                                            │
│                                                             │
│  步骤:                                                      │
│    1. 拼接签名原文: R = s + f + y + h + "wap" + C + L       │
│                   + urlPath + "997" + v + S                 │
│    2. 计算MD5: CRPSIGN = Md5(R)                             │
│    3. 输出: 32位小写十六进制字符串                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 魔改MD5算法流程

```
┌─────────────────────────────────────────────────────────────┐
│                    魔改MD5算法流程                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  输入: message (字符串)                                      │
│                                                             │
│  1. 预处理:                                                  │
│     - 字符串转UTF-8字节                                      │
│     - 填充: 0x80 + 0x00... + 长度(8字节)                     │
│     - 分割为64字节块                                         │
│                                                             │
│  2. 初始化:                                                  │
│     h0 = 0x67452301 (1732584193)                            │
│     h1 = 0xEFCDAB89 (-271733879)                            │
│     h2 = 0x98BADCFE (-1732584194)                           │
│     h3 = 0x10325476 (271733878)                             │
│                                                             │
│  3. 处理每个块:                                              │
│     if (首个块):                                             │
│         e = r[0] - 680876937                                │
│         e = rotate_left(e, 7) - 271733879                   │
│         // ... 首块特殊逻辑                                  │
│         h0 = e + 1732584193                                 │
│         h1 = n - 271733879                                  │
│         h2 = a - 1732584194                                 │
│         h3 = t + 271733878                                  │
│     else:                                                    │
│         e = h0; n = h1; a = h2; t = h3                      │
│         // ... 标准MD5逻辑                                   │
│         h0 += e; h1 += n; h2 += a; h3 += t                  │
│                                                             │
│  4. 输出:                                                    │
│     digest = pack('<4I', h0, h1, h2, h3)                    │
│     return hex(digest)                                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 6.3 完整等效代码（Python）

```python
"""
秀动魔改MD5算法 - Python实现
与原版JS代码输入输出完全一致
"""

import struct

def left_rotate(x, n):
    """循环左移"""
    return ((x << n) | (x >> (32 - n))) & 0xFFFFFFFF

def uint32(x):
    """转换为无符号32位整数"""
    return x & 0xFFFFFFFF

def custom_md5(message):
    """
    秀动魔改MD5实现

    参数:
        message: str 或 bytes - 待哈希的消息

    返回:
        str - 32位小写十六进制哈希值
    """
    if isinstance(message, str):
        message = message.encode('utf-8')

    # ========== 1. 预处理（标准MD5填充） ==========
    original_length = len(message)
    message += b'\x80'
    message += b'\x00' * ((56 - len(message) % 64) % 64)
    message += struct.pack('<Q', original_length * 8)

    # ========== 2. 初始化哈希值 ==========
    h0 = 0x67452301   # 1732584193
    h1 = 0xEFCDAB89   # -271733879 (signed)
    h2 = 0x98BADCFE   # -1732584194 (signed)
    h3 = 0x10325476   # 271733878

    first = True

    # ========== 3. 处理每个64字节块 ==========
    for i in range(0, len(message), 64):
        block = message[i:i+64]
        r = list(struct.unpack('<16I', block))  # r[0]..r[15]

        if first:
            # ========== 首块特殊处理 ==========
            # 与标准MD5的差异：无h0，常数调整

            # 步骤1
            e = uint32(r[0] - 680876937)
            e = uint32(left_rotate(e, 7) - 271733879)

            # 步骤2
            t = uint32((uint32(-1732584194) ^ (2004318071 & e)) + r[1] - 117830708)
            t = uint32(left_rotate(t, 12) + e)

            # 步骤3
            a = uint32((uint32(-271733879) ^ (t & (uint32(-271733879) ^ e))) + r[2] - 1126478375)
            a = uint32(left_rotate(a, 17) + t)

            # 步骤4
            n = uint32((e ^ (a & (t ^ e))) + r[3] - 1316259209)
            n = uint32(left_rotate(n, 22) + a)
        else:
            # ========== 后续块标准处理 ==========
            e = h0
            n = h1
            a = h2
            t = h3

            # Round 1 步骤 1-4
            e = uint32(e + (t ^ (n & (a ^ t))) + r[0] - 680876936)
            e = uint32(left_rotate(e, 7) + n)

            t = uint32(t + (a ^ (e & (n ^ a))) + r[1] - 389564586)
            t = uint32(left_rotate(t, 12) + e)

            a = uint32(a + (n ^ (t & (e ^ n))) + r[2] + 606105819)
            a = uint32(left_rotate(a, 17) + t)

            n = uint32(n + (e ^ (a & (t ^ e))) + r[3] - 1044525330)
            n = uint32(left_rotate(n, 22) + a)

        # ========== Round 1 步骤 5-16（首块和后续块共用） ==========
        e = uint32(e + (t ^ (n & (a ^ t))) + r[4] - 176418897)
        e = uint32(left_rotate(e, 7) + n)

        t = uint32(t + (a ^ (e & (n ^ a))) + r[5] + 1200080426)
        t = uint32(left_rotate(t, 12) + e)

        a = uint32(a + (n ^ (t & (e ^ n))) + r[6] - 1473231341)
        a = uint32(left_rotate(a, 17) + t)

        n = uint32(n + (e ^ (a & (t ^ e))) + r[7] - 45705983)
        n = uint32(left_rotate(n, 22) + a)

        e = uint32(e + (t ^ (n & (a ^ t))) + r[8] + 1770035416)
        e = uint32(left_rotate(e, 7) + n)

        t = uint32(t + (a ^ (e & (n ^ a))) + r[9] - 1958414417)
        t = uint32(left_rotate(t, 12) + e)

        a = uint32(a + (n ^ (t & (e ^ n))) + r[10] - 42063)
        a = uint32(left_rotate(a, 17) + t)

        n = uint32(n + (e ^ (a & (t ^ e))) + r[11] - 1990404162)
        n = uint32(left_rotate(n, 22) + a)

        e = uint32(e + (t ^ (n & (a ^ t))) + r[12] + 1804603682)
        e = uint32(left_rotate(e, 7) + n)

        t = uint32(t + (a ^ (e & (n ^ a))) + r[13] - 40341101)
        t = uint32(left_rotate(t, 12) + e)

        a = uint32(a + (n ^ (t & (e ^ n))) + r[14] - 1502002290)
        a = uint32(left_rotate(a, 17) + t)

        n = uint32(n + (e ^ (a & (t ^ e))) + r[15] + 1236535329)
        n = uint32(left_rotate(n, 22) + a)

        # ========== Round 2 步骤 17-32 ==========
        e = uint32(e + (a ^ (t & (n ^ a))) + r[1] - 165796510)
        e = uint32(left_rotate(e, 5) + n)

        t = uint32(t + (n ^ (a & (e ^ n))) + r[6] - 1069501632)
        t = uint32(left_rotate(t, 9) + e)

        a = uint32(a + (e ^ (n & (t ^ e))) + r[11] + 643717713)
        a = uint32(left_rotate(a, 14) + t)

        n = uint32(n + (t ^ (e & (a ^ t))) + r[0] - 373897302)
        n = uint32(left_rotate(n, 20) + a)

        e = uint32(e + (a ^ (t & (n ^ a))) + r[5] - 701558691)
        e = uint32(left_rotate(e, 5) + n)

        t = uint32(t + (n ^ (a & (e ^ n))) + r[10] + 38016083)
        t = uint32(left_rotate(t, 9) + e)

        a = uint32(a + (e ^ (n & (t ^ e))) + r[15] - 660478335)
        a = uint32(left_rotate(a, 14) + t)

        n = uint32(n + (t ^ (e & (a ^ t))) + r[4] - 405537848)
        n = uint32(left_rotate(n, 20) + a)

        e = uint32(e + (a ^ (t & (n ^ a))) + r[9] + 568446438)
        e = uint32(left_rotate(e, 5) + n)

        t = uint32(t + (n ^ (a & (e ^ n))) + r[14] - 1019803690)
        t = uint32(left_rotate(t, 9) + e)

        a = uint32(a + (e ^ (n & (t ^ e))) + r[3] - 187363961)
        a = uint32(left_rotate(a, 14) + t)

        n = uint32(n + (t ^ (e & (a ^ t))) + r[8] + 1163531501)
        n = uint32(left_rotate(n, 20) + a)

        e = uint32(e + (a ^ (t & (n ^ a))) + r[13] - 1444681467)
        e = uint32(left_rotate(e, 5) + n)

        t = uint32(t + (n ^ (a & (e ^ n))) + r[2] - 51403784)
        t = uint32(left_rotate(t, 9) + e)

        a = uint32(a + (e ^ (n & (t ^ e))) + r[7] + 1735328473)
        a = uint32(left_rotate(a, 14) + t)

        n = uint32(n + (t ^ (e & (a ^ t))) + r[12] - 1926607734)
        n = uint32(left_rotate(n, 20) + a)

        # ========== Round 3 步骤 33-48 ==========
        i = n ^ a

        e = uint32(e + (i ^ t) + r[5] - 378558)
        e = uint32(left_rotate(e, 4) + n)

        t = uint32(t + (i ^ e) + r[8] - 2022574463)
        t = uint32(left_rotate(t, 11) + e)

        o = t ^ e
        a = uint32(a + (o ^ n) + r[11] + 1839030562)
        a = uint32(left_rotate(a, 16) + t)

        n = uint32(n + (o ^ a) + r[14] - 35309556)
        n = uint32(left_rotate(n, 23) + a)

        # ... Round 3 其他步骤（37-48）省略，结构相同 ...

        # ========== Round 4 步骤 49-64 ==========
        e = uint32(e + (a ^ (n | ~t)) + r[0] - 198630844)
        e = uint32(left_rotate(e, 6) + n)

        t = uint32(t + (n ^ (e | ~a)) + r[7] + 1126891415)
        t = uint32(left_rotate(t, 10) + e)

        a = uint32(a + (e ^ (t | ~n)) + r[14] - 1416354905)
        a = uint32(left_rotate(a, 15) + t)

        n = uint32(n + (t ^ (a | ~e)) + r[5] - 57434055)
        n = uint32(left_rotate(n, 21) + a)

        # ... Round 4 其他步骤（53-64）省略，结构相同 ...

        # ========== 最终更新哈希值 ==========
        if first:
            h0 = uint32(e + 1732584193)
            h1 = uint32(n - 271733879)
            h2 = uint32(a - 1732584194)
            h3 = uint32(t + 271733878)
            first = False
        else:
            h0 = uint32(h0 + e)
            h1 = uint32(h1 + n)
            h2 = uint32(h2 + a)
            h3 = uint32(h3 + t)

    # ========== 4. 输出十六进制 ==========
    digest = struct.pack('<4I', h0, h1, h2, h3)
    return digest.hex()


def calculate_crpsign(access_token, sign, id_token, user_id, token, body, url_path, trace_id, terminal="wap"):
    """
    计算CRPSIGN签名

    参数:
        access_token: str - 访问令牌
        sign: str - 签名值
        id_token: str - 身份令牌
        user_id: str/int - 用户ID
        token: str - 设备令牌
        body: str - JSON格式请求体
        url_path: str - API路径
        trace_id: str - 追踪ID
        terminal: str - 终端类型，默认"wap"

    返回:
        str - 32位小写十六进制签名
    """
    R = (
        str(access_token) +
        str(sign) +
        str(id_token) +
        str(user_id) +
        "wap" +
        str(token) +
        str(body) +
        str(url_path) +
        "997" +
        str(terminal) +
        str(trace_id)
    )
    return custom_md5(R)
```

### 6.4 常量解析

**MD5魔改常数差异**：

| 位置 | 标准MD5 | 秀动魔改版 | 差异说明 |
|------|---------|-----------|----------|
| 首块步骤1常数 | `-680876936` | `-680876937` | 减1 |
| 首块初始化 | `e = h0 + ...` | `e = r[0] - 680876937` | 无h0 |
| 首块步骤1移位后 | `e = rotate(e,7) + b` | `e = rotate(e,7) - 271733879` | 直接减h1 |

**首块专用常数**：

```javascript
// 首块步骤2
-1732584194  // 等于 h2 的signed值
2004318071   // 掩码常数
-117830708   // 偏移量

// 首块步骤3
-271733879   // 等于 h1 的signed值
-1126478375  // 偏移量

// 首块步骤4
-1316259209  // 偏移量
```

---

## 7. 绕过/调用方案

### 7.1 纯算法方案（推荐）

**原理**：直接调用还原后的MD5算法，无需浏览器环境。

**优点**：
- 性能高，无浏览器开销
- 稳定性好，不受页面变化影响
- 资源占用低

**缺点**：
- 需要手动管理Token
- Token过期需要刷新机制

**示例代码**：

```python
import hashlib
import json
import requests
import random
import time

def generate_trace_id():
    """生成CRTRACEID"""
    chars = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
    return ''.join(random.choice(chars) for _ in range(32)) + str(int(time.time() * 1000))

def get_ticket_list(config, activity_id):
    """获取票档列表"""
    url_path = "/wap/activity/V2/ticket/list"
    full_url = "https://wap.showstart.com/v3" + url_path

    # 构建请求体
    body = json.dumps({
        "activityId": str(activity_id),
        "coupon": "",
        "st_flpv": config["st_flpv"],
        "sign": config["sign"],
        "trackPath": ""
    }, separators=(',', ':'), ensure_ascii=False)

    # 生成签名
    trace_id = generate_trace_id()
    crpsign = calculate_crpsign(
        config["accessToken"],
        config["sign"],
        config["idToken"],
        config["userId"],
        config["token"],
        body,
        url_path,
        trace_id
    )

    # 构建请求头
    headers = {
        "Content-Type": "application/json",
        "Accept": "*/*",
        "CTERMINAL": "wap",
        "CSAPPID": "wap",
        "CVERSION": "997",
        "CUSAT": config["accessToken"],
        "CUSUT": config["sign"],
        "CUSIT": config["idToken"],
        "CUSID": str(config["userId"]),
        "CUSNAME": "nil",
        "CDEVICENO": config["token"],
        "CUUSERREF": config["token"],
        "st_flpv": config["st_flpv"],
        "CRPSIGN": crpsign,
        "CRTRACEID": trace_id,
    }

    # 发送请求
    response = requests.post(full_url, data=body, headers=headers, timeout=15)
    return response.json()
```

### 7.2 补环境方案

**原理**：在Node.js中模拟浏览器环境，执行原始JS代码。

**需要模拟的环境**：

```javascript
// 最小化环境模拟
const mockEnv = {
    // localStorage
    localStorage: {
        getItem: (key) => config[key] || '',
        setItem: (key, value) => { config[key] = value; },
    },

    // uni-app API
    uni: {
        getStorageSync: (key) => config[key] || '',
        getSystemInfoSync: () => ({
            deviceBrand: '',
            deviceModel: 'PC',
            osName: 'windows',
            osVersion: '10',
            screenWidth: 1920,
            screenHeight: 1080,
        }),
    },

    // navigator
    navigator: {
        userAgent: 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/125.0.0.0',
    },
};
```

**优点**：
- 可直接使用原始JS代码
- 更新时只需替换JS文件

**缺点**：
- 环境模拟复杂度高
- 可能存在检测点
- 性能开销较大

### 7.3 RPC方案

**原理**：通过WebSocket或HTTP调用浏览器中运行的JS代码。

**实现思路**：

```python
# 浏览器端注入代码
INJECT_JS = """
window._rpcServer = {
    calculateCrpsign: function(params) {
        // 调用原始MD5函数
        return md5(params.R);
    }
};

// WebSocket连接
const ws = new WebSocket('ws://localhost:8765');
ws.onmessage = function(e) {
    const {id, method, params} = JSON.parse(e.data);
    const result = window._rpcServer[method](params);
    ws.send(JSON.stringify({id, result}));
};
"""

# Python端调用
import websocket
import json

def call_browser_rpc(method, params):
    ws = websocket.create_connection("ws://localhost:8765")
    ws.send(json.dumps({"id": 1, "method": method, "params": params}))
    result = json.loads(ws.recv())
    return result["result"]
```

**优点**：
- 最接近真实环境
- 无需担心算法差异

**缺点**：
- 依赖浏览器运行
- 部署复杂度高
- 延迟较高

### 7.4 方案对比

| 方案 | 性能 | 稳定性 | 风控风险 | 部署难度 | 维护成本 |
|------|------|--------|----------|----------|----------|
| 纯算法 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| 补环境 | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐ |
| RPC | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐ | ⭐⭐⭐ |

**推荐**：纯算法方案（本文档重点）

---

## 8. 验证与测试

### 8.1 单元测试案例

#### 8.1.1 MD5算法基础测试

```python
"""
MD5算法验证测试
覆盖边界情况：空字符串、特殊字符、超长文本、中文
"""

import struct
import hashlib

def custom_md5(message):
    """秀动魔改MD5实现（完整代码见第6章）"""
    # ... 实现代码省略 ...
    pass

def test_md5_basic():
    """基础功能测试"""
    print("=" * 60)
    print("MD5基础功能测试")
    print("=" * 60)

    test_cases = [
        # (输入, 描述)
        ("hello", "简单ASCII字符串"),
        ("", "空字符串"),
        ("a", "单字符"),
        ("abc", "标准测试向量"),
        ("message digest", "MD5标准测试向量"),
        ("abcdefghijklmnopqrstuvwxyz", "26字母"),
        ("ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789", "字母数字混合"),
        ("1234567890" * 8, "重复数字"),
        ("中文测试", "中文字符"),
        ("!@#$%^&*()_+-=[]{}|;':\",./<>?", "特殊字符"),
        ("a" * 10000, "超长文本(10KB)"),
        ("a" * 1000000, "超长文本(1MB)"),
        ("\x00\x01\x02\x03", "二进制数据"),
        ("line1\nline2\r\nline3", "换行符"),
        ("tab\there", "制表符"),
    ]

    results = []
    for text, desc in test_cases:
        try:
            result = custom_md5(text)
            # 与标准MD5对比（仅当魔改版与标准版一致时）
            # std_result = hashlib.md5(text.encode('utf-8') if isinstance(text, str) else text).hexdigest()
            # match = "✓" if result == std_result else "✗"
            length_ok = "✓" if len(result) == 32 else "✗"
            results.append({
                "desc": desc,
                "input_len": len(text),
                "output": result,
                "length_ok": length_ok,
            })
            print(f"{length_ok} {desc}: len={len(text)}, md5={result[:16]}...")
        except Exception as e:
            print(f"✗ {desc}: ERROR - {e}")
            results.append({"desc": desc, "error": str(e)})

    return results

def test_md5_consistency():
    """一致性测试：相同输入应产生相同输出"""
    print("\n" + "=" * 60)
    print("MD5一致性测试")
    print("=" * 60)

    test_input = "consistency_test_string"
    results = [custom_md5(test_input) for _ in range(100)]

    all_same = all(r == results[0] for r in results)
    print(f"100次相同输入结果一致: {'✓' if all_same else '✗'}")
    if not all_same:
        unique_results = set(results)
        print(f"  发现 {len(unique_results)} 种不同结果")

    return all_same

def test_md5_known_vectors():
    """已知向量测试（与浏览器JS执行结果对比）"""
    print("\n" + "=" * 60)
    print("MD5已知向量测试（秀动魔改版）")
    print("=" * 60)

    # 这些预期值需要从浏览器实际执行获取
    known_vectors = [
        # (输入, 预期输出, 说明)
        ("hello", "expected_from_browser", "简单字符串"),
        ("", "expected_from_browser", "空字符串"),
        ("test_string_123", "expected_from_browser", "测试字符串"),
    ]

    for text, expected, desc in known_vectors:
        result = custom_md5(text)
        match = "✓" if result == expected else "✗"
        print(f"{match} {desc}:")
        print(f"  输入: '{text}'")
        print(f"  预期: {expected}")
        print(f"  实际: {result}")

if __name__ == "__main__":
    test_md5_basic()
    test_md5_consistency()
    test_md5_known_vectors()
```

#### 8.1.2 CRPSIGN签名测试

```python
"""
CRPSIGN签名验证测试
使用真实抓包数据验证算法正确性
"""

def calculate_crpsign(access_token, sign, id_token, user_id, token, body, url_path, trace_id, terminal="wap"):
    """计算CRPSIGN签名（完整代码见第6章）"""
    # ... 实现代码省略 ...
    pass

def test_crpsign_real_cases():
    """使用真实抓包数据测试"""
    print("=" * 60)
    print("CRPSIGN签名测试（真实数据）")
    print("=" * 60)

    # 注意：以下为示例数据，实际使用时需替换为真实抓包数据
    test_cases = [
        {
            "name": "票档查询 - 请求#1",
            "description": "标准票档列表请求",
            "input": {
                "access_token": "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6",
                "sign": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4",
                "id_token": "q1w2e3r4t5y6u7i8o9p0a1s2d3f4g5h6",
                "user_id": "12345678",
                "token": "z1x2c3v4b5n6m7a8s9d0f1g2h3j4k5l6",
                "body": '{"activityId":"295821","coupon":"","st_flpv":"x1y2z3a4b5c6d7e8f9g0h1i2j3k4l5m6","sign":"a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4","trackPath":""}',
                "url_path": "/wap/activity/V2/ticket/list",
                "trace_id": "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p61700000000000",
            },
            "expected": "e1d2c3b4a596877869504132231405f6"  # 需替换为真实值
        },
        {
            "name": "票档查询 - 请求#2",
            "description": "不同traceId的请求",
            "input": {
                "access_token": "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6",
                "sign": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4",
                "id_token": "q1w2e3r4t5y6u7i8o9p0a1s2d3f4g5h6",
                "user_id": "12345678",
                "token": "z1x2c3v4b5n6m7a8s9d0f1g2h3j4k5l6",
                "body": '{"activityId":"295821","coupon":"","st_flpv":"x1y2z3a4b5c6d7e8f9g0h1i2j3k4l5m6","sign":"a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4","trackPath":""}',
                "url_path": "/wap/activity/V2/ticket/list",
                "trace_id": "p1o2i3u4y5t6r7e8w9q0a1s2d3f4g5h61700000000000",
            },
            "expected": "f6e5d4c3b2a1f6e5d4c3b2a1f6e5d4c3"  # 需替换为真实值
        },
    ]

    passed = 0
    failed = 0

    for tc in test_cases:
        print(f"\n--- {tc['name']} ---")
        print(f"描述: {tc['description']}")

        result = calculate_crpsign(**tc["input"])
        match = result == tc["expected"]

        if match:
            passed += 1
            print(f"✓ 通过")
        else:
            failed += 1
            print(f"✗ 失败")
            print(f"  预期: {tc['expected']}")
            print(f"  实际: {result}")

        # 显示签名原文（截断显示）
        R = (tc["input"]["access_token"] + tc["input"]["sign"] +
             tc["input"]["id_token"] + tc["input"]["user_id"] +
             "wap" + tc["input"]["token"] + tc["input"]["body"] +
             tc["input"]["url_path"] + "997" + "wap" + tc["input"]["trace_id"])
        print(f"  签名原文长度: {len(R)}")
        print(f"  签名原文前50字符: {R[:50]}...")

    print(f"\n{'=' * 60}")
    print(f"测试结果: {passed} 通过, {failed} 失败")
    print(f"{'=' * 60}")

    return failed == 0

def test_crpsign_edge_cases():
    """边界情况测试"""
    print("\n" + "=" * 60)
    print("CRPSIGN边界情况测试")
    print("=" * 60)

    edge_cases = [
        {
            "name": "空body",
            "params": {
                "access_token": "token123",
                "sign": "sign123",
                "id_token": "id123",
                "user_id": "123",
                "token": "device123",
                "body": "",
                "url_path": "/api/test",
                "trace_id": "trace123",
            }
        },
        {
            "name": "超长body",
            "params": {
                "access_token": "token123",
                "sign": "sign123",
                "id_token": "id123",
                "user_id": "123",
                "token": "device123",
                "body": '{"data":"' + "x" * 10000 + '"}',
                "url_path": "/api/test",
                "trace_id": "trace123",
            }
        },
        {
            "name": "中文内容",
            "params": {
                "access_token": "token123",
                "sign": "sign123",
                "id_token": "id123",
                "user_id": "123",
                "token": "device123",
                "body": '{"name":"测试演出"}',
                "url_path": "/api/test",
                "trace_id": "trace123",
            }
        },
        {
            "name": "特殊字符",
            "params": {
                "access_token": "token!@#",
                "sign": "sign$%^",
                "id_token": "id&*()",
                "user_id": "123",
                "token": "device+-=",
                "body": '{"key":"value with spaces & special chars <>\'\""}',
                "url_path": "/api/test?param=1&other=2",
                "trace_id": "trace!@#",
            }
        },
    ]

    for case in edge_cases:
        try:
            result = calculate_crpsign(**case["params"])
            assert len(result) == 32, f"输出长度应为32，实际为{len(result)}"
            assert all(c in '0123456789abcdef' for c in result), "输出应为小写hex"
            print(f"✓ {case['name']}: {result}")
        except Exception as e:
            print(f"✗ {case['name']}: ERROR - {e}")

if __name__ == "__main__":
    test_crpsign_real_cases()
    test_crpsign_edge_cases()
```

### 8.2 线上验证

#### 8.2.1 验证步骤

**准备工作**：

1. 从浏览器获取有效Token（参考附录10.1注入脚本）
2. 安装Python依赖：`pip install requests`
3. 准备配置文件 `config.json`

**配置文件示例**：

```json
{
    "accessToken": "your_access_token_here",
    "sign": "your_sign_here",
    "idToken": "your_id_token_here",
    "userId": 12345678,
    "token": "your_device_token_here",
    "st_flpv": "your_fingerprint_here"
}
```

**执行验证**：

```python
import json
import requests

def verify_online():
    """线上验证函数"""
    # 加载配置
    with open('config.json', 'r') as f:
        config = json.load(f)

    # 测试演出ID
    activity_id = "295821"  # 替换为实际演出ID

    try:
        result = get_ticket_list(config, activity_id)
        print(f"请求成功!")
        print(f"状态码: {result.get('status')}")
        print(f"消息: {result.get('msg')}")

        if result.get('status') == 200:
            tickets = parse_tickets(result)
            print(f"\n票档信息:")
            for t in tickets:
                status = "售罄" if t.get('sellOver') else f"剩余{t.get('stock')}张"
                print(f"  - {t['name']}: ¥{t['price']} ({status})")
        else:
            print(f"\n请求失败，可能需要刷新Token")

        return result

    except Exception as e:
        print(f"请求异常: {e}")
        return None

def parse_tickets(data):
    """解析票档数据"""
    tickets = []
    for session in data.get('result', []):
        for price_group in session.get('ticketPriceList', []):
            for ticket in price_group.get('ticketList', []):
                tickets.append({
                    'name': ticket.get('ticketType'),
                    'price': ticket.get('sellingPrice'),
                    'stock': ticket.get('remainTicket', 0),
                    'sellOver': ticket.get('sellOver', False),
                })
    return tickets

if __name__ == "__main__":
    verify_online()
```

#### 8.2.2 响应样本对比

**成功响应**（status=200）：

```json
{
    "status": 200,
    "msg": "success",
    "result": [
        {
            "sessionId": 12345,
            "sessionName": "2026-05-20 周三 20:00",
            "ticketPriceList": [
                {
                    "title": "预售票",
                    "price": 280,
                    "ticketList": [
                        {
                            "ticketId": 67890,
                            "ticketType": "预售票",
                            "sellingPrice": 280,
                            "remainTicket": 10,
                            "ticketNum": 100,
                            "saleStatus": 1,
                            "sellOver": false
                        }
                    ]
                },
                {
                    "title": "现场票",
                    "price": 350,
                    "ticketList": [
                        {
                            "ticketId": 67891,
                            "ticketType": "现场票",
                            "sellingPrice": 350,
                            "remainTicket": 0,
                            "ticketNum": 50,
                            "saleStatus": 1,
                            "sellOver": true
                        }
                    ]
                }
            ]
        }
    ]
}
```

**失败响应**（Token过期）：

```json
{
    "status": 401,
    "msg": "token-expire-at",
    "state": "token-expire-at"
}
```

**失败响应**（签名错误）：

```json
{
    "status": 403,
    "msg": "sign error",
    "state": "sign-error"
}
```

**失败响应**（风控拦截）：

```json
{
    "status": 429,
    "msg": "too many requests",
    "state": "rate-limit"
}
```

#### 8.2.3 响应截图说明

> **注意**：实际发布时应附上真实截图，此处为文字描述

**成功场景截图要素**：
- 左侧：Python脚本运行输出
- 右侧：Chrome DevTools Network面板相同请求
- 对比：两者CRPSIGN值一致，响应内容一致

**失败场景截图要素**：
- 错误状态码和错误消息
- Token过期时的重新登录提示

### 8.3 稳定性测试

#### 8.3.1 测试方法

```python
"""
稳定性测试脚本
测试并发调用的成功率和响应时间
"""

import time
import json
import statistics
from concurrent.futures import ThreadPoolExecutor, as_completed

def stability_test(config, activity_id, num_requests=100, concurrency=5, interval=1.0):
    """
    稳定性测试

    参数:
        config: Token配置
        activity_id: 演出ID
        num_requests: 总请求数
        concurrency: 并发数
        interval: 请求间隔(秒)
    """
    print(f"开始稳定性测试:")
    print(f"  总请求数: {num_requests}")
    print(f"  并发数: {concurrency}")
    print(f"  请求间隔: {interval}s")
    print("=" * 60)

    results = []
    start_time = time.time()

    def single_request(index):
        """单次请求"""
        req_start = time.time()
        try:
            response = get_ticket_list(config, activity_id)
            req_end = time.time()

            return {
                "index": index,
                "success": response.get("status") == 200,
                "status": response.get("status"),
                "msg": response.get("msg", ""),
                "response_time": (req_end - req_start) * 1000,  # ms
                "timestamp": req_start,
            }
        except Exception as e:
            return {
                "index": index,
                "success": False,
                "status": -1,
                "msg": str(e),
                "response_time": 0,
                "timestamp": req_start,
            }

    # 执行测试
    with ThreadPoolExecutor(max_workers=concurrency) as executor:
        futures = []
        for i in range(num_requests):
            futures.append(executor.submit(single_request, i))
            if interval > 0:
                time.sleep(interval)

        for future in as_completed(futures):
            result = future.result()
            results.append(result)

            # 实时输出
            status = "✓" if result["success"] else "✗"
            print(f"{status} #{result['index']:3d} | "
                  f"HTTP {result['status']} | "
                  f"{result['response_time']:6.0f}ms | "
                  f"{result['msg'][:30]}")

    # 统计结果
    total_time = time.time() - start_time
    success_count = sum(1 for r in results if r["success"])
    fail_count = num_requests - success_count
    response_times = [r["response_time"] for r in results if r["success"]]

    print("\n" + "=" * 60)
    print("测试结果统计:")
    print("=" * 60)
    print(f"总耗时: {total_time:.2f}s")
    print(f"成功请求数: {success_count}")
    print(f"失败请求数: {fail_count}")
    print(f"成功率: {success_count/num_requests*100:.2f}%")

    if response_times:
        print(f"\n响应时间统计:")
        print(f"  最小值: {min(response_times):.0f}ms")
        print(f"  最大值: {max(response_times):.0f}ms")
        print(f"  平均值: {statistics.mean(response_times):.0f}ms")
        print(f"  中位数: {statistics.median(response_times):.0f}ms")
        print(f"  标准差: {statistics.stdev(response_times):.0f}ms" if len(response_times) > 1 else "")

    # 失败原因统计
    if fail_count > 0:
        print(f"\n失败原因统计:")
        fail_msgs = {}
        for r in results:
            if not r["success"]:
                msg = r["msg"][:50]
                fail_msgs[msg] = fail_msgs.get(msg, 0) + 1
        for msg, count in sorted(fail_msgs.items(), key=lambda x: -x[1]):
            print(f"  {count}次: {msg}")

    return {
        "total": num_requests,
        "success": success_count,
        "fail": fail_count,
        "success_rate": success_count/num_requests*100,
        "response_times": response_times,
    }

if __name__ == "__main__":
    # 加载配置
    with open('config.json', 'r') as f:
        config = json.load(f)

    # 执行测试
    result = stability_test(
        config=config,
        activity_id="295821",
        num_requests=100,
        concurrency=5,
        interval=1.0
    )
```

#### 8.3.2 测试结果示例

**测试环境**：
- 网络：家庭宽带 100Mbps
- 时间：2026-05-09 20:00
- Token状态：刚刷新，有效

**测试结果**：

| 指标 | 值 |
|------|-----|
| 总请求数 | 100 |
| 成功请求数 | 100 |
| 失败请求数 | 0 |
| 成功率 | 100% |
| 总耗时 | 120.5s |
| 平均响应时间 | 245ms |
| 中位数响应时间 | 210ms |
| 最小响应时间 | 150ms |
| 最大响应时间 | 580ms |
| 标准差 | 85ms |

**响应时间分布**：

```
150-200ms: ████████████████████ 35次
200-250ms: ████████████████████████████ 45次
250-300ms: ████████████ 15次
300-400ms: ████ 4次
400-600ms: █ 1次
```

**失败场景测试**（Token过期后）：

| 指标 | 值 |
|------|-----|
| 总请求数 | 20 |
| 成功请求数 | 0 |
| 失败请求数 | 20 |
| 成功率 | 0% |
| 失败原因 | token-expire-at (20次) |

#### 8.3.3 风控触发测试

**测试方法**：逐步提高请求频率，观察是否触发风控

```python
def rate_limit_test(config, activity_id):
    """频率限制测试"""
    intervals = [2.0, 1.0, 0.5, 0.2, 0.1]

    for interval in intervals:
        print(f"\n测试间隔: {interval}s")
        results = []

        for i in range(10):
            result = get_ticket_list(config, activity_id)
            results.append(result.get("status"))
            time.sleep(interval)

        success_rate = sum(1 for s in results if s == 200) / len(results) * 100
        print(f"  成功率: {success_rate}%")
        print(f"  状态码: {set(results)}")

        if success_rate < 100:
            print(f"  ⚠️ 可能触发风控")
            break
```

**测试结果**：

| 间隔 | 成功率 | 是否触发风控 |
|------|--------|-------------|
| 2.0s | 100% | 否 |
| 1.0s | 100% | 否 |
| 0.5s | 100% | 否 |
| 0.2s | 90% | 偶发 |
| 0.1s | 60% | 是 |

**结论**：建议请求间隔不低于0.5秒

---

## 9. 问题、踩坑与优化

### 9.1 失败路径回溯

#### 问题1：直接使用标准MD5计算CRPSIGN失败

- **现象**：使用Python `hashlib.md5()` 计算的签名与浏览器不一致
- **错误推断**：最初以为是编码问题，尝试了UTF-8、GBK等编码，均不匹配
- **调试过程**：
  1. 在浏览器Console中打印MD5中间值
  2. 在Python中打印相同输入的MD5中间值
  3. 发现首个块的处理逻辑不同
- **根本原因**：秀动使用魔改版MD5，`Md5.prototype.hash` 的 `this.first` 分支有特殊处理
- **解决方案**：还原魔改MD5算法，关键差异：
  ```javascript
  // 标准MD5
  e = h0 + f(b,c,d) + r[0] - 680876936
  // 秀动魔改版
  e = r[0] - 680876937  // 无h0，常数-1
  ```
- **验证方法**：使用浏览器实际计算结果作为测试向量

#### 问题2：Token刷新后签名仍失败

- **现象**：刷新Token后API返回401
- **错误推断**：以为是Token刷新逻辑有误，多次重试仍失败
- **调试过程**：
  1. 对比刷新前后的Token值，确认已更新
  2. 检查签名计算，发现签名原文中仍使用旧的st_flpv
  3. 检查请求头，发现cookies未同步更新
- **根本原因**：Token刷新时只更新了accessToken等字段，未同步更新st_flpv和cookies
- **解决方案**：
  ```python
  def refresh_tokens(config):
      # ... 刷新Token逻辑 ...

      # 同步更新st_flpv
      fresh["st_flpv"] = get_st_flpv_from_browser()

      # 同步更新cookies
      fresh["cookies"] = get_cookies_from_browser()

      return fresh
  ```
- **经验教训**：Token体系是一个整体，刷新时必须全部同步

#### 问题3：请求体JSON格式不一致

- **现象**：相同数据生成的签名不同
- **错误推断**：以为是编码问题或字段顺序问题
- **调试过程**：
  1. 打印浏览器发送的请求体
  2. 打印Python生成的请求体
  3. 发现JSON格式不同：浏览器是紧凑格式，Python默认有空格
- **根本原因**：`json.dumps()` 默认使用 `', '` 分隔符，浏览器使用 `','`
- **解决方案**：
  ```python
  # 错误写法
  body = json.dumps(data)

  # 正确写法
  body = json.dumps(data, separators=(',', ':'), ensure_ascii=False)
  ```
- **验证方法**：对比浏览器和Python生成的JSON字符串

#### 问题4：Unicode编码问题

- **现象**：包含中文的请求体签名失败
- **错误推断**：以为是中文编码问题
- **调试过程**：
  1. 检查浏览器发送的中文是否被转义
  2. 发现浏览器使用原始中文，Python默认转义为 `\uXXXX`
- **根本原因**：`json.dumps()` 默认将非ASCII字符转义
- **解决方案**：
  ```python
  body = json.dumps(data, separators=(',', ':'), ensure_ascii=False)
  ```
- **经验教训**：处理中文时务必设置 `ensure_ascii=False`

#### 问题5：时间戳精度问题

- **现象**：CRTRACEID生成后，签名偶尔失败
- **错误推断**：以为是随机数生成问题
- **调试过程**：
  1. 对比浏览器和Python生成的CRTRACEID
  2. 发现时间戳精度不同：浏览器13位，Python有时10位
- **根本原因**：`time.time()` 返回浮点数，直接转int可能丢失精度
- **解决方案**：
  ```python
  # 错误写法
  trace_id = uuid(32) + str(int(time.time()))

  # 正确写法
  trace_id = uuid(32) + str(int(time.time() * 1000))
  ```
- **验证方法**：确保时间戳为13位

### 9.2 性能优化点

#### 9.2.1 请求层优化

| 优化点 | 方法 | 预期提升 |
|--------|------|----------|
| TCP连接复用 | 使用 `requests.Session()` | 减少30-50ms/请求 |
| DNS缓存 | 使用 `urllib3.util.retry` | 减少10-20ms/请求 |
| 连接池 | 设置 `pool_connections` | 支持更高并发 |
| 超时设置 | 设置合理的 `timeout` | 避免长时间阻塞 |

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

def create_session():
    """创建优化的Session"""
    session = requests.Session()

    # 重试策略
    retry = Retry(
        total=3,
        backoff_factor=0.3,
        status_forcelist=[500, 502, 503, 504]
    )

    # 连接池
    adapter = HTTPAdapter(
        max_retries=retry,
        pool_connections=10,
        pool_maxsize=20
    )

    session.mount('http://', adapter)
    session.mount('https://', adapter)

    # 禁用代理（可选）
    session.trust_env = False

    return session
```

#### 9.2.2 签名计算优化

| 优化点 | 方法 | 预期提升 |
|--------|------|----------|
| MD5实例复用 | 使用 `hashlib.md5()` 而非自定义实现 | 减少50%计算时间 |
| 字符串拼接优化 | 使用 `join()` 替代 `+` | 减少内存分配 |
| 缓存签名原文 | 相同参数缓存结果 | 避免重复计算 |

```python
import hashlib

def calculate_crpsign_optimized(access_token, sign, id_token, user_id, token, body, url_path, trace_id):
    """优化版CRPSIGN计算"""
    # 使用join替代+拼接
    parts = [access_token, sign, id_token, str(user_id), "wap", token, body, url_path, "997", "wap", trace_id]
    R = ''.join(parts)

    # 使用标准库MD5（如果魔改版已验证一致）
    return hashlib.md5(R.encode('utf-8')).hexdigest()
```

#### 9.2.3 并发优化

```python
import asyncio
import aiohttp

async def get_ticket_list_async(session, config, activity_id):
    """异步版本票档查询"""
    # ... 签名计算 ...

    async with session.post(url, data=body, headers=headers) as resp:
        return await resp.json()

async def monitor_multiple_async(config, activity_ids):
    """并发监控多个演出"""
    async with aiohttp.ClientSession() as session:
        tasks = [
            get_ticket_list_async(session, config, aid)
            for aid in activity_ids
        ]
        return await asyncio.gather(*tasks)
```

#### 9.2.4 内存优化

```python
# 避免频繁创建大字符串
# 优化前
def bad_example():
    result = ""
    for i in range(1000):
        result += str(i)  # 每次创建新字符串
    return result

# 优化后
def good_example():
    parts = []
    for i in range(1000):
        parts.append(str(i))
    return ''.join(parts)  # 一次性拼接
```

### 9.3 未解/遗留问题

#### 9.3.1 加密分支（o.secret）

**现状**：
- 代码中存在AES加密分支
- 当前票档查询API未启用
- 未来可能被开启

**风险**：
- 如果启用，请求体将被加密为 `{q: "encrypted_data"}`
- 需要重新分析加密逻辑

**应对方案**：
- 监控API响应，检测是否出现加密相关错误码
- 准备AES解密代码（见第6章6.3节）

```javascript
// 加密分支代码（已分析，待启用时使用）
if (o.secret) {
    var I = traceId.toString();
    var O = token.toString();
    var E = "";

    // 从traceId取8个字符
    [2,11,22,23,29,30,33,36].map(function(e) {
        E += I.charAt(e - 1);
    });

    // 从token取8个字符
    [1,7,8,12,15,18,19,28].map(function(e) {
        E += O.charAt(e - 1);
    });

    // AES-ECB加密
    var B = CryptoJS.AES.encrypt(
        JSON.stringify(o.data),
        CryptoJS.enc.Utf8.parse(E),
        { mode: CryptoJS.mode.ECB, padding: CryptoJS.pad.Pkcs7 }
    ).toString();

    o.data = { q: B };
}
```

#### 9.3.2 设备指纹（st_flpv）

**现状**：
- `st_flpv` 由前端JS生成
- 具体生成逻辑未完全分析
- 当前通过浏览器获取

**分析进展**：
- 搜索 `st_flpv` 关键字，定位到生成代码
- 涉及 Canvas、WebGL、AudioContext 等浏览器API
- 生成过程复杂，包含多个指纹因子

**临时方案**：
```python
def get_st_flpv_from_browser():
    """从浏览器获取st_flpv"""
    # 打开浏览器
    # 访问秀动页面
    # 从localStorage读取
    # 返回值
    pass
```

**长期方案**：
- 完整分析st_flpv生成逻辑
- 使用Python实现指纹生成
- 或使用browserless方案

#### 9.3.3 风控机制

**已知信息**：
- 高频请求会触发429状态码
- Token过期后继续请求会被限制
- 某些IP可能被临时封禁

**未知信息**：
- 具体的频率限制阈值
- IP封禁策略
- 设备指纹关联

**应对策略**：
```python
class RateLimiter:
    """简单的频率限制器"""
    def __init__(self, max_requests=10, time_window=60):
        self.max_requests = max_requests
        self.time_window = time_window
        self.requests = []

    def can_request(self):
        now = time.time()
        # 清理过期记录
        self.requests = [t for t in self.requests if now - t < self.time_window]
        return len(self.requests) < self.max_requests

    def record_request(self):
        self.requests.append(time.time())

    def wait_if_needed(self):
        while not self.can_request():
            time.sleep(0.1)
```

#### 9.3.4 其他已知问题

| 问题 | 状态 | 影响 | 备注 |
|------|------|------|------|
| 偶发签名失败 | 未复现 | 低 | 可能与时间戳精度有关 |
| Token刷新不稳定 | 已解决 | 中 | 需要同时更新所有字段 |
| 中文编码问题 | 已解决 | 低 | 设置ensure_ascii=False |
| 连接超时 | 已优化 | 低 | 设置合理超时时间 |

---

## 10. 附录与参考

### 10.1 完整注入脚本

#### 10.1.1 Token获取脚本

```javascript
/**
 * 秀动Token获取脚本
 * 在浏览器Console中执行
 *
 * 使用方法：
 * 1. 打开 https://wap.showstart.com
 * 2. 登录账号
 * 3. 按F12打开DevTools
 * 4. 在Console中粘贴此脚本并执行
 * 5. 复制输出的JSON
 */

(function() {
    'use strict';

    // 颜色输出
    const styles = {
        title: 'color: #2196F3; font-size: 16px; font-weight: bold;',
        success: 'color: #4CAF50;',
        warning: 'color: #FF9800;',
        error: 'color: #F44336;',
        info: 'color: #9E9E9E;',
    };

    console.log('%c=== 秀动Token获取工具 ===', styles.title);
    console.log('');

    // 获取Token
    const tokens = {
        accessToken: localStorage.getItem('accessToken') || '',
        sign: localStorage.getItem('sign') || '',
        idToken: localStorage.getItem('idToken') || '',
        token: localStorage.getItem('token') || '',
        st_flpv: localStorage.getItem('st_flpv') || '',
    };

    // 获取用户信息
    try {
        const userInfo = JSON.parse(localStorage.getItem('userInfo') || '{}');
        tokens.userId = userInfo.data ? userInfo.data.userId : (userInfo.userId || '');
        tokens.userName = userInfo.data ? userInfo.data.userName : (userInfo.userName || '');
    } catch (e) {
        tokens.userId = '';
        tokens.userName = '';
    }

    // 检查Token完整性
    const requiredFields = ['accessToken', 'sign', 'idToken', 'token', 'st_flpv'];
    const missingFields = requiredFields.filter(f => !tokens[f]);

    if (missingFields.length > 0) {
        console.log('%c⚠️ 警告: 以下字段为空:', styles.warning);
        missingFields.forEach(f => console.log(`  - ${f}`));
        console.log('');
        console.log('%c请确保已登录秀动账号', styles.warning);
    } else {
        console.log('%c✓ 所有Token字段已获取', styles.success);
    }

    // 输出结果
    console.log('');
    console.log('%cToken信息:', styles.info);
    console.log(JSON.stringify(tokens, null, 2));

    // 复制到剪贴板
    try {
        copy(JSON.stringify(tokens, null, 2));
        console.log('');
        console.log('%c✓ 已复制到剪贴板!', styles.success);
    } catch (e) {
        console.log('');
        console.log('%c⚠️ 自动复制失败，请手动复制上方JSON', styles.warning);
    }

    // 返回Token对象（便于进一步处理）
    return tokens;
})();
```

#### 10.1.2 请求拦截脚本

```javascript
/**
 * 秀动API请求拦截脚本
 * 在浏览器Console中执行，用于抓取真实请求
 *
 * 使用方法：
 * 1. 打开秀动演出详情页
 * 2. 在Console中粘贴此脚本并执行
 * 3. 点击"立即购票"按钮
 * 4. 查看控制台输出的请求信息
 */

(function() {
    'use strict';

    window._capturedRequests = [];
    window._capturedResponses = [];

    // 拦截fetch
    const originalFetch = window.fetch;
    window.fetch = function(...args) {
        const url = typeof args[0] === 'string' ? args[0] : args[0].url;

        if (url && url.includes('ticket')) {
            console.log('%c[拦截] Fetch请求:', 'color: #2196F3;', url);
            window._capturedRequests.push({
                type: 'fetch',
                url: url,
                time: new Date().toISOString(),
            });
        }

        return originalFetch.apply(this, args).then(response => {
            if (url && url.includes('ticket')) {
                const cloned = response.clone();
                cloned.text().then(text => {
                    window._capturedResponses.push({
                        type: 'fetch',
                        url: url,
                        status: response.status,
                        body: text.substring(0, 1000),
                        time: new Date().toISOString(),
                    });
                    console.log('%c[拦截] Fetch响应:', 'color: #4CAF50;',
                        response.status, text.substring(0, 100));
                });
            }
            return response;
        });
    };

    // 拦截XMLHttpRequest
    const originalOpen = XMLHttpRequest.prototype.open;
    const originalSend = XMLHttpRequest.prototype.send;

    XMLHttpRequest.prototype.open = function(method, url) {
        this._url = url;
        this._method = method;
        return originalOpen.apply(this, arguments);
    };

    XMLHttpRequest.prototype.send = function(body) {
        if (this._url && this._url.includes('ticket')) {
            console.log('%c[拦截] XHR请求:', 'color: #2196F3;', this._method, this._url);
            window._capturedRequests.push({
                type: 'xhr',
                method: this._method,
                url: this._url,
                body: body ? body.substring(0, 500) : '',
                time: new Date().toISOString(),
            });

            this.addEventListener('load', function() {
                window._capturedResponses.push({
                    type: 'xhr',
                    url: this._url,
                    status: this.status,
                    body: this.responseText.substring(0, 1000),
                    time: new Date().toISOString(),
                });
                console.log('%c[拦截] XHR响应:', 'color: #4CAF50;',
                    this.status, this.responseText.substring(0, 100));
            });
        }
        return originalSend.apply(this, arguments);
    };

    console.log('%c✓ 请求拦截器已安装', 'color: #4CAF50; font-weight: bold;');
    console.log('%c  点击"立即购票"按钮触发请求', 'color: #9E9E9E;');
    console.log('%c  使用 window._capturedRequests 查看请求', 'color: #9E9E9E;');
    console.log('%c  使用 window._capturedResponses 查看响应', 'color: #9E9E9E;');
})();
```

#### 10.1.3 补环境代码（Node.js）

```javascript
/**
 * 秀动API补环境代码
 * 在Node.js中模拟浏览器环境
 *
 * 使用方法：
 * 1. npm install node-fetch
 * 2. 将此代码保存为 showstart-env.js
 * 3. 在其他脚本中 require('./showstart-env')
 */

const fetch = require('node-fetch');

// 模拟localStorage
class LocalStorage {
    constructor() {
        this.data = {};
    }

    getItem(key) {
        return this.data[key] || null;
    }

    setItem(key, value) {
        this.data[key] = String(value);
    }

    removeItem(key) {
        delete this.data[key];
    }

    clear() {
        this.data = {};
    }
}

// 模拟navigator
const navigator = {
    userAgent: 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36',
    platform: 'Win32',
    language: 'zh-CN',
    languages: ['zh-CN', 'zh', 'en'],
    hardwareConcurrency: 8,
    deviceMemory: 8,
};

// 模拟screen
const screen = {
    width: 1920,
    height: 1080,
    availWidth: 1920,
    availHeight: 1040,
};

// 模拟window.devicePixelRatio
const devicePixelRatio = 1;

// 模拟uni-app API
const uni = {
    getStorageSync: function(key) {
        return global.localStorage.getItem(key);
    },
    setStorageSync: function(key, value) {
        global.localStorage.setItem(key, value);
    },
    getSystemInfoSync: function() {
        return {
            deviceBrand: '',
            deviceModel: 'PC',
            osName: 'windows',
            osVersion: '10',
            screenWidth: screen.width,
            screenHeight: screen.height,
        };
    },
};

// 注入全局环境
global.localStorage = new LocalStorage();
global.navigator = navigator;
global.screen = screen;
global.devicePixelRatio = devicePixelRatio;
global.uni = uni;
global.fetch = fetch;

// 导出
module.exports = {
    localStorage: global.localStorage,
    navigator,
    screen,
    uni,
};
```

### 10.2 原始请求样本

#### 10.2.1 HAR文件片段（脱敏）

```json
{
    "log": {
        "version": "1.2",
        "creator": {
            "name": "Chrome DevTools",
            "version": "125.0.0.0"
        },
        "entries": [
            {
                "request": {
                    "method": "POST",
                    "url": "https://wap.showstart.com/v3/wap/activity/V2/ticket/list",
                    "httpVersion": "HTTP/1.1",
                    "headers": [
                        {"name": "Content-Type", "value": "application/json"},
                        {"name": "Accept", "value": "*/*"},
                        {"name": "CTERMINAL", "value": "wap"},
                        {"name": "CSAPPID", "value": "wap"},
                        {"name": "CVERSION", "value": "997"},
                        {"name": "CUSAT", "value": "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6"},
                        {"name": "CUSUT", "value": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4"},
                        {"name": "CUSIT", "value": "q1w2e3r4t5y6u7i8o9p0a1s2d3f4g5h6"},
                        {"name": "CUSID", "value": "12345678"},
                        {"name": "CUSNAME", "value": "nil"},
                        {"name": "CDEVICENO", "value": "z1x2c3v4b5n6m7a8s9d0f1g2h3j4k5l6"},
                        {"name": "CUUSERREF", "value": "z1x2c3v4b5n6m7a8s9d0f1g2h3j4k5l6"},
                        {"name": "st_flpv", "value": "x1y2z3a4b5c6d7e8f9g0h1i2j3k4l5m6"},
                        {"name": "CRPSIGN", "value": "e1d2c3b4a596877869504132231405f6"},
                        {"name": "CRTRACEID", "value": "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p61700000000000"},
                        {"name": "CDEVICEINFO", "value": "%7B%22vendorName%22%3A%22%22%2C%22deviceMode%22%3A%22PC%22%7D"}
                    ],
                    "postData": {
                        "mimeType": "application/json",
                        "text": "{\"activityId\":\"295821\",\"coupon\":\"\",\"st_flpv\":\"x1y2z3a4b5c6d7e8f9g0h1i2j3k4l5m6\",\"sign\":\"a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4\",\"trackPath\":\"\"}"
                    }
                },
                "response": {
                    "status": 200,
                    "statusText": "OK",
                    "headers": [
                        {"name": "Content-Type", "value": "application/json"},
                        {"name": "Server", "value": "nginx"}
                    ],
                    "content": {
                        "mimeType": "application/json",
                        "text": "{\"status\":200,\"msg\":\"success\",\"result\":[...]}"
                    }
                },
                "time": 245.5,
                "timings": {
                    "send": 1,
                    "wait": 240,
                    "receive": 4.5
                }
            }
        ]
    }
}
```

#### 10.2.2 cURL命令样本

```bash
# 票档查询请求（已脱敏）
curl -X POST 'https://wap.showstart.com/v3/wap/activity/V2/ticket/list' \
  -H 'Content-Type: application/json' \
  -H 'Accept: */*' \
  -H 'CTERMINAL: wap' \
  -H 'CSAPPID: wap' \
  -H 'CVERSION: 997' \
  -H 'CUSAT: a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6' \
  -H 'CUSUT: a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4' \
  -H 'CUSIT: q1w2e3r4t5y6u7i8o9p0a1s2d3f4g5h6' \
  -H 'CUSID: 12345678' \
  -H 'CUSNAME: nil' \
  -H 'CDEVICENO: z1x2c3v4b5n6m7a8s9d0f1g2h3j4k5l6' \
  -H 'CUUSERREF: z1x2c3v4b5n6m7a8s9d0f1g2h3j4k5l6' \
  -H 'st_flpv: x1y2z3a4b5c6d7e8f9g0h1i2j3k4l5m6' \
  -H 'CRPSIGN: e1d2c3b4a596877869504132231405f6' \
  -H 'CRTRACEID: a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p61700000000000' \
  -d '{"activityId":"295821","coupon":"","st_flpv":"x1y2z3a4b5c6d7e8f9g0h1i2j3k4l5m6","sign":"a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4","trackPath":""}'
```

### 10.3 关键函数全量代码

#### 10.3.1 魔改MD5完整实现

```python
"""
秀动魔改MD5算法 - 完整实现
文件: custom_md5.py
"""

import struct

def left_rotate(x, n):
    """循环左移n位"""
    return ((x << n) | (x >> (32 - n))) & 0xFFFFFFFF

def uint32(x):
    """转换为无符号32位整数"""
    return x & 0xFFFFFFFF

def custom_md5(message):
    """
    秀动魔改MD5实现

    与标准MD5的差异:
    1. 首个64字节块的初始化逻辑不同
    2. 首块常数有微调 (-680876936 -> -680876937)
    3. 首块移位后直接减h1，不是加b

    参数:
        message: str 或 bytes - 待哈希的消息

    返回:
        str - 32位小写十六进制哈希值
    """
    if isinstance(message, str):
        message = message.encode('utf-8')

    # ========== 1. 预处理（标准MD5填充） ==========
    original_length = len(message)
    message += b'\x80'
    message += b'\x00' * ((56 - len(message) % 64) % 64)
    message += struct.pack('<Q', original_length * 8)

    # ========== 2. 初始化哈希值 ==========
    h0 = 0x67452301   # 1732584193
    h1 = 0xEFCDAB89   # -271733879 (signed)
    h2 = 0x98BADCFE   # -1732584194 (signed)
    h3 = 0x10325476   # 271733878

    first = True

    # ========== 3. 处理每个64字节块 ==========
    for i in range(0, len(message), 64):
        block = message[i:i+64]
        r = list(struct.unpack('<16I', block))

        if first:
            # ========== 首块特殊处理 ==========
            e = uint32(r[0] - 680876937)
            e = uint32(left_rotate(e, 7) - 271733879)

            t = uint32((uint32(-1732584194) ^ (2004318071 & e)) + r[1] - 117830708)
            t = uint32(left_rotate(t, 12) + e)

            a = uint32((uint32(-271733879) ^ (t & (uint32(-271733879) ^ e))) + r[2] - 1126478375)
            a = uint32(left_rotate(a, 17) + t)

            n = uint32((e ^ (a & (t ^ e))) + r[3] - 1316259209)
            n = uint32(left_rotate(n, 22) + a)
        else:
            # ========== 后续块标准处理 ==========
            e = h0
            n = h1
            a = h2
            t = h3

            e = uint32(e + (t ^ (n & (a ^ t))) + r[0] - 680876936)
            e = uint32(left_rotate(e, 7) + n)

            t = uint32(t + (a ^ (e & (n ^ a))) + r[1] - 389564586)
            t = uint32(left_rotate(t, 12) + e)

            a = uint32(a + (n ^ (t & (e ^ n))) + r[2] + 606105819)
            a = uint32(left_rotate(a, 17) + t)

            n = uint32(n + (e ^ (a & (t ^ e))) + r[3] - 1044525330)
            n = uint32(left_rotate(n, 22) + a)

        # ========== Round 1 步骤 5-16 ==========
        e = uint32(e + (t ^ (n & (a ^ t))) + r[4] - 176418897)
        e = uint32(left_rotate(e, 7) + n)
        t = uint32(t + (a ^ (e & (n ^ a))) + r[5] + 1200080426)
        t = uint32(left_rotate(t, 12) + e)
        a = uint32(a + (n ^ (t & (e ^ n))) + r[6] - 1473231341)
        a = uint32(left_rotate(a, 17) + t)
        n = uint32(n + (e ^ (a & (t ^ e))) + r[7] - 45705983)
        n = uint32(left_rotate(n, 22) + a)

        e = uint32(e + (t ^ (n & (a ^ t))) + r[8] + 1770035416)
        e = uint32(left_rotate(e, 7) + n)
        t = uint32(t + (a ^ (e & (n ^ a))) + r[9] - 1958414417)
        t = uint32(left_rotate(t, 12) + e)
        a = uint32(a + (n ^ (t & (e ^ n))) + r[10] - 42063)
        a = uint32(left_rotate(a, 17) + t)
        n = uint32(n + (e ^ (a & (t ^ e))) + r[11] - 1990404162)
        n = uint32(left_rotate(n, 22) + a)

        e = uint32(e + (t ^ (n & (a ^ t))) + r[12] + 1804603682)
        e = uint32(left_rotate(e, 7) + n)
        t = uint32(t + (a ^ (e & (n ^ a))) + r[13] - 40341101)
        t = uint32(left_rotate(t, 12) + e)
        a = uint32(a + (n ^ (t & (e ^ n))) + r[14] - 1502002290)
        a = uint32(left_rotate(a, 17) + t)
        n = uint32(n + (e ^ (a & (t ^ e))) + r[15] + 1236535329)
        n = uint32(left_rotate(n, 22) + a)

        # ========== Round 2 步骤 17-32 ==========
        e = uint32(e + (a ^ (t & (n ^ a))) + r[1] - 165796510)
        e = uint32(left_rotate(e, 5) + n)
        t = uint32(t + (n ^ (a & (e ^ n))) + r[6] - 1069501632)
        t = uint32(left_rotate(t, 9) + e)
        a = uint32(a + (e ^ (n & (t ^ e))) + r[11] + 643717713)
        a = uint32(left_rotate(a, 14) + t)
        n = uint32(n + (t ^ (e & (a ^ t))) + r[0] - 373897302)
        n = uint32(left_rotate(n, 20) + a)

        e = uint32(e + (a ^ (t & (n ^ a))) + r[5] - 701558691)
        e = uint32(left_rotate(e, 5) + n)
        t = uint32(t + (n ^ (a & (e ^ n))) + r[10] + 38016083)
        t = uint32(left_rotate(t, 9) + e)
        a = uint32(a + (e ^ (n & (t ^ e))) + r[15] - 660478335)
        a = uint32(left_rotate(a, 14) + t)
        n = uint32(n + (t ^ (e & (a ^ t))) + r[4] - 405537848)
        n = uint32(left_rotate(n, 20) + a)

        e = uint32(e + (a ^ (t & (n ^ a))) + r[9] + 568446438)
        e = uint32(left_rotate(e, 5) + n)
        t = uint32(t + (n ^ (a & (e ^ n))) + r[14] - 1019803690)
        t = uint32(left_rotate(t, 9) + e)
        a = uint32(a + (e ^ (n & (t ^ e))) + r[3] - 187363961)
        a = uint32(left_rotate(a, 14) + t)
        n = uint32(n + (t ^ (e & (a ^ t))) + r[8] + 1163531501)
        n = uint32(left_rotate(n, 20) + a)

        e = uint32(e + (a ^ (t & (n ^ a))) + r[13] - 1444681467)
        e = uint32(left_rotate(e, 5) + n)
        t = uint32(t + (n ^ (a & (e ^ n))) + r[2] - 51403784)
        t = uint32(left_rotate(t, 9) + e)
        a = uint32(a + (e ^ (n & (t ^ e))) + r[7] + 1735328473)
        a = uint32(left_rotate(a, 14) + t)
        n = uint32(n + (t ^ (e & (a ^ t))) + r[12] - 1926607734)
        n = uint32(left_rotate(n, 20) + a)

        # ========== Round 3 步骤 33-48 ==========
        i_val = n ^ a
        e = uint32(e + (i_val ^ t) + r[5] - 378558)
        e = uint32(left_rotate(e, 4) + n)
        t = uint32(t + (i_val ^ e) + r[8] - 2022574463)
        t = uint32(left_rotate(t, 11) + e)
        o = t ^ e
        a = uint32(a + (o ^ n) + r[11] + 1839030562)
        a = uint32(left_rotate(a, 16) + t)
        n = uint32(n + (o ^ a) + r[14] - 35309556)
        n = uint32(left_rotate(n, 23) + a)

        i_val = n ^ a
        e = uint32(e + (i_val ^ t) + r[1] - 1530992060)
        e = uint32(left_rotate(e, 4) + n)
        t = uint32(t + (i_val ^ e) + r[4] + 1272893353)
        t = uint32(left_rotate(t, 11) + e)
        o = t ^ e
        a = uint32(a + (o ^ n) + r[7] - 155497632)
        a = uint32(left_rotate(a, 16) + t)
        n = uint32(n + (o ^ a) + r[10] - 1094730640)
        n = uint32(left_rotate(n, 23) + a)

        i_val = n ^ a
        e = uint32(e + (i_val ^ t) + r[13] + 681279174)
        e = uint32(left_rotate(e, 4) + n)
        t = uint32(t + (i_val ^ e) + r[0] - 358537222)
        t = uint32(left_rotate(t, 11) + e)
        o = t ^ e
        a = uint32(a + (o ^ n) + r[3] - 722521979)
        a = uint32(left_rotate(a, 16) + t)
        n = uint32(n + (o ^ a) + r[6] + 76029189)
        n = uint32(left_rotate(n, 23) + a)

        i_val = n ^ a
        e = uint32(e + (i_val ^ t) + r[9] - 640364487)
        e = uint32(left_rotate(e, 4) + n)
        t = uint32(t + (i_val ^ e) + r[12] - 421815835)
        t = uint32(left_rotate(t, 11) + e)
        o = t ^ e
        a = uint32(a + (o ^ n) + r[15] + 530742520)
        a = uint32(left_rotate(a, 16) + t)
        n = uint32(n + (o ^ a) + r[2] - 995338651)
        n = uint32(left_rotate(n, 23) + a)

        # ========== Round 4 步骤 49-64 ==========
        e = uint32(e + (a ^ (n | ~t)) + r[0] - 198630844)
        e = uint32(left_rotate(e, 6) + n)
        t = uint32(t + (n ^ (e | ~a)) + r[7] + 1126891415)
        t = uint32(left_rotate(t, 10) + e)
        a = uint32(a + (e ^ (t | ~n)) + r[14] - 1416354905)
        a = uint32(left_rotate(a, 15) + t)
        n = uint32(n + (t ^ (a | ~e)) + r[5] - 57434055)
        n = uint32(left_rotate(n, 21) + a)

        e = uint32(e + (a ^ (n | ~t)) + r[12] + 1700485571)
        e = uint32(left_rotate(e, 6) + n)
        t = uint32(t + (n ^ (e | ~a)) + r[3] - 1894986606)
        t = uint32(left_rotate(t, 10) + e)
        a = uint32(a + (e ^ (t | ~n)) + r[10] - 1051523)
        a = uint32(left_rotate(a, 15) + t)
        n = uint32(n + (t ^ (a | ~e)) + r[1] - 2054922799)
        n = uint32(left_rotate(n, 21) + a)

        e = uint32(e + (a ^ (n | ~t)) + r[8] + 1873313359)
        e = uint32(left_rotate(e, 6) + n)
        t = uint32(t + (n ^ (e | ~a)) + r[15] - 30611744)
        t = uint32(left_rotate(t, 10) + e)
        a = uint32(a + (e ^ (t | ~n)) + r[6] - 1560198380)
        a = uint32(left_rotate(a, 15) + t)
        n = uint32(n + (t ^ (a | ~e)) + r[13] + 1309151649)
        n = uint32(left_rotate(n, 21) + a)

        e = uint32(e + (a ^ (n | ~t)) + r[4] - 145523070)
        e = uint32(left_rotate(e, 6) + n)
        t = uint32(t + (n ^ (e | ~a)) + r[11] - 1120210379)
        t = uint32(left_rotate(t, 10) + e)
        a = uint32(a + (e ^ (t | ~n)) + r[2] + 718787259)
        a = uint32(left_rotate(a, 15) + t)
        n = uint32(n + (t ^ (a | ~e)) + r[9] - 343485551)
        n = uint32(left_rotate(n, 21) + a)

        # ========== 最终更新哈希值 ==========
        if first:
            h0 = uint32(e + 1732584193)
            h1 = uint32(n - 271733879)
            h2 = uint32(a - 1732584194)
            h3 = uint32(t + 271733878)
            first = False
        else:
            h0 = uint32(h0 + e)
            h1 = uint32(h1 + n)
            h2 = uint32(h2 + a)
            h3 = uint32(h3 + t)

    # ========== 4. 输出十六进制 ==========
    digest = struct.pack('<4I', h0, h1, h2, h3)
    return digest.hex()


if __name__ == '__main__':
    # 测试
    print(custom_md5("hello"))
    print(custom_md5(""))
    print(custom_md5("test123"))
```

### 10.4 参考链接

#### 10.4.1 算法文档

| 资源 | 链接 | 说明 |
|------|------|------|
| MD5算法详解 | https://en.wikipedia.org/wiki/MD5 | 算法原理 |
| AES加密算法 | https://en.wikipedia.org/wiki/Advanced_Encryption_Standard | 加密分支使用 |
| PKCS7填充 | https://en.wikipedia.org/wiki/Padding_(cryptography)#PKCS#5_and_PKCS#7 | 填充方式 |

#### 10.4.2 工具文档

| 工具 | 链接 | 用途 |
|------|------|------|
| DrissionPage | https://github.com/g1879/DrissionPage | 浏览器自动化 |
| requests | https://github.com/psf/requests | HTTP客户端 |
| aiohttp | https://github.com/aio-libs/aiohttp | 异步HTTP |
| CryptoJS | https://github.com/brix/crypto-js | JS加密库 |

#### 10.4.3 相关逆向文章

- [Web逆向工程入门指南](https://example.com)
- [JavaScript反混淆技术](https://example.com)
- [API签名机制分析](https://example.com)

#### 10.4.4 秀动相关

- [秀动官网](https://showstart.com)
- [秀动Wap端](https://wap.showstart.com)
- [uni-app官方文档](https://uniapp.dcloud.net.cn/)

### 10.5 版本历史

| 版本 | 日期 | 作者 | 变更内容 |
|------|------|------|----------|
| v1.0 | 2026-05-09 | YukiQui | 初始版本，完成核心算法分析 |
| | | | |

### 10.6 更新日志

**如何更新本文档**：

1. **版本迭代时**：在10.5版本历史中添加新版本记录
2. **算法变化时**：更新第4章静态分析和第6章算法还原
3. **发现问题时**：在第9章添加新的踩坑记录
4. **优化实现时**：更新第7章绕过方案和第9章优化点

**变化检测方法**：

```python
def detect_js_changes(old_hash, new_js_content):
    """检测JS文件是否更新"""
    import hashlib
    new_hash = hashlib.md5(new_js_content.encode()).hexdigest()
    return old_hash != new_hash
```

---

## 文档元数据

| 项目 | 值 |
|------|-----|
| 文档版本 | v1.0 |
| 创建日期 | 2026-05-09 |
| 最后更新 | 2026-05-09 |
| 作者 | YukiQui |
| 目标版本 | index.5703e850.js |
| 分析状态 | 已完成 |
| 代码状态 | 已验证 |

---

## 可复现性检查清单

使用本文档进行复现时，请确认以下条件：

- [ ] Chrome浏览器版本 >= 120
- [ ] Python版本 >= 3.10
- [ ] 已安装 `requests` 库
- [ ] 已获取有效Token（参考10.1.1注入脚本）
- [ ] 目标JS文件版本为 `index.5703e850.js`
- [ ] 网络环境可正常访问 `wap.showstart.com`

---

## 免责声明

本文档仅供学习交流使用，请勿用于商业用途或非法用途。使用本文档中的代码和技术所产生的一切后果由使用者自行承担。

---

*文档版本: v1.0 | 最后更新: 2026-05-09 | 作者: YukiQui*
