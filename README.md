# E-SMILES (Extended SMILES) 格式规范 | E-SMILES Format Specification

**Date:** 2026-05-30
**Status:** Specification
**Scope:** Molecular representation for Markush structures, attachment points, and abstract rings in chemical documents
本仓库仅为个人对该格式的理解不代表完全正确  
---

## 概述 | Overview

E-SMILES (Extended SMILES) 是一种分子表示格式，通过 XML 风格扩展标签，在标准 RDKit 兼容 SMILES 字符串的基础上，表达无法用正则 SMILES 单独表示的化学结构——具体包括：**Markush 结构**、**连接点/虚拟原子**、**抽象环**。

> **论文来源：** MolParser: End-to-end Visual Recognition of Molecule Structures in the Wild — Section 3.1 "E-SMILES: Extended SMILES"
> 原文链接：https://arxiv.org/abs/2411.11098
> PDF 下载：[2411.11098.pdf](https://arxiv.org/pdf/2411.11098.pdf)

---

## 何时使用 E-SMILES | When to Use E-SMILES

| 场景 | 标准 SMILES | E-SMILES |
|------|-------------|----------|
| 完整定义的分子（无 Markush 特征） | ✅ | ❌（退化为标准 SMILES） |
| Markush R 基团（特定原子位点） | ❌ | ✅ |
| 连接点 / 虚拟原子 | ❌ | ✅ |
| 抽象环（Ar、B 等） | ❌ | ✅ |
| 环上不确定连接位点 | ❌ | ✅ |

---

## 一、总体格式 | 1. Overall Format

E-SMILES 的通用结构为：

```
SMILES<sep>EXTENSION
```

| 组成部分 | 是否必需 | 说明 |
|----------|----------|------|
| `SMILES` | **是** | 标准 RDKit 兼容 SMILES 字符串 |
| `<sep>` | **条件必需** | 分隔符。仅当 EXTENSION 存在时使用 |
| `EXTENSION` | 否 | XML 风格标签，描述 Markush 特征。无此类特征时省略 |

### 格式规则 | Format Rules

1. **无 Markush 特征时**，EXTENSION 为空，整个字符串退化为标准 SMILES：
   ```
   CC(=O)O   ← 合法的 E-SMILES（退化为标准 SMILES）{但是根据该论文发表的数据库来看，所有的smiles都会添加<sep>，以确保格式正确}
   ```
2. **存在 Markush 特征时**，格式 **必须** 包含 `<sep>`：
   ```
   *c1ccccc1<sep><a>0:R[1]</a>
   ```
3. SMILES 部分 **必须** 能被 RDKit 独立解析（不带 EXTENSION）。

---

## 二、SMILES 部分规则 | 2. SMILES Portion Rules

### 2.1 RDKit 兼容性

SMILES 部分 **必须** 是有效的 RDKit SMILES 字符串。RDKit 仅使用此部分进行：
- 分子对象创建（`Chem.MolFromSmiles`）
- 结构图像渲染
- 标准 SMILES 生成
- 所有 RDKit 属性计算

### 2.2 Root 原子（非 Markush 分子）

对于嵌入 E-SMILES 的完全定义（非 Markush）分子，建议使用 `rootedAtAtom=0` 生成 SMILES，以保证一致性：

```python
from rdkit import Chem

mol = Chem.MolFromSmiles("CC(=O)O", sanitize=False)
smiles = Chem.MolToSmiles(mol, rootedAtAtom=0)
# → 'CC(=O)O'
```

### 2.3 虚拟原子（`*`）作为占位符

对于 Markush 结构，**虚拟原子**（`*`）必须插入到 SMILES 部分中，用以占据以下位置：
- R 基团取代位点
- 连接点 / 虚拟原子位点
- 抽象环连接位点

> **重要：** `*` 原子仅为**占位符**，其化学含义完全由对应的 EXTENSION 标签定义。RDKit 将 `*` 解析为原子序为 0 的虚拟原子。

| 场景 | SMILES 写法 | EXTENSION 标签 |
|------|-----------|----------------|
| 原子位点被 R 基团占据 | `*` | `<a>INDEX:R[n]</a>` |
| 连接点（开放化学键） | `*` | `<a>INDEX:<dum></a>` |
| 环上不确定连接 | 环正常写出 | `<r>RING_INDEX:GROUP</r>` |

---

## 三、EXTENSION 部分规则 | 3. EXTENSION Portion Rules

### 3.1 标签类型总览 | Tag Type Overview

EXTENSION 采用 XML 风格标签，共 **三种主要标签类型** 和 **一个特殊标记**：

| 标签 | 完整形式 | 含义 | 适用场景 |
|------|----------|------|----------|
| `<a>` | `<a>ATOM_INDEX:GROUP</a>` | 原子取代基（Atom substituent） | 特定原子上的 R 基团、缩写基团、连接点 |
| `<r>` | `<r>RING_INDEX:GROUP</r>` | 环不确定连接（Ring uncertain attachment） | 基团连接在环的某个未指定位置 |
| `<c>` | `<c>CIRCLE_INDEX:NAME</c>` | 抽象环（Abstract ring / Circle） | 用命名圆圈表示的抽象环（如 B、Ar） |
| `<dum>` | `<a>ATOM_INDEX:<dum></a>` | 连接点（Connection point / Dummy） | 表示化学键的开放端点 |

### 3.2 通用字段格式 | Common Field Format

三种标签内部均使用如下格式：

```
INDEX:GROUP_NAME
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `INDEX` | 整数（从 0 开始） | SMILES 字符串中的原子索引，或环索引 |
| `GROUP_NAME` | 字符串 | 基团名称 |

### 3.3 基团名称选项 | Group Name Options

`GROUP_NAME` 字段接受以下值：

| 基团类型 | 示例 | 说明 |
|----------|------|------|
| Markush R 基团 | `R`, `R[1]`, `R[2]`, `R[3]` | 方括号表示上下标 |
| 卤素基团 | `X`, `Y`, `Z` | |
| 常见缩写基团 | `Ph`, `Me`, `OMe`, `Et`, `Pr`, `Bu`, `CF3` | 标准化学缩写 |
| 连接点 | `<dum>` | 特殊 token，带尖括号 |
| 一般取代基 | 任意字符串 | 自由形式名称 |

---

## 四、各标签详解 | 4. Tag Specifications

### 4.1 `<a>` — 原子取代基标签

**用途：** 声明 SMILES 字符串中 `ATOM_INDEX` 位置的原子为取代基（R 基团、缩写或连接点）。

**语法：**
```
<a>ATOM_INDEX:GROUP_NAME</a>
```

**规则：**
1. `ATOM_INDEX` 是 SMILES 字符串中 `*` 原子的 0 基位置索引
2. `GROUP_NAME` 遵循 §3.3 中的选项
3. 多个 `<a>` 标签可引用同一个 `ATOM_INDEX`（同一原子上有多个取代基）
4. 多个 `<a>` 标签可对应不同 `ATOM_INDEX` 值

**示例：**

单 R 基团取代：
```
*c1ccccc1<sep><a>0:R[1]</a>
```
- SMILES 索引 0 的 `*` 声明为 R¹

带开放连接点的羧酸（**按论文图 4 原文**）：
```
*C(=O)=O<sep><a>0:<dum></a>
```
- SMILES 索引 0 的 `*` 声明为连接点（开放化学键）
- 原文使用 `*C(=O)=O`，本规范严格遵循论文文本

> **注：** 从羧酸化学结构角度，`*C(=O)O` 和 `*C(=O)=O` 均可能，但若声称"完全参考论文"，应以论文图 4 中的文本为准。

多取代基：
```
*c1ccc(C(*)cc1)cc2<sep><a>0:R[1]</a><a>5:R[2]</a>
```
- 索引 0 → R¹
- 索引 5 → R²

---

### 4.2 `<r>` — 环不确定连接标签

**用途：** 声明某基团连接在**环的未指定位置**（非特定原子）。

**语法：**
```
<r>RING_INDEX:GROUP_NAME</r>
```

**规则：**
1. `RING_INDEX` 是目标环的 0 基索引（非原子索引）
2. 基团连接在该环的**任意可用位置**（位置不确定）
3. 多个 `<r>` 标签可引用同一个 `RING_INDEX`（多个基团连接在同一环的不同不确定位置）
4. SMILES 中的环部分正常写出（无需 `*`）

**示例：**

带两个不确定取代基的苯环：
```
c1ccccc1<sep><r>0:R[1]</r><r>0:R[2]</r>
```
- RING INDEX 0 = 苯环
- R¹ 和 R² 均以不确定位置连接在该环上

不确定连接带重复标记（论文复杂 Markush 示例）：
```
c1ccccc1<sep><r>0:R[1]</r><r>0:R[2]?n</r>
```
- `?n` 后缀表示重复单元标记

---

### 4.3 `<c>` — 抽象环标签

**用途：** 声明命名抽象环（如 Ar 表示芳基、B 表示苯环等）。

**语法：**
```
<c>CIRCLE_INDEX:CIRCLE_NAME</c>
```

**规则：**
1. `CIRCLE_INDEX` 是抽象环的 0 基索引
2. `CIRCLE_NAME` 是环的名称（常用 `B`、`Ar` 等）
3. 抽象环在 SMILES 中**没有具体原子**，仅为概念性表示

**示例：**

名称为 B 的抽象环：
```
<sep><c>0:B</c>
```
- CIRCLE INDEX 0 声明为抽象环 B

---

### 4.4 `<dum>` — 连接点标记

**用途：** 声明某原子为**开放连接点**（化学键的开放端点）。

**语法：** 仅作为 `<a>` 标签内的 GROUP NAME 值使用，**不是独立标签**：
```
<a>ATOM_INDEX:<dum></a>
```

**注意：** `<dum>` 本身**带尖括号**，是论文规范中明确的特殊 token，并非"不带尖括号"。

---

## 五、完整示例 | 5. Complete Examples

### 示例 1：苯环单取代（简单 Markush）

**结构：** 苯环上有一个 Markush R¹ 取代基。

```
*c1ccccc1<sep><a>0:R[1]</a>
```

| 部分 | 内容 | 说明 |
|------|------|------|
| SMILES | `*c1ccccc1` | `*` 是索引 0 的 Markush 位点 |
| 分隔符 | `<sep>` | 存在 EXTENSION，必须包含 |
| Extension | `<a>0:R[1]</a>` | 原子索引 0 为 R¹ |

---

### 示例 2：带连接点的羧酸（**按论文原文**）

**结构：** 羧基带一个开放连接点。

```
*C(=O)=O<sep><a>0:<dum></a>
```

| 部分 | 内容 | 说明 |
|------|------|------|
| SMILES | `*C(=O)=O` | `*` 是索引 0 的连接点 |
| 分隔符 | `<sep>` | 存在 EXTENSION，必须包含 |
| Extension | `<a>0:<dum></a>` | 原子索引 0 为连接点 |

---

### 示例 3：苯环两个不确定取代基

**结构：** 苯环上有两个未指定位置的取代基。

```
c1ccccc1<sep><r>0:R[1]</r><r>0:R[2]</r>
```

| 部分 | 内容 | 说明 |
|------|------|------|
| SMILES | `c1ccccc1` | 标准苯环，无需占位符 |
| 分隔符 | `<sep>` | 存在 EXTENSION，必须包含 |
| Extension | `<r>0:R[1]</r><r>0:R[2]</r>` | 两基团均以不确定位置连接在环 0 上 |

---

### 示例 4：复杂 Markush（含多 R 基团与环连接）

**结构：** 杂环 Markush，含 R²、R³、R⁴、R⁵、X、Z 及一个环连接。

```
**C1*C(*)=C(C(*)(*)C2=CC=NC=C2)N=1<sep><a>0:R[4]</a><a>1:X</a><r>1:R[5]?n</r><a>3:Z</a><a>5:R[3]</a><a>8:R[2]</a><a>9:R[1]</a>
```

| 标签 | 原子/环索引 | 基团 | 含义 |
|------|------------|------|------|
| `<a>0:R[4]</a>` | 索引 0 | R⁴ | 第一个 `*` 原子为 R⁴ |
| `<a>1:X</a>` | 索引 1 | X | 第二个 `*` 原子为卤素 X |
| `<r>1:R[5]?n</r>` | 环 1 | R⁵ 带 `?n` | R⁵ 连接在环 1 上，带重复标记 |
| `<a>3:Z</a>` | 索引 3 | Z | 第四个 `*` 原子为 Z |
| `<a>5:R[3]</a>` | 索引 5 | R³ | 第六个 `*` 原子为 R³ |
| `<a>8:R[2]</a>` | 索引 8 | R² | 第九个 `*` 原子为 R² |
| `<a>9:R[1]</a>` | 索引 9 | R¹ | 第十个 `*` 原子为 R¹ |

---

### 示例 5：含抽象环与多 R 基团

**结构：** 抽象环 B 上连接多个 R 基团。

```
********<sep><a>0:R[1]</a><a>4:R[3]</a><a>5:R[2]</a><a>7:R[5]</a><a>8:R[4]</a><c>0:B</c><a>13:R[7]</a><a>14:R[6]</a>
```

| 标签 | 原子/环索引 | 基团 | 说明 |
|------|------------|------|------|
| `<a>0:R[1]</a>` | 索引 0 | R¹ | |
| `<a>4:R[3]</a>` | 索引 4 | R³ | |
| `<a>5:R[2]</a>` | 索引 5 | R² | |
| `<a>7:R[5]</a>` | 索引 7 | R⁵ | |
| `<a>8:R[4]</a>` | 索引 8 | R⁴ | |
| `<c>0:B</c>` | 环索引 0 | B | 抽象环 B |
| `<a>13:R[7]</a>` | 索引 13 | R⁷ | |
| `<a>14:R[6]</a>` | 索引 14 | R⁶ | |

---

## 六、书写 Checklist | 6. Writing Checklist

### 步骤 1：构建标准 SMILES

为分子骨架写出标准 RDKit 兼容 SMILES。

### 步骤 2：确定 Root 原子（非 Markush 分子）

```python
smiles = Chem.MolToSmiles(mol, rootedAtAtom=0)
```

### 步骤 3：插入虚拟原子 `*`

将所有 Markush 位点、连接点、抽象环连接点替换为 `*`。

### 步骤 4：分配索引

从 0 开始，为每个 `*` 和每个环计数。

### 步骤 5：编写 EXTENSION

| 特征 | 标签语法 |
|------|---------|
| 原子位点（R 基团） | `<a>ATOM_INDEX:R[n]</a>` |
| 原子位点（缩写基团） | `<a>ATOM_INDEX:Abbrev</a>` |
| 连接点 | `<a>ATOM_INDEX:<dum></a>` |
| 环不确定连接 | `<r>RING_INDEX:GROUP</r>` |
| 抽象环 | `<c>CIRCLE_INDEX:NAME</c>` |

### 步骤 6：组装

```
SMILES + "<sep>" + EXTENSION
```

EXTENSION 为空时，**省略** `<sep>`。

### 步骤 7：RDKit 验证

**仅将 SMILES 部分传入 RDKit：**

```python
smiles_part = esmiles.split("<sep>")[0]
mol = Chem.MolFromSmiles(smiles_part)
assert mol is not None, "SMILES 部分必须能被 RDKit 解析"
```

---

## 七、不支持的结构 | 7. Unsupported Structures

以下结构**目前无法**用 E-SMILES 规范表达（论文图 11）：

| 不支持类型 | 说明 |
|-----------|------|
| 虚线抽象环（Dashed abstract ring） | 用虚线表示的抽象环（非命名圆圈） |
| 配位键（Coordination bond） | 如金属配合物中的配位键 |
| 特殊符号 Markush | 非标准符号（如彩色圆点、特殊几何图形）表示的 Markush |
| 长结构段重复 | 骨架上重复的是长结构片段而非单个原子/基团（如聚合物链段重复） |

---

## 八、RDKit 交互要点 | 8. RDKit Interaction Guide

### 8.1 输入 RDKit

仅使用 `<sep>` **之前**的 SMILES 部分：

```python
esmiles = "*c1ccccc1<sep><a>0:R[1]</a>"
smiles_part = esmiles.split("<sep>")[0]  # "*c1ccccc1"
mol = Chem.MolFromSmiles(smiles_part, sanitize=False)
```

### 8.2 从 RDKit 输出

```python
# 对于嵌入 E-SMILES 的非 Markush 分子：
smiles = Chem.MolToSmiles(mol, rootedAtAtom=0)
```

### 8.3 EXTENSION 解析

`<sep>` 部分**不由 RDKit 处理**，需单独解析：

```python
import re

def parse_esmiles_extension(esmiles: str) -> dict:
    """将 E-SMILES 扩展标签解析为结构化字典。"""
    if "<sep>" not in esmiles:
        return {"smiles": esmiles, "extension": None}

    smiles_part, ext_part = esmiles.split("<sep>", 1)
    result = {"smiles": smiles_part, "atoms": [], "rings": [], "circles": []}

    atom_tags = re.findall(r"<a>(\d+):([^<]+)</a>", ext_part)
    for idx, group in atom_tags:
        result["atoms"].append({"index": int(idx), "group": group})

    ring_tags = re.findall(r"<r>(\d+):([^<]+)</r>", ext_part)
    for idx, group in ring_tags:
        result["rings"].append({"index": int(idx), "group": group})

    circle_tags = re.findall(r"<c>(\d+):([^<]+)</c>", ext_part)
    for idx, name in circle_tags:
        result["circles"].append({"index": int(idx), "name": name})

    return result
```

### 8.4 往返注意事项

| 方向 | 方法 | 说明 |
|------|------|------|
| E-SMILES → RDKit | `esmiles.split("<sep>")[0]` → `MolFromSmiles` | 仅 SMILES 部分 |
| RDKit → E-SMILES | `MolToSmiles(mol, rootedAtAtom=0)` + EXTENSION | SMILES 部分为标准形式 |
| E-SMILES → 图像 | 仅渲染 SMILES 部分 | EXTENSION 信息在图像中丢失 |
| E-SMILES → 搜索 | 使用 SMILES 部分做 Tanimoto 等 | EXTENSION 为结构元数据 |

---

## 九、语法速查 | 9. Quick Reference

### 语法摘要

```
E-SMILES  :=  SMILES [ '<sep>' EXTENSION ]
SMILES    :=  <valid RDKit SMILES string>
EXTENSION :=  TAG [ TAG ... ]
TAG       :=  ATOM_TAG | RING_TAG | CIRCLE_TAG
ATOM_TAG  :=  '<a>' INDEX ':' GROUP '</a>'
RING_TAG :=  '<r>' INDEX ':' GROUP '</r>'
CIRCLE_TAG:=  '<c>' INDEX ':' NAME '</c>'
INDEX     :=  <非负整数，从 0 开始>
GROUP     :=  R[n] | X | Y | Z | Abbrev | '<dum>'
NAME      :=  B | Ar | <其他命名环>
```

### 关键规则

| 规则 | 说明 |
|------|------|
| **分隔符** | `<sep>`（一个 token，非 `<<sep>`） |
| **索引基** | 所有索引均为 **0 基**（从 0 开始） |
| **标签顺序** | EXTENSION 内各标签顺序任意 |
| **重复索引** | 允许，多个标签可引用同一索引 |
| **RDKit 验证** | SMILES 部分必须能独立被 RDKit 解析 |

---

## 十、与 SMARTS 的关系 | 10. Relationship to SMARTS

E-SMILES 是**表示格式**（用于存储/传输结构），而非模式匹配格式（如 SMARTS）。

| 格式 | 用途 | RDKit 解析对象 |
|------|------|---------------|
| SMILES | 描述特定分子 | 完整字符串 |
| SMARTS | 描述分子模式 | 完整字符串 |
| E-SMILES | 描述 Markush/泛化结构 | 仅 SMILES 部分 |

EXTENSION 部分编码的是 SMILES 和 SMARTS 均无法表达的**结构元数据**。

---

*文档版本 1.1 — 修正版，基于 arXiv:2411.11098v4*
