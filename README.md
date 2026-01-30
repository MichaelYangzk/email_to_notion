# Email Ingestion and Notion Sync

![Screenshot placeholder: end-to-end flow](docs/flow.png)

This repo pulls a OneDrive-hosted Jobs.xlsx into a local working copy, classifies new rows with an LLM, and syncs structured results to Notion.

---

## Program features (PM view)

1) Run-it-all button
- One command copies from OneDrive, merges safely, labels new emails, and updates Notion. No extra steps needed.

2) Clear, consistent labels
- The LLM assigns stage, priority, next action, and an importance score with timestamps. Outputs stay within allowed values, so you don’t get surprises.
- Example: stage set to `interview_scheduled`, priority updated to `high`, next action determined as `schedule`, importance score calculated as `0.82`, summary generated as `"Confirm 2pm interview with Acme"`.
```
- Example: stage=`interview_scheduled`, priority=`high`, next_action=`schedule`, importance_score=`0.82`, summary=`"Confirm 2pm interview with Acme"`.
```


3) Notion stays in sync
- Missing properties are created for you, pages are updated or created using saved page ids, and duplicate body text is avoided to keep timelines readable.

4) Protect work in progress
- Local edits stay in control during merge, NEW rows are protected unless you force-refresh, and terminal stages cannot be rolled back.

5) Easy to review in Excel
- Excel artifacts are cleaned, table styling is preserved when possible, and the sheet remains easy to scan by humans.

---

## How does it work
1. [main.py](main.py) loads configuration and calls `run_full`.
2. [local_copy_manager.py](local_copy_manager.py) copies the OneDrive sheet to a local working copy and smart-merges with any existing local file (local rows with matching `message_id` stay authoritative unless `force_refresh=True`).
3. [LLM.py](LLM.py) queues rows where `llm_status` is blank/NEW, cleans bodies, builds prompts, calls the OpenAI Responses API, validates enums/ranges, stamps timestamps, and clears errors.
4. [notion_sync/runner.py](notion_sync/runner.py) iterates DONE/ERROR rows, ensures Notion property types, and either creates or updates pages via [notion_sync/notion_client.py](notion_sync/notion_client.py).
5. Stage updates respect forward-only progression and duplicate-body suppression in [notion_sync/idempotency.py](notion_sync/idempotency.py) and [notion_sync/page_template.py](notion_sync/page_template.py).
6. The updated dataframe is written back to the local Excel file through [notion_sync/excel_io.py](notion_sync/excel_io.py).

![Screenshot placeholder: Notion sync](docs/notion_properties.png)

---

## How to use it
### Prerequisites
- Python 3.10+ on macOS.
- Dependencies: `pip install -r requirements.txt`.
- Environment variables: `OPENAI_API_KEY` (and optional `OPENAI_MODEL`), `NOTION_TOKEN`, `NOTION_DATABASE_ID`. A `.env` next to [LLM.py](LLM.py) is supported via `dotenv`.
- Access to the OneDrive source file (default path in [main.py](main.py)).

### Run the full flow
```bash
python main.py
```
This will copy/merge from OneDrive, run LLM classification, and sync to Notion.

### Options
- Force refresh (ignore local NEW rows):
```bash
python -c "from main import run_full; run_full(force_refresh=True)"
```
- Point to a different OneDrive path:
```bash
python -c "from main import run_full; run_full(onedrive_path='PATH/TO/Jobs.xlsx')"
```

### Working only on the local copy
```bash
python -c "from local_copy_manager import copy_and_merge_to_local; copy_and_merge_to_local('/path/to/Jobs.xlsx')"
```

### Syncing back to OneDrive (manual)
After processing, copy the local Jobs.xlsx back to OneDrive using Finder or a shell copy command.

### File map
- Orchestration: [main.py](main.py)
- Copy/merge: [local_copy_manager.py](local_copy_manager.py)
- LLM prompt + schema: [LLM.py](LLM.py)
- Notion sync: [notion_sync/runner.py](notion_sync/runner.py) and helpers in [notion_sync](notion_sync)
- Docs: [docs/PRODUCT_FEATURES.md](docs/PRODUCT_FEATURES.md), [docs/LOCAL_COPY_WORKFLOW.md](docs/LOCAL_COPY_WORKFLOW.md)

![Screenshot placeholder: LLM results](docs/notion_example.png)
