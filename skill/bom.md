# BOM 导入

实现：`backend/app/services/bom.py` 的 `parse_upload`。只接受 `.xls`、`.xlsx`。先读成字符串矩阵，合并单元格的值铺满整个区域，再识别格式。失败抛 `BomError`，接口变成 400，不写项目。

## 识别顺序

1. 前 15 行出现「立创商城BOM报价单」，或前 40 行有同时包含「商品编号」和 `Designator` 的表头行 → `lcsc_quote`
2. 否则表头同时含 `Manufacturer Part`、`LCSC Price`、`Value` → `easyeda_v131`
3. 报价单标题在但找不到表头，或两种都不符 → 「无法识别文件格式」
4. 格式认得但没有有效明细 → 「没有可导入的物料明细」

表头比较忽略大小写和多余空白。

## 立创 EDA `easyeda_v131`

表头在数据前的某一行，样例在第一行。已用列：

| 列 | 写入 |
| --- | --- |
| No. | `line_no` |
| Quantity | `quantity_per_board` 和 `quantity_required` |
| Footprint | `footprint` |
| Manufacturer | `brand` |
| Manufacturer Part | `mpn` |
| Value | `value` |
| LCSC Price | 只用于识别格式，不入库 |

其余有值的列拼进 `params`，格式 `列名: 值`，分隔符 `；`。`lcsc_code`、`designators`、`product_name`、`category` 为空。`purchase_sets = 1`。

跳过全空行，以及型号、标称值、封装、品牌都空的行。

样例 `temp/BOM_V1.31_V1.31_2026-10-23.xlsx`：45 条有效物料。首条 `CL10A105KB8NNNC` / `C0603` / `1uF`。末条序号 45，`0603WAF1002T5E` / `R0603` / `10kΩ` / 用量 21。文件总行数 47（表头 + 45 条 + 末尾空行）。

## 立创商城报价单 `lcsc_quote`

表头前的单元格用正则取：

- `BOM名称：` → 项目名
- `采购套数：` → `purchase_sets`，缺省 1
- `BOM配单编号：` → `order_no`

样例表头在第 6 行（0 起下标 5）。列：

| 列 | 写入 |
| --- | --- |
| ID | `line_no` |
| Designator | `designators` |
| Quantity | `quantity_per_board` |
| 购买数量 | 有该列且单元格非空时作为 `quantity_required`；否则 `Quantity × 采购套数` |
| 商品编号 | `lcsc_code` |
| 型号，否则 Manufacturer Part | `mpn` |
| 品牌，否则 Manufacturer | `brand` |
| 封装，否则 Footprint | `footprint`。两者都有且不同时，把 `原始封装: ...` 追加到参数 |
| 目录 | `category` |
| 商品名称 | `product_name` |
| 原参数、参数 | 拼进 `params` |

标称值优先级：

1. 参数里的 `阻值：`、`容值：`、`电感量：`、`电感值：` 后的第一段
2. 否则 `Name` 列看起来像标称值（含数字，且含 Ω/ohm/F/H 或以 k、m 结尾）
3. 否则 `value` 为空。`Name` 里的型号不会当成标称值

有效行至少要有立创编号、型号、位号、商品名称、标称值之一。

样例 `temp/树莓派_...立创_20261020.xls`：9 条明细，采购套数 2，配单编号 `BOM261020000440`。`C105580` 的位号是 `R13,R14`，单板用量 2，需求总量取购买数量 100，型号 `RC0603FR-075K1L`，封装 `0603`，标称值 `5.1kΩ`，类目「电阻」。
