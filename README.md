# mcp-kit

**Universal MCP (Model Context Protocol) toolkit** for building MCP servers that work across all JavaScript runtimes.

## Features

- 🌍 **Universal Runtime Support**: Single codebase works in Node.js, Bun, Cloudflare Workers, Vercel Edge, Netlify Edge, and other JavaScript environments
- ⚡ **Hono Framework**: Ultra-fast, lightweight HTTP framework with zero cold starts
- 🔄 **Cross-Runtime Compatibility**: Automatic runtime detection and conditional feature loading
- 🧰 **Type-Safe Tool Handlers**: Strongly typed MCP tool definitions with validation
- 📦 **Multi-Target Builds**: Separate optimized builds for different runtime environments
- 🛠️ **Framework Agnostic**: No external dependencies in universal builds

### Local Runtime Features (Node.js/Bun)

- 🔀 **Multi-Instance Coordination**: Lock-based main/proxy pattern with automatic role assignment
- 🔄 **Version-Based Upgrades**: Seamless transitions when deploying new versions
- 🌐 **Reverse Proxy Gateway**: Lightweight HTTP proxy for load distribution
- 🔌 **STDIO Proxy**: Forward MCP requests over STDIO transport (Claude Desktop compatible)
- 🚪 **Port Management**: Automatic port allocation, conflict resolution, and cleanup
- ♻️ **Process Lifecycle**: Graceful shutdown handlers and signal management
- 🔒 **Stale Lock Recovery**: Automatic cleanup of crashed instance locks

## Installation

```bash
npm install github:omar391/mcp-kit
# or
pnpm add github:omar391/mcp-kit
```

## Quick Start

```typescript
import { createHonoMcpServer } from '@omar391/mcp-kit/server/core/hono-mcp';
import { createToolHandlers } from '@omar391/mcp-kit/server/handlers';

const toolHandlers = createToolHandlers([
  {
    name: 'greet',
    description: 'Greet someone by name',
    inputSchema: {
      type: 'object',
      properties: {
        name: { type: 'string' }
      },
      required: ['name']
    },
    handler: async ({ name }) => ({
      content: [{ type: 'text', text: `Hello, ${name}!` }]
    })
  }
]);

const app = createHonoMcpServer({
  serverInfo: { name: 'my-mcp-server', version: '1.0.0' },
  toolHandlers
});

// Deploy anywhere:
// - Node.js: app.listen(3000)
// - Cloudflare Workers: export default { fetch: app.fetch }
// - Vercel Edge: export default app
```

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Your MCP Server                      │
└─────────────────────────────────────────────────────────┘
                           │
                           ├─ Runtime Detection
                           │
        ┌──────────────────┴──────────────────┐
        │                                     │
        ▼                                     ▼
┌──────────────────┐              ┌────────────────────┐
│  Universal Core  │              │  Local Features    │
│  (All Runtimes)  │              │  (Node.js/Bun)     │
│                  │              │                    │
│  • Hono Server   │              │  • Instance Mgmt   │
│  • Tool Handlers │              │  • Proxy Gateway   │
│  • Middleware    │              │  • Port Manager    │
│  • Type System   │              │  • Process Mgmt    │
└──────────────────┘              └────────────────────┘
        │                                     │
        └─────────────┬───────────────────────┘
                      ▼
        ┌──────────────────────────┐
        │    Deployment Target     │
        ├──────────────────────────┤
        │ • Node.js / Bun          │
        │ • Cloudflare Workers     │
        │ • Vercel Edge            │
        │ • Netlify Edge           │
        │ • Deno                   │
        └──────────────────────────┘
```

## Runtime Compatibility

| Runtime | Universal Core | Local Features | Notes |
|---------|----------------|----------------|-------|
| Node.js | ✅ | ✅ | Full feature support |
| Bun | ✅ | ✅ | Full feature support |
| Cloudflare Workers | ✅ | ❌ | Universal only |
| Vercel Edge | ✅ | ❌ | Universal only |
| Netlify Edge | ✅ | ❌ | Universal only |
| Deno | ✅ | ❌ | Universal only |
| Browser | ✅ | ❌ | Universal only |

## Multi-Instance Coordination

**Available in Node.js/Bun environments only**

### Architecture Overview

```
                    ┌─────────────────────────┐
                    │   Lock File System      │
                    │ /tmp/mcp-kit-{port}.lock│
                    └───────────┬─────────────┘
                                │
                    First process acquires lock
                                │
        ┌───────────────────────┴───────────────────────┐
        │                                               │
        ▼                                               ▼
┌───────────────┐                            ┌──────────────────┐
│     MAIN      │                            │  PROXY INSTANCES │
│   INSTANCE    │                            │  (HTTP or STDIO) │
│               │                            │                  │
│ • Holds Lock  │◄───── Forward Requests ────┤  • No Lock       │
│ • Port: 8989  │                            │  • Auto Port     │
│ • Runs Tools  │                            │  • Transparent   │
└───────┬───────┘                            └────────▲─────────┘
        │                                              │
        │                                              │
        ├─ Control Endpoints:                          │
        │  • /__version                                │
        │  • /__shutdown                               │
        │  • /__transition                             │
        │                                              │
        └──────────────┬───────────────────────────────┘
                       │
                       ▼
              ┌────────────────┐
              │  MCP Clients   │
              │ (Claude, etc.) │
              └────────────────┘
```

### How It Works

**1. Lock-Based Election**
   - First instance creates lock file → becomes MAIN
   - Subsequent instances detect lock → become PROXY

**2. Version Management**
   - New version detects old MAIN
   - Requests graceful transition via `/__transition`
   - Old MAIN becomes proxy, new becomes MAIN
   - Seamless upgrades without downtime

**3. Stale Lock Recovery**
   - Detects crashed processes (PID check)
   - Auto-cleans stale locks
   - Promotes proxy to MAIN if needed

**4. Proxy Modes**
   - **HTTP Mode**: Reverse proxy forwards all requests
   - **STDIO Mode**: MCP client proxy for Claude Desktop integration

### Version Upgrade Flow

```
   Time ──────────────────────────────────────────────►

   ┌─────────────────┐
   │  MAIN v1.0.0    │  Running, handling requests
   │  Port: 8989     │
   └────────┬────────┘
            │
            │  New deployment starts
            │
            ▼
   ┌─────────────────┐     ┌─────────────────┐
   │  MAIN v1.0.0    │     │  NEW v2.0.0     │
   │  Port: 8989     │     │  Detects lock   │
   └────────┬────────┘     └────────┬────────┘
            │                       │
            │◄──── POST /__transition ───┤
            │                       │
            ▼                       │
   ┌─────────────────┐              │
   │  Becomes PROXY  │              │
   │  Removes lock   │              │
   └─────────────────┘              │
                                    ▼
                          ┌─────────────────┐
                          │  MAIN v2.0.0    │
                          │  Port: 8989     │
                          │  Handles reqs   │
                          └─────────────────┘
```

## API Reference

### Core Functions

| Function | Runtime | Description |
|----------|---------|-------------|
| `createHonoMcpServer(options)` | Universal | Create MCP server on Hono framework |
| `createToolHandlers(tools)` | Universal | Type-safe tool handler definitions |
| `startMcpServer(config)` | Node.js/Bun | High-level starter with coordination |
| `detectRuntime()` | Universal | Get runtime information |
| `isNodeLike()` | Universal | Check if Node.js or Bun |

### Local Features (Node.js/Bun)

| Component | Purpose |
|-----------|---------|
| `InstanceManager` | Multi-instance coordination with locks |
| `ProxyManager` | HTTP reverse proxy gateway |
| `coordinateInstanceRole()` | Automatic main/proxy election |
| `startStdioProxy()` | STDIO transport proxy |
| `ensurePortAvailable()` | Port conflict resolution |
| `registerSignalHandlers()` | Graceful shutdown handling |

See inline documentation and TypeScript types for detailed API usage.


## Usage Examples

### Basic Server
```typescript
import { startMcpServer, createToolHandlers } from '@omar391/mcp-kit/server';

await startMcpServer({
  serverName: 'my-server',
  serverVersion: '1.0.0',
  toolHandlers: createToolHandlers([/* tools */]),
  defaultPort: 8989
});
```

### With Lifecycle Hooks
```typescript
await startMcpServer({
  serverName: 'my-server',
  serverVersion: '1.0.0',
  toolHandlers,
  localMode: {
    onLocalStart: async (instanceManager) => {
      console.log(`Main instance on port ${instanceManager.port}`);
    },
    onShutdown: async (instanceManager) => {
      await instanceManager.removeLock();
    }
  }
});
```

### Edge Runtime (Universal)
```typescript
import { createHonoMcpServer } from '@omar391/mcp-kit/server/core/hono-mcp';

const app = createHonoMcpServer({ serverInfo, toolHandlers });

// Cloudflare Workers
export default { fetch: app.fetch };

// Vercel Edge
export const config = { runtime: 'edge' };
export default app;
```

### STDIO Mode (Claude Desktop)
```bash
node server.js --mode=stdio
```

## Build Targets

The package provides multiple build targets for optimal deployment:

- **`universal`**: Works everywhere, no Node.js APIs (default)
- **`node`**: Includes Node.js-specific features and APIs
- **`browser`**: Minimal build for browser environments

Use conditional exports to automatically get the right build:

```json
{
  "imports": {
    "@omar391/mcp-kit": {
      "node": "@omar391/mcp-kit/dist/node/index.js",
      "default": "@omar391/mcp-kit/dist/index.js"
    }
  }
}
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Port already in use | Run with `--kill-existing` flag or `lsof -i :PORT` to check |
| Stale lock file | Auto-cleaned if process dead, or manual: `rm /tmp/mcp-kit-{port}.lock` |
| Version conflict | New version auto-requests transition via `/__transition` endpoint |
| Proxy not connecting | Verify main instance running and no firewall blocking localhost |
| Lock permissions | Ensure `/tmp` is writable and process has file permissions |

## License

MIT

## Related

- [MCP Specification](https://modelcontextprotocol.io)
- [Hono Framework](https://hono.dev)
- [@modelcontextprotocol/sdk](https://github.com/modelcontextprotocol/sdk)
