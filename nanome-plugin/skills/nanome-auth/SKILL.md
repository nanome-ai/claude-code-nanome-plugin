---
name: nanome-auth
description: Authenticate with Nanome by creating an API key at mara.nanome.ai/settings. Use when the user needs to set up Nanome authentication, when ~/.nanome/config.json is missing or has an invalid token, or when the nanome-workspace skill fails with 401/403 errors. Trigger on phrases like "nanome login", "nanome auth", "set up nanome", "authenticate with nanome", or any Nanome-related 401/403 error.
---

# Nanome Authentication Skill

This skill authenticates users with Nanome via an API key created at mara.nanome.ai/settings. The API key is saved to `~/.nanome/config.json` for use by other Nanome skills (e.g., `nanome-workspace`). API keys are long-lived and use the standard `Authorization: Bearer` header.

## Workflow

### Step 1: Check Existing Auth

Before starting the login flow, check if the user already has a valid token.

Run this bash script:

```bash
CONFIG_FILE="$HOME/.nanome/config.json"
if [ -f "$CONFIG_FILE" ]; then
  TOKEN=$(python3 -c "import json; print(json.load(open('$CONFIG_FILE')).get('token', ''))" 2>/dev/null)
  if [ -n "$TOKEN" ]; then
    HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
      -H "Authorization: Bearer $TOKEN" \
      "https://api.nanome.ai/user/session")
    if [ "$HTTP_CODE" = "200" ]; then
      echo "VALID"
    else
      echo "INVALID"
    fi
  else
    echo "MISSING"
  fi
else
  echo "MISSING"
fi
```

- If output is `VALID` — inform the user they are already authenticated. Done.
- If output is `INVALID` — tell the user their API key has expired and proceed to Step 2.
- If output is `MISSING` — proceed to Step 2.

### Step 2: Open browser to settings page

Open the Nanome settings page where the user can create an API key:

```bash
open "https://mara.nanome.ai/settings" 2>/dev/null || xdg-open "https://mara.nanome.ai/settings" 2>/dev/null || echo "Please open https://mara.nanome.ai/settings in your browser"
```

### CRITICAL: Message the user with instructions

After opening the browser, you MUST display these instructions to the user. Do NOT skip this step. Do NOT proceed until the user has pasted their API key.

---

A browser window has opened to **mara.nanome.ai/settings**. To get your API key:

1. **Log in** to your Nanome account (or create one at mara.nanome.ai)
2. Go to the **System** tab in Settings
3. Under **API Keys**, click **Create** — the key is automatically copied to your clipboard
4. **Paste the key here**

---

### Step 3: Receive the API key from the user

Wait for the user to paste their API key. The user will paste it as a message. Once received, proceed to Step 4.

### Step 4: Save and validate the API key

After the user provides the key, save it to `~/.nanome/config.json` and validate it.

Run this bash script, replacing `{API_KEY}` with the key the user provided:

```bash
API_KEY="{API_KEY}"
CONFIG_FILE="$HOME/.nanome/config.json"

# Create directory if needed
mkdir -p "$(dirname "$CONFIG_FILE")"

# Read existing config or start fresh
if [ -f "$CONFIG_FILE" ]; then
  CONFIG=$(cat "$CONFIG_FILE")
else
  CONFIG='{}'
fi

# Write updated config with the new token
python3 -c "
import json, sys
config = json.loads('''$CONFIG''') if '''$CONFIG'''.strip() else {}
config['api_base'] = config.get('api_base', 'https://workspaces.nanome.ai')
config['token'] = '$API_KEY'
with open('$CONFIG_FILE', 'w') as f:
    json.dump(config, f, indent=2)
print('Config saved to $CONFIG_FILE')
"

# Validate the key
HTTP_CODE=$(curl -s -o /tmp/nanome_session.json -w "%{http_code}" \
  -H "Authorization: Bearer $API_KEY" \
  "https://api.nanome.ai/user/session")

if [ "$HTTP_CODE" = "200" ]; then
  NAME=$(python3 -c "import json; print(json.load(open('/tmp/nanome_session.json')).get('results', {}).get('user', {}).get('name', ''))" 2>/dev/null)
  rm -f /tmp/nanome_session.json
  echo "SUCCESS:$NAME"
else
  rm -f /tmp/nanome_session.json
  echo "FAILED:$HTTP_CODE"
fi
```

Parse the output:
- `SUCCESS:{name}` — Authentication complete. Tell the user: "Authenticated as {name}. API key saved to ~/.nanome/config.json."
- `FAILED:{code}` — Validation failed. Tell the user the key appears to be invalid (HTTP {code}) and ask them to double-check it or create a new one at mara.nanome.ai/settings.

### After Successful Auth

Confirm to the user:
1. Authentication was successful (include their name if available)
2. API key is saved at `~/.nanome/config.json`
3. Proceed with whatever task triggered the auth (e.g., workspace creation)

---

## Error Handling

| Error | Cause | Resolution |
|-------|-------|------------|
| Browser doesn't open | No `open` or `xdg-open` available | Tell the user to open `https://mara.nanome.ai/settings` manually |
| Validation returns 401/403 | Key is invalid or expired | Ask user to create a new key at mara.nanome.ai/settings |
| Validation returns other error | Network issue or API down | Save the key anyway and suggest the user try creating a workspace to test |
| `~/.nanome/` directory can't be created | Permissions issue | Suggest: `mkdir -p ~/.nanome` |
| User doesn't see API Keys section | Not on the System tab | Direct them to the System tab in Settings |

---

## URL Reference

- **Nanome settings (API keys):** `https://mara.nanome.ai/settings`
- **Nanome login:** `https://mara.nanome.ai/login`
- **Session validation:** `GET https://api.nanome.ai/user/session` with `Authorization: Bearer {key}`
- **Workspace API:** `https://workspaces.nanome.ai` (used by nanome-workspace skill)
- **Config file:** `~/.nanome/config.json`
