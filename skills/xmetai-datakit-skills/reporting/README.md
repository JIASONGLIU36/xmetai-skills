# XMetAI Reporting

把**已有的评测产物**变成图表与一份 Markdown 报告。不重算指标、不猜数据、
不靠目录名认能力。

能力契约、触发词与硬性规则的唯一权威是 [`SKILL.md`](./SKILL.md)；本文件只讲
"它是什么、怎么装、怎么用"。

## 能做什么

| 链路 | 输入 | 产出 |
|---|---|---|
| 确定性 / 集合降水 TS | `diagnostics/categorical_wide.csv`（集合另加 `probability_wide.csv`） | 最多 11 张 SVG + 六章 `REPORT.md` |
| 连续场 RMSE / ACC / 活跃度 / 谱 | `scores.csv` | PNG 图 + `REPORT.md` |
| RMSE 批次多模型对比 | N 份标准产物目录 | 多模型对比报告 |
| 台风路径 / 强度 | `typhoon/tc*.csv` + `*_meta.json` | 六章报告 + 统计 CSV/JSON + 逐台风路径图 |
| 暴雨过程个例 | `{编号}_24h/` 与 `{编号}_total/` 两套段 | 按过程分节的报告 |
| 空间分布 | 取数脚本产出的宽表 CSV | 3 行（确定性）/ 4 行（集合）SVG |

## 输入契约

统一入口只吃**标准产物目录**：

```text
<产物目录>/
  manifest.json                 # 必需；resolved_config.pipeline 是能力判据
  scores.csv                    # 连续场必需
  diagnostics/
    categorical_wide.csv        # TS 必需
    probability_wide.csv        # 集合可选
  typhoon/                      # 台风必需
    tc<编号>_<起报>.csv
    tc<编号>_<起报>_meta.json
```

必需列、缺列处理、`grade` 归一化与"不可直接比较"的判定，见
[`SKILL.md`](./SKILL.md) 的「输入：标准评测产物目录」。
**缺列会明确报错并列出缺哪几列，不会补默认值。**

## 安装

Python >= 3.10。在仓库根执行：

```bash
python -m pip install -r requirements.txt
# 需要空间取数（Zarr / NetCDF）或台风地图时：
python -m pip install "./skills/xmetai-datakit-skills/reporting[all]"
```

仓库根的 `requirements.txt` **只指向本目录的 `pyproject.toml`**
（`-e ./skills/xmetai-datakit-skills/reporting[test]`），依赖版本只在
`pyproject.toml` 里维护，不在 requirements 里重复一份。

**不安装也能直接跑**：入口脚本会把自身所在的 `scripts/` 插进 `sys.path`，
因此复制整个 `reporting/` 目录到任何位置都可运行。

## 使用

```bash
python scripts/render_report.py <产物目录> --result report/<名字> --model <模型名>
```

多模型对比（TS）：

```bash
python scripts/render_report.py outputs/results/weather_ts_single_fgvp \
  --result report/fgvp_vs_fuxi --model FGVP \
  --compare "FuXi=outputs/results/weather_ts_single_fuxi/diagnostics/categorical_wide.csv"
```

其他入口：

| 场景 | 命令 |
|---|---|
| 多模型 RMSE（单成员 / 集合） | `render_det_report.py` / `render_ens_report.py --model NAME=目录` |
| 暴雨过程个例 | `render_rainstorm_report.py --model NAME=目录 --process 202503,202521` |
| 挑暴雨过程 | `render_rainstorm_summary.py` → `summary_long.csv` |
| 空间分布取数（服务器） | `extract_spatial_distribution.py` / `extract_spatial_distribution_ens.py` |
| 空间分布出图 | `render_spatial_distribution.py --csv ... --model ...` |
| 台风专用统计 | `render_report.py` + `--leads` / `--fixed-leads` / `--bootstrap` |

## 目录结构

```text
reporting/
├── SKILL.md           # 触发词、输入契约、能力路由、硬性规则
├── README.md          # 本文件
├── pyproject.toml     # 独立安装与依赖声明
├── agents/            # openai.yaml（Codex 元数据）
├── assets/templates/  # 6 份报告格式契约
├── references/        # 9 篇口径与判读规则
└── scripts/
    ├── _bootstrap.py  _spatial_common.py
    ├── render_*.py  extract_*.py     # 9 个 CLI 入口
    └── reporting/                    # 运行库
        ├── dispatch.py               # 认能力 + 派发
        ├── core/                     # 样式 / 表格 / writer / 语义
        ├── sources/                  # 契约 / ERA5 / 布局 / 台风几何
        ├── spectra/                  # 指标基类 / 球谐带
        └── reports/                  # 各条产出链路
```

## 测试

在仓库根执行：

```bash
python -m pytest tests -q
```

`tests/test_reporting.py` 会把本 skill 复制到临时目录，验证它在脱离原仓库、
未安装的情况下仍能跑通全部入口；缺少科学计算依赖时自动跳过。

## 边界

- **不重算指标**：指标由评测流水线产生，本 skill 只消费产物。
- **不合成缺的表**：没有宽表就报错，不做长表 → 宽表转换。
- **不靠目录名认能力**：以 `manifest.json` 的 `resolved_config.pipeline` 为准。
- 输入只读；输出目录须不存在或为空。
