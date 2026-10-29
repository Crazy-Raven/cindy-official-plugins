# 读帖子：列表、详情与子版块

本文件是 `taptap-community-management` 的「读帖子」场景细节。场景路由、CRITICAL 规则和输出边界以 [community-management 手册](taptap-suite/references/taptap-community-management/MANUAL.md) 为准。

## 调用 scope 与目标论坛

查帖前先区分两个角色，不要把论坛 URL 中的 App ID 直接当作调用 scope：

| 字段 | 角色与来源 |
| --- | --- |
| `args.developer_id` | 当前账号已确认可用的厂商 ID |
| `args.app_id` | 上述厂商下已确认可用的游戏 ID |
| `args.data.forum_app_id` | 实际要查的论坛所属 App ID，十进制字符串，放在 `args.data` 中；省略时查调用 scope 的 App 论坛 |

例如论坛地址为 `https://www.taptap.cn/app/<targetAppId>/topic`，则将其中的 App ID 填入 `forum_app_id`。沿用已确认可用的厂商与游戏 scope；缺少调用 scope 时转 [identity 手册](taptap-suite/references/taptap-identity/MANUAL.md) 发现，不能猜 ID 或将不属于该厂商的目标 App 填入 `args.app_id`。多个候选且无法确定调用 scope 时让用户选择。

`list-community-moments` 与 `list-community-group-labels` 均支持通过 `forum_app_id` 指定目标论坛，结果仅包含当前用户可查看的内容。读取结果不代表拥有目标论坛的发帖、回帖或管理权限。

## 命令

| 目标 | 命令 |
| --- | --- |
| 帖子列表 | `list-community-moments` |
| 帖子正文与图片 | `get-community-moment` |
| 子版块列表 | `list-community-group-labels` |

`list-community-moments` 支持分页（`page` / `page_size`，也可写进 `args.data`）；`get-community-moment` 和 `list-community-group-labels` 不分页。

```text
# 最近回复的帖子（默认 order=commented）
call_tool(name:"community-management list-community-moments", args:{
  developer_id:"<developerId>", app_id:"<appId>",
  data:{page:1, page_size:20}})

# 按发布时间
call_tool(name:"community-management list-community-moments", args:{
  developer_id:"<developerId>", app_id:"<appId>",
  data:{order:"published", page:1, page_size:20}})

# 跨论坛：两个 scope 占位符必须替换为已确认可用的厂商与游戏 ID
call_tool(name:"community-management list-community-moments", args:{
  developer_id:"<authorizedDeveloperId>", app_id:"<authorizedAppId>",
  data:{forum_app_id:"<targetAppId>", moment_intent:["1", "2"], order:"published", page:1, page_size:100}})

# 子版块列表（省略 forum_app_id 时查询调用 scope 的论坛）
call_tool(name:"community-management list-community-group-labels", args:{
  developer_id:"<developerId>", app_id:"<appId>"})

# 目标论坛子版块（与查帖使用相同的 forum_app_id）
call_tool(name:"community-management list-community-group-labels", args:{
  developer_id:"<authorizedDeveloperId>", app_id:"<authorizedAppId>",
  data:{forum_app_id:"<targetAppId>"}})

# 帖子详情（正文与图片）
call_tool(name:"community-management get-community-moment", args:{
  developer_id:"<developerId>", app_id:"<appId>",
  data:{moment_id:"<momentId>"}})
```

跨论坛列表返回 `moment_id` 后，读取详情时保留同一个已授权调用 scope，原样传入该字符串 ID。`get-community-moment` 按 ID 读取公开可见帖子，不要把目标论坛 App 改填到 `args.app_id`，也不要给未声明该字段的接口追加 `forum_app_id`。

## 权限错误排查

收到 `authorization/permission_denied` 或 HTTP 403 时，先报告“本次请求被拒绝”，不要直接断言目标论坛不可读或要求换账号：

1. 先用 `list_tools(category:"community-management")` 查当前操作，必要时通过 `call_tool(name:"schema", args:{_positional:["community-management","<method>"]})` 读取 `list-community-moments` 或 `list-community-group-labels` 的实时 schema，确认当前版本是否声明 `forum_app_id`；实时契约优先于旧示例，不向旧版本猜传字段。
2. 检查 `developer_id` 与 `app_id` 是否为同一厂商下已确认可用的调用 scope，以及是否误把目标论坛 App 填入了 `app_id`。通用“目标资源无权限”错误本身不能定位失败字段。
3. 若确认参数角色用错，保持用户要查的论坛不变，将调用 scope 改为已授权组合、目标 App 放入 `forum_app_id`，重新执行一次只读查询。这是纠正请求，不是绕过权限。
4. 正确参数仍被拒绝时停止，不轮换游戏或账号探测权限；保留实际错误，根据错误信息说明下一步；错误未明确原因时，不推断具体权限限制。

## 列表参数

| 参数 | 行为 |
| --- | --- |
| `forum_app_id` | 目标论坛所属 App ID（字符串）；省略时查调用 scope 的 App 论坛 |
| `order=commented` | 按最后回复时间倒序（默认） |
| `order=published` | 按发布时间倒序 |
| `group_label_id` | 只看目标论坛的该子版块；先用相同 `forum_app_id` 查询 `list-community-group-labels` 取得，不传表示不限 |
| `moment_intent=["1"]` | 问答求助 |
| `moment_intent=["2"]` | 反馈建议 |
| `moment_intent=["1","2"]` | 两类都返回（多值之间是「或」） |
| `page` / `page_size` | 分页，`page_size` 取 1–100，默认 20 |

`order` 与 `moment_intent` 是枚举：非法值会被直接拒绝，不会静默降级。

> **小贴士**：想快速捞「玩家求助」和「反馈建议」时，优先用 `moment_intent`，比在正文里搜关键字更准：`"1"` = 问答求助，`"2"` = 反馈建议。只看求助传 `["1"]`，只看反馈传 `["2"]`，两类一起看传 `["1","2"]`（多个值是「或」）；不传表示不限意图。

## 子版块（group label）

`list-community-group-labels` 接受可选 `forum_app_id`（字符串），省略时查询调用 scope 的 App，传入时查询目标 App 的论坛。按官方顺序返回（先按配置的排序值升序，再按创建时间倒序），**不分页**，一次返回当前用户可查看的全部子版块。

| 字段 | 说明 |
| --- | --- |
| `group_label_id` | 子版块 ID（字符串），直接作为 `list-community-moments` 的 `group_label_id` 筛选参数 |
| `name` | 论坛管理中心配置的名称 |
| `display_name` | 展示名称；可能为 `null`，为空时回退 `name` |
| `is_disabled` | 禁止普通用户在该子版块发帖（版块仍可见、可浏览），只有论坛管理员可发；与另外两个标记相互独立 |
| `is_isolated` | 隔离版块：该子版块的帖不进入论坛首页聚合信息流，仅作为独立版块展示 |
| `is_offline` | 下线：该子版块在前台（首页与发帖页）整体隐藏，只有论坛管理中心可见 |

- 结果仅包含当前用户可查看的子版块；版块状态以返回的标记为准，未返回某个版块不代表它不存在。
- 论坛未开通时无法取得子版块。
- 直接用返回的 `group_label_id` 过滤帖子，不要按版块名猜 ID。
- 子版块查询与后续帖子查询必须使用同一个 `forum_app_id`，不能混用其他论坛的 `group_label_id`。查询失败时按实际错误处理，不推断未返回版块的状态。

## 帖子字段

列表项（`moment`）：

| 字段 | 说明 |
| --- | --- |
| `moment_id` | 帖子 ID（字符串，原样回传） |
| `title` | 标题；长贴与视频动态有标题，图文动态为 `null` |
| `summary` | 摘要；长贴取摘要、视频动态取视频描述，图文动态为 `null` |
| `url` | 帖子在 TapTap 社区的访问地址 |
| `author` | 发帖人；资料不可见时为 `null`（`user_id` / `url` 仍返回） |
| `publish_time` | 发布时间（Unix 秒）；定时未发布是未来时间 |
| `commented_time` | 最后回复时间（Unix 秒） |
| `group_label` / `group_label_id` | 所属子版块名称 / ID；未绑定子版块时为 `null` |
| `has_video` | 是否带视频信息（不等于帖子就是视频帖） |
| `stat` | 互动统计：`comments`（含子回复）/ `favorites` / `pv_total` / `reposts` / `ups` |
| `status` | 管理状态：`is_top` / `is_elite` / `is_down` / `is_focus` / `is_quiz` / `is_treasure` / `is_official` / `is_group_label_top` / `is_scheduled` |

详情额外返回：`content`（正文原文，保留 BBCode；无正文为空串）、`images`（正文图 + 尾部组图，同地址去重）、`video`（无视频为 `null`；只含 `video_id` / `cover_url` / `duration_seconds` / `aspect_ratio`，**不含播放地址**）。

## 取数与输出

- `list-community-moments` 的 `total` 是官方统计口径；已删除或不可见的帖会被剔除，因此单页条数可能少于 `page_size`。这不是缺页。
- 取数前先用 `group_label_id`、`moment_intent`、`order` 缩小范围，再按需翻少量页；优先不用 `page_all` 全量拉。
- `content` 保留原帖 BBCode 标记（如 `[b]` / `[i]`）；转述时不要静默改写、删除标记或把标记当正文。`images` 给 URL，不做图片内容判断。
- `title` / `summary` 可能为 `null`，`author` 资料不可见时也为 `null`；不要编造标题或昵称，也不要回退展示 `user_id`。
- 子版块名称优先用 `display_name`，为 `null` 时回退 `name`；不要把停用 / 隔离 / 下线说成「不存在」。
- `has_video` 只表示「有视频信息可看」，不等于帖子本身就是视频帖；`video` 只含封面与元信息，**没有播放地址**，不要承诺能播放。
- `moment_id` / `user_id` 只在用户明确要排查时展示；默认用标题、时间和正文摘要指代。

## 组合配方

| 目标 | 命令与 `args.data` |
| --- | --- |
| 最近回复的帖子 | `list-community-moments` `{"page":1,"page_size":20}` |
| 按发布时间 | `list-community-moments` `{"order":"published","page":1,"page_size":20}` |
| 跨论坛求助与反馈 | `list-community-moments` `{"forum_app_id":"<targetAppId>","moment_intent":["1","2"],"order":"published","page":1,"page_size":100}`；保留已授权调用 scope |
| 问答求助 | `list-community-moments` `{"moment_intent":["1"],"page":1,"page_size":20}` |
| 某子版块 | `list-community-moments` `{"group_label_id":"<id>","page":1,"page_size":20}` |
| 论坛子版块 | `list-community-group-labels` `{"forum_app_id":"<targetAppId>"}`；保留已授权调用 scope，不传 `forum_app_id` 则查该 scope 的论坛 |
| 帖子详情 | `get-community-moment` `{"moment_id":"<momentId>"}` |
