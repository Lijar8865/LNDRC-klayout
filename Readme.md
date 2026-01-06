# KLayout DRC 使用说明文档

## 目录
- [简介](#简介)
- [DRC基本概念](#drc基本概念)
- [文件结构](#文件结构)
- [配置说明](#配置说明)
- [图层定义](#图层定义)
- [设计规则检查](#设计规则检查)
- [密度检查](#密度检查)
- [使用方法](#使用方法)
- [参考资源](#参考资源)

---

## 简介

本文档基于KLayout官方文档，详细说明光子集成电路(Photonic IC)设计中的DRC (Design Rule Check，设计规则检查)脚本使用方法。

DRC是集成电路设计中的关键验证步骤，用于确保版图设计符合工艺制造要求。KLayout提供了强大的DRC脚本引擎，支持复杂的几何规则检查。

### 主要功能
- ? 基础几何规则检查（宽度、间距）
- ? 高级几何规则检查（包围、角度、投影）
- ? 连接性检查
- ? 密度检查
- ? 自定义规则脚本
- ? 分块并行处理

---

## DRC基本概念

### 1. DRC引擎模式

KLayout DRC支持多种运行模式：

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| **Flat模式** | 将整个版图展平后检查 | 小规模设计 |
| **Deep模式** | 层次化检查，保留设计层次 | 大规模设计（推荐） |
| **Tiling模式** | 分块处理 | 超大规模设计 |

本脚本使用 `deep` 模式，适合复杂的层次化设计。

### 2. 核心DRC函数

#### 几何检查类

```ruby
# 宽度检查
layer.width(min_width, options)

# 间距检查  
layer.space(min_space, options)

# 包围检查
layer1.enclosing(layer2, min_enclosure)

# 面积检查
layer.area
```

#### 选项参数

| 选项 | 说明 | 示例 |
|------|------|------|
| `projection` | 投影模式测量 | `projection` |
| `euclidian` | 欧式距离测量 | `euclidian` |
| `angle_limit(n)` | 角度限制 | `angle_limit(10)` - 限制±10度 |
| `projection_limits(min,max)` | 投影长度限制 | `projection_limits(3.nm, 50.mm)` |

---

## 文件结构

DRC脚本采用XML格式封装，包含以下关键部分：

```xml
<?xml version="1.0" encoding="utf-8"?>
<klayout-macro>
  <category>drc</category>              <!-- 脚本类别 -->
  <interpreter>dsl</interpreter>        <!-- 解释器类型 -->
  <dsl-interpreter-name>drc-dsl-xml</dsl-interpreter-name>
  <text>
    <!-- DRC脚本主体 -->
  </text>
</klayout-macro>
```

---

## 配置说明

### 1. 性能配置

```ruby
# 分块处理配置
tiles(500.um)          # 每个分块大小为 500×500 μm
tile_borders(91.um)    # 分块边界重叠区域 91 μm（避免边界误报）

# 并行处理
threads(14)            # 使用14个线程并行处理
```

> [!TIP]
> **性能优化建议**
> - `tiles`: 根据设计大小调整，通常500-1000μm
> - `tile_borders`: 应大于最大设计规则值
> - `threads`: 设置为CPU核心数或稍小

### 2. 输入输出配置

```ruby
# 输入文件配置
inputFile = $input      # 从命令行或UI获取输入文件
inputCell = $cell       # 指定检查的单元格

if inputCell
  source(inputFile, inputCell)  # 指定单元格
else
  source(inputFile)             # 检查顶层单元格
end

# 报告配置
report("HK_LN", reportName)     # 生成DRC报告
```

### 3. 基本参数设置

```ruby
# 容差设置
tolerance = 0.001              # 测量容差 1nm
angle_tolerance = 0.1          # 角度容差 0.1度

# 投影参数
min_proj = 3.nm               # 最小投影长度
max_proj = 50.mm              # 最大投影长度

# 角度参数
normal_Angle = 10             # 常规角度限制 ±10度
slab_enc_rib_Angle = 0.1      # 包围角度容差
grat_enc_rib_Angle = 30       # 光栅角度限制
```

---

## 图层定义

### 光子器件图层

```ruby
# 光波导图层
SLABWG = input(3, 0)    # 平板波导层 (GDS层 3:0)
STRIPWG = input(2, 0)   # 条形波导层 (GDS层 2:0)

# 金属图层
M1 = input(1, 0)        # 金属层1 (GDS层 1:0)
M2 = input(5, 0)        # 金属层2 (GDS层 5:0)

# 功能图层
HT = input(10, 0)       # 加热器层 (GDS层 10:0)
```

> [!NOTE]
> 图层编号格式为 `layer:datatype`，例如 `(3, 0)` 表示GDS中的layer 3, datatype 0

### 设计规则参数

| 图层 | 最小宽度 | 最小间距 |
|------|----------|----------|
| **SLABWG** (平板波导) | 0.4 μm | 0.4 μm |
| **STRIPWG** (条形波导) | 0.4 μm | 0.4 μm |
| **M1/M2** (金属) | 1.0 μm | 0.2 μm |
| **HT** (加热器) | 1.0 μm | 1.0 μm |

---

## 设计规则检查

### 1. 平板波导 (SLABWG) 规则

```ruby
# Slab.Wid.01: 宽度检查
SLABWG.width(slab_w - tolerance, projection, 
             angle_limit(normal_Angle), 
             projection_limits(min_proj, max_proj))
      .polygons(0)
      .output("Slab.Wid.01", "Slab_width below 0.4um")
```

**规则说明：**
- 检查所有平板波导的宽度 ≥ 0.4μm
- 使用投影模式测量
- 角度限制：±10度
- `.polygons(0)` 返回原始违规多边形（不是扩展后的）

```ruby
# Slab.Spac.01: 间距检查
SLABWG.space(slab_space - tolerance, euclidian, 
             angle_limit(normal_Angle), 
             projection_limits(min_proj, max_proj))
      .polygons(0)
      .output("Slab.Spac.01", "Slab_space below 0.4um\n Or check the misalignment!")
```

**规则说明：**
- 检查平板波导间距 ≥ 0.4μm
- 使用欧式距离测量
- 该检查也可发现对齐错误

### 2. 条形波导 (STRIPWG) 规则

#### 基础几何规则

```ruby
# Rib.Wid.01: 宽度检查
STRIPWG.width(rib_w - tolerance, projection, 
              angle_limit(normal_Angle), 
              projection_limits(min_proj, max_proj))
       .polygons(0)
       .output("Rib.Wid.01", "Rib_width below 0.4um")

# Rib.Spac.01: 间距检查  
STRIPWG.space(rib_space - tolerance, euclidian, 
              angle_limit(normal_Angle), 
              projection_limits(min_proj, max_proj))
       .polygons(0)
       .output("Rib.Spac.01", "Rib_space below 0.4um\n Or check the misalignment!")
```

#### 连接性检查

> [!IMPORTANT]
> 连接性检查用于检测波导连接处的对齐问题，这是光子器件设计中的关键检查

```ruby
# Rib.Connect.01: 检查垂直边缘（可能的对齐错误）
STRIPWG_connect = STRIPWG.with_angle(270.degree - angle_tolerance, 
                                      270.degree + angle_tolerance)
STRIPWG_connect.output("Rib.Connect.01", "Please Check the misalignment!")
```

**检查逻辑：**
- 查找270度（垂直）方向的边缘
- 容差范围：270° ± 0.1°
- 正常水平波导不应有垂直边，若有则可能是连接错位

```ruby
# Rib.Connect.02: 检查连接间隙
STRIPWG_connect2 = STRIPWG.merged.edges
                         .with_length(1.3.um - tolerance, 1.3.um + tolerance)
                         .space(13.um, angle_limit(1))
STRIPWG_connect2.output("Rib.Connect.02", "Exist Gap!\nPlease Check the connection!")
```

**检查逻辑：**
- 查找长度为1.3μm的边缘（特征连接段）
- 检查这些边缘间的间距是否为13μm
- 用于检测特定连接结构的间隙问题

#### 包围规则

```ruby
# Rib in Slab.01: 条形波导必须被平板波导包围
SLABWG.enclosing(STRIPWG, 0.2.um)
      .output("Rib in Slab.01", "Minimum inclusion of Rib in Slab: 0.4 um")
```

**规则说明：**
- 条形波导(STRIPWG)必须完全在平板波导(SLABWG)内部
- 最小包围距离：0.2μm（实际建议0.4μm）

### 3. 金属层 (M1/M2) 规则

```ruby
# M1.Wid.01: M1宽度检查
M1.width(metal_w - tolerance, projection, 
         angle_limit(normal_Angle), 
         projection_limits(min_proj, max_proj))
  .polygons(0)
  .output("M1.Wid.01", "M1_width should greater 1um")

# M1.Spac.01: M1间距检查
M1.space(metal_space - tolerance, euclidian, 
         angle_limit(normal_Angle), 
         projection_limits(min_proj, max_proj))
  .polygons(0)
  .output("M1.Spac.01", "M1_space should greater 0.2um")
```

**M2规则类似：**
- M2最小宽度：1.0μm
- M2最小间距：0.2μm

### 4. 加热器 (HT) 规则

```ruby
# HT.Wid.01: 加热器宽度检查
HT.width(HT_w - tolerance, projection, 
         angle_limit(normal_Angle), 
         projection_limits(min_proj, max_proj))
  .polygons(0)
  .output("HT.Wid.01", "HT_width should greater 1um")

# HT.Spac.01: 加热器间距检查
HT.space(HT_s - tolerance, euclidian, 
         angle_limit(normal_Angle), 
         projection_limits(min_proj, max_proj))
  .polygons(0)
  .output("HT.Spac.01", "HT_space should greater 1um")
```

---

## 密度检查

密度检查用于确保各图层的面积占比符合工艺要求，避免CMP(化学机械平坦化)等工艺问题。

### 启用密度检查

```ruby
DENSITY = false    # 默认关闭，需要时设置为 true
```

### 密度规则

```ruby
# 获取芯片总面积
CHIP = extent.sized(0.0)

if DENSITY
  # SLABWG密度检查: < 10%
  if (SLABWG.area / CHIP.area >= 0.1)
    CHIP.output("SLABWG_density", 
                "The density of SLABWG should be less than 10%: #{(100 * SLABWG.area / CHIP.area).to_s}%")
  end
  
  # STRIPWG密度检查: < 20%
  if (STRIPWG.area / CHIP.area >= 0.2)
    CHIP.output("STRIPWG_density", 
                "The density of STRIPWG should be less than 20%: #{(100 * STRIPWG.area / CHIP.area).to_s}%")
  end
  
  # M1密度检查: 10% - 50%
  if ((M1.area / CHIP.area <= 0.1) || (M1.area / CHIP.area >= 0.5))
    CHIP.output("M1_density", 
                "The density of M1 should range from 10% to 50%: #{(100 * M1.area / CHIP.area).to_s}%")
  end
  
  # HT密度检查: < 50%
  if (HT.area / CHIP.area > 0.5)
    CHIP.output("HT_density", 
                "The density of HT should be less than 10%: #{(100 * HT.area / CHIP.area).to_s}%")
  end
  
  # M2密度检查: < 50%
  if (M2.area / CHIP.area > 0.5)
    CHIP.output("M2_density", 
                "The density of M2 should be less than 50%: #{(100 * M2.area / CHIP.area).to_s}%")
  end
end
```

### 密度要求汇总

| 图层 | 密度要求 | 说明 |
|------|----------|------|
| SLABWG | < 10% | 平板波导 |
| STRIPWG | < 20% | 条形波导 |
| M1 | 10% - 50% | 金属层1（有上下限） |
| HT | < 50% | 加热器 |
| M2 | < 50% | 金属层2 |

---

## 使用方法

### 方法1: KLayout GUI运行

1. **打开KLayout**
   - 加载GDS文件：`File → Open`

2. **运行DRC脚本**
   - 菜单：`Tools → DRC → [脚本名称]`
   - 或使用宏编辑器：`Macros → Macro Development`

3. **查看结果**
   - DRC结果会显示在Marker Browser中
   - 双击错误项可定位到版图位置

### 方法2: 命令行运行

```bash
# 基本命令
klayout -b -r LN_HK_DRC2.lydrc -rd input=layout.gds

# 指定单元格
klayout -b -r LN_HK_DRC2.lydrc -rd input=layout.gds -rd cell=TOP_CELL

# 多线程运行
klayout -b -r LN_HK_DRC2.lydrc -rd input=layout.gds -rd threads=16
```

**参数说明：**
- `-b`: 批处理模式（无GUI）
- `-r`: 指定DRC脚本
- `-rd`: 传递变量（`input`, `cell`, `threads`等）

### 方法3: Python API调用

```python
import klayout.db as db
import klayout.rdb as rdb

# 加载版图
layout = db.Layout()
layout.read("layout.gds")

# 运行DRC
report = rdb.ReportDatabase()
# ... 执行DRC逻辑 ...
report.save("drc_report.lyrdb")
```

---

## DRC报告解读

### 报告文件

DRC检查完成后，会生成 `[文件名]_HK_result.lyrdb` 报告文件。

### 错误类别示例

| 规则ID | 描述 | 严重程度 |
|--------|------|----------|
| `Slab.Wid.01` | 平板波导宽度不足 | ? Critical |
| `Rib.Spac.01` | 条形波导间距不足 | ? Critical |
| `Rib.Connect.01` | 连接对齐错误 | ? Warning |
| `M1.Wid.01` | 金属宽度不足 | ? Critical |
| `SLABWG_density` | 平板波导密度超标 | ? Important |

### 错误定位

1. 在KLayout中打开DRC报告：`File → Load Report Database`
2. 在Marker Browser中浏览错误
3. 双击错误项跳转到版图位置
4. 查看错误详细信息

---

## 高级技巧

### 1. 自定义规则

```ruby
# 示例：检查特定角度的边缘
custom_edges = STRIPWG.edges.with_angle(45.degree - 1, 45.degree + 1)
custom_edges.output("Custom.Angle.01", "Found 45-degree edges")
```

### 2. 布尔运算

```ruby
# 交集
overlap = STRIPWG & SLABWG

# 差集
outside = STRIPWG - SLABWG

# 并集
combined = M1 | M2
```

### 3. 调试技巧

```ruby
# 输出图层信息
print "STRIPWG count: " + STRIPWG.count.to_s + "\n"
print "STRIPWG area: " + STRIPWG.area.to_s + "\n"

# 输出中间结果
temp_layer = STRIPWG.sized(0.1.um)
temp_layer.output("Debug.Sized", "Debug intermediate result")
```

### 4. 性能优化

> [!TIP]
> **优化建议**
> - 使用 `deep` 模式处理大型层次化设计
> - 合理设置 `tiles` 和 `tile_borders`
> - 避免不必要的 `.merged` 操作
> - 使用 `polygons(0)` 而非默认的扩展多边形

---

## 常见问题

### Q1: DRC运行很慢怎么办？

**A:** 尝试以下优化：
```ruby
# 增加分块大小
tiles(1000.um)

# 增加线程数
threads(24)

# 确保使用deep模式
deep
```

### Q2: 如何只检查特定规则？

**A:** 注释掉不需要的规则：
```ruby
# SLABWG.width(...).output(...)  # 注释此行
STRIPWG.width(...).output(...)   # 仅运行此规则
```

### Q3: 如何处理误报？

**A:** 调整容差或角度限制：
```ruby
tolerance = 0.005           # 增加容差
angle_tolerance = 0.5       # 放宽角度限制
normal_Angle = 15           # 扩大角度范围
```

### Q4: 密度检查失败怎么办？

**A:** 
1. 检查是否需要添加填充图案(fill patterns)
2. 调整设计减少/增加金属覆盖
3. 如不需要密度检查，设置 `DENSITY = false`

---

## 参考资源

### KLayout官方文档

- **DRC基础教程**: [https://www.klayout.de/doc/about/drc_basics.html](https://www.klayout.de/doc/about/drc_basics.html)
- **DRC参考手册**: [https://www.klayout.de/doc/about/drc_ref.html](https://www.klayout.de/doc/about/drc_ref.html)
- **DRC Runset编写指南**: [https://www.klayout.de/doc/manual/drc_runsets.html](https://www.klayout.de/doc/manual/drc_runsets.html)

### 脚本API参考

- **Ruby API (RBA)**: [https://www.klayout.de/doc/code/index.html](https://www.klayout.de/doc/code/index.html)
- **Python API (pya)**: [https://www.klayout.org/doc-qt5/code/index.html](https://www.klayout.org/doc-qt5/code/index.html)

### 社区资源

- **KLayout论坛**: [https://www.klayout.de/forum/](https://www.klayout.de/forum/)
- **GitHub**: [https://github.com/KLayout/klayout](https://github.com/KLayout/klayout)

---

## 版本信息

- **文档版本**: 1.0
- **KLayout版本**: 0.27+
- **适用工艺**: 光子集成电路 (Photonic IC)
- **脚本语言**: Ruby (DSL)

---

## 许可与支持

本文档基于KLayout官方文档编写，遵循相同的开源协议。

如有疑问，请参考KLayout官方文档或社区支持。

---

*最后更新：2026年1月*
