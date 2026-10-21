# mcp-crawl4ai-bridge

A small stdio ↔ SSE bridge that exposes a remote [Crawl4AI](https://github.com/unclecode/crawl4ai) MCP server to MCP clients that only speak stdio (opencode, Claude Code, Gemini CLI, etc.).

```
MCP client ──stdio──> mcp-crawl4ai-bridge ──SSE/HTTP──> Crawl4AI /mcp/sse
```

## Why this exists

Crawl4AI mounts its MCP server on the SSE transport (`/mcp/sse` + `/mcp/messages/`). Many desktop MCP clients either only support local stdio servers or have incomplete SSE support. This bridge is a transparent stdio-to-SSE adapter.

It also fixes a subtle failure mode of naive passthrough bridges:

1. MCP over SSE creates a **new server-side session on every SSE reconnect**.
2. The MCP handshake (`initialize` + `notifications/initialized`) is **per session**.
3. A passthrough bridge initializes once. After any reconnect it keeps sending `tools/call` into a fresh, uninitialized session.
4. Crawl4AI then answers `-32602 Invalid request parameters` to every request, including `resources/list`, until the client is restarted.

This bridge caches the handshake and replays it on every new SSE session, so reconnects are transparent.

## Features

- MCP stdio on one side, remote Crawl4AI SSE on the other
- replays `initialize` / `initialized` after every reconnect
- filters the internal re-initialization response so the client never sees it
- retries a request once on HTTP 400/404 by reconnecting
- logs session numbers and reconnects to stderr
- no third-party Python dependencies

## Requirements

- Python 3.8+
- A reachable Crawl4AI server with MCP enabled (tested with Crawl4AI 0.9.2)

## Install

```sh
install -m 755 mcp-crawl4ai-bridge ~/.local/bin/
```

Make sure `~/.local/bin` is in `PATH`, or use the absolute path in the client config.

## Configuration

The bridge is configured with environment variables:

| Variable | Default | Description |
|---|---|---|
| `CRAWL4AI_URL` | `http://127.0.0.1:11235` | Base URL of the Crawl4AI server |
| `CRAWL4AI_TOKEN` | *(empty)* | Bearer token; sent only when non-empty |
| `CRAWL4AI_SSE_PATH` | `/mcp/sse` | SSE endpoint path |

### opencode

`~/.config/opencode/opencode.jsonc`:

```jsonc
{
  "mcp": {
    "crawl4ai": {
      "type": "local",
      "command": ["mcp-crawl4ai-bridge"],
      "enabled": true,
      "environment": {
        "CRAWL4AI_URL": "http://crawl4ai.example:11235",
        "CRAWL4AI_TOKEN": "your-token"
      }
    }
  }
}
```

If the bridge is not in `PATH`, use the absolute path:

```jsonc
"command": ["/home/you/.local/bin/mcp-crawl4ai-bridge"]
```

### Claude Code / Gemini CLI / other clients

Most clients accept the classic `mcpServers` shape:

```json
{
  "mcpServers": {
    "crawl4ai": {
      "command": "/home/you/.local/bin/mcp-crawl4ai-bridge",
      "env": {
        "CRAWL4AI_URL": "http://crawl4ai.example:11235",
        "CRAWL4AI_TOKEN": "your-token"
      }
    }
  }
}
```

## How it works

1. Opens `GET $CRAWL4AI_URL$CRAWL4AI_SSE_PATH` and waits for the `endpoint` event.
2. Every JSON-RPC line from stdin is POSTed to the session URL from that event.
3. JSON-RPC messages received over SSE are written to stdout, one per line.
4. The first `initialize` and `notifications/initialized` messages are cached.
5. When a new session appears, the bridge sends the cached handshake with the synthetic id `bridge-reinit`, drops its response, and then forwards the pending request.
6. HTTP 400/404 means the session is gone; the bridge closes the stream, reconnects, and retries once.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `-32602 Invalid request parameters` on every call after the server has been running for a while | The client is talking to an uninitialized session. Use this bridge (or restart the client if you are running an older version). |
| `Timeout waiting for SSE endpoint` | Wrong `CRAWL4AI_URL`/`CRAWL4AI_SSE_PATH`, missing token, or the Crawl4AI MCP server is not running. |
| `SSE connection error: ... Retrying in 2s` | Network hiccup or server restart. The bridge reconnects automatically and re-initializes the new session. |

Bridge logs go to stderr; most MCP clients expose them in their MCP/server logs.

## Limitations

- Requests are forwarded one line at a time; it is not a multiplexing proxy.
- If the SSE connection drops in the middle of a request, that response can be lost. The client should retry.
- Only bearer-token auth is supported (no client certificates or custom headers).

## License

No license has been chosen yet. Add one before redistributing.
