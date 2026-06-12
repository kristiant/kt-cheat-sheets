# Docling

**What it is:** An open-source (IBM, MIT-licensed) Python library that converts PDFs, Office docs, and HTML into clean, structured Markdown/JSON — with real layout analysis, table reconstruction, and reading-order recovery.

**Why people use it:** Produces RAG-ready Markdown locally — no per-page API fees (cf. Adobe PDF Extract ~$0.003/page), no data leaving your infra. `HybridChunker` produces chunks that respect headings and tables.

**Typically used for:** RAG ingestion pipelines — PDF → Markdown → chunks → embeddings. Also table extraction and OCR of scanned docs.

---

## Ways to run it

Docling's engine is **Python** — there is no pure-JS converter. Three ways to use it, by where Python lives:

| Scenario | Use | Python in your codebase? |
|---|---|---|
| Python service / Lambda | `pip install docling` directly | Yes |
| TS/Node app | **Docling Serve** (Docker) + HTTP | No — runs in the container |
| Reading Docling JSON in TS | `@docling/docling-core` | No |

For a TS stack, run **Docling Serve in Docker** and call it over HTTP — see [Calling from TypeScript](#calling-from-typescript).

---

## Core usage (Python)

```bash
pip install docling          # needs Python 3.10+
```

```python
from docling.document_converter import DocumentConverter

doc = DocumentConverter().convert("report.pdf").document
print(doc.export_to_markdown())          # also: export_to_dict(), export_to_html()
```

Source can be a local path or a URL:

```python
doc = DocumentConverter().convert("https://arxiv.org/pdf/2408.09869").document
```

CLI:

```bash
docling report.pdf --to md
docling report.pdf --to json --output ./out
docling https://arxiv.org/pdf/2206.01062 --to md
```

---

## HybridChunker (RAG)

Splits along document structure (headings, tables, lists), token-aware, and repeats table headers in each chunk so chunks are self-contained. Prefer it over hand-rolled Markdown splitting.

```python
from docling.document_converter import DocumentConverter
from docling.chunking import HybridChunker

doc = DocumentConverter().convert("handbook.pdf").document

chunker = HybridChunker(max_tokens=1500, merge_peers=True)

for chunk in chunker.chunk(dl_doc=doc):
    text = chunker.contextualize(chunk)   # ← prepends heading path; embed THIS
    embed(text)
```

`chunk.text` is raw text; `chunker.contextualize(chunk)` prefixes it with the heading path ("Benefits > Leave > Annual Leave: ..."). Embed the contextualised version.

> **Grounded:** Recommended RAG path in Docling's [hybrid chunking docs](https://docling-project.github.io/docling/examples/hybrid_chunking/). RHEL AI and OpenSearch both ship Docling-`HybridChunker` ingestion as their reference pipeline.

---

## OCR & pipeline options

OCR is off by default (born-digital PDFs extract better without it). Enable for scans:

```python
from docling.document_converter import DocumentConverter, PdfFormatOption
from docling.datamodel.pipeline_options import PdfPipelineOptions
from docling.datamodel.base_models import InputFormat

opts = PdfPipelineOptions(do_ocr=True, do_table_structure=True)

converter = DocumentConverter(
    format_options={InputFormat.PDF: PdfFormatOption(pipeline_options=opts)}
)
doc = converter.convert("scanned.pdf").document
```

---

## Calling from TypeScript

Run the engine as a container, call it over HTTP:

```bash
docker run -p 5001:5001 ghcr.io/docling-project/docling-serve:latest
```

With [`docling-sdk`](https://github.com/btwld/docling-sdk):

```ts
import { Docling } from "docling-sdk";

const client = new Docling({ api: { baseUrl: "http://localhost:5001" } });
const result = await client.convert("./report.pdf", "report.pdf", { to_formats: ["md"] });

console.log(result.document.md_content);
```

Or the REST API directly (no SDK):

```ts
const res = await fetch("http://localhost:5001/v1/convert/source", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ http_sources: [{ url: "https://arxiv.org/pdf/2206.01062" }] }),
});
const { document } = await res.json();   // document.md_content
```

---

## Practical recipes

**Convert one PDF to Markdown (CLI):**
```bash
docling report.pdf --to md
```

**Convert every PDF in a folder (CLI):**
```bash
docling ./docs --to md --output ./converted
```

**RAG ingest: PDF → contextualised chunks (Python):**
```python
from docling.document_converter import DocumentConverter
from docling.chunking import HybridChunker

doc = DocumentConverter().convert("manual.pdf").document
chunker = HybridChunker(max_tokens=1500, merge_peers=True)
chunks = [chunker.contextualize(c) for c in chunker.chunk(dl_doc=doc)]
```

**Run the engine for a TS app (Docker), convert over HTTP:**
```bash
docker run -p 5001:5001 ghcr.io/docling-project/docling-serve:latest
```
```ts
await new Docling({ api: { baseUrl: "http://localhost:5001" } })
  .convert("./report.pdf", "report.pdf", { to_formats: ["md"] });
```

**Cache the structured JSON; re-derive Markdown later for free (Python):**
```python
import json
doc = DocumentConverter().convert("input.pdf").document
json.dump(doc.export_to_dict(), open("input.docling.json", "w"))
# later: rebuild markdown/chunks from JSON without re-parsing the PDF
```

**OCR a scanned PDF only when text extraction is poor (CLI):**
```bash
docling scanned.pdf --to md --ocr
```

---

## Gotchas & tips

- Embed `contextualize(chunk)`, not `chunk.text` — else chunks lose heading context.
- Parsing is expensive and deterministic — convert once, cache `export_to_dict()` JSON, re-derive from that.
- Keep OCR off for born-digital PDFs; slower and less accurate than native extraction.
- In AWS Lambda, package as a container image — Docling's ML models are too big for a zip layer.
- `@docling/docling-core` only reads Docling JSON; conversion needs the Python engine (direct or Serve).
- MIT-licensed, fully local — safe for on-prem / data-residency.
