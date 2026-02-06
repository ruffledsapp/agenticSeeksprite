# GLM-4.7 + ZAI + AgenticSeek ("Zebra") on macOS

This guide gives you:

1. A quick **current-state review**.
2. **Five concrete improvement areas**.
3. The easiest path to use your **GLM-4.7 API key from ZAI**.
4. A clean **macOS install walkthrough**.

> Note: AgenticSeek does not currently have a dedicated `zai` provider name. The simplest path is to use the existing `openai` provider mode with a ZAI-compatible OpenAI endpoint.

---

## Where we are today (quick assessment)

AgenticSeek already has what you need for GLM-4.7 through its `openai` provider path:

- `provider_name = openai` works with OpenAI-compatible APIs.
- `provider_server_address` can point to a custom compatible endpoint.
- API keys are read from environment variables.

That means you can use ZAI right now, but setup is still a little manual and spread across `.env`, shell exports, and `config.ini` edits.

---

## Five high-impact improvement areas

### 1) Add a first-class `zai` provider alias

**Why improve:** Today, users must infer that ZAI should be configured through `openai` compatibility mode.

**Simple improvement:**
- Add `zai` to provider list and route it to the same logic as `openai_fn`.
- Read `ZAI_API_KEY` (fallback to `OPENAI_API_KEY`).

**Benefit:** Removes confusion and reduces setup mistakes.

---

### 2) Add a one-command setup helper for API providers

**Why improve:** New users edit multiple files manually.

**Simple improvement:**
- Add a script like `scripts/setup_api_provider.sh` that asks for:
  - provider (`zai`)
  - model (`glm-4.7`)
  - endpoint URL
  - API key
- Script writes valid `.env` + `config.ini` entries.

**Benefit:** Faster onboarding, fewer typo/config errors.

---

### 3) Provide a minimal macOS "happy path" in docs

**Why improve:** Current install docs include many options; users wanting "just make it run" may hesitate.

**Simple improvement:**
- Add one short "10-minute macOS quickstart" section.
- Keep advanced options below it.

**Benefit:** Better first-run success rate.

---

### 4) Add a startup config validator

**Why improve:** Misconfigured `provider_server_address` or missing key failures happen late.

**Simple improvement:**
- On startup, validate:
  - API key presence
  - endpoint URL format
  - model string non-empty
- Print actionable errors before agent launch.

**Benefit:** Immediate feedback and less troubleshooting.

---

### 5) Add tested config examples for OpenAI-compatible providers

**Why improve:** Users copy snippets from issues/discussions with inconsistent formats.

**Simple improvement:**
- Ship docs examples for:
  - ZAI + GLM-4.7
  - OpenRouter
  - local llama.cpp server

**Benefit:** Standardized, trusted examples reduce support load.

---

## Super-simple GLM-4.7 (ZAI) setup

## Step 1) Install on macOS

From the project root:

```bash
chmod +x install.sh scripts/macos_install.sh
./install.sh
```

This runs the macOS installer and sets up Python dependencies.

---

## Step 2) Set your ZAI API key

You have two common options.

### Option A (recommended): put it in `.env`

Create/edit `.env` in project root:

```dotenv
OPENAI_API_KEY="your_zai_api_key_here"
```

### Option B: export in terminal session

```bash
export OPENAI_API_KEY="your_zai_api_key_here"
```

---

## Step 3) Configure `config.ini`

Set your provider to OpenAI-compatible mode:

```ini
[MAIN]
is_local = True
provider_name = openai
provider_model = glm-4.7
provider_server_address = api.z.ai/v1
```

Notes:
- Keep `is_local = True` when using a custom OpenAI-compatible `provider_server_address`.
- If your endpoint requires scheme, use `https://api.z.ai/v1` format where applicable.
- If requests fail, try toggling between `api.z.ai/v1` and `https://api.z.ai/v1` based on provider expectations.

---

## Step 4) Run AgenticSeek

CLI run:

```bash
uv run python cli.py
```

If you use Docker workflow, follow normal compose startup, then launch as documented in `README.md`.

---

## Quick troubleshooting

- **"API key not found"**
  - Confirm `OPENAI_API_KEY` is set in `.env` or current shell.
- **Connection/refused errors**
  - Re-check `provider_server_address` and whether it needs `https://`.
- **Model not found**
  - Verify exact GLM model id from your ZAI account (e.g., `glm-4.7` vs a full published model slug).
- **Unexpected output formatting**
  - Switch to a stronger reasoning-capable model variant if available.

---

## macOS one-page quickstart (copy/paste)

```bash
git clone https://github.com/Fosowl/agenticSeek.git
cd agenticSeek
cp .env.example .env 2>/dev/null || true
chmod +x install.sh scripts/macos_install.sh
./install.sh

# Add your key
printf '\nOPENAI_API_KEY="%s"\n' "your_zai_api_key_here" >> .env

# Then set config.ini values:
# is_local = True
# provider_name = openai
# provider_model = glm-4.7
# provider_server_address = api.z.ai/v1

uv run python cli.py
```

