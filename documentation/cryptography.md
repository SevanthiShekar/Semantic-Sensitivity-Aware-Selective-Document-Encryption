# Module 4 — Selective PDF Cryptography

## Purpose

The cryptography module selectively protects content in a PDF using token sensitivity labels and page-layout metadata. The implementation uses hybrid cryptography:

- **AES-GCM** : encrypts content bytes and authenticates them.

- **RSA-OAEP** : protects the random session key so only the holder of the matching private key can recover it.

- **HKDF-SHA-256** : derives independent encryption keys for the sensitivity tiers from the session key.

- **Authenticated additional data (AAD)** : binds ciphertext to metadata such as page, word index, content type, and sensitivity label.

The module also renders pages to images, groups overlapping word rectangles into image patches, masks protected regions in the output PDF, stores encrypted payload data, and reconstructs the protected content during decryption.

## Inputs and outputs

### Encryption inputs

- Original input PDF.

- `pdf_layout.json` produced by PDF extraction.

- `tokens_with_sensitivity.csv` produced by sensitivity classification.

- RSA public key in PEM format.

- Output directory.

- Rendering DPI (default `300`).

### Encryption outputs

- `encrypted_document.pdf` — PDF with protected content masked/replaced and encrypted payload attached as implemented.

- `encrypted_session_key.json` — session-key and payload metadata required for decryption; the session key itself is RSA-wrapped rather than stored as plaintext.

- The exact internal payload structure is versioned by the code.

### Decryption inputs

- Encrypted PDF.

- Encrypted session-key JSON.

- Matching RSA private key.

- Destination path for the reconstructed PDF.

### Decryption output

- `decrypted_document.pdf` — Reconstructed/decrypted PDF at the specified output path.

## Libraries and why they are used

| Library/module | Role |
| --- | --- |
| `cryptography` | Provides the cryptographic primitives used for RSA key operations, AES-GCM authenticated encryption, HKDF key derivation, hashes, serialization, and cryptographic exceptions. |
| `cryptography.hazmat.primitives.asymmetric.rsa` | Generates and validates RSA key objects. |
| `cryptography.hazmat.primitives.asymmetric.padding` | Configures OAEP padding for RSA session-key wrapping/unwrapping. |
| `cryptography.hazmat.primitives.ciphers.aead.AESGCM` | Encrypts/decrypts bytes with AES-GCM and authenticates ciphertext and AAD. |
| `cryptography.hazmat.primitives.kdf.hkdf.HKDF` | Derives distinct tier-specific AES keys from the session key and salt. |
| `cryptography.hazmat.primitives.hashes.SHA256` | Supplies SHA-256 to OAEP and HKDF and for file digests. |
| `cryptography.hazmat.primitives.serialization` | Loads PEM keys and serializes key objects to PEM files. |
| `PyMuPDF` (`fitz`) | Opens and edits PDFs, reads page geometry, renders pages, inserts replacement text, and stores/retrieves the embedded payload. |
| `Pillow` (`PIL.Image`) | Crops and processes rendered page images and background pixels. |
| `pandas` | Reads the token CSV and builds a per-token sensitivity map. |
| `json` | Reads layout/payload JSON and serializes structured metadata. |
| `base64` | Encodes binary nonces/ciphertexts for JSON storage and decodes them before decryption. Base64 is encoding, not encryption. |
| `hashlib` | Computes SHA-256 file digests. |
| `io.BytesIO` | Temporarily stores PNG image bytes in memory before encryption. |
| `os` | Generates random nonce bytes and handles file/path operations. |
| `pathlib.Path` | Handles key and file paths. |
| `argparse` / `sys` | Support command-line execution and argument handling where used by the module. |

## Key-generation function

### `generate_key_pair(output_dir="keys") -> None`

- **What it does:** Generates a 2048-bit RSA private key and its matching public key, then saves both as PEM files.

- **Inputs:** `output_dir`, the directory where the key files are stored.

- **Outputs:** Writes `private_key.pem` and `public_key.pem`; the function also prints the paths.

- **Why needed:** RSA keys allow the system to protect a randomly generated session key. The public key can be used for encryption; the private key is required for recovery.

- **Implementation note:** In this notebook version, the private key is written without a password (`NoEncryption`). The key-generation behavior is documented here to describe how the academic prototype handles its key files.

## Helper functions in the cryptography support cell

### `b64e(data: bytes) -> str`

- **What it does:** Base64-encodes bytes.

- **Inputs:** Raw bytes.

- **Outputs:** ASCII Base64 string.

- **Why needed:** Allows binary ciphertext and nonce data to be stored in JSON.

### `b64d(value: str) -> bytes`

- **What it does:** Decodes an ASCII Base64 string.

- **Inputs:** Base64 string.

- **Outputs:** Original bytes.

- **Why needed:** Converts serialized nonce/ciphertext values back into bytes for cryptographic operations.

### `derive_word_key(master_key: bytes, record_id: str) -> bytes`

- **What it does:** Derives a separate AES-256 key for an individual encrypted word using the master key and record identifier.

- **Inputs:** `master_key` bytes and a stable `record_id`.

- **Outputs:** Derived key bytes.

- **Why needed:** Provides key separation between word records in this helper implementation.

- **Note:** The main pipeline uses tier-derived keys in its primary word/patch encryption functions; this helper belongs to the support code and should be understood in the context where it is called.

### `make_aad(record_id: str, page_number: int, word_index: int, sensitivity: str) -> bytes`

- **What it does:** Constructs authenticated metadata for a token record.

- **Inputs:** Record identifier, page number, word index, and sensitivity label.

- **Outputs:** Serialized AAD bytes.

- **Why needed:** If authenticated metadata is altered, AES-GCM authentication should fail during decryption.

### `load_public_key(path) -> RSA public key`

- **What it does:** Reads a PEM file and deserializes the public key, validating that it is an RSA public key.

- **Inputs:** Key file path.

- **Outputs:** RSA public-key object.

- **Why needed:** Loads the key used to wrap the session key.

### `load_private_key(path) -> RSA private key`

- **What it does:** Reads a PEM file and deserializes the private key, validating that it is an RSA private key.

- **Inputs:** Key file path.

- **Outputs:** RSA private-key object.

- **Why needed:** Loads the key used to unwrap the session key during decryption.

### `load_sidecars(layout_path, tokens_path) -> tuple`

- **What it does:** Loads the extraction layout JSON and token/sensitivity CSV sidecar files.

- **Inputs:** Paths to the layout JSON and token CSV.

- **Outputs:** Loaded layout and token/sensitivity data in the return structure used by the implementation.

- **Why needed:** The PDF's visible content must be linked to the correct page coordinates and sensitivity labels.

### `choose_font_alias(word) -> str`

- **What it does:** Chooses a built-in PDF font approximation based on a word's available font metadata.

- **Inputs:** Word metadata dictionary.

- **Outputs:** Font alias used for text insertion.

- **Why needed:** The decryption/restoration stage may need to reinsert text without embedding the original font file.

- **Limitation:** A built-in font is an approximation and may not exactly match the original typography.

### `normalized_rgb(rgb) -> normalized color`

- **What it does:** Normalizes an RGB color value to the form expected by the PDF insertion routines.

- **Inputs:** RGB color data.

- **Outputs:** Normalized RGB representation used by the implementation.

- **Why needed:** PyMuPDF's drawing/text APIs expect colors in a specific numeric form.

### `word_insert_point(word) -> tuple/point`

- **What it does:** Estimates a text baseline/insertion point from a word's bounding box.

- **Inputs:** Word metadata, including its bounding box.

- **Outputs:** Insertion-point coordinates.

- **Why needed:** A span's origin is not necessarily the correct baseline for every individual word, so this helper estimates a position for restoration.

## Main cryptography-module helper functions

### `b64e(data: bytes) -> str` and `b64d(value: str) -> bytes`

- **What they do:** Convert between raw bytes and Base64 text.

- **Inputs/outputs:** `b64e` takes bytes and returns a string; `b64d` takes a string and returns bytes.

- **Why needed:** JSON is text-based, so binary nonce and ciphertext values need a safe text representation.

### `canonical_json(data) -> bytes`

- **What it does:** Serializes a dictionary to deterministic JSON using sorted keys, compact separators, and UTF-8.

- **Inputs:** JSON-serializable metadata.

- **Outputs:** UTF-8 bytes.

- **Why needed:** AAD must serialize identically at encryption and decryption. Canonical formatting avoids authentication failures caused only by key order or whitespace differences.

### `sha256_file(path: str) -> str`

- **What it does:** Calculates a SHA-256 digest by reading the file in chunks.

- **Inputs:** File path.

- **Outputs:** Hexadecimal digest string.

- **Why needed:** Supports file identity/integrity checks and diagnostic metadata; hashing alone does not encrypt or authenticate arbitrary external data unless used in a verified protocol.

### `normalize_sensitivity(value) -> str`

- **What it does:** Validates and normalizes a sensitivity label.

- **Inputs:** Label value.

- **Outputs:** One of the allowed tier names defined by `TIERS`, normalized to title case.

- **Errors:** Raises `ValueError` for missing or unsupported labels.

- **Why needed:** Prevents unrecognized labels from silently selecting the wrong encryption key.

### `normalize_bbox(bbox) -> list[float]`

- **What it does:** Validates a four-coordinate bounding box, converts coordinates to floats, and orders x/y edges correctly.

- **Inputs:** Bounding box in `[x0, y0, x1, y1]` form.

- **Outputs:** Normalized coordinate list.

- **Errors:** Raises `ValueError` for malformed boxes.

- **Why needed:** Geometry operations require consistent rectangle ordering.

### `clamp_bbox(bbox, width, height) -> list[int]`

- **What it does:** Rounds and clamps a bounding box to the bounds of an image and ensures a non-zero width/height where possible.

- **Inputs:** Bounding box and image width/height.

- **Outputs:** Integer pixel coordinates.

- **Why needed:** Prevents crop operations from requesting pixels outside the rendered page.

### `load_public_key(path) -> RSA public key`

- **What it does:** Loads a PEM public key and verifies its type.

- **Inputs:** Public-key path.

- **Outputs:** RSA public-key object.

- **Why needed:** RSA-OAEP uses the public key to encrypt/wrap the session key.

### `load_private_key(path, password=None) -> RSA private key`

- **What it does:** Loads a PEM private key, optionally using a password, and verifies its type.

- **Inputs:** Private-key path and optional password bytes.

- **Outputs:** RSA private-key object.

- **Why needed:** RSA-OAEP uses the matching private key to recover the wrapped session key.

### `wrap_session_key(session_key: bytes, public_key) -> bytes`

- **What it does:** Encrypts/wraps the random session key with RSA-OAEP using SHA-256.

- **Inputs:** Session-key bytes and recipient's RSA public-key object.

- **Outputs:** RSA ciphertext bytes.

- **Why needed:** RSA is used for the small session key, not for the full PDF content. This is more practical than applying RSA directly to a document.

### `unwrap_session_key(wrapped_key: bytes, private_key) -> bytes`

- **What it does:** Recovers the session key using the matching RSA private key and the same OAEP parameters.

- **Inputs:** Wrapped session-key bytes and RSA private-key object.

- **Outputs:** Original session-key bytes.

- **Why needed:** The AES keys cannot be derived until the session key has been recovered.

### `derive_tier_keys(session_key: bytes, salt: bytes) -> dict`

- **What it does:** Uses HKDF-SHA-256 to derive separate keys for the sensitivity tiers listed in `TIERS`.

- **Inputs:** Session key and random salt.

- **Outputs:** Dictionary mapping each tier name to derived key bytes.

- **Why needed:** Key separation ensures that different sensitivity tiers use different AES keys derived from the same session secret.

- **Important:** The tier key lengths are defined by the code's `TIERS` mapping; consult that constant in the source if documenting exact lengths.

### `load_layout(layout_path: str) -> dict`

- **What it does:** Loads layout JSON and checks that it contains a `pages` list.

- **Inputs:** Layout JSON path.

- **Outputs:** Parsed layout dictionary.

- **Errors:** Raises `ValueError` if the required structure is missing.

- **Why needed:** The encryption process relies on page metadata to align word records and coordinates with the PDF.

### `get_page_number_column(df) -> str`

- **What it does:** Finds a supported page-number column (`page` or `page_number`) in a DataFrame.

- **Inputs:** Token DataFrame.

- **Outputs:** Name of the detected column.

- **Errors:** Raises `ValueError` if neither column exists.

- **Why needed:** Establishes the page key used to match token records with layout data.

### `build_sensitivity_map(tokens_path: str) -> dict`

- **What it does:** Reads the token CSV and creates a dictionary keyed by `(page_number, word_index)` with validated sensitivity labels.

- **Inputs:** Path to the token CSV.

- **Outputs:** Mapping from unique page/word keys to tier names.

- **Errors:** Raises `ValueError` for missing columns/values, duplicate token keys, invalid labels, or an empty input.

- **Why needed:** The encryption module needs an unambiguous sensitivity label for each extracted word.

### `validate_layout_against_pdf(layout, pdf) -> bool`

- **What it does:** Checks that the number of layout pages matches the PDF page count and that page identifiers are unique.

- **Inputs:** Parsed layout dictionary and open PyMuPDF document.

- **Outputs:** `True` if these structural checks pass; otherwise raises `ValueError`.

- **Why needed:** Prevents coordinate metadata from being applied to the wrong page sequence.

### `pdf_index_for_page(page_number, layout) -> int`

- **What it does:** Converts the page label used in the CSV/layout into a zero-based PDF page index, handling zero-based versus one-based labels.

- **Inputs:** Page number and layout dictionary.

- **Outputs:** Zero-based page index.

- **Errors:** Raises `ValueError` if the page number is out of range.

- **Why needed:** Python PDF libraries index pages from zero, while extracted data may label pages starting at one.

### `get_layout_page(layout, page_number) -> dict`

- **What it does:** Finds the layout record corresponding to a page number.

- **Inputs:** Layout dictionary and page label.

- **Outputs:** The matching page dictionary.

- **Errors:** Raises `KeyError` if the page is not present.

- **Why needed:** Retrieves page-specific geometry and metadata.

### `word_bbox_to_display_points(word, pdf_page) -> list`

- **What it does:** Converts a word bounding box to displayed-page coordinates, applying the page rotation matrix for native text when needed.

- **Inputs:** Word dictionary and PyMuPDF page.

- **Outputs:** Bounding box in PDF-point coordinates.

- **Why needed:** Text and image coordinates must line up on rotated pages.

- **Limitation:** The code assumes native and OCR coordinates follow the conventions described in its docstring; verify these assumptions against the extraction output.

### `points_bbox_to_pixels(bbox_points, scale, page_rect, pixel_width, pixel_height, padding_pixels=2) -> list[int]`

- **What it does:** Converts a PDF-point bounding box to rendered-image pixels, adds a small padding, and clamps it to image boundaries.

- **Inputs:** Point-space box, render scale, page rectangle, image dimensions, optional pixel padding.

- **Outputs:** Pixel-space bounding box.

- **Why needed:** The masking stage edits image pixels, which use a different coordinate system from PDF points.

### `render_page(pdf_page, dpi) -> tuple[Image, float]`

- **What it does:** Renders a PDF page to an RGB Pillow image and returns the point-to-pixel scale.

- **Inputs:** PyMuPDF page and DPI.

- **Outputs:** Rendered image and scale (`dpi / 72`).

- **Why needed:** Creates a pixel representation for locating and masking protected regions.

### `estimate_background(image, bbox, ring=4) -> tuple[int, int, int]`

- **What it does:** Estimates the background color around a rectangle by taking per-channel medians from a surrounding ring.

- **Inputs:** Image, rectangle, and ring thickness.

- **Outputs:** RGB tuple.

- **Why needed:** Helps mask text while matching the local page background instead of always painting white.

- **Limitation:** It is an estimate and may not represent textured or changing backgrounds accurately.

### `rectangles_overlap(a, b) -> bool`

- **What it does:** Checks whether two rectangles overlap.

- **Inputs:** Two boxes.

- **Outputs:** Boolean.

- **Why needed:** Identifies word regions that should be merged into a single visual patch.

### `union_bbox(a, b) -> list`

- **What it does:** Returns the smallest bounding rectangle that contains both input boxes.

- **Inputs:** Two boxes.

- **Outputs:** Combined bounding box.

- **Why needed:** Expands patch regions as overlapping word rectangles are grouped.

### `merge_patch_components(word_items) -> list[dict]`

- **What it does:** Groups overlapping word rectangles into larger patch components and assigns each component the highest sensitivity tier among its member words.

- **Inputs:** List of word items containing pixel bounding boxes and sensitivity labels.

- **Outputs:** List of components, each with a combined box, member items, and the highest sensitivity.

- **Why needed:** Overlapping boxes should not be masked independently if doing so could leave small portions of text visible. Using the highest tier avoids protecting a combined patch with a lower-tier key than one of its words requires.

### `encrypt_bytes(data: bytes, key: bytes, aad: bytes) -> dict`

- **What it does:** Generates a fresh 12-byte nonce and encrypts data using AES-GCM with authenticated additional data.

- **Inputs:** Plaintext bytes, AES key, and AAD bytes.

- **Outputs:** Dictionary containing Base64-encoded `nonce` and `ciphertext` (which includes the GCM authentication tag).

- **Why needed:** AES-GCM provides confidentiality and integrity/authenticity for each encrypted record.

- **Implementation note:** This function generates a fresh random nonce for each encryption call, as used by the AES-GCM workflow.

### `decrypt_bytes(record: dict, key: bytes, aad: bytes) -> bytes`

- **What it does:** Decodes the nonce and ciphertext and decrypts/authenticates them using AES-GCM.

- **Inputs:** Record containing `nonce` and `ciphertext`, AES key, and the expected AAD.

- **Outputs:** Original plaintext bytes if authentication succeeds.

- **Errors:** Authentication/decryption fails if the key, nonce, ciphertext, or AAD is incorrect or altered.

- **Why needed:** Ensures protected content is recovered only when the ciphertext and its authenticated context are valid.

### `encrypt_word_text(text_value, page_number, word_index, sensitivity, tier_keys) -> dict`

- **What it does:** Encrypts a word's UTF-8 text with the key for its sensitivity tier and binds its page/index/tier/version metadata as AAD.

- **Inputs:** Text, page number, word index, sensitivity label, and tier-key dictionary.

- **Outputs:** Encrypted record containing nonce/ciphertext and the identifying metadata needed for later decryption.

- **Why needed:** Protects text at word granularity while retaining a link to the word's original position.

### `encrypt_patch(image, component, page_number, patch_index, tier_keys) -> dict`

- **What it does:** Crops a grouped image patch from the rendered page, serializes it as PNG bytes, and encrypts it using the key for the component's sensitivity tier.

- **Inputs:** Rendered page image, patch component, page number, patch index, and tier keys.

- **Outputs:** Encrypted patch record with encrypted image data and metadata, according to the implementation.

- **Why needed:** Masks visual representations of protected text, including content that might otherwise remain visible in the rendered page.

### `attach_payload(pdf_document, payload) -> None`

- **What it does:** Attaches the encrypted payload/metadata to the PDF using the mechanism implemented in the module.

- **Inputs:** Open PyMuPDF document and payload dictionary.

- **Outputs:** Modifies the PDF document; no separate return value is required.

- **Why needed:** Keeps the encrypted content records associated with the protected PDF for later reconstruction.

### `read_payload(pdf_document) -> dict`

- **What it does:** Reads and parses the encrypted payload previously attached to a PDF.

- **Inputs:** Open PyMuPDF document.

- **Outputs:** Payload dictionary.

- **Why needed:** Decryption needs the records and metadata associated with the protected PDF.

### `encrypt_pdf(input_pdf_path, layout_path, tokens_path, public_key_path, output_dir, dpi=300) -> None`

- **What it does:** Orchestrates the end-to-end encryption workflow. It validates inputs, loads the PDF/layout/token labels and public key, creates session/tier keys, renders pages, encrypts sensitive word text and visual patches, masks protected areas, stores the payload, and writes the protected PDF and wrapped session-key metadata.

- **Inputs:**

  - `input_pdf_path`: source PDF.

  - `layout_path`: extraction layout JSON.

  - `tokens_path`: token CSV with sensitivity labels.

  - `public_key_path`: RSA public key PEM.

  - `output_dir`: destination directory.

  - `dpi`: page rendering resolution.

- **Outputs:** Writes the encrypted PDF and encrypted session-key JSON to the configured output directory.

- **Why needed:** This is the main entry point that combines the classification output, layout data, and cryptographic primitives into the protected document.

- **Important:** Exact output names and payload fields are defined by the implementation's constants and write operations.

### `decrypt_pdf(encrypted_pdf_path, session_key_path, private_key_path, output_pdf_path) -> None`

- **What it does:** Orchestrates decryption and reconstruction. It loads the encrypted PDF/payload and wrapped session-key metadata, loads the private key, unwraps the session key, derives the same tier keys, authenticates/decrypts protected records, restores protected text/image regions, and saves the reconstructed PDF.

- **Inputs:**

  - `encrypted_pdf_path`: protected PDF.

  - `session_key_path`: encrypted session-key JSON.

  - `private_key_path`: matching RSA private-key PEM.

  - `output_pdf_path`: destination path for the reconstructed PDF.

- **Outputs:** Writes the reconstructed/decrypted PDF.

- **Why needed:** Provides the reverse workflow and allows the project to test whether the protected content can be recovered with the correct key material.

## Convenience wrappers

### `run_encryption(input_pdf, layout=..., tokens=..., public_key=..., output_dir=..., dpi=300) -> None`

- **What it does:** Calls `encrypt_pdf` with notebook-friendly default paths and prints the expected output locations.

- **Inputs:** Source PDF and optional paths for layout, token labels, public key, output directory, and DPI.

- **Outputs:** Same files produced by `encrypt_pdf`, plus console messages.

- **Why needed:** Simplifies running the main encryption function without repeatedly specifying every path.

### `run_decryption(encrypted_pdf=..., session_key=..., private_key=..., output_pdf=...) -> None`

- **What it does:** Calls `decrypt_pdf` with configured defaults and prints the output path.

- **Inputs:** Encrypted PDF, wrapped session-key JSON, private key, and destination PDF path.

- **Outputs:** Reconstructed PDF plus console message.

- **Why needed:** Provides a simple entry point for testing the decryption workflow.

## Cryptographic flow

1. Generate an RSA key pair.

2. Create a random session key and salt in the encryption workflow.

3. Wrap the session key with RSA-OAEP using the recipient's public key.

4. Derive separate AES keys for sensitivity tiers with HKDF-SHA-256.

5. Encrypt each protected text record and visual patch with AES-GCM using a fresh nonce and authenticated metadata.

6. Store ciphertext and metadata in the protected PDF payload and store the RSA-wrapped session key in the sidecar JSON.

7. During decryption, use the private key to unwrap the session key, derive the same tier keys, verify/decrypt records, and reconstruct the document.

## Security and implementation notes

- **Hybrid encryption:** RSA is used for the small session key; AES-GCM handles the document data.

- **Authentication:** AES-GCM detects modification of ciphertext and authenticated metadata.

- **Base64 is not encryption:** it only makes binary values serializable in JSON.

- **Private-key storage:** the key-generation code writes the private key as a PEM file without password-based encryption. This describes the key-file format used in the current prototype.

- **Selective masking:** the output PDF contains masked regions and an embedded encrypted payload. The masking and payload-reconstruction steps are part of the prototype's selective document-protection workflow.

- **Layout dependence:** page numbering, token indexes, bounding boxes, and page-rotation conventions must remain consistent between extraction, encryption, and decryption for masking and restoration to align correctly.

- **Testing:** the project evaluates the decryption workflow by checking the reconstructed text and visual layout, in addition to confirming that the functions complete successfully.
