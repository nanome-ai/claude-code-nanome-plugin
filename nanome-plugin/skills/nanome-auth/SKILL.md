---
name: nanome-auth
description: Authenticate with Nanome using browser-based device code login. Use when the user needs to set up Nanome authentication, when ~/.nanome/config.json is missing or has an invalid token, or when the nanome-workspace skill fails with 401/403 errors. Trigger on phrases like "nanome login", "nanome auth", "set up nanome", "authenticate with nanome", or any Nanome-related 401/403 error.
---

# Nanome Authentication Skill

This skill authenticates users with Nanome via a browser-based device code flow. It connects to the Nanome Easy Auth WebSocket, obtains a 4-character login code, opens the user's browser, and waits for them to enter the code on mara.nanome.ai. Once authenticated, the token is saved to `~/.nanome/config.json` for use by other Nanome skills (e.g., `nanome-workspace`).

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
- If output is `INVALID` — tell the user their token has expired and proceed to Step 2.
- If output is `MISSING` — proceed to Step 2.

### Step 2: Run Easy Auth Device Code Flow

This flow uses TWO separate script executions with a message to the user in between. This is critical — the user MUST see the login code and instructions before the wait phase begins.

Detect the execution environment:

1. **Bash tool available** — use Script Mode (below)
2. **No Bash tool** — use Manual Mode (bottom of this document)

---

## Script Mode

### Phase 1: Get device code and open browser

Write this script to `/tmp/nanome_auth_phase1.py` and execute it. It completes quickly (< 5 seconds).

```python
import asyncio
import json
import os
import subprocess
import sys
import platform

async def get_code():
    try:
        import websockets
    except ImportError:
        subprocess.check_call([sys.executable, "-m", "pip", "install", "websockets", "-q"])
        import websockets

    ws_url = "wss://api.nanome.ai/easy-auth"

    async with websockets.connect(ws_url) as ws:
        await ws.send(json.dumps({"type": "NEW_CODE"}))

        while True:
            msg = json.loads(await ws.recv())
            if msg["type"] == "CODE":
                code = msg["data"]["code"]
                break

    # Open browser
    url = "https://mara.nanome.ai"
    try:
        if platform.system() == "Darwin":
            subprocess.Popen(["open", url], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
        elif platform.system() == "Linux":
            subprocess.Popen(["xdg-open", url], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
        elif platform.system() == "Windows":
            os.startfile(url)
    except Exception:
        pass

    # Output ONLY the code — Claude will parse this
    print(code)

asyncio.run(get_code())
```

The script outputs ONLY the 4-character code (e.g., `AB1X`). Parse this from the output.

### CRITICAL: Message the user BEFORE Phase 2

After Phase 1 completes and you have the code, you MUST send a message to the user with these exact instructions. Do NOT skip this step. Do NOT proceed to Phase 2 until you have displayed this message:

---

**Your login code is: `{CODE}`**

A browser window has opened to mara.nanome.ai. To complete authentication:

1. **Log in** to your Nanome account (or create one)
2. **Click "Login via Device Code"** — it's the headset icon in the left sidebar
3. **Enter the code:** `{CODE}`

Waiting for you to complete authentication...

---

Replace `{CODE}` with the actual 4-character code from Phase 1.

### Phase 2: Wait for token

AFTER displaying the message above, write this script to `/tmp/nanome_auth_phase2.py` and execute it with the code from Phase 1 as an argument. Use a **timeout of at least 330 seconds** (5.5 minutes) on the Bash tool since it waits for user interaction.

```bash
python3 /tmp/nanome_auth_phase2.py {CODE_FROM_PHASE1}
```

```python
import asyncio
import json
import os
import sys
import subprocess

async def wait_for_token():
    try:
        import websockets
    except ImportError:
        subprocess.check_call([sys.executable, "-m", "pip", "install", "websockets", "-q"])
        import websockets

    code = sys.argv[1] if len(sys.argv) > 1 else None
    ws_url = "wss://api.nanome.ai/easy-auth"
    config_path = os.path.expanduser("~/.nanome/config.json")

    async with websockets.connect(ws_url) as ws:
        # Reuse the existing code from Phase 1
        if code:
            await ws.send(json.dumps({"type": "OLD_CODE", "data": code}))
        else:
            await ws.send(json.dumps({"type": "NEW_CODE"}))

        # Wait for TOKEN (ignore CODE responses, handle errors)
        try:
            while True:
                raw = await asyncio.wait_for(ws.recv(), timeout=300)
                msg = json.loads(raw)
                if msg["type"] == "TOKEN":
                    token = msg["data"]
                    break
                elif msg["type"] == "ERROR":
                    print("CODE_EXPIRED")
                    sys.exit(1)
                # Ignore CODE responses — we already have the code
        except asyncio.TimeoutError:
            print("TIMEOUT")
            sys.exit(1)

    # Save token to config
    os.makedirs(os.path.dirname(config_path), exist_ok=True)
    config = {}
    if os.path.exists(config_path):
        try:
            with open(config_path) as f:
                config = json.load(f)
        except (json.JSONDecodeError, IOError):
            config = {}

    config["api_base"] = config.get("api_base", "https://workspaces.nanome.ai")
    config["token"] = token

    with open(config_path, "w") as f:
        json.dump(config, f, indent=2)

    # Validate
    import urllib.request
    req = urllib.request.Request(
        "https://api.nanome.ai/user/session",
        headers={"Authorization": f"Bearer {token}"}
    )
    try:
        with urllib.request.urlopen(req) as resp:
            if resp.status == 200:
                data = json.loads(resp.read())
                name = data.get("results", {}).get("user", {}).get("name", "")
                print(f"SUCCESS:{name}")
            else:
                print("SUCCESS:")
    except Exception:
        print("SUCCESS:")

asyncio.run(wait_for_token())
```

Parse the output:
- `SUCCESS:{name}` — Authentication complete. Inform the user: "Authenticated as {name}. Token saved to ~/.nanome/config.json."
- `TIMEOUT` — Timed out. Tell the user and offer to retry by re-running from Phase 1.
- `CODE_EXPIRED` — The device code expired between Phase 1 and Phase 2. Re-run from Phase 1 to get a fresh code.

### After Successful Auth

Confirm to the user:
1. Authentication was successful (include their name if available)
2. Token is saved at `~/.nanome/config.json`
3. Proceed with whatever task triggered the auth (e.g., workspace creation)

---

## Manual Mode

When the Bash tool is not available, walk the user through manual authentication:

1. **Direct them to log in:**
   > Go to https://mara.nanome.ai/login and log in (or create an account).

2. **Direct them to the settings page:**
   > Once logged in, go to https://mara.nanome.ai/settings and create an API key.
   > Alternatively, open browser developer tools (F12), go to Application > Local Storage > mara.nanome.ai, and look for the `nanome-token` key.

3. **Have them create the config file:**
   > Create the file `~/.nanome/config.json` with this content:
   > ```json
   > {
   >   "api_base": "https://workspaces.nanome.ai",
   >   "token": "YOUR_TOKEN_HERE"
   > }
   > ```
   > Replace `YOUR_TOKEN_HERE` with the token you copied.

4. **Confirm it works:**
   > Try creating a workspace with the nanome-workspace skill to confirm authentication works.

---

## Error Handling

| Error | Cause | Resolution |
|-------|-------|------------|
| `websockets` install fails | No pip or restricted environment | Fall back to Manual Mode |
| WebSocket connection refused | Network issue or API down | Retry once; if still failing, fall back to Manual Mode |
| Timeout after 5 minutes | User didn't complete browser flow | Re-run Phase 1 to get a fresh code |
| Token validation returns non-200 | Token may be for wrong environment | Check `api_base` in config; token from mara.nanome.ai should work with `api.nanome.ai` |
| `~/.nanome/` directory can't be created | Permissions issue | Suggest the user create the directory manually: `mkdir -p ~/.nanome` |

---

## URL Reference

- **Nanome login (browser):** `https://mara.nanome.ai/login`
- **Nanome settings:** `https://mara.nanome.ai/settings`
- **Easy Auth WebSocket:** `wss://api.nanome.ai/easy-auth`
- **Session validation:** `GET https://api.nanome.ai/user/session` with `Authorization: Bearer {token}`
- **Workspace API:** `https://workspaces.nanome.ai` (used by nanome-workspace skill)
- **Config file:** `~/.nanome/config.json`
