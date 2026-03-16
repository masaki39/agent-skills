---
name: pdf-extract
description: Extract figures and charts from PDF using pdfimages CLI. Use when user wants to extract images/figures from a PDF file.
allowed-tools: Bash, Read, Glob
---

# PDF Figure Extraction Skill

Extract meaningful figures, charts, diagrams, and tables from a PDF file using `pdfimages`, then classify each image with AI to keep only content images (discarding logos, icons, and decorations).

## Step 1: Check pdfimages installation

Run:
```bash
which pdfimages || echo "NOT FOUND"
```

If not found, instruct the user to install poppler and stop:
- macOS: `brew install poppler`
- Linux (Debian/Ubuntu): `sudo apt install poppler-utils`
- Linux (RHEL/Fedora): `sudo dnf install poppler-utils`
- Windows: `choco install poppler` or `winget install poppler`

## Step 2: Get inputs

Ask the user for:
1. PDF file path (or infer from conversation context)
2. Output directory path (where to save the extracted figures)

## Step 3: Extract all images to a temp directory

Create a temp directory:
```bash
TMPDIR=$(mktemp -d)
echo $TMPDIR
```

Extract all images from the PDF as PNG files:
```bash
pdfimages -png -p "<pdf_path>" "$TMPDIR/img"
```

- `-png`: output as PNG
- `-p`: include page number in filename (e.g. `img-001-000.png`)
- Always quote paths to handle spaces

Then list what was extracted:
```bash
ls "$TMPDIR"/*.png 2>/dev/null | wc -l
ls "$TMPDIR"/*.png 2>/dev/null
```

If no PNG files found, report and exit.

## Step 4: Classify each image with AI

For each extracted PNG file, use the `Read` tool to load the image and classify it.

Classification criteria:
- **KEEP**: figure / chart / graph / diagram / table / screenshot / meaningful content image from an academic paper or technical document
- **SKIP**: logo / icon / decorative element / background / border / bullet point / small graphic

Use this classification prompt mentally when viewing each image:
> "This image is from a PDF. Is it a figure, chart, diagram, table, or meaningful content image (not a logo, icon, or decoration)? Classify as KEEP or SKIP."

Process each image one by one, noting the decision.

## Step 5: Copy KEEP images to output directory

Ensure the output directory exists:
```bash
mkdir -p "<output_dir>"
```

For each KEEP image:
```bash
cp "<tmpdir>/<image_filename>" "<output_dir>/<image_filename>"
```

## Step 6: Clean up and report

Remove the temp directory:
```bash
rm -rf "$TMPDIR"
```

Report to the user:
- Total images extracted
- Number of KEEP images saved
- Number of SKIP images discarded
- List of saved files in the output directory
