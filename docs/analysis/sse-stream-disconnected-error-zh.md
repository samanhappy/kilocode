# MCP 服务器 SSE 流断开错误分析

## 错误信息
```
SSE stream disconnected: TypeError: terminated
```

## 概述
当使用服务器发送事件（SSE）传输的模型上下文协议（MCP）服务器出现意外断开连接时，会发生此错误。当流被终止时，底层 EventSource 实现会抛出此错误。

## 根本原因

### 1. 网络连接问题
**描述**: 最常见的原因是客户端（Kilo Code）和 MCP 服务器之间的网络连接问题。

**场景**:
- 互联网连接丢失或不稳定
- 服务器无法访问（DNS 问题、路由问题）
- 防火墙或代理阻止连接
- 网络超时

**代码位置**: `src/services/mcp/McpHub.ts` 第 771-814 行

**相关代码**:
```typescript
// SSE 连接设置
const reconnectingEventSourceOptions = {
    max_retry_time: 5000, // 最大重试时间（毫秒）
    withCredentials: configInjected.headers?.["Authorization"] ? true : false,
    fetch: (url: string | URL, init: RequestInit) => {
        const headers = new Headers({ ...(init?.headers || {}), ...(configInjected.headers || {}) })
        return fetch(url, {
            ...init,
            headers,
        })
    },
}
```

### 2. 服务器端终止
**描述**: MCP 服务器有意或无意地终止 SSE 连接。

**场景**:
- 服务器崩溃或重启
- 服务器关闭进行维护
- 服务器达到资源限制（内存、CPU、连接数）
- 服务器端超时配置
- 服务器应用程序错误导致进程退出

**影响**: 当服务器终止连接时，EventSource 会抛出 `TypeError: terminated`，该错误会被传输层的错误处理程序捕获。

### 3. 身份验证/授权失败
**描述**: SSE 连接因身份验证问题被拒绝或终止。

**场景**:
- 无效或过期的 API 密钥/令牌
- 缺少必需的身份验证标头
- 授权标头格式错误
- 服务器在会话期间撤销访问权限

**代码位置**: `src/services/mcp/McpHub.ts` 第 773-776 行

**相关代码**:
```typescript
const reconnectingEventSourceOptions = {
    withCredentials: configInjected.headers?.["Authorization"] ? true : false,
    // ...
}
```

### 4. 协议级问题
**描述**: SSE 协议实现或 HTTP 层的问题。

**场景**:
- HTTP 协议错误（4xx、5xx 状态码）
- Content-Type 不匹配（服务器未发送 `text/event-stream`）
- SSE 消息格式错误
- 连接升级失败
- CORS（跨源资源共享）违规

### 5. 客户端资源约束
**描述**: 客户端应用程序（Kilo Code）遇到资源问题。

**场景**:
- VS Code 扩展被重新加载或禁用
- 系统内存不足
- 进程被操作系统终止
- 浏览器标签/窗口关闭（针对基于 Web 的场景）

### 6. Reconnection-EventSource 库限制
**描述**: `reconnecting-eventsource` 库存在限制和边缘情况。

**库详情**:
- 包: `reconnecting-eventsource@1.6.4`（来自 `src/package.json`）
- 用途: 当 SSE 连接断开时自动重新连接
- 配置: `max_retry_time: 5000`（最大重试时间 5 秒）

**已知问题**:
- 当底层 EventSource 关闭时，该库可能会抛出 `TypeError: terminated`
- 如果服务器永久不可用，重新连接尝试可能会失败
- 错误处理可能无法区分临时故障和永久故障

## 错误流程

```
1. 建立 SSE 连接
   ↓
2. EventSource 主动监听
   ↓
3. 断开连接事件（上述根本原因之一）
   ↓
4. reconnecting-eventsource 检测到关闭
   ↓
5. 抛出 TypeError: terminated
   ↓
6. transport.onerror 处理程序捕获错误
   ↓
7. 连接状态更新为 "disconnected"
   ↓
8. 错误消息添加到服务器的错误历史记录
   ↓
9. 通知 Webview 服务器状态变化
```

## 当前错误处理

`McpHub.ts` 中的代码通过两个主要处理程序处理 SSE 错误：

### 1. 错误处理程序（第 797-805 行）
```typescript
transport.onerror = async (error) => {
    console.error(`Transport error for "${name}":`, error)
    const connection = this.findConnection(name, source)
    if (connection) {
        connection.server.status = "disconnected"
        this.appendErrorMessage(connection, error instanceof Error ? error.message : `${error}`)
    }
    await this.notifyWebviewOfServerChanges()
}
```

**行为**:
- 将错误记录到控制台
- 将服务器状态更新为 "disconnected"
- 将错误消息附加到服务器的错误历史记录（最多 100 个错误）
- 截断长度超过 1000 个字符的错误消息
- 通知 webview UI 状态变化

### 2. 关闭处理程序（第 807-813 行）
```typescript
transport.onclose = async () => {
    const connection = this.findConnection(name, source)
    if (connection) {
        connection.server.status = "disconnected"
    }
    await this.notifyWebviewOfServerChanges()
}
```

**行为**:
- 将服务器状态更新为 "disconnected"
- 通知 webview UI
- 不记录错误（正常关闭）

## 重新连接行为

`reconnecting-eventsource` 库在连接丢失时会自动尝试重新连接：

1. **自动重试**: 使用指数退避尝试重新连接
2. **最大重试时间**: 尝试之间的间隔限制为 5000 毫秒（5 秒）
3. **无限重试**: 持续重试直到成功或手动停止
4. **凭据处理**: 如果存在 Authorization 标头，则包含凭据

**注意**: 即使启用了自动重新连接，用户在重新连接尝试期间仍可能暂时看到"SSE stream disconnected: TypeError: terminated"错误。

## 对用户的影响

当此错误发生时：

1. **连接状态**: 服务器在 UI 中显示为"已断开连接"
2. **工具可用性**: 来自此服务器的 MCP 工具变得不可用
3. **错误显示**: 错误消息显示在服务器的错误历史记录中
4. **自动恢复**: 如果问题是暂时的，连接可能会自动恢复
5. **手动干预**: 如果问题持续存在，用户可能需要：
   - 检查网络连接
   - 验证服务器正在运行
   - 检查身份验证凭据
   - 手动重启连接

## 调试建议

### 对于用户

1. **检查网络连接**:
   - 验证互联网连接是否稳定
   - 测试服务器 URL 在浏览器中是否可访问
   - 检查 VPN 或代理干扰

2. **验证服务器状态**:
   - 确认 MCP 服务器正在运行
   - 检查服务器日志中的错误
   - 确保服务器没有限制请求速率

3. **身份验证**:
   - 验证 API 密钥/令牌是否有效
   - 检查凭据是否已过期
   - 确保在 `mcp.json` 中正确配置了标头

4. **查看配置**:
   - 检查 `mcp.json` 或 `.kilocode/mcp.json` 的正确设置
   - 验证 URL 是否正确且可访问
   - 确保存在必需的标头

### 对于开发人员

1. **启用详细日志记录**:
   - 检查浏览器控制台（F12）获取详细错误消息
   - 查看 VS Code 输出面板获取扩展日志
   - 监控网络选项卡获取 HTTP 错误

2. **手动测试连接**:
   - 使用 curl 或 Postman 测试 SSE 端点
   - 验证服务器响应 `text/event-stream` 内容类型
   - 检查来自不同来源访问时的 CORS 标头

3. **服务器端调查**:
   - 查看服务器日志中的崩溃或错误
   - 监控服务器资源使用情况（CPU、内存）
   - 检查连接限制或超时配置

## 潜在改进

### 1. 增强错误消息
**当前**: 通用的"TypeError: terminated"消息
**建议**: 包含额外上下文：
- 断开连接时的网络状态
- 进行的重试尝试次数
- 最后成功消息的时间戳
- 如果可用，包含特定的 HTTP 错误代码

### 2. 连接健康监控
**建议**:
- 添加定期健康检查
- 跟踪连接正常运行时间/停机时间统计
- 主动向用户发出不稳定连接的警报

### 3. 优雅降级
**建议**:
- 缓存最近的服务器功能
- 在断开连接期间允许只读访问缓存的工具/资源
- 在连接恢复时排队重试请求

### 4. 可配置的重试策略
**当前**: 固定的 5 秒最大重试时间
**建议**: 允许用户配置：
- 最大重试尝试次数
- 退避策略（指数、线性、固定）
- 放弃前的最大重试持续时间

### 5. 更好的错误分类
**建议**: 区分：
- 临时网络问题（自动重试）
- 身份验证失败（需要用户操作）
- 服务器错误（通知管理员）
- 客户端问题（建议故障排除步骤）

## 相关文件

- `src/services/mcp/McpHub.ts` - 主要的 MCP 连接管理
- `src/services/mcp/kilocode/NotificationService.ts` - 处理通知
- `apps/kilocode-docs/docs/features/mcp/server-transports.md` - SSE 传输文档
- `src/package.json` - 依赖项，包括 `reconnecting-eventsource@1.6.4`

## 参考资料

- [服务器发送事件规范](https://html.spec.whatwg.org/multipage/server-sent-events.html)
- [reconnecting-eventsource 库](https://github.com/fanout/reconnecting-eventsource)
- [模型上下文协议文档](https://modelcontextprotocol.io/)
- Kilo Code MCP 文档: `/docs/features/mcp/server-transports.md`

## 结论

"SSE stream disconnected: TypeError: terminated" 错误是 Kilo Code 与使用 SSE 传输的 MCP 服务器之间连接丢失的症状。虽然当前实现能够优雅地处理错误并尝试自动重新连接，但根本原因多种多样，可能需要不同的解决策略。错误处理可以通过更详细的错误消息、更好的错误分类和可配置的重试策略来增强，以改善用户体验。
