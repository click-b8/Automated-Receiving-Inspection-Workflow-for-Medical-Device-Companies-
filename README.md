# Medical Device Receiving Inspection Automation

Automates the receiving inspection workflow for medical device companies operating under
ISO 13485 / FDA 21 CFR Part 820. Designed to run inside **Claude Code**.

## What it does

Given a Purchase Order folder containing shipping docs, drawings, and certificates,
this tool:

1. **Scans** all subfolders and matches documents to each part by part number
2. **Extracts** data from POs, packing slips, receiving lists, COCs, and CAD drawings
3. **Cross-references** all documents for consistency (part#, rev, lot, qty, dates)
4. **Fills** a Quality Product Release Form (PRF) Excel template automatically
5. **Compiles** all documents for each part into one ordered PDF package
6. **Exports** the PRF as PDF with correct header, logo, and fonts on every page

## Folder structure expected

```
PO26-0085/
├── PO26-0085.pdf
├── Shipping Docs/          ← packing slips, receiving list
├── Drawings/               ← one CAD drawing PDF per part
├── COCs/                   ← vendor COCs, sterilization COCs, travelers
├── In-Progress/            ← compiled PDFs output here (created automatically)
└── Inspections/            ← PRF Excel + PDF output here (created automatically)
```

Subfolder names are detected by content — they don't need to match exactly.

## Quick start

### 1. Install dependencies

```bash
pip install anthropic pdfplumber openpyxl pymupdf
```

Also install [LibreOffice](https://libreoffice.org) for PDF export.

### 2. Set your Anthropic API key

**Windows:**
Add `ANTHROPIC_API_KEY` as a User Environment Variable under
Settings → System → Environment Variables.

**Mac/Linux:**
```bash
export ANTHROPIC_API_KEY="sk-ant-..."
```

### 3. Place required files

Put these in your working `Receivings/` folder (parent of all PO folders):
- `CLAUDE.md` — this file (Claude Code reads it automatically)
- `QF_7_4_3-02_Rev_E_template.xlsx` — your PRF Excel template

### 4. Run

Open Claude Code in your `Receivings/` folder and type:

```
process PO26-0085
```

Claude Code reads `CLAUDE.md`, scans the PO folder, processes every part on the order,
and outputs a filled PRF + compiled PDF for each.

## Output per part

| File | Location | Purpose |
|------|----------|---------|
| `PRF-[PART]_Rev_[X].xlsx` | `Inspections/` | Filled Excel — add measurements, get signatures |
| `PRF-[PART]_Rev_[X].pdf` | `Inspections/` | PDF version with header/logo on every page |
| `PO##-##### Receiving Part X - [PART].pdf` | `In-Progress/` | Compiled receiving package |

## PRF auto-fill coverage

| Section | Auto-filled | You fill |
|---------|-------------|----------|
| 1.1 Part info | ✓ PO#, supplier, date, part#, rev, lot, qty | Inspector signature |
| 1.2 Qty inspected | ✓ N/A or 100% checkbox | AQL / qty if applicable |
| 1.3 QDA | ✓ N/A or YES/NO | Last QDA date if applicable |
| 2.1–2.6 Checklist | ✓ All checkboxes with logic | — |
| 3 Data report | ✓ Characteristics, limits, gaging method | **Measured values** (blue cells) |
| 4 Comments | ✓ Off-the-shelf notes, DVT references | Any additional notes |
| 5 Discrepancies | ✓ N/A or flagged issues | — |
| 6 Disposition | ✓ Raw/Finished goods checkbox | QM signature |

## Adapting to your QMS

The `CLAUDE.md` is the core — edit it to match your:
- **PRF template** — update cell coordinates in Step 8 if your form differs
- **Signing workflow** — update Step 7 and the "What the engineer does after" section
- **Vendor rules** — update Special Cases with your confidential vendor list
- **Document numbering** — update the header stamp in Step 9

## Requirements

- Python 3.10+
- Claude Code (https://docs.claude.com/claude-code)
- Anthropic API key (https://console.anthropic.com)
- LibreOffice (https://libreoffice.org) — for PDF export

## Security note

`CLAUDE.md` enforces strict file access — Claude Code can only read/write files inside
the folder it was opened in. It will never access files outside the PO folder tree.

## License

MIT — adapt freely for your quality management system.
