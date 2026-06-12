# Docling

**What it is:** A document conversion toolkit that parses PDFs, Office files, HTML, images, and more into structured documents (Markdown, JSON, HTML).

**Why people use it:** Extracts text, tables, layout, and reading order into formats useful for RAG — without sending files to a SaaS API.

**Typically used for:** PDF-to-Markdown, RAG preprocessing, table extraction, OCR workflows, and bulk document ingestion pipelines.

---

## Two TS packages to know

| Package | What it does |
|---|---|
| [`@docling/docling-core`](https://github.com/docling-project/docling-ts) | Official. TypeScript types + utilities for consuming Docling's JSON output. |
| [`docling-sdk`](https://github.com/btwld/docling-sdk) | Third-party. Runs Docling conversions from TS via Docling Serve API, the CLI wrapper, or in-browser WebGPU/WASM. |

---

## @docling/docling-core — consuming Docling output in TS

Install:

```bash
npm i @docling/docling-core
```

Parse and iterate a Docling JSON file:

```ts
import { iterateDocumentItems, isDocling } from "@docling/docling-core";
import conversion from "./input.docling.json";

for (const [item, level] of iterateDocumentItems(conversion)) {
  if (isDocling.TextItem(item)) {
    console.log(item.text);
  }
}
```

Useful for post-processing: chunking, filtering by item type, extracting tables, mapping to embeddings.

---

## docling-sdk — running conversions from TS

Install:

```bash
npm i docling-sdk
```

Three client modes — pick one based on your setup:

### API client (Docling Serve running somewhere)

```ts
import { DoclingClient } from "docling-sdk";

const client = new DoclingClient({ baseUrl: "http://localhost:5001" });
const result = await client.convert({ source: "./report.pdf" });

console.log(result.document.export_to_markdown());
```

### CLI client (Python Docling installed locally)

```ts
import { DoclingCLIClient } from "docling-sdk";

const client = new DoclingCLIClient();
const result = await client.convert({ source: "./report.pdf" });
```

### Web client (browser — no server needed, runs via WebGPU/WASM)

```ts
import { DoclingWebClient } from "docling-sdk";

const client = new DoclingWebClient();
const result = await client.convert({ source: file }); // File object
```

---

## Spinning up Docling Serve (for the API client)

Docling Serve is a REST API wrapper around the Python library:

```bash
docker run -p 5001:5001 ghcr.io/docling-project/docling-serve:latest
```

Then point `DoclingClient` at `http://localhost:5001`. You get full Docling conversion over HTTP, no Python in your TS project.

---

## Advanced

### Chunking for RAG (TS)

```ts
import { iterateDocumentItems, isDocling } from "@docling/docling-core";

const chunks: string[] = [];
let buffer = "";

for (const [item] of iterateDocumentItems(conversion)) {
  if (isDocling.TextItem(item)) {
    buffer += item.text + "\n";
    if (buffer.length > 1000) {
      chunks.push(buffer.trim());
      buffer = "";
    }
  }
}
if (buffer) chunks.push(buffer.trim());
```

### Pair with LangChain (Python side)

```bash
pip install langchain-docling
```

```python
from langchain_docling import DoclingLoader

loader = DoclingLoader(file_path="report.pdf")
docs = loader.load()
```

### Python CLI — quick one-offs

When you just need a file converted and don't need TS:

```bash
pip install docling
docling report.pdf --to md
docling report.pdf --to json
docling https://arxiv.org/pdf/2206.01062 --to md
docling ./input-docs --to md --output ./converted
```

### Python API — batch conversion pipeline

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

---

## Supported input types

```
PDF, DOCX, PPTX, XLSX, HTML, EPUB, PNG, JPEG, TIFF, EML, LaTeX
```

## Output formats

```
Markdown, JSON, HTML, DocTags
```

---

## Practical recipes

**Run Docling Serve locally via Docker and convert via HTTP:**
```bash
docker run -p 5001:5001 ghcr.io/docling-project/docling-serve:latest
```
```ts
const result = await new DoclingClient({ baseUrl: "http://localhost:5001" })
  .convert({ source: "./report.pdf" });
```

**Convert a PDF to Markdown in one line (Python CLI):**
```bash
docling report.pdf --to md
```

**Convert all PDFs in a folder (bash + Python CLI):**
```bash
for f in *.pdf; do docling "$f" --to md --output converted; done
```

**Load a Docling JSON and extract only text items (TS):**
```ts
import { iterateDocumentItems, isDocling } from "@docling/docling-core";

const text = [...iterateDocumentItems(doc)]
  .filter(([item]) => isDocling.TextItem(item))
  .map(([item]) => (item as any).text)
  .join("\n");
```

**Cache the JSON output, parse cheaply later:**
```python
Path("input.docling.json").write_text(json.dumps(doc.export_to_dict()))
```

---

## Tips

- Use Docling Serve + `DoclingClient` to keep conversion in Python while calling it from TS — best of both.
- `@docling/docling-core` is for *consuming* already-converted JSON output, not running conversions.
- `docling-ts` (the official project) is an unstable draft — expect API changes.
- Convert once, cache JSON. PDF parsing is expensive.
- Prefer Markdown for RAG text; keep JSON when table structure or provenance matters.
- `docling-sdk` runs in Node, Bun, Deno, browsers, and Cloudflare Workers.
