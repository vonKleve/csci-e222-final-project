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

**Required: Google Colab Pro — A100 GPU (40 GB VRAM)**

- `MedGemma-4B` is loaded in 4-bit quantization (QLoRA/Unsloth), which requires ~4 GB of VRAM for weights, training brings peak usage to approximately 20 GB.
- `Bio_ClinicalBERT` (NER) uses batch size 16 at 512 tokens, A100 fits comfortably this model in terms of memory, and gives significant performance boost due to its hardware compared to L4 and older, smaller models.

### Colab Setup From 0

This will re-fine tune all the models, fetch them and evaluate them.

Models notebooks: `010-gemma4bit.ipynb`, `011c-bioclinicalbert.ipynb`, `020-gemma4bit.ipynb`, `020-bioclinicalbert-qa.ipynb`
Comparative evaluation notebook: `comparative_analysis.ipynb`

1. Open models notebooks from this repository in Google Colab.
2. In the menu: **Runtime -> Change runtime type -> A100 GPU**.
3. Press **Run All Cells**. Depending on the notebook, this may take some time. Refer to the Approximate Runtimes (A100 40 GB) for more details.

### Colab Setup for Evaluation Only

This will fetch fine tuned models and evaluate them.

1. Open `comparative_analysis.ipynb` in Google Colab.
2. In the menu: **Runtime -> Change runtime type -> A100 GPU**.
3. Press **Run All Cells**.

## Notebooks — Full Execution Order

Run the training notebooks before the comparative analysis. The comparative analysis loads checkpoints from the Hub, so the training notebooks must have been run (and models pushed) at least once.

```
010-gemma4bit.ipynb           # 1. Fine-tune MedGemma-4B for NER
011c-bioclinicalbert.ipynb    # 2. Fine-tune Bio_ClinicalBERT for NER
020-gemma4bit.ipynb           # 3. Fine-tune MedGemma-4B for QA
020-bioclinicalbert-qa.ipynb  # 4. Fine-tune Bio_ClinicalBERT for QA
comparative_analysis.ipynb    # 5. Head-to-head comparison (requires Hub checkpoints)
```

Steps 1–4 are independent of each other and can be run in any order (or in parallel on separate Colab sessions).

## Notebook Descriptions

### `010-gemma4bit.ipynb` — MedGemma NER
Treats NER as a **text-to-text generation** task. Fine-tunes `google/medgemma-4b-it` with QLoRA (rank 16) via Unsloth on MedMentions. Monitored with training/validation loss.

### `011c-bioclinicalbert.ipynb` — Bio_ClinicalBERT NER
Treats NER as **token classification**. Implements a `WeightedTrainer` with inverse frequency class weights to alleviate the `O` label imbalance in MedMentions. Evaluated with seqeval precision/recall/F1 on the test split. Includes per-class F1 bar chart.

### `020-bioclinicalbert-qa.ipynb` — Bio_ClinicalBERT QA
Fine-tunes `Bio_ClinicalBERT` as `AutoModelForMultipleChoice`. Each question is fed as 4 `[Question, Option]` pairs; a linear head over the `[CLS]` token scores each. Metrics: accuracy, macro F1, and Expected Calibration Error (ECE).

### `020-gemma4bit.ipynb` — MedGemma QA
Treats QA as **text-to-text generation**. The model is trained to generate exactly the answer letter (`A`/`B`/`C`/`D`) followed by `<eos>`. Implements a custom `MedicalQAEvaluationCallback` that estimates accuracy and macro F1 on a validation sample at each epoch. Final evaluation runs on the full test split.

### `comparative_analysis.ipynb` — Head-to-Head Comparison
Loads all four fine-tuned checkpoints from the Hub and evaluates them:
- **NER**: BERT on the full test split, Gemma on a 10% random subset (runtime constraint). Reports seqeval precision/recall/F1.
- **QA**: Both models on 200 test samples. Reports accuracy, macro F1, bar chart comparison.

Includes conclusions on architectural trade offs and suggested improvements.

## Datasets

| Dataset | Task | Size | Source |
|---|---|---|---|
| MedMentions-MTI881-NER | NER | ~4,392 documents / ~350K entity mentions | [HuggingFace](https://huggingface.co/datasets/Ben10x/MedMentions-MTI881-NER) |
| MedQA-USMLE-4-options | Multiple Choice QA | ~12,723 USMLE-style questions | [HuggingFace](https://huggingface.co/datasets/GBaker/MedQA-USMLE-4-options) |

## Approximate Runtimes (A100 40 GB)

| Notebook | Estimated Time |
|---|---|
| `010-gemma4bit.ipynb` (300 steps) | ~60–90 min |
| `011c-bioclinicalbert.ipynb` (up to 20 epochs, early stop) | ~30–45 min |
| `020-gemma4bit.ipynb` (2 epochs) | ~60–90 min |
| `020-bioclinicalbert-qa.ipynb` (up to 10 epochs, early stop) | ~30–60 min |
| `comparative_analysis.ipynb` (inference only) | ~20–40 min |
