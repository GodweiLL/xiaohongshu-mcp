# 多用户隔离功能实现计划

## 1. 需求概述

### 背景
将 xiaohongshu-mcp 部署到 Linux 服务器，供 Agent 系统调用。需要支持多个用户同时使用，每个用户有独立的登录状态。

### 核心需求
- Agent 通过 `user_id` (UUID) 标识不同用户
- 每个用户有独立的 Cookie（登录状态）
- 支持并发：不同用户可同时操作
- 未登录时自动返回登录二维码

### 设计决策
| 决策项 | 选择 | 说明 |
|--------|------|------|
| Cookie 存储 | 文件系统 | `cookies/{user_id}.json` |
| 同用户并发 | 并行执行 | 允许同一用户同时发多篇内容 |
| 未登录处理 | 返回二维码 | 自动触发登录流程 |

---

## 2. 架构设计

### 2.1 当前架构（单用户）

```
Request → Service → Browser(固定Cookie) → 小红书
```

### 2.2 目标架构（多用户）

```
Request(user_id=xxx)
    ↓
Service.Method(ctx, userID, ...)
    ↓
CookieManager.GetPath(userID) → cookies/{userID}.json
    ↓
Browser(用户专属Cookie) → 小红书
```

### 2.3 Cookie 存储结构

```
cookies/
├── .gitkeep
├── a1b2c3d4-e5f6-7890-abcd-ef1234567890.json
├── b2c3d4e5-f6a7-8901-bcde-f12345678901.json
└── ...
```

---

## 3. 改动清单

### 3.1 Cookie 模块 (`cookies/cookies.go`)

**改动内容**：
- 新增 `GetUserCookiePath(userID string) string` 函数
- 新增 `EnsureCookieDir()` 确保目录存在

**代码示意**：
```go
func GetUserCookiePath(userID string) string {
    return filepath.Join("cookies", userID+".json")
}
```

### 3.2 Browser 模块 (`browser/browser.go`)

**改动内容**：
- `NewBrowser` 增加 `WithUserID(userID string)` Option
- 根据 userID 加载对应的 Cookie 文件

**代码示意**：
```go
func WithUserID(userID string) Option {
    return func(c *browserConfig) {
        c.userID = userID
    }
}
```

### 3.3 Service 层 (`service.go`)

**改动内容**：
- 所有公开方法增加 `userID string` 参数
- `newBrowser()` 改为 `newBrowserForUser(userID string)`
- `saveCookies` 改为 `saveCookiesForUser(userID, page)`

**受影响的方法**：
| 方法 | 改动 |
|------|------|
| `CheckLoginStatus` | 增加 userID 参数 |
| `GetLoginQrcode` | 增加 userID 参数 |
| `DeleteCookies` | 增加 userID 参数 |
| `PublishContent` | 增加 userID 参数 |
| `PublishVideo` | 增加 userID 参数 |
| `ListFeeds` | 增加 userID 参数 |
| `SearchFeeds` | 增加 userID 参数 |
| `GetFeedDetail` | 增加 userID 参数 |
| `UserProfile` | 增加 userID 参数 |
| `PostCommentToFeed` | 增加 userID 参数 |
| `LikeFeed` | 增加 userID 参数 |
| `UnlikeFeed` | 增加 userID 参数 |
| `FavoriteFeed` | 增加 userID 参数 |
| `UnfavoriteFeed` | 增加 userID 参数 |
| `ReplyCommentToFeed` | 增加 userID 参数 |
| `GetMyProfile` | 增加 userID 参数 |

### 3.4 HTTP API (`handlers_api.go`, `routes.go`)

**改动内容**：
- 所有请求增加 `user_id` 字段（Header 或 Body）
- 提取并传递 userID 到 Service 层

**请求示例**：
```json
{
  "user_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "title": "标题",
  "content": "内容",
  "images": ["..."]
}
```

### 3.5 MCP Handler (`mcp_handlers.go`)

**改动内容**：
- 所有 Tool 参数增加 `UserID` 字段
- 调用 Service 时传递 userID

### 3.6 类型定义 (`types.go`)

**改动内容**：
- 所有 Request 结构体增加 `UserID string` 字段

---

## 4. 实现步骤

### Phase 1: Cookie 模块改造
1. 修改 `cookies/cookies.go`，支持多用户路径
2. 创建 `cookies/` 目录

### Phase 2: Browser 模块改造
3. 修改 `browser/browser.go`，支持按用户加载 Cookie

### Phase 3: Service 层改造
4. 修改 `service.go`，所有方法增加 userID 参数
5. 修改辅助函数 `newBrowser`、`saveCookies`

### Phase 4: API 层改造
6. 修改 `types.go`，Request 增加 UserID 字段
7. 修改 `handlers_api.go`，提取并传递 userID
8. 修改 `mcp_handlers.go`，Tool 参数增加 UserID

### Phase 5: 测试验证
9. 本地测试多用户场景
10. 格式化代码

---

## 5. 向后兼容考虑

### 5.1 默认用户
如果请求未提供 `user_id`，使用默认值 `"default"`，保持与旧版行为一致。

```go
func getUserID(userID string) string {
    if userID == "" {
        return "default"
    }
    return userID
}
```

### 5.2 旧 Cookie 迁移
启动时检查旧的 `cookies.json`，自动迁移到 `cookies/default.json`。

---

## 6. 风险与注意事项

| 风险 | 应对措施 |
|------|----------|
| 磁盘空间 | 定期清理过期 Cookie 文件 |
| 文件冲突 | UUID 保证唯一性 |
| 二维码超时 | 保持现有 4 分钟超时逻辑 |
| 并发写文件 | Cookie 文件按用户隔离，不存在冲突 |

---

## 7. 后续优化（可选）

- [ ] Cookie 过期自动清理
- [ ] 用户操作频率限制
- [ ] 登录状态缓存（避免每次都检查）
- [ ] 支持 Redis 存储（集群部署）
