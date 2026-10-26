# Module 1 — PDF Text Extraction

## Purpose

The PDF extraction module converts a PDF into structured, word-level data that later modules can analyze and protect. It combines native PDF text extraction with OCR for text that exists only as pixels (for example, scanned pages). It also records page geometry, word bounding boxes, text source, and available formatting/layout metadata.

**This module does not classify sensitivity and does not encrypt or decrypt the PDF.**

## Inputs and outputs

### Inputs

- **Input PDF:** the document to analyze.
- **DPI:** resolution used to render pages for OCR and visual quality checks; higher DPI may improve OCR but takes more time and memory.
- **OCR language:** Tesseract language code, defaulting to English (`eng`).
- **Page segmentation mode (`psm`):** Tesseract's assumption about the page's text layout.
- **Minimum OCR confidence:** OCR words below this threshold can be excluded.
- **Background sampling / page-render options:** controls optional background estimates and saved page images.

### Outputs

- `tokens.csv` — one row per extracted word, with page number, word index, bounding box, text source, and available layout/style fields.
- `pdf_layout.json` — structured page-level layout information, dimensions, rotation, image placements, and diagnostics.
- `page_renders/page_0001.png`, etc. — optional rendered page images for visual inspection.

## Libraries and why they are used

| Library/module | Role in this module |
|---|---|
| `PyMuPDF` (`fitz`) | Opens PDFs, extracts native words and text spans, reads page dimensions/rotation/drawings/images, and renders pages. Native extraction retains PDF-provided text and layout information when available. |
| `pytesseract` | Python interface to the Tesseract OCR executable. It returns recognized text, word boxes, and OCR confidence for scanned/image-based text. |
| `Tesseract OCR` executable | Performs the actual optical character recognition. Installing the Python wrapper alone is not sufficient; the executable must also be installed and accessible. |
| `Pillow` (`PIL.Image`) | Represents rendered pages as images, supports pixel sampling and image conversion, and allows background estimates around text. |
| `csv` | Writes word-level extraction rows to CSV. |
| `json` | Serializes structured page/layout metadata to JSON. |
| `hashlib` | Calculates a SHA-256 digest of the source file for integrity/identity checks. |
| `pathlib.Path` | Handles input/output paths in a platform-friendly way. |
| `re` | Normalizes whitespace and text for matching/deduplication. |
| `math` | Checks numeric values and supports safe numeric calculations. |
| `collections.Counter` | Counts categories/diagnostic values during extraction. |
| `statistics` | Supports summary statistics used by extraction diagnostics. |
| `logging` | Provides structured logging through the module's logger. |
| `shutil` | Checks whether the Tesseract executable is available on `PATH`. |
| `datetime` / `timezone` | Produces timestamp metadata where required. |
| `typing` / `__future__` | Adds type annotations and forward-compatible annotation behavior. |

## Function reference

### `sha256_file(path: Path) -> str`

- **What it does:** Reads a file in chunks and calculates its SHA-256 digest. Chunked reading avoids loading the entire PDF into memory.
- **Inputs:**
  - `path`: path to the file to hash.
- **Outputs:** A hexadecimal SHA-256 string.
- **Why needed:** Provides a stable fingerprint for identifying the input file and supporting integrity/diagnostic metadata. A hash does not encrypt the file.

### `safe_float(value: Any) -> Optional[float]`

- **What it does:** Attempts to convert a value to a floating-point number and rejects non-finite values such as NaN or infinity.
- **Inputs:** `value`, the candidate value.
- **Outputs:** A finite `float`, or `None` if conversion fails or the result is not finite.
- **Why needed:** PDF metadata can be missing or malformed; this helper keeps invalid numeric values from propagating into output metadata.

### `rect_list(rect: Any) -> list[float]`

- **What it does:** Converts a rectangle-like object into a list of rounded coordinates.
- **Inputs:** `rect`, an iterable of four rectangle coordinates.
- **Outputs:** Four floating-point coordinates rounded to three decimal places.
- **Why needed:** Creates a consistent, JSON/CSV-friendly representation of bounding boxes.

### `normalize_text(text: str) -> str`

- **What it does:** Collapses repeated whitespace, trims the ends, and case-folds text.
- **Inputs:** `text`, the text to normalize.
- **Outputs:** A normalized string.
- **Why needed:** Supports comparison of native-extracted words with OCR words even when casing or spacing differs.

### `color_to_rgb(value: Optional[int]) -> Optional[list[int]]`

- **What it does:** Converts a packed integer color into red, green, and blue channel values.
- **Inputs:** `value`, the PDF color integer, or `None`.
- **Outputs:** `[R, G, B]` values from 0–255, or `None`.
- **Why needed:** Makes native text colors easier to store and inspect.

### `rgb_to_hex(rgb: Optional[list[int]]) -> Optional[str]`

- **What it does:** Formats RGB channels as a hexadecimal color string.
- **Inputs:** `rgb`, a three-element RGB list, or `None`.
- **Outputs:** A string such as `#1A2B3C`, or `None`.
- **Why needed:** Provides a familiar representation for downstream inspection and possible text restoration.

### `intersection_over_union(a, b) -> float`

- **What it does:** Computes the intersection-over-union (IoU) ratio between two rectangles.
- **Inputs:** `a`, `b`, each represented by `[x0, y0, x1, y1]`.
- **Outputs:** A value between 0 and 1; 0 means no overlap, and 1 means identical rectangles.
- **Why needed:** Helps determine whether an OCR word and a native word occupy approximately the same location.

### `approximately_same_box(a, b, tolerance=5.0) -> bool`

- **What it does:** Checks whether corresponding coordinates in two boxes differ by no more than a tolerance.
- **Inputs:** Two four-coordinate boxes and an optional coordinate tolerance.
- **Outputs:** `True` if every coordinate is within tolerance; otherwise `False`.
- **Why needed:** Provides a simple geometric comparison for layout matching.

### `check_tesseract() -> str`

- **What it does:** Checks that the Tesseract executable is installed and can be launched.
- **Inputs:** None. It also respects an explicitly configured `pytesseract` executable path.
- **Outputs:** The detected Tesseract version as a string.
- **Errors:** Raises `RuntimeError` if Tesseract cannot be found or started.
- **Why needed:** Fails early with a useful error rather than allowing OCR to fail later without a clear cause.

### `extract_native_words(page, page_number) -> list[dict]`

- **What it does:** Extracts words already represented as text in the PDF and associates each word with the best-overlapping text span to obtain available formatting metadata.
- **Inputs:**
  - `page`: a PyMuPDF page object.
  - `page_number`: the page identifier written to each word record.
- **Outputs:** A list of word dictionaries containing text, word/page indexes, bounding boxes, block/line indexes, source=`native`, font family/size/flags, bold/italic indicators, color, line direction, and span metadata when available.
- **Why needed:** Native extraction is generally more accurate than OCR for PDFs with a real text layer and preserves PDF metadata that OCR cannot recover.
- **Limitations:** Formatting information can be unavailable for a word if no matching span is found. Font flags depend on the source PDF's metadata.

### `render_page(page, dpi) -> PIL.Image`

- **What it does:** Renders a PDF page to an RGB image.
- **Inputs:** A PyMuPDF page and the desired DPI.
- **Outputs:** A Pillow RGB image.
- **Why needed:** OCR requires pixels, and rendered pages can also be inspected for visual quality assurance.

### `extract_ocr_words(image, page_number, dpi, language, psm, min_confidence) -> list[dict]`

- **What it does:** Runs Tesseract over a rendered page and converts recognized words into the module's word-record format.
- **Inputs:** Rendered page image, page number, DPI, OCR language, Tesseract page-segmentation mode, and minimum confidence threshold.
- **Outputs:** A list of OCR word records with recognized text, bounding boxes converted from pixels to PDF-point coordinates, confidence, and source=`ocr`.
- **Why needed:** Recovers text that is part of a scan or image and therefore is not available through native PDF text extraction.
- **Limitations:** OCR does not recover the original font or exact PDF text color, and confidence is an estimate rather than a guarantee of correctness.

### `merge_native_and_ocr(native_words, ocr_words, iou_threshold=0.25) -> list[dict]`

- **What it does:** Suppresses OCR records that appear to duplicate a native word based on normalized text and overlapping geometry.
- **Inputs:** Native word records, OCR word records, and the IoU threshold used to judge overlap.
- **Outputs:** A combined word list, preferring native records where both sources appear to identify the same word.
- **Why needed:** A PDF may contain both a text layer and a rendered image of the same text; retaining both would create duplicate tokens.
- **Limitations:** This is a heuristic. Similar words in different locations should be retained, but imperfect OCR or coordinates can still cause missed or false matches.

### `inspect_native_backgrounds(page) -> list/dict of drawing metadata`

- **What it does:** Inspects PDF-native vector drawing fills that may provide background-color information.
- **Inputs:** A PyMuPDF page.
- **Outputs:** Structured information about inspectable vector fills, according to the implementation.
- **Why needed:** Background information can help preserve the appearance of a document when text is masked or restored.
- **Limitations:** Vector fill inspection is not a complete description of every possible page background.

### `sample_background(image, bbox, dpi) -> background estimate`

- **What it does:** Samples rendered pixels around a word's bounding box to estimate the local background.
- **Inputs:** Rendered page image, word bounding box, and DPI.
- **Outputs:** A background-color estimate in the representation used by the implementation.
- **Why needed:** A mask that matches the local background can look less intrusive than a fixed white rectangle.
- **Limitations:** This is a pixel-based estimate and may be inaccurate near gradients, images, borders, or other text.

### `inspect_image_placements(page) -> image-placement metadata`

- **What it does:** Inspects image placements on a PDF page.
- **Inputs:** A PyMuPDF page.
- **Outputs:** Image-placement details in the structure defined by the implementation.
- **Why needed:** Image placement and page geometry are useful for diagnostics and for understanding scanned or image-heavy pages.

### `word_to_csv_row(word) -> dict`

- **What it does:** Converts an internal word record into a flat row suitable for CSV output.
- **Inputs:** One word dictionary.
- **Outputs:** A dictionary of CSV column values.
- **Why needed:** Keeps the serialized token schema consistent despite internal records containing nested or optional metadata.

### `extract_pdf(input_pdf, output_dir, dpi=300, language='eng', psm=3, min_confidence=35.0, background_sampling=True, save_page_renders=True)`

- **What it does:** Orchestrates the extraction pipeline: opens the PDF, checks OCR availability, processes pages, combines native and OCR words, gathers layout/background/image metadata, and writes the output artifacts.
- **Inputs:**
  - `input_pdf`: source PDF path.
  - `output_dir`: directory for CSV, JSON, and optional page images.
  - `dpi`: rendering resolution.
  - `language`: Tesseract language.
  - `psm`: Tesseract page segmentation mode.
  - `min_confidence`: minimum OCR confidence.
  - `background_sampling`: whether to estimate local backgrounds.
  - `save_page_renders`: whether to save rendered page images.
- **Outputs:** Writes `tokens.csv`, `pdf_layout.json`, and optional page-render PNGs. The function's direct return value depends on the implementation; the primary result is the output files.
- **Why needed:** Provides one entry point for running all extraction steps with a consistent configuration.

### `validate_outputs(csv_path, json_path) -> validation result`

- **What it does:** Performs a structural smoke test on the generated CSV and JSON outputs.
- **Inputs:** Paths to the token CSV and layout JSON.
- **Outputs:** A validation result or exception/report according to the implementation.
- **Why needed:** Catches missing files or malformed output structure before downstream NLP and encryption consume the artifacts.
- **Limitations:** The code's own docstring notes that this does not certify OCR accuracy.

## How this module connects to the rest of the project

1. PDF extraction creates `tokens.csv` and `pdf_layout.json`.
2. The NLP module adds linguistic and entity features to the token rows and writes `tokens_nlp.csv`.
3. Sensitivity classification adds a sensitivity label for each token.
4. Cryptography uses token labels and layout coordinates to selectively protect content while retaining the page structure.
