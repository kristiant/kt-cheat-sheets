# Docling

**What it is:** A Python document conversion toolkit for parsing PDFs, Office files, HTML, images, audio transcripts, and more into structured documents.

**Why people use it:** It extracts text, layout, tables, figures, reading order, and metadata into formats that are useful for RAG and document AI.

**Typically used for:** PDF-to-Markdown, document ingestion pipelines, RAG preprocessing, table extraction, OCR workflows, and converting files to structured JSON.

> Source: https://docling-project.github.io/docling/ and https://www.docling.ai/

---

## Setup

```bash
python -m venv .venv
source .venv/bin/activate
pip install docling
```

Docling requires Python 3.10+ in current releases.

## Most common commands

```bash
docling input.pdf
docling input.pdf --to md
docling input.pdf --to json
docling https://arxiv.org/pdf/2206.01062
docling ./docs
docling --help
```

> The official start page shows `pip install docling`, `docling <url>`, and `DocumentConverter().convert(...)`.

---

## Convert in Python

```python
from docling.document_converter import DocumentConverter

source = "input.pdf"
converter = DocumentConverter()
result = converter.convert(source)

print(result.document.export_to_markdown())
```

Convert a URL:

```python
from docling.document_converter import DocumentConverter

source = "https://arxiv.org/pdf/2408.09869"
doc = DocumentConverter().convert(source).document

print(doc.export_to_markdown())
```

Export JSON:

```python
import json
from docling.document_converter import DocumentConverter

doc = DocumentConverter().convert("input.pdf").document

with open("input.docling.json", "w") as f:
    json.dump(doc.export_to_dict(), f, indent=2)
```

Export HTML:

```python
doc = DocumentConverter().convert("input.pdf").document

with open("input.html", "w") as f:
    f.write(doc.export_to_html())
```

---

## CLI examples

Convert a PDF to Markdown:

```bash
docling report.pdf --to md
```

Convert to JSON:

```bash
docling report.pdf --to json
```

Convert a URL:

```bash
docling https://arxiv.org/pdf/2206.01062 --to md
```

Convert a directory:

```bash
docling ./input-docs --to md
```

Write output to a directory:

```bash
mkdir -p converted
docling ./input-docs --to md --output converted
```

---

## Supported input types

Docling's docs list many formats, including:

```text
PDF, DOCX, PPTX, XLSX, HTML, EPUB,
PNG, JPEG, TIFF, WAV, MP3, WebVTT,
EML, MSG, LaTeX, plain text
```

## Useful output types

```text
Markdown
HTML
JSON
DocTags
DocLang
WebVTT
```

---

## Basic ingestion pipeline

```python
from pathlib import Path
from docling.document_converter import DocumentConverter

converter = DocumentConverter()

for path in Path("docs").glob("*.pdf"):
    doc = converter.convert(path).document
    out = Path("markdown") / f"{path.stem}.md"
    out.parent.mkdir(exist_ok=True)
    out.write_text(doc.export_to_markdown())
```

## Chunk for RAG

Simple Markdown chunking:

```python
from docling.document_converter import DocumentConverter

doc = DocumentConverter().convert("handbook.pdf").document
markdown = doc.export_to_markdown()

chunks = [
    markdown[i : i + 1200]
    for i in range(0, len(markdown), 1000)
]
```

Keep source metadata:

```python
records = [
    {
        "id": f"handbook-{i}",
        "text": chunk,
        "metadata": {"source": "handbook.pdf", "chunk": i},
    }
    for i, chunk in enumerate(chunks)
]
```

## Pair with LangChain

```bash
pip install langchain-docling
```

```python
from langchain_docling import DoclingLoader

loader = DoclingLoader(file_path="report.pdf")
docs = loader.load()
```

> LangChain documents a Docling loader for parsing files into documents suitable for RAG.

---

## OCR notes

Use OCR only when needed:

```bash
docling scanned.pdf --to md
```

For born-digital PDFs, OCR can be slower and less accurate than native text extraction.

## Batch conversion

```python
from pathlib import Path
from docling.document_converter import DocumentConverter

converter = DocumentConverter()

for source in Path("inbox").iterdir():
    if source.suffix.lower() not in {".pdf", ".docx", ".pptx", ".html"}:
        continue

    try:
        doc = converter.convert(source).document
        Path("out", source.stem + ".md").write_text(doc.export_to_markdown())
    except Exception as exc:
        print(f"FAILED {source}: {exc}")
```

## Tables

Markdown export usually preserves tables well enough for RAG:

```python
doc = DocumentConverter().convert("statement.pdf").document
markdown = doc.export_to_markdown()
```

For high-value tables, inspect JSON:

```python
data = doc.export_to_dict()
```

---

## Practical recipes

**Convert every PDF in a directory to Markdown:**
```bash
for f in *.pdf; do docling "$f" --to md --output converted; done
```

**Convert a remote PDF and inspect the first lines:**
```bash
docling https://arxiv.org/pdf/2206.01062 --to md | head -40
```

**Convert once, then write Markdown yourself:**
```python
doc = DocumentConverter().convert("input.pdf").document
Path("input.md").write_text(doc.export_to_markdown())
```

**Store lossless-ish structured output for later processing:**
```python
Path("input.docling.json").write_text(json.dumps(doc.export_to_dict()))
```

**Skip unsupported files in a folder:**
```python
allowed = {".pdf", ".docx", ".pptx", ".xlsx", ".html", ".png", ".jpg", ".jpeg"}
files = [p for p in Path("docs").iterdir() if p.suffix.lower() in allowed]
```

**Use Docling before embedding:**
```python
markdown = DocumentConverter().convert("manual.pdf").document.export_to_markdown()
chunks = markdown.split("\n## ")
```

---

## Tips

- Prefer Markdown for RAG text; keep JSON when layout, tables, or provenance matter.
- Convert documents once and cache outputs. PDF parsing is expensive.
- OCR scanned documents; avoid OCR for normal PDFs unless extraction is poor.
- Store source filename, page/chunk IDs, and conversion timestamp with embeddings.
- Inspect a few converted files manually before bulk-ingesting a corpus.
- Keep sensitive documents local; Docling can run locally without sending files to a SaaS API.
