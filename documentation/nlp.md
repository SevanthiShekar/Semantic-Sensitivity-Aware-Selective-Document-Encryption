# Module 2 — NLP Feature Extraction

## Purpose

The NLP module enriches the word-level records produced by PDF extraction. It adds normalized token values and features such as stop-word status, punctuation, numeric values, URLs, email addresses, currency indicators, and named-entity labels.

These features give the sensitivity-classification module more information than raw text alone. **This module creates features; it does not assign the final sensitivity label and does not encrypt the PDF.**

## Inputs and outputs

### Input

- `/content/data/processed/tokens.csv` — the extraction output. The code searches for a token column among `token`, `text`, `word`, `token_text`, and `original_token`.
- It optionally searches for a page column among `page`, `page_num`, `page_number`, and `page_index`. If found, named-entity recognition is run page by page; otherwise all rows are treated as one text group.

### Output

- `/content/data/processed/tokens_nlp.csv` — the original rows plus `base_word`, Boolean feature columns, `is_entity`, and `entity_label`.
- The original `tokens.csv` is not overwritten.

## Libraries and why they are used

| Library | Role |
|---|---|
| `pandas` | Loads the token CSV into a DataFrame, adds feature columns, groups rows, and writes the enriched CSV. |
| `spaCy` (`spacy`) | Tokenizes text and provides lexical attributes such as `is_stop`, `is_punct`, `is_alpha`, `like_num`, `like_url`, `like_email`, and `is_currency`; its English pipeline also provides named-entity recognition. |
| `en_core_web_sm` | The English spaCy pipeline loaded by `spacy.load`. It supplies the trained components needed for NER and other linguistic analysis. It must be installed separately from the `spacy` package. |
| `re` | Removes punctuation from the beginning and end of normalized lookup words without removing internal punctuation. |
| `unicodedata` | Applies Unicode NFKC normalization so compatible Unicode forms are made more consistent for matching. |
| `pathlib.Path` | Defines input/output file paths and checks whether the input file exists. |

## Configuration and data preparation

The module defines `INPUT_CSV` and `OUTPUT_CSV`, verifies that the input exists, loads `en_core_web_sm`, and reads the CSV with `keep_default_na=False` so token strings that resemble missing-value markers are not automatically converted to NaN.

It then detects the token column from a list of supported names. If none is present, it raises a `ValueError` with the available columns. The page column is optional.

The currency fallback lists common currency codes (for example, `INR`, `USD`, and `EUR`) and names (for example, `rupee`, `dollar`, and `yen`) because a currency may appear as a word or code rather than as a currency symbol.

## Function reference

### `make_base_word(value) -> str`

- **What it does:** Converts a token to a normalized lookup form using Unicode NFKC normalization, trimming, lowercasing, and removal of punctuation at the token's outer edges.
- **Inputs:** `value`, the original token value.
- **Outputs:** A normalized string used for dataset lookups.
- **Why needed:** The same word may occur with different casing or surrounding punctuation. Normalization makes lookup matching more consistent.
- **Important detail:** The original token text and its layout metadata are not replaced; the normalized form is stored separately in `base_word`.

### `extract_token_features(original, base_word) -> dict`

- **What it does:** Uses spaCy lexical attributes to calculate seven Boolean features for one extracted token.
- **Inputs:**
  - `original`: the original token text.
  - `base_word`: normalized lookup form returned by `make_base_word`.
- **Outputs:** A dictionary containing:
  - `is_stopword` — true when the normalized value is one spaCy token and spaCy marks it as a stop word.
  - `is_punct` — true when the original value tokenizes entirely as punctuation.
  - `is_url` — true if spaCy identifies a URL-like token.
  - `is_email` — true if spaCy identifies an email-like token.
  - `is_currency` — true if spaCy identifies a currency symbol or the fallback recognizes a listed currency code/name.
  - `is_word` — true when the normalized value is a single alphabetic spaCy token.
  - `is_num` — true if spaCy identifies any token in the original text as number-like.
- **Why needed:** These features help the next module apply explicit sensitivity rules before consulting the reference dataset.
- **Limitations:** spaCy's lexical attributes are heuristics. A feature being true does not prove that the token is sensitive, and a false value does not prove that it is safe.

### Named-entity recognition (NER) pipeline

NER is implemented as a pipeline section rather than a separate named function.

- **What it does:** Reconstructs the tokens into page text, runs the full spaCy pipeline, and checks which token character spans overlap a recognized entity span.
- **Inputs:** Token text rows and, when available, page grouping.
- **Outputs:** `is_entity` (Boolean) and `entity_label` (one or more entity labels separated by `|`). Labels may include examples such as `PERSON`, `ORG`, and `GPE`, depending on what the model recognizes.
- **Why needed:** Names of people, organizations, and places may need more careful treatment during sensitivity classification.
- **Limitations:** Entity detection is not always accurate. Since the module reconstructs text by joining token rows with spaces, results depend on tokenization and row order.

## Output columns added

| Column | Meaning |
|---|---|
| `base_word` | Normalized value used for matching the sensitivity reference dataset. |
| `is_stopword` | Whether the normalized token is a spaCy stop word. |
| `is_punct` | Whether the original token consists only of punctuation. |
| `is_url` | Whether the token appears URL-like. |
| `is_email` | Whether the token appears email-like. |
| `is_currency` | Whether the token is recognized as a currency symbol/code/name. |
| `is_word` | Whether the normalized value is one alphabetic token. |
| `is_num` | Whether the token appears numeric. |
| `is_entity` | Whether the token overlaps a recognized named entity. |
| `entity_label` | Entity class or classes assigned by spaCy. |

## Why the output is saved separately

The NLP module writes `tokens_nlp.csv` rather than replacing `tokens.csv`. This preserves the extraction-stage artifact and makes it easier to compare raw extraction output with enriched NLP output when debugging or documenting the pipeline.

## Pipeline summary

1. Load and validate `tokens.csv`.
2. Detect the token and optional page columns.
3. Create `base_word` for each token.
4. Calculate lexical feature flags.
5. Run named-entity recognition by page where page identifiers exist.
6. Normalize Boolean columns and save `tokens_nlp.csv`.
7. Pass the enriched file to sensitivity classification.
