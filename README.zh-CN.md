# Wake Up Supabase

[English](./README.md) | **简体中文**

一个极简的 GitHub Actions 小工具，用于防止免费版 Supabase 项目因不活跃而被自动暂停。

Supabase 会在大约一周不活跃后暂停免费版项目。
如果项目持续暂停 90 天，将被永久删除。

Supabase 的 7 天不活跃计时器只统计**数据库活动** —— 而非 API 请求、
访问 dashboard 或认证健康检查。因此本 workflow 每天通过 PostgREST 数据接口
（`/rest/v1/<table>`）对每个项目里的一张真实表执行一次轻量查询。这条查询会
真正命中 Postgres，从而重置计时器，让你的项目保持活跃。

---

## 🚀 特性

- 可同时保活任意数量的 Supabase 项目
- 即使项目分布在多个 Supabase 账号下也能工作
- 密钥只存放在 GitHub Secrets 中（绝不写入仓库）
- 全自动 —— 每天运行一次
- 完全免费

---

## 🛡 安全性

- 只使用 anon key（绝不使用 service_role key）
- 密钥安全地存放在 GitHub Actions Secrets 中
- workflow 不会打印任何密钥
- 仓库本身不含任何敏感信息

---

## 🧩 配置步骤

### 1. Fork 本仓库
点击 GitHub 页面右上角的 Fork 按钮。

---

### 2. 把你的 Supabase 项目添加到 GitHub Secret

进入：

Settings → Secrets and variables → Actions → New repository secret

创建一个名为如下的 secret：

    SUPABASE_PROJECTS_JSON

按以下格式粘贴你的项目：

    [
      {
        "url": "https://your-project-1.supabase.co",
        "anon_key": "YOUR_PROJECT_1_ANON_PUBLIC_KEY",
        "table": "your_table_1"
      },
      {
        "url": "https://your-project-2.supabase.co",
        "anon_key": "YOUR_PROJECT_2_ANON_PUBLIC_KEY",
        "table": "your_table_2"
      }
    ]

可以添加任意数量的项目。

💡 只使用 anon key —— 绝不要用 service_role key。
anon key 可在以下位置找到：Supabase Dashboard → Project Settings → API

### `table` —— 重要

`table` 的值必须是一张 **`anon` 角色能够 `SELECT` 的真实表**。
workflow 会对它执行 `SELECT * ... LIMIT 1`，正是这条查询让项目保持活跃。

- 该表必须存在，并已通过 API 暴露。
- 它的行级安全策略（RLS）必须允许 `anon` 角色读取，否则请求会返回 `401`/`403`。
- 如果你没有合适的表，可以新建一张很小的表（例如 `keep_alive`），放入一行数据，
  并配置一条允许 `anon` 进行 `SELECT` 的 RLS 策略。

---

### 3. 启用 GitHub Actions

进入 Actions 标签页 → 如有需要，启用 workflows。

---

### 4.（可选）手动运行一次

Actions 标签页 → 选择该 workflow → Run workflow。

---

## ⏱ 工作原理

本 action 每天 06:00 UTC 运行一次。

对于 SUPABASE_PROJECTS_JSON secret 中的每个项目，它会：

1. 构造数据接口 URL
   `https://your-project.supabase.co/rest/v1/<table>?select=*&limit=1`

2. 在 `apikey` 和 `Authorization: Bearer` 两个请求头中携带 anon key
3. 执行该 `SELECT ... LIMIT 1` 查询并检查 HTTP 状态码

这条查询会真正命中 Postgres —— 而这正是 Supabase 所统计的活动，因此
不活跃计时器会被重置，项目不会被暂停。

某个项目失败不会中断整次运行：缺字段、curl 出错或返回非 2xx 状态都会被记录，
随后循环继续处理下一个项目。

> ⚠️ 本 workflow 用于**预防**暂停 —— 它无法唤醒一个已被暂停的项目。
> 如果项目已经被暂停，请先在 Supabase dashboard 中手动 Restore 恢复一次，
> 之后每天的查询就能持续让它保持活跃。

---

## 🧪 日志输出示例

    📦 Found 2 Supabase project(s).
    🌐 Querying DATABASE via: https://abc123.supabase.co/rest/v1/keep_alive
    ✅ https://abc123.supabase.co responded with HTTP 200 — database query succeeded, timer reset.
    🌐 Querying DATABASE via: https://def456.supabase.co/rest/v1/keep_alive
    ✅ https://def456.supabase.co responded with HTTP 200 — database query succeeded, timer reset.

---

## ❓ 常见问题

**为什么是查询一张表，而不是 ping `/auth/v1/health`？**
Supabase 的不活跃计时器只统计**数据库活动**。认证健康检查接口完全不会触及
Postgres，所以 ping 它并不会重置计时器，项目依然会被暂停。对一张真实表执行
`SELECT` 才是一次真正的数据库查询，因此能起作用。

**这能用于多个 Supabase 账号吗？**
可以 —— 只需把你所有项目（url、anon_key、table）都加入 secret 即可。

**这个仓库可以公开吗？**
可以 —— 所有敏感数据都保存在 GitHub Secrets 中。

**这会违反 Supabase 的规则吗？**
不会。你只是在按设计意图正常使用公开的 anon 接口。

---

## ❤️ 贡献

本项目刻意保持精简。
欢迎提交 issue 或 PR 来添加诸如以下的功能：

- Slack / Discord 提醒
- 错误通知
- JSON schema 校验
- 多区域健康检查
- 可选的日志看板

---

祝使用愉快！
@wilhelmsendk
