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

## Step 1: Check prerequisites

### Obsidian CLI
```bash
which obsidian || echo "NOT FOUND"
```
If not found, instruct the user to enable it in Obsidian: **Settings → General → Command line interface → Enable**, then stop.

### Zotero installation
```bash
ls /Applications/Zotero.app 2>/dev/null && echo "FOUND" || echo "NOT FOUND"
```
If not found, instruct the user to install Zotero from https://www.zotero.org.

### Better BibTeX installation
Find the Zotero profile directory:
```bash
ls ~/Library/Application\ Support/Zotero/Profiles/
```
Check for Better BibTeX in extensions:
```bash
PROFILE_DIR=$(ls ~/Library/Application\ Support/Zotero/Profiles/ | grep default | head -1)
grep -c "better-bibtex" ~/Library/Application\ Support/Zotero/Profiles/$PROFILE_DIR/extensions.json 2>/dev/null || echo "0"
```
If count is 0, instruct the user to install Better BibTeX for Zotero from https://retorque.re/zotero-better-bibtex/ and restart Zotero, then stop.

Store the profile path for later:
```bash
PROFILE_DIR=$(ls ~/Library/Application\ Support/Zotero/Profiles/ | grep default | head -1)
echo "Profile: $PROFILE_DIR"
```

---

## Step 2: Find Obsidian vaults

List available vaults:
```bash
cat ~/Library/Application\ Support/obsidian/obsidian.json 2>/dev/null | python3 -c "import json,sys; vaults=json.load(sys.stdin).get('vaults',{}); [print(v.get('path','')) for v in vaults.values()]"
```

---

## Step 3: Ask user for configuration

Use `AskUserQuestion` to collect the following. Show defaults clearly.

### Questions:

1. **Obsidian vault path**
   - Default: the first vault found in Step 2
   - Example: `/Users/you/Documents/MyVault`

2. **CSL JSON export path** (where Zotero will write the bibliography file)
   - Recommended: inside the vault root so Obsidian can detect changes
   - Default: `{vault}/MyLibrary.json`
   - Example: `/Users/you/Documents/MyVault/MyLibrary.json`

3. **Literature notes folder** (inside the vault)
   - Default: `Literatures`

4. **Auto-add literature notes when bibliography updates?**
   - Recommended: `true` (fully automatic workflow)

5. **Auto-update literature notes when file opens?**
   - Default: `false` (can cause slowness with large libraries)

6. **Postscript for Better BibTeX** (adds extra fields to each item)
   - Show these options and let user choose (multiple allowed):
     - `[key]` — Zotero item key (needed for Zotero URI links in notes)
     - `[pdf]` — Local PDF file path
     - `[collections]` — Collection names the item belongs to
     - `[none]` — No postscript

7. **Optional fields to show in notes** (must match postscript choices)
   - Only ask if user chose `[key]`, `[pdf]`, or `[collections]`
   - Pre-fill based on postscript choices (e.g. `key` → `key`, `pdf` → `pdf`)

---

## Step 4: Configure Zotero via user.js

### 4-1: Build the postscript

Based on user's choices in Step 3, construct the postscript content:

```javascript
// Base (always include if any field is selected)
if (Translator.BetterCSLJSON) {
  // Add lines based on choices:
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

### 4-2: URL-encode the JSON path

```bash
JSON_PATH="/path/to/library.json"  # replace with actual path from Step 3
python3 -c "import urllib.parse; print(urllib.parse.quote('$JSON_PATH', safe=''))"
```

### 4-3: Generate timestamp

```bash
python3 -c "import time; print(int(time.time() * 1000))"
```

### 4-4: Write user.js

Write (or append to) `user.js` in the Zotero profile directory. The `user.js` file is read by Zotero on startup and safely overrides `prefs.js` without modifying it.

```bash
PROFILE_DIR=$(ls ~/Library/Application\ Support/Zotero/Profiles/ | grep default | head -1)
PREFS_PATH=~/Library/Application\ Support/Zotero/Profiles/$PROFILE_DIR/user.js
```

Write the following entries to `user.js`. If the file already exists, read it first and replace existing `simple-citations-setup` block rather than appending.

Use this format for the file content (replace placeholders):

```
// === simple-citations-setup ===
// Enable local HTTP API
user_pref("extensions.zotero.httpServer.localAPI.enabled", true);

// Better BibTeX postscript
user_pref("extensions.zotero.translators.better-bibtex.postscript", "POSTSCRIPT_ESCAPED");

// Better BibTeX auto-export
user_pref("extensions.zotero.translators.better-bibtex.autoExport.URL_ENCODED_PATH", "JSON_VALUE");
// === end simple-citations-setup ===
```

Where:
- `POSTSCRIPT_ESCAPED`: the postscript from 4-1, with `\n` for newlines and `\"` for quotes (JSON string escaping)
- `URL_ENCODED_PATH`: output from 4-2
- `JSON_VALUE`: JSON string with this structure (escape the whole thing as a pref string):
  ```json
  {"created":TIMESTAMP,"path":"FULL_JSON_PATH","translatorID":"f4b52ab0-f878-4556-85a0-c7aeedd09dfc","type":"library","id":1,"status":"idle","error":"","recursive":false,"updated":TIMESTAMP,"enabled":true}
  ```

Use Python to write the file correctly:
```bash
python3 << 'PYEOF'
import json, time, urllib.parse, os, re

profile_dir = os.path.expanduser(
    "~/Library/Application Support/Zotero/Profiles/"
    + sorted(os.listdir(os.path.expanduser("~/Library/Application Support/Zotero/Profiles/")))[0]
)
user_js_path = os.path.join(profile_dir, "user.js")

json_path = "FILL_IN_PATH"  # from Step 3
postscript = "FILL_IN_POSTSCRIPT"  # from Step 4-1

ts = int(time.time() * 1000)
encoded_path = urllib.parse.quote(json_path, safe='')
auto_export_val = json.dumps({"created": ts, "path": json_path,
    "translatorID": "f4b52ab0-f878-4556-85a0-c7aeedd09dfc",
    "type": "library", "id": 1, "status": "idle", "error": "",
    "recursive": False, "updated": ts, "enabled": True})

block = f"""
// === simple-citations-setup ===
user_pref("extensions.zotero.httpServer.localAPI.enabled", true);
user_pref("extensions.zotero.translators.better-bibtex.postscript", {json.dumps(postscript)});
user_pref("extensions.zotero.translators.better-bibtex.autoExport.{encoded_path}", {json.dumps(auto_export_val)});
// === end simple-citations-setup ===
"""

# Read existing file and replace block if present
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

## Step 5: Check if Zotero is running

```bash
curl -s http://localhost:23119/api/ 2>/dev/null && echo "RUNNING" || echo "NOT RUNNING"
```

- If **RUNNING**: Inform the user that Zotero must be **restarted** for the `user.js` settings to take effect. Ask them to restart Zotero now before continuing.
- If **NOT RUNNING**: Start Zotero:
  ```bash
  open -a Zotero
  ```
  Wait a moment, then verify the HTTP API is now available:
  ```bash
  sleep 5 && curl -s http://localhost:23119/api/ | head -c 100
  ```

After Zotero is running with the new settings, trigger the export via the HTTP API:
```bash
JSON_PATH="FILL_IN_PATH"
curl -s "http://localhost:23119/better-bibtex/export/library?translator=f4b52ab0-f878-4556-85a0-c7aeedd09dfc&exportNotes=false&output=$JSON_PATH" > /dev/null && echo "Export triggered" || echo "Export failed"
```

Then verify the file was created:
```bash
ls -lh "FILL_IN_JSON_PATH"
```

---

## Step 6: Configure Obsidian plugin settings

Build the optional fields string (newline-separated, based on user's choices):

```bash
# Example if user chose key + pdf:
OPTIONAL_FIELDS="key\npdf"
```

Use `obsidian eval` to update plugin settings. The `app.plugins.plugins` map uses the plugin ID `simple-citations`:

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

## Step 7: Verify and report

Check the plugin data.json to confirm settings were applied:
```bash
cat "VAULT_PATH/.obsidian/plugins/simple-citations/data.json"
```

Report to the user:
- Zotero auto-export configured to: `{json_path}`
- Postscript fields: `{chosen fields}`
- Literature notes folder: `{folder}`
- Auto-add on update: `{bool}`
- Auto-update on open: `{bool}`
- Optional fields: `{fields}`

If `autoAddCitations` is true, the user can now add items to Zotero and they will automatically appear as literature notes in Obsidian.
Remind the user to run **"Add literature note"** command in Obsidian once to populate existing items.
