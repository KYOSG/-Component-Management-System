# 前端

Vue 3 + Vue CLI 5 + Element Plus + Vue Router 4。源码在 `frontend/src`。界面文案为简体中文。主色是 Element Plus 变量上的靛蓝 `#3f51b5`，写在 `styles.css`。

`npm run serve` 端口 8080，`vue.config.js` 把 `/api` 代理到 `127.0.0.1:8000`。`npm run build` 产出 `frontend/dist`。

## 路由

守卫调用 `GET /api/auth/me`。失败则去 `/login?redirect=`，且 redirect 只接受以 `/` 开头的站内路径。

| 路径 | 文件 | 作用 |
| --- | --- | --- |
| `/login` `/register` | `views/Login.vue` `views/Register.vue` | 公开 |
| `/items` | `views/Items.vue` | 物料表、筛选、分页、抽屉新增编辑、移出、删除 |
| `/map` | `views/Map.vue` | 统计、盒子 CRUD、完整格子、空格放入或新建、有料编辑或移出、类目筛选、项目位号高亮 |
| `/projects` | `views/Projects.vue` | 拖拽上传后进入明细 |
| `/projects/:id` | `views/ProjectDetail.vue` | 匹配表、重新匹配、跳到 `/map?project=` |
| `/users` | `views/Users.vue` | 账户增删改查。路由 `meta.admin`，非管理员访问时回到 `/items` |

布局是 `layouts/AdminLayout.vue`：左侧菜单、顶栏账号与退出。只有 `is_admin` 为真时显示「账户管理」。登录页账号框提交字段仍是 `email`。

## 共用行为

- `api.js`：axios `baseURL=/api`，带 Cookie。401 不弹消息，其它错误用 `detail` 弹错误。
- `components/ItemDrawer.vue`：新增和编辑共用。立创编号旁的「查询」调用 `GET /api/lcsc/{code}`，回填资料，但保留用户已填的库存、盒子、行列、备注。
- 保存始终提交完整字段（`format.js` 的 `itemPayload`）。有盒子却没有行列时，前端先提示，不发请求。
- 匹配状态标签：`matched` 已匹配，`insufficient` 库存不足，`suggested` 建议替换，`not_found` 未找到。
- 格式标签：`easyeda_v131` 立创 EDA，`lcsc_quote` 立创商城报价单。
- 明细上「按编号加入物料库」只在有立创编号且状态为 `not_found` 或 `suggested` 时出现。保存后调用重新匹配。
- 地图高亮读 `highlights[物料id]`。类目筛选时其它类目格子变淡，仍显示为占用，不能当成空格放入。

## 环境变量

见项目根目录 `.env.example`。

| 变量 | 作用 |
| --- | --- |
| DATABASE_URL | 默认 `sqlite:///./data/app.db`。Docker 使用 `sqlite:////data/app.db` |
| SECRET_KEY | Cookie 签名 |
| DEEPSEEK_API_KEY | 空则关闭智能建议 |
| DEEPSEEK_BASE_URL | 默认 `https://api.deepseek.com` |
| DEEPSEEK_MODEL | 默认 `deepseek-chat` |
| COOKIE_SECURE | `1` 时 Cookie 仅 HTTPS |
| STATIC_DIR | 指向 Vue `dist` 时由 FastAPI 托管页面 |

密钥只读环境变量，不进前端包，不写进仓库。
