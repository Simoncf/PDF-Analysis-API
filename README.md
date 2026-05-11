# PDF Analysis API

ASP.NET Core 8 Web API that extracts structured data from PDF files using a locally running
LLaMA model via Ollama.

## Architecture

```
Client  ──POST /api/documents/extract──►  DocumentsController
                                               │
                                    ┌──────────┴──────────┐
                                    ▼                     ▼
                          PdfTextExtractor          OllamaService
                          (UglyToad.PdfPig)         (HTTP → Ollama)
                                    │                     │
                               Raw text         Structured JSON
                                    └──────────┬──────────┘
                                               ▼
                                        ExtractionResult
```

## Prerequisites

### 1. Ollama
```bash
# Install (Linux / macOS)
curl -fsSL https://ollama.com/install.sh | sh

# Pull a model (llama3.2 recommended — fast & accurate for extraction)
ollama pull llama3.2

# Confirm it's running
curl http://localhost:11434/api/tags
```

### 2. .NET 8 SDK
```bash
dotnet --version   # should be 8.x
```

## Running

```bash
cd PdfAnalysisApi
dotnet run
```

Swagger UI → http://localhost:5000/swagger

---

## Endpoints

### `POST /api/documents/extract`

Extracts structured fields from a PDF.

**Form fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `file` | binary | ✅ | The PDF file |
| `fieldsJson` | string (JSON) | ✅ | Array of `{name, description, type}` objects |
| `extraInstructions` | string | ❌ | Additional prompt instructions |

**`fieldsJson` schema:**
```json
[
  { "name": "invoice_number", "description": "Invoice reference number", "type": "string" },
  { "name": "vendor_name",    "description": "Name of the vendor",       "type": "string" },
  { "name": "total_amount",   "description": "Total amount due",         "type": "number" },
  { "name": "issue_date",     "description": "Invoice date, ISO-8601",   "type": "date"   },
  { "name": "line_items",     "description": "List of purchased items",  "type": "array"  }
]
```

**Supported types:** `string`, `number`, `boolean`, `date`, `array`

---

### Example — Invoice extraction

```bash
curl -X POST http://localhost:5000/api/documents/extract \
  -F "file=@invoice.pdf" \
  -F 'fieldsJson=[
        {"name":"invoice_number","description":"Invoice ID","type":"string"},
        {"name":"vendor","description":"Vendor name","type":"string"},
        {"name":"total","description":"Total amount due","type":"number"},
        {"name":"date","description":"Invoice date ISO-8601","type":"date"}
      ]' \
  -F 'extraInstructions=All monetary values should be plain numbers without currency symbols.'
```

**Response:**
```json
{
  "fileName": "invoice.pdf",
  "pageCount": 2,
  "charactersExtracted": 3847,
  "data": {
    "invoice_number": "INV-2024-00142",
    "vendor": "Acme AB",
    "total": 12500.00,
    "date": "2024-11-15"
  },
  "modelUsed": "llama3.2",
  "elapsedMs": 1823
}
```

---

### Example — Contract extraction

```bash
curl -X POST http://localhost:5000/api/documents/extract \
  -F "file=@contract.pdf" \
  -F 'fieldsJson=[
        {"name":"parties","description":"Names of all signing parties","type":"array"},
        {"name":"effective_date","description":"Date the contract takes effect","type":"date"},
        {"name":"termination_clause","description":"Summary of termination conditions","type":"string"},
        {"name":"governing_law","description":"Jurisdiction / governing law","type":"string"}
      ]'
```

---

### `GET /api/documents/models`

Lists all models currently pulled in Ollama. Useful to confirm the model in
`appsettings.json` is available before sending extraction requests.

```bash
curl http://localhost:5000/api/documents/models
```

---

## Switching models

Edit `appsettings.json` (or override with env vars) and restart:

```json
"Ollama": {
  "Model": "mistral"
}
```

Or with environment variable:
```bash
Ollama__Model=mistral dotnet run
```

## Tips for better extractions

- **Temperature 0.0** — always keep this for extraction; higher values introduce hallucinations.
- **Large PDFs** — text is truncated at ~60 k characters. For very long documents consider
  splitting pages and calling the endpoint per section.
- **Scanned PDFs** — PdfPig only handles text-layer PDFs. Run OCR (e.g. Tesseract) first and
  feed the text-layer PDF.
- **Model choice** — `mistral` often outperforms `llama3.2` on structured forms;
  `phi3:mini` is fastest for simple key-value extraction.
