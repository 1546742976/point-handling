# 丝绸之路轨迹数据生成器 — CLAUDE.md

## 项目概述

使用自然三次样条插值（C² 连续）生成等弧长采样的丝绸之路路径轨迹数据。提供 Python CLI 和 Web 两个版本，算法完全一致。

## 核心算法

### 1. 自然三次样条拟合 (`fit_natural_cubic_spline`)
- 输入：t-knots（弦长累积参数化）、控制点 Y 坐标
- 输出：分段三次多项式系数 `(a, b, c, d)`
- 使用三对角矩阵求解（Thomas 算法），自然边界条件（端点二阶导 = 0）

### 2. 系数混合法 — 长度控制 (`blend_spline_coeffs` + `find_alpha_for_target_length`)
- **核心创新**：在自然样条与折线之间做系数凸组合
- `a_i' = α·y_i + (1-α)·a_i`, `b_i' = (1-α)·b_i`, 以此类推
- α=0 → 纯自然样条（最长），α=1 → 折线（最短 = 弦长总和）
- 弧长 L(α) 单调递减 → 二分搜索 α 匹配目标长度
- **轨迹始终经过所有控制点**（a_i = y_i 恒成立）

### 3. 等弧长重参数化 (`arc_length_reparam`)
- 在密集 t 网格（20000 点）上积分弧长 → 构建 s(t) 映射
- 线性插值反演 t(s) → 均匀弧长步长采样
- 输出 N 个等距采样点及其一阶/二阶导数

### 4. 后处理平滑 (`smooth_output_trajectory`)
- Laplacian 平滑：迭代移动每个点到邻域均值
- **控制点位置锁定**：平滑后强制将最近输出点对齐回每个控制点
- 平滑前也做控制点对齐（弥补离散采样误差）

### 5. 有符号曲率半径 (`compute_signed_radius`)
- R = (x'² + y'²)^(3/2) / (x'y'' - y'x'')
- 符号由叉积方向决定（左转/右转）
- NaN/Inf 处理，异常值裁剪到 ±1e8

## 项目结构

```
generate_trajectory.py   # Python CLI 版（~1200 行），核心算法 + openpyxl 导出 + matplotlib 可视化
index.html               # Web 版（~1900 行），单文件 HTML，Canvas 可视化，相同算法 JS 实现
control_points.txt        # 默认控制点：31 个丝绸之路城市坐标，6000×2000 地图
output.txt / output.xlsx  # 默认输出文件
```

## Python CLI 用法

```bash
python generate_trajectory.py [options] <输入文件> [输出文件]

选项:
  --plot               生成 PNG 轨迹地图可视化
  --xlsx               导出 Excel (.xlsx)，含"轨迹数据"和"生成参数"两个 sheet
  --smooth S           控制点平滑强度 [0, 1]，默认 0
  --n-output N         等弧长采样点数，默认 2000
  --target-length L    期望轨迹总长度（系数混合法，始终经过控制点）
  -h, --help           显示帮助
```

## 输入文件格式

```
地图宽度 地图高度    # 第 1 个有效数据行
控制点数量 N          # 第 2 个有效数据行
X1 Y1                 # N 行控制点坐标
...
XN YN
```
- 空行和 `#` 开头的行为注释
- 分隔符支持空格/逗号/制表符

## 输出格式

- **文本输出**：N 行，制表符分隔，`X\tY\tR`，两位小数，无表头
- **Excel 输出**：两个 sheet
  - "轨迹数据"：2000 行 × 4 列（序号, X, Y, 曲率半径 R）
  - "生成参数"：参数键值对

## 关键参数默认值

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `n_output` | 2000 | 等弧长采样点数 |
| `n_dense` | 20000 | 弧长积分的密集采样点数 |
| `smooth_strength` | 0.0 | 不平滑 |
| `smooth_n_iter` | 5 | 平滑迭代次数 |
| α 二分精度 | 1e-8 | `find_alpha_for_target_length` 收敛阈值 |
| α 二分区间 | [0, 1] | 自然样条(0) ↔ 折线(1) |

## 已知注意事项

- Python 依赖：`numpy`, `matplotlib`（--plot 时）, `openpyxl`（--xlsx 时）
- Web 版的 canvas 控件绘制在 6000×2000 坐标空间，需等比例缩放显示
- `output_xlsx()` 的 `ctrl_pts` 参数当前未使用（仅保留接口兼容性）
- 控制点对齐步骤会修改输出点中离每个控制点最近的那个点，确保轨迹精确经过所有控制点
- 当 `target_length < chord_sum`（弦长总和）时，无法达到目标长度，α → 1（折线）

## 代码风格

- Python：numpy 风格，中文注释，节间分隔线 `# ═══`
- JS：函数式风格，async/await 生成流程，Canvas 2D 渲染
- 两个版本函数命名一一对应（snake_case in Python, camelCase in JS）
