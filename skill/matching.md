# 匹配

纯函数 `match_lines` 在 `backend/app/services/match.py`。`apply_match` 读取该用户全部物料和项目明细，写回 `match_status`、`matched_item_id`、`suggestions_json`、`matched_at`。导入和「重新匹配」都走它。

精确命中优先于替换建议。精确命中的明细不会发给 DeepSeek。

## 状态

| status | 含义 |
| --- | --- |
| matched | 精确命中，且选中物料库存 ≥ 需求总量 |
| insufficient | 精确命中，但库存 < 需求总量 |
| suggested | 无精确命中，规则预筛或 DeepSeek 给出了候选 |
| not_found | 无精确命中，也没有可展示的候选 |

## 精确命中

满足其一即可，同一物料只计一次：

- 立创编号去空白后忽略大小写相等，且两边都非空
- 否则厂家型号去掉全部空白后忽略大小写相等，且两边都非空

多条时选一条，优先级：

1. 库存够且封装主名一致
2. 库存够
3. 封装主名一致
4. 库存更大
5. id 更小

`suggestions.source = "exact"`。编号命中时 `reason` 为「立创编号一致」，否则「厂家型号一致」。`exact_hits` 含全部命中，选中项在前。`candidates` 为空。

## 封装主名与标称值

`footprint_main` 从封装文本提取：

- 片式：`0201` `0402` `0603` `0805` `1206` `1210` `1812` `2010` `2512`。`R0603`、`C0603`、`LED0603` 都是 `0603`
- `SOT-23-3` 与 `SOT23` → `SOT-23`；`SOT-223` 保持 `SOT-223`
- `SOD-123`、`SOD-123FL` 彼此不同
- `SOP-8`、`ESOP-8`、`QFN`、`DFN`、`LQFP`、`TO-220` 等保留数字
- 以 `SMA` / `SMB` / `SMC` 开头的用该名字

`norm_value`：小写，`μ/µ` 视为 `u`，去掉 `ohm` 和 `Ω`。因此 `5.1kΩ` 与 `5.1k`、`4.7μF` 与 `4.7uF` 相同。

类目标签看 `category`，为空时再看商品名称或产品名。关键词：电阻、电容、电感、二极管、MOS、三极管、LED、连接器、电源，含常见英文。

## 规则预筛

未精确命中的库内物料，满足以下任一即入候选，最多 20 条：

- 封装主名相同且都非空
- 标称值归一化后相同且都非空
- 类目标签有交集

排序：命中条件数多的在前，然后库存大，然后 id 小。

理由由命中的条件拼成，例如「封装主名同为 0603，标称值同为 10kΩ」。封装主名都有但不相同，风险加「封装」。类目都有但无交集，风险加「类目」。

未启用 DeepSeek，或该条模型调用失败时：有候选则 `suggested`，选第一条为 `matched_item_id`，`source` 为 `rule`。没有候选则 `not_found`。

同封装同阻值的电阻即使没有立创编号、也没有 DeepSeek，也必须是 `suggested`，理由里同时有封装主名和标称值。

## DeepSeek

仅当 `DEEPSEEK_API_KEY` 非空。接口 `{DEEPSEEK_BASE_URL}/chat/completions`，默认基址 `https://api.deepseek.com`，模型 `deepseek-chat`，OpenAI 兼容，`response_format` 为 json_object。每批最多 5 条明细。

只发送未精确命中且已有规则候选的明细。模型返回：

```json
{"results":[{"line_id":1,"picks":[{"item_id":2,"suitable":true,"reason":"...","risks":["精度"]}]}]}
```

`suitable` 不为真，或 `item_id` 不在该明细候选中的，丢弃。风险只保留：耐压、精度、功率、温度、封装、库存。

- 本批请求失败：该批 `line_id` 对应 `None`，改用规则结果
- 请求成功但没有可用 pick：`not_found`，`source` 为 `ai`
- 有可用 pick：`suggested`，`source` 为 `ai`，理由优先用模型原文

`GET /api/projects/{id}` 的 `ai_enabled` 只反映当前是否配置了密钥。前端在为 false 时显示「未启用智能建议」。

## suggestions_json

```json
{
  "source": "exact",
  "reason": "立创编号一致",
  "risks": [],
  "exact_hits": [],
  "candidates": []
}
```

`exact_hits` 与 `candidates` 的元素含 `item_id, lcsc_code, mpn, name, footprint, value, category, quantity, box_name, row, col, suitable, reason, risks`。

地图只给 `matched_item_id` 标位号，不给其余候选标。位号按逗号、中文逗号和空白拆开。没有位号则不标。
