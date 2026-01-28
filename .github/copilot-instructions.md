# Yukki Music Bot - AI Agent Instructions

This is a Telegram music streaming bot built with Pyrogram and py-tgcalls. Focus on async/await patterns, plugin architecture, and multi-source music streaming.

## Architecture Overview

**Core Structure**: Bot initialization in `YukkiMusic/__init__.py` creates:
- `app` (YukkiBot) - Main bot client extending Pyrogram Client
- `userbot` (Userbot) - Assistant account for voice calls
- Platform APIs (YouTube, Spotify, Apple Music, SoundCloud, etc.)
- `Yukki` - Call manager (pytgcalls) for streaming audio/video to groups

**Plugin System**: Auto-discovery via glob patterns in `YukkiMusic/plugins/__init__.py`:
```python
ALL_MODULES = sorted(__list_all_modules())  # Discovers all .py files recursively
```
Plugins organized by category: `admins/`, `play/`, `sudo/`, `bot/`, `tools/`, `misc/`, `devs/`

**Database Layer**: Three abstraction levels in `YukkiMusic/utils/database/`:
- `mongodatabase.py` - MongoDB persistence (MongoDB Atlas via MONGO_DB_URI)
- `memorydatabase.py` - In-memory cache layer
- `assistantdatabase.py` - Separate assistant-specific data

## Critical Data Flows

1. **Music Playback**: `play/play.py` → platform APIs (`YouTube`, `Spotify`, etc.) → `stream.stream()` → `Yukki.stream_call()`
2. **Multi-platform Support**: Each platform in `platforms/` returns consistent format (URL, duration, metadata)
3. **Language System**: YAML-based in `strings/langs/` (en.yml, ar.yml, etc.), accessed via `get_string()` decorator
4. **Decorators**: Custom permission/state decorators in `utils/decorators/`:
   - `@PlayWrapper` - Validates play environment and user permissions
   - `@LanguageStart` - Injects user language into message handlers

## Configuration & Environment

**Mandatory Vars** (`config/config.py`):
- `API_ID`, `API_HASH`, `BOT_TOKEN`, `MONGO_DB_URI`
- `LOG_GROUP_ID`, `MUSIC_BOT_NAME`, `OWNER_ID`, `STRING_SESSION`

**Key Settings**:
- `DURATION_LIMIT_MIN` - Max song duration (default 60 mins)
- `SONG_DOWNLOAD_DURATION` - Download limit (default 180 mins)
- `BANNED_USERS` - Global user blacklist loaded from MongoDB
- `SUDOERS` - Admin users with elevated permissions

## Development Patterns

### Adding New Commands
Create handler in appropriate category under `plugins/`:
```python
from pyrogram import filters
from YukkiMusic import app
from strings import get_command, get_string

COMMAND = get_command("COMMAND_NAME")  # Define in strings/command.yml

@app.on_message(
    filters.command(COMMAND)
    & filters.group
    & ~filters.edited
    & ~BANNED_USERS
)
async def command_handler(client, message):
    pass
```

### Accessing Music Queue
Import from `utils/stream/queue.py`:
```python
from YukkiMusic.utils.stream.queue import get_queue, put_queue
queue = await get_queue(chat_id)  # List[str] of video_ids
```

### Database Operations
Use MongoDB helpers from `utils/database/mongodatabase.py`:
```python
from YukkiMusic.utils.database import add_served_chat, get_banned_users
await add_served_chat(chat_id)
banned = await get_banned_users()  # Returns list of user IDs
```

### Multi-language Support
Handler receives language parameter from `@LanguageStart` or `@languageCB` decorators:
```python
_("SONG_PLAYING_STRING").format(title=name)  # Returns localized string
```

### Platform-Agnostic Queries
Support for searches across platforms:
- YouTube: `YouTubeAPI().search()` (via youtube-search-python)
- Spotify: `SpotifyAPI().search()` (requires SPOTIFY_CLIENT_ID/SECRET)
- Direct URL formats: Telegram files, YouTube links, Spotify/Apple/SoundCloud URIs

## Key Files & Imports

| File | Purpose |
|------|---------|
| [YukkiMusic/core/bot.py](YukkiMusic/core/bot.py) | Bot initialization, command registration |
| [YukkiMusic/core/call.py](YukkiMusic/core/call.py) | Call manager (Yukki instance) |
| [YukkiMusic/utils/stream/stream.py](YukkiMusic/utils/stream/stream.py) | Audio/video streaming logic |
| [YukkiMusic/plugins/play/play.py](YukkiMusic/plugins/play/play.py) | Primary play command (large, handles all sources) |
| [config/config.py](config/config.py) | All environment variables & settings |

## Testing & Debugging

**Common Setup**:
1. Create Telegram bot at @BotFather, get BOT_TOKEN
2. Create MongoDB Atlas cluster, get MONGO_DB_URI
3. Generate STRING_SESSION from @YukkiStringBot
4. Set API_ID/API_HASH from my.telegram.org
5. Create private Telegram group, get LOG_GROUP_ID

**Logging**: Use `from YukkiMusic.logging import LOGGER`
```python
LOGGER(__name__).info("Message")  # Available: debug, info, warning, error
```

**Voice Chat Prerequisites**:
- Bot must be admin in target group
- Voice chat must be active (not ended/closed)
- Assistant account (userbot) required for calls
- Test stream in [__main__.py](YukkiMusic/__main__.py) line ~46 validates setup

## Common Pitfalls

- **Missing userbot**: Multiple STRING_SESSION vars (STRING1-5) required for reliability
- **MongoDB connection**: MONGO_DB_URI must be valid; bot exits if unavailable
- **Voice chat state**: NoActiveGroupCall exceptions occur when voice chat ended; catch specifically
- **File size limits**: TG_AUDIO_FILESIZE_LIMIT (100MB) and TG_VIDEO_FILESIZE_LIMIT (1GB) enforced
- **Language mismatches**: Always use `get_string()` via decorators, not direct YAML imports
