# Yield Map Generator

Sensor 良率统计与缺陷 Map 图生成工具，支持 TR / TW 两类产品，按 WaferID 分别统计良率并生成可视化缺陷分布图。

## 环境依赖

- Python 3.10+

- polars >= 1.0

- matplotlib >= 3.7

- openpyxl >= 3.0

- tkinter (Python 标准库)

## 启动

```bash
python yieldmap_gui.py
```

### 必需资源文件

程序运行需要 `TR&TW AddrID Vs ChipID.xlsx`（坐标 → ChipID 映射表），该文件**未随仓库分发**，请自行获取后放到与 `yieldmap_gui.py`（或打包后的 `YieldMap.exe`）同级目录。缺失时 ChipID 解析失败，缺陷 Map 将无法正常显示。

打包运行时，程序以 `YieldMap.exe` 所在目录作为资源根目录查找 `TR&TW AddrID Vs ChipID.xlsx` 与 `yieldmap_config.json`。

## 数据源

| 类型 | 根目录                                    |
| -- | -------------------------------------- |
| TR | `\\mdtprod02\MDTWPDC\ZJG_Data\UF190B\Sensor\TR02V02\TR02V02_DW-J4`    |
| TW | `\\mdtprod02\MDTWPDC\ZJG_Data\UF190B\Sensor\TW02V02_DW` |

根目录下的子目录名即为 LotNo（如 `25B086`、`265023`）。程序只读取 Lot 目录下 `MPAD` 子目录中的 CSV 文件（其他目录如 `STML_ELG`、`MAP` 等被忽略）。

### TW 文件格式

TW 支持两种 EPS 格式，由 GUI 的 **TW Format** 单选切换：

| 格式           | 文件名特征                  | 说明                                                          |
| ------------ | ---------------------- | ----------------------------------------------------------- |
| EPSH   | `Sensor-TW-1S-EPSH-V1` | HD01\~HD32 电阻值直接在 EPSH 文件中                              |
| EPS-V3  | `Sensor-TW-1S-EPS-V3`  | EPS-V3 文件无 HD01\~HD32 电阻列，电阻值需从 HD1-LR/HD2-LR 文件的 R 列读取 |

TR 仅使用 EPSH 格式，且只采用 `RMIN` 文件（`RMAX` 被忽略），避免同一 Wafer 两次测量重复计数。

### 重复测量处理

同一 WaferID 同类测试可能存在多个文件（重复测量或目录副本），程序通过 **Duplicate File** 单选选择保留策略，避免重复计数：

| 模式            | 说明                                                                       |
| ------------- | ------------------------------------------------------------------------ |
| `By Size`（默认） | 取文件字节大小最大的一份（测量数据最完整）                                                    |
| `By Newest`   | 取最新版本：优先比较文件名中嵌入的测量时间戳（如 `..._2026-05-31 14-07-28.csv`），无法解析时回退到文件系统修改时间 |

该选项对 EPS 文件与 HD1-LR / HD2-LR 文件同时生效，配置可通过 **Save Config / Load Config** 持久化。

## 使用方法

1. 选择 Sensor Type（TR / TW）
2. 若选择 TW，再选择 **TW Format**：`EPSH` 或 `EPS-V3`
3. 输入或选择 LotNo
4. 点击 **Generate** 生成良率统计与缺陷 Map
5. 通过 **Wafer** 下拉框切换查看各晶圆或全部（ALL）
6. 点击 **Export Excel** 导出结果：选择 `ALL` 时导出每个 WaferID 一个独立 Excel 到指定目录；选择具体 Wafer 时导出单个文件

## 缺陷判定逻辑

每片 Wafer 独立判定，5 类缺陷按优先级合并（IND > DAT > SEV > PTC > ELG）：

| 缺陷类型       | 判定条件                                                                        | 缺陷码 |
| ---------- | --------------------------------------------------------------------------- | --- |
| Inductance | HD1/HD2 的 L 电感值超出上下限（仅 TW）                                                  | IND |
| Data       | HD01\~HD32 任一通道超出上下限（EPSH 格式从 EPSH 文件读取；EPS-V3 格式从 HD1-LR/HD2-LR 文件的 R 列读取） | DAT |
| Servo      | S1 或 S2 超出上下限                                                               | SEV |
| PTC        | HD01\~HD32\_PTC 任一 < PTC下限(默认1,500,000)                                     | PTC |
| ELG        | 中心6通道（RB3,RA2,RA3,RA4,RA5,RB4）中 >2 个超出各自上下限                              | ELG |

### IND 电感缺陷判定（仅 TW）

TW 晶圆额外读取 `*HD1-LR*` 和 `*HD2-LR*` CSV 文件，提取每个 die 位置的 32 个 L 值，按 die 位置 (SHOT\_ROW, SHOT\_COL, WIS\_ROW) 聚合后 pivot 为 L01\~L32 宽表。

HD1-LR / HD2-LR 文件的 `Column` 即通道位置（HD0n 对应 `Column = n + 1`）：

| 文件   | Column 取值      | 对应通道                    |
| ---- | -------------- | ----------------------- |
| HD1  | 2, 4, ..., 32  | 奇数通道 L01, L03, ..., L31 |
| HD2  | 3, 5, ..., 33  | 偶数通道 L02, L04, ..., L32 |

GUI **IND Mode** 单选提供三种判定模式（仅 TW 有效，TR 不读取 L 数据）：

| 模式               | 判定逻辑                                                                                                    |
| ---------------- | ------------------------------------------------------------------------------------------------------- |
| `3Sigma`（默认）     | WithinWafer 3Sigma：对整片 Wafer 的全部 L 值计算 COP 上下限（mean ± 3σ，中心 80% 数据），die 的任一 L 超出 \[L\_LL, L\_UL] 即判 IND |
| `WithinChip CV%` | WithinChip CV%：对每个 die 计算 CV% = std(L01..L32) / mean(L01..L32) × 100%，CV% > 设定阈值即判 IND                  |
| `3Sigma+CV%`     | 同时采用 WithinWafer 3Sigma 和 WithinChip CV%，任一条件 NG 即判 IND（并集）                                             |

CV% 阈值通过 **CV% UL** 输入框设置（默认 3.0），仅在 `WithinChip CV%` / `3Sigma+CV%` 模式启用。

L 上下限（3Sigma 模式用）默认按 COP 自动计算，也可在 **Use Custom Limits** 中手动设置（L LL / L UL）。IND 优先级最高。

### Data 电阻缺陷（TW EPS-V3 格式）

对于 TW 新 `EPS-V3` 格式，EPS-V3 文件本身不包含 `HD01~HD32` 电阻列（仅含 `S1_R`/`S2_R`/`HD{i}_PTC_R`/`RA{i}`/`RB{i}`）。程序改从 HD1-LR / HD2-LR 文件的 `R` 列读取电阻值：

- HD1-LR 文件（`Column` = 2,4,...,32）→ 奇数通道 HD01, HD03, ..., HD31

- HD2-LR 文件（`Column` = 3,5,...,33）→ 偶数通道 HD02, HD04, ..., HD32

即通道号 `n = Column - 1`，两文件共同覆盖 HD01\~HD32。

按 die 位置 (SHOT\_ROW, SHOT\_COL, WIS\_ROW) 聚合，任一通道电阻超出 COP 上下限即判定 DAT 缺陷。

旧 `EPSH` 格式仍直接从 EPSH 文件的 `HD{i}_R` 列读取电阻。

### Data 判定模式

GUI **Data Mode** 单选提供四种判定模式，对每个 die 的 32 个 Data 通道电阻值（HD01\~HD32）做 NG 判定：

| 模式               | 判定逻辑                                                                                                                                 |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `3Sigma`（默认）     | WithinWafer 3Sigma：对整片 Wafer 全部 Data 值计算 COP 上下限（mean ± 3σ，中心 70% 数据），die 的任一通道超出 \[LL, UL] 即判 DAT                                   |
| `WithinChip CV%` | WithinChip CV%：对每个 die 计算 CV% = std(HD01..HD32) / mean(HD01..HD32) × 100%，CV% > 设定阈值即判 DAT                                           |
| `HF_Sigma`       | WithinChip 移动极差标准差：对每个 die 计算 HF\_Sig = std(\|HD\_{i+1}-HD\_i\|)（31 个移动极差的样本标准差），HF\_Sig > 设定 UL 即判 DAT；任一通道为 null 的 die 跳过判定（视为 OK） |
| `3Sigma+CV%`     | 同时采用 WithinWafer 3Sigma 和 WithinChip CV%，任一条件 NG 即判 DAT（并集）                                                                          |

- CV% 阈值通过 **CV% UL** 输入框设置（默认 6.0），仅在 `WithinChip CV%` / `3Sigma+CV%` 模式启用

- HF\_Sig UL 通过 **HF\_Sig UL** 输入框设置（默认 0.5），仅在 `HF_Sigma` 模式启用

各 wafer 实际使用的上下限在 Excel 输出的 **Limits** Sheet 中记录：

- Data LL / Data UL：始终按 COP（中心 80% 数据，mean ± 3σ）计算并填入，无论模式如何，供参考

- Data HF\_Sig UL：在 `HF_Sigma` 模式填入设定值，否则显示 `-`

- Data CV% UL：在 `WithinChip CV%` / `3Sigma+CV%` 模式填入设定值，否则显示 `-`

HF\_Sig 的具体定义和按晶圆分布可视化详见下文 [HF\_Sig Sheet](#hf_sig-sheet) 章节。

### 上下限计算（COP方式）

默认按数据分布自动计算：

1. 取中心 80% 分位数范围 \[P10, P90] 的数据
2. 计算均值 mean 与标准差 sigma
3. LL = mean - 3*sigma, UL = mean + 3*sigma

### ELG 通道定义

| 类型 | ELGA           | ELGB           | 总计   |
| -- | -------------- | -------------- | ---- |
| TR | RA1\~RA6 (6ch) | RB1\~RB6 (6ch) | 12ch |
| TW | RA1\~RA5 (5ch) | RB1\~RB7 (7ch) | 12ch |

ELG 判定仅使用中心 6 个通道：**RB3, RA2, RA3, RA4, RA5, RB4**。其中超过 2 个通道超出规格即判定 ELG 缺陷。RA 通道使用 ELGA 上下限，RB 通道使用 ELGB 上下限。ELGA/ELGB 上下限仍按全部通道 COP 计算。

## 自定义 Limits

勾选 **Use Custom Limits** 后可手动设置各项上下限：

- L LL / L UL（仅 TW 有效）

- Data LL / UL

- Servo LL / UL

- ELGA LL / UL

- ELGB LL / UL

- PTC LL

参数可通过 **Save Config / Load Config** 持久化到 `yieldmap_config.json`。

## Map 图坐标

DefectMap 按 ChipID 解析的 ChipRow/ChipCol 位置绘制：

- Y 轴 = ChipRow（1-176）

- X 轴 = ChipCol（1-8）

## 缺陷颜色

| 状态  | 颜色           |
| --- | ------------ |
| OK  | 绿色 `#4CAF50` |
| IND | 深红 `#FF1744` |
| DAT | 红色 `#F44336` |
| SEV | 橙色 `#FF9800` |
| PTC | 紫色 `#9C27B0` |
| ELG | 蓝色 `#2196F3` |

## Excel 输出结构

**按 WaferID 单独输出独立 Excel 文件**：Export Excel 时，若 Wafer 选择 `ALL`，则选择输出目录，每个 WaferID 生成一个独立的 `.xlsx` 文件（文件名 `LotNo_Type_WaferID_YieldMap_时间戳.xlsx`）；若选择具体 Wafer，则导出单个文件。

每个 Excel 文件包含以下 sheet（针对该 Wafer）：

| Sheet         | 内容                                                                                                              |
| ------------- | --------------------------------------------------------------------------------------------------------------- |
| Yield Summary | Lot 信息、该 Wafer 良率明细、合计                                                                                          |
| Limits        | 该 Wafer 的上下限参数（含 L LL/UL、Data LL/UL、Data HF\_Sig UL 等）                                                          |
| Map           | 该 Wafer 的缺陷 Map 图（按 ChipRow/ChipCol 位置排列，颜色标记）                                                                  |
| HF\_Sig       | 该 Wafer 的 HF\_Sig（移动极差标准差）按 ChipRow/ChipCol 分布的热力图，颜色按本 Wafer 内 min/max 归一化为绿→黄→红梯度                             |
| Detail Data   | 该 Wafer 的明细数据，以 WaferID + ChipID 为索引，含 IND/DATA/SERVO/PTC/ELG 各项判定及 FINL\_MAP                                   |
| Raw\_Data     | 该 Wafer 的原始测量数据，以 WaferID + ChipID 为索引，含 L01~~L32（仅 TW）、HD01~~HD32（Data）、S1/S2（Servo）、RA\*/RB\*（ELG），**不含 PTC** |

Map sheet 的列宽已加宽（18 字符）、行高已缩窄（5 磅），使 8 列 × \~176 行的晶圆图能正常显示。

### HF\_Sig Sheet

每片 Wafer 一个 HF\_Sig Sheet（多 Wafer 合并导出时为 `HF_Sig_{WaferID}`，单 Wafer 导出时为 `HF_Sig`），用于直观观察每片晶圆各 die 的移动极差稳定性。

- **定义**：对每个 die 取其 32 个 Data 通道（HD01\~HD32），计算 31 个移动极差 MR\_i = |HD\_{i+1} - HD\_i|，再取样本标准差 HF\_Sig = std(MR\_1..MR\_31)。仅当 die 的 32 个通道全部非 null 时计算，否则为空。

- **布局**：按 ChipRow × ChipCol 二维网格排列，与 Map Sheet 一致。ChipID 查找失败的 die 置灰。

- **着色**：HF\_Sig 值按本 Wafer 内 \[min, max] 归一化到 \[0, 1]，映射为绿(低)→黄→红(高)梯度；字体颜色依据亮度自适应黑/白以保证可读性。

- **范围提示**：标题旁标注本 Wafer 的 min/max 值便于参考。

该 Sheet 的 HF\_Sig 值与 Data 判定模式 `HF_Sigma` 使用的指标完全一致；切换到 `HF_Sigma` 模式并以 **HF\_Sig UL** 阈值即可基于此值判定 DAT 缺陷。

## ChipID 映射

参照 `TR&TW AddrID Vs ChipID.xlsx` 文件，将晶圆内每个 die 的坐标 (SHOT\_ROW, SHOT\_COL, WIS\_ROW, WIS\_COL) 转换为 ChipID（如 `027W04`、`141R05`）。

- ChipID 格式：`NNN[T]CC`，其中 NNN 为行号(1-176)，T 为类型(R/W)，CC 为列号(1-8)

- ChipRow/ChipCol 用于 DefectMap 的网格定位

- 边缘 die 无对应 ChipID（不在映射表中）

## 坐标系说明

CSV 原始列与内部变量的映射关系（参照 `yieldmapcombine.py`）：

| CSV 列名  | 内部变量      | 含义              |
| ------- | --------- | --------------- |
| Field-Y | SHOT\_ROW | Shot（场）行号       |
| Field-X | SHOT\_COL | Shot（场）列号       |
| Row     | WIS\_ROW  | Die 在 Shot 内的行号 |
| Column  | WIS\_COL  | Die 在 Shot 内的列号 |

HD1-LR/HD2-LR 文件中，`Column` 为 HD 通道位置（HD0n 对应 `Column = n + 1`；HD1: 2,4,...32 为奇数通道，HD2: 3,5,...33 为偶数通道），不参与 die 位置定位。

## 文件说明

| 文件                            | 说明                               | 随仓库分发 |
| ----------------------------- | -------------------------------- | ----- |
| `yieldmap_gui.py`             | 主程序（GUI + 数据处理 + 可视化 + Excel 导出） | 是     |
| `yieldmapcombine.py`          | 原始缺陷判定参考脚本                       | 否     |
| `TR&TW AddrID Vs ChipID.xlsx` | 坐标到 ChipID 的映射表（运行必需，需自行放置）       | 否     |
| `yieldmap_config.json`        | 用户配置（运行时自动生成）                    | 否     |
| `YieldMap.exe`                | PyInstaller 打包产物（自建）              | 否     |

