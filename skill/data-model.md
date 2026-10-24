# 数据模型

定义在 `backend/app/models.py`。启动时 `create_all`。默认 SQLite，`DATABASE_URL` 可改为 `mysql+pymysql://...?...charset=utf8mb4`。

空字符串立创编号在写入前变成 `NULL`（`clean_code`）。SQLite 与 MySQL 的唯一索引都允许多个 NULL。

## users

| 字段 | 含义 |
| --- | --- |
| email | 登录账号，小写保存，全局唯一。自助注册必须是邮箱；初始管理员账号是 `admin` |
| password_hash | bcrypt |
| is_admin | 是否管理员。自助注册为否。初始账号 `admin` 为是 |
| created_at | UTC |

无邮件找回密码。启动时若数据库不存在则创建。账号 `admin` 不存在时写入，密码 `admin`，已存在则不改密码。

删除账号时，按该 `user_id` 删除其项目明细、项目、物料和物料盒，并先把指向这些物料的 `matched_item_id` 置空。不能删除当前登录账号，也不能取消或删掉最后一名管理员。

## boxes

创建时写入当前用户的 `user_id`。同一 `user_id` 下 `name` 唯一。

| 字段 | 含义 |
| --- | --- |
| rows, cols | 创建时指定，盒子之间可以不同。合法范围 1–100 |
| remark | 可空字符串，默认空 |

## items

一条记录是库中一种料及其位置。创建时写入当前用户的 `user_id`。

| 字段 | 含义 |
| --- | --- |
| lcsc_code | 如 `C105580`。有值时 `(user_id, lcsc_code)` 唯一 |
| mpn, name, brand, footprint, category, value | 厂家型号、商品名称、品牌、封装、类目、标称值 |
| params | 参数原文，给人看，也参与展示 |
| quantity | 自有库存，整数，≥ 0 |
| box_id, row, col | 空表示未入盒。有盒子时行、列必填且在范围内 |
| datasheet_url, image_url, remark | |

约束：`(box_id, row, col)` 唯一。未入盒时这三列为 NULL，可有多条。

放置规则在 `apply_fields`：

- 无 `box_id` 时强制 `row/col = NULL`
- 格子已被其他物料占用：400「该格子已有物料」
- 超出盒子：400「格子超出物料盒范围」
- 编号重复：400「立创编号已存在」

## projects

导入的项目，不改库存。

| 字段 | 含义 |
| --- | --- |
| name | 报价单用 BOM 名称，否则用文件名 |
| source_filename | 仅文件名 |
| format | `easyeda_v131` 或 `lcsc_quote` |
| purchase_sets | 采购套数。EDA 固定 1 |
| order_no | 报价单的 BOM 配单编号，EDA 为空 |
| imported_at | |

## project_lines

| 字段 | 含义 |
| --- | --- |
| line_no | EDA 用 `No.`，报价单用 `ID` |
| designators | 位号原文，如 `R13,R14`。EDA 为空 |
| quantity_per_board | 单板用量 |
| quantity_required | 需求总量 |
| lcsc_code, mpn, brand, footprint, value, params, product_name, category | 明细字段，空则为 `""` |
| match_status | `matched` `insufficient` `suggested` `not_found`，导入后立即写入 |
| matched_item_id | 选中的库内物料。建议替换时是排序后的第一条候选 |
| suggestions_json | 见 [matching.md](matching.md) |
| matched_at | 本次匹配时间 |

## 删除与位置

| 动作 | 结果 |
| --- | --- |
| 移出 `POST /api/items/{id}/unbox` | 只清空 `box_id/row/col` |
| 删除物料 | 先把指向它的 `matched_item_id` 置空，再删物料 |
| 删除账号 | 先清空指向其物料的 `matched_item_id`，再删该用户的项目明细、项目、物料、物料盒和用户 |
| 删除盒子 | 盒内物料变为未入盒，再删盒子 |
| 缩小行列 | 存在 `row > 新行数` 或 `col > 新列数` 的物料则 400，不保存 |

坐标不合法（空、小于 1、超出当前行列）的物料不占格子，出现在地图卡片下方的 `invalid_items`。
