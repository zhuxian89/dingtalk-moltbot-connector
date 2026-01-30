# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

DingTalk-Moltbot-Connector bridges DingTalk (钉钉) chatbots to Moltbot/Clawdbot Gateway with AI Card streaming support. It has two implementations:

1. **TypeScript Plugin** (`plugin.ts`) - Runs as a Moltbot/Clawdbot plugin
2. **Python Connector** (`src/dingtalk_moltbot_connector/`) - Standalone process

Both implementations:
- Connect to DingTalk via Stream mode (WebSocket, no public IP needed)
- Forward messages to Gateway's `/v1/chat/completions` SSE endpoint
- Display responses using DingTalk AI Card streaming (typewriter effect)

## Build & Run Commands

### TypeScript Plugin (Moltbot/Clawdbot)
```bash
# Install plugin
clawdbot plugins install https://github.com/DingTalk-Real-AI/dingtalk-moltbot-connector.git

# Local development
clawdbot plugins install -l ./dingtalk-moltbot-connector

# Restart gateway after config changes
clawdbot gateway restart

# Verify plugin loaded
clawdbot plugins list
```

### Python Connector
```bash
# Install in development mode
pip install -e .

# Run interactively
python examples/quick_start.py

# Run via CLI
dingtalk-moltbot --gateway-url http://127.0.0.1:18789 --model default
```

## Architecture

```
DingTalk User → Stream WebSocket → Connector → HTTP SSE → Gateway /v1/chat/completions
                                      ↓
                              AI Card streaming API → DingTalk User
```

### TypeScript Plugin Flow (`plugin.ts`)
- `DWClient` from `dingtalk-stream` handles WebSocket connection
- `handleDingTalkMessage()` is the main message handler
- `streamFromGateway()` async generator consumes SSE from Gateway
- `createAICard()` / `streamAICard()` / `finishAICard()` manage AI Card lifecycle
- Session management via `userSessions` Map with timeout-based auto-renewal

### Python Connector Flow
- `MoltbotConnector` (`connector.py`) - Entry point, manages DingTalk Stream client
- `MoltbotChatbotHandler` (`handler.py`) - Processes messages, calls Gateway SSE
- `ConnectorConfig` (`config.py`) - Configuration with env var support
- Uses `dingtalk-stream` SDK's built-in `ai_markdown_card_start()` / `ai_streaming()` / `ai_finish()`

## Key Implementation Details

### AI Card Template
Both implementations use DingTalk's standard AI Card template: `382e4302-551d-4880-bf29-a30acfab2e71.schema`

### Session Management (TS Plugin)
- Sessions keyed by `dingtalk:<senderId>` or `dingtalk:<senderId>:<timestamp>`
- Default 30-minute timeout (`sessionTimeout` config)
- New session commands: `/new`, `/reset`, `/clear`, `新会话`, `重新开始`, `清空对话`

### Image Upload (TS Plugin)
- System prompt instructs LLM to output local file paths
- Post-processing (`processLocalImages()`) uploads to DingTalk and replaces paths with `media_id`
- Regex patterns match `file://`, `MEDIA:`, `/tmp/`, `/var/`, `/Users/` paths

### Gateway Authentication
Both `gatewayToken` and `gatewayPassword` are sent as `Bearer` token in Authorization header.

## Configuration

Plugin config goes in `~/.clawdbot/clawdbot.json` under `channels.dingtalk-ai`. Required fields:
- `clientId` - DingTalk AppKey
- `clientSecret` - DingTalk AppSecret

Gateway must have `http.endpoints.chatCompletions.enabled: true`.

## Dependencies

- TypeScript: `dingtalk-stream`, `axios`, `form-data`
- Python: `dingtalk-stream>=0.17.0`, `httpx>=0.24.0`
