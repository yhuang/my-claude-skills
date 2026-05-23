# my-claude-skills

Personal [Claude Code](https://claude.ai/code) plugin marketplace with three skills.

| Skill | Trigger | Purpose |
|-------|---------|---------|
| `doc-sync` | `/doc-sync` | Verify every `file:line` reference and code example in markdown docs matches current source |
| `code-doc-audit` | `/code-doc-audit` | Audit docs/comments for concept, design, and implementation accuracy (does not check file:line — pair with `doc-sync` for that) |
| `ship` | `/ship` | Commit pending changes, push, and open a GitHub PR whose body summarizes all commits since main |

## Install in a project

### From GitHub (once the repo is public)

Add to your project's `.claude/settings.json`:

```json
{
  "enabledPlugins": {
    "doc-sync@my-claude-skills": true,
    "code-doc-audit@my-claude-skills": true,
    "ship@my-claude-skills": true
  },
  "extraKnownMarketplaces": {
    "my-claude-skills": {
      "source": {
        "source": "github",
        "repo": "yhuang/my-claude-skills",
        "ref": "main"
      }
    }
  }
}
```

### From a local directory (during development)

```json
{
  "enabledPlugins": {
    "doc-sync@my-claude-skills": true,
    "code-doc-audit@my-claude-skills": true,
    "ship@my-claude-skills": true
  },
  "extraKnownMarketplaces": {
    "my-claude-skills": {
      "source": {
        "source": "directory",
        "path": "/path/to/my-claude-skills"
      }
    }
  }
}
```

## Repository structure

```
.claude-plugin/
  marketplace.json   ← marketplace index (lists all plugins)
  plugin.json        ← top-level plugin metadata
doc-sync/
  .claude-plugin/plugin.json
  skills/doc-sync/SKILL.md
code-doc-audit/
  .claude-plugin/plugin.json
  skills/code-doc-audit/SKILL.md
ship/
  .claude-plugin/plugin.json
  skills/ship/SKILL.md
```
