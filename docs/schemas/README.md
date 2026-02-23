# UAAPS Schemas

JSON Schema ([2020-12](https://json-schema.org/draft/2020-12/json-schema-core)) files for programmatic validation of all UAAPS-defined structured files.

> **Important**: The prose specification is authoritative. These JSON Schema files are **informational aids** for validation. In case of conflict between a schema and the prose spec, the prose governs.

## Files That Require a Schema

| File / Pattern | Schema | Format | Notes |
|---|---|---|---|
| `package.agent.json` | [`package-manifest.schema.json`](package-manifest.schema.json) | JSON | Required in every package |
| `package.agent.yaml` | [`package-manifest.schema.json`](package-manifest.schema.json) | YAML | Same schema; YAML variant |
| `package.agent.lock` | [`package-lock.schema.json`](package-lock.schema.json) | JSON | Auto-generated; commit to VCS |
| `hooks/hooks.json` | [`hooks.schema.json`](hooks.schema.json) | JSON | Lifecycle event handlers |
| `mcp/servers.json` | [`mcp-servers.schema.json`](mcp-servers.schema.json) | JSON | MCP server configuration |
| `agents/<name>/agent.yaml` | [`agent.schema.json`](agent.schema.json) | YAML | Agent definition |
| `agents/<name>/system-prompt.md` *(frontmatter)* | [`system-prompt-frontmatter.schema.json`](system-prompt-frontmatter.schema.json) | YAML frontmatter | System-prompt metadata block |
| `skills/<name>/SKILL.md` *(frontmatter)* | [`skill-frontmatter.schema.json`](skill-frontmatter.schema.json) | YAML frontmatter | Skill metadata block |
| `rules/<name>/RULE.md` *(frontmatter)* | [`rule-frontmatter.schema.json`](rule-frontmatter.schema.json) | YAML frontmatter | Rule metadata block |
| `commands/<name>.md` *(frontmatter)* | [`command-frontmatter.schema.json`](command-frontmatter.schema.json) | YAML frontmatter | Command metadata block (including parameterized templates) |
| `dist/<name>.bundle.aam` → `bundle.json` (internal) | [`bundle.schema.json`](bundle.schema.json) | JSON | Portable bundle manifest |

## Usage

### CLI validation (ajv)
```bash
npx ajv validate -s docs/schemas/package-manifest.schema.json -d package.agent.json --spec=draft2020
```

### Python (jsonschema)
```python
import json, jsonschema, pathlib

schema = json.loads(pathlib.Path("docs/schemas/package-manifest.schema.json").read_text())
instance = json.loads(pathlib.Path("package.agent.json").read_text())
jsonschema.validate(instance, schema)
```

### YAML frontmatter
Extract the frontmatter block between `---` delimiters, parse it as YAML, then validate the resulting object against the relevant `*-frontmatter.schema.json`.

```python
import yaml, jsonschema, pathlib

def load_frontmatter(md_path):
    text = pathlib.Path(md_path).read_text()
    parts = text.split("---", 2)
    return yaml.safe_load(parts[1]) if len(parts) >= 3 else {}

schema = json.loads(pathlib.Path("docs/schemas/skill-frontmatter.schema.json").read_text())
instance = load_frontmatter("skills/my-skill/SKILL.md")
jsonschema.validate(instance, schema)
```

### VS Code integration
Add to `.vscode/settings.json` for in-editor validation:
```json
{
  "json.schemas": [
    {
      "fileMatch": ["**/package.agent.json"],
      "url": "./docs/schemas/package-manifest.schema.json"
    },
    {
      "fileMatch": ["**/package.agent.lock"],
      "url": "./docs/schemas/package-lock.schema.json"
    },
    {
      "fileMatch": ["**/hooks/hooks.json"],
      "url": "./docs/schemas/hooks.schema.json"
    },
    {
      "fileMatch": ["**/mcp/servers.json"],
      "url": "./docs/schemas/mcp-servers.schema.json"
    },
    {
      "fileMatch": ["**/agents/*/agent.yaml"],
      "url": "./docs/schemas/agent.schema.json"
    },
    {
      "fileMatch": ["**/bundle.json"],
      "url": "./docs/schemas/bundle.schema.json"
    }
  ]
}
```
