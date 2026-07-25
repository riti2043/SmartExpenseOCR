# SmartExpenseOCR

An OCR engine for receipt processing. It has two separate paths: raw text extraction through Tesseract, and structured data extraction through an LLM that turns a receipt image directly into parsed JSON.

## Demo output


## How it works

Tesseract handles pure text extraction. It reads the image and gives back whatever text it detects, with no understanding of what that text means. That's useful when you just need the words on the receipt.

The LLM path does more. It takes the image, sends it to a vision-capable model, and asks for output matching a schema you define. The model figures out which text is the merchant name, which lines are items, what the total is, and returns it as JSON. No manual parsing rules, no regex for prices or dates. The schema is what tells the model what to look for.

Both paths are independent. You can use one without the other.

## Project structure

- `src/smart_expense_ocr/` - the LLM-based module. Providers, processors, schema handling, CLI, and a FastAPI service.
- `src/tesseract_ocr/` - the Tesseract module. CLI, FastAPI app, and image preprocessing utilities.

## Providers

The LLM module talks to any OpenAI-compatible API, which covers more than just OpenAI itself.

| Provider | Notes |
|---|---|
| OpenAI | `gpt-4o` and similar vision models |
| Gemini | via Google's OpenAI-compatible endpoint |
| Groq | fast inference, smaller models |

Switching providers is a matter of changing the base URL and model name. The processing logic doesn't change.

## Schema-driven extraction

You pass a schema, the processor returns data shaped to it. Two levels of strictness are supported:

- A simplified dict format (`"field": "type"`), good for quick use
- A full JSON Schema object, which maps to OpenAI's structured outputs and enforces the shape more strictly

The default schema pulls merchant name, address, transaction date and time, total amount, and a list of line items with quantity and price. You're not locked into that shape. If you only care about totals, the schema can be that small.

Response formatting is also configurable: `json_object` for a standard object, `json_schema` for strict structured output, or `text` if you want the raw model response instead.

## Tesseract module

This is the simpler, deterministic path. No API key, no external calls, everything runs locally through the Tesseract binary. Accuracy depends heavily on image quality: lighting, skew, and how clearly the receipt edges are captured all matter more here than in the LLM path, since there's no model filling in gaps or correcting for noise.

It's the right choice when you just need the text and don't need it parsed, or when you'd rather not depend on an LLM provider at all.

## API surface

Both modules expose a FastAPI service with the same basic shape:

- `GET /health`
- `POST /ocr/` - accepts an image file, returns text (Tesseract) or structured JSON (LLM path, with an optional custom schema)

The LLM service additionally exposes interactive docs at `/docs`.

## Tech stack

| Category | Technology |
|---|---|
| OCR (raw) | Tesseract |
| OCR (structured) | OpenAI-compatible LLMs (OpenAI, Gemini, Groq) |
| API | FastAPI |
| Packaging | uv |
| Linting | ruff |
| Testing | pytest |

## Basic usage

```bash
pip install smart-expense-ocr
export OPENAI_API_KEY="your_key"
smart-expense-ocr images/receipt.jpg
```

```python
from smart_expense_ocr.processors import ReceiptProcessor
from smart_expense_ocr.providers import OpenAIProvider

provider = OpenAIProvider(api_key="your_key", base_url="your_base_url")
processor = ReceiptProcessor(provider)
result = processor.process_receipt("receipt.jpg", json_schema, "gpt-4.1")
```

## Contributing

Fork the repo, branch off, keep the code style consistent with what's there. Run `uv sync --all-extras --dev` to get a working environment, `pytest` for tests, `ruff check` and `ruff format` before you commit. Open an issue first for anything sizable.

## License

MIT.