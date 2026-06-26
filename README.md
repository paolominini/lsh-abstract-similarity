# Finding Similar Items in arXiv Abstracts

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/paolominini/lsh-abstract-similarity/blob/main/similarity.ipynb)

A scalable pipeline for detecting pairs of highly similar paper abstracts in the
[arXiv dataset](https://www.kaggle.com/datasets/Cornell-University/arxiv)
(~3 million documents), using **Shingling → MinHashing → Locality-Sensitive
Hashing (LSH)**. The approach finds near-duplicate pairs without the infeasible
O(n²) all-pairs comparison, and runs end-to-end on the **full dataset** within
the memory limits of a free Google Colab instance.

Project for the *Algorithms for Massive Data* course
(Master in Data Science for Economics, Università degli Studi di Milano).

---

## Method

The pipeline approximates Jaccard similarity over abstracts in four stages:

1. **Shingling** — each abstract is normalized (lowercased, LaTeX and punctuation
   stripped, accents folded) into word tokens, then turned into a set of
   3-word shingles, each hashed to a 32-bit integer with `zlib.crc32`.
2. **MinHashing** — every document's shingle set is compressed into a fixed-length
   signature of 100 minimum-hash values, computed with vectorized NumPy
   (broadcast affine hashing + `np.minimum.reduceat`) and streamed to an on-disk
   `np.memmap` so RAM stays bounded.
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

## Results — full dataset

Run on the complete arXiv snapshot, on a standard Colab CPU runtime.

**Ingestion:** 3,080,258 abstracts scanned → **3,064,847 kept** after dropping
15,411 abstracts shorter than 20 words (mostly withdrawal notices that produced
spurious matches).

**Pipeline output:**

| Metric | Value |
|---|---|
| Documents processed | 3,064,847 |
| MinHash time | 28.4 min |
| LSH time | 2.6 min |
| **Total time** | **31.0 min** |
| Candidate pairs | 19,875 |
| Precision (1,000 sampled, exact Jaccard ≥ t) | **62.6%** |
| Peak memory (RSS) | **~4.5 GB** |

---

## Scalability

The pipeline was benchmarked on increasing prefixes of the dataset, timing the
MinHash and LSH stages separately and tracking resident memory. Both stages grow
near-linearly with the number of documents — no quadratic blow-up — and memory
stays flat regardless of corpus size.

| Documents (n) | MinHash | LSH | Total | Candidates | RSS |
|---|---|---|---|---|---|
| 10,000 | 5.8 s | 0.2 s | 6.0 s | 30 | 4.51 GB |
| 20,000 | 7.9 s | 0.4 s | 8.3 s | 70 | 4.51 GB |
| 50,000 | 24.9 s | 2.5 s | 27.4 s | 296 | 4.52 GB |
| 100,000 | 44.1 s | 5.6 s | 49.8 s | 758 | 4.54 GB |
| **3,064,847** | **28.4 min** | **2.6 min** | **31.0 min** | **19,875** | **4.50 GB** |

Two things to read off this table:
- **Time** scales with `n` and never explodes — going from 100k to the full 3M
  docs (≈30×) raises total time by roughly the same factor, confirming the
  pipeline bypasses the O(n²) bottleneck.
- **Memory** stays flat at ~4.5 GB across a 300× range in `n`, comfortably under
  Colab's ~12.7 GB limit. This is the payoff of streaming ingestion, batched
  MinHash, and on-disk memory-mapped signatures.

---

## Repository contents

| File | Description |
|---|---|
| `similarity.ipynb` | The complete pipeline (the deliverable) |
| `report.pdf` | Project report |
| `images/` | Figures (S-curve, candidate-quality histogram, scalability plot) |
| `README.md` | This file |

---

## License

The arXiv dataset is distributed under CC0-1.0 by Cornell University via Kaggle.
