# Nuxt 4 Supabase Drizzle Security Review 中文说明

## 用途

这个 skill 用于审查 Nuxt 4 + Supabase + Drizzle 项目的安全边界，重点是密钥暴露、前端直连业务表、服务端鉴权、RLS 和 migration 风险。

## 审查重点

- `NUXT_DATABASE_URL` 不能进入前端。
- `NUXT_SUPABASE_SERVICE_ROLE_KEY` 不能进入前端。
- `runtimeConfig.public` 只能放公开 Supabase URL/key。
- 受保护业务数据应通过 Nitro/server services 访问。
- 权限判断不能只依赖浏览器状态。
- 涉及 Supabase exposed schema 时需要关注 RLS。
- migration 里创建 view/function 时需要额外检查安全属性。

## database 放在根目录的原因

审查目录结构时，应把根目录 `database/` 视为推荐边界：

1. Drizzle Kit 不运行在 Nuxt runtime 中，不能直接识别 Nuxt `~` 路径映射。schema 放进 `server/` 后，如果引用依赖 Nuxt alias，生成 migration 可能失败。
2. Nuxt layer 场景下可能存在多个 `server/` 目录。数据库如果挂在某个 `server/` 下，跨 layer 引用会让路径和所有权不明确。

这不是单纯代码风格问题，而是为了减少工具链报错、路径映射补丁和跨 layer 引用歧义。

## 与 Supabase 官方 skills 的关系

当涉及 Supabase 当前产品能力、RLS 细节、CLI 或安全规则时，应优先使用：

- `supabase`
- `supabase-postgres-best-practices`

本 skill 只负责 Nuxt 4 + Drizzle 项目中的审查流程和边界提醒。

## 不应写入的项目专用内容

- 当前项目的错误码。
- 当前项目的 webhook 事件类型。
- 当前项目的管理员字段名。
- 当前项目的业务权限模型。

## 后续优化方向

- 可以积累真实项目中的密钥暴露案例。
- 可以补充 RLS 审查清单。
- 可以补充 server-only 文件、client plugin、public runtime config 的误用案例。
