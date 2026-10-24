# 接口

除特别说明外都要登录。未登录 401，`detail` 为「请先登录」。数据只属于当前用户。

Cookie 名 `cms_session`，HttpOnly，SameSite=Lax，路径 `/`，14 天。值为 `userId.expires.hmac`。密钥 `SECRET_KEY`。`COOKIE_SECURE=1` 时加 Secure。

开发时前端 `localhost:8080` 通过 axios `withCredentials` 访问后端。CORS 允许 localhost / 127.0.0.1 任意端口。

错误体是 FastAPI 的 `{ "detail": "中文说明" }`。

## 登录

| 方法 | 路径 | 行为 |
| --- | --- | --- |
| POST | `/api/auth/register` | `{email, password}`。邮箱转小写。密码 6–72 字节。成功后写 Cookie |
| POST | `/api/auth/login` | 账号或邮箱均可。失败 400「账号或密码错误」 |
| POST | `/api/auth/logout` | 删 Cookie |
| GET | `/api/auth/me` | `{id, email, is_admin, created_at}` |

初始管理员账号 `admin`，密码 `admin`。`is_admin` 为真时前端才显示账户管理。

## 账户管理

仅管理员。普通用户 403「需要管理员权限」。

| 方法 | 路径 | 行为 |
| --- | --- | --- |
| GET | `/api/users` | 全部账号 |
| POST | `/api/users` | `{email, password, is_admin}`。账号转小写，密码至少 6 位 |
| PUT | `/api/users/{id}` | 同样的字段。`password` 为空则不改密码 |
| DELETE | `/api/users/{id}` | 删除该用户及其物料、物料盒、项目和明细 |

不能删除当前登录账号。不能把最后一名管理员改成普通用户或删掉。账号重复时 400「账号已存在」。

## 立创

`GET /api/lcsc/{code}`

服务端请求 `https://wmsc.lcsc.com/ftps/wm/product/detail?productCode={code}`。失败 400「查询失败，请手填」。

返回字段：`lcsc_code, mpn, name, brand, footprint, category, value, params, datasheet_url, image_url, mall_stock, mall_price`。

映射在 `map_product`：

- 型号 `productModel`，名称 `productNameEn`，品牌 `brandNameEn`，封装 `encapStandard`
- 类目由 `parentCatalogName` / `catalogName` 归成中文：电阻、电容、电感、二极管、MOS、三极管、连接器、电源
- 标称值取参数名 阻值 / 容值 / 电感量 / 电感值（英文名 resistance、capacitance、inductance 亦可）
- `params` 为 `参数名: 参数值`，用 `；` 连接 `paramVOList`
- 图片用 `productImages[0]`，价格用 `productPriceList[0]`

## 物料

`GET /api/items` 分页。查询参数：`page` `page_size`（最大 100）以及 `lcsc_code` `mpn` `name` `footprint` `category` `remark` 的包含筛选。

`POST /api/items`、`PUT /api/items/{id}` 的主体是完整物料字段，不是部分更新。

`DELETE /api/items/{id}`、`POST /api/items/{id}/unbox`。

## 物料盒与地图

| 方法 | 路径 |
| --- | --- |
| GET, POST | `/api/boxes` |
| PUT, DELETE | `/api/boxes/{id}` |
| GET | `/api/map` |

`GET /api/map` 查询参数：`category`、`project_id`。

返回：

- `stats`：`item_count` 物料种类、`box_count`、`quantity_total` 库存合计。统计不受类目筛选影响
- `categories`：库中出现过的类目
- `boxes[]`：`cells` 按行优先铺满，含空格子。`dimmed` 表示该类目筛选下不是当前类目，格子仍占用
- `invalid_items`：盒内坐标不合法的物料
- `unboxed`：未入盒
- `highlights`：键是物料 id 字符串，值是位号列表。只标 `matched`、`insufficient`、`suggested` 且有 `matched_item_id` 的明细

## 项目

| 方法 | 路径 | 行为 |
| --- | --- | --- |
| GET | `/api/projects` | 不含明细，含 `line_count` |
| GET | `/api/projects/{id}` | 含明细。`ai_enabled` 表示当前进程是否配置了 DeepSeek，不是历史记录 |
| POST | `/api/projects/import` | multipart 字段名 `file`。最大 10MB。解析成功后自动匹配 |
| POST | `/api/projects/{id}/match` | 用当前物料库重新匹配 |
| POST | `/api/projects/{id}/lines/{lineId}/adopt` | 按明细立创编号预填。无编号则 400 |

`adopt` 的 `quantity` 固定为 0。立创失败时用 BOM 字段预填，并带 `warning`。`mall_stock` 仍只供展示。

明细额外字段：

- `reason`、`risks`、`locations`、`stock` 从 `suggestions_json` 抽出
- `stock` 是第一条命中或第一条候选的库存
- `locations` 形如 `型号 · 盒子 第R行第C列（库存 N）`

生产环境设置 `STATIC_DIR` 后，FastAPI 用同一进程提供 Vue 构建结果。未知路径回退 `index.html`，`/api` 路由先注册。
