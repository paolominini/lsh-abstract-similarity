# Finding Similar Items in arXiv Abstracts

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/paolominini/lsh-abstract-similarity/blob/main/similarity.ipynb)

A scalable pipeline for detecting pairs of highly similar paper abstracts in the
[arXiv dataset](https://www.kaggle.com/datasets/Cornell-University/arxiv)
(~3 million documents), using **Shingling → MinHashing → Locality-Sensitive
Hashing (LSH)**. The approach finds near-duplicate pairs without the infeasible
O(n²) all-pairs comparison, and runs end-to-end within the memory limits of a
free Google Colab instance.

Project for the *Algorithms for Massive Data* course.

---

## Method

The pipeline approximates Jaccard similarity over abstracts in four stages:

1. **Shingling** — each abstract is normalized (lowercased, LaTeX and punctuation
   stripped, accents folded) into word tokens, then turned into a set of
   3-word shingles, each hashed to a 32-bit integer.
2. **MinHashing** — every document's shingle set is compressed into a fixed-length
   signature of 100 minimum-hash values, computed with vectorized NumPy and
   streamed to an on-disk memory map so RAM stays bounded.
3. **LSH banding** — signatures are split into 20 bands of 5 rows; documents that
   collide in at least one band become candidate pairs. With these parameters the
   similarity threshold is `t = (1/20)^(1/5) ≈ 0.55`.
4. **Validation** — candidate quality is measured by recomputing exact Jaccard
   similarity on a random sample of candidates, and visualized against the
   theoretical LSH S-curve.

---

## How to run

The project is a single notebook, intended for Google Colab.

1. Open the notebook in Colab via the badge above (or upload `similarity.ipynb`).
2. Get a Kaggle API token: Kaggle → *Account* → *Create New API Token*
   (downloads `kaggle.json`).
3. In the **Data Ingestion** cell, fill in your credentials, replacing the
   placeholders:
   ```python
   os.environ["KAGGLE_USERNAME"] = "your_username"
   os.environ["KAGGLE_KEY"]      = "your_key"
   ```
4. Run all cells top to bottom.

By default the notebook processes the **full dataset** (`USE_WHOLE_DATA = True`).
To run on a smaller sample instead, set `USE_WHOLE_DATA = False` and choose
`SUBSAMPLE_SIZE`.

> **Note:** the committed notebook ships with credentials blanked as
> `"XXXXXXX"`. Never commit a real Kaggle key.

---

## Configuration

All parameters live as constants in the **Setup** cell:

| Parameter | Default | Meaning |
|---|---|---|
| `SHINGLE_K` | 3 | words per shingle |
| `N_HASHES` | 100 | MinHash signature length |
| `BANDS`, `ROWS` | 20, 5 | LSH banding (`BANDS × ROWS == N_HASHES`) |
| `BATCH_SIZE` | 2000 | documents per MinHash batch |
| `MIN_ABSTRACT_WORDS` | 20 | drop abstracts shorter than this |
| `USE_WHOLE_DATA` | True | full dataset vs subsample |
| `SEED` | 42 | reproducibility |

---

## Results

On the full snapshot (**3,080,258** abstracts, **3,064,847** kept after filtering
out abstracts too short to shingle reliably), measured on a standard Colab CPU
runtime:

| Stage | Time |
|---|---|
| MinHash (3M docs) | ~28 min |
| LSH banding | ~2.6 min |

- **~19,900 candidate pairs** retrieved.
- **~63% precision** on a random sample of candidates (exact Jaccard ≥ threshold).
- **Peak memory ~4.5 GB**, flat across data sizes — comfortably under Colab's
  ~12.7 GB limit.

Both MinHash and LSH scale near-linearly with the number of documents, with no
quadratic blow-up — the basis for the scalability claim.

---

## Repository contents

| File | Description |
|---|---|
| `similarity.ipynb` | The complete pipeline (the deliverable) |
| `README.md` | This file |

---

## License

The arXiv dataset is distributed under CC0-1.0 by Cornell University via Kaggle.
