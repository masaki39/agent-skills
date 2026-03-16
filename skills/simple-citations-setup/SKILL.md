---
name: simple-citations-setup
description: Setup the simple-citations Obsidian plugin with Zotero Better BibTeX integration. Use when user wants to configure simple-citations or set up Zotero-Obsidian bibliography sync.
allowed-tools: Bash, Read, Write, Edit, Glob, AskUserQuestion
---

# Simple Citations Setup Skill

Setup the [simple-citations](https://github.com/masaki39/simple-citations) Obsidian plugin by:
1. Configuring Zotero's Better BibTeX auto-export
2. Setting the Better BibTeX postscript
3. Enabling the Zotero local HTTP API
4. Configuring Obsidian plugin settings

---

## Step 1: Detect OS and set platform-specific paths

```bash
python3 -c "
import platform, os, sys

os_name = platform.system()  # 'Darwin', 'Windows', 'Linux'
home = os.path.expanduser('~')

if os_name == 'Darwin':
    zotero_profiles = os.path.join(home, 'Library/Application Support/Zotero/Profiles')
    obsidian_config = os.path.join(home, 'Library/Application Support/obsidian/obsidian.json')
    zotero_app_check = '/Applications/Zotero.app'
elif os_name == 'Windows':
    appdata = os.environ.get('APPDATA', '')
    zotero_profiles = os.path.join(appdata, 'Zotero/Profiles')
    obsidian_config = os.path.join(appdata, 'obsidian/obsidian.json')
    zotero_app_check = os.path.join(os.environ.get('PROGRAMFILES','C:/Program Files'), 'Zotero/Zotero.exe')
else:  # Linux
    zotero_profiles = os.path.join(home, '.zotero/zotero')
    obsidian_config = os.path.join(home, '.config/obsidian/obsidian.json')
    zotero_app_check = '/usr/bin/zotero'

print(f'OS={os_name}')
print(f'ZOTERO_PROFILES={zotero_profiles}')
print(f'OBSIDIAN_CONFIG={obsidian_config}')
print(f'ZOTERO_APP={zotero_app_check}')
"
```

Store these values. All subsequent steps use these variables.

---

## Step 2: Check prerequisites

### Obsidian CLI
```bash
which obsidian || echo "NOT FOUND"
```
If not found, instruct the user to enable it in Obsidian: **Settings → General → Command line interface → Enable**, then stop.

### Zotero installation

Check using the `ZOTERO_APP` path from Step 1:
- macOS/Linux: `ls <ZOTERO_APP> 2>/dev/null && echo "FOUND" || echo "NOT FOUND"`
- Windows: `dir "<ZOTERO_APP>" 2>nul && echo FOUND || echo NOT FOUND`

If not found, instruct the user to install Zotero from https://www.zotero.org, then stop.

### Better BibTeX installation

Find the Zotero profile directory using `ZOTERO_PROFILES` from Step 1:
```bash
python3 -c "
import os
profiles_dir = 'FILL_IN_ZOTERO_PROFILES'
entries = [e for e in os.listdir(profiles_dir) if 'default' in e]
print(entries[0] if entries else 'NOT FOUND')
"
```

Check for Better BibTeX:
```bash
python3 -c "
import os, json
profile = os.path.join('FILL_IN_ZOTERO_PROFILES', 'FILL_IN_PROFILE_DIR', 'extensions.json')
with open(profile) as f:
    data = json.load(f)
found = any('better-bibtex' in str(e) for e in data.get('addons', []))
print('FOUND' if found else 'NOT FOUND')
"
```

If NOT FOUND, instruct the user to install Better BibTeX from https://retorque.re/zotero-better-bibtex/ and restart Zotero, then stop.

---

## Step 3: Find Obsidian vaults

Using `OBSIDIAN_CONFIG` from Step 1:
```bash
python3 -c "
import json, os
config = 'FILL_IN_OBSIDIAN_CONFIG'
with open(config) as f:
    data = json.load(f)
vaults = data.get('vaults', {})
for v in vaults.values():
    print(v.get('path', ''))
"
```

---

## Step 4: Ask user for configuration

Use `AskUserQuestion` to collect the following. Show defaults clearly.

### Questions:

1. **Obsidian vault path**
   - Default: the first vault found in Step 3

2. **CSL JSON export path** (where Zotero will write the bibliography file)
   - Recommended: inside the vault root so Obsidian can detect changes
   - Default: `{vault}/MyLibrary.json`

3. **Literature notes folder** (inside the vault)
   - Default: `Literatures`

4. **Auto-add literature notes when bibliography updates?**
   - Recommended: `true` (fully automatic workflow)

5. **Auto-update literature notes when file opens?**
   - Default: `false` (can cause slowness with large libraries)

6. **Postscript for Better BibTeX** (adds extra fields to each item)
   - Show these options and let user choose (multiple allowed):
     - `[key]` — Zotero item key (enables Zotero URI links in notes)
     - `[pdf]` — Local PDF file path
     - `[collections]` — Collection names the item belongs to
     - `[none]` — No postscript

7. **Optional fields to show in notes** (must match postscript choices)
   - Only ask if user chose `[key]`, `[pdf]`, or `[collections]`
   - Pre-fill based on postscript choices (e.g. `key` → `key`, `pdf` → `pdf`)

---

## Step 5: Configure Zotero via user.js

### 5-1: Build the postscript

Based on user's choices in Step 4, construct the postscript. Include only the chosen fields:

```javascript
if (Translator.BetterCSLJSON) {
  // [key]:
  csl.key = zotero.key;

  // [pdf]:
  csl.pdf = zotero.attachments
    .filter(a => a.localPath && a.localPath.toLowerCase().endsWith('.pdf'))
    .map(a => a.localPath);

  // [collections]:
  function path(key) {
    const coll = Translator.collections[key];
    if (!coll) return '';
    if (!coll.parent) { return `${coll.name}`; }
    else { return `${path(coll.parent)}/${coll.name}`; }
  }
  let collections = [];
  zotero.collections.forEach((key) => { collections.push(path(key)); });
  csl.collections = collections;
}
```

If `[none]` was chosen, postscript is an empty string `""`.

### 5-2: Write user.js via Python

The `user.js` file is read by Zotero on startup and safely overrides `prefs.js`. If the file already exists, replace only the `simple-citations-setup` block.

```bash
python3 << 'PYEOF'
import json, time, urllib.parse, os, re, platform

# --- Fill in from previous steps ---
json_path    = "FILL_IN_JSON_PATH"
postscript   = "FILL_IN_POSTSCRIPT"
profiles_dir = "FILL_IN_ZOTERO_PROFILES"
profile_dir  = "FILL_IN_PROFILE_DIR"
# -----------------------------------

user_js_path = os.path.join(profiles_dir, profile_dir, "user.js")

ts = int(time.time() * 1000)
encoded_path = urllib.parse.quote(json_path, safe='')
auto_export_val = json.dumps({
    "created": ts, "path": json_path,
    "translatorID": "f4b52ab0-f878-4556-85a0-c7aeedd09dfc",
    "type": "library", "id": 1, "status": "idle", "error": "",
    "recursive": False, "updated": ts, "enabled": True
})

block = f"""
// === simple-citations-setup ===
user_pref("extensions.zotero.httpServer.localAPI.enabled", true);
user_pref("extensions.zotero.translators.better-bibtex.postscript", {json.dumps(postscript)});
user_pref("extensions.zotero.translators.better-bibtex.autoExport.{encoded_path}", {json.dumps(auto_export_val)});
// === end simple-citations-setup ===
"""

existing = ""
if os.path.exists(user_js_path):
    with open(user_js_path) as f:
        existing = f.read()

pattern = r"\n?// === simple-citations-setup ===.*?// === end simple-citations-setup ===\n?"
existing_clean = re.sub(pattern, "", existing, flags=re.DOTALL)

with open(user_js_path, "w") as f:
    f.write(existing_clean.rstrip() + "\n" + block)

print(f"Written to: {user_js_path}")
PYEOF
```

---

## Step 6: Start/restart Zotero and trigger export

```bash
curl -s http://localhost:23119/api/ 2>/dev/null && echo "RUNNING" || echo "NOT RUNNING"
```

- If **RUNNING**: Zotero must be **restarted** for `user.js` to take effect. Ask the user to restart Zotero now, then wait for confirmation.
- If **NOT RUNNING**: Launch Zotero:
  - macOS: `open -a Zotero`
  - Windows: `start "" "FILL_IN_ZOTERO_APP"`
  - Linux: `zotero &`

  Then wait for startup:
  ```bash
  sleep 8 && curl -s http://localhost:23119/api/ | head -c 100
  ```

After Zotero is running, trigger the initial export:
```bash
JSON_PATH="FILL_IN_JSON_PATH"
curl -s "http://localhost:23119/better-bibtex/export/library?translator=f4b52ab0-f878-4556-85a0-c7aeedd09dfc&exportNotes=false&output=$JSON_PATH" > /dev/null && echo "Export triggered" || echo "Export failed"
```

Verify the file was created:
```bash
python3 -c "import os; path='FILL_IN_JSON_PATH'; print('OK:', os.path.getsize(path), 'bytes') if os.path.exists(path) else print('NOT FOUND')"
```

---

## Step 7: Configure Obsidian plugin settings

Use `obsidian eval` to update plugin settings in the running Obsidian instance:

```bash
obsidian eval code="
const plugin = app.plugins.plugins['simple-citations'];
if (!plugin) throw new Error('simple-citations plugin not found or not enabled');
plugin.settings.jsonPath = 'FILL_IN_JSON_PATH';
plugin.settings.folderPath = 'FILL_IN_FOLDER';
plugin.settings.autoAddCitations = FILL_IN_BOOL;
plugin.settings.autoUpdateCitations = FILL_IN_BOOL;
plugin.settings.optionalFields = 'FILL_IN_OPTIONAL_FIELDS';
await plugin.saveSettings();
'Settings saved successfully';
"
```

Reload the plugin to apply:
```bash
obsidian plugin:reload simple-citations
```

---

## Step 8: Verify and report

Check the plugin data.json:
```bash
python3 -c "
import json
with open('VAULT_PATH/.obsidian/plugins/simple-citations/data.json') as f:
    s = json.load(f)
print('jsonPath:', s.get('jsonPath'))
print('folderPath:', s.get('folderPath'))
print('autoAddCitations:', s.get('autoAddCitations'))
print('optionalFields:', s.get('optionalFields'))
"
```

Report to the user:
- Zotero auto-export → `{json_path}`
- Postscript fields: `{chosen fields}`
- Literature notes folder: `{folder}`
- Auto-add on update: `{bool}`
- Optional fields: `{fields}`

Remind the user to run the **"Add literature note"** command in Obsidian once to import existing library items.
