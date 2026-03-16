# CLAUDE.md

## README Skills Table

When adding a new skill, regenerate the skills table in README.md using:

```bash
python3 -c "
import json
with open('.claude-plugin/marketplace.json') as f:
    data = json.load(f)
print('| Name | Description |')
print('|------|-------------|')
for p in data['plugins']:
    print(f'| {p[\"name\"]} | {p[\"description\"]} |')
"
```

Then replace the table in README.md with the output.
