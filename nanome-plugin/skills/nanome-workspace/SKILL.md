---
name: nanome-workspace
description: Generate Nanome workspaces and scenes with molecular structures from RCSB PDB, AlphaFold, PubChem, ChEMBL, SWISS-MODEL, COD, and local files. Use this skill whenever the user wants to create a Nanome workspace, set up molecular visualization sessions, load structures into Nanome, organize molecular data into scenes, or describe a drug discovery/biochemistry session in Nanome. Trigger on any mention of Nanome + workspaces, scenes, molecules, PDB IDs, RCSB structures, or molecular visualization setup.
---

# Nanome Workspace Creation Skill

This skill creates NEW Nanome workspaces populated with molecular structures, organized into scenes with custom visualization components. It is the "on-ramp" for Nanome — get a workspace up and running quickly with the right structures, sensible scene organization, and useful default views. Once created, hand off to MARA (the in-session AI assistant) for interactive refinement, selection-based edits, and live exploration.

This skill does NOT edit existing workspaces. It generates fresh workspaces from scratch based on user intent.

## Workflow

The user triggering this skill IS the go-ahead. Do NOT ask for confirmation — resolve, build, execute, and deliver the workspace URL. Follow these four steps without interruption.

### Step 1: Understand Intent & Resolve Structures

Parse what the user wants to visualize and why. Infer aggressively:

- **What structures?** Specific PDB IDs, UniProt accessions, compound names, file paths, or vague descriptions ("the ABL kinase with imatinib").
- **What purpose?** Drug discovery (binding site analysis, SAR), structural comparison, teaching/education, publication figure, general exploration.

**Decision point — ask vs infer:**
- If the user gives specific IDs (e.g., "make a workspace with 1HVR and 6LU7"), proceed directly.
- If the user names a well-known target (e.g., "EGFR with gefitinib"), infer the best PDB entry and proceed.
- **Only ask** if the request is genuinely ambiguous (e.g., "set up something for kinase research" with no target specified). Keep it to 1 question max.

Immediately resolve identifiers to concrete database sources and fetch metadata (chain count, ligands, resolution) to inform scene design.

### Step 2: Design Scenes

Design 2-4 scenes with appropriate visualization components using the heuristics below. Do not present for approval — proceed directly to script generation.

- **Structure type:** Proteins get ribbon + surface; small molecules get ball-and-stick; complexes get ribbon for protein + ball-and-stick for ligand.
- **User purpose:** Drug discovery emphasizes binding sites and ligand interactions; teaching emphasizes full structure overviews; comparison emphasizes aligned views.
- **Structure count:** Single structure gets more detailed per-scene breakdown; many structures get overview scenes with grouped entries.

**Decision point — `add-default` vs custom components:**
- Use `add-default` when the user has no specific visualization preferences and the structure type has clear defaults.
- Use custom components when the user specifies representation preferences, when mixing structure types, or when the purpose demands specific views.

### Step 3: Build Workspace

Detect the execution environment and use the appropriate mode:

1. **Bash tool available** → **Script Mode** — generate and execute a bash script with curl commands (see Script Template below)
2. **`nanome_*` MCP tools available** → **MCP Mode** — call MCP tools sequentially (see MCP Mode below)
3. **Neither** → **Artifact Mode** — generate a self-contained HTML artifact the user runs in their browser (see Artifact Template below)

Detection is implicit — check which tools are in your tool list. Do not ask the user which mode to use.

### Step 4: Deliver Results

After execution completes (or after generating the artifact), present:
1. The workspace URL: `https://mara.nanome.ai/workspaces/{id}`
2. A brief summary of what was built (structures, scenes)
3. MARA follow-up suggestions tailored to the workspace content

**Artifact Mode note:** The workspace URL is shown immediately in the artifact UI after the user clicks "Build Workspace". In your text response, explain that the artifact will create the workspace when they click the button.

---

## Authentication

Auth must NEVER block workspace creation. Build the workspace first; handle auth issues after if they arise.

### Config File

Authentication credentials are stored in `~/.nanome/config.json`:

```json
{
  "api_base": "https://workspaces.nanome.ai",
  "token": "your-nanome-auth-token"
}
```

### Auth Resolution (Non-Blocking)

1. **Read config** — check `~/.nanome/config.json` for `api_base` and `token`.
2. **If config exists** — use the key. Proceed with workspace creation.
3. **If config is missing** — invoke the `nanome-auth` skill to authenticate the user via the browser-based device code flow. This opens their browser, displays a 4-character code, and saves the token automatically. After auth completes, proceed with workspace creation.
4. **If an API call fails with 401/403** — the token may have expired. Invoke the `nanome-auth` skill to re-authenticate, then retry the failed operation.
5. **Never interrupt the build process to ask about auth** — the auth skill handles everything automatically.

### Script Auth Template

Every generated script must include this auth setup at the top:

```bash
# Load auth config
CONFIG_FILE="$HOME/.nanome/config.json"
API_BASE="https://workspaces.nanome.ai"
TOKEN=""

if [ -f "$CONFIG_FILE" ]; then
  API_BASE=$(python3 -c "import json; print(json.load(open('$CONFIG_FILE')).get('api_base', '$API_BASE'))" 2>/dev/null || echo "$API_BASE")
  TOKEN=$(python3 -c "import json; print(json.load(open('$CONFIG_FILE')).get('token', ''))" 2>/dev/null || echo "")
fi

AUTH_ARGS=()
if [ -n "$TOKEN" ]; then
  AUTH_ARGS=(-H "Authorization: Bearer $TOKEN")
fi
```

All curl commands in the script must include `"${AUTH_ARGS[@]}"` to conditionally pass auth:

```bash
curl -s "${AUTH_ARGS[@]}" -X POST "$API_BASE/workspaces" ...
```

### URL Reference

- **Workspace API (backend):** `https://workspaces.nanome.ai` — all REST API calls go here
- **Workspace URL (user-facing):** `https://mara.nanome.ai/workspaces/{id}` — this is the link you give to users
- **MARA login:** `https://mara.nanome.ai/login` — for browser-based auth
- **API discovery:** `https://mara.nanome.ai/info` — returns `workspace_api_url` dynamically

---

## Structure Resolution

### Supported Databases

| Source | Identifier Format | Download URL | Output Format |
|--------|------------------|-------------|---------------|
| RCSB PDB | 4-char code (e.g., `1HVR`) | `https://files.rcsb.org/download/{ID}.pdb` | PDB |
| RCSB PDB (large) | 4-char code | `https://files.rcsb.org/download/{ID}.cif` | CIF |
| AlphaFold DB | UniProt ID (e.g., `P00533`) | `https://alphafold.ebi.ac.uk/files/AF-{ID}-F1-model_v4.pdb` | PDB (validate v4 via HEAD) |
| PubChem | CID (e.g., `2244`) | `https://pubchem.ncbi.nlm.nih.gov/rest/pug/compound/CID/{ID}/SDF` | SDF |
| ChEMBL | ChEMBL ID (e.g., `CHEMBL25`) | `https://www.ebi.ac.uk/chembl/api/data/molecule/{ID}.sdf` | SDF |
| SWISS-MODEL | UniProt ID | `https://swissmodel.expasy.org/repository/uniprot/{ID}.pdb` | PDB |
| COD | COD ID (e.g., `1000000`) | `https://www.crystallography.net/cod/{ID}.cif` | CIF |
| Local files | File path | Direct upload | PDB/SDF/CIF/PQR |

### Resolution Logic

When the user provides an identifier or description, apply these pattern-matching rules in order:

1. **4-character alphanumeric** (e.g., `1HVR`, `6LU7`) — treat as RCSB PDB ID. Use CIF instead of PDB when the structure has more than 62 chains (PDB format limitation).
2. **UniProt accession** — matches pattern `[OPQ][0-9][A-Z0-9]{3}[0-9]` or `[A-NR-Z][0-9]([A-Z][A-Z0-9]{2}[0-9]){1,2}` (e.g., `P00533`, `Q9Y6K9`) — route to AlphaFold DB first. Fall back to SWISS-MODEL if AlphaFold has no prediction.
3. **"CID" prefix or bare number in small-molecule context** (e.g., `CID 2244`, or user says "aspirin, CID 2244") — PubChem.
4. **"CHEMBL" prefix** (e.g., `CHEMBL25`, `CHEMBL941`) — ChEMBL.
5. **"COD" prefix or crystallography context** (e.g., `COD 1000000`) — COD.
6. **File path** (starts with `/`, `./`, `~`, or contains common extensions `.pdb`, `.sdf`, `.cif`, `.pqr`) — local file upload.
7. **Natural language** (e.g., "ABL kinase", "aspirin", "insulin receptor") — search via APIs:
   - For proteins: `https://search.rcsb.org/rcsbsearch/v2/query` with a text search query.
   - For small molecules: `https://pubchem.ncbi.nlm.nih.gov/rest/pug/compound/name/{NAME}/cids/JSON`
   - Present top results and let the user confirm before proceeding.

### Metadata Fetch

Before designing scenes, fetch metadata to understand structure contents:

**RCSB PDB:**
```bash
curl -s "https://data.rcsb.org/rest/v1/core/entry/{ID}"
```
Key fields: `rcsb_entry_info.resolution_combined`, `rcsb_entry_info.polymer_entity_count`, `struct.title`, entity descriptions, ligand identifiers.

**PubChem:**
```bash
curl -s "https://pubchem.ncbi.nlm.nih.gov/rest/pug/compound/CID/{ID}/property/MolecularFormula,MolecularWeight,IUPACName/JSON"
```
Key fields: `MolecularFormula`, `MolecularWeight`, `IUPACName`.

**AlphaFold:**
```bash
curl -s -I "https://alphafold.ebi.ac.uk/files/AF-{ID}-F1-model_v4.pdb"
```
A `200` response confirms the prediction exists at version 4. If not, try earlier versions or fall back to SWISS-MODEL.

### Download Validation Helper

Include this function in every generated script to handle downloads with error checking:

```bash
download_structure() {
  local url="$1" output="$2"
  local http_code
  http_code=$(curl -s -o "$output" -w "%{http_code}" "$url")
  if [ "$http_code" != "200" ] || [ ! -s "$output" ]; then
    echo "ERROR: Failed to download $url (HTTP $http_code)" >&2
    rm -f "$output"
    return 1
  fi
  echo "Downloaded: $output"
}
```

Use this for every structure download in the script:
```bash
download_structure "https://files.rcsb.org/download/1HVR.pdb" "$WORKDIR/1HVR.pdb" || exit 1
```

---

## Component Schema

Components define how atoms from a structure are filtered and visualized within a scene. Each component is a JSON object sent individually via:

```
POST /workspaces/{id}/scenes/{serial}/components/add
```

**One component per call** — do NOT send an array. Each call adds a single component.

### Top-Level Structure

```json
{
  "name": "Protein",
  "filters": [
    { "kind": "EntrySerial", "entrySerial": 1 },
    { "kind": "CurrentModel" },
    { "kind": "Substructure", "substructureType": "Protein" }
  ],
  "visualizations": [
    {
      "kind": "Ribbon",
      "visible": true,
      "ribbonStyle": "Cartoon",
      "size": { "sizeOption": "Uniform", "scale": 1.0 },
      "coloring": {
        "kind": "Property",
        "carbonsOnly": false,
        "propertySource": "ChainInstance"
      }
    }
  ]
}
```

### Key Structural Rules

- **`visible`** (boolean, required) — present on every visualization object. Default `true`.
- **`size`** (object, **required on ALL visualization types**) — `{ "sizeOption": "NoSize"|"Uniform"|"Physical"|"Disorder", "scale": float }`. Omitting `size` causes Internal Server Error. For Ribbon and MolecularSurface, use `{ "sizeOption": "Uniform", "scale": 1.0 }` as default.
- **`coloring`** is **nested inside each visualization**, NOT a peer field on the component. Every visualization has its own `coloring` object.
- **`carbonsOnly`** (boolean, required) — present on every coloring object. Default `false`. When `true`, only carbon atoms use the specified coloring scheme; other elements use element-type coloring.
- **Hiding components** — `ComponentInput` does not have a `hidden` field. To hide a component after creation, use a separate call: `POST /workspaces/{id}/scenes/{serial}/components/toggle-hidden` with body `{ "componentSerials": [N], "hidden": true }`. Use this for solvent/ion components that should be hidden by default.

### Filter Types Reference

Filters narrow which atoms a component targets. The `filters` array on a component applies all filters together (intersection by default).

**EntrySerial** — select atoms from a specific loaded structure by its entry serial number:
```json
{ "kind": "EntrySerial", "entrySerial": 1 }
```

**CurrentModel** — select atoms from the currently active model (for multi-model structures like NMR ensembles):
```json
{ "kind": "CurrentModel" }
```

**ModelSerials** — select specific models by serial number:
```json
{ "kind": "ModelSerials", "serials": [1, 2, 3, 5] }
```

**ChainNames** — select specific chains by name:
```json
{ "kind": "ChainNames", "names": ["A", "B"] }
```

**ResidueNames** — select residues by 3-letter code:
```json
{ "kind": "ResidueNames", "names": ["ALA", "TRP"] }
```

**ResidueSerials** — select residues by serial number:
```json
{ "kind": "ResidueSerials", "serials": [1, 2, 3, 100, 150] }
```

**AtomNames** — select atoms by name:
```json
{ "kind": "AtomNames", "names": ["CA", "N", "O"] }
```

**EnumeratedAtoms** — select atoms by serial number:
```json
{ "kind": "EnumeratedAtoms", "serials": [1, 2, 3, 10, 50] }
```

**ConnectedResidue** — select residues connected to the current selection:
```json
{ "kind": "ConnectedResidue" }
```

**Substructure** — select atoms by macromolecular substructure type:
```json
{ "kind": "Substructure", "substructureType": "Protein" }
```
Valid `substructureType` values: `Protein`, `DNA`, `RNA`, `PNA`, `Ligand`, `Saccharide`, `Ion`, `Solvent`, `Nucleic`, `Polymer`.

**Distance** — select atoms within a distance (angstroms) of atoms matching `containedFilters`:
```json
{
  "kind": "Distance",
  "distance": 5.0,
  "wholeResidue": true,
  "ignoreSelf": true,
  "containedFilters": [
    { "kind": "Substructure", "substructureType": "Ligand" }
  ]
}
```
- `distance` — radius in angstroms.
- `wholeResidue` — if `true`, include the entire residue when any atom is within distance.
- `ignoreSelf` — if `true`, exclude the atoms that define the distance center.
- `containedFilters` — array of filters defining the center atoms.

**Union** — atoms matching ANY child filter:
```json
{
  "kind": "Union",
  "containedFilters": [
    { "kind": "Substructure", "substructureType": "Ligand" },
    { "kind": "Substructure", "substructureType": "Ion" }
  ]
}
```

**Intersect** — atoms matching ALL child filters:
```json
{
  "kind": "Intersect",
  "containedFilters": [
    { "kind": "ChainNames", "names": ["A"] },
    { "kind": "ResidueSerials", "serials": { "ranges": ["50-100"], "sets": [] } }
  ]
}
```

**Except** — exclude atoms matching child filters:
```json
{
  "kind": "Except",
  "containedFilters": [
    { "kind": "Substructure", "substructureType": "Solvent" }
  ]
}
```

### Compound Filter Example

Binding site residues within 5A of a ligand:
```json
{
  "kind": "Intersect",
  "containedFilters": [
    { "kind": "EntrySerial", "entrySerial": 1 },
    { "kind": "CurrentModel" },
    {
      "kind": "Distance",
      "distance": 5.0,
      "wholeResidue": true,
      "ignoreSelf": true,
      "containedFilters": [
        { "kind": "Substructure", "substructureType": "Ligand" }
      ]
    }
  ]
}
```

---

## Visualization Types

Each entry in the `visualizations` array on a component defines a rendering style. All visualizations require `visible` and `coloring`.

| Kind | Required Properties |
|------|-------------------|
| Atomistic | `visible`, `size`, `coloring`, `renderAtom` (All/NonBondedOnly/None), `renderBond` (bool), `bondScale` (float). Optional: `renderHydrogen` (None/PolarOnly/All) |
| MolecularSurface | `visible`, `size`, `coloring`, `opacity` (0-1), `subsurface` (bool). Optional: `wireframe` (bool) |
| Ribbon | `visible`, `size`, `coloring`, `ribbonStyle` (Cartoon/Coil) |
| ResidueLabel | `visible`, `size`, `coloring` |
| Interaction | `visible`, `size`, `coloring`, `interactionType`, `bondScale` (float), `gapScale` (float) |

**IMPORTANT: `size` is required on EVERY visualization type.** Omitting it causes HTTP 500. Use `{"sizeOption":"Uniform","scale":1.0}` as the default for Ribbon, MolecularSurface, and ResidueLabel.

---

## Coloring Types

Coloring is always nested inside each visualization object. Every coloring object requires `carbonsOnly`.

| Kind | Required Properties |
|------|-------------------|
| ElementType | `carbonsOnly` |
| Property | `carbonsOnly`, `propertySource` (BFactor/Occupancy/ChainName/ChainInstance/AtomSerial/AtomName/ModelInstance). Optional: `paletteType` (Default/Interpolated) + `colors` (hex array) |
| ResidueType | `carbonsOnly` |
| SecondaryStructure | `carbonsOnly` |
| Uniform | `carbonsOnly`, `uniformColor` (hex string e.g. `"00FF00"`) |
| Enumerated | `carbonsOnly`, `colors` (array of hex strings) |
| HydrogenBonding | `carbonsOnly` |
| Electrostatic | `carbonsOnly` |

---

## Interaction Type Enum Values

Use these EXACT API enum values — they differ from common shorthand:

| Interaction | API Enum Value |
|-------------|---------------|
| Hydrogen bonds | `HydrogenBonds` |
| Hydrogen bonds (polymer-polymer) | `HydrogenBondsPolymerPolymer` |
| Hydrophobic | `Hydrophobic` |
| Salt bridges | `SaltBridges` |
| Pi-cation | `PiCation` |
| Pi stacking | `PiStacking` |
| Halogen | `Halogen` |
| Metal coordination | `MetalCoordination` |
| Clash | `Clash` |

---

## Visualization Presets

Copy-paste-ready visualization JSON objects for common representation styles.

### Ball and Stick
```json
{ "kind": "Atomistic", "visible": true, "size": { "sizeOption": "Physical", "scale": 0.25 }, "coloring": { "kind": "Property", "carbonsOnly": true, "propertySource": "ChainInstance" }, "renderAtom": "All", "renderBond": true, "bondScale": 0.2 }
```

### Stick
```json
{ "kind": "Atomistic", "visible": true, "size": { "sizeOption": "Uniform", "scale": 0.2 }, "coloring": { "kind": "Property", "carbonsOnly": true, "propertySource": "ChainInstance" }, "renderAtom": "All", "renderBond": true, "bondScale": 0.2 }
```

### Wire
```json
{ "kind": "Atomistic", "visible": true, "size": { "sizeOption": "Uniform", "scale": 0.1 }, "coloring": { "kind": "Property", "carbonsOnly": true, "propertySource": "ChainInstance" }, "renderAtom": "NonBondedOnly", "renderBond": true, "bondScale": 0.05 }
```

### Van der Waals
```json
{ "kind": "Atomistic", "visible": true, "size": { "sizeOption": "Physical", "scale": 1.0 }, "coloring": { "kind": "Property", "carbonsOnly": true, "propertySource": "ChainInstance" }, "renderAtom": "All", "renderBond": false, "bondScale": 0 }
```

### Cartoon
```json
{ "kind": "Ribbon", "visible": true, "size": { "sizeOption": "Uniform", "scale": 1.0 }, "coloring": { "kind": "Property", "carbonsOnly": false, "propertySource": "ChainInstance" }, "ribbonStyle": "Cartoon" }
```

### Surface
```json
{ "kind": "MolecularSurface", "visible": true, "size": { "sizeOption": "Uniform", "scale": 1.0 }, "coloring": { "kind": "Property", "carbonsOnly": false, "propertySource": "ChainInstance" }, "opacity": 0.5, "subsurface": false }
```

### Residue Label
```json
{ "kind": "ResidueLabel", "visible": true, "size": { "sizeOption": "Uniform", "scale": 1.0 }, "coloring": { "kind": "Uniform", "carbonsOnly": false, "uniformColor": "FFFFFF" } }
```

---

## Coloring Presets

### Cold to Warm (B-Factor)
```json
{ "kind": "Property", "carbonsOnly": false, "propertySource": "BFactor", "paletteType": "Interpolated", "colors": ["0000FF", "7F7FFF", "FFFFFF", "FF7F7F", "FF0000"] }
```

### Rainbow (by Atom Serial)
```json
{ "kind": "Property", "carbonsOnly": false, "propertySource": "AtomSerial", "paletteType": "Interpolated", "colors": ["8F00FF", "4B0082", "0000FF", "00FF00", "FFFF00", "FF7F00", "FF0000"] }
```

---

## Interaction Presets

Default parameters for each interaction type:

| Interaction | uniformColor | size scale | bondScale | gapScale |
|-------------|-------------|------------|-----------|----------|
| HydrogenBonds | FFFF00 | 0.05 | 0.2 | 0.2 |
| PiCation | 0080FF | 0.06 | 0.2 | 0.2 |
| Clash | CC003D | 0.5 | 0.05 | 0.0 |
| Halogen | 3AE2AA | 0.05 | 0.15 | 0.25 |
| Hydrophobic | 808080 | 0.03 | 0.12 | 0.3 |
| MetalCoordination | A020F0 | 0.075 | 0.35 | 0.25 |
| PiStacking | 6443FF | 0.04 | 0.25 | 0.35 |
| SaltBridges | FF6600 | 0.06 | 0.3 | 0.15 |

### Interaction Visualization Template

Use this template for any interaction type, substituting values from the table above:

```json
{
  "kind": "Interaction",
  "visible": true,
  "interactionType": "HydrogenBonds",
  "size": { "sizeOption": "Uniform", "scale": 0.05 },
  "coloring": { "kind": "Uniform", "carbonsOnly": false, "uniformColor": "FFFF00" },
  "bondScale": 0.2,
  "gapScale": 0.2
}
```

Use the `uniformColor`, `size scale`, `bondScale`, and `gapScale` values from the table above for each interaction type.

---

## Scene Design Heuristics

Use this table to map from user goals to scene/component strategies:

| User Goal | Scenes | Component Strategy |
|-----------|--------|-------------------|
| General protein visualization | Overview | Protein: Cartoon/ChainInstance coloring. Ligands: Ball-and-stick/ElementType |
| Drug binding analysis | Overview, Binding Site, Surface | + Distance filter 5A around ligand, stick for nearby residues, HydrogenBonds + Hydrophobic interactions, transparent surface |
| Structure comparison | Per-structure overview | Each entry: Cartoon with distinct Uniform color |
| DNA/RNA-protein complex | Overview, Interface | Protein + nucleic acid as Cartoon with different colors, interface residues within 4A |
| Small molecule library | Ligand gallery | Each ligand: Ball-and-stick, ElementType coloring |
| AlphaFold prediction | Confidence view | Cartoon colored by BFactor (pLDDT), Cold-to-Warm palette |
| Teaching/presentation | Progressive scenes | Scene 1: full protein. Scene 2: zoom to region. Scene 3: details |
| Unrecognized / other | Overview only | Fallback: use `add-default` per entry |

### General Rules

- Always create an Overview scene as scene 1
- Prefer one component per logical selection (per entry + substructure type)
- Use per-entry `add-default` as fallback when user's goal doesn't match any heuristic
- Max 20 scenes per workspace — keep it under 10 for usability
- Solvent and ions hidden by default unless explicitly requested
- When using `add-default` with multiple entries, call the per-entry route: `POST /workspaces/{id}/scenes/{serial}/entries/{entrySerial}/components/add-default`

### Worked Example 1: Drug Binding Analysis (PDB 1IEP — ABL kinase + imatinib)

**Scene 1 "Protein Overview":**

Component "Protein":
```bash
curl -s -X POST "$API_BASE/workspaces/$WORKSPACE_ID/scenes/1/components/add" \
  "${AUTH_ARGS[@]}" \
  -H "Content-Type: application/json" \
  -d @- <<'COMPONENT'
{
  "name": "Protein",
  "filters": [
    { "kind": "EntrySerial", "entrySerial": 1 },
    { "kind": "CurrentModel" },
    { "kind": "Substructure", "substructureType": "Protein" }
  ],
  "visualizations": [
    {
      "kind": "Ribbon",
      "visible": true,
      "ribbonStyle": "Cartoon",
      "size": { "sizeOption": "Uniform", "scale": 1.0 },
      "coloring": {
        "kind": "Property",
        "carbonsOnly": false,
        "propertySource": "ChainInstance"
      }
    }
  ]
}
COMPONENT
```

Component "Ligand":
```bash
curl -s -X POST "$API_BASE/workspaces/$WORKSPACE_ID/scenes/1/components/add" \
  "${AUTH_ARGS[@]}" \
  -H "Content-Type: application/json" \
  -d @- <<'COMPONENT'
{
  "name": "Ligand",
  "filters": [
    { "kind": "EntrySerial", "entrySerial": 1 },
    { "kind": "CurrentModel" },
    { "kind": "Substructure", "substructureType": "Ligand" }
  ],
  "visualizations": [
    {
      "kind": "Atomistic",
      "visible": true,
      "size": { "sizeOption": "Physical", "scale": 0.25 },
      "coloring": { "kind": "ElementType", "carbonsOnly": false },
      "renderAtom": "All",
      "renderBond": true,
      "bondScale": 0.2
    }
  ]
}
COMPONENT
```

**Scene 2 "Binding Site":**

First, create and rename the scene:
```bash
SCENE2=$(curl -s -X POST "$API_BASE/workspaces/$WORKSPACE_ID/scenes" "${AUTH_ARGS[@]}")
SCENE2_SERIAL=$(echo "$SCENE2" | python3 -c "import sys,json; print(json.load(sys.stdin)['serial'])")

curl -s -X POST "$API_BASE/workspaces/$WORKSPACE_ID/scenes/$SCENE2_SERIAL/rename" \
  "${AUTH_ARGS[@]}" \
  -H "Content-Type: application/json" \
  -d '{"name": "Binding Site"}'
```

Component "Imatinib":
```bash
curl -s -X POST "$API_BASE/workspaces/$WORKSPACE_ID/scenes/$SCENE2_SERIAL/components/add" \
  "${AUTH_ARGS[@]}" \
  -H "Content-Type: application/json" \
  -d @- <<'COMPONENT'
{
  "name": "Imatinib",
  "filters": [
    { "kind": "EntrySerial", "entrySerial": 1 },
    { "kind": "CurrentModel" },
    { "kind": "Substructure", "substructureType": "Ligand" }
  ],
  "visualizations": [
    {
      "kind": "Atomistic",
      "visible": true,
      "size": { "sizeOption": "Physical", "scale": 0.25 },
      "coloring": { "kind": "ElementType", "carbonsOnly": false },
      "renderAtom": "All",
      "renderBond": true,
      "bondScale": 0.2
    }
  ]
}
COMPONENT
```

Component "Binding Residues":
```bash
curl -s -X POST "$API_BASE/workspaces/$WORKSPACE_ID/scenes/$SCENE2_SERIAL/components/add" \
  "${AUTH_ARGS[@]}" \
  -H "Content-Type: application/json" \
  -d @- <<'COMPONENT'
{
  "name": "Binding Residues",
  "filters": [
    {
      "kind": "Intersect",
      "containedFilters": [
        { "kind": "EntrySerial", "entrySerial": 1 },
        { "kind": "CurrentModel" },
        {
          "kind": "Distance",
          "distance": 5.0,
          "wholeResidue": true,
          "ignoreSelf": true,
          "containedFilters": [
            { "kind": "Substructure", "substructureType": "Ligand" }
          ]
        }
      ]
    }
  ],
  "visualizations": [
    {
      "kind": "Atomistic",
      "visible": true,
      "size": { "sizeOption": "Uniform", "scale": 0.2 },
      "coloring": { "kind": "Property", "carbonsOnly": true, "propertySource": "ChainInstance" },
      "renderAtom": "All",
      "renderBond": true,
      "bondScale": 0.2
    }
  ]
}
COMPONENT
```

Component "Interactions" (hydrogen bonds):
```bash
curl -s -X POST "$API_BASE/workspaces/$WORKSPACE_ID/scenes/$SCENE2_SERIAL/components/add" \
  "${AUTH_ARGS[@]}" \
  -H "Content-Type: application/json" \
  -d @- <<'COMPONENT'
{
  "name": "H-Bond Interactions",
  "filters": [
    { "kind": "EntrySerial", "entrySerial": 1 },
    { "kind": "CurrentModel" },
    { "kind": "Substructure", "substructureType": "Ligand" }
  ],
  "visualizations": [
    {
      "kind": "Interaction",
      "visible": true,
      "interactionType": "HydrogenBonds",
      "size": { "sizeOption": "Uniform", "scale": 0.05 },
      "coloring": { "kind": "Uniform", "carbonsOnly": false, "uniformColor": "FFFF00" },
      "bondScale": 0.2,
      "gapScale": 0.2
    },
    {
      "kind": "Interaction",
      "visible": true,
      "interactionType": "Hydrophobic",
      "size": { "sizeOption": "Uniform", "scale": 0.03 },
      "coloring": { "kind": "Uniform", "carbonsOnly": false, "uniformColor": "808080" },
      "bondScale": 0.12,
      "gapScale": 0.3
    }
  ]
}
COMPONENT
```

**Scene 3 "Surface View":**

```bash
SCENE3=$(curl -s -X POST "$API_BASE/workspaces/$WORKSPACE_ID/scenes" "${AUTH_ARGS[@]}")
SCENE3_SERIAL=$(echo "$SCENE3" | python3 -c "import sys,json; print(json.load(sys.stdin)['serial'])")

curl -s -X POST "$API_BASE/workspaces/$WORKSPACE_ID/scenes/$SCENE3_SERIAL/rename" \
  "${AUTH_ARGS[@]}" \
  -H "Content-Type: application/json" \
  -d '{"name": "Surface View"}'
```

Component "Protein Surface":
```bash
curl -s -X POST "$API_BASE/workspaces/$WORKSPACE_ID/scenes/$SCENE3_SERIAL/components/add" \
  "${AUTH_ARGS[@]}" \
  -H "Content-Type: application/json" \
  -d @- <<'COMPONENT'
{
  "name": "Protein Surface",
  "filters": [
    { "kind": "EntrySerial", "entrySerial": 1 },
    { "kind": "CurrentModel" },
    { "kind": "Substructure", "substructureType": "Protein" }
  ],
  "visualizations": [
    {
      "kind": "MolecularSurface",
      "visible": true,
      "size": { "sizeOption": "Uniform", "scale": 1.0 },
      "coloring": { "kind": "Property", "carbonsOnly": false, "propertySource": "ChainInstance" },
      "opacity": 0.5,
      "subsurface": false
    }
  ]
}
COMPONENT
```

Component "Ligand in Surface":
```bash
curl -s -X POST "$API_BASE/workspaces/$WORKSPACE_ID/scenes/$SCENE3_SERIAL/components/add" \
  "${AUTH_ARGS[@]}" \
  -H "Content-Type: application/json" \
  -d @- <<'COMPONENT'
{
  "name": "Ligand in Surface",
  "filters": [
    { "kind": "EntrySerial", "entrySerial": 1 },
    { "kind": "CurrentModel" },
    { "kind": "Substructure", "substructureType": "Ligand" }
  ],
  "visualizations": [
    {
      "kind": "Atomistic",
      "visible": true,
      "size": { "sizeOption": "Physical", "scale": 0.25 },
      "coloring": { "kind": "ElementType", "carbonsOnly": false },
      "renderAtom": "All",
      "renderBond": true,
      "bondScale": 0.2
    }
  ]
}
COMPONENT
```

### Worked Example 2: Structure Comparison (two PDBs side by side)

**Scene 1 "Comparison":**

Rename the auto-created scene 1:
```bash
curl -s -X POST "$API_BASE/workspaces/$WORKSPACE_ID/scenes/1/rename" \
  "${AUTH_ARGS[@]}" \
  -H "Content-Type: application/json" \
  -d '{"name": "Comparison"}'
```

Component "Structure 1" (blue):
```bash
curl -s -X POST "$API_BASE/workspaces/$WORKSPACE_ID/scenes/1/components/add" \
  "${AUTH_ARGS[@]}" \
  -H "Content-Type: application/json" \
  -d @- <<'COMPONENT'
{
  "name": "Structure 1",
  "filters": [
    { "kind": "EntrySerial", "entrySerial": 1 },
    { "kind": "CurrentModel" },
    { "kind": "Substructure", "substructureType": "Protein" }
  ],
  "visualizations": [
    {
      "kind": "Ribbon",
      "visible": true,
      "ribbonStyle": "Cartoon",
      "size": { "sizeOption": "Uniform", "scale": 1.0 },
      "coloring": {
        "kind": "Uniform",
        "carbonsOnly": false,
        "uniformColor": "4287f5"
      }
    }
  ]
}
COMPONENT
```

Component "Structure 2" (red):
```bash
curl -s -X POST "$API_BASE/workspaces/$WORKSPACE_ID/scenes/1/components/add" \
  "${AUTH_ARGS[@]}" \
  -H "Content-Type: application/json" \
  -d @- <<'COMPONENT'
{
  "name": "Structure 2",
  "filters": [
    { "kind": "EntrySerial", "entrySerial": 2 },
    { "kind": "CurrentModel" },
    { "kind": "Substructure", "substructureType": "Protein" }
  ],
  "visualizations": [
    {
      "kind": "Ribbon",
      "visible": true,
      "ribbonStyle": "Cartoon",
      "size": { "sizeOption": "Uniform", "scale": 1.0 },
      "coloring": {
        "kind": "Uniform",
        "carbonsOnly": false,
        "uniformColor": "f54242"
      }
    }
  ]
}
COMPONENT
```

---

## API Reference

### Endpoints

| Action | Method | Endpoint | Body/Notes |
|--------|--------|----------|------------|
| Create workspace | POST | `/workspaces` | `{ "name": "...", "description": "..." }`. Returns `{ "id": "GUID", "name": "..." }` |
| Edit workspace | POST | `/workspaces/{id}` | `{ "name": "...", "description": "..." }` |
| Load structure | POST | `/workspaces/{id}/entries/load` | Multipart file upload. Returns `[{ "serial": 1, "name": "..." }]` (array) |
| Rename entry | POST | `/workspaces/{id}/entries/{serial}/rename` | `{ "name": "..." }` |
| Create scene | POST | `/workspaces/{id}/scenes` | Returns single object `{ "serial": 2, "name": "Scene" }` (NOT an array) |
| Rename scene | POST | `/workspaces/{id}/scenes/{serial}/rename` | `{ "name": "..." }` |
| Add default components (scene-wide) | POST | `/workspaces/{id}/scenes/{serial}/components/add-default` | Adds defaults for all entries |
| Add default components (per-entry) | POST | `/workspaces/{id}/scenes/{serial}/entries/{entrySerial}/components/add-default` | Preferred for multi-entry workspaces |
| Add custom component | POST | `/workspaces/{id}/scenes/{serial}/components/add` | Single ComponentInput JSON (not array) |

All endpoints require `Authorization: Bearer {token}` when auth is active.

### Complete Script Template

```bash
#!/bin/bash
# Nanome Workspace: {NAME}
# Structures: {list}
# Scenes: {list}
#
# Requirements: curl, python3
# Usage: bash create_workspace.sh
#   Or: API_BASE=... TOKEN=... bash create_workspace.sh

set -e

# --- Config ---
CONFIG_FILE="$HOME/.nanome/config.json"
if [ -f "$CONFIG_FILE" ]; then
  API_BASE=$(python3 -c "import json; print(json.load(open('$CONFIG_FILE')).get('api_base', 'https://workspaces.nanome.ai'))")
  TOKEN=$(python3 -c "import json; print(json.load(open('$CONFIG_FILE')).get('token', ''))")
fi
API_BASE="${API_BASE:-https://workspaces.nanome.ai}"

AUTH_ARGS=()
if [ -n "$TOKEN" ]; then
  AUTH_ARGS=(-H "Authorization: Bearer $TOKEN")
fi

download_structure() {
  local url="$1" output="$2"
  local http_code
  http_code=$(curl -s -o "$output" -w "%{http_code}" "$url")
  if [ "$http_code" != "200" ] || [ ! -s "$output" ]; then
    echo "ERROR: Failed to download $url (HTTP $http_code)" >&2
    rm -f "$output"
    return 1
  fi
}

# --- Create Workspace ---
echo "Creating workspace: {NAME}..."
WORKSPACE=$(curl -s -X POST "$API_BASE/workspaces" \
  "${AUTH_ARGS[@]}" \
  -H "Content-Type: application/json" \
  -d '{"name": "{NAME}", "description": "{DESCRIPTION}"}')
WORKSPACE_ID=$(echo "$WORKSPACE" | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
echo "Workspace ID: $WORKSPACE_ID"

# --- Download & Upload Structures ---
WORKDIR=$(mktemp -d)
trap 'rm -rf "$WORKDIR"' EXIT

echo "Downloading {PDB_ID}..."
download_structure "https://files.rcsb.org/download/{PDB_ID}.pdb" "$WORKDIR/{PDB_ID}.pdb"

echo "Uploading {PDB_ID}..."
ENTRY=$(curl -s -X POST "$API_BASE/workspaces/$WORKSPACE_ID/entries/load" \
  "${AUTH_ARGS[@]}" \
  -F "file=@$WORKDIR/{PDB_ID}.pdb")
ENTRY_SERIAL=$(echo "$ENTRY" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d[0]['serial'] if isinstance(d,list) else d['serial'])")

# --- Scene 1 (auto-created, rename it) ---
curl -s -X POST "$API_BASE/workspaces/$WORKSPACE_ID/scenes/1/rename" \
  "${AUTH_ARGS[@]}" \
  -H "Content-Type: application/json" \
  -d '{"name": "{SCENE_1_NAME}"}'

# --- Add components to Scene 1 ---
curl -s -X POST "$API_BASE/workspaces/$WORKSPACE_ID/scenes/1/components/add" \
  "${AUTH_ARGS[@]}" \
  -H "Content-Type: application/json" \
  -d @- <<'COMPONENT'
{JSON component payload}
COMPONENT

# --- Create Scene 2 ---
SCENE2=$(curl -s -X POST "$API_BASE/workspaces/$WORKSPACE_ID/scenes" \
  "${AUTH_ARGS[@]}")
SCENE2_SERIAL=$(echo "$SCENE2" | python3 -c "import sys,json; print(json.load(sys.stdin)['serial'])")

curl -s -X POST "$API_BASE/workspaces/$WORKSPACE_ID/scenes/$SCENE2_SERIAL/rename" \
  "${AUTH_ARGS[@]}" \
  -H "Content-Type: application/json" \
  -d '{"name": "{SCENE_2_NAME}"}'

# --- Output ---
echo ""
echo "Workspace created successfully!"
echo "URL: https://mara.nanome.ai/workspaces/$WORKSPACE_ID"

# Cleanup handled by trap on EXIT
```

---

## MCP Mode

When `nanome_*` MCP tools are available (Claude Chat with Nanome MCP server), use tool calls instead of generating a bash script. The component JSON is identical — only the execution mechanism changes.

### Tool Call Sequence

1. `nanome_create_workspace({ name, description })` → get `workspace_id`
2. For each structure: `nanome_load_structure({ workspace_id, source, id })` → get `serial`
3. Scene 1 (auto-created): `nanome_rename_scene({ workspace_id, scene_serial: 1, name })`
4. Add components to scene 1: `nanome_add_component({ workspace_id, scene_serial: 1, component })` — one call per component
5. For each additional scene: `nanome_create_scene({ workspace_id })` → get `serial`, then `nanome_rename_scene`, then `nanome_add_component` for each component

### Example: Drug Binding Workspace (ABL Kinase + Imatinib)

```
nanome_create_workspace({ name: "ABL Kinase — Imatinib", description: "Drug binding analysis" })
→ { id: "abc-123", name: "ABL Kinase — Imatinib" }

nanome_load_structure({ workspace_id: "abc-123", source: "rcsb", id: "1IEP" })
→ { serial: 1, name: "1IEP" }

nanome_rename_scene({ workspace_id: "abc-123", scene_serial: 1, name: "Protein Overview" })

nanome_add_component({ workspace_id: "abc-123", scene_serial: 1, component: {
  "name": "Protein",
  "filters": [
    { "kind": "EntrySerial", "entrySerial": 1 },
    { "kind": "CurrentModel" },
    { "kind": "Substructure", "substructureType": "Protein" }
  ],
  "visualizations": [{
    "kind": "Ribbon", "visible": true, "ribbonStyle": "Cartoon",
    "size": { "sizeOption": "Uniform", "scale": 1.0 },
    "coloring": { "kind": "Property", "carbonsOnly": false, "propertySource": "ChainInstance" }
  }]
}})

nanome_add_component({ workspace_id: "abc-123", scene_serial: 1, component: {
  "name": "Ligand",
  "filters": [
    { "kind": "EntrySerial", "entrySerial": 1 },
    { "kind": "CurrentModel" },
    { "kind": "Substructure", "substructureType": "Ligand" }
  ],
  "visualizations": [{
    "kind": "Atomistic", "visible": true,
    "size": { "sizeOption": "Physical", "scale": 0.25 },
    "coloring": { "kind": "ElementType", "carbonsOnly": false },
    "renderAtom": "All", "renderBond": true, "bondScale": 0.2
  }]
}})

nanome_create_scene({ workspace_id: "abc-123" })
→ { serial: 2, name: "Scene" }

nanome_rename_scene({ workspace_id: "abc-123", scene_serial: 2, name: "Binding Site" })

nanome_add_component({ workspace_id: "abc-123", scene_serial: 2, component: {
  "name": "Imatinib",
  "filters": [
    { "kind": "EntrySerial", "entrySerial": 1 },
    { "kind": "CurrentModel" },
    { "kind": "Substructure", "substructureType": "Ligand" }
  ],
  "visualizations": [{
    "kind": "Atomistic", "visible": true,
    "size": { "sizeOption": "Physical", "scale": 0.25 },
    "coloring": { "kind": "ElementType", "carbonsOnly": false },
    "renderAtom": "All", "renderBond": true, "bondScale": 0.2
  }]
}})

nanome_add_component({ workspace_id: "abc-123", scene_serial: 2, component: {
  "name": "Binding Residues",
  "filters": [{
    "kind": "Intersect",
    "containedFilters": [
      { "kind": "EntrySerial", "entrySerial": 1 },
      { "kind": "CurrentModel" },
      { "kind": "Distance", "distance": 5.0, "wholeResidue": true, "ignoreSelf": true,
        "containedFilters": [{ "kind": "Substructure", "substructureType": "Ligand" }] }
    ]
  }],
  "visualizations": [{
    "kind": "Atomistic", "visible": true,
    "size": { "sizeOption": "Uniform", "scale": 0.2 },
    "coloring": { "kind": "Property", "carbonsOnly": true, "propertySource": "ChainInstance" },
    "renderAtom": "All", "renderBond": true, "bondScale": 0.2
  }]
}})

nanome_add_component({ workspace_id: "abc-123", scene_serial: 2, component: {
  "name": "H-Bond Interactions",
  "filters": [
    { "kind": "EntrySerial", "entrySerial": 1 },
    { "kind": "CurrentModel" },
    { "kind": "Substructure", "substructureType": "Ligand" }
  ],
  "visualizations": [{
    "kind": "Interaction", "visible": true, "interactionType": "HydrogenBonds",
    "size": { "sizeOption": "Uniform", "scale": 0.06 },
    "coloring": { "kind": "Uniform", "carbonsOnly": false, "uniformColor": "FFFF00" },
    "bondScale": 0.2, "gapScale": 0.2
  }]
}})
```

### Key Differences from Script Mode

- No file downloads or uploads — `nanome_load_structure` handles both internally
- No auth setup — MCP server handles auth from `~/.nanome/config.json` or env vars
- No script generation — direct tool calls
- Same component JSON — identical `filters`, `visualizations`, `coloring` objects as Script Mode

---

## Artifact Mode

When neither Bash nor `nanome_*` MCP tools are available (Claude Chat without MCP server), generate a self-contained HTML artifact that builds the workspace in the user's browser.

### How to Use

Generate an HTML artifact using the template below. Fill in the four data constants (`WORKSPACE_NAME`, `WORKSPACE_DESCRIPTION`, `STRUCTURES`, `SCENES`) with the resolved workspace data. The `SCENES` array uses the exact same component JSON schema as Script Mode and MCP Mode.

The user clicks "Build Workspace" in the artifact, enters their Nanome token, and the JavaScript executes all API calls sequentially with progress logging.

### Placeholder Reference

| Placeholder | Type | Example |
|-------------|------|---------|
| `WORKSPACE_NAME` | string | `"ABL Kinase — Imatinib"` |
| `WORKSPACE_DESCRIPTION` | string | `"Drug binding analysis"` |
| `STRUCTURES` | array of `{ source, id, filename }` | `[{ source: "rcsb", id: "1IEP", filename: "1IEP.pdb" }]` |
| `SCENES` | array of `{ name, components: ComponentInput[] }` | `[{ name: "Overview", components: [/* same JSON as API */] }]` |

### Template

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Nanome Workspace Builder</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: -apple-system, system-ui, sans-serif; max-width: 640px; margin: 40px auto; padding: 0 20px; color: #1a1a1a; }
    h2 { margin-bottom: 16px; }
    .field { margin-bottom: 12px; }
    .field label { display: block; font-weight: 600; margin-bottom: 4px; font-size: 14px; }
    .field input { width: 100%; padding: 8px; border: 1px solid #ccc; border-radius: 4px; font-size: 14px; }
    button { background: #2563eb; color: white; border: none; padding: 10px 20px; border-radius: 4px; font-size: 14px; cursor: pointer; }
    button:disabled { background: #93b4f5; cursor: not-allowed; }
    #log { margin-top: 16px; padding: 12px; background: #f5f5f5; border-radius: 4px; font-family: monospace; font-size: 13px; white-space: pre-wrap; max-height: 400px; overflow-y: auto; display: none; }
    .success { color: #16a34a; font-weight: 600; }
    .error { color: #dc2626; font-weight: 600; }
    #result { margin-top: 16px; display: none; }
    #result a { color: #2563eb; font-weight: 600; }
  </style>
</head>
<body>
  <h2>/* WORKSPACE_NAME */</h2>
  <p style="margin-bottom:16px;color:#666">/* WORKSPACE_DESCRIPTION */</p>
  <div class="field">
    <label>Nanome Token (<a href="https://mara.nanome.ai/login" target="_blank">get token</a>)</label>
    <input type="password" id="token" placeholder="Paste your token here" />
  </div>
  <button id="btn" onclick="build()">Build Workspace</button>
  <div id="result"></div>
  <pre id="log"></pre>

  <script>
    const API = 'https://workspaces.nanome.ai';
    const WORKSPACE_NAME = '/* fill in */';
    const WORKSPACE_DESCRIPTION = '/* fill in */';
    const STRUCTURES = /* fill in: [{ source: "rcsb", id: "1IEP", filename: "1IEP.pdb" }] */;
    const SCENES = /* fill in: [{ name: "Overview", components: [{ name: "Protein", filters: [...], visualizations: [...] }] }] */;

    const SOURCE_URLS = {
      rcsb: (id) => `https://files.rcsb.org/download/${id}.pdb`,
      alphafold: (id) => `https://alphafold.ebi.ac.uk/files/AF-${id}-F1-model_v4.pdb`,
      pubchem: (id) => `https://pubchem.ncbi.nlm.nih.gov/rest/pug/compound/CID/${id}/SDF`,
      chembl: (id) => `https://www.ebi.ac.uk/chembl/api/data/molecule/${id}.sdf`,
      swissmodel: (id) => `https://swissmodel.expasy.org/repository/uniprot/${id}.pdb`,
      cod: (id) => `https://www.crystallography.net/cod/${id}.cif`,
    };

    const logEl = document.getElementById('log');
    const btn = document.getElementById('btn');
    const resultEl = document.getElementById('result');

    function appendLog(msg, cls) {
      logEl.style.display = 'block';
      const span = document.createElement('span');
      if (cls) span.className = cls;
      span.textContent = msg + '\n';
      logEl.appendChild(span);
      logEl.scrollTop = logEl.scrollHeight;
    }

    function showResult(wsId) {
      resultEl.style.display = 'block';
      resultEl.textContent = '';
      const label = document.createElement('strong');
      label.textContent = 'Workspace URL: ';
      const link = document.createElement('a');
      link.href = `https://mara.nanome.ai/workspaces/${wsId}`;
      link.target = '_blank';
      link.textContent = `https://mara.nanome.ai/workspaces/${wsId}`;
      resultEl.appendChild(label);
      resultEl.appendChild(link);
    }

    async function apiPost(path, body, token, isFormData) {
      const headers = {};
      if (token) headers['Authorization'] = `Bearer ${token}`;
      if (!isFormData) headers['Content-Type'] = 'application/json';
      const opts = { method: 'POST', headers };
      if (body) opts.body = isFormData ? body : JSON.stringify(body);
      const res = await fetch(`${API}${path}`, opts);
      if (!res.ok) {
        const text = await res.text();
        if (res.status === 401) throw new Error('Authentication failed (401). Check your token or get a new one at https://mara.nanome.ai/login');
        throw new Error(`API error ${res.status}: ${text}`);
      }
      return res.json();
    }

    async function build() {
      const token = document.getElementById('token').value.trim();
      if (!token) { alert('Please enter your Nanome token'); return; }
      btn.disabled = true;
      logEl.textContent = '';
      resultEl.style.display = 'none';

      try {
        // 1. Create workspace
        appendLog(`Creating workspace: ${WORKSPACE_NAME}...`);
        const ws = await apiPost('/workspaces', { name: WORKSPACE_NAME, description: WORKSPACE_DESCRIPTION }, token);
        const wsId = ws.id;
        appendLog(`Workspace created: ${wsId}`, 'success');
        showResult(wsId);

        // 2. Load structures
        for (const struct of STRUCTURES) {
          try {
            const url = SOURCE_URLS[struct.source](struct.id);
            appendLog(`Downloading ${struct.source}:${struct.id}...`);
            const dlRes = await fetch(url);
            if (!dlRes.ok) throw new Error(`Download failed: ${dlRes.status}`);
            const blob = await dlRes.blob();
            const form = new FormData();
            form.append('file', blob, struct.filename);
            appendLog(`Uploading ${struct.filename}...`);
            const entry = await apiPost(`/workspaces/${wsId}/entries/load`, form, token, true);
            const serial = Array.isArray(entry) ? entry[0].serial : entry.serial;
            appendLog(`Loaded entry serial ${serial}`, 'success');
          } catch (err) {
            appendLog(`Warning: Failed to load ${struct.source}:${struct.id} — ${err.message}`, 'error');
          }
        }

        // 3. Build scenes
        for (let i = 0; i < SCENES.length; i++) {
          const scene = SCENES[i];
          let sceneSerial;
          if (i === 0) {
            sceneSerial = 1;
            appendLog(`Renaming scene 1 to "${scene.name}"...`);
            await apiPost(`/workspaces/${wsId}/scenes/1/rename`, { name: scene.name }, token);
          } else {
            appendLog(`Creating scene "${scene.name}"...`);
            const s = await apiPost(`/workspaces/${wsId}/scenes`, null, token);
            sceneSerial = s.serial;
            await apiPost(`/workspaces/${wsId}/scenes/${sceneSerial}/rename`, { name: scene.name }, token);
          }
          for (const comp of scene.components) {
            appendLog(`  Adding component: ${comp.name}...`);
            await apiPost(`/workspaces/${wsId}/scenes/${sceneSerial}/components/add`, comp, token);
          }
          appendLog(`Scene "${scene.name}" complete`, 'success');
        }

        appendLog('\nWorkspace build complete!', 'success');
      } catch (err) {
        appendLog(`\nBuild failed: ${err.message}`, 'error');
      } finally {
        btn.disabled = false;
      }
    }
  </script>
</body>
</html>
```

### Error Handling

- **401 Unauthorized** — shows "Authentication failed" with link to `https://mara.nanome.ai/login`
- **CORS blocked** — browser shows a fetch network error; suggest using MCP server mode instead
- **Structure download failure** — logs warning for that structure and continues with remaining structures (does not abort)
- **Other API errors** — shows which step failed, HTTP status, and response body

### CORS Notes

Verified CORS support: RCSB (`files.rcsb.org`), PubChem, AlphaFold. Unverified: ChEMBL, SWISS-MODEL, COD — if these fail in browser, the artifact logs a warning and continues. MCP Mode is unaffected (server-side downloads).

---

## Output Format

After executing the workspace creation script, produce three things:

### 1. Workspace URL

```
https://mara.nanome.ai/workspaces/{workspace-id}
```

### 2. Summary

Provide the workspace name, structure count, and a scene list with short descriptions of what each scene shows.

### 3. MARA Suggestions

Provide 3-5 contextual follow-up prompts the user can give to MARA (the in-session AI) to refine the workspace. Tailor suggestions to the actual workspace content:

- Workspace has ligands — suggest interaction analysis, surface views
- Workspace has multiple chains — suggest comparison, alignment
- Workspace has AlphaFold model — suggest pLDDT (BFactor) coloring
- Workspace has DNA/RNA — suggest interface analysis
- Workspace has multiple entries — suggest structural alignment
- Always include at least one coloring suggestion and one show/hide suggestion

### Example Output

```
Workspace created: https://mara.nanome.ai/workspaces/a1b2c3d4-...

"HIV Protease Analysis" — 2 structures (1HVR, 1HPV), 3 scenes:
  1. Protein Overview — ribbon cartoon, ligands as ball-and-stick
  2. Binding Site — inhibitor + nearby residues within 5A, H-bond interactions
  3. Surface View — transparent molecular surface around active site

Continue in MARA to refine:
  - "Color the protein by B-factor to highlight flexible regions"
  - "Add salt bridge interactions to the binding site"
  - "Show a transparent surface around the ligand pocket"
  - "Hide solvent molecules"
```

---

## Common Mistakes to Avoid

- **`size` is required on EVERY visualization** — including Ribbon, MolecularSurface, ResidueLabel. Omitting it causes HTTP 500. Use `{"sizeOption":"Uniform","scale":1.0}` as default.
- **Scene creation returns a single object** `{"serial": N}` — use `d['serial']` directly. Entry loading returns an array `[{"serial": 1}]` — use `d[0]['serial']`. Don't mix these up.
- **Don't assume auth is always required** — make it conditional with `AUTH_ARGS`
- **Scene serial 1 is auto-created with workspace** — rename it, don't create a new one
- **Entry serials start at 1** and increment per workspace
- **Component endpoint takes ONE component per call**, not an array
- **Use exact interaction enum names** — `HydrogenBonds` not `HydrogenBond`, `SaltBridges` not `SaltBridge`, `PiCation` not `CationPi`
- **Always include `visible: true`** on visualizations and **`carbonsOnly: false`** on coloring (unless specifically using carbonsOnly)
- **Use per-entry `add-default` route** for multi-entry workspaces
- **Validate downloads before uploading** — check HTTP status code and non-empty file
- **Large PDB files (>50MB) may time out** — warn the user
- **Clean up downloaded files** after upload
