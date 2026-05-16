# File Processing

> Last updated: 2026-05-15

## TL;DR

A comparison-driven catalogue of Python file-extraction libraries for an async FastAPI service. Pre-decides the *comparison framework* (which libraries to consider per content type, which decision axes matter) and the default service-boundary shape; does **not** pre-decide which library to use — that's a business-layer decision.

**Use this when:**
- choosing a file-extraction library for a new content type
- evaluating a swap (e.g., from `pypdfium2` to `pymupdf4llm` for layout)
- defining the file-extraction port shape for a service

**Don't use this for:**
- vector-store indexing → [`./milvus.md`](./milvus.md)
- caching parsed results → [`./py-cache.md`](./py-cache.md)
- durable upload storage → [`./py-aws-s3.md`](./py-aws-s3.md)
- HTTP fetching → [`./py-web-crawling.md`](./py-web-crawling.md)

## Table of Contents

| Phase | Section |
|-------|---------|
| 1. Concepts | [Decision Axes](#decision-axes) |
| 2. Compare | [PDFs (Digital)](#pdfs-digital), [PDFs (Scanned) — Cloud API](#pdfs-scanned--cloud-api), [PDFs (Scanned) — Local Execution](#pdfs-scanned--local-execution), [Image OCR](#image-ocr), [Office Documents](#office-documents), [HTML](#html), [Markdown](#markdown), [Structured Data](#structured-data) |
| 3. Wire | [Wire Up](#wire-up) |
| 4. Operate | [Async Wrapping](#async-wrapping), [Testing](#testing), [Production Checklist](#production-checklist), [Pitfalls](#pitfalls) |
| 5. Specialize | [Domain Specialization](#domain-specialization) |

## Decision Axes

Eleven factors a coding agent (or the business stakeholder being consulted) weighs when picking a file-extraction library. Compare every candidate against the same axes. Volatile table facts such as pricing, release recency, model size, and 2026 maturity must be verified against current upstream or vendor sources before production or procurement decisions.

1. **Layout parsing.** Preserves paragraphs, headers, lists, and tables? Citation-quality extraction needs this; downstream RAG retrieval ranking degrades without it.

2. **BBox output.** Coordinate output per element. Normalized `[0, 1]` (Textract, Azure DI) vs pixel-space (pdfplumber, PyMuPDF). Required for UI highlighting and source-page recall.

3. **Language coverage.** English-only, multilingual, or specific language list. Tesseract degrades silently on non-English without explicit `lang=`. PaddleOCR / olmOCR / Google Document AI dominate multilingual.

4. **Execution model — Cloud API vs Local execution.** Cloud APIs (AWS Textract, Azure Document Intelligence, Google Document AI, Mistral OCR, LlamaParse, Mathpix) trade per-call billing for managed infrastructure, automatic accuracy upgrades, and IAM-bound credentials. Local execution (`docling`, `marker-pdf`, `unstructured`, Tesseract, PaddleOCR, vision-language models) trades compute and model storage for full control, air-gapped deployment, and no per-page billing.

5. **Cost / volume curve.** Cloud per-page billing scales linearly; local execution is fixed compute that amortizes against volume. Crossover point is workload-specific.

6. **Model size / deployment footprint.** Tesseract ~10 MB; pypdfium2 ~15 MB; pymupdf4llm ~40 MB; PaddleOCR ~500 MB; docling/marker-pdf ~500 MB–1 GB; vision-language models 7 GB–150 GB. Edge / serverless filter on this axis first.

7. **Speed / latency.** Approximate ms per page on commodity hardware. Hot-path callers need single-digit ms; batch ingestion tolerates seconds.

8. **License.** AGPL (`pymupdf4llm`, PyMuPDF) limits commercial use without source disclosure. MIT / Apache 2.0 / BSD are permissive defaults. Proprietary cloud APIs have separate terms.

9. **2026 maturity.** Active maintenance, last release, production adoption. `EasyOCR` is effectively deprecated; `Nougat` is stable but lower activity; new entrants like olmOCR need more bake-in for prod.

10. **GPU dependency.** CPU-only (Tesseract, pymupdf4llm); GPU-accelerated (PaddleOCR, marker-pdf — works on CPU, much faster on GPU); GPU-required (olmOCR, vision-language models). Filter early if no GPU is in the deployment.

11. **Use when (per-library, in the comparison tables below).** One-line guidance describing the business context where the library is the right pick.

## PDFs (Digital)

PDFs with an embedded text layer. Byte parsers extract directly — no OCR.

| Library | Layout | BBox | Lang | Exec | Cost | Size | Speed | License | 2026 | GPU | Use when |
|---|---|---|---|---|---|---|---|---|---|---|---|
| `pymupdf4llm` | Yes (Markdown) | No | 100+ | Local | Free | 40 MB | 120–150 ms | **AGPL-3.0** | Active | No | RAG pipelines needing Markdown + structure; commercial use requires license review |
| `pypdfium2` | No | Pixel | 100+ | Local | Free | 15 MB | 30–50 ms | Apache 2.0 | Active | No | Bulk text extraction at speed; Lambda / serverless |
| `pdfplumber` | Tables + chars | Pixel (char-level) | English-primary | Local | Free | 30 MB | 100–150 ms | MIT | Active | No | Financial / tabular PDFs; character-level coords |
| `PyMuPDF` (raw) | Basic | Pixel | 100+ | Local | Free | 20 MB | 50–100 ms | **AGPL-3.0** | Active | No | Direct PyMuPDF API access (no Markdown layer); commercial use requires license review |
| `pikepdf` | No | Text blocks | N/A | Local | Free | 15 MB | 80–120 ms | MPL-2.0 | Mature | No | Encryption-aware reads; PDF manipulation |
| `pdfminer.six` | Char-level | Pixel | Multiple | Local | Free | 25 MB | 200–300 ms | MIT | Stable (lower activity) | No | Fine-grained layout analysis; legacy systems |
| `pypdf` | None | None | English-primary | Local | Free | 5 MB | 200–500 ms | BSD-3-Clause | Active | No | Zero-C-deps constraint; simple documents |

See also: [pymupdf4llm docs](https://pymupdf.readthedocs.io/en/latest/pymupdf4llm/), [pypdfium2 docs](https://pypdfium2.readthedocs.io/), [pdfplumber repo](https://github.com/jsvine/pdfplumber).

## PDFs (Scanned) — Cloud API

Image-only PDFs (no text layer); OCR required. Cloud APIs trade per-call billing for managed accuracy.

| Service | Layout | BBox | Lang | Cost | Latency | 2026 | Use when |
|---|---|---|---|---|---|---|---|
| AWS Textract | Tables (Analysis API), forms | Normalized | 30+ | Detect $1.50 / Analyze $15 / Forms $65 per 1M pages | 500–2000 ms | Mature | AWS-native; tight IAM; forms/receipts/tables |
| Azure Document Intelligence | Prebuilt invoice / receipt / ID models | Normalized polygon | 200+ | Read $1.50 / Prebuilt $10 per 1M pages | 300–1500 ms | Mature | Highest reported accuracy; strongest tables; multilingual |
| Google Document AI | Custom processors | Normalized | 100+ (context-dependent) | OCR $0.60 / processor-tier varies per 1M pages | 400–1800 ms | Mature | >5M pages/month volume; best CJK and multilingual |
| Mistral OCR | Tables, equations, layouts | Pixel + semantic | 90+ | ~$0.006/page | 800–3000 ms | New (2025) | Complex documents requiring math / formula support |
| LlamaParse v2 | AI-driven structure | Logical (Markdown-like) | 90+ | Fast $0.003 / Cost-effective $0.009 / Agentic $0.030 per page | 600–2500 ms | Active (v2 2025) | RAG-optimized; Markdown output native |
| Mathpix | Equations, formulas | Pixel + semantic | Math-centric | $0.02–0.05/page | 1000–4000 ms | Specialist | Scientific PDFs; equations; chemical structures |

Bounding boxes are normalized `[0, 1]` ratios (top-left origin) for Textract, Azure DI, Google Document AI. Multiply by page width/height to get pixels; never treat the floats as pixel coordinates directly. IAM permissions (Textract example): `textract:DetectDocumentText`, `textract:AnalyzeDocument` — plus `textract:StartDocumentAnalysis` / `textract:GetDocumentAnalysis` for the async path (>10 MB / 11 pages).

See also: [Textract API reference](https://docs.aws.amazon.com/textract/latest/dg/API_Reference.html), [Azure Document Intelligence](https://learn.microsoft.com/azure/ai-services/document-intelligence/), [Google Document AI](https://cloud.google.com/document-ai/docs), [Mistral OCR](https://docs.mistral.ai/capabilities/vision), [LlamaParse v2](https://www.llamaindex.ai/blog/introducing-llamaparse-v2-simpler-better-cheaper).

## PDFs (Scanned) — Local Execution

Self-hosted alternatives to cloud OCR. Trade compute and model storage for control + no per-page billing.

| Library | Layout | BBox | Lang | Size | Speed | License | 2026 | GPU | Use when |
|---|---|---|---|---|---|---|---|---|---|
| `docling` | DocLayNet (tables, headers, lists) | Normalized + semantic | 80+ (via Tesseract / EasyOCR backends) | 500 MB | 3000–4500 ms | MIT | Active | Recommended | Enterprise RAG; accuracy > speed; table precision; 58k+ GitHub stars |
| `marker-pdf` | Surya + optional LLM cleanup | Logical structure | 90+ (Surya-backed) | 800 MB | 1500–3000 ms (+30s with LLM) | Apache 2.0 | Active | Recommended | General-purpose; layout-faithful Markdown extraction |
| `unstructured` | Chunking + NLP partitioning | Logical elements | 100+ | 300 MB–1 GB | 2000–6000 ms | Elastic License + SSPL | Active | Optional | Typed elements (`Title`, `Table`, `NarrativeText`); batch ingestion; hybrid cloud/local |
| `olmOCR` | Tables, equations, handwriting | BBox + reading order | 100+ | 7B–13B params (35–65 GB) | 2000–5000+ ms | Apache 2.0 | New (Feb 2025, v0.4 Oct 2025) | Required (vLLM / SGLang multi-GPU) | Maximum accuracy on complex docs; handwriting; AllenAI-backed |
| `surya-ocr` | Layout + text detection | BBox | 90+ | 500 MB | 1500–2500 ms | Apache 2.0 | Active (powers marker-pdf) | Recommended | Standalone layout + OCR; multi-column, tables |
| `Nougat` | Academic-document-specific | Logical (Markdown) | English-primary | 340 MB | 4000–8000 ms | **CC-BY-NC 4.0** (non-commercial) | Lower activity | Recommended | arXiv-style scientific papers only |

See also: [docling docs](https://ds4sd.github.io/docling/), [marker-pdf repo](https://github.com/VikParuchuri/marker), [unstructured docs](https://docs.unstructured.io/), [olmOCR (AllenAI)](https://allenai.org/blog/olmocr), [Surya OCR](https://github.com/VikParuchuri/surya).

## Image OCR

PNG / JPEG / TIFF / other image formats. OCR-only path.

| Library | Layout | BBox | Lang | Size | Speed | License | 2026 | GPU | Use when |
|---|---|---|---|---|---|---|---|---|---|
| Tesseract (`pytesseract`) | Line/word segments | Pixel (HOCR) | 100+ | 10 MB | 160–200 ms | Apache 2.0 | Mature | No | Speed-critical; clean scans; minimal deps; Lambda-friendly |
| PaddleOCR | Column + table detection | Pixel + confidence | 80+ (strong CJK) | 500 MB (PP-OCRv4) | 4850 ms (accurate) | Apache 2.0 | Active | Optional | Best multilingual accuracy; structured layouts |
| `RapidOCR` | Basic block detection | Pixel | 20–30 | 80 MB (ONNX-optimized) | 212 ms (fast) | Apache 2.0 | Active (Apr 2026) | No | Lightweight deployment; minimal framework bloat |
| `Surya-OCR` | Advanced layout analysis | BBox + structure | 90+ | 500 MB | 1500–2500 ms | Apache 2.0 | Active | Recommended | Multi-column, complex layouts, mixed scripts |
| `DocTR` | Page layout understanding | Normalized | 15+ | 400 MB | 1800–2200 ms | Apache 2.0 | Mature (lower activity) | Optional | Structured document layout analysis |
| Qwen2.5-VL | Tables, equations, mixed scripts | BBox + semantic | 90+ | 7B–72B params (15–150 GB) | 3000–8000 ms | Research license | New (late 2025) | Required | Max accuracy on complex docs; tables, formulas |
| MiniCPM-o v2.6 | Tables, handwriting | Pixel + semantic | 100+ | 8B params (16 GB) | 2500–4000 ms | **CC-BY-NC 4.0** | Tops OCRBench 2026 | Required | Highest OCR benchmark accuracy; handwriting |
| Mistral OCR (Cloud API) | Equations, tables, handwriting | Pixel | 90+ | N/A (cloud) | 800–3000 ms | Proprietary API | New (2025) | Managed | Complex docs; math/formulas; cloud preference |
| ~~`EasyOCR`~~ | ~~Basic~~ | ~~Pixel~~ | ~~80+~~ | ~~1.5 GB~~ | ~~656 ms~~ | ~~Apache 2.0~~ | **Deprecated** — systematic financial-symbol failures | ~~Optional~~ | **Do not use** |

See also: [pytesseract repo](https://github.com/madmaze/pytesseract), [PaddleOCR docs](https://github.com/PaddlePaddle/PaddleOCR), [RapidOCR](https://github.com/RapidAI/RapidOCR).

## Office Documents

DOCX / PPTX / XLSX from Microsoft Office; OpenDocument equivalents handled via separate tools.

### DOCX

| Library | Read/Write | Layout | Size | Speed | License | 2026 | Use when |
|---|---|---|---|---|---|---|---|
| `python-docx` | Read + write | Paragraph / run-level | 8 MB | <100 ms/doc | MIT | Active | Standard DOCX manipulation; document assembly |
| `mammoth` | Read-only | HTML-like conversion | 5 MB | Fast | BSD-2-Clause | Stable | DOCX → HTML preserving semantic markup |
| `docx2python` | Read-only | Tables, text, images | 3 MB | Fast | MIT | Active | Extract tables + text; minimal overhead |

### PPTX

| Library | Read/Write | Layout | BBox | Size | License | 2026 | Use when |
|---|---|---|---|---|---|---|---|
| `python-pptx` | Read + write | Slide / shape structure | Pixel (shape bounds) | 12 MB | MIT | Active | Standard PPTX read/write; slide automation |

### XLSX

| Library | Read/Write | Speed (100K rows) | License | 2026 | Use when |
|---|---|---|---|---|---|
| `openpyxl` | Read + write | ~500 ms | MIT | Active | Standard XLSX; formatting preservation |
| `XlsxWriter` | Write-only | ~300 ms | BSD-2-Clause | Stable | Feature-rich XLSX generation; charts, formulas |
| `python-calamine` | Read-only (Rust FFI) | **3 ms** (10–50× faster) | MIT | New (Feb 2026) | Massive spreadsheets; speed mandatory |
| `polars.read_excel` | Read-only | ~80 ms (calamine backend) | MIT | Active | DataFrame workflows; speed-critical |
| `pandas.read_excel` | Read-only (delegates) | ~400 ms (openpyxl) / ~15 ms (calamine backend) | BSD-3-Clause | Mature | DataFrame workflows; broad ecosystem |

Native libraries (`python-docx`, `python-pptx`, `openpyxl`) are faster and simpler than `unstructured` for single-format pipelines. Reach for `unstructured` when typed `Element` output across mixed document types is the goal.

See also: [python-docx](https://python-docx.readthedocs.io/), [python-pptx](https://python-pptx.readthedocs.io/), [openpyxl](https://openpyxl.readthedocs.io/), [python-calamine](https://pypi.org/project/python-calamine/).

## HTML

Parsing and HTML→Markdown conversion are separate concerns.

### HTML parsing

| Library | Backend | Selectors | Speed | License | 2026 | Use when |
|---|---|---|---|---|---|---|
| `selectolax` | lexbor (C) | CSS / XPath | ~0.3 ms (30× faster than BeautifulSoup) | MIT | Active | High-scale parsing; millions of pages |
| BeautifulSoup + `html5lib` | Python | DOM / CSS / XPath | ~10 ms | MIT | Mature | Rapid prototyping; malformed markup tolerance |
| `lxml.html` | libxml2 (C) | XPath + XSLT | ~2 ms | BSD-3-Clause | Mature | Performance + XPath complexity; XML fallback |
| `parsel` | lxml-backed | CSS + XPath (dual-mode) | ~1 ms | BSD-3-Clause | Stable | Scrapy integration; dual-selector workflows |

### HTML → Markdown

| Library | Fidelity | Speed | License | 2026 | Use when |
|---|---|---|---|---|---|
| `html2text` | Plain-text-ish + Markdown-like | ~10 ms | GPL-3.0 | Stable | Readable text extraction from HTML |
| `markdownify` | Semantic preservation | ~5 ms | MIT | Active | HTML → MD with tag-level configurability |
| `readability-lxml` | Article extraction + Markdown | ~20 ms | Apache 2.0 | Active | Article extraction; ad/nav removal |

Sanitize untrusted HTML with `bleach` (allowlist tags + attributes; default-deny) before persistence, rendering, or conversion. HTTP fetching (retries, robots.txt, rate limiting, JS rendering) lives in [`./py-web-crawling.md`](./py-web-crawling.md).

See also: [selectolax](https://github.com/rushter/selectolax), [bleach](https://bleach.readthedocs.io/), [`html-to-markdown` (modern fork)](https://pypi.org/project/html-to-markdown/).

## Markdown

| Library | Output | Speed | License | 2026 | Use when |
|---|---|---|---|---|---|
| stdlib UTF-8 decode + `normalize_markdown` | Raw text | <1 ms | Built-in | N/A | Plain "load and chunk" workloads |
| `markdown-it-py` | AST tokens | ~1 ms | MIT | Active | Standards-compliant parsing; AST access |
| `mistune` | AST + plugins | ~0.5 ms | BSD-3-Clause | Stable | Speed-critical Markdown parsing |
| `python-markdown` | Extensible plugins | ~5 ms | BSD-3-Clause | Mature | Complex Markdown with custom extensions |
| `marko` | Standards-compliant | ~2 ms | MIT | Active | CommonMark + GFM compliance |
| `unstructured.partition.md` | Typed `Element` output | ~10 ms | Elastic License | Active | Markdown partitioning for RAG; uniform `Element` typing across formats |

## Structured Data

JSON / YAML / XML / CSV / plain text. Many libraries are stdlib + faster third-party alternatives.

### JSON

| Library | Speed | Streaming | Type-aware | License | 2026 | Use when |
|---|---|---|---|---|---|---|
| stdlib `json` | ~100K/s | No | Limited | Built-in | Mature | Compatibility; small payloads |
| `orjson` | ~1M/s (10× stdlib); Rust-backed | No | dataclass / datetime / numpy aware | MIT | Active | Performance-critical; complex types |
| `msgspec` | 5–10× stdlib; Rust-backed | Yes | Schema validation at parse | MIT | Active | Streaming large JSON; validation seam |
| `ujson` | 2–3× stdlib | No | Limited | BSD-3-Clause | Stable | Simple fast parsing; legacy systems |
| `rapidjson` | C++ binding | No | C++-backed | MIT | Lower activity | C++ interop; legacy codebases |

### YAML

| Library | Spec | Round-trip | License | 2026 | Use when |
|---|---|---|---|---|---|
| `PyYAML` | 1.1 | No | MIT | Mature | Standard YAML; **use `yaml.safe_load`** for untrusted input |
| `ruamel.yaml` | 1.2 | Yes (preserves comments/order) | MIT | Active | Edit + re-serialize preserving formatting; YAML 1.2 |

### XML

| Library | Speed | Streaming | Security | License | 2026 | Use when |
|---|---|---|---|---|---|---|
| `lxml` | ~1 ms (libxml2-backed); XPath / XSLT | Yes (`iterparse`) | Trusted input only | BSD-3-Clause | Mature | High-performance; XPath; large files |
| `defusedxml` | ~5 ms | No | Safe against XXE / billion-laughs | Python license | Active | Untrusted input — wrap stdlib `xml.etree` |
| `xml.etree.ElementTree` (stdlib) | ~10 ms | No | **Vulnerable to XXE** without `defusedxml` | Built-in | Mature | No deps; trusted simple XML only |

### CSV

| Library | Speed (1K rows) | Streaming | License | 2026 | Use when |
|---|---|---|---|---|---|
| stdlib `csv` | ~10–50 ms | Yes (native) | Built-in | Mature | Constant-memory row iteration |
| `pandas.read_csv` | ~50–200 ms | Yes (chunking) | BSD-3-Clause | Mature | DataFrame workflows; analysis |
| `polars.read_csv` | ~10–30 ms (parallel) | Yes (streaming) | MIT | Active | High-speed CSV; parallel I/O |

### Plain text + encoding detection

| Library | Purpose | Accuracy | License | 2026 | Use when |
|---|---|---|---|---|---|
| `chardet` (7.0+ rewrite) | Encoding detection | 99.3% (47× faster than 6.x) | MIT (relicensed; review before adoption) | New (v7.0, 2026) | Auto-detect encoding on legacy / scraped text |
| `charset-normalizer` | Universal encoding detector | Confidence + candidates | MIT | Active (3.4.6) | `requests`-default; rich API; actively maintained |

## Wire Up

The file-extraction port is one of the cleanest port-adapter shapes in the project — an async domain capability with many possible implementations.

### Purpose

Provide an asynchronous extraction boundary that hides parser libraries from routes and keeps parser-specific objects behind adapters. The service depends on a domain-named port and receives either plain text or typed `Element`s for downstream chunkers and indexers.

### Responsibilities

- Keep parser dispatch behind the service boundary. Routes collect request data and call a service; they do not import parser libraries or choose parser adapters directly.
- Use `content_type` and/or `options.filename` as adapter hints and validation inputs. Dynamic selection across multiple extractors is allowed when the business requirement needs it, but the selection belongs in a service or routing adapter, not in the FastAPI route handler.
- Wrap synchronous libraries (`pymupdf4llm`, `python-docx`, Tesseract, etc.) with `asyncio.to_thread` so the event loop stays responsive.
- Translate vendor-specific exceptions (`TextractError`, `PdfReadError`, `BadZipFile`) into domain errors (`UnsupportedFileType`, `ExtractionFailed`, `ExtractionOptionsRequired`).
- Return domain types — never an SDK object, never a parser-specific page object.

**Must not own:**
- Chunking → the chunker downstream of extraction.
- Storage → [`./py-aws-s3.md`](./py-aws-s3.md). Callers persist; the extractor reads bytes.
- Cache layer → [`./py-cache.md`](./py-cache.md). Include the adapter name and extraction strategy in any cache key so a parser swap invalidates correctly.
- HTTP fetching → [`./py-web-crawling.md`](./py-web-crawling.md).

### Lifecycle

- Adapter scope: **App Scope**. Use `Singleton` for stateless adapters or adapters that hold reusable SDK clients; use `Resource` when the adapter owns explicit startup/shutdown lifecycle such as model weights, GPU memory, temp directories, or connection pools.
- Created by: the DI container at startup or first provider access, depending on container wiring. Model weights, cloud SDK clients, and connection pools amortize across requests.
- Service scope: **Request Scope**. Services are usually request-owned factories and receive app-scoped extraction ports from the container.
- Request data: tenant ID, filename, language hints, content type, and strategy are passed through DTOs such as `ExtractionOptions`; adapters must not store request-derived mutable state.
- Cleaned up by: container teardown for app-scoped `Resource` providers; request-scoped dependencies clean up through FastAPI `Depends` / `yield` only when they own per-request resources.

### Port & adapter granularity is business-driven

There is no single "right" port shape, but the default is a domain-named async port with per-library adapters. Common shapes:

- **Domain port, multiple per-library adapters.** `DocumentExtractor` port; `PyMuPDFDocumentExtractor`, `TextractDocumentExtractor`, `UnstructuredDocumentExtractor`, etc. Container picks which adapter(s) to wire. A multi-format service composes typed adapter references explicitly in the service layer.
- **Per-format ports.** `PDFExtractor` / `ImageExtractor` / `HTMLExtractor` ports — each with its own adapter — when content types diverge enough that a shared signature would be lossy.
- **Format-agnostic ports.** One `FileExtractor` Protocol with `extract_text(file_bytes, content_type, options) -> str` — the simplest shape when the service treats all uploads uniformly and no domain noun adds useful constraints.
- **Per-workload adapters.** `HighFidelityPDFExtractor` (composes `marker-pdf` + a table-validation pass) when the workload is more specific than any single library handles.

Pick the granularity that matches the business layer. The corpus pre-decides the *libraries to consider* and the *axes to weigh*, not the exact port name — that's a domain call.

### Relationships

```text
[App Scope]
    ├── [DocumentExtractor]            (domain port — name per your domain)
    │       │ implemented by (deploy-time wiring picks adapter)
    │       ▼
    │   [PyMuPDFDocumentExtractor]     (Singleton adapter, stateless)
    │   [TextractDocumentExtractor]    (Singleton/Resource adapter, holds SDK client)
    │   [DoclingDocumentExtractor]     (Resource adapter, holds model weights)
    │   [PythonDocxDocumentExtractor]  (Singleton adapter, stateless)
    │   ...
    │
[Request Scope]
    └── [DocumentIngestionService]     (request-owned Factory)
            │ depends on
            ▼
       [DocumentExtractor]             (port, injected — one adapter in single-format
                                        services; multiple adapters selected by
                                        service/routing adapter only when the
                                        business requirement needs it)
```

### Constraints

- Must not be called directly from a route. Routes obtain a service from DI; the service holds the port. See [`./py-backend-architecture.md#services-and-fastapi`](./py-backend-architecture.md#services-and-fastapi).
- Must not leak parser objects (PyMuPDF `Page`, `unstructured.documents.elements.Element` subclasses) across the port boundary — map to the domain `Element` type defined alongside the port.
- Must not own request-scoped state. An adapter that needs per-request context (tenant ID, language hint) takes it via `ExtractionOptions`.

### Port shape

```python
from dataclasses import dataclass
from typing import Protocol


@dataclass(frozen=True, slots=True, kw_only=True)
class ExtractionOptions:
    languages: tuple[str, ...] | None = None    # required for OCR adapters at call-time
    filename: str | None = None                 # disambiguation hint when content_type is None
    strategy: str | None = None                 # adapter-specific: "fast" | "hi_res" | "ocr_only" | ...


ElementMetadataScalar = str | int | float | bool | None
ElementMetadataValue = ElementMetadataScalar | tuple[ElementMetadataScalar, ...]


class Element(Protocol):
    type: str  # "paragraph" | "heading" | "table" | "image" | ...
    text: str
    metadata: dict[str, ElementMetadataValue]


class FileExtractor(Protocol):
    async def extract_text(
        self,
        *,
        file_bytes: bytes,
        content_type: str | None = None,
        options: ExtractionOptions | None = None,
    ) -> str: ...

    async def extract_elements(
        self,
        *,
        file_bytes: bytes,
        content_type: str | None = None,
        options: ExtractionOptions | None = None,
    ) -> list[Element]: ...
```

`ExtractionOptions` is an internal trust-boundary DTO (frozen `dataclass`, not a `BaseModel`). `content_type` is a hint passed to the adapter, not a route-level dispatch key — each per-library adapter validates the content type it was wired for and raises `UnsupportedFileType` outside its domain. OCR adapters validate `options.languages` and raise `ExtractionOptionsRequired` (a `CustomError` — see [`./py-errors.md`](./py-errors.md)) when missing. See [Pitfalls](#pitfalls).

Keep `metadata` simple and typed: scalar values or compact tuples such as normalized bbox coordinates. Promote large nested structures to explicit domain fields instead of hiding them in `metadata`.

Container wiring (which adapter binds to the port, how cloud credentials are injected, how to override in tests) lives in [`./dependency-injector.md`](./dependency-injector.md).

## Async Wrapping

Most extraction libraries above are synchronous. `asyncio.to_thread` is the canonical wrapper: stdlib, no extra dependency, uses the default executor (~32 threads at default `max_workers`). Concurrency = thread-pool size, not CPU count — IO-bound calls (cloud OCR) scale well; CPU-heavy local OCR saturates the pool quickly.

```python
import asyncio


async def extract_text(
    self,
    *,
    file_bytes: bytes,
    content_type: str | None = None,
    options: ExtractionOptions | None = None,
) -> str:
    return await asyncio.to_thread(_sync_extract, file_bytes, content_type, options)
```

`anyio.to_thread.run_sync` is the multi-backend alternative (asyncio + trio + curio). Reach for it when the codebase commits to AnyIO or when trio compatibility matters.

For heavy local OCR (PaddleOCR, marker-pdf, olmOCR, vision-language models), move work to a queue (RQ / Celery / SQS-consumer worker) rather than a thread — thread pools share the GIL on Python-side glue and don't help with CPU-bound model inference.

See also: [`asyncio.to_thread` docs](https://docs.python.org/3/library/asyncio-task.html#asyncio.to_thread), [AnyIO threads](https://anyio.readthedocs.io/en/stable/threads.html).

## Testing

Mock the port boundary, not the parser internals. The double is `MockFileExtractor` — it implements the `FileExtractor` Protocol with deterministic return values.

```python
class MockFileExtractor:
    async def extract_text(
        self,
        *,
        file_bytes: bytes,
        content_type: str | None = None,
        options: ExtractionOptions | None = None,
    ) -> str:
        return "hello world"

    async def extract_elements(
        self,
        *,
        file_bytes: bytes,
        content_type: str | None = None,
        options: ExtractionOptions | None = None,
    ) -> list[Element]:
        return [...]
```

Unit tests do **not**:
- parse real PDFs
- hit Textract / Azure / Google / Mistral / LlamaParse
- spawn Tesseract / LibreOffice / Pandoc subprocesses
- download model weights

Integration tests for real adapters use small fixture files under `tests/fixtures/` behind `pytest -m integration` — the default `pytest` invocation stays fast and offline. Fixture files < 50 KB each; version-controlled.

See [`./py-tests.md`](./py-tests.md) for the API-test boundary pattern.

## Production Checklist

| Item | Default |
|---|---|
| Upload size cap | Enforced at proxy + FastAPI + extractor (`max_upload_bytes`). |
| MIME validation | Validate content type **and** sniff magic bytes; never trust the client. |
| Durable storage before extraction | Persist raw upload to S3 before long extraction; survives restarts. |
| OCR languages explicit | `options.languages` always supplied; never rely on Tesseract's English default. |
| Cloud retries / backoff | Exponential backoff on rate-limit / throttling exceptions; per-tenant rate limiting. |
| Async wrapping | Every sync extraction call wrapped in `asyncio.to_thread`. |
| Parsed element cache | Cache by `(sha256(bytes), adapter_name, strategy)`; see [`./py-cache.md`](./py-cache.md). |
| HTML sanitization | `bleach.clean` before persistence or Markdown conversion of untrusted input. |
| Temp file cleanup | `tempfile.NamedTemporaryFile` / `TemporaryDirectory` context managers. |
| Extraction metadata | Record adapter name, adapter version, page count, text length, duration. |
| Empty-text outcome | Empty extracted text is a valid result for scanned PDFs, not a failure. |
| License compliance | Audit AGPL libraries (`pymupdf4llm`, `PyMuPDF`); non-commercial licenses (Nougat CC-BY-NC, MiniCPM-o CC-BY-NC) blocked from commercial deployment. |
| PII / sensitive content | Layer a detection pass (regex or ML tag) on extracted text before persistence in regulated industries (HIPAA, GDPR, PCI-DSS). |

## Pitfalls

- **Sync OCR in an async route.** Tesseract, `pymupdf4llm`, `python-docx` are all blocking. Wrap with `asyncio.to_thread` at the adapter, never in the route.
- **OCR without `ExtractionOptions.languages`.** Tesseract silently returns gibberish for non-English scripts — the failure is silent gibberish, not an exception. OCR adapters validate `options.languages` and raise `ExtractionOptionsRequired` (`CustomError`) when missing. Never default to English silently.
- **`yaml.load` on untrusted input.** Executes arbitrary Python via tag deserialization. Use `yaml.safe_load`. Treat `yaml.load` without an explicit `SafeLoader=` as a security bug.
- **`xml.etree` on untrusted input.** Vulnerable to billion-laughs / XXE / DTD retrieval. Use `defusedxml` for any XML from the network or user uploads.
- **`EasyOCR` on production text.** Deprecated 2026 — systematic failure modes on financial symbols, accounting marks, mixed numeric content. Use Tesseract, PaddleOCR, or RapidOCR.
- **Treating cloud OCR bboxes as pixels.** Textract / Azure DI / Google Document AI return `[0, 1]` ratios with top-left origin. Multiply by page dimensions before drawing.
- **`unstructured[all-docs]` on a small container.** Pulls `detectron2` + `torch` + every format-specific dep (~3 GB). On Lambda / small ECS tasks this exceeds the image-size budget. Pin to the format-specific extra (`[pdf]`, `[docx]`).
- **Per-call cloud client.** Constructing a new `boto3.client("textract")` per request defeats connection pooling and re-loads credentials. Bind as a singleton in the container.
- **Cache key missing strategy.** Caching parsed elements by `sha256(bytes)` alone means a swap from `fast` to `hi_res` returns stale results forever. Include adapter name + strategy in the key.
- **AGPL contamination.** `pymupdf4llm` and `PyMuPDF` are AGPL-3.0. A SaaS deployment that ships `pymupdf4llm` as part of a closed-source product without an Artifex commercial license violates AGPL. Audit before shipping; switch to `pypdfium2` + `pdfplumber` for permissive alternatives.
- **GPU-required model in CPU-only deployment.** `olmOCR`, Qwen2.5-VL, MiniCPM-o won't run without a GPU. Filter early in the model-size / GPU-dependency decision axes.
- **PII leaking into vector stores or logs.** Extracted text from regulated content (HIPAA, GDPR, PCI-DSS) needs a PII / PHI detection pass before persistence; no extractor strips it for you.

## Domain Specialization

`FileExtractor` is the primitive port name — file-level and intentionally generic. Prefer a domain port when the business layer has a meaningful noun for the extraction capability:

```python
from typing import Protocol


class DocumentExtractor(Protocol):
    async def extract_elements(
        self,
        *,
        file_bytes: bytes,
        content_type: str | None = None,
        options: ExtractionOptions | None = None,
    ) -> list[Element]: ...


class ImageTextExtractor(Protocol):
    async def extract_text(
        self,
        *,
        file_bytes: bytes,
        content_type: str | None = None,
        options: ExtractionOptions | None = None,
    ) -> str: ...
```

Adapters follow the same specialization: `PyMuPDFDocumentExtractor`, `TextractImageTextExtractor`, etc. The port + adapter shape from [Wire Up](#wire-up) doesn't change; only the names and capabilities do.

Do not create empty subclasses just to rename `FileExtractor`. Either define the domain protocol directly, as above, or keep the primitive name when the domain doesn't add meaningful constraints. Naming is a cost; pay it only when the domain noun carries real information.
