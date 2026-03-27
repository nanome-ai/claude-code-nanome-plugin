# Nanome Plugin for Claude Code

A Claude Code plugin for creating [Nanome](https://nanome.ai) molecular visualization workspaces. Load structures from RCSB PDB, AlphaFold, PubChem, ChEMBL, SWISS-MODEL, and COD, organize them into scenes with custom visualizations, and get a shareable workspace URL.

## What You Get

- **`nanome:nanome-auth`** -- Browser-based login using Nanome's device code flow. Opens your browser, displays a 4-character code, and saves your token automatically. No manual config needed.
- **`nanome:nanome-workspace`** -- High-level workspace builder. Describe what you want to visualize and it resolves structures, designs scenes, and builds the workspace end-to-end.

## Installation

```bash
claude --plugin-dir /path/to/nanome-plugin
```

Use `/reload-plugins` inside a session to pick up changes without restarting.

### Requirements

- **Python 3** (for the auth skill's WebSocket flow)
- The `websockets` Python package (auto-installed if missing)
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

The plugin handles auth automatically. When no valid token is found, the `nanome-auth` skill:

1. Opens your browser to `mara.nanome.ai`
2. Displays a 4-character login code in your terminal
3. You log in and click **"Login via Device Code"** in the left sidebar (headset icon)
4. Enter the code -- your token is saved to `~/.nanome/config.json`

To re-authenticate, just tell Claude:

> set up nanome auth

### Manual configuration

If you prefer to set up auth manually, create `~/.nanome/config.json`:

```json
{
  "api_base": "https://workspaces.nanome.ai",
  "token": "your-token-here"
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
