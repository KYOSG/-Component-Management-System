---
name: component-mapping
description: >-
  物品映射系统的业务逻辑索引。修改物料库、物料盒地图、BOM 导入、立创查询、库存匹配、登录、账户管理或相关接口时先读本技能。
  触发词：物料、物料盒、BOM、立创、LCSC、匹配、DeepSeek、物品映射、管理员、账户。
---

# 物品映射

个人物料库。范围只有登录、物料、物料盒地图、BOM 导入与匹配。不含设备监控、围栏、日志、产品中心、扫码填表。

改代码前先按问题打开对应文件，不要通读全部。

| 问题 | 读 |
| --- | --- |
| 表、字段、唯一约束、删除副作用 | [data-model.md](data-model.md) |
| URL、登录、错误文案、请求体 | [api.md](api.md) |
| xls/xlsx 识别、列映射、数量 | [bom.md](bom.md) |
| 已匹配 / 库存不足 / 建议替换 / 未找到 | [matching.md](matching.md) |
| 页面、路由、谁调用哪个接口 | [frontend.md](frontend.md) |

## 不可改错的规则

- 所有业务数据按 `users.id` 隔离。查询必须带当前用户。物料和物料盒写入时必须带 `user_id`。
- 启动时数据库不存在则创建。账号 `admin` 不存在则写入管理员，密码 `admin`。已存在则不重置密码。
- 删除用户会连带删除该用户的物料、物料盒、项目和明细。
- 自有库存是 `items.quantity`。立创返回的商城库存和价格只展示，不写入 `quantity`。
- 空立创编号存 `NULL`，不存空字符串。同一用户有值时唯一。
- 同一盒子的同一 `(row, col)` 只能一条物料。行列从 1 开始。
- 缩小盒子时，若已有物料的行或列超出新范围，拒绝保存，文案含「移走」。
- 删除盒子只把盒内物料的 `box_id/row/col` 清空，不删物料。
- 移出盒子同样只清空位置。
- 删除物料前把引用它的 `project_lines.matched_item_id` 置空。
- BOM 导入不改库存。识别失败不创建空项目。
- `POST .../adopt` 只预填表单，不入库。用户保存走 `POST /api/items`。
- 未配置 `DEEPSEEK_API_KEY` 时仍做精确匹配和规则预筛，页面说明未启用智能建议。
- DeepSeek 只能从该明细的候选 `item_id` 里选。候选外的 id 丢弃。

## 代码入口

| 职责 | 路径 |
| --- | --- |
| 应用与静态页 | `backend/app/main.py` |
| 表 | `backend/app/models.py` |
| 登录 Cookie | `backend/app/security.py`、`backend/app/deps.py`、`backend/app/routers/auth.py` |
| 启动建库与初始管理员 | `backend/app/bootstrap.py` 的 `prepare_database` |
| 账户管理 | `backend/app/routers/users.py` |
| 物料放置校验 | `backend/app/routers/items.py` 的 `apply_fields` |
| 盒子与地图 | `backend/app/routers/boxes.py` |
| BOM 解析 | `backend/app/services/bom.py` 的 `parse_upload` |
| 匹配纯函数 | `backend/app/services/match.py` 的 `match_lines` |
| 匹配落库 | `backend/app/services/matching.py` 的 `apply_match` |
| 立创查询 | `backend/app/services/lcsc.py` 的 `fetch_lcsc` |
| DeepSeek | `backend/app/services/deepseek.py` 的 `suggest` |
| 前端路由 | `frontend/src/router/index.js` |
| 接口客户端 | `frontend/src/api.js` |

样例文件在 `temp/`。EDA 样例有效物料是 45 条（序号 1–45），末尾有空行。报价单采购套数是 2，含 `C105580` 与位号 `R13,R14`。

测试：在 `backend` 目录执行 `.venv\Scripts\python -m pytest`。
