# Nanome Plugin for Claude Code

A Claude Code plugin for creating [Nanome](https://nanome.ai) molecular visualization workspaces. Load structures from RCSB PDB, AlphaFold, PubChem, ChEMBL, SWISS-MODEL, and COD, organize them into scenes with custom visualizations, and get a shareable workspace URL.

## What You Get

- **`nanome:nanome-auth`** -- Opens your browser to create an API key at mara.nanome.ai/settings and saves it automatically. No manual config needed.
- **`nanome:nanome-workspace`** -- High-level workspace builder. Describe what you want to visualize and it resolves structures, builds a scene with sensible defaults, and delivers a shareable workspace URL. Creates one focused scene by default; ask for multiple if needed.

## Installation

```bash
claude --plugin-dir /path/to/nanome-plugin
```

Use `/reload-plugins` inside a session to pick up changes without restarting.

### Requirements

- **Python 3** and **curl** (for API calls and config management)
- A [Nanome](https://mara.nanome.ai) account (free to create)

## Quick Start

Just ask Claude to create a workspace:

```
Create a Nanome workspace with PDB 1IEP
```

If you're not authenticated, the plugin will automatically open your browser and walk you through login. After that, the workspace is built and you get a URL like:

```
https://mara.nanome.ai/workspaces/abc123-...
```


## Authentication

The plugin handles auth automatically. When no valid API key is found, the `nanome-auth` skill:

1. Opens your browser to `mara.nanome.ai/settings`
2. You log in, go to the **System** tab, and click **Create** under API Keys
3. The key is auto-copied to your clipboard
4. Paste it back in Claude -- it's saved to `~/.nanome/config.json`

API keys are long-lived, so you rarely need to re-authenticate. To set up auth manually or re-authenticate, tell Claude:

> set up nanome auth

### Manual configuration

If you prefer to set up auth manually, create `~/.nanome/config.json`:

```json
{
  "api_base": "https://workspaces.nanome.ai",
  "token": "your-api-key-here"
}
```

Environment variables `NANOME_API_BASE` and `NANOME_TOKEN` override the config file.

## Plugin Structure

```
nanome-plugin/
  .claude-plugin/
    plugin.json            # Plugin manifest
  skills/
    nanome-auth/
      SKILL.md             # Authentication skill
    nanome-workspace/
      SKILL.md             # Workspace creation skill
  README.md
```

## Supported Structure Databases

| Database | Identifier | Example |
|----------|-----------|---------|
| RCSB PDB | 4-char PDB code | `1HVR`, `6LU7`, `5CEO` |
| AlphaFold | UniProt accession | `P00533`, `Q9Y6K9` |
| PubChem | CID number | `2244` (aspirin) |
| ChEMBL | ChEMBL ID | `CHEMBL25` |
| SWISS-MODEL | UniProt accession | `P00533` |
| COD | COD ID | `1000000` |

## Visualization Capabilities

The workspace skill supports these visualization types:

- **Ribbon/Cartoon** -- secondary structure visualization
- **Ball-and-stick** -- atomic detail with bonds
- **Molecular surface** -- solvent-accessible surface with adjustable opacity
- **Wire** -- minimal bond representation
- **Van der Waals** -- space-filling spheres
- **Residue labels** -- text annotations
- **Interactions** -- hydrogen bonds, hydrophobic contacts, salt bridges, pi-stacking, and more

Coloring options include element type, chain, B-factor, secondary structure, residue type, and custom uniform colors.

## License

MIT
