# Telegram MCP

A Model Context Protocol (MCP) server for Telegram. Search your personal messages, look up contacts, and send messages to individuals or groups — directly from Claude Desktop, Cursor, or any MCP-compatible AI client.

![Telegram MCP example usage](example-use-2.png)

It connects to your **personal Telegram account** via the Telegram API (using [Telethon](https://github.com/LonamiWebs/Telethon)). Your message history is stored locally in a SQLite database and is only sent to an LLM when the agent explicitly accesses it through the tools you control.

## Features

- 🔍 **List & search messages** — with filters and surrounding context
- 👤 **Search contacts** — find people by name or username
- 💬 **List chats** — see all available chats with metadata
- 🔗 **Resolve chats** — find a direct chat with a contact, or all chats involving them
- ⏱️ **Last interaction & message context** — get the latest message with someone, or context around a specific message
- ✉️ **Send messages** — to a username, group, or chat ID
- 🔄 **Local sync** — messages are kept up to date in a local SQLite database

## Architecture

The project has two main components:

```
telegram-bridge/          # Connects to Telegram, handles auth, stores data
├── api/                  # Telegram API client & models
├── database/             # SQLAlchemy ORM models & repositories
├── server/               # FastAPI server exposing bridge functionality
├── config.py
├── service.py             # Core service logic
└── main.py                # Entry point — run this to authenticate & sync

telegram-mcp-server/      # MCP layer exposed to the AI client
├── main.py                # MCP server entry point
└── telegram/
    ├── models.py           # Data classes for messages, chats, contacts
    ├── database.py         # DB access via SQLAlchemy
    ├── api.py               # Client for the Telegram Bridge
    └── display.py            # Formatting for message/chat output
```

- **telegram-bridge** authenticates with Telegram, keeps the connection alive, syncs messages in near real-time, and persists everything to a local SQLite database (`telegram-bridge/store/`).
- **telegram-mcp-server** implements the MCP protocol and exposes tools that query the bridge / database and format results for the AI client.

### Data flow

1. Claude sends a tool request to the MCP server
2. The MCP server queries the Telegram bridge or the SQLite database directly
3. The bridge keeps the database in sync with the Telegram API
4. Results flow back through the chain to Claude
5. When sending a message, the request flows from Claude → MCP server → bridge → Telegram

## Requirements

- Python 3.6+
- Claude Desktop app (or Cursor)

## Installation

1. **Clone the repository**

   ```bash
   git clone https://github.com/vibecoderkz/telegram-mcp.git
   cd telegram-mcp
   ```

2. **Create and activate a virtual environment**

   ```bash
   python3 -m venv myenv
   source myenv/bin/activate
   ```

3. **Install dependencies**

   ```bash
   pip install -r requirements.txt
   ```

   This single environment is used for both the bridge and the MCP server.

## Configuration

1. **Get your Telegram API credentials**

   - Go to https://my.telegram.org/auth
   - Log in → **API development tools**
   - Create a new application and note your **API ID** and **API hash**

2. **Set up the bridge's environment file**

   ```bash
   cd telegram-bridge
   cp .env.example .env
   ```

   Fill in `.env`:

   ```
   TELEGRAM_API_ID=your_api_id
   TELEGRAM_API_HASH=your_api_hash
   TELEGRAM_PHONE=your_phone_number
   ```

   (Alternatively, export `TELEGRAM_API_ID` / `TELEGRAM_API_HASH` as environment variables instead of using `.env`.)

## Running the bridge

From `telegram-bridge/`:

```bash
python main.py
```

On first run you'll be prompted for your phone number and the verification code sent to Telegram (and your 2FA password, if enabled). After that, your session is cached locally and reused.

## Connecting the MCP server

1. Update `run_telegram_server.sh` with your absolute repo path:

   ```bash
   nano run_telegram_server.sh
   # Change:
   BASE_DIR="/path/to/telegram-mcp"
   # To your actual path — get it with `pwd` from the repo root
   ```

2. Add the server to your MCP client config, using the same `BASE_DIR`:

   ```json
   {
     "mcpServers": {
       "telegram": {
         "command": "/bin/bash",
         "args": [
           "{{BASE_DIR}}/run_telegram_server.sh"
         ]
       }
     }
   }
   ```

   - **Claude Desktop** — save as `claude_desktop_config.json` in:
     `~/Library/Application Support/Claude/claude_desktop_config.json`
   - **Cursor** — save as `mcp.json` in:
     `~/.cursor/mcp.json`

3. **Restart** Claude Desktop or Cursor. Telegram should now appear as an available integration.

## MCP Tools

| Tool | Description |
|---|---|
| `search_contacts` | Search contacts by name or username |
| `list_messages` | Retrieve messages with optional filters and context |
| `list_chats` | List available chats with metadata |
| `get_chat` | Get information about a specific chat |
| `get_direct_chat_by_contact` | Find a direct chat with a specific contact |
| `get_contact_chats` | List all chats involving a specific contact |
| `get_last_interaction` | Get the most recent message with a contact |
| `get_message_context` | Retrieve context around a specific message |
| `send_message` | Send a message to a username or chat ID |

## Troubleshooting

Make sure **both** the Telegram bridge and the MCP server are running.

**Session expired** — delete `telegram-bridge/store/telegram_session.session` and restart the bridge to re-authenticate.

**Two-factor authentication** — you'll be prompted for your password during auth if 2FA is enabled.

**No messages loading** — after first authentication, syncing full history can take several minutes for accounts with many chats.

For Claude Desktop integration issues, see the [MCP documentation](https://modelcontextprotocol.io/quickstart/server#claude-for-desktop-integration-issues).

## Roadmap

- 🖼️ Rich media support (photos, videos, other media types)
- 🧵 Better message threading / conversation visualization
- ✏️ Message editing support
- 🌐 Simple web UI for configuration and monitoring
- 🔎 Expanded search (e.g. analysis over unread chats)
- 🔒 Improved security — stronger auth & encryption for local data

## Contributing

Contributions are welcome! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines before opening a PR.

## Security & Privacy

This tool operates directly on your personal Telegram account and data. Your session credentials and message data are stored locally — review `telegram-bridge/database` and `.env` handling before deploying anywhere beyond your own machine. Never commit your `.env` file or session files.

## License

See repository license terms.
