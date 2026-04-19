---
id: gdrive-to-brain
name: GDrive-to-Brain
version: 0.11.1
description: Google Docs from a chosen Drive root folder become searchable brain pages. Deterministic export, agent judgment for enrichment.
category: sense
requires: [credential-gateway]
secrets:
  - name: CLAWVISOR_URL
    description: ClawVisor gateway URL (Option A — recommended, handles OAuth for you)
    where: https://clawvisor.com — create an agent, activate Google Drive / Docs service
  - name: CLAWVISOR_AGENT_TOKEN
    description: ClawVisor agent token (Option A)
    where: https://clawvisor.com — agent settings, copy the agent token
  - name: GOOGLE_CLIENT_ID
    description: Google OAuth2 client ID (Option B — direct Drive/Docs API access)
    where: https://console.cloud.google.com/apis/credentials — create OAuth 2.0 Client ID
  - name: GOOGLE_CLIENT_SECRET
    description: Google OAuth2 client secret (Option B)
    where: https://console.cloud.google.com/apis/credentials — same page as client ID
health_checks:
  - type: any_of
    label: "Auth provider"
    checks:
      - type: http
        url: "$CLAWVISOR_URL/health"
        label: "ClawVisor"
      - type: env_exists
        name: GOOGLE_CLIENT_ID
        label: "Google OAuth"
setup_time: 20 min
cost_estimate: "$0 (both options are free)"
---

# GDrive-to-Brain: Google Docs Become Searchable Brain Pages

Pick a root folder in Google Drive. Every Google Doc underneath it becomes a
stable brain page with source metadata, sync state, and deterministic text
export. The collector does the mechanical work. The agent does the judgment.

## IMPORTANT: Instructions for the Agent

**You are the installer.** Follow these steps precisely.

**The core pattern: code for data, LLMs for judgment.**
Drive collection is split into two layers:
1. DETERMINISTIC: code lists Docs under a configured Drive root, exports text,
   stores metadata, and tracks sync state by doc ID.
2. LATENT: you (the agent) decide which docs matter, how to summarize them,
   how to link them, and what other brain pages to update.

**Do not export Docs by hand via ad hoc API calls.** Use a collector script.
It handles recursion, MIME type filtering, stable IDs, modified timestamps, and
incremental sync. If you improvise this via one-off calls, you will miss files,
duplicate content, or lose stable references.

**Why the chosen-root-folder scope matters:**
- It keeps v1 reviewable and avoids importing the user's whole Drive.
- It gives the user a predictable boundary they can reason about.
- It creates a clean contract for incremental sync and future expansion.

## Architecture

```
Google Drive (chosen root folder)
  ↓ (ClawVisor credential gateway or Google OAuth)
Drive Collector (deterministic Node.js script)
  ↓ Outputs:
  ├── brain/integrations/gdrive/docs/{slug}-{docId}.md   (stable doc pages)
  ├── brain/integrations/gdrive/.raw/{docId}.txt         (raw exported text)
  ├── brain/integrations/gdrive/.raw/manifest.json       (metadata + folder paths)
  └── brain/integrations/gdrive/state.json               (doc ID + modified time)
  ↓
Agent reads imported pages
  ↓ Judgment calls:
  ├── Summarize important docs
  ├── Link docs to people / companies / projects
  ├── Update compiled truth on existing pages
  └── Ignore low-signal documents
```

## Opinionated Defaults

**Scope:**
- Sync ONE configured Drive root folder
- Recurse into subfolders under that root
- Import Google Docs only (`application/vnd.google-apps.document`)
- Skip PDFs, Sheets, Slides, images, comments, permissions, and sharing logic in v1
- Follow-up: evaluate Google Docs comments separately, because comment content may matter and is not guaranteed to appear in the plain text export path

**Page shape:**
Each Google Doc becomes one stable page with source metadata at top:

```markdown
# Quarterly Planning Notes

Source: Google Drive
Doc ID: 1AbCdEfGhIjKlMn
Drive URL: https://docs.google.com/document/d/1AbCdEfGhIjKlMn/edit
Modified: 2026-04-18T15:02:11Z
Folder: /Company/Planning/Q2

---

[exported document text]
```

**Stable identity:**
- Use Google Doc ID as the primary key
- File path can change if title changes, but state tracking keys off doc ID
- If a title changes, update the page filename and preserve the underlying identity

## Prerequisites

1. **GBrain installed and configured** (`gbrain doctor` passes)
2. **Node.js 18+** (for the collector script)
3. **Google Drive / Docs access** via ONE of:
   - **Option A: ClawVisor** (recommended, handles OAuth and token refresh)
   - **Option B: Google OAuth2 directly** (you manage tokens, no extra service needed)

## Setup Flow

### Step 1: Validate Credential Gateway

Ask the user: "How do you want to connect to Google Drive / Docs?

**Option A: ClawVisor (recommended)**
ClawVisor handles OAuth, token refresh, and encryption. If you already use it
for Gmail or Calendar, this should be the same auth path.

**Option B: Google OAuth2 directly**
Connect to Google Drive / Docs APIs directly. No extra service needed, but you
manage OAuth tokens yourself."

#### Option A: ClawVisor (recommended)

Tell the user:

"I need your ClawVisor URL and agent token.
1. Go to https://clawvisor.com
2. Create an agent (or use an existing one)
3. Activate the Google Drive / Docs service
4. Create a standing task with purpose: 'Full executive assistant access to
   Google Drive and Google Docs including folder traversal, doc listing,
   metadata lookup, text export, and historical document access for connected
   Google accounts.'
   IMPORTANT: Be EXPANSIVE in the task purpose. Narrow purposes like 'read one
   folder' can cause legitimate sync requests to fail verification.
5. Copy the gateway URL and agent token"

Validate:
```bash
curl -sf "$CLAWVISOR_URL/health" && echo "PASS: ClawVisor reachable" || echo "FAIL"
```

**STOP until ClawVisor validates.**

#### Option B: Google OAuth2 directly

Tell the user:
"I need Google OAuth2 credentials for Drive / Docs access. Here's how:

1. Go to https://console.cloud.google.com/apis/credentials
2. Click **'+ CREATE CREDENTIALS'** > **'OAuth client ID'**
3. If prompted, configure the OAuth consent screen:
   - User type: **External** (or Internal for Google Workspace)
   - App name: 'GBrain Drive' (anything works)
   - Scopes: add:
     - `https://www.googleapis.com/auth/drive.readonly`
     - `https://www.googleapis.com/auth/documents.readonly`
   - Test users: add your own email address
4. Create the OAuth client ID:
   - Application type: **Desktop app**
   - Name: 'GBrain'
5. Copy the **Client ID** and **Client Secret**
6. Enable the APIs:
   - Google Drive API: https://console.cloud.google.com/apis/library/drive.googleapis.com
   - Google Docs API: https://console.cloud.google.com/apis/library/docs.googleapis.com"

Validate:
```bash
[ -n "$GOOGLE_CLIENT_ID" ] && [ -n "$GOOGLE_CLIENT_SECRET" ]   && echo "PASS: Google OAuth credentials set"   || echo "FAIL: Missing GOOGLE_CLIENT_ID or GOOGLE_CLIENT_SECRET"
```

**STOP until OAuth credentials validate.**

### Step 2: Ask for the Drive Root Folder

Ask the user:
"What should be the root of this sync? Give me either:
- a Google Drive folder URL, or
- a folder ID, or
- the exact name of a folder if we need to look it up"

Record this value in the collector config as `rootFolderId`.

Use an explicit config file so the setup is reproducible:

```json
{
  "rootFolderId": "<google-drive-folder-id>",
  "provider": "clawvisor"
}
```

`provider` can be `clawvisor` or `google-oauth`. The collector should read one config file and one auth path, not guess.

**Default behavior:** recurse under the chosen root folder. Do not scan outside it.

### Step 3: Set Up the Drive Collector

Create the collector directory and script:

```bash
mkdir -p gdrive-collector/data/.raw
cd gdrive-collector
npm init -y
```

The collector script needs these capabilities:
1. **discover** — list child items under `rootFolderId`, recurse into subfolders,
   keep only Google Docs MIME type
2. **export** — export each eligible Doc to plain text
3. **materialize** — create one stable brain page per Doc with source metadata
4. **state tracking** — remember doc IDs and modified times to avoid re-exporting unchanged Docs

Key design rules for the collector:
- Filter by MIME type in CODE, not by agent judgment
- Use doc ID as the stable identity
- Store raw exported text separately from the final page materialization
- Record folder path in metadata for each Doc
- Keep all sync state in `data/state.json`
- Track the last materialized page path for each doc ID so title changes can rename the page cleanly

Minimum file contract:
- `gdrive.config.json` — collector config (`rootFolderId`, auth provider)
- `data/.raw/manifest.json` — discovered Docs for the current run
- `data/.raw/{docId}.txt` — exported plain text per Doc
- `data/state.json` — per-doc sync state, including `modifiedTime` and last successfully materialized `pagePath`

Minimum schema contract:
- `manifest.json` entries should include: `id`, `name`, `modifiedTime`, `folderPath`
- `state.json.docs[docId]` should include: `modifiedTime`, `title`, `folderPath`, `pagePath`

Provider boundary:
- The recipe defines what the collector must do, not the exact API client implementation
- The provider layer only needs two capabilities: `list children for a folder` and `export one Doc to plain text`
- Whether that provider is implemented via ClawVisor or direct Google OAuth is an implementation choice

### Step 4: Run First Sync

```bash
node gdrive-collector.mjs discover
node gdrive-collector.mjs export
node gdrive-collector.mjs materialize
```

Verify:
- `brain/integrations/gdrive/docs/` contains one page per Doc
- each page has a working Google Docs URL
- only Docs under the chosen root folder were imported

### Step 5: Enrich Brain Pages

This is YOUR job (the agent). Read the imported pages. For each Doc:

1. **Decide if it matters**: is this an important project, person, company, or process doc?
2. **Link it**: add links to existing pages where relevant
3. **Summarize if valuable**: if the Doc is long or central, add a short summary near the top or update a related page
4. **Update compiled truth**: if the Doc contains stable facts that belong on a canonical page, extract and update them
5. **Sync**: run `gbrain sync --no-pull --no-embed` to index changes

### Step 6: Set Up Cron

The collector should run every 6 hours by default:

```bash
0 */6 * * * cd /path/to/gdrive-collector && node gdrive-collector.mjs discover && node gdrive-collector.mjs export && node gdrive-collector.mjs materialize
```

More frequent sync is fine for small folders, but the default should be conservative.

### Step 7: Log Setup Completion

```bash
mkdir -p ~/.gbrain/integrations/gdrive-to-brain
echo '{"ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","event":"setup_complete","source_version":"0.11.1","status":"ok","details":{"scope":"docs-only","selection":"root-folder"}}' >> ~/.gbrain/integrations/gdrive-to-brain/heartbeat.jsonl
```

## Implementation Guide

These are the production rules for v1.

### MIME Type Filter (Deterministic)

```
DOC_MIME = 'application/vnd.google-apps.document'

is_supported(item):
  return item.mimeType == DOC_MIME
```

Simple and strict. Do not include Sheets, Slides, PDFs, or folders in the final document set.

### Recursive Traversal

```
walk(folderId, path):
  items = drive.list(parent=folderId)
  for item in items:
    if item.mimeType == 'application/vnd.google-apps.folder':
      walk(item.id, path + '/' + item.name)
    else if is_supported(item):
      yield { id: item.id, name: item.name, path }
```

The collector should traverse only the configured root and its descendants.

### Incremental Sync

```
should_export(item, state):
  prev = state.docs[item.id]
  return !prev || prev.modifiedTime != item.modifiedTime
```

This is the key to cheap re-runs. Compare Google modified time against saved state. During export, update sync metadata like `modifiedTime`, but do not overwrite the last materialized `pagePath` until materialization succeeds.

If a Doc disappears from the configured root, the conservative v1 behavior is: stop updating it, but do not delete the existing brain page automatically. Deletion and moved-out-of-scope cleanup can be added later as a separate policy.

### Stable Page Paths

```
page_path(item):
  slug = slugify(item.name)
  return `brain/integrations/gdrive/docs/${slug}-${item.id}.md`

materialize(item, state):
  newPath = page_path(item)
  oldPath = state.docs[item.id]?.pagePath
  write newPath
  if oldPath && oldPath != newPath:
    delete oldPath
  state.docs[item.id].pagePath = newPath
```

The doc ID suffix prevents collisions when two docs have the same title. Use a deterministic slug function and keep it stable across runs. The collector should also remove the old page when a rename changes the slug, otherwise renamed Docs leave stale duplicates behind. Important sequencing rule: `oldPath` must refer to the last successfully materialized page, not a path precomputed during export.

### Follow-up Work

- Add explicit support for Google Docs comments in a later version. Treat this as a separate design problem from plain-text export, because comments may require different API handling and should not silently disappear from the brain.

### What the Agent Should Test After Setup

1. **Scope boundary:** Put one Doc inside the chosen root and one outside it. Sync. Verify only the in-scope Doc appears.
2. **MIME filtering:** Add a Sheet in the root folder. Sync. Verify it is skipped.
3. **Incremental sync:** Run the collector twice with no changes. Verify unchanged Docs are not re-exported.
4. **Modified doc:** Edit a synced Google Doc. Re-run sync. Verify only that Doc page updates.
5. **Rename stability:** Rename a synced Doc. Re-run sync. Verify identity remains tied to doc ID and the old slugged page is removed.

## Cost Estimate

| Component | Monthly Cost |
|-----------|-------------|
| ClawVisor (free tier) | $0 |
| Google Drive / Docs APIs | $0 (within free quota) |
| **Total** | **$0** |

## Troubleshooting

**No Docs imported:**
- Check ClawVisor health: `curl $CLAWVISOR_URL/health`
- Check the standing task has Drive / Docs access enabled
- Verify the configured root folder actually contains Google Docs

**Only some Docs imported:**
- Verify recursion is enabled
- Verify unsupported MIME types are being skipped as expected
- Check whether docs live in a shared drive or location not covered by the current auth scope

**Exports are stale:**
- Check `data/state.json` for modified times
- Edit one Doc and verify the new modified time is observed on the next sync

**Renamed Docs create duplicates:**
- Check whether `state.docs[docId].pagePath` is being updated after materialization
- Ensure the collector deletes the old page path when the slug changes

---

*Part of the [GBrain Skillpack](../docs/GBRAIN_SKILLPACK.md). See also: [Credential Gateway](credential-gateway.md), [Email-to-Brain](email-to-brain.md), [Calendar-to-Brain](calendar-to-brain.md)*
