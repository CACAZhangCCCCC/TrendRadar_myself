# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

TrendRadar is a hot news aggregation and analysis tool that crawls multiple platforms (Baidu, Weibo, Zhihu, etc.) and RSS feeds, filters content by keywords, performs AI-powered analysis and translation, and sends notifications to multiple channels.

**Version**: 6.0.0 (current)
**Python**: >=3.10
**Key Dependencies**: litellm, feedparser, boto3, fastmcp, PyYAML

## Quick Start Commands

### Running the Application

```bash
# Main program - run news crawler and push
python -m trendradar

# MCP Server - for AI client integration (Claude Desktop, Cherry Studio, etc.)
trendradar-mcp

# Check push/AI status
python -m trendradar --show-push-status
python -m trendradar --show-ai-status

# Force execution (bypass once_per_day limits)
python -m trendradar --force-push
python -m trendradar --force-ai

# Reset execution state
python -m trendradar --reset-push-state
python -m trendradar --reset-ai-state
```

### Docker Deployment

```bash
# Using pre-built image
cd docker
docker-compose up -d

# Build locally
docker-compose -f docker-compose-build.yml up -d

# Manage container
python manage.py {start|stop|restart|logs|status}
```

### Installation

```bash
pip install -r requirements.txt
# or
pip install -e .  # editable install
```

## Architecture

### Execution Flow

```
1. Crawl (collect)
   ├─ Platforms: Fetch hot news from configured platforms via API
   └─ RSS: Parse RSS/Atom feeds

2. Store
   └─ SQLite databases: output/news/{date}.db, output/rss/{date}.db

3. Analyze
   ├─ Frequency matching: Match news against frequency_words.txt
   ├─ AI analysis: Deep insights using configured LLM (optional)
   └─ AI translation: Translate content to target language (optional)

4. Push
   ├─ Notifications: Send to configured channels (Feishu, Dingtalk, Telegram, etc.)
   └─ Reports: Generate HTML reports in output/html/
```

### Core Modules

- **`trendradar/core/`**: Core logic
  - `scheduler.py` - Timeline scheduling system (v6.0.0)
  - `config.py` - Configuration loader
  - `frequency.py` - Keyword matching engine
  - `analyzer.py` - News aggregation and statistics
  - `loader.py` - Data loading from storage

- **`trendradar/crawler/`**: Data fetching
  - `fetcher.py` - Platform crawler (uses newsnow API)
  - `rss/` - RSS/Atom feed parser

- **`trendradar/ai/`**: AI capabilities
  - `analyzer.py` - AI-powered news analysis
  - `translator.py` - AI-powered translation
  - `client.py` - LiteLLM wrapper (supports 100+ providers)

- **`trendradar/storage/`**: Persistence layer
  - `local.py` - SQLite backend
  - `remote.py` - S3-compatible remote storage (R2, OSS, COS)
  - `manager.py` - Unified storage interface

- **`trendradar/notification/`**: Push delivery
  - `dispatcher.py` - Multi-channel notification dispatcher
  - `formatters.py` - Platform-specific formatting
  - `senders.py` - Channel implementations

- **`trendradar/report/`**: Report generation
  - `html.py` - HTML report renderer
  - `generator.py` - Report data preparation

- **`mcp_server/`**: Model Context Protocol server
  - `server.py` - FastMCP application
  - `tools/` - MCP tools (data_query, analytics, search, notification, etc.)
  - `services/` - Business logic layer

### Configuration System (v6.0.0 Breaking Changes)

**Primary Configs**:
- `config/config.yaml` (v2.0.0) - Main configuration
  - 7 sections: app, platforms, rss, report, display, notification, storage, ai, ai_analysis, ai_translation, advanced
  - Environment variables override config values (e.g., `AI_API_KEY`, `S3_ENDPOINT_URL`)

- `config/timeline.yaml` (v1.0.0) - Unified scheduling system
  - Replaces old `push_window` and `analysis_window` configs
  - Controls when to collect/analyze/push
  - Supports presets: `always_on`, `morning_evening`, `office_hours`, `night_owl`, `custom`
  - Uses periods + day_plans + week_map model

- `config/frequency_words.txt` (v1.1.0) - Keyword configuration
  - Syntax: keywords, `/regex/`, `=> display_name`, `[group_name]`, `+required`, `!exclude`, `@max_count`
  - Supports `[GLOBAL_FILTER]` section and `[WORD_GROUPS]` section

- `config/ai_analysis_prompt.txt` (v2.0.0) - AI analysis prompt template
- `config/ai_translation_prompt.txt` - AI translation prompt template

**Visual Config Editor**: https://sansan0.github.io/TrendRadar/

### Timeline Scheduling System (v6.0.0)

The new scheduler (`trendradar/core/scheduler.py`) provides unified time-based control:

```python
# Key concepts:
# - periods: Named time ranges with behavior flags (collect/analyze/push)
# - day_plans: Named configurations for different day types
# - week_map: Maps weekdays to day_plans
# - default: Fallback behavior when no period matches

# Resolved at runtime into:
class ResolvedSchedule:
    collect: bool      # Whether to crawl data
    analyze: bool      # Whether to run AI analysis
    push: bool         # Whether to send notifications
    report_mode: str   # daily/current/incremental
    ai_mode: str       # follow_report/daily/current/incremental
    once_analyze: bool # Dedupe AI analysis per day/week
    once_push: bool    # Dedupe push per day/week
```

### Report Modes

- **`daily`** (全天汇总): All matched news + new items section + scheduled push
- **`current`** (当前榜单): Currently trending + new items section + scheduled push
- **`incremental`** (增量监控): Only push when new items appear

Display modes:
- **`keyword`**: Group by keywords (default)
- **`platform`**: Group by platforms/sources

### AI Integration (LiteLLM)

Supports 100+ providers via unified interface:

```yaml
ai:
  model: "deepseek/deepseek-chat"  # Format: provider/model_name
  api_key: ""                       # Or use AI_API_KEY env var
  api_base: ""                      # Optional custom endpoint
```

Common models:
- DeepSeek: `deepseek/deepseek-chat`
- OpenAI: `openai/gpt-4o`, `openai/gpt-4o-mini`
- Gemini: `gemini/gemini-2.5-flash`
- Claude: `anthropic/claude-3-5-sonnet`
- Local: `ollama/llama3`

AI features:
- **Analysis** (`ai_analysis.enabled`): Trend insights, sentiment analysis, cross-platform correlation
- **Translation** (`ai_translation.enabled`): Translate push content to any language
- **Independent mode control**: AI can analyze different mode than push (e.g., push=incremental, analyze=current)

### Storage Backends

- **Local** (`storage.backend: local`):
  - SQLite databases in `output/news/` and `output/rss/`
  - HTML reports in `output/html/`

- **Remote** (`storage.backend: remote`):
  - S3-compatible storage (R2, OSS, COS, MinIO)
  - Configured via `storage.remote` section or env vars (`S3_BUCKET_NAME`, etc.)

- **Auto** (`storage.backend: auto`):
  - Detects GitHub Actions + remote config → use remote
  - Otherwise → use local

### Notification Channels

Supported (configure in `notification.channels`):
- Feishu (飞书)
- Dingtalk (钉钉)
- WeWork (企业微信) - markdown or text mode
- Telegram
- Email (with auto-detected SMTP settings)
- ntfy
- Bark
- Slack
- Generic Webhook (for Discord, Matrix, IFTTT, etc.)

Multi-account support: Use `;` separator for multiple accounts (up to 3 per channel).

## Important Implementation Details

### Frequency Word Matching

The keyword matching engine (`trendradar/core/frequency.py`) supports:
- Plain text matching
- Regex patterns: `/\bAI\b/` (use `\b` for word boundaries)
- Display names: `/pattern/ => Display Name`
- Group names: `[Group Name]` on first line
- Required words: `+word` (all must match)
- Exclude words: `!word` (excludes if matched)
- Max count limit: `@5` (limit to 5 items)

### Data Schema

SQLite tables (see `trendradar/storage/schema.sql`):
- `news` - Platform hot news with rank tracking
- `rss_items` - RSS feed items
- `metadata` - DB version and timestamps
- `scheduler_state` - Timeline execution tracking (v6.0.0)

### GitHub Actions Workflow

File: `.github/workflows/crawler.yml`

- Triggered by cron schedule (UTC time, convert from Beijing +8)
- Auto-expires after 7 days (trial mode feature)
- Secrets required: `AI_API_KEY`, notification webhooks, S3 credentials (if using remote storage)

### MCP Server Tools

The MCP server (`mcp_server/`) provides 20+ tools for AI clients:

**Data Query**:
- `get_today_news`, `get_news_by_date`, `search_news`
- `get_latest_rss`, `search_rss`, `get_rss_feeds_status`

**Analytics**:
- `get_trending_topics`, `analyze_keyword_trend`
- `aggregate_news`, `compare_periods`

**Search**:
- `find_related_news`, `search_across_dates`

**Article Reading** (via Jina AI):
- `read_article`, `read_articles_batch`

**System**:
- `check_version`, `get_config_summary`

**Notification** (v4.0.0):
- `send_notification` - Push AI-generated content to all channels

### Context and Helpers

`trendradar/context.py` - `AppContext` class:
- Centralizes access to configuration, storage, time functions
- Creates managers: storage_manager, notification_dispatcher, push_manager (deprecated in v6.0.0)
- Provides helper methods: `get_time()`, `format_date()`, `format_time()`, `load_frequency_words()`, etc.

## Configuration Migration Notes (v6.0.0)

If upgrading from v5.x:

**Removed**:
- `notification.push_window` → Use `timeline.yaml` periods
- `ai_analysis.analysis_window` → Use `timeline.yaml` periods
- `trendradar/notification/push_manager.py` → Merged into scheduler

**New**:
- `config/timeline.yaml` - Required for scheduling
- `config.schedule.preset` - Choose timeline preset
- `ai_analysis.include_standalone` - AI analysis of standalone sections

**Changed**:
- `config.yaml` version: 1.2.0 → 2.0.0
- `ai_analysis_prompt.txt` version: 1.x → 2.0.0 (format/stability improvements)

## Development Tips

### When Modifying Configs

1. Use the visual editor (https://sansan0.github.io/TrendRadar/) for validation
2. Config version tracking in file headers - bump when making breaking changes
3. Test locally before pushing to GitHub Actions
4. Environment variables always override config file values

### When Adding RSS Feeds

Format in `config.yaml`:
```yaml
rss:
  feeds:
    - id: "unique-id"
      name: "Display Name"
      url: "https://example.com/feed.xml"
      max_age_days: 3  # Optional, override global freshness filter
      enabled: true    # Optional, default true
```

### When Modifying Notification Channels

1. Update `trendradar/notification/senders.py` for new channel
2. Update `trendradar/notification/formatters.py` for formatting
3. Add config section in `config.yaml`
4. Update batch size in `advanced.batch_size` if needed
5. For MCP integration, update `mcp_server/tools/notification.py`

### Debugging

Enable debug mode in `config.yaml`:
```yaml
advanced:
  debug: true  # Enables verbose logging and stack traces
```

Check state files:
```bash
# SQLite state (v6.0.0)
sqlite3 output/news/{date}.db "SELECT * FROM scheduler_state;"

# Legacy state files (v5.x, removed in v6.0.0)
# - Were in output/state/push_history.db
```

## Common Gotchas

1. **Cron times are in UTC**: GitHub Actions cron uses UTC. Beijing time = UTC + 8 hours.

2. **Timeline config priority**:
   - `config.yaml` total switches (e.g., `notification.enabled`) override timeline periods
   - Timeline only controls "when", not "whether"

3. **Freshness filter timing**: RSS freshness filtering happens at push time, not crawl time. All items are stored in DB regardless of age.

4. **AI mode vs report mode**: AI can analyze in different mode than push (configured via `ai_analysis.mode`).

5. **S3 region quirks**: Some S3-compatible services (like Cloudflare R2) don't require `region`, others do. Check provider docs.

6. **Regex escaping**: In YAML strings, backslashes need escaping: `\\b` for word boundary. Better to use single quotes: `'/\bAI\b/'`.

## Testing Changes

### Local Testing
```bash
# Dry run (check config validity)
python -m trendradar --show-push-status

# Force run (bypass time windows and once_per_day)
python -m trendradar --force-push --force-ai

# Reset state to test "first run" behavior
python -m trendradar --reset-push-state --reset-ai-state
```

### GitHub Actions Testing
1. Push changes to GitHub
2. Go to Actions tab → "Get Hot News" workflow
3. Click "Run workflow" → Select branch → Run
4. Monitor logs for errors

### MCP Server Testing
1. Add to Claude Desktop config (see README-MCP-FAQ.md)
2. Restart Claude Desktop
3. Check MCP connection indicator
4. Test tools via prompts like "Get today's trending news"

## Upstream Sync

Original repository: https://github.com/sansan0/TrendRadar

```bash
# Add upstream remote (first time only)
git remote add upstream https://github.com/sansan0/TrendRadar.git

# Fetch and merge updates
git fetch upstream
git merge upstream/master

# Push to your fork
git push origin master
```

Check for breaking changes in:
- `version` file - TrendRadar version
- `version_configs` file - Config file versions
- `version_mcp` file - MCP server version
- README changelog section
