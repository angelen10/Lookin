# Bugfix: MCP Server 多次连接失败问题

## 问题描述

在使用 `claude mcp list` 命令检查 Lookin MCP Server 健康状态时，第一次连接成功，但第二次及后续连接都会失败，显示 `✗ Failed to connect`。

### 现象

```bash
# 第一次执行
$ claude mcp list
lookin: http://127.0.0.1:47199/mcp (HTTP) - ✓ Connected

# 第二次执行
$ claude mcp list
lookin: http://127.0.0.1:47199/mcp (HTTP) - ✗ Failed to connect
```

## 根本原因

### 问题分析

通过日志分析发现：

**第一次连接的请求/响应**：
1. POST `/mcp` - `initialize` 请求 → 返回 200 成功
   ```json
   {"id":0,"jsonrpc":"2.0","result":{"capabilities":{"tools":{"listChanged":false}},"protocolVersion":"2025-11-25","serverInfo":{"name":"lookin-mcp-server","version":"1.0.0"}}}
   ```
2. POST `/mcp` - `notifications/initialized` 通知 → 返回 202
3. GET `/mcp` - 尝试建立 SSE 连接 → 返回 405 Method Not Allowed

**第二次连接的请求/响应**：
1. POST `/mcp` - `initialize` 请求 → 返回 200 但包含错误
   ```json
   {"error":{"code":-32600,"data":{"detail":"Server is already initialized"},"message":"Invalid Request: Server is already initialized"},"id":0,"jsonrpc":"2.0"}
   ```
2. 客户端收到错误后立即关闭连接

### 核心问题

原始实现中，在 `LookinMCPServer.start()` 方法中创建了**单例的** `Server` 和 `StatelessHTTPServerTransport` 实例：

```swift
// 错误的实现
public func start() async throws {
    // 创建一次 Transport 和 Server
    transport = StatelessHTTPServerTransport(logger: logger)
    server = Server(name: "lookin-mcp-server", version: "1.0.0", ...)

    // 注册 Tools
    await LookinMCPToolHandler.registerTools(on: server, dataSource: dataSource)

    // 启动 MCP Server
    try await server.start(transport: transport)

    // 启动 HTTP Server...
}
```

这导致：
1. 所有 HTTP 连接共享同一个 `Server` 实例
2. MCP `Server` 在第一次 `initialize` 后进入已初始化状态
3. 第二次连接尝试 `initialize` 时，`Server` 拒绝请求并返回错误
4. 客户端认为连接失败

### MCP 协议特性

- MCP Server 是有状态的，一旦初始化就不能再次初始化
- `StatelessHTTPServerTransport` 虽然名为"无状态"，但底层的 `Server` 本身是有状态的
- 每个 HTTP 连接应该被视为独立的会话，需要独立的 `Server` 实例

## 解决方案

### 实现方式

为每个 HTTP 请求创建独立的 `Server` 和 `Transport` 实例，确保每次连接都能正常初始化。

**修改前**：
```swift
// 在 start() 中创建单例
private var transport: StatelessHTTPServerTransport?
private var server: Server?

public func start() async throws {
    transport = StatelessHTTPServerTransport(logger: logger)
    server = Server(...)
    try await server.start(transport: transport)
    // ...
}

func handleHTTPRequest(_ request: HTTPRequest) async -> HTTPResponse {
    guard let transport = transport else { ... }
    return await transport.handleRequest(request)
}
```

**修改后**：
```swift
// 移除单例变量
// private var transport: StatelessHTTPServerTransport? ❌
// private var server: Server? ❌

public func start() async throws {
    // 只启动 HTTP Server，不创建 MCP Server
    let group = MultiThreadedEventLoopGroup(numberOfThreads: System.coreCount)
    // ...
}

func handleHTTPRequest(_ request: HTTPRequest) async -> HTTPResponse {
    // 为每个请求创建独立的 Server 和 Transport
    let transport = StatelessHTTPServerTransport(logger: logger)
    let server = Server(
        name: "lookin-mcp-server",
        version: "1.0.0",
        capabilities: Server.Capabilities(
            tools: .init(listChanged: false)
        )
    )

    // 注册 Tools
    await LookinMCPToolHandler.registerTools(on: server, dataSource: dataSource)

    // 启动 MCP Server
    try await server.start(transport: transport)

    // 处理请求
    let response = await transport.handleRequest(request)

    // 停止 Server
    await server.stop()

    return response
}
```

### 关键改动

1. **移除单例变量**：删除 `transport` 和 `server` 实例变量
2. **按需创建**：在 `handleHTTPRequest` 中为每个请求创建新的实例
3. **生命周期管理**：请求处理完成后调用 `server.stop()` 清理资源

## 技术细节

### 为什么不使用 StatefulHTTPServerTransport？

`StatefulHTTPServerTransport` 支持会话管理和 SSE streaming，但：
- 需要客户端支持 SSE（Server-Sent Events）
- `claude mcp list` 使用简单的 HTTP 请求，不支持 SSE
- 使用 `StatelessHTTPServerTransport` 更适合这种场景

### 性能考虑

每次请求都创建新的 `Server` 实例会有一定开销，但：
- MCP Server 初始化很快（主要是注册 tools）
- 健康检查请求频率不高
- 相比连接失败的问题，这个开销是可以接受的

### 其他尝试过的方案

1. ❌ 修改 HTTP pipeline 配置（`withPipeliningAssistance`、`maxMessagesPerRead`）
2. ❌ 添加 `Content-Length` 头
3. ❌ 保持 `EventLoopGroup` 引用
4. ✅ 为每个连接创建独立的 Server 实例

## 验证

修复后，多次执行 `claude mcp list` 都能成功连接：

```bash
$ claude mcp list
lookin: http://127.0.0.1:47199/mcp (HTTP) - ✓ Connected

$ claude mcp list
lookin: http://127.0.0.1:47199/mcp (HTTP) - ✓ Connected

$ claude mcp list
lookin: http://127.0.0.1:47199/mcp (HTTP) - ✓ Connected
```

使用 curl 测试也能正常工作：

```bash
$ curl http://127.0.0.1:47199/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":"test-1"}'
# 返回完整的 tools 列表
```

## 相关文件

- `LookinMCP/Sources/LookinMCP/LookinMCPServer.swift` - 主要修改
- `LookinClient/MCP/MCPServerManager.swift` - 清理调试日志

## 日期

2026-03-23
