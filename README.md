# Email ↔ Notion — Bidirectional Sync

Two-way bridge between email and Notion for job application tracking.

**PUSH** (Email → Notion): Inbox → LLM classification → Notion database
**PULL** (Notion → Email): Notion action commands → job-auto-apply email engine

Forked from [shuaiyy-ux/email_to_notion](https://github.com/shuaiyy-ux/email_to_notion). Extended with Gmail IMAP source, Anthropic Claude LLM, Notion trigger, and [job-auto-apply](https://github.com/MichaelYangzk/job-auto-apply) integration.

---

## Platform Dependencies

| Service | Purpose | Free Tier | Required |
|---------|---------|-----------|----------|
| [Notion](https://notion.so) | Email tracking database + control terminal | Free | Yes |
| [Anthropic](https://console.anthropic.com) | LLM classification (Claude Haiku) | Pay-per-use | Yes (or OpenAI) |
| [OpenAI](https://platform.openai.com) | Alternative LLM backend | Pay-per-use | Optional |
| [job-auto-apply](https://github.com/MichaelYangzk/job-auto-apply) | Email sending engine (Resend/SMTP) | — | Yes (for PULL) |
| Gmail IMAP | Read incoming replies | Free | Yes (for PUSH) |

## Architecture

```
              Notion Database (Control Terminal)
              ┌─────────────────────────────────┐
              │  Companies │ Contacts │ Emails   │
              │  Stage │ Priority │ Next Action  │
              │  [Action Confirm] checkbox       │
              └──────────┬──────────┬────────────┘
                   PULL ↓          ↑ PUSH
            ┌────────────┘          └────────────┐
            │                                    │
    notion_trigger.py                  gmail_source.py
    (poll for checked actions)         (IMAP fetch)
    → email_actions.py                       ↓
    → job-auto-apply CLI              LLM.py (classify)
    → Resend / SMTP send                   ↓
            │                     notion_sync/ (sync)
            └────────────┐          ┌────────────┘
                         ↓          ↑
                  ┌──────────────────────┐
                  │  Resend (send) +     │
                  │  Gmail IMAP (receive) │
                  └──────────────────────┘
```

## Quickstart

```bash
# 1. Clone
git clone https://github.com/MichaelYangzk/email_to_notion.git
cd email_to_notion

# 2. Install Python deps
pip install -r requirements.txt

# 3. Configure
cp .env.example .env
# Edit .env — see Configuration section below

# 4. Run bidirectional sync
python main.py           # One full cycle (push + pull)
python main.py push      # Email → Notion only
python main.py pull      # Notion → Email only
python main.py loop      # Continuous loop (every 2 min)
python main.py excel     # Legacy Excel-based flow
```

## How the Bidirectional Flow Works

### PUSH: Email → Notion

1. `gmail_source.py` fetches recent emails via Gmail IMAP
2. `LLM.py` classifies each email using Claude Haiku (or GPT-4o-mini)
3. `notion_sync/` creates or updates Notion database pages
4. Forward-only stage protection prevents regression (e.g., can't go from "offer" back to "applied")

### PULL: Notion → Email

1. User checks the **"Action Confirm"** checkbox on a Notion row
2. `notion_trigger.py` detects checked rows and reads the **"Next Action"** value
3. `email_actions.py` bridges to `job-auto-apply` Node.js CLI
4. Action executes (send cold email, schedule followup, archive, etc.)
5. Notion row is updated with results, checkbox unchecked

### Supported Actions (via "Next Action" column)

| Action | Effect |
|--------|--------|
| `send_cold` | Send initial cold email to the contact |
| `reply` | Schedule a followup email |
| `follow_up` | Queue followup emails |
| `schedule` | Schedule interview-related email |
| `archive` | Mark as withdrawn/not-interested |
| `ignore` | Skip, do nothing |

## Configuration

### LLM Provider

| Variable | Default | Description |
|----------|---------|-------------|
| `LLM_PROVIDER` | `anthropic` | `anthropic` or `openai` |
| `ANTHROPIC_API_KEY` | — | Required if using Anthropic |
| `ANTHROPIC_MODEL` | `claude-3-5-haiku-20241022` | Claude model for classification |
| `OPENAI_API_KEY` | — | Required if using OpenAI |
| `OPENAI_MODEL` | `gpt-4o-mini` | OpenAI model for classification |

### Notion

| Variable | Description |
|----------|-------------|
| `NOTION_TOKEN` | Notion internal integration token ([create here](https://www.notion.so/my-integrations)) |
| `NOTION_DATABASE_ID` | Target Notion database ID (from the database URL) |

### Email (for PUSH direction)

| Variable | Description |
|----------|-------------|
| `FROM_EMAIL` | Gmail address for IMAP reading |
| `GMAIL_APP_PASSWORD` | Gmail App Password (16 chars) |

### Bridge (for PULL direction)

| Variable | Default | Description |
|----------|---------|-------------|
| `JOB_APPLY_DIR` | `../job-auto-apply` | Path to job-auto-apply project |
| `NODE_BIN` | `node` | Node.js binary path |

## Notion Database Setup

1. Create a Notion integration at https://www.notion.so/my-integrations
2. Create a database and share it with your integration
3. The system auto-creates required properties on first sync:

| Property | Type | Purpose |
|----------|------|---------|
| Name | title | Email subject |
| Company | rich_text | Sender's company |
| Stage | select | applied, received, interview_scheduled, etc. |
| Priority | select | extremely high, high, medium, low |
| Next Action | rich_text | What to do next |
| Action Confirm | checkbox | Check to trigger the action |
| Importance Score | number | 0.0 - 1.0 |
| Summary | rich_text | LLM-generated summary |
| From | rich_text | Sender address |
| Subject | rich_text | Email subject line |
| Received UTC | date | When email was received |

## File Map

```
email_to_notion/
├── main.py                 # Bidirectional orchestrator
├── gmail_source.py         # Gmail IMAP reader (replaces OneDrive/Excel)
├── email_actions.py        # Bridge to job-auto-apply CLI
├── notion_trigger.py       # Poll Notion → trigger email actions
├── LLM.py                  # LLM classification (Anthropic / OpenAI)
├── schema_converter.py     # DataFrame type normalization
├── local_copy_manager.py   # Legacy: OneDrive Excel merge
├── notion_sync/
│   ├── runner.py           # Sync orchestrator (Excel + dict rows)
│   ├── notion_client.py    # Notion REST API client
│   ├── mapping.py          # Property mapping
│   ├── idempotency.py      # Forward-only stage logic
│   ├── page_template.py    # Page content builder
│   ├── excel_io.py         # Excel read/write
│   └── config.py           # Notion credentials
├── .env.example            # Environment template
└── requirements.txt        # Python dependencies
```

## Related Projects

- [job-auto-apply](https://github.com/MichaelYangzk/job-auto-apply) — Email sending engine (Resend / Gmail SMTP / SendGrid) with custom domain setup guide
- [notion-trigger](https://github.com/MichaelYangzk/Tensor_revive/tree/main/tools/notion-trigger) — TypeScript Notion task orchestration (original inspiration)

## Credits

- Original `email_to_notion` by [shuaiyy-ux](https://github.com/shuaiyy-ux/email_to_notion)
- Bidirectional integration by [MichaelYangzk](https://github.com/MichaelYangzk)

## License

MIT
