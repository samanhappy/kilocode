# MCP Server SSE Stream Disconnection Error Analysis

## Error Message
```
SSE stream disconnected: TypeError: terminated
```

## Overview
This error occurs when a Model Context Protocol (MCP) server using Server-Sent Events (SSE) transport experiences an unexpected disconnection. The error is thrown by the underlying EventSource implementation when the stream is terminated.

## Root Causes

### 1. Network Connectivity Issues
**Description**: The most common cause is network connectivity problems between the client (Kilo Code) and the MCP server.

**Scenarios**:
- Internet connection is lost or becomes unstable
- Server becomes unreachable (DNS issues, routing problems)
- Firewall or proxy blocks the connection
- Network timeout occurs

**Code Location**: `src/services/mcp/McpHub.ts` lines 771-814

**Relevant Code**:
```typescript
// SSE connection setup
const reconnectingEventSourceOptions = {
    max_retry_time: 5000, // Maximum retry time in milliseconds
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

### 2. Server-Side Termination
**Description**: The MCP server intentionally or unintentionally terminates the SSE connection.

**Scenarios**:
- Server crashes or restarts
- Server is shut down for maintenance
- Server reaches resource limits (memory, CPU, connections)
- Server-side timeout configuration
- Server application error causes the process to exit

**Impact**: When the server terminates the connection, the EventSource throws a `TypeError: terminated` which is caught by the transport's error handler.

### 3. Authentication/Authorization Failures
**Description**: The SSE connection is rejected or terminated due to authentication issues.

**Scenarios**:
- Invalid or expired API keys/tokens
- Missing required authentication headers
- Authorization header is malformed
- Server revokes access during the session

**Code Location**: `src/services/mcp/McpHub.ts` lines 773-776

**Relevant Code**:
```typescript
const reconnectingEventSourceOptions = {
    withCredentials: configInjected.headers?.["Authorization"] ? true : false,
    // ...
}
```

### 4. Protocol-Level Issues
**Description**: Issues with the SSE protocol implementation or HTTP layer.

**Scenarios**:
- HTTP protocol errors (4xx, 5xx status codes)
- Content-Type mismatch (server doesn't send `text/event-stream`)
- Malformed SSE messages
- Connection upgrade failures
- CORS (Cross-Origin Resource Sharing) violations

### 5. Client-Side Resource Constraints
**Description**: The client application (Kilo Code) encounters resource issues.

**Scenarios**:
- VS Code extension is reloaded or disabled
- System runs out of memory
- Process is terminated by the operating system
- Browser tab/window closes (for web-based scenarios)

### 6. Reconnection-EventSource Library Limitations
**Description**: The `reconnecting-eventsource` library has limitations and edge cases.

**Library Details**:
- Package: `reconnecting-eventsource@1.6.4` (from `src/package.json`)
- Purpose: Automatically reconnects when SSE connections drop
- Configuration: `max_retry_time: 5000` (5 seconds maximum retry time)

**Known Issues**:
- The library may throw `TypeError: terminated` when the underlying EventSource is closed
- Reconnection attempts may fail if the server is permanently unavailable
- Error handling may not distinguish between temporary and permanent failures

## Error Flow

```
1. SSE Connection Established
   ↓
2. EventSource actively listening
   ↓
3. Disconnection Event (one of the root causes above)
   ↓
4. reconnecting-eventsource detects closure
   ↓
5. Throws TypeError: terminated
   ↓
6. transport.onerror handler catches the error
   ↓
7. Connection status updated to "disconnected"
   ↓
8. Error message added to server's error history
   ↓
9. Webview notified of server status change
```

## Current Error Handling

The code in `McpHub.ts` handles SSE errors through two main handlers:

### 1. Error Handler (lines 797-805)
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

**Behavior**:
- Logs the error to console
- Updates server status to "disconnected"
- Appends error message to server's error history (max 100 errors)
- Truncates error messages longer than 1000 characters
- Notifies the webview UI of the status change

### 2. Close Handler (lines 807-813)
```typescript
transport.onclose = async () => {
    const connection = this.findConnection(name, source)
    if (connection) {
        connection.server.status = "disconnected"
    }
    await this.notifyWebviewOfServerChanges()
}
```

**Behavior**:
- Updates server status to "disconnected"
- Notifies the webview UI
- Does NOT log an error (graceful closure)

## Reconnection Behavior

The `reconnecting-eventsource` library automatically attempts to reconnect when a connection is lost:

1. **Automatic Retry**: Attempts to reconnect with exponential backoff
2. **Max Retry Time**: Limited to 5000ms (5 seconds) between attempts
3. **Indefinite Retries**: Continues retrying until successful or manually stopped
4. **Credential Handling**: Includes credentials if Authorization header is present

**Note**: Even with automatic reconnection, users may see the "SSE stream disconnected: TypeError: terminated" error in the UI temporarily during reconnection attempts.

## Impact on Users

When this error occurs:

1. **Connection Status**: Server shows as "disconnected" in the UI
2. **Tool Availability**: MCP tools from this server become unavailable
3. **Error Display**: Error message is shown in the server's error history
4. **Automatic Recovery**: If the issue is temporary, the connection may automatically recover
5. **Manual Intervention**: If the issue persists, users may need to:
   - Check their network connection
   - Verify the server is running
   - Check authentication credentials
   - Manually restart the connection

## Debugging Recommendations

### For Users

1. **Check Network Connectivity**:
   - Verify internet connection is stable
   - Test if the server URL is accessible in a browser
   - Check for VPN or proxy interference

2. **Verify Server Status**:
   - Confirm the MCP server is running
   - Check server logs for errors
   - Ensure the server is not rate-limiting requests

3. **Authentication**:
   - Verify API keys/tokens are valid
   - Check if credentials have expired
   - Ensure headers are properly configured in `mcp.json`

4. **Review Configuration**:
   - Check `mcp.json` or `.kilocode/mcp.json` for correct settings
   - Verify the URL is correct and accessible
   - Ensure required headers are present

### For Developers

1. **Enable Detailed Logging**:
   - Check browser console (F12) for detailed error messages
   - Review VS Code output panel for extension logs
   - Monitor network tab for HTTP errors

2. **Test Connection Manually**:
   - Use curl or Postman to test the SSE endpoint
   - Verify server responds with `text/event-stream` content type
   - Check for CORS headers if accessing from different origin

3. **Server-Side Investigation**:
   - Review server logs for crashes or errors
   - Monitor server resource usage (CPU, memory)
   - Check for connection limits or timeout configurations

## Potential Improvements

### 1. Enhanced Error Messages
**Current**: Generic "TypeError: terminated" message
**Proposed**: Include additional context:
- Network status at time of disconnection
- Number of retry attempts made
- Last successful message timestamp
- Specific HTTP error codes if available

### 2. Connection Health Monitoring
**Proposed**:
- Add periodic health checks
- Track connection uptime/downtime statistics
- Alert users proactively about unstable connections

### 3. Graceful Degradation
**Proposed**:
- Cache recent server capabilities
- Allow read-only access to cached tools/resources during disconnection
- Queue requests for retry when connection is restored

### 4. Configurable Retry Strategy
**Current**: Fixed 5-second max retry time
**Proposed**: Allow users to configure:
- Maximum retry attempts
- Backoff strategy (exponential, linear, fixed)
- Maximum retry duration before giving up

### 5. Better Error Classification
**Proposed**: Distinguish between:
- Temporary network issues (auto-retry)
- Authentication failures (require user action)
- Server errors (notify administrators)
- Client-side issues (suggest troubleshooting steps)

## Related Files

- `src/services/mcp/McpHub.ts` - Main MCP connection management
- `src/services/mcp/kilocode/NotificationService.ts` - Handles notifications
- `apps/kilocode-docs/docs/features/mcp/server-transports.md` - SSE transport documentation
- `src/package.json` - Dependencies including `reconnecting-eventsource@1.6.4`

## References

- [Server-Sent Events Specification](https://html.spec.whatwg.org/multipage/server-sent-events.html)
- [reconnecting-eventsource Library](https://github.com/fanout/reconnecting-eventsource)
- [Model Context Protocol Documentation](https://modelcontextprotocol.io/)
- Kilo Code MCP Documentation: `/docs/features/mcp/server-transports.md`

## Conclusion

The "SSE stream disconnected: TypeError: terminated" error is a symptom of connection loss between Kilo Code and an MCP server using SSE transport. While the current implementation handles the error gracefully and attempts automatic reconnection, the root causes are diverse and may require different resolution strategies. The error handling could be enhanced with more detailed error messages, better error classification, and configurable retry strategies to improve the user experience.
