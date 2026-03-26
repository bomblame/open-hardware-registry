# Open Hardware Registry Specification

## Overview

A community-curated directory of open hardware projects stored as individual JSON files. Consumed by BOMtrack's public instance to pre-populate available projects.

## Repository Structure

```
open-hardware-registry/
├── README.md
├── CONTRIBUTING.md
├── schema.json                 # JSON Schema for validation
├── projects/
│   ├── hackrf-one.json
│   ├── prusa-bear-upgrade.json
│   ├── esp32-wrover-kit.json
│   └── ...
└── .github/
    └── workflows/
        └── validate.yml        # PR validation
```

**One file per project** — cleaner PRs, no merge conflicts, easier review.

**Filename convention:** `kebab-case.json` derived from project name (becomes the ID).

---

## Schema

### Required Fields

```json
{
  "url": "https://github.com/greatscottgadgets/hackrf",
  "name": "HackRF One"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `url` | string | Repository URL (GitHub, GitLab, Gitea, etc.) |
| `name` | string | Human-readable project name |

That's it. Everything else is optional.

---

### Optional Fields

```json
{
  "url": "https://github.com/greatscottgadgets/hackrf",
  "name": "HackRF One",
  
  "description": "Software defined radio peripheral",
  "license": "GPL-2.0",
  "eda": "kicad",
  "path": "/hardware/hackrf-one",
  
  "host": "github",
  "branch": "master",
  
  "tags": ["sdr", "rf", "usb"],
  "website": "https://greatscottgadgets.com/hackrf/",
  
  "maintainer": {
    "name": "Great Scott Gadgets",
    "url": "https://greatscottgadgets.com"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `description` | string | Short project description |
| `license` | string | SPDX license identifier |
| `eda` | string | EDA tool hint: `kicad`, `altium`, `eagle`, `orcad`, `other` |
| `path` | string | Subdirectory containing design files (if not root) |
| `host` | string | Git host type: `github`, `gitlab`, `gitea`, `bitbucket`, `custom` |
| `branch` | string | Default branch to track (if not `main`/`master`) |
| `tags` | array | Searchable tags |
| `website` | string | Project homepage |
| `maintainer` | object | Maintainer info |

---

### Host Detection

If `host` is omitted, derive from URL:

| URL Pattern | Inferred Host |
|-------------|---------------|
| `github.com` | `github` |
| `gitlab.com` | `gitlab` |
| `codeberg.org` | `gitea` |
| `bitbucket.org` | `bitbucket` |
| Other | `custom` (may need `host` hint) |

For self-hosted instances (corporate GitLab, private Gitea), `host` field is required:

```json
{
  "url": "https://git.example.com/team/project",
  "name": "Internal Project",
  "host": "gitea"
}
```

---

## Validation Rules

1. `url` must be valid URL
2. `url` must be unique across all files
3. `name` must be non-empty string
4. `eda` if present must be one of: `kicad`, `altium`, `eagle`, `orcad`, `other`
5. `host` if present must be one of: `github`, `gitlab`, `gitea`, `bitbucket`, `custom`
6. `license` if present should be valid SPDX identifier

---

## JSON Schema (schema.json)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["url", "name"],
  "properties": {
    "url": {
      "type": "string",
      "format": "uri"
    },
    "name": {
      "type": "string",
      "minLength": 1
    },
    "description": {
      "type": "string"
    },
    "license": {
      "type": "string"
    },
    "eda": {
      "type": "string",
      "enum": ["kicad", "altium", "eagle", "orcad", "other"]
    },
    "path": {
      "type": "string"
    },
    "host": {
      "type": "string",
      "enum": ["github", "gitlab", "gitea", "bitbucket", "custom"]
    },
    "branch": {
      "type": "string"
    },
    "tags": {
      "type": "array",
      "items": { "type": "string" }
    },
    "website": {
      "type": "string",
      "format": "uri"
    },
    "maintainer": {
      "type": "object",
      "properties": {
        "name": { "type": "string" },
        "url": { "type": "string", "format": "uri" }
      }
    }
  },
  "additionalProperties": true
}
```

`additionalProperties: true` allows future expansion without breaking existing files.

---

## Example Files

### Minimal (hackrf-one.json)
```json
{
  "url": "https://github.com/greatscottgadgets/hackrf",
  "name": "HackRF One"
}
```

### Expanded (prusa-bear-upgrade.json)
```json
{
  "url": "https://github.com/gregsaun/prusa_i3_bear_upgrade",
  "name": "Prusa Bear Upgrade",
  "description": "Aluminum frame upgrade for Prusa i3 MK2/MK3",
  "license": "GPL-3.0",
  "eda": "other",
  "tags": ["3d-printer", "mechanical", "prusa"],
  "website": "https://github.com/gregsaun/prusa_i3_bear_upgrade/wiki"
}
```

### Self-hosted Gitea (internal-project.json)
```json
{
  "url": "https://git.company.com/hardware/sensor-board",
  "name": "Sensor Board",
  "host": "gitea",
  "path": "/pcb",
  "eda": "kicad"
}
```

---

## GitHub Actions Validation

```yaml
# .github/workflows/validate.yml
name: Validate Projects

on:
  pull_request:
    paths:
      - 'projects/*.json'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Validate JSON Schema
        uses: dsanders11/json-schema-validate-action@v1
        with:
          schema: schema.json
          files: projects/*.json
      
      - name: Check unique URLs
        run: |
          urls=$(jq -r '.url' projects/*.json | sort)
          dupes=$(echo "$urls" | uniq -d)
          if [ -n "$dupes" ]; then
            echo "Duplicate URLs found: $dupes"
            exit 1
          fi
```

---

## BOMtrack Integration

Public instance reads registry on startup/schedule:

```elixir
# Fetch registry index
def sync_registry do
  base = "https://raw.githubusercontent.com/yourorg/open-hardware-registry/main/projects"
  
  list_projects()
  |> Enum.each(fn filename ->
    {:ok, json} = fetch("#{base}/#{filename}")
    upsert_project(Jason.decode!(json))
  end)
end
```

Or clone the repo and read locally for faster access.

---

## Future Extensions

Schema allows additional fields. Potential additions:
- `variants` — list of board variants
- `bom_path` — explicit path to BOM file if non-standard
- `certifications` — OSHWA cert ID, CE, FCC
- `maturity` — `prototype`, `production`, `deprecated`

Add when needed, existing files remain valid.
