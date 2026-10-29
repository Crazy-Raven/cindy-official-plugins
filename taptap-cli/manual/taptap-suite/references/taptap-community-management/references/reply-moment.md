# 回帖:一级回复与子回复

本文件是 `taptap-community-management` 的「回帖」场景细节。场景路由、CRITICAL 规则和输出边界以 [community-management 手册](taptap-suite/references/taptap-community-management/MANUAL.md) 为准。

## 这条路径是什么

两个写操作,都是**以调用方个人身份**回复,成功返回新回复(审核通过前不代表已对外可见):

| 场景 | operation | 必需参数 |
| --- | --- | --- |
| 回复帖子本身(一级回复) | `create-community-moment-reply` | `moment_id`、`contents` |
| 回复某条一级回复(子回复) | `create-community-moment-reply-child` | `moment_id`、`parent_reply_id`、`contents` |

```text
call_tool(name:"community-management create-community-moment-reply", args:{
  developer_id:"<developerId>", app_id:"<appId>",
  data:{moment_id:"<momentId>",
        contents:"[{\"type\":\"paragraph\",\"children\":[{\"text\":\"收到,我们看下\"}]}]"},
  dry_run:true})          # 用户确认后改成 yes:true

# 子回复:parent_reply_id 取一级回复的 comment_id
call_tool(name:"community-management create-community-moment-reply-child", args:{
  developer_id:"<developerId>", app_id:"<appId>",
  data:{moment_id:"<momentId>", parent_reply_id:"<commentId>",
        contents:"[{\"type\":\"paragraph\",\"children\":[{\"text\":\"…\"}]}]"},
  dry_run:true})          # 用户确认后改成 yes:true
```

`dry_run:true` 打印请求预览;写调用缺 `dry_run` / `yes` 会被插件拒绝(`CONFIRM_REQUIRED`)。

## 前置条件

- 帖子必须**公开可见**:审核中、已删除或非公开的帖子无法回复。
- 权限:看**作用域** App 的论坛版主 / 主管理员权限,不是目标帖子所在论坛的权限;`developer_id` / `app_id` 用 scope 参数传,不要重复写进 `data`。
- 回帖没有官方身份选项:固定按个人身份发布。

## 作用域与投递目标

- `app_id` / `developer_id` 是**作用域**:决定这条回复挂在哪个 App 的作者名下,鉴权只看它。
- **回复目标由 `moment_id` 决定**,可以和作用域 App 不同。例如作用域保持 `<appId>`,去回复 app `<targetAppId>` 论坛里的帖子,不需要把 `app_id` 改成 `<targetAppId>`;改了反而会把回复挂到另一个 App 名下。
- 权限按**作用域** App 判定,与目标帖子所在论坛无关:作用域具备论坛管理权限,就能回复其它 App 论坛下的帖子。

## 参数

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `moment_id` | string | 帖子 ID,十进制字符串,原样回传 |
| `contents` | string | 正文,**JSON 字符串**,见下节限制 |
| `parent_reply_id` | string | 仅子回复:父级**一级回复**的 `comment_id`,取自 `list-community-moment-replies` |
| `reply_to_reply_id` | string \| null | 仅子回复:被回复的回复 ID,可为任意层级但需同帖;不传表示不指定。填写后会自动 at 被回复的人 |

**CRITICAL — `parent_reply_id` 必须是一级回复。** 想回复「回复下的回复」时,`parent_reply_id` 仍传那条一级回复,用 `reply_to_reply_id` 指向更深的那条;把子回复 ID 直接当 `parent_reply_id` 会被服务端拒绝。

**CRITICAL — 填了 `reply_to_reply_id` 就不要在正文里重复 at。** 服务端会自动 at 被回复的人,正文再手写 `mention` 会出现两次 at;只有还需要额外提醒其他人时,才自己在正文加 `mention`。

## 正文限制

`contents` 是 `ContentJsonItem[]` 序列化后的字符串,但回复编辑器比发帖窄得多:

| 层级 | 一级回复 | 子回复 |
| --- | --- | --- |
| 顶层节点 | `paragraph`、`image` | 仅 `paragraph` |
| 行内片段 | 纯文字、`mention`(`info.id` + `info.name`)、`tap_emoji`(`info.id`) | 同左 |
| 图片 | 最多 9 张,地址需来自 `+upload-image` 或平台已收录 | 不支持 |
| 样式 | 不支持粗体、斜体、下划线、删除线、行内代码、高亮、颜色、对齐、引用块、链接、hashtag、列表 | 同左 |
| 长度 | ≤ 20000 字符 | 同左 |

**CRITICAL — 不要回传读接口里的正文。** 读接口返回的正文可能带加粗、颜色、链接等样式,原样回传会被服务端拒绝;回复正文要按上面的白名单自己构造。

## 读回与验证

| 手段 | 能看到什么 |
| --- | --- |
| `list-community-moment-replies` | 一级回复列表,`comment_id` 就是后续子回复要用的 `parent_reply_id` |
| `list-community-moment-child-replies` | 某条一级回复下的子回复 |
| `get-community-moment` | 帖子详情;审核中会 422 |

新回复在过审前可能不出现在列表里,这属正常状态,不要重复提交。

## 幂等与重试

同一条命令按同一意图重试不会重复回复(CLI 会复用同一次写入意图),但**结果未知不能当未执行**:

- 超时、`ambiguous_outcome` 或连接中断后,先核对远端实际状态:用 `list-community-moment-replies` / `list-community-moment-child-replies` 看这条回复是否已经出现。帖子审核中查不到**不能**据此断定没发出去。
- 只有确认这次写入没有生效,才用相同内容、同一意图重试;不要改文案重发,否则可能多出一条回复。
- 核对不了(读接口也不可用)时,向用户说明这次回复结果不确定、可能已经发出;先不要重发,等能读到列表或详情后再判断。

## 失败怎么解释

| 现象 | 含义 | 下一步 |
| --- | --- | --- |
| `CONFIRM_REQUIRED`(CLI exit 10) | 缺 `yes:true` | 说明这是写操作,确认后带 `yes:true` 重跑 |
| 400 提示 `contents` 非法 | 用了富文本样式、图片超 9 张、正文超 20000 字符,或原样回传了读接口里带样式的正文 | 按白名单重建正文 |
| 400 提示图片地址非法 | 图片不是 `+upload-image` 产物或平台未收录 | 先上传拿地址再回帖 |
| 403 `permission_denied` | 不是该 App 的论坛管理员 | 换账号或补论坛权限 |
| 404 / 422 帖子不可见 | `moment_id` 不存在、已删除,或帖子仍在审核 | 确认帖子已公开可见再回帖 |
| 422 提示父回复不合法 | `parent_reply_id` 不是该帖的一级回复 | 用 `list-community-moment-replies` 的 `comment_id` 重试 |
