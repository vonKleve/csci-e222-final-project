# NLP for Healthcare: Medical Text Analysis
### CSCI E-222 — Foundations of Large Language Models — Final Project

Oleksandr Mykhailovskyi

## Project Overview

Fine tunes and compares two medical domain specialized models on NER and QA tasks:

| Task | Dataset | Models |
|---|---|---|
| Named Entity Recognition (NER) | [Ben10x/MedMentions-MTI881-NER](https://huggingface.co/datasets/Ben10x/MedMentions-MTI881-NER) | `Bio_ClinicalBERT`, `MedGemma-4B` |
| Multiple-Choice QA | [GBaker/MedQA-USMLE-4-options](https://huggingface.co/datasets/GBaker/MedQA-USMLE-4-options) | `Bio_ClinicalBERT`, `MedGemma-4B` |

All datasets are downloaded automatically via the Hugging Face `datasets` library.

Fine-tuned model checkpoints are published to the Hugging Face Hub under `alexd063/`:

| Checkpoint | Task |
|---|---|
| [`alexd063/bio-clinicalbert-finetuned-medmentions-v1`](https://huggingface.co/alexd063/bio-clinicalbert-finetuned-medmentions-v1) | NER (classification) |
| [`alexd063/gemma4bit-finetuned-medmentions`](https://huggingface.co/alexd063/gemma4bit-finetuned-medmentions) | NER (generative) |
| [`alexd063/bio-clinicalbert-finetuned-medqa`](https://huggingface.co/alexd063/bio-clinicalbert-finetuned-medqa) | QA (classification) |
| [`alexd063/gemma4bit-finetuned-medqa`](https://huggingface.co/alexd063/gemma4bit-finetuned-medqa) | QA (generative) |

## Compute Environment

**MedGemma-4B training requires Google Colab Pro — A100 GPU (40 GB VRAM)**

- `MedGemma-4B` is loaded in 4-bit quantization (QLoRA/Unsloth), which requires ~4 GB of VRAM for weights; training peaks at approximately 20 GB.
- `Bio_ClinicalBERT` notebooks, the EDA notebook, and the comparative analysis notebook can run on a **Colab L4 GPU** (24 GB VRAM), though training will be ~4–5× slower than on A100.

### Colab Setup From 0

This will re-fine-tune all models, push checkpoints to the Hub, and evaluate them.

Training notebooks (steps 1–5 are independent and can run in parallel on separate sessions):

| Step | Notebook | GPU |
|---|---|---|
| 1 | `ner/03-medgemma-lora.ipynb` | A100 |
| 2 | `ner/01-bioclinicalbert-baseline.ipynb` | A100 or L4 |
| 3 | `ner/02-bioclinicalbert-weighted.ipynb` | A100 or L4 |
| 4 | `qa/02-medgemma-lora.ipynb` | A100 |
| 5 | `qa/01-bioclinicalbert.ipynb` | A100 or L4 |

For each notebook:
1. Open in Google Colab.
2. **Runtime → Change runtime type → select GPU**.
3. Press **Run All Cells**.

After all training notebooks complete, run `99-comparative-analysis.ipynb` (loads published Hub checkpoints).

### Colab Setup for Evaluation Only

This will load fine-tuned models from the Hugging Face Hub and run inference only.

1. Open `99-comparative-analysis.ipynb` in Google Colab.
2. **Runtime → Change runtime type → A100 GPU**.
3. Press **Run All Cells**.

## Notebooks — Full Execution Order

Run the training notebooks before the comparative analysis. The comparative analysis loads checkpoints from the Hub, so the training notebooks must have been run (and models pushed) at least once.

```
ner/03-medgemma-lora.ipynb             # 1. Fine-tune MedGemma-4B for NER       (A100, ~70 min)
ner/01-bioclinicalbert-baseline.ipynb  # 2. Fine-tune Bio_ClinicalBERT NER baseline (A100/L4, ~33 min)
ner/02-bioclinicalbert-weighted.ipynb  # 3. Fine-tune Bio_ClinicalBERT NER weighted (A100/L4, ~27 min)
qa/02-medgemma-lora.ipynb              # 4. Fine-tune MedGemma-4B for QA        (A100, ~70 min)
qa/01-bioclinicalbert.ipynb            # 5. Fine-tune Bio_ClinicalBERT for QA   (A100/L4, ~8 min)
00-eda.ipynb                           # 6. Exploratory Data Analysis           (CPU/any, ~5 min)
99-comparative-analysis.ipynb          # 7. Head-to-head comparison             (A100, ~30 min)
```

Steps 1–5 are independent of each other and can be run in any order (or in parallel on separate Colab sessions).

## Notebook Descriptions

### `ner/01-bioclinicalbert-baseline.ipynb` — Bio_ClinicalBERT NER (Baseline)
Treats NER as **token classification**. Fine-tunes `Bio_ClinicalBERT` as `AutoModelForTokenClassification` with a standard cross-entropy loss over 127 UMLS semantic type labels. Uses cosine LR scheduling, 10% warmup, and early stopping (patience=3) on seqeval F1. Produces training curves, per-entity F1 bar chart, classification report, and sample predictions.

### `ner/02-bioclinicalbert-weighted.ipynb` — Bio_ClinicalBERT NER (Weighted)
Identical to the baseline but replaces the standard `Trainer` with a custom `WeightedTrainer` using exponential inverse-frequency class weights to address the severe `O`-label imbalance in MedMentions. The weighted test loss is not comparable to the baseline due to the loss function change. Produces training curves, per-entity F1 breakdown (top/bottom entities), and confusion matrix.

### `ner/03-medgemma-lora.ipynb` — MedGemma-4B NER
Treats NER as a **text-to-text generation** task. Fine-tunes `google/medgemma-4b-it` with QLoRA (rank 16, attention layers only) via Unsloth for 300 steps (~20% of one epoch). Produces training/validation loss curves, quantitative seqeval evaluation on 100 test samples, and qualitative prediction samples.

### `qa/01-bioclinicalbert.ipynb` — Bio_ClinicalBERT QA
Fine-tunes `Bio_ClinicalBERT` as `AutoModelForMultipleChoice`. Each question is presented as 4 `[Question, Option]` pairs; a linear classifier over the `[CLS]` token scores each option. Metrics: accuracy, macro F1, ECE, reliability diagram, and 4×4 confusion matrix. Uses left-side truncation at 512 tokens.

### `qa/02-medgemma-lora.ipynb` — MedGemma-4B QA
Treats QA as **text-to-text generation**. Fine-tunes `google/medgemma-4b-it` with QLoRA (rank 16, attention + MLP layers) via Unsloth for 2 epochs. The model is trained to generate exactly the answer letter (`A`/`B`/`C`/`D`) followed by `<eos>`. Implements a custom `MedicalQAEvaluationCallback` (50-sample validation subset per epoch). Produces training curves, 4×4 confusion matrix, and per-option classification report.

### `00-eda.ipynb` — Exploratory Data Analysis
Analyses both datasets prior to modelling. Covers NER label distribution (log scale), token length distribution by split, entity co-occurrence heatmap, entity support buckets, and MedQA answer option distribution and input length analysis. Does not require a GPU.

### `99-comparative-analysis.ipynb` — Head-to-Head Comparison
Loads all four fine-tuned checkpoints from the Hub (Gemma models via Unsloth `FastLanguageModel`) and evaluates them side by side:
- **NER**: Bio_ClinicalBERT pipeline and MedGemma-4B generative output, both on 100 test samples. Reports seqeval precision/recall/F1.
- **QA**: Both models on 200 test samples. Reports accuracy, macro F1, side-by-side confusion matrices, summary table, and radar chart.

## Datasets

| Dataset | Task | Size | Source |
|---|---|---|---|
| MedMentions-MTI881-NER | NER | ~4,392 documents / ~350K entity mentions | [HuggingFace](https://huggingface.co/datasets/Ben10x/MedMentions-MTI881-NER) |
| MedQA-USMLE-4-options | Multiple Choice QA | ~11,500 USMLE-style questions | [HuggingFace](https://huggingface.co/datasets/GBaker/MedQA-USMLE-4-options) |

## Approximate Runtimes (A100 40 GB)

| Notebook | Estimated Time |
|---|---|
| `ner/03-medgemma-lora.ipynb` (300 steps) | ~60–70 min |
| `ner/01-bioclinicalbert-baseline.ipynb` (up to 20 epochs, early stop) | ~33 min |
| `ner/02-bioclinicalbert-weighted.ipynb` (up to 20 epochs, early stop) | ~27 min |
| `qa/02-medgemma-lora.ipynb` (2 epochs) | ~60–70 min |
| `qa/01-bioclinicalbert.ipynb` (up to 10 epochs, early stop) | ~8 min |
| `00-eda.ipynb` (no GPU required) | ~5 min |
| `99-comparative-analysis.ipynb` (inference only) | ~20–40 min |
