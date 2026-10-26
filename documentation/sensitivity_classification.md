# Module 3 — Sensitivity Classification

## Purpose

The sensitivity-classification module assigns a sensitivity label to each token in `tokens_nlp.csv`. It combines two sources of information:

1. Explicit rules based on NLP features.
2. A reference dataset (`combined_dataset.csv`) mapping known tokens to a predefined `sensitivity_level`.

The resulting `tokens_with_sensitivity.csv` is consumed by the cryptography module to decide which words or visual regions receive protection.

## Inputs and outputs

### Inputs

- `combined_dataset.csv` — expected to contain `token` and `sensitivity_level` columns.
- `tokens_nlp.csv` — the enriched token file from the NLP module, expected to contain `base_word` and the feature flags used by the rules.
- `input_dir` — directory containing both files.

### Output

- `tokens_with_sensitivity.csv` — a copy of the NLP-enriched rows with an added `sensitivity` column.

The function defaults to `input_dir="data/processed"`, but the actual path must match where the preceding modules saved their files. In a notebook using `/content/data/processed`, pass that directory explicitly or update the default consistently.

## Libraries and why they are used

| Library | Role |
|---|---|
| `pandas` | Reads both CSV files, builds/uses tabular data, applies row-wise classification, and writes the result. |
| `os` | Joins directory and filename components into paths and creates the output directory if necessary. |

## Function reference

### `populate_token_sensitivity(input_dir="data/processed", combined_filename="combined_dataset.csv", tokens_filename="tokens_nlp.csv", output_filename="tokens_with_sensitivity.csv") -> None`

- **What it does:** Loads the reference sensitivity dataset and the NLP-enriched token data, applies the classification rules to every token, and saves the resulting CSV.
- **Inputs:**
  - `input_dir`: directory containing the input files and receiving the output.
  - `combined_filename`: reference dataset filename.
  - `tokens_filename`: NLP-enriched token filename.
  - `output_filename`: name for the classified output.
- **Outputs:** Writes `tokens_with_sensitivity.csv`. The function prints the path after saving; it does not return the DataFrame.
- **Why needed:** Separates the policy/rule stage from extraction and NLP, and creates a per-token label for the encryption module.

### `apply_rules(row) -> str` (nested function inside `populate_token_sensitivity`)

- **What it does:** Assigns the sensitivity label for one token row using a fixed priority order.
- **Inputs:** A pandas Series representing one row from `tokens_nlp.csv`, including the feature flags and `base_word`.
- **Outputs:** One label string: `"High"`, `"Low"`, or the value found in the reference dataset.
- **Why needed:** Applies consistent classification policy before encryption.
- **Rule order used by the code:**
  1. **High:** If any of `is_num`, `is_currency`, `is_url`, `is_email`, or `is_entity` is true, return `"High"`.
  2. **Low:** Otherwise, if `is_stopword` or `is_punct` is true, return `"Low"`.
  3. **Reference lookup:** Otherwise, if `base_word` exists in the lookup dictionary made from `combined_dataset.csv`, return that token's `sensitivity_level`.
  4. **Fallback:** If the token is not found, return `"High"`.

## How the reference mapping works

The module constructs a dictionary equivalent to:

`token -> sensitivity_level`

using the `token` and `sensitivity_level` columns in `combined_dataset.csv`. It then looks up each row's `base_word`. This means the reference dataset's `token` values should be normalized consistently with `base_word`; otherwise, a valid token may fail to match and trigger the fallback label.

Duplicates have been removed from the reference dataset as much as possible. If the reference dataset contains duplicate `token` values, Python's `dict` construction keeps the last value for that token. The code does not explicitly report duplicates, so the reference dataset should be checked for uniqueness.

## Classification priority and implications

| Priority | Condition | Label returned |
|---|---|---|
| 1 | Numeric, currency, URL, email, or named-entity feature is true | `High` |
| 2 | Stop-word or punctuation feature is true, and no earlier high rule matched | `Low` |
| 3 | `base_word` is found in the reference mapping | Dataset's `sensitivity_level` |
| 4 | No reference match | `High` |

The order matters. For example, a token that is both numeric-like and otherwise matches a low category is labelled `High`, because the high-sensitivity rule runs first.

## Important implementation notes

- The fallback for unknown tokens is deliberately `High` in the current code. This is a conservative default, but it may increase the amount of content protected.
- The code converts feature values with Python's `bool(...)`. CSV values should therefore be loaded as actual Boolean values or carefully normalized; arbitrary non-empty strings such as `"False"` can evaluate to `True` in Python. Verify the data types produced by `pandas.read_csv` if classifications look unexpected.
- The reference lookup uses `base_word` but builds the dictionary from the reference column named `token`. These values need compatible normalization.
- The function does not validate that reference labels are limited to the three expected tiers. The cryptography module performs its own tier validation when loading sensitivity labels.
- This module is rule-based; it does not train a machine-learning classifier.

## Pipeline summary

1. Read the reference dataset and NLP output.
2. Build a token-to-sensitivity lookup.
3. Apply high-priority feature rules.
4. Apply low-priority stop-word/punctuation rules.
5. Look up the normalized token in the reference dataset.
6. Default unknown tokens to `High`.
7. Save `tokens_with_sensitivity.csv` for cryptographic processing.
