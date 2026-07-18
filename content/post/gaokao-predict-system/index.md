---
title: "重庆高考志愿填报预测系统：从数据抽取到智能推荐的全链路技术报告"
date: 2026-07-18T00:00:00+08:00
draft: false
tags: ["Python", "Flask", "数据分析", "PDF解析", "算法", "高考"]
categories: ["技术"]
summary: "一个完整的高考志愿填报辅助系统的技术剖析，涵盖 PDF 逆向解析、位次驱动的录取概率模型、百分位联考换算、冲稳保分档推荐策略，以及多端适配架构设计。"
---

> **文档版本**: v1.0
> **撰写日期**: 2026-07-18
> **作者**: kris
> **项目状态**: 线上运行中（gaokao.711110.xyz）
> **技术栈**: Python / Flask / pandas / pdfplumber / SheetJS

---

## 目录

- [1. 项目概览与问题定义](#1-项目概览与问题定义)
- [2. 技术栈与架构设计](#2-技术栈与架构设计)
- [3. 数据抽取：PDF 逆向解析](#3-数据抽取pdf-逆向解析)
- [4. 核心算法：位次驱动的录取概率模型](#4-核心算法位次驱动的录取概率模型)
- [5. 联考换算：百分位映射方案](#5-联考换算百分位映射方案)
- [6. 推荐策略：分档选取与梯度优化](#6-推荐策略分档选取与梯度优化)
- [7. Web 服务与 API 设计](#7-web-服务与-api-设计)
- [8. 多端适配：SPA / 小程序 / CLI](#8-多端适配spa--小程序--cli)
- [9. 问题、踩坑与迭代优化](#9-问题踩坑与迭代优化)
- [10. 附录与参考](#10-附录与参考)

---

## 1. 项目概览与问题定义

### 1.1 背景

重庆高考志愿填报，考生面临的核心困境：

- **信息不对称**：录取数据散落在教育考试院的 PDF 文件中，数百页表格人肉翻阅效率极低
- **概率判断主观**：「这个分数能不能上某学校」基本靠经验和猜测，缺乏量化依据
- **梯度设计凭感觉**：冲/稳/保 三档比例如何分配、每档选多少专业，缺乏系统性方法
- **联考成绩无法直接参考**：诊断考试（模拟考）分数和高考分数之间存在映射关系，考生难以换算

**核心目标**：构建一个自动化系统，接受考生分数/位次/联考成绩，输出量化的录取概率和分档推荐。

### 1.2 目标范围

| 功能模块 | 说明 | 输入 | 输出 |
|----------|------|------|------|
| 分数位次换算 | 基于一分一段表 | 分数 | 全省排名位次 |
| 录取概率预测 | 位次比 → 分段线性概率 | 考生位次 + 录取最低位次 | 5%~90% 概率值 |
| 联考换算 | 百分位映射 | 联考分/联考位次 | 预测高考等效分 |
| 艺术类综合分 | 官方公式计算 | 文化分 + 专业统考分 | 综合分 |
| 智能推荐 | 冲/稳/保 分档策略 | 位次 + 关键词 | Top N 推荐列表 |
| 志愿评估 | 已选志愿梯度分析 | 志愿清单 | 分档统计 + 梯度建议 |

### 1.3 运行环境

| 项目 | 值 | 备注 |
|------|-----|------|
| 数据源 | 重庆市教育考试院（cqzk.com.cn） | PDF 格式，需逆向解析 |
| 后端 | Python 3.10+ / Flask 3.1 | Railway 部署 |
| 前端 | Bootstrap 5 / Jinja2 模板 | SSR + API 双模式 |
| 域名 | gaokao.711110.xyz | Cloudflare CDN |
| 数据格式 | Excel (.xlsx) | pandas + openpyxl 处理 |

---

## 2. 技术栈与架构设计

### 2.1 依赖清单

| 层面 | 技术 | 用途 |
|------|------|------|
| Web 框架 | Flask 3.1+ | 路由、模板渲染、API |
| WSGI 服务器 | gunicorn (2 workers, gthread) | 生产部署 |
| 数据处理 | pandas 2.2+ / openpyxl 3.1+ | Excel 读写、数据分析 |
| PDF 解析 | pdfplumber | 从官方 PDF 抽取表格 |
| 前端框架 | Bootstrap 5.3.3 (CDN) | 响应式 UI |
| 纯静态版 | SheetJS (xlsx) CDN | 浏览器端解析 Excel |
| 微信小程序 | 原生小程序 | 移动端客户端 |
| 部署 | Railway + Cloudflare | PaaS + CDN |

### 2.2 系统架构

```
用户浏览器 / 微信小程序
        │
        ▼
  Cloudflare CDN (gaokao.711110.xyz)
        │
        ▼
  Railway (gunicorn)
        │
        ▼
  Flask app (app.py)
        │
        ▼
  AdmissionEngine (内存驻留)
        │
        ├── hist.xlsx          ← 历史类录取表
        ├── phys.xlsx          ← 物理类录取表
        ├── 艺术批.xlsx         ← 艺术类录取表
        ├── 历史类一分一段表.xlsx ← 分数↔位次映射
        ├── 物理类一分一段表.xlsx
        └── mock_exams/        ← 联考数据（动态扫描）
             ├── 历史_九龙坡二诊.xlsx
             ├── 物理_康德一诊.xlsx
             └── ...
```

### 2.3 文件结构

```
gaokao/
├── app.py                      # Flask Web 主入口（599行）
├── admission_service.py        # 核心业务引擎（725行）
├── extract_gaokao.py           # PDF 数据抽取工具（291行）
├── cq_admit_predict.py         # CLI 版预测脚本（295行）
├── requirements.txt            # Python 依赖
├── Procfile                    # Railway 部署命令
│
├── static/
│   ├── style.css               # 全站样式
│   └── spa.js                  # 纯浏览器版（819行，完整算法重实现）
│
├── templates/
│   ├── index.html              # 高考分预测页
│   ├── mock_convert.html       # 联考换算页
│   ├── admin.html              # 后台管理页
│   └── about.html              # 关于页
│
├── mock_exams/                 # 联考一分一段数据
│
└── wechat_miniprogram/         # 微信小程序客户端
    ├── pages/predict/          # 高考分预测
    └── pages/mock/             # 联考换算
```

---

## 3. 数据抽取：PDF 逆向解析

### 3.1 数据源分析

重庆市教育考试院的录取数据以 PDF 形式发布在 `cqzk.com.cn`，包含数百页的表格数据：

```
历史类: https://www.cqzk.com.cn/userfiles/fileupload/202507/1947205523449028610.pdf
物理类: https://www.cqzk.com.cn/userfiles/fileupload/202507/1947205223841505282.pdf
```

**挑战**：这些 PDF 是文本型 PDF（非扫描件），但表格结构复杂——专业名称过长会导致跨行换行，且没有明确的表格边框线。

### 3.2 表格提取策略

使用 `pdfplumber` 的 text-based 表格提取，精心调参：

```python
# extract_gaokao.py - 表格提取参数
table_settings = {
    "vertical_strategy": "text",      # 基于文本对齐推断列
    "horizontal_strategy": "text",    # 基于文本对齐推断行
    "intersection_tolerance": 3,
    "snap_tolerance": 3,
    "join_tolerance": 3,
    "edge_min_length": 3,
    "min_words_vertical": 1,
    "min_words_horizontal": 1,
    "text_x_tolerance": 1,
    "text_y_tolerance": 3,
}
```

### 3.3 续行检测：核心难点

PDF 表格中，当专业名称过长时会折行，导致一条记录被拆成两行。必须准确检测这种续行并合并：

```python
def is_continuation_row(row: List[str]) -> bool:
    """
    检测是否为上一行的续行（专业名称太长导致换行）。
    启发式：
    - 只有一个非空单元格，且该单元格不是数字 → 大概率是续行文本
    - 前3列为空，第4列有内容，后续列为空 → 专业名称续行
    """
    nonempty = [c for c in row if c]
    if not nonempty:
        return True
    if len(nonempty) == 1 and not looks_numeric_or_placeholder(nonempty[0]):
        return True
    if len(row) >= 4:
        if all(not row[i] for i in range(min(3, len(row)))) \
           and any(row[3:4]) \
           and all(not c for c in row[4:]):
            return True
    return False

def append_continuation(prev_row: List[str], row: List[str]) -> None:
    """将续行文本追加到上一行的专业名称字段。"""
    text = " ".join([c for c in row if c]).strip()
    if not text:
        return
    if len(prev_row) >= 4 and prev_row[3]:
        prev_row[3] = (prev_row[3] + " " + text).strip()  # 追加到专业名称
    elif len(prev_row) >= 2:
        prev_row[1] = (prev_row[1] + " " + text).strip()  # 追加到院校名称
```

### 3.4 行对齐逻辑

PDF 提取的行列数不固定，需要启发式对齐到标准 8 列结构：

```python
EXPECTED_COLS = [
    "院校代号", "院校名称", "专业代号", "专业名称",
    "计划数", "录取数", "最低分", "最低位次",
]

def align_row(row: List[str]) -> List[str]:
    """
    启发式对齐：最后4列固定为数字列（计划数/录取数/最低分/最低位次），
    多余的文本列合并到专业名称字段。
    """
    row = trim_row(row)
    if len(row) == len(EXPECTED_COLS):
        return row

    if len(row) > len(EXPECTED_COLS):
        tail = row[-4:]                     # 数字尾部
        head = row[:-4]                     # 文本头部
        if len(head) > 4:
            merged = " ".join([c for c in head[3:] if c])
            head = head[:3] + [merged]      # 多余文本合并到专业名称
        return head + tail

    return row + [""] * (len(EXPECTED_COLS) - len(row))  # 补齐
```

### 3.5 数据清洗流水线

```
原始 PDF
  ↓  pdfplumber.extract_tables()
每页原始行列表
  ↓  clean_cell()：全角空格→半角、换行→空格
  ↓  is_header_row()：跳过表头行
  ↓  is_empty_row()：跳过空行
原始行列表
  ↓  is_continuation_row() + append_continuation()：续行合并
  ↓  align_row()：对齐到 8 列
结构化记录列表
  ↓  pd.DataFrame → to_excel()
最终 Excel 文件
```

---

## 4. 核心算法：位次驱动的录取概率模型

### 4.1 设计思路

传统的分数线对比存在一个本质问题：**同一个分数在不同年份对应的竞争力不同**（因为试卷难度、考生人数变化）。位次（全省排名）则是更稳定的锚点——排名第 5000 名的考生，无论试卷难易，其竞争力相对位置是一致的。

**核心概念——位次比**：

```
位次比 = 考生位次 / 该专业往年录取最低位次
```

- 位次比 < 1.0：考生位次优于录取线，越小越有把握
- 位次比 = 1.0：恰好压线
- 位次比 > 1.0：考生位次低于录取线，越大越危险

### 4.2 分档规则

```python
# admission_service.py
SAFE_RATIO = 0.80     # 位次比 <= 0.80 → "保"
STEADY_RATIO = 1.10   # 位次比 <= 1.10 → "稳"
                      # 位次比 > 1.10  → "冲"
```

这套阈值经过迭代优化。早期 CLI 版本使用 `SAFE_RATIO = 0.90`，后来调整为 `0.80`——更宽的「稳」区间让推荐结果更实用。

### 4.3 概率计算：分段线性模型

这是整个系统最核心的函数，将位次比映射到 5%~90% 的概率空间：

```python
def calc_probability(ratio: float) -> float:
    """
    分段线性概率模型。
    5 个区间，每个区间内线性插值。
    """
    if ratio <= 0.65:
        p = 0.84                                              # 极高把握
    elif ratio <= 0.80:  # SAFE_RATIO
        p = 0.66 + (0.80 - ratio) * (0.18 / 0.15)            # 66%~84%
    elif ratio <= 1.10:  # STEADY_RATIO
        p = 0.40 + (1.10 - ratio) * (0.26 / 0.30)            # 40%~66%
    elif ratio <= 1.30:
        p = 0.20 + (1.30 - ratio) * (0.20 / 0.20)            # 20%~40%
    else:
        p = 0.08                                              # 高风险
    p = max(0.05, min(0.90, p))                               # 截断到 [5%, 90%]
    return round(float(p), 4)
```

**概率映射表**：

| 位次比区间 | 分档 | 概率范围 | 语义 |
|-----------|------|---------|------|
| ≤ 0.65 | 保 | 84% | 极高把握 |
| 0.65 ~ 0.80 | 保 | 66% ~ 84% | 高把握 |
| 0.80 ~ 1.10 | 稳 | 40% ~ 66% | 较稳妥/可尝试 |
| 1.10 ~ 1.30 | 冲 | 20% ~ 40% | 冲刺项 |
| > 1.30 | 冲 | 8% | 高风险 |

> **为什么不用 0% 和 100%？** 高考录取受多因素影响（大小年、专业调剂、投档规则变化等），用 [5%, 90%] 的概率区间更诚实——没有任何预测能给出绝对保证。

### 4.4 分数↔位次换算

从一分一段表构建映射，处理「xxx分及以上」等特殊格式：

```python
def load_score_table(path: str):
    """
    一分一段表解析。
    处理多种格式：纯数字、"650分"、"690分及以上"、"650-651" 区间。
    """
    score_to_rank_map = {}
    top_score, top_rank = None, None

    for _, row in df.iterrows():
        segment = str(row["分数段"]).strip()
        rank = int(row["累计人数"])

        if "及以上" in segment:
            # "690分及以上" → 特殊处理
            top_score = float(re.findall(r"\d+", segment)[0])
            top_rank = rank
            continue

        # 常规分数行
        cleaned = segment.replace("分", "").strip()
        score_to_rank_map[float(cleaned)] = rank

    return score_to_rank_map, top_score, top_rank

def score_to_rank(score, score_map, top_score, top_rank):
    """查表换算。高于最高分段则返回 top_rank。"""
    if top_score and score >= top_score:
        return top_rank
    # 二分查找：取 ≤ score 的最大分数段
    sorted_scores = sorted(score_map.keys(), reverse=True)
    for s in sorted_scores:
        if score >= s:
            return score_map[s]
    return score_map[sorted_scores[-1]]
```

### 4.5 艺术类综合分计算

重庆艺术类录取使用官方综合分公式，使用 `Decimal` 确保银行家舍入精度：

```python
from decimal import Decimal, ROUND_HALF_UP

def calc_art_composite(art_mode, culture_score, art_score):
    """
    综合分 = (文化分/750 × 300 × 50%) + 专业统考成绩 × 50%
    使用 Decimal 精确计算，ROUND_HALF_UP 四舍五入保留两位。
    """
    value = (culture_score / 750.0) * 300.0 * 0.5 + art_score * 0.5
    return float(
        Decimal(str(value)).quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)
    )
```

---

## 5. 联考换算：百分位映射方案

### 5.1 问题定义

高三学生在正式高考前会参加多次诊断考试（联考），比如「九龙坡二诊」「康德一诊」等。这些考试的参与人数、试卷难度与高考完全不同，分数无法直接对比。

**核心思路**：用百分位（百分等级）作为桥梁——如果你在联考中排名前 20%，那么预测你在高考中也大约排名前 20%。

### 5.2 算法实现

```python
def convert_rank_by_percentile(source_rank, source_total, target_total):
    """
    百分位映射：联考位次 → 高考等效位次。
    source_rank / source_total = target_rank / target_total
    """
    ratio = float(source_rank) / float(source_total)
    ratio = max(1.0 / float(source_total), min(ratio, 1.0))  # 防溢出
    target_rank = int(round(ratio * float(target_total)))
    return max(1, min(target_rank, int(target_total)))
```

### 5.3 换算数据流

```
联考成绩 500
  ↓ 查联考一分一段表
联考位次 3000
  ↓ 计算百分位
3000 / 15000(联考总人数) = 20%
  ↓ 映射到高考
180000(高考总人数) × 20% = 36000
  ↓ 反查高考一分一段表
预测高考分 495
```

### 5.4 联考数据动态发现

系统自动扫描 `mock_exams/` 目录，通过文件名正则自动识别科类和考试名称，零配置添加新联考数据：

```python
def parse_mock_meta_from_filename(filename):
    """
    从文件名自动提取科类和考试名。
    "历史类_九龙坡二诊.xlsx" → stream="历史", label="九龙坡二诊"
    "物理_康德一诊.xlsx"     → stream="物理", label="康德一诊"
    """
    stem = os.path.splitext(filename)[0]
    # 识别科类
    if "历史" in stem or "hist" in stem.lower():
        stream = "历史"
    elif "物理" in stem or "phys" in stem.lower():
        stream = "物理"
    else:
        return "", ""

    # 清理得到考试名称
    label = stem
    label = re.sub(r"(历史类?|物理类?)", "", label)
    label = re.sub(r"(联考|一分一段表|分数段)", "", label)
    label = re.sub(r"[_\-\s]+", " ", label).strip()
    return stream, label or "联考"
```

只需把 `物理_新联考.xlsx` 扔进 `mock_exams/` 目录，系统自动识别并添加到下拉选项中。

---

## 6. 推荐策略：分档选取与梯度优化

### 6.1 分档名额分配

按照经典的志愿填报「冲稳保」策略，固定比例分配推荐名额：

```python
TIER_DISTRIBUTION = {
    "冲": 0.25,   # 25% 的名额给冲刺项
    "稳": 0.45,   # 45% 的名额给稳妥项
    "保": 0.30,   # 30% 的名额给保底项
}
```

### 6.2 选取算法

`_select_by_tier_strategy` 是推荐引擎的核心，采用三阶段选取策略：

```python
def _select_by_tier_strategy(self, ranked_df, top_n, per_school_limit=2):
    """
    三阶段选取：
    1. 按比例从每个分档选取（每校限 2 个专业）
    2. 分档不足则从全排序列表补选（仍限制每校 2 个）
    3. 仍不够则放宽学校限制继续补选
    """
    desired = self._build_tier_targets(top_n)  # {"冲": 7, "稳": 14, "保": 9}

    # 阶段 1：按分档选取
    for tier in ["冲", "稳", "保"]:
        pick_from(tier_frames[tier], desired[tier], enforce_school_limit=True)

    # 阶段 2：分档不够，从全排序补选
    if len(selected) < top_n:
        pick_from(full_sorted, top_n - len(selected), enforce_school_limit=True)

    # 阶段 3：放宽学校限制
    if len(selected) < top_n:
        pick_from(full_sorted, top_n - len(selected), enforce_school_limit=False)
```

**排序依据——「贴合度」**：

```python
df["贴合度"] = (df["位次比"] - 1.0).abs()
```

贴合度 = |位次比 - 1.0|，越接近 0 表示该专业和考生实力最匹配。每个分档内按贴合度排序，优先推荐最贴合的选项。

### 6.3 梯度评估

当用户提供自己的志愿清单时，系统会分析梯度合理性并给出建议：

```python
def _build_gradient_tips(rows):
    tips = []

    if tier_count["保"] == 0:
        tips.append("当前志愿缺少保底项，建议至少增加 1-2 个保底专业。")

    if tier_count["冲"] / total > 0.6:
        tips.append("冲刺项占比偏高，整体风险较大。")

    if tier_count["冲"] == 0 and tier_count["保"] / total >= 0.7:
        tips.append("方案偏保守，可加入少量冲刺项提升上限。")

    if max(probabilities) - min(probabilities) < 0.2:
        tips.append("志愿概率分布较集中，梯度不够明显。")

    first_three = valid[:3]
    if all(r["tier"] == "冲" for r in first_three):
        tips.append("前 3 志愿均为冲刺，建议前段加入至少 1 个稳妥项。")
```

---

## 7. Web 服务与 API 设计

### 7.1 路由架构

| 路由 | 方法 | 功能 | 备注 |
|------|------|------|------|
| `/` | GET/POST | 高考分预测页 | SSR 表单 |
| `/mock-convert` | GET/POST | 联考换算页 | SSR 表单 |
| `/about` | GET | 关于页 | |
| `/api/predict` | POST | JSON API - 预测 | 供小程序/SPA 调用 |
| `/api/mock-convert` | POST | JSON API - 联考换算 | |
| `/api/mock-options` | GET | 获取可用联考选项 | |
| `/api/export-wishlist` | POST | 导出志愿表 Excel | 流式下载 |
| `/ops-console-*` | GET/POST | 后台管理 | 隐藏路径 |
| `/reload` | POST | 热重载数据 | Token 保护 |
| `/health` | GET | 健康检查 | |

### 7.2 API 设计示例

**请求**：
```json
POST /api/predict
{
    "stream": "历史",
    "score": "580",
    "top_n": "30",
    "school_keyword": "师范"
}
```

**响应**：
```json
{
    "ok": true,
    "data": {
        "stream": "历史",
        "score": 580,
        "user_rank": 12450,
        "recommendations": [
            {
                "school_name": "华中师范大学",
                "major_name": "汉语言文学",
                "min_score": 575,
                "min_rank": 13200,
                "rank_ratio": 0.9432,
                "tier": "稳",
                "probability": 0.5567,
                "probability_desc": "较稳妥：录取机会较高，适合作为稳妥志愿"
            }
        ],
        "tier_stats": {"冲": 8, "稳": 14, "保": 8}
    }
}
```

### 7.3 安全设计

**管理后台**：

```python
# 后台路径可通过环境变量自定义，默认使用不可猜测的路径
ADMIN_PATH = normalize_admin_path(
    os.environ.get("ADMIN_PATH", "/ops-console-kris-1027")
)

# 登录认证基于 Flask session
def is_admin_authed():
    return bool(session.get("admin_ok"))
```

**路径遍历防护**（联考文件删除）：

```python
# 防止 ../../etc/passwd 类攻击
target_path = os.path.normpath(os.path.join(BASE_DIR, key))
allowed_root = os.path.normpath(os.path.join(BASE_DIR, MOCK_SCORE_DIR))
if target_path.startswith(allowed_root) and os.path.exists(target_path):
    os.remove(target_path)
else:
    message = "删除失败：非法路径。"
```

**输入清洗**：

```python
def safe_label(label: str) -> str:
    """清洗用户输入的标签，防注入。"""
    text = re.sub(r"\s+", " ", str(label or "").strip())
    text = re.sub(r"[^\w\u4e00-\u9fff\- ]", "", text)   # 只保留中英文/数字/横线
    text = text.replace(" ", "_")
    return text[:40].strip("_")
```

### 7.4 志愿导出

前端勾选推荐项后，通过 `/api/export-wishlist` 生成 Excel 文件流式下载：

```python
@app.route("/api/export-wishlist", methods=["POST"])
def api_export_wishlist():
    rows = build_export_rows(payload)
    df = pd.DataFrame(rows)

    output = io.BytesIO()
    with pd.ExcelWriter(output, engine="openpyxl") as writer:
        df.to_excel(writer, index=False, sheet_name="志愿表")
        # 自动列宽
        ws = writer.sheets["志愿表"]
        for idx, column in enumerate(df.columns, start=1):
            values = [str(column)] + [str(v) for v in df[column].tolist()]
            width = max(10, min(36, max(len(v) for v in values) + 2))
            ws.column_dimensions[get_column_letter(idx)].width = width

    output.seek(0)
    filename = f"{source}_{stream}_志愿表_{datetime.now().strftime('%Y%m%d_%H%M%S')}.xlsx"
    return send_file(output, as_attachment=True, download_name=filename)
```

---

## 8. 多端适配：SPA / 小程序 / CLI

### 8.1 四种运行模式

项目提供四种完全独立的运行模式，覆盖不同场景：

| 模式 | 入口 | 后端依赖 | 适用场景 |
|------|------|---------|---------|
| Flask Web | `python app.py` | Flask + pandas | 线上部署 |
| 纯静态 SPA | `index.html` | 无（浏览器端 SheetJS） | 离线使用 |
| 微信小程序 | `wechat_miniprogram/` | 调用 Flask API | 移动端 |
| CLI | `python cq_admit_predict.py` | pandas | 命令行快速查询 |

### 8.2 算法一致性

Python 后端 (`admission_service.py`, 725行) 和 JavaScript 前端 (`spa.js`, 819行) **完整复制了同一套算法**，包括概率函数的每个分段参数都完全一致。这确保了无论用哪种模式访问，结果完全相同。

### 8.3 微信小程序

```javascript
// wechat_miniprogram/config.js
module.exports = {
  API_BASE: 'https://gaokao.711110.xyz'
};
```

小程序纯粹作为 API 客户端，所有计算逻辑在服务端完成。两个核心页面：
- `pages/predict/` — 高考分预测
- `pages/mock/` — 联考换算

---

## 9. 问题、踩坑与迭代优化

### 9.1 模型迭代

| 版本 | 分档阈值 | 概率模型 | 改进 |
|------|---------|---------|------|
| CLI v1 | SAFE=0.90 | 固定值（保70%/稳50%/冲30%） | 初版，概率粗糙 |
| Web v1 | SAFE=0.80 | 分段线性（5区间） | 更精细的概率映射 |
| Web v1.1 | SAFE=0.80 | 同上 + 联考换算 | 添加诊断考试支持 |

**为什么从固定概率升级到分段线性？** 固定概率模型只有三个值（30%/50%/70%），无法区分「刚好过线的稳」和「远超录取线的稳」。分段线性模型在每个档位内提供连续的概率值，让考生更直观地感受不同选项的风险差异。

### 9.2 PDF 解析踩坑

1. **全角空格**：PDF 中的空格实际是 `\u3000`（全角空格），需要统一替换
2. **续行误判**：部分院校代号只有一个字符（如「A」），容易被误判为续行文本。通过 `looks_numeric_or_placeholder()` 辅助判断
3. **表头重复**：每页 PDF 都重复表头，需要 `is_header_row()` 检测跳过
4. **列数不固定**：不同页面的列数可能不同，`align_row()` 的启发式对齐解决了这个问题

### 9.3 精度问题

艺术类综合分使用 Python 内置 `float` 计算会产生浮点误差：

```python
# 错误方式
>>> (400/750) * 300 * 0.5 + 230 * 0.5
195.00000000000003  # 浮点误差

# 正确方式：使用 Decimal
>>> float(Decimal(str((400/750)*300*0.5 + 230*0.5)).quantize(
...     Decimal("0.01"), rounding=ROUND_HALF_UP))
195.0  # 精确
```

### 9.4 热重载

生产环境中，上传新的联考数据后需要重新加载所有数据文件。通过 `/reload` 接口实现热重载，无需重启服务：

```python
@app.route("/reload", methods=["POST"])
def reload_data():
    token = request.headers.get("X-Admin-Token", "")
    if token != os.environ.get("RELOAD_TOKEN", ""):
        return {"ok": False, "message": "unauthorized"}, 403
    engine.reload()  # 重新扫描所有数据文件
    return {"ok": True, "message": "数据已重新加载"}
```

后台上传联考文件后自动调用 `engine.reload()`，实现即传即用。

---

## 10. 附录与参考

### 10.1 核心数据流总览

```
用户输入（分数 / 位次 / 联考分 / 艺术文化+专业分）
    │
    ▼
rank_from_input() ─────────────────────────────────┐
    │                                               │
    ├─ 直接位次？→ 使用用户位次                     │
    ├─ 艺术类？→ 计算综合分 → 查录取表位次映射       │
    ├─ 联考模式？→ 联考分→联考位次→百分位→高考位次   │
    └─ 普通分数？→ 查一分一段表 → 得到位次           │
    │                                               │
    ▼                                               │
  user_rank（统一的位次值）◄─────────────────────────┘
    │
    ▼
recommend() ── 对录取表每行计算：
    │  位次比 = user_rank / 录取最低位次
    │  分档 = calc_tier(位次比)
    │  概率 = calc_probability(位次比)
    │  贴合度 = |位次比 - 1.0|
    │
    ├─ 有关键词？→ 过滤 → 按贴合度排序 → 取 top_n
    └─ 无关键词？→ _select_by_tier_strategy()
         │  按 冲25% 稳45% 保30% 分配名额
         │  每档内按贴合度排序
         │  每校限 2 个专业
         │  不足则逐级放宽补选
         │
         ▼
  推荐列表 + 分档统计 + 概率说明
```

### 10.2 概率解释映射

```python
PROBABILITY_GUIDE = [
    {"min": 0.75, "label": "高把握",  "desc": "基本属于保底区间，可优先考虑专业偏好"},
    {"min": 0.60, "label": "较稳妥",  "desc": "录取机会较高，适合作为稳妥志愿"},
    {"min": 0.45, "label": "可尝试",  "desc": "存在机会，建议与保底志愿搭配"},
    {"min": 0.30, "label": "冲刺项",  "desc": "风险偏高，建议放在前段冲刺"},
    {"min": 0.00, "label": "高风险",  "desc": "录取不确定性较大，谨慎填报"},
]
```

### 10.3 关键文件索引

| 文件 | 行数 | 核心职责 |
|------|------|---------|
| `admission_service.py` | 725 | 位次换算、概率计算、推荐算法、联考换算 |
| `app.py` | 599 | Flask 路由、API、后台管理、志愿导出 |
| `extract_gaokao.py` | 291 | PDF 逆向解析、表格抽取、数据清洗 |
| `cq_admit_predict.py` | 295 | CLI 版预测工具（早期版本） |
| `static/spa.js` | 819 | 浏览器端完整算法重实现 |

---

## 免责声明

本系统基于历年录取数据的统计分析，提供概率参考而非录取保证。录取结果受当年报考人数、院校计划变动、投档规则调整等多因素影响，预测概率仅供志愿填报参考。建议考生综合考虑个人兴趣、地域偏好、专业前景等因素，做出最终决策。
