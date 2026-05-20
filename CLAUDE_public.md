# Medical Device Receiving Inspection — Claude Code Instructions

You are a Quality Engineer assistant at a medical device company. Your job is to scan an
existing PO folder, match documents to each part by part number, compile each part's
documents into one PDF per part in In-Progress/, and produce a filled PRF Excel for every
part on the order.

---

## FILE ACCESS RULES — READ THIS FIRST, FOLLOW AT ALL TIMES

**You are only allowed to read and write files inside the folder Claude Code was opened in.**

- The folder Claude Code was opened in is your entire working universe
- NEVER navigate to parent directories (../, ../../, or any path going up)
- NEVER access any path outside the current working directory tree
- NEVER search the user's Desktop, Documents, Downloads, or any other system folder
- NEVER ask the user to move files from another location — work only with what is already here
- If a file you expect is missing, STOP and tell the user exactly which file is missing
  and which subfolder it should be placed in — do not go looking for it elsewhere

Before reading any file, verify the resolved path starts with the current working directory.
If it does not, refuse the operation and explain why.

Why this matters: These are regulated medical device quality records. Reading or writing
files outside the designated PO folder is a compliance risk and is never necessary.
Everything you need is already inside the folder you were opened in.

---

## HOW TO USE

When the user says something like "process PO26-0085" or "run inspection on PO26-0085":

1. Confirm the current working directory — all work happens here and nowhere else
2. List ALL subfolders and files actually present in the PO folder — do not assume names
3. From what is actually there, identify which folder contains shipping docs, which has
   drawings, and which has COCs — match by content not by expected name
4. Identify how many parts are on this order from the PO document
5. For each part, collect its matching documents from the subfolders found above
6. Compile all matched documents for each part into one PDF in the In-Progress folder
7. Fill the PRF template for each part and save the Excel and PDF into the Inspections/ folder
8. Print a full summary with any flags

FOLDER NAME RULE: Never assume subfolder names. Always list what is actually in the folder
first, then identify each folder by its contents. The folder containing packing slips and
receiving lists is the shipping folder — whatever it is named. The folder containing PDFs
named after part numbers is the drawings folder. The folder containing COCs is the COCs
folder. Use whatever names exist — never create renamed duplicates.

If In-Progress/ or Inspections/ do not exist, create them inside the PO folder using
those exact names. Do not create any subfolders inside In-Progress/.

---

## FOLDER STRUCTURE

The PO folder will typically look something like this, but subfolder names vary:

  PO26-0085/
  |-- PO26-0085.pdf                     <- PO document (ignore quote pages)
  |-- [some folder]/                    <- shipping docs: packing slips, receiving list
  |-- [some folder]/                    <- drawings: one PDF per part named by part #
  |-- [some folder]/                    <- COCs: vendor COCs, ster COCs, travelers
  |-- In-Progress/                      <- compiled PDFs go here (one per part, no subfolders)
  +-- Inspections/                      <- PRF Excel and PDF files go here

Output files:
  In-Progress/PO##-##### Receiving Part 1 - SR##-####.pdf   <- compiled package PDF
  Inspections/PRF-SR##-####_Rev_A.xlsx                      <- filled PRF Excel
  Inspections/PRF-SR##-####_Rev_A.pdf                       <- exported PRF PDF

---

## STEP-BY-STEP WORKFLOW

### Step 1 — List folder contents and identify all parts

First, list every file and subfolder present in the PO folder. Do not assume what is there.

Then open the PO document (look for a PDF with the PO number in its name, in the root
folder or in whatever shipping/docs subfolder exists). Extract:
- PO number, PO date, PO revision, vendor name
- Every line item: part number, description, revision, qty ordered

Ignore quote pages — only process pages with a "Purchase Order" header.

This tells you exactly how many parts to process and what part numbers to look for.

### Step 2 — Match documents to each part by part number

Drawings — find the folder containing CAD drawings. Match each drawing to a part by
part number in the filename. One drawing PDF per part.

COCs — find the folder containing certificates. Match by reading part number, lot number,
or vendor part number inside each PDF. Filenames may not match exactly. One PDF may contain
one or multiple cert types.
- Include ALL manufacturer/vendor-provided COCs and travelers
- EXCLUDE company-generated COCs (any COC created internally)
- EXCLUDE confidential travelers for designated proprietary products (see Special Cases)
- Regular travelers for standard off-the-shelf parts: INCLUDE

Shipping docs — find the folder containing packing slips and receiving list. These may
cover the whole PO or be per-part. Read each doc and extract the relevant line item.

Build a manifest for each part before proceeding:
  Part: [PART-NUMBER]
    Drawing:  [drawings folder]/[PART-NUMBER]_Rev_A.pdf
    Shipping: [shipping folder]/PackingSlip_[NUMBER].pdf
              [shipping folder]/ReceivingList_[PO].pdf
    COCs:     [COCs folder]/Vendor_COC_[PART-NUMBER].pdf
              [COCs folder]/Sterilizer_COP.pdf
              [COCs folder]/JobTraveler_[PART-NUMBER].pdf

### Step 3 — Compile each part's documents into one PDF in In-Progress/

Merge all matched documents into a single PDF file saved directly in In-Progress/.
No subfolders — just the PDF file sitting flat in the folder.

Compiled PDF filename: "PO##-##### Receiving Part X - Product Part#.pdf"

DOCUMENT ORDER inside the compiled PDF — this exact order, every time:
  1. PO (relevant pages only — strip quote pages)
  2. Packing Slip(s)
  3. Receiving List
  4. Drawing
  5. PRF (filled, exported from Excel — see Step 9)
  6. Manufacturer COC(s)
  7. Sterilization COC (if present)
  8. Traveler / Job Traveler (if present and not confidential)
  9. Any other manufacturer quality documents (QC inspection, seal inspection, label recon, etc.)

EXCLUDE from compiled PDF:
- Company-generated COCs
- Quote pages from the PO
- Confidential proprietary product travelers
- Duplicate pages

Use PyMuPDF to merge:
```python
import fitz

def compile_pdf(doc_paths: list, output_path: str):
    merged = fitz.open()
    for path in doc_paths:
        doc = fitz.open(path)
        merged.insert_pdf(doc)
        doc.close()
    merged.save(output_path)
    merged.close()
```

If PyMuPDF is not installed: `pip install pymupdf`

### Step 4 — Extract data from each part's documents

From PO: PO#, date, revision, vendor, part#, description, part revision, qty ordered

From Packing Slip: Pack slip#, vendor, PO#, part#, description, qty shipped, lot#,
Operations signature + date present, "No Discrepancies" noted, Quality signature present

From Receiving List: Order#, part#, description, qty to receive, qty received,
receiving date, receiving location, lot#, exp date, rev#, mfg date, ster date

From Manufacturer COC: PO#, vendor part#, lot#, qty, mfg date, exp date, ster date,
revision, deviations, approval signature + date, standards cited (FDA/ISO references)

From Sterilization COC (if present): Sterilizer name, work order#, irradiation date,
lot#, min/max dose spec vs result, pass/fail, signed by, date

From Manufacturer Inspection Report (if present): Dims in spec Y/N, discrepancies

From Travelers (if present): Vendor name, all fields filled or marked N/A

### Step 5 — Parse the CAD drawing

Extract:
- Company part#, description, drawing revision
- Supplier, supplier part#, supplier revision
- Material callout (or N/A), finish callout (or N/A)
- Drawn by, checked by, dates
- General tolerances from title block
- ALL numbered inspection callouts marked with the inspection symbol
  For each: number, description, nominal, upper limit, lower limit, gaging method
  (infer gaging method if not explicit: calipers for linear, pin gauge for holes, etc.)

If no numbered callouts exist -> Section 3 is N/A.

### Step 6 — Cross-reference all documents

Flag any mismatch:
- Part number consistent across PO, packing slip, receiving list, COC, drawing
- Revision consistent across PO, receiving list, COC, drawing
- Lot number consistent across packing slip, COC, receiving list
- Qty ordered (PO) = qty received (receiving list) = qty shipped (packing slip)
- PO number consistent across all documents
- Manufacturing date != sterilization date (must be different)
- COC vendor part number matches supplier part number on drawing
- If revision received differs from revision ordered: flag, note both

### Step 7 — Apply Receiving Checklist logic

2.1 - Receiving list and packing slip signed:
  "Completed" if packing slip has Operations sig + date AND receiving list is present

2.2 - Mfg certs match drawing notes:
  "N/A" if drawing material AND finish are both N/A
  "Yes" if all required material and finishing COCs are present
  "No" if any required cert is missing (note which in Section 5)
  Note: general COC covers material+finish only if it is the ONLY document given

2.3 - Manufacturer inspection report, dims in spec:
  "N/A" if no numbered inspection callouts on drawing
  "Yes" if report present and all dims pass
  "No" if discrepancies found (describe in Section 5)

2.4 - Contract manufacturer travelers reviewed:
  "N/A" if vendor uses confidential travelers (see Special Cases)
  "N/A" if off-the-shelf part with no travelers
  "Yes" if travelers present and all fields complete
  "No" if travelers missing and required

2.5 - Sterilization COC reviewed:
  "Yes" if ster COC present and dose parameters passed
  "N/A" if part is not sterilized
  "No" if parameters failed or COC missing when required

2.6 - Confidential docs / Director review:
  "Yes" ONLY for proprietary products with confidential batch records
        (Director of Operations must sign BEFORE Quality Manager)
  "No" for all other parts

Section 3:
  N/A: restore V2 to 'N/A' (right-aligned), check W2 — if no numbered inspection callouts
  If dims exist: fill chars D6-M6, criteria D7-M7, upper D8-M8, lower D9-M9, gaging D10-M10,
  uncheck W2, leave measurement rows D11:M20 blank with light blue fill DEEAF1 (engineer fills)
  DO NOT overwrite array formulas in D21:M21

Section 4 comments:
  Off-the-shelf parts: "These parts are off-the-shelf from [Vendor] therefore,
  sections 2.X & 2.X not applicable. No dimensional inspection required."
  (List only the sections that are actually N/A for this part)
  2.6 YES: "Batch records have been reviewed by Director of Operations"
  DVT accepted: "DVT accepted per report #[number]" — ask user for report number
  Clean pass with no other notes: mark section N/A

Section 5: N/A if no issues. Otherwise describe discrepancy clearly, state NCR decision.

Section 6:
  No discrepancies: 6.1 = No, 6.2 = N/A
  Raw Goods: packaged/assembled/sterilized items
  Finished Goods: reusable metal instruments, trays, anodized implants
  Discrepancy disposition requires Quality Manager sign-off per your SOPs

### Step 8 — Fill the PRF template

CRITICAL RULES:
1. ALWAYS load with openpyxl.load_workbook() — never recreate from scratch
2. Use save_preserving_drawings() (see Step 9) — never wb.save() directly
3. ONLY write to cells that have a value to fill
4. NEVER write None, "", or any empty value to any cell
5. NEVER clear, delete, or overwrite existing cell content unless replacing with real data
6. NEVER modify formatting of cells you are not writing to

Find the PRF template (QF_7_4_3-02_Rev_E_template.xlsx or equivalent) in the current
working directory tree. If not found, STOP and tell the user to place it in the folder.

SHEET "Sections 1-2" — write only these cells, only if the value is non-empty:
  E3  = PO number
  E4  = Supplier / vendor name
  E5  = Today's inspection date (M/D/YYYY format)
  E6  = Quantity received
  M3  = Product name / description
  M4  = Company product part number
  M5  = Revision
  M6  = Lot / batch number

CHECKBOX CELLS — COMPLETE VERIFIED IMPLEMENTATION

The template's VML checkboxes are invisible outside desktop Excel. The script must
explicitly write both ☑ (selected) and ☐ (unselected) to every checkbox cell on every
run — do not rely on the template having ☐ pre-baked.

Two functions are required:

```python
from openpyxl.styles import Font, Alignment

def _cb_font(cell):
    """Read the cell's existing font — never override it."""
    f = cell.font
    return Font(
        name  = f.name if f.name else 'Calibri',
        size  = f.size if f.size else 11,
        bold  = f.bold if f.bold else False,
    )

def check_box(ws, coord, with_label=None):
    """Write ☑ to coord. with_label appends the label text inline."""
    cell = ws[coord]
    cell.value = f'☑  {with_label}' if with_label else '☑'
    cell.font  = _cb_font(cell)
    cell.alignment = Alignment(
        horizontal='center' if not with_label else 'left',
        vertical='center'
    )

def uncheck_box(ws, coord, with_label=None):
    """Write ☐ to coord — makes the empty box visible."""
    cell = ws[coord]
    cell.value = f'☐  {with_label}' if with_label else '☐'
    cell.font  = _cb_font(cell)
    cell.alignment = Alignment(
        horizontal='center' if not with_label else 'left',
        vertical='center'
    )
```

ALWAYS write to EVERY checkbox cell in every row — selected gets ☑, unselected gets ☐.
Never skip a cell, never leave a checkbox cell untouched from the template.

SECTION 1.2 — Quantity Inspected:
  D8  = N/A checkbox           (label text already in cell)
  G8  = 100% Inspection checkbox  (label text already in cell)
  Write both — check the applicable one, uncheck the other.
  Alignment: horizontal='center', vertical='center'

  Off-the-shelf / no per-piece inspection: check D8 (N/A), uncheck G8
  100% inspection: uncheck D8, check G8

SECTION 1.3 — QDA:
  D13 = N/A checkbox           (label 'N/A' already in cell)
  N13 = YES checkbox           (label 'YES' already in cell)
  P13 = NO checkbox            (label 'NO' already in cell)
  Write all three every time. Alignment: horizontal='center', vertical='center'

  Off-the-shelf / no QDA: check D13 (N/A), uncheck N13, uncheck P13
  QDA done — mfg date in window: uncheck D13, check N13, uncheck P13
  QDA done — mfg date NOT in window: uncheck D13, uncheck N13, check P13

SECTION 2:
  2.1  → check_box(ws, 'I20', with_label='Completed')
         (I20 is the only option for 2.1 — no uncheck needed)

  2.2  → check ONE, uncheck the other two:
         I21 = Yes  |  L21 = No  |  O21 = N/A

  2.3  → I22 = Yes  |  L22 = No  |  O22 = N/A
  2.4  → I23 = Yes  |  L23 = No  |  O23 = N/A
  2.5  → I24 = Yes  |  L24 = No  |  O24 = N/A
  2.6  → I25 = Yes  |  L25 = No   (no N/A option for 2.6)

  Label cells K21-K25 "Yes", N21-N25 "No", Q21-Q24 "N/A" — NEVER touch.

SHEET "Section 3":
  N/A checkbox:
    V2 = "N/A" label — restore to 'N/A', right-aligned, Calibri 11
    W2 = ☑ checkbox (centered) when Section 3 is N/A
    W2 = ☐ checkbox (centered) when Section 3 has dimensions

  When Section 3 is N/A:
    ws['V2'].value = 'N/A'
    ws['V2'].font = Font(name='Calibri', size=11)
    ws['V2'].alignment = Alignment(horizontal='right', vertical='center')
    check_box(ws, 'W2')

  When Section 3 has dimensions:
    ws['V2'].value = 'N/A'
    ws['V2'].font = Font(name='Calibri', size=11)
    ws['V2'].alignment = Alignment(horizontal='right', vertical='center')
    uncheck_box(ws, 'W2')
    # Then fill characteristic data:
    D6-M6   = Characteristic descriptions
    D7-M7   = Criteria / nominal dimension
    D8-M8   = Upper limit (numeric only)
    D9-M9   = Lower limit (numeric only)
    D10-M10 = Gaging method
    D11:M20 = Do not write — apply light blue fill DEEAF1 (engineer fills)
    D21:M21 = Array formulas — DO NOT TOUCH UNDER ANY CIRCUMSTANCES

  R2 and U2 are formula-linked from Sections 1-2 — do not write to them.

SHEET "Sections 4-6":
  Section 4 N/A / comment:
    G3 = N/A checkbox
    If N/A:      check_box(ws, 'G3', with_label='N/A')   — do not write to B4
    If comment:  uncheck_box(ws, 'G3', with_label='N/A') — write comment text to B4

  Section 5 N/A / discrepancy:
    L12 = N/A checkbox
    If N/A:        check_box(ws, 'L12', with_label='N/A')    — do not write to B14
    If discrepancy: uncheck_box(ws, 'L12', with_label='N/A') — write text to B14

  Section 6.1:
    I30 = Yes checkbox  |  M30 = No checkbox
    No discrepancies: check_box(ws, 'M30'), uncheck_box(ws, 'I30')
    Discrepancies:    check_box(ws, 'I30'), uncheck_box(ws, 'M30')
                      also yellow fill FFFF00 on I30
    Label cells K30 "Yes", N30 "No" — NEVER touch.

  Section 6.2 N/A:
    B31 currently contains '6.2\n         \n       N/A'
    Overwrite with compact same-line format, vertically centered:
    ws['B31'].value = '6.2\n☐ N/A'
    ws['B31'].font = Font(name='Calibri', size=11)
    ws['B31'].alignment = Alignment(wrap_text=True, horizontal='center', vertical='center')
    (Only write this when there are NO discrepancies — 6.2 is N/A)

  Disposition:
    I36 = Raw Goods checkbox  |  M36 = Finished Goods checkbox
    Raw Goods:      check_box(ws, 'I36'), uncheck_box(ws, 'M36')
    Finished Goods: check_box(ws, 'M36'), uncheck_box(ws, 'I36')
    Label cells J36 "Raw Goods", N36 "Finished Goods" — NEVER touch.

Text cells:
  B4  = Section 4 comment text (when section has comment, replaces placeholder)
  B14 = Section 5 discrepancy text (when section has discrepancy)

Yellow fill FFFF00 — use only on cells needing manual review, never on checkbox cells.

### Step 9 — Save Excel, export PRF PDF, insert into compiled PDF

BOTH files are required every time. Do not skip either one.

CRITICAL — openpyxl strips all drawing files (section underlines, logo, form controls)
when saving normally. Always use this drawing-preserving save function:

```python
import zipfile, io
from pathlib import Path

def save_preserving_drawings(wb, template_path: str, output_path: str):
    """Save workbook while restoring all drawing/VML files from original template."""
    buf = io.BytesIO()
    wb.save(buf)
    buf.seek(0)
    saved_bytes = buf.read()

    preserve_prefixes = (
        'xl/drawings/', 'xl/media/', 'xl/worksheets/_rels/',
        'xl/ctrlProps/', 'xl/printerSettings/', 'customXml/',
    )
    preserved = {}
    with zipfile.ZipFile(template_path, 'r') as tz:
        for name in tz.namelist():
            if any(name.startswith(p) for p in preserve_prefixes):
                preserved[name] = tz.read(name)

    final = io.BytesIO()
    with zipfile.ZipFile(io.BytesIO(saved_bytes), 'r') as sz:
        with zipfile.ZipFile(final, 'w', zipfile.ZIP_DEFLATED) as oz:
            for item in sz.infolist():
                oz.writestr(item, sz.read(item.filename))
            for name, data in preserved.items():
                oz.writestr(name, data)

    with open(output_path, 'wb') as f:
        f.write(final.getvalue())
```

Usage: `save_preserving_drawings(wb, template_path, output_xlsx_path)`
Never call `wb.save(output_path)` directly.

**1. Save the filled Excel:**
  Inspections/PRF-<PartNumber>_Rev_<Revision>.xlsx

**2. Export PRF as PDF using LibreOffice headless, then stamp header on all pages:**
  Inspections/PRF-<PartNumber>_Rev_<Revision>.pdf

```python
import subprocess, shutil, os, fitz, zipfile
from pathlib import Path

def _load_font(windows_filename, fitz_fallback):
    """Load a font by Windows filename, fall back to built-in fitz font."""
    win_path = os.path.join(r'C:\Windows\Fonts', windows_filename)
    if os.path.exists(win_path):
        return fitz.Font(fontfile=win_path)
    return fitz.Font(fontname=fitz_fallback)

def export_and_stamp_pdf(xlsx_path: str, out_dir: str, template_path: str) -> str:
    """Export xlsx to PDF via LibreOffice, then stamp logo + header on all pages."""

    # Step 1: LibreOffice export
    for lo in ["libreoffice", "soffice",
               "/Applications/LibreOffice.app/Contents/MacOS/soffice",
               r"C:\Program Files\LibreOffice\program\soffice.exe"]:
        if shutil.which(lo) or Path(lo).exists():
            r = subprocess.run([lo, "--headless", "--convert-to", "pdf",
                                "--outdir", out_dir, xlsx_path],
                               capture_output=True, timeout=60)
            if r.returncode != 0:
                print("LibreOffice export failed")
                return None
            break
    else:
        print("LibreOffice not found. Install from https://libreoffice.org")
        return None

    raw_pdf = str(Path(out_dir) / (Path(xlsx_path).stem + ".pdf"))

    # Step 2: Extract logo from template
    with zipfile.ZipFile(template_path, 'r') as z:
        logo_bytes = z.read('xl/media/image1.png')

    # Step 3: Load fonts
    # Title:      Comic Sans MS Bold 13.2pt
    # Doc number: Times New Roman 7.92pt
    font_title  = _load_font('comicbd.ttf', 'helv')        # Comic Sans MS Bold
    font_docnum = _load_font('times.ttf',   'Times-Roman') # Times New Roman

    TITLE = "Quality Product Release Form"
    DOCNUM_LINES = [
        "Document Number: QF 7.4.3-02",
        "Date: 01/2026",
        "Revision E"
    ]

    # Step 4: Stamp header on every page
    tmp = raw_pdf + ".tmp"
    shutil.copy(raw_pdf, tmp)
    doc = fitz.open(tmp)

    for page in doc:
        pw = page.rect.width

        # Redact existing header area (removes LibreOffice _x000a_ artifacts)
        page.add_redact_annot(fitz.Rect(130, 0, pw, 58), fill=(1, 1, 1))
        page.apply_redactions()

        # Logo top-left
        page.insert_image(fitz.Rect(28, 8, 133, 46), stream=logo_bytes)

        # Title: Comic Sans MS Bold 13.2pt, centered
        tw = font_title.text_length(TITLE, fontsize=13.2)
        tx = (pw / 2) - (tw / 2)
        page.insert_text((tx, 30), TITLE,
                         fontsize=13.2, font=font_title, color=(0, 0, 0))

        # Doc number: Times New Roman 7.92pt, top-right, 3 lines
        line_h  = 8.5
        x_right = pw - 5
        y_start = 18
        for i, line in enumerate(DOCNUM_LINES):
            lw = font_docnum.text_length(line, fontsize=7.92)
            page.insert_text((x_right - lw, y_start + i * line_h), line,
                             fontsize=7.92, font=font_docnum, color=(0, 0, 0))

    doc.save(raw_pdf, garbage=4, deflate=True)
    doc.close()
    os.unlink(tmp)

    return raw_pdf
```

**3. Insert PRF PDF into compiled package:**
Build the compiled PDF in one merge operation in the exact order listed in Step 3.
The PRF PDF is document #5 (after Drawing, before COCs).

```python
import fitz

def compile_pdf(doc_paths: list, output_path: str):
    merged = fitz.open()
    for path in doc_paths:
        doc = fitz.open(path)
        merged.insert_pdf(doc)
        doc.close()
    merged.save(output_path)
    merged.close()
```

**Output confirmation — report both files in the summary:**
  Inspections/PRF-[PART-NUMBER]_Rev_[REV].xlsx  saved ✓
  Inspections/PRF-[PART-NUMBER]_Rev_[REV].pdf   exported ✓

---

## SPECIAL CASES

Off-the-shelf purchased parts (no internal manufacturing):
  2.4 = N/A, 2.6 = No (unless confidential batch record product)
  Section 3 = N/A
  Section 4: "These parts are off-the-shelf from [Vendor] therefore, sections 2.4 & 2.6
  not applicable. No dimensional inspection required."

Confidential batch record products:
  2.6 = Yes — flag clearly: Director of Operations reviews and signs BEFORE Quality Manager

Revision mismatch (ordered rev A, received rev B):
  Include BOTH drawings in compiled PDF
  Use RECEIVED revision for all PRF fields
  Describe in Section 5, flag for engineer review

DVT accepted:
  Section 4: "DVT accepted per report #______" — ask user for the report number
  DVT does NOT count for AQL — mark 1.2 AQL as N/A if only DVT was done

Multiple finishings/materials on drawing:
  One COC per material AND one per finishing required
  General COC only acceptable if it is the sole document given
  Flag any missing COCs in Section 5

No drawing on file:
  Section 3 = N/A, 2.2 = N/A, 2.3 = N/A
  Section 4: "No engineering drawing on file for this part."

---

## OUTPUT SUMMARY FORMAT

  =========================================
  [PO Number] - Processing Complete
  =========================================
  Vendor: [Vendor] | Parts processed: [N]

  Part 1: [PART#]  Rev [X]  Lot [LOT]
    In-Progress/[PO] Receiving Part 1 - [PART#].pdf  compiled ([N] pages)
    Inspections/PRF-[PART#]_Rev_[X].xlsx  saved
    Inspections/PRF-[PART#]_Rev_[X].pdf   exported
    2.1 Completed | 2.2 [X] | 2.3 [X] | 2.4 [X] | 2.5 [X] | 2.6 [X]
    Section 3: [N/A or N characteristics]
    Disposition: [Raw Goods / Finished Goods]
    Flags: [None or description]
  Cross-reference: [all checks passed / flags listed]

  =========================================
  Next steps:
  - Resolve any flags before signing
  - Fill blue measurement cells in Section 3 if applicable
  - Sign compiled PDF and route for QA signatures
  =========================================

---

## WHAT THE ENGINEER DOES AFTER

- Resolve any yellow-flagged cells
- Fill measured dimension values in light blue Section 3 cells
- Sign the compiled PDF and send out for signatures per your QMS SOPs:
  - 2.4 YES → Quality Manager initials required
  - 2.6 YES → Director of Operations signs first
  - Section 6 discrepancies → designated reviewer signs before Quality Manager
  - Final Quality Manager signature on Section 6

---

## TOOLS TO USE

- Read — to read PDFs and the Excel template (current directory only)
- Write / Edit — to write filled PRF Excel files and compiled PDFs (current directory only)
- Python with `anthropic`, `pdfplumber`, `openpyxl`, `pymupdf` — extraction, Excel, PDF merge
- Claude API (`claude-opus-4-6`) — document classification, extraction, decisions
- LibreOffice headless — PRF PDF export

Install all Python packages:
  pip install anthropic pdfplumber openpyxl pymupdf

Every file path must be constructed relative to `pathlib.Path.cwd()`. Reject any path
that resolves outside the current working directory.

---
*Last updated: May 2026*
*Designed for ISO 13485 / FDA 21 CFR Part 820 receiving inspection workflows.*
*Adapt the PRF template, signing rules, and vendor-specific logic to your QMS.*
