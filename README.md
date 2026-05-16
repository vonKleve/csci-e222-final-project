# NLP for Healthcare: Medical Text Analysis
CSCI E-222 — Foundations of Large Language Models — Final Project · Oleksandr Mykhailovskyi

Fine-tunes and compares `Bio_ClinicalBERT` and `MedGemma-4B` on medical NER and multiple-choice QA.

| Task | Dataset | Models |
|---|---|---|
| Named Entity Recognition | [MedMentions-MTI881-NER](https://huggingface.co/datasets/Ben10x/MedMentions-MTI881-NER) | Bio_ClinicalBERT (baseline + weighted), MedGemma-4B |
| Multiple-Choice QA | [MedQA-USMLE-4-options](https://huggingface.co/datasets/GBaker/MedQA-USMLE-4-options) | Bio_ClinicalBERT, MedGemma-4B |

Fine-tuned checkpoints are on the Hub under [`alexd063/`](https://huggingface.co/alexd063).

## Results

| Task | Bio_ClinicalBERT | MedGemma-4B |
|---|---|---|
| NER (seqeval F1) | **0.586** | 0.253 |
| QA (accuracy) | 0.300 | **0.565** |

BERT is better at token-level NER; Gemma handles the reasoning-heavy QA much better.

## Running

All notebooks run on Google Colab. MedGemma needs an **A100** (training peaks at ~20 GB VRAM); BERT notebooks are fine on an **L4**.

Training notebooks are independent and can run in parallel on separate sessions:

| Notebook | GPU | ~Time |
|---|---|---|
| `ner/03-medgemma-lora.ipynb` | A100 | 70 min |
| `ner/01-bioclinicalbert-baseline.ipynb` | A100 / L4 | 33 min |
| `ner/02-bioclinicalbert-weighted.ipynb` | A100 / L4 | 27 min |
| `qa/02-medgemma-lora.ipynb` | A100 | 70 min |
| `qa/01-bioclinicalbert.ipynb` | A100 / L4 | 8 min |
| `00-eda.ipynb` | CPU | 5 min |
| `99-comparative-analysis.ipynb` (eval only, loads from Hub) | A100 | 30 min |

To just reproduce the evaluation without retraining, open `99-comparative-analysis.ipynb` on an A100 and run all cells.
