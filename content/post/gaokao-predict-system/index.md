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
- [9. 前端交互细节：志愿勾选与导出机制](#9-前端交互细节志愿勾选与导出机制)
- [10. 问题、踩坑与迭代优化](#10-问题踩坑与迭代优化)
- [11. 附录与参考](#11-附录与参考)

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

### 3.3 单元格清洗：`clean_cell()`

PDF 抽取的原始文本包含各种非标准字符，`clean_cell()` 是所有数据进入流水线的第一道关卡：

```python
def clean_cell(cell) -> str:
    """
    清洗单元格文本。
    处理 PDF 提取中常见的各种脏数据：
    - None 值 → 空字符串
    - 全角空格 \u3000 → 半角空格
    - 内部换行符 \n → 空格（PDF 同一单元格内换行）
    - 连续空白 → 单个空格
    - 首尾空白 → 移除
    """
    if cell is None:
        return ""
    text = str(cell)
    text = text.replace("\u3000", " ")     # 全角空格 → 半角
    text = text.replace("\n", " ")         # 内部换行 → 空格
    text = re.sub(r"\s+", " ", text)       # 合并连续空白
    return text.strip()
```

**为什么要处理全角空格？** 重庆教育考试院的 PDF 在表格单元格之间使用 `\u3000`（中文全角空格）作为间隔符，而不是标准的 ASCII 空格。如果不统一处理，后续的字符串匹配和数值解析会全部失败。

### 3.4 表头检测：`is_header_row()`

每页 PDF 都会重复打印表头行（如「院校代号 | 院校名称 | 专业代号 | ...」），必须检测并跳过：

```python
HEADER_KEYWORDS = {"院校代号", "院校名称", "专业代号", "专业名称",
                   "计划数", "录取数", "最低分", "最低位次",
                   "院校", "代号", "专业"}

def is_header_row(row: List[str]) -> bool:
    """
    检测是否为表头行。
    策略：如果一行中包含 2 个及以上表头关键词，判定为表头行。
    """
    if not row:
        return False
    text = " ".join(clean_cell(c) for c in row)
    count = sum(1 for kw in HEADER_KEYWORDS if kw in text)
    return count >= 2
```

**为什么阈值是 2？** 单个关键词可能出现在院校名称中（比如某大学叫「XX代号学院」），但同时出现两个表头关键词的概率极低。实测中这个阈值零误判。

### 3.5 数值判断：`looks_numeric_or_placeholder()`

续行检测的关键辅助函数——判断一个字符串是否「看起来像数值或占位符」：

```python
def looks_numeric_or_placeholder(text: str) -> bool:
    """
    判断文本是否为数值或常见占位符。
    用于区分「续行文本」和「单列有效数据」。

    匹配：
    - 纯数字："1234"
    - 带小数："98.5"
    - 带负号："-"（占位）
    - 带逗号分隔："1,234"
    - 分数范围："650-660"
    """
    if not text or not text.strip():
        return False
    cleaned = text.strip()
    # 常见占位符
    if cleaned in ("-", "—", "/", "//", "*", ".", ".."):
        return True
    # 移除逗号后尝试解析为数字
    try:
        float(cleaned.replace(",", ""))
        return True
    except ValueError:
        pass
    # 分数范围格式 "650-660"
    if re.match(r"^\d+\s*[-–]\s*\d+$", cleaned):
        return True
    return False
```

### 3.6 续行检测与合并：核心难点

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

### 3.7 行对齐逻辑

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

### 3.8 记录构建：`build_records()` 完整流程

`build_records()` 是从原始行列表到结构化记录的核心转换函数，协调上述所有子函数：

```python
def build_records(raw_rows: List[List[str]]) -> List[List[str]]:
    """
    将 PDF 提取的原始行列表转换为结构化记录。
    完整处理流程：
    1. 清洗每个单元格
    2. 跳过空行和表头行
    3. 检测续行并合并到前一行
    4. 对齐到标准 8 列结构
    5. 验证关键字段（院校代号、最低分/位次）

    返回：对齐后的 8 列记录列表
    """
    records = []
    pending = None   # 待合并的当前行

    for raw_row in raw_rows:
        # 第一步：清洗每个单元格
        row = [clean_cell(c) for c in raw_row]

        # 第二步：跳过空行和表头行
        if is_empty_row(row):
            continue
        if is_header_row(row):
            continue

        # 第三步：续行检测
        if is_continuation_row(row):
            if pending is not None:
                append_continuation(pending, row)
            continue

        # 第四步：保存上一条完整记录
        if pending is not None:
            aligned = align_row(pending)
            if _is_valid_record(aligned):
                records.append(aligned)

        # 第五步：当前行成为新的 pending
        pending = row

    # 处理最后一条
    if pending is not None:
        aligned = align_row(pending)
        if _is_valid_record(aligned):
            records.append(aligned)

    return records

def _is_valid_record(row: List[str]) -> bool:
    """
    验证记录有效性。
    至少需要：院校代号非空、最低分或最低位次至少有一个是数字。
    """
    if len(row) < 8:
        return False
    if not row[0].strip():       # 院校代号
        return False
    # 最低分或最低位次至少有一个有效
    return looks_numeric_or_placeholder(row[6]) or looks_numeric_or_placeholder(row[7])
```

### 3.9 数值转换：`to_int()`

PDF 抽取的数值可能包含逗号分隔符、占位符等，需要安全转换：

```python
def to_int(val, default=None):
    """
    安全地将各种格式的文本转为整数。
    处理：
    - "1,234" → 1234（逗号分隔）
    - "650分" → 650（带单位后缀）
    - "-" → default（占位符）
    - "" → default（空值）
    - "650.0" → 650（浮点格式）
    """
    if val is None:
        return default
    text = str(val).strip()
    if not text or text in ("-", "—", "/", "*"):
        return default
    # 移除常见后缀
    text = text.replace("分", "").replace("人", "").replace("名", "")
    # 移除逗号
    text = text.replace(",", "")
    try:
        return int(float(text))
    except (ValueError, TypeError):
        return default
```

### 3.10 数据源加载：`load_source()`

支持从本地文件路径或 HTTP URL 加载 PDF 数据源：

```python
def load_source(source: str) -> bytes:
    """
    加载数据源，支持本地路径和 HTTP URL。
    URL 模式下自动添加浏览器 User-Agent 避免被拦截。
    """
    if source.startswith("http://") or source.startswith("https://"):
        import urllib.request
        req = urllib.request.Request(
            source,
            headers={
                "User-Agent": (
                    "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
                    "AppleWebKit/537.36 (KHTML, like Gecko) "
                    "Chrome/120.0.0.0 Safari/537.36"
                )
            }
        )
        with urllib.request.urlopen(req, timeout=30) as resp:
            return resp.read()
    else:
        with open(source, "rb") as f:
            return f.read()
```

### 3.11 完整构建流程：`build_dataframe()`

将所有步骤串联，输出最终的 pandas DataFrame：

```python
def build_dataframe(source: str) -> pd.DataFrame:
    """
    从 PDF 源构建完整的录取数据 DataFrame。
    完整流程：
    1. 加载源文件（本地/URL）
    2. 逐页提取表格
    3. 逐行清洗、续行合并、对齐
    4. 构建 DataFrame 并类型转换
    5. 去重、排序、导出
    """
    pdf_bytes = load_source(source)
    all_rows = []

    with pdfplumber.open(io.BytesIO(pdf_bytes)) as pdf:
        for page_num, page in enumerate(pdf.pages, 1):
            tables = page.extract_tables(table_settings)
            if not tables:
                continue
            for table in tables:
                all_rows.extend(table)
            if page_num % 50 == 0:
                print(f"  已处理 {page_num} 页...")

    print(f"  总计提取 {len(all_rows)} 原始行")

    # 构建记录
    records = build_records(all_rows)
    print(f"  有效记录 {len(records)} 条")

    # 创建 DataFrame
    df = pd.DataFrame(records, columns=EXPECTED_COLS)

    # 类型转换
    for col in ["计划数", "录取数", "最低分", "最低位次"]:
        df[col] = df[col].apply(lambda v: to_int(v))

    # 去掉全部数值列都为空的行
    df = df.dropna(subset=["最低分", "最低位次"], how="all")

    # 按院校代号 + 专业代号排序
    df = df.sort_values(["院校代号", "专业代号"]).reset_index(drop=True)

    return df
```

### 3.12 数据清洗流水线总览

```
原始 PDF
  ↓  load_source()：下载/读取
  ↓  pdfplumber.open() + extract_tables()
每页原始行列表
  ↓  clean_cell()：全角空格→半角、换行→空格
  ↓  is_header_row()：跳过表头行（关键词检测）
  ↓  is_empty_row()：跳过空行
原始行列表
  ↓  is_continuation_row() + append_continuation()：续行合并
  ↓  align_row()：对齐到 8 列
  ↓  _is_valid_record()：验证有效性
结构化记录列表
  ↓  pd.DataFrame + 类型转换 + 去重排序
  ↓  to_excel()
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

### 4.5 位次反查分数：`rank_to_score()`

与 `score_to_rank()` 互逆——给定一个位次，反查对应的分数。联考换算的最后一步需要用到：

```python
def rank_to_score(rank, score_map, top_score, top_rank):
    """
    位次 → 分数反查。
    遍历一分一段表，找到 ≤ rank 的最小位次对应的分数。

    逻辑：
    一分一段表是「分数 → 累计人数（位次）」的映射，分数越高位次越小。
    给定目标位次 rank，需要找到满足 score_map[s] >= rank 的最大分数 s。

    边界处理：
    - rank <= top_rank：返回 top_score（极高分段）
    - rank 超出表的最大位次：返回表中最低分
    """
    if top_rank is not None and rank <= top_rank:
        return top_score

    # 按分数从高到低遍历
    best_score = None
    for s in sorted(score_map.keys(), reverse=True):
        if score_map[s] >= rank:
            best_score = s
            break
        best_score = s  # 兜底取最低分

    return best_score
```

### 4.6 艺术类综合分计算

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

### 4.7 艺术类位次映射构建：`build_rank_map_from_admission_scores()`

艺术类没有官方的「一分一段表」，需要从录取数据反向构建位次映射。核心思路是把所有录取最低分排序后分配人工位次：

```python
def build_rank_map_from_admission_scores(df: pd.DataFrame):
    """
    从艺术类录取表构建伪一分一段表。

    步骤：
    1. 提取所有有效的「最低分」
    2. 去重、降序排列
    3. 为每个分数分配累计位次（每个分数占 1 个位次）

    返回：score_to_rank_map, top_score, top_rank
    与 load_score_table() 返回格式一致，可复用同一套换算逻辑。
    """
    valid_scores = df["最低分"].dropna().unique()
    valid_scores = sorted(valid_scores, reverse=True)

    score_to_rank = {}
    for i, score in enumerate(valid_scores, start=1):
        score_to_rank[float(score)] = i

    top_score = valid_scores[0] if valid_scores else None
    top_rank = 1 if valid_scores else None

    return score_to_rank, top_score, top_rank
```

**为什么不用录取人数加权？** 因为艺术类各专业录取人数极少（通常 1-5 人），加权后的位次分辨率反而不如直接按分数排序。实测中，直接排序法的概率预测误差最小。

### 4.8 AdmissionEngine 类：完整初始化流程

`AdmissionEngine` 是整个系统的核心引擎类，在启动时加载所有数据到内存，后续请求直接查表计算：

```python
class AdmissionEngine:
    """
    高考录取预测引擎。
    初始化时加载所有数据文件到内存，提供：
    - recommend()：智能推荐
    - predict()：单专业概率预测
    - mock_convert()：联考换算
    - evaluate_wishlist()：志愿评估
    """

    def __init__(self, base_dir="."):
        self.base_dir = base_dir

        # 录取数据（DataFrame）
        self.hist_df = None          # 历史类
        self.phys_df = None          # 物理类
        self.art_df = None           # 艺术批

        # 一分一段表（score_to_rank_map, top_score, top_rank）
        self.hist_score_table = None
        self.phys_score_table = None
        self.art_score_table = None  # 从录取表构建

        # 联考数据 {(stream, label): (score_table, total)}
        self.mock_tables = {}

        # 元数据
        self.last_loaded = None
        self.load_errors = []

        # 执行初始加载
        self.reload()
```

### 4.9 `reload()` 方法：数据加载细节

`reload()` 是热加载的入口，也是初始化时调用的方法。它的设计要求：加载过程中不能影响正在处理的请求。

```python
def reload(self):
    """
    重新加载所有数据文件。
    设计要点：
    - 失败时保留旧数据（不清空已有数据）
    - 记录每个文件的加载错误，不因单文件失败中断
    - 加载完成后更新时间戳
    """
    errors = []

    # 1. 加载录取表
    for name, attr, filename in [
        ("历史类", "hist_df", "hist.xlsx"),
        ("物理类", "phys_df", "phys.xlsx"),
        ("艺术批", "art_df", "艺术批.xlsx"),
    ]:
        path = os.path.join(self.base_dir, filename)
        if os.path.exists(path):
            try:
                df = pd.read_excel(path, engine="openpyxl")
                df.columns = [normalizeCol(c) for c in df.columns]
                setattr(self, attr, df)
            except Exception as e:
                errors.append(f"{name}: {e}")
        else:
            errors.append(f"{name}: 文件不存在 ({filename})")

    # 2. 加载一分一段表
    for name, attr, filename in [
        ("历史类一分一段", "hist_score_table", "历史类一分一段表.xlsx"),
        ("物理类一分一段", "phys_score_table", "物理类一分一段表.xlsx"),
    ]:
        path = os.path.join(self.base_dir, filename)
        if os.path.exists(path):
            try:
                setattr(self, attr, load_score_table(path))
            except Exception as e:
                errors.append(f"{name}: {e}")

    # 3. 构建艺术类伪一分一段表
    if self.art_df is not None:
        self.art_score_table = build_rank_map_from_admission_scores(self.art_df)

    # 4. 扫描联考数据
    self._load_mock_tables()

    # 5. 更新元数据
    self.last_loaded = datetime.now().isoformat()
    self.load_errors = errors
```

`normalizeCol()` 函数用于统一列名格式，处理各种空白字符和全角/半角差异：

```python
def normalizeCol(col: str) -> str:
    """
    标准化 Excel 列名。
    处理：
    - 前后空白
    - 全角空格
    - 连续空白
    - 换行符
    """
    if not col:
        return ""
    col = str(col).strip()
    col = col.replace("\u3000", " ")
    col = col.replace("\n", " ")
    col = re.sub(r"\s+", " ", col)
    return col.strip()
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

### 5.5 `_load_mock_tables()` 的完整扫描逻辑

联考数据扫描不只看 `mock_exams/` 目录——系统会扫描多个可能的目录，并且处理去重和 reserved 保护：

```python
def _load_mock_tables(self):
    """
    完整的联考数据扫描流程：
    1. 定义搜索路径列表（mock_exams/ 和额外目录）
    2. reserved 集合保护核心文件不被误认
    3. 遍历所有 .xlsx 文件，解析元数据
    4. 去重：相同 (stream, label) 的文件只保留最新的
    """
    # reserved：核心数据文件名，不参与联考扫描
    reserved = {
        "hist.xlsx", "phys.xlsx", "艺术批.xlsx",
        "历史类一分一段表.xlsx", "物理类一分一段表.xlsx",
    }

    # 搜索目录列表
    search_dirs = [
        os.path.join(self.base_dir, MOCK_SCORE_DIR),   # mock_exams/
    ]
    # 额外：base_dir 本身的 xlsx 也可能是联考数据
    # （但 reserved 中的文件会被排除）

    seen = {}   # {(stream, label): (path, mtime)}

    for search_dir in search_dirs:
        if not os.path.isdir(search_dir):
            continue
        for fname in os.listdir(search_dir):
            if not fname.endswith(".xlsx"):
                continue
            if fname in reserved:
                continue

            stream, label = parse_mock_meta_from_filename(fname)
            if not stream:
                continue

            fpath = os.path.join(search_dir, fname)
            mtime = os.path.getmtime(fpath)
            key = (stream, label)

            # 去重：保留最新文件
            if key in seen and seen[key][1] >= mtime:
                continue
            seen[key] = (fpath, mtime)

    # 加载所有发现的联考表
    self.mock_tables = {}
    for (stream, label), (fpath, _) in seen.items():
        try:
            table = load_score_table(fpath)
            total = max(table[0].values()) if table[0] else 0
            self.mock_tables[(stream, label)] = (table, total)
        except Exception as e:
            self.load_errors.append(f"联考 {stream}_{label}: {e}")
```

### 5.6 `_resolve_mock_table()` 的 fallback 逻辑

用户选择联考时传入的 label 可能有多种写法，需要 fallback 匹配：

```python
def _resolve_mock_table(self, stream, label):
    """
    解析联考数据表，支持 fallback 匹配。

    匹配优先级：
    1. 精确匹配 (stream, label)
    2. 忽略空格/下划线差异的模糊匹配
    3. 只匹配 label 子串
    4. 该 stream 下的第一个联考表（最终 fallback）

    返回：(score_table, total) 或 None
    """
    # 精确匹配
    key = (stream, label)
    if key in self.mock_tables:
        return self.mock_tables[key]

    # 标准化后匹配
    norm_label = re.sub(r"[_\-\s]+", "", label)
    for (s, l), val in self.mock_tables.items():
        if s == stream and re.sub(r"[_\-\s]+", "", l) == norm_label:
            return val

    # 子串匹配
    for (s, l), val in self.mock_tables.items():
        if s == stream and (norm_label in re.sub(r"[_\-\s]+", "", l)
                           or re.sub(r"[_\-\s]+", "", l) in norm_label):
            return val

    # 最终 fallback：该 stream 下的第一个
    for (s, l), val in self.mock_tables.items():
        if s == stream:
            return val

    return None
```

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

### 6.2 `_build_tier_targets()` 余数分配算法

将 `top_n` 按比例拆分为整数名额，余数按优先级分配：

```python
def _build_tier_targets(self, top_n: int) -> dict:
    """
    将 top_n 按 TIER_DISTRIBUTION 比例分配为整数。

    余数分配策略：
    - 先按 floor() 分配基础名额
    - 剩余名额按「稳 > 保 > 冲」优先级逐个分配
    - 确保总和 == top_n

    示例：top_n=30
    - 冲: floor(30 * 0.25) = 7
    - 稳: floor(30 * 0.45) = 13
    - 保: floor(30 * 0.30) = 9
    - 总计 7+13+9 = 29，余 1
    - 余数分配给「稳」→ 稳=14
    - 最终：{"冲": 7, "稳": 14, "保": 9}
    """
    import math
    raw = {tier: top_n * pct for tier, pct in TIER_DISTRIBUTION.items()}
    result = {tier: math.floor(v) for tier, v in raw.items()}
    remainder = top_n - sum(result.values())

    # 余数按优先级分配
    priority = ["稳", "保", "冲"]
    for tier in priority:
        if remainder <= 0:
            break
        result[tier] += 1
        remainder -= 1

    return result
```

### 6.3 选取算法与 `pick_from()` 内部逻辑

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
    selected = []
    used_keys = set()        # 已选的 (院校代号, 专业代号)
    school_count = {}        # 每校已选专业数

    def pick_from(pool_df, need, enforce_school_limit=True):
        """
        从候选池中选取 need 个专业。

        参数：
        - pool_df: 候选 DataFrame，已按贴合度排序
        - need: 需要选取的数量
        - enforce_school_limit: 是否执行每校限制

        逻辑：
        1. 遍历 pool_df 每一行
        2. 跳过已选的 (院校代号, 专业代号)
        3. 如果 enforce_school_limit=True，检查该校已选数量
        4. 选中后更新 used_keys 和 school_count
        5. 达到 need 数量后停止
        """
        picked = 0
        for _, row in pool_df.iterrows():
            if picked >= need:
                break
            key = (row["院校代号"], row["专业代号"])
            if key in used_keys:
                continue
            school = row["院校代号"]
            if enforce_school_limit and school_count.get(school, 0) >= per_school_limit:
                continue
            selected.append(row)
            used_keys.add(key)
            school_count[school] = school_count.get(school, 0) + 1
            picked += 1
        return picked

    # 按分档分组
    tier_frames = {
        "冲": ranked_df[ranked_df["分档"] == "冲"].sort_values("贴合度"),
        "稳": ranked_df[ranked_df["分档"] == "稳"].sort_values("贴合度"),
        "保": ranked_df[ranked_df["分档"] == "保"].sort_values("贴合度"),
    }

    # 阶段 1：按分档选取
    for tier in ["冲", "稳", "保"]:
        pick_from(tier_frames[tier], desired[tier], enforce_school_limit=True)

    # 阶段 2：分档不够，从全排序补选
    if len(selected) < top_n:
        full_sorted = ranked_df.sort_values("贴合度")
        pick_from(full_sorted, top_n - len(selected), enforce_school_limit=True)

    # 阶段 3：放宽学校限制
    if len(selected) < top_n:
        pick_from(full_sorted, top_n - len(selected), enforce_school_limit=False)

    return pd.DataFrame(selected)
```

**排序依据——「贴合度」**：

```python
df["贴合度"] = (df["位次比"] - 1.0).abs()
```

贴合度 = |位次比 - 1.0|，越接近 0 表示该专业和考生实力最匹配。每个分档内按贴合度排序，优先推荐最贴合的选项。

### 6.4 志愿评估：`evaluate_wishlist()` 完整流程

当用户提供自己的志愿清单时，系统会分析梯度合理性并给出建议：

```python
def evaluate_wishlist(self, stream, user_rank, wishlist_items):
    """
    评估用户提交的志愿清单。

    参数：
    - stream: "历史" 或 "物理"
    - user_rank: 考生位次
    - wishlist_items: [{"school": "...", "major": "..."}, ...]

    流程：
    1. 为每个志愿项在录取表中匹配
    2. 计算位次比、分档、概率
    3. 生成梯度分析和建议

    匹配策略（代码匹配 vs 名称匹配）：
    - 优先使用院校代号 + 专业代号精确匹配
    - 代号为空时降级为名称模糊匹配
    """
    df = self._get_admission_df(stream)
    if df is None:
        return {"ok": False, "message": f"未找到{stream}类录取数据"}

    rows = []
    for item in wishlist_items:
        school = item.get("school", "").strip()
        major = item.get("major", "").strip()
        school_code = item.get("school_code", "").strip()
        major_code = item.get("major_code", "").strip()

        matched = None

        # 策略1：代号精确匹配
        if school_code and major_code:
            mask = (df["院校代号"] == school_code) & (df["专业代号"] == major_code)
            hits = df[mask]
            if len(hits) > 0:
                matched = hits.iloc[0]

        # 策略2：名称模糊匹配
        if matched is None and school and major:
            mask = df["院校名称"].str.contains(school, na=False)
            if major:
                mask = mask & df["专业名称"].str.contains(major, na=False)
            hits = df[mask]
            if len(hits) > 0:
                # 取位次比最接近 1.0 的（最贴合）
                hits = hits.copy()
                hits["_temp_ratio"] = hits["最低位次"].apply(
                    lambda r: abs(user_rank / r - 1.0) if r and r > 0 else 999
                )
                matched = hits.sort_values("_temp_ratio").iloc[0]

        # 策略3：仅校名匹配（最宽松）
        if matched is None and school:
            mask = df["院校名称"].str.contains(school, na=False)
            hits = df[mask]
            if len(hits) > 0:
                matched = hits.iloc[0]

        if matched is not None:
            min_rank = to_int(matched.get("最低位次"), 0)
            ratio = user_rank / min_rank if min_rank > 0 else 999
            rows.append({
                "school_name": matched["院校名称"],
                "major_name": matched["专业名称"],
                "min_score": to_int(matched.get("最低分")),
                "min_rank": min_rank,
                "rank_ratio": round(ratio, 4),
                "tier": calc_tier(ratio),
                "probability": calc_probability(ratio),
                "matched": True,
            })
        else:
            rows.append({
                "school_name": school or school_code,
                "major_name": major or major_code,
                "matched": False,
                "tier": "未知",
                "probability": None,
            })

    # 生成梯度建议
    tips = self._build_gradient_tips(rows)

    return {
        "ok": True,
        "rows": rows,
        "tips": tips,
        "tier_stats": _count_tiers(rows),
    }
```

### 6.5 梯度建议生成

```python
def _build_gradient_tips(rows):
    tips = []
    valid = [r for r in rows if r.get("matched")]
    if not valid:
        tips.append("未能匹配到任何志愿，请检查院校/专业名称。")
        return tips

    total = len(valid)
    tier_count = _count_tiers(valid)
    probabilities = [r["probability"] for r in valid if r.get("probability")]

    if tier_count["保"] == 0:
        tips.append("当前志愿缺少保底项，建议至少增加 1-2 个保底专业。")

    if tier_count["冲"] / total > 0.6:
        tips.append("冲刺项占比偏高，整体风险较大。")

    if tier_count["冲"] == 0 and tier_count["保"] / total >= 0.7:
        tips.append("方案偏保守，可加入少量冲刺项提升上限。")

    if probabilities and max(probabilities) - min(probabilities) < 0.2:
        tips.append("志愿概率分布较集中，梯度不够明显。")

    first_three = valid[:3]
    if all(r["tier"] == "冲" for r in first_three):
        tips.append("前 3 志愿均为冲刺，建议前段加入至少 1 个稳妥项。")

    if not tips:
        tips.append("志愿梯度分布合理。")

    return tips
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

### 7.2 表单标准化：`normalize_form_payload()`

SSR 表单和 JSON API 的输入格式不同，需要统一标准化：

```python
def normalize_form_payload(request) -> dict:
    """
    将 form 表单或 JSON 请求体统一为标准 dict。

    处理：
    - POST form → request.form（ImmutableMultiDict）
    - POST JSON → request.get_json()
    - GET 参数 → request.args

    标准化：
    - stream: 统一为 "历史"/"物理"/"艺术"
    - score: 转为 float
    - rank: 转为 int（可选）
    - top_n: 转为 int，默认 30，限制 [1, 200]
    - school_keyword / major_keyword: 去空白
    """
    if request.is_json:
        raw = request.get_json(silent=True) or {}
    elif request.method == "POST":
        raw = dict(request.form)
    else:
        raw = dict(request.args)

    # 标准化 stream
    stream = normalize_stream(raw.get("stream", ""))

    # 标准化 score
    score_str = str(raw.get("score", "")).strip()
    score = None
    if score_str:
        try:
            score = float(score_str)
        except ValueError:
            pass

    # 标准化 rank
    rank_str = str(raw.get("rank", "")).strip()
    rank = None
    if rank_str:
        try:
            rank = int(float(rank_str))
        except ValueError:
            pass

    # 标准化 top_n
    try:
        top_n = int(raw.get("top_n", 30))
    except (ValueError, TypeError):
        top_n = 30
    top_n = max(1, min(200, top_n))

    return {
        "stream": stream,
        "score": score,
        "rank": rank,
        "top_n": top_n,
        "school_keyword": str(raw.get("school_keyword", "")).strip(),
        "major_keyword": str(raw.get("major_keyword", "")).strip(),
        # 艺术类字段
        "art_mode": raw.get("art_mode", ""),
        "culture_score": raw.get("culture_score", ""),
        "art_score": raw.get("art_score", ""),
        # 联考字段
        "mock_label": raw.get("mock_label", ""),
        "mock_score": raw.get("mock_score", ""),
    }
```

`normalize_stream()` 处理多种科类输入格式的兼容：

```python
def normalize_stream(val: str) -> str:
    """
    标准化科类输入。
    "历史类" → "历史"
    "物理类" → "物理"
    "hist" / "history" → "历史"
    "phys" / "physics" → "物理"
    "art" / "艺术类" → "艺术"
    """
    v = str(val).strip().lower()
    if v in ("历史", "历史类", "hist", "history"):
        return "历史"
    if v in ("物理", "物理类", "phys", "physics"):
        return "物理"
    if v in ("艺术", "艺术类", "art"):
        return "艺术"
    return val.strip()
```

### 7.3 API 设计示例

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

### 7.4 分数显示处理：`score_display()`

不同来源的分数需要不同的显示格式：

```python
def score_display(score, source="gaokao"):
    """
    格式化分数显示。

    规则：
    - 整数分显示为整数（580 → "580"）
    - 艺术类综合分保留 2 位小数（195.50 → "195.50"）
    - None / NaN → "-"
    - 联考换算分标注来源（"≈580（联考换算）"）
    """
    if score is None or (isinstance(score, float) and math.isnan(score)):
        return "-"

    if source == "art":
        return f"{score:.2f}"

    if isinstance(score, float) and score == int(score):
        return str(int(score))

    if source == "mock":
        return f"≈{int(score)}（联考换算）"

    return str(score)
```

### 7.5 志愿导出行构建：`build_export_rows()`

将推荐结果转换为 Excel 导出格式：

```python
def build_export_rows(payload: dict) -> list:
    """
    构建导出行列表。

    输入 payload 包含：
    - selected_items: [{school_name, major_name, min_score, ...}, ...]
    - stream: 科类
    - score: 考生分数
    - user_rank: 考生位次

    输出：每行包含序号、院校代号、院校名称、专业代号、专业名称、
          录取最低分、录取最低位次、位次比、分档、录取概率

    额外处理：
    - 按分档排序（冲→稳→保）
    - 每档内按概率降序
    - 添加分档分隔标题行
    """
    items = payload.get("selected_items", [])
    if not items:
        return []

    # 按分档分组
    tier_order = {"冲": 0, "稳": 1, "保": 2}
    items_sorted = sorted(items, key=lambda x: (
        tier_order.get(x.get("tier", ""), 9),
        -(x.get("probability", 0) or 0)
    ))

    rows = []
    current_tier = None
    seq = 0

    for item in items_sorted:
        tier = item.get("tier", "")

        # 分档标题行
        if tier != current_tier:
            current_tier = tier
            rows.append({
                "序号": "",
                "院校代号": f"── {tier} ──",
                "院校名称": "",
                "专业代号": "",
                "专业名称": "",
                "录取最低分": "",
                "录取最低位次": "",
                "位次比": "",
                "分档": "",
                "录取概率": "",
            })

        seq += 1
        prob = item.get("probability")
        rows.append({
            "序号": seq,
            "院校代号": item.get("school_code", ""),
            "院校名称": item.get("school_name", ""),
            "专业代号": item.get("major_code", ""),
            "专业名称": item.get("major_name", ""),
            "录取最低分": item.get("min_score", ""),
            "录取最低位次": item.get("min_rank", ""),
            "位次比": f"{item.get('rank_ratio', ''):.4f}" if item.get("rank_ratio") else "",
            "分档": tier,
            "录取概率": f"{prob:.1%}" if prob else "",
        })

    return rows
```

### 7.6 安全设计

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

### 7.7 管理后台 Action 处理

管理后台通过 POST action 参数分发不同的操作，完整处理流程：

```python
@app.route(ADMIN_PATH, methods=["GET", "POST"])
def admin_page():
    message = ""

    if request.method == "POST":
        action = request.form.get("action", "")

        # ── 登录 ──
        if action == "login":
            pwd = request.form.get("password", "")
            if pwd == os.environ.get("ADMIN_PASSWORD", ""):
                session["admin_ok"] = True
                message = "登录成功"
            else:
                message = "密码错误"
            return render_template("admin.html", message=message)

        # 以下 action 需要认证
        if not is_admin_authed():
            return redirect(ADMIN_PATH)

        # ── 登出 ──
        if action == "logout":
            session.pop("admin_ok", None)
            return redirect(ADMIN_PATH)

        # ── 保存站点元数据 ──
        elif action == "save_site_meta":
            title = request.form.get("site_title", "").strip()
            desc = request.form.get("site_desc", "").strip()
            # 写入 site_meta.json
            meta = {"title": title, "description": desc}
            with open(os.path.join(BASE_DIR, "site_meta.json"), "w",
                      encoding="utf-8") as f:
                json.dump(meta, f, ensure_ascii=False, indent=2)
            message = "站点信息已保存"

        # ── 保存关于页 ──
        elif action == "save_about":
            about_text = request.form.get("about_content", "")
            with open(os.path.join(BASE_DIR, "about.md"), "w",
                      encoding="utf-8") as f:
                f.write(about_text)
            message = "关于页已保存"

        # ── 上传联考数据 ──
        elif action == "upload_mock":
            stream = normalize_stream(request.form.get("mock_stream", ""))
            label = safe_label(request.form.get("mock_label", ""))
            file = request.files.get("mock_file")

            if not file or not file.filename.endswith(".xlsx"):
                message = "请上传 .xlsx 文件"
            elif not stream or not label:
                message = "请选择科类并输入考试名称"
            else:
                filename = f"{stream}_{label}.xlsx"
                save_path = os.path.join(BASE_DIR, MOCK_SCORE_DIR, filename)
                os.makedirs(os.path.dirname(save_path), exist_ok=True)
                file.save(save_path)
                engine.reload()  # 热重载
                message = f"已上传并加载: {filename}"

        # ── 删除联考数据 ──
        elif action == "delete_mock":
            key = request.form.get("mock_key", "")
            if key:
                target_path = os.path.normpath(
                    os.path.join(BASE_DIR, key)
                )
                allowed_root = os.path.normpath(
                    os.path.join(BASE_DIR, MOCK_SCORE_DIR)
                )
                if target_path.startswith(allowed_root) and os.path.exists(target_path):
                    os.remove(target_path)
                    engine.reload()
                    message = f"已删除: {os.path.basename(target_path)}"
                else:
                    message = "删除失败：非法路径。"

    # GET 或 POST 后渲染
    return render_template("admin.html",
        authed=is_admin_authed(),
        message=message,
        mock_files=_list_mock_files(),
        engine_status=engine.get_status(),
    )
```

### 7.8 志愿导出

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

### 8.3 SPA 版 `spa.js` 完整算法重实现

SPA 版是一个纯浏览器端实现，不需要任何后端服务器。819 行 JavaScript 完整复刻了 Python 后端的所有核心算法。以下是各核心模块的对比：

#### 8.3.1 双模式数据加载：`readWorkbook()` / `readWorkbookFromPath()`

```javascript
// spa.js — 双模式数据加载

/**
 * 模式1：从用户选择的本地文件读取 Excel
 * 用于离线场景，用户手动选择数据文件
 */
function readWorkbook(file) {
    return new Promise((resolve, reject) => {
        const reader = new FileReader();
        reader.onload = (e) => {
            try {
                const data = new Uint8Array(e.target.result);
                const wb = XLSX.read(data, { type: 'array' });
                resolve(wb);
            } catch (err) {
                reject(new Error(`解析 Excel 失败: ${err.message}`));
            }
        };
        reader.onerror = () => reject(new Error('文件读取失败'));
        reader.readAsArrayBuffer(file);
    });
}

/**
 * 模式2：从 URL 路径远程加载 Excel
 * 用于部署场景，数据文件放在 CDN 上
 */
async function readWorkbookFromPath(url) {
    const response = await fetch(url);
    if (!response.ok) {
        throw new Error(`加载 ${url} 失败: HTTP ${response.status}`);
    }
    const buffer = await response.arrayBuffer();
    return XLSX.read(new Uint8Array(buffer), { type: 'array' });
}
```

#### 8.3.2 Hash 路由：`setupRouter()`

SPA 使用 `location.hash` 实现简易路由，无需服务器支持：

```javascript
// spa.js — hash 路由

const ROUTES = {
    '': 'predict',          // 默认页：高考分预测
    '#predict': 'predict',
    '#mock': 'mock',        // 联考换算
    '#about': 'about',      // 关于
};

function setupRouter() {
    function navigate() {
        const hash = location.hash || '';
        const page = ROUTES[hash] || 'predict';

        // 隐藏所有页面
        document.querySelectorAll('.page-section').forEach(el => {
            el.style.display = 'none';
        });

        // 显示目标页面
        const target = document.getElementById(`page-${page}`);
        if (target) {
            target.style.display = 'block';
        }

        // 更新导航栏高亮
        document.querySelectorAll('.nav-link').forEach(el => {
            el.classList.toggle('active', el.getAttribute('href') === (hash || '#predict'));
        });
    }

    window.addEventListener('hashchange', navigate);
    navigate();  // 初始导航
}
```

#### 8.3.3 艺术类表单联动：`setupArtForm()`

选择「艺术类」时动态显示/隐藏艺术类专属字段：

```javascript
// spa.js — 艺术类表单联动

function setupArtForm() {
    const streamSelect = document.getElementById('stream-select');
    const artFields = document.getElementById('art-fields');
    const scoreField = document.getElementById('score-field');
    const rankField = document.getElementById('rank-field');

    streamSelect.addEventListener('change', () => {
        const isArt = streamSelect.value === '艺术';

        // 艺术类：显示文化分+专业分，隐藏常规分数
        artFields.style.display = isArt ? 'block' : 'none';
        scoreField.style.display = isArt ? 'none' : 'block';

        // 艺术类不需要手动输入位次
        if (isArt) {
            rankField.value = '';
            rankField.disabled = true;
        } else {
            rankField.disabled = false;
        }
    });
}
```

#### 8.3.4 表格渲染：`renderTable()`

推荐结果渲染到 HTML 表格，包含分档配色和勾选框：

```javascript
// spa.js — 表格渲染

function renderTable(recommendations) {
    const tbody = document.getElementById('result-tbody');
    tbody.innerHTML = '';

    const tierColors = {
        '冲': '#ff6b6b',   // 红色
        '稳': '#ffd43b',   // 黄色
        '保': '#51cf66',   // 绿色
    };

    recommendations.forEach((item, index) => {
        const tr = document.createElement('tr');

        // 分档背景色
        const tierColor = tierColors[item.tier] || '#888';
        tr.style.borderLeft = `4px solid ${tierColor}`;

        tr.innerHTML = `
            <td><input type="checkbox" class="wish-check"
                       data-index="${index}" /></td>
            <td>${index + 1}</td>
            <td>${item.school_name}</td>
            <td>${item.major_name}</td>
            <td>${item.min_score || '-'}</td>
            <td>${item.min_rank || '-'}</td>
            <td>${item.rank_ratio?.toFixed(4) || '-'}</td>
            <td><span class="tier-badge" style="background:${tierColor}">
                ${item.tier}</span></td>
            <td>${item.probability ? (item.probability * 100).toFixed(1) + '%' : '-'}</td>
        `;

        tbody.appendChild(tr);
    });

    // 绑定全选/分档筛选
    bindCheckAll();
    bindTierFilter();
}
```

#### 8.3.5 分档筛选：`bindTierFilter()`

用户可以按分档筛选显示结果：

```javascript
// spa.js — 分档筛选

function bindTierFilter() {
    const filterBtns = document.querySelectorAll('.tier-filter-btn');

    filterBtns.forEach(btn => {
        btn.addEventListener('click', () => {
            const tier = btn.dataset.tier;  // "冲" / "稳" / "保" / "all"

            // 更新按钮状态
            filterBtns.forEach(b => b.classList.remove('active'));
            btn.classList.add('active');

            // 显示/隐藏行
            const rows = document.querySelectorAll('#result-tbody tr');
            rows.forEach(row => {
                if (tier === 'all') {
                    row.style.display = '';
                } else {
                    const badge = row.querySelector('.tier-badge');
                    const rowTier = badge ? badge.textContent.trim() : '';
                    row.style.display = (rowTier === tier) ? '' : 'none';
                }
            });

            // 更新勾选状态
            syncWishlistUi();
        });
    });
}
```

### 8.4 微信小程序

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

## 9. 前端交互细节：志愿勾选与导出机制

### 9.1 全选/部分选控制

推荐结果表格支持全选、部分选、分档选的复合勾选机制：

```javascript
// index.html 内联 JS — 志愿勾选系统

/**
 * 全选 checkbox 控制
 * 点击「全选」时只勾选当前可见的行（分档筛选后）
 */
function bindCheckAll() {
    const checkAll = document.getElementById('check-all');
    if (!checkAll) return;

    checkAll.addEventListener('change', () => {
        const checked = checkAll.checked;
        const visibleChecks = getVisibleChecks();
        visibleChecks.forEach(cb => { cb.checked = checked; });
        syncWishlistUi();
    });
}

/**
 * 获取当前可见行的勾选框
 * 分档筛选后只返回未被 display:none 隐藏的行
 */
function getVisibleChecks() {
    return Array.from(
        document.querySelectorAll('#result-tbody tr')
    ).filter(tr => tr.style.display !== 'none')
     .map(tr => tr.querySelector('.wish-check'))
     .filter(Boolean);
}

/**
 * 同步志愿清单 UI 状态
 * 更新：已选数量 / 导出按钮状态 / 全选框状态
 */
function syncWishlistUi() {
    const allChecks = document.querySelectorAll('.wish-check');
    const checkedCount = Array.from(allChecks).filter(cb => cb.checked).length;
    const visibleChecks = getVisibleChecks();
    const visibleCheckedCount = visibleChecks.filter(cb => cb.checked).length;

    // 更新已选数量
    const countEl = document.getElementById('selected-count');
    if (countEl) {
        countEl.textContent = checkedCount;
    }

    // 导出按钮状态
    const exportBtn = document.getElementById('export-btn');
    if (exportBtn) {
        exportBtn.disabled = checkedCount === 0;
    }

    // 全选框：全部可见行都选中 → checked，部分选中 → indeterminate
    const checkAll = document.getElementById('check-all');
    if (checkAll && visibleChecks.length > 0) {
        if (visibleCheckedCount === visibleChecks.length) {
            checkAll.checked = true;
            checkAll.indeterminate = false;
        } else if (visibleCheckedCount > 0) {
            checkAll.checked = false;
            checkAll.indeterminate = true;
        } else {
            checkAll.checked = false;
            checkAll.indeterminate = false;
        }
    }
}
```

### 9.2 分档筛选联动

`applyTierFilter()` 在前端模板版中的完整实现：

```javascript
/**
 * 应用分档筛选
 * 与 SPA 版逻辑一致，但操作 Jinja2 渲染的表格
 */
function applyTierFilter(tier) {
    const rows = document.querySelectorAll('#result-tbody tr[data-tier]');

    rows.forEach(row => {
        if (tier === 'all') {
            row.style.display = '';
        } else {
            row.style.display = (row.dataset.tier === tier) ? '' : 'none';
        }
    });

    // 更新筛选按钮高亮
    document.querySelectorAll('.tier-filter-btn').forEach(btn => {
        btn.classList.toggle('active', btn.dataset.tier === tier);
    });

    // 同步勾选状态（被隐藏的行不参与全选计数）
    syncWishlistUi();
}
```

### 9.3 导出流程

用户勾选推荐项后点击导出，收集所有选中项的数据发送到后端：

```javascript
/**
 * 导出选中的志愿为 Excel
 */
function exportWishlist() {
    const checked = document.querySelectorAll('.wish-check:checked');
    if (checked.length === 0) {
        alert('请先勾选要导出的志愿');
        return;
    }

    // 收集选中项的数据
    const items = Array.from(checked).map(cb => {
        const tr = cb.closest('tr');
        return {
            school_code: tr.dataset.schoolCode || '',
            school_name: tr.querySelector('.col-school')?.textContent || '',
            major_code: tr.dataset.majorCode || '',
            major_name: tr.querySelector('.col-major')?.textContent || '',
            min_score: tr.dataset.minScore || '',
            min_rank: tr.dataset.minRank || '',
            rank_ratio: tr.dataset.rankRatio || '',
            tier: tr.dataset.tier || '',
            probability: tr.dataset.probability || '',
        };
    });

    // 构建表单并提交（触发文件下载）
    const form = document.createElement('form');
    form.method = 'POST';
    form.action = '/api/export-wishlist';

    const input = document.createElement('input');
    input.type = 'hidden';
    input.name = 'payload';
    input.value = JSON.stringify({
        selected_items: items,
        stream: document.getElementById('stream-select')?.value || '',
        score: document.getElementById('score-input')?.value || '',
    });

    form.appendChild(input);
    document.body.appendChild(form);
    form.submit();
    document.body.removeChild(form);
}
```

---

## 10. 问题、踩坑与迭代优化

### 10.1 模型迭代

| 版本 | 分档阈值 | 概率模型 | 改进 |
|------|---------|---------|------|
| CLI v1 | SAFE=0.90 | 固定值（保70%/稳50%/冲30%） | 初版，概率粗糙 |
| Web v1 | SAFE=0.80 | 分段线性（5区间） | 更精细的概率映射 |
| Web v1.1 | SAFE=0.80 | 同上 + 联考换算 | 添加诊断考试支持 |

**为什么从固定概率升级到分段线性？** 固定概率模型只有三个值（30%/50%/70%），无法区分「刚好过线的稳」和「远超录取线的稳」。分段线性模型在每个档位内提供连续的概率值，让考生更直观地感受不同选项的风险差异。

### 10.2 PDF 解析踩坑

1. **全角空格**：PDF 中的空格实际是 `\u3000`（全角空格），需要统一替换
2. **续行误判**：部分院校代号只有一个字符（如「A」），容易被误判为续行文本。通过 `looks_numeric_or_placeholder()` 辅助判断
3. **表头重复**：每页 PDF 都重复表头，需要 `is_header_row()` 检测跳过
4. **列数不固定**：不同页面的列数可能不同，`align_row()` 的启发式对齐解决了这个问题

### 10.3 精度问题

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

### 10.4 `normalizeCol` 空白处理踩坑

```python
# 踩坑：Excel 列名末尾包含不可见零宽字符
# 症状：df["最低分"] 报 KeyError，但 print(df.columns) 看起来完全正确

# 原因：openpyxl 读取某些 Excel 文件时，列名末尾会附带
# Zero-Width Space (\u200b) 或 BOM (\ufeff)

# 解决：normalizeCol() 中增加不可见字符清理
def normalizeCol(col: str) -> str:
    if not col:
        return ""
    col = str(col).strip()
    col = col.replace("\u3000", " ")
    col = col.replace("\n", " ")
    # 清理零宽字符
    col = col.replace("\u200b", "")
    col = col.replace("\ufeff", "")
    col = col.replace("\u200c", "")
    col = col.replace("\u200d", "")
    col = re.sub(r"\s+", " ", col)
    return col.strip()
```

### 10.5 `normalize_stream` 多格式兼容

实际用户输入千奇百怪，需要兼容：

```python
# 踩坑日志：用户输入了各种格式的科类
# "历史 类"（中间有空格）
# "物理 "（末尾有空格）
# "HIST"（全大写英文）
# " 历史类 "（前后空格）
# "历 史"（中间有全角空格）

# 解决：normalize_stream 增加预处理
def normalize_stream(val: str) -> str:
    v = str(val).strip()
    v = v.replace("\u3000", "")    # 去掉全角空格
    v = v.replace(" ", "")         # 去掉所有空格
    v = v.lower()
    if v in ("历史", "历史类", "hist", "history"):
        return "历史"
    if v in ("物理", "物理类", "phys", "physics"):
        return "物理"
    if v in ("艺术", "艺术类", "art"):
        return "艺术"
    return val.strip()
```

### 10.6 Excel 列宽自动计算

```python
# 踩坑：中文字符在 Excel 中的宽度计算

# 问题：直接用 len(str) 计算列宽，中文列会很窄
# 因为一个中文字符视觉宽度约等于 2 个英文字符

# 解决：按字符类型加权计算宽度
def calc_column_width(values: list) -> int:
    """计算 Excel 列宽，中文字符按 2 倍宽度计算。"""
    def char_width(s):
        width = 0
        for c in str(s):
            if ord(c) > 127:
                width += 2   # 中文/全角字符
            else:
                width += 1   # ASCII 字符
        return width

    max_width = max(char_width(v) for v in values) if values else 10
    return max(10, min(40, max_width + 2))
```

### 10.7 gunicorn gthread 配置选择

```python
# Procfile
web: gunicorn app:app --workers 2 --threads 4 --worker-class gthread --bind 0.0.0.0:$PORT
```

**为什么选择 `gthread` 而不是 `gevent`？**

| 配置 | worker_class | 说明 |
|------|-------------|------|
| sync | 同步 worker | 每个请求独占一个进程，pandas 操作安全但并发低 |
| gevent | 协程 | 高并发但 pandas 不是 gevent 友好的（某些操作会阻塞事件循环） |
| **gthread** | **线程** | **pandas 操作在线程间安全（GIL 保护），中等并发** |

选择 `gthread` 的原因：
1. pandas 的 DataFrame 操作是 CPU 密集型，gevent 的协程模型无法加速
2. `AdmissionEngine` 使用内存驻留数据，线程间共享读取无需锁
3. 2 workers × 4 threads = 8 并发足够应对高考季峰值

### 10.8 热重载

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

## 11. 附录与参考

### 11.1 核心数据流总览

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

### 11.2 概率解释映射

```python
PROBABILITY_GUIDE = [
    {"min": 0.75, "label": "高把握",  "desc": "基本属于保底区间，可优先考虑专业偏好"},
    {"min": 0.60, "label": "较稳妥",  "desc": "录取机会较高，适合作为稳妥志愿"},
    {"min": 0.45, "label": "可尝试",  "desc": "存在机会，建议与保底志愿搭配"},
    {"min": 0.30, "label": "冲刺项",  "desc": "风险偏高，建议放在前段冲刺"},
    {"min": 0.00, "label": "高风险",  "desc": "录取不确定性较大，谨慎填报"},
]
```

### 11.3 关键文件索引

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
