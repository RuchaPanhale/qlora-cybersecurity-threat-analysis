# QLoRA Fine-Tuning for Cybersecurity Threat Analysis JSON Generation

## Project Overview

This project fine-tunes a small open-weight instruction model for a cybersecurity structured generation task.

The goal is to convert cybersecurity logs or incident descriptions into structured threat-analysis JSON with the following fields:

- `attack_type`
- `severity`
- `affected_asset`
- `recommended_action`
- `confidence`

The project compares a base model with zero-shot prompting, few-shot prompting, QLoRA v1, and QLoRA v2.

## Base Model

The base model used in this project is:

`Qwen/Qwen2.5-1.5B-Instruct`

## Fine-Tuning Method

The model is fine-tuned using QLoRA/LoRA adapters with 4-bit quantization. This avoids full fine-tuning of the entire model and trains only a small number of adapter parameters.

The v1 LoRA adapter trained approximately 9.23 million trainable parameters out of approximately 1.55 billion total parameters.

## Dataset

The raw dataset source was a Kaggle mirror of the UNSW-NB15 cybersecurity dataset:

`harshwardhanbhangale/unsw-complete-dataset`

Raw dataset size:

- 2,540,047 rows

The raw dataset was not used directly. It was curated through the following steps:

1. Loaded the four raw UNSW-NB15 CSV files and the feature-name file.
2. Assigned official column names to the raw CSV files.
3. Cleaned whitespace and inconsistent attack category labels.
4. Mapped original UNSW-NB15 attack categories into a smaller project taxonomy.
5. Replaced missing attack categories with `benign` for rows where the binary label was 0.
6. Sampled a mostly class-balanced subset for v1.
7. Converted each selected network-flow row into a natural-language cybersecurity event description.
8. Created structured JSON targets with `attack_type`, `severity`, `affected_asset`, `recommended_action`, and `confidence`.
9. Split curated examples into train/dev using stratified sampling by `attack_type`.

The `attack_type` label was mapped from the UNSW-NB15 attack category. The `severity`, `affected_asset`, `recommended_action`, and `confidence` fields were derived using transparent rule-based heuristics designed for this structured generation task.

## Dataset Sizes

| Split | Examples |
|---|---:|
| Curated v1 total | 2,850 |
| Train v1 | 2,422 |
| Dev v1 | 428 |
| Train v2 | 2,614 |
| Manual iteration eval | 30 |
| Final blind eval | 30 |

## Attack Type Taxonomy

The project uses the following attack types:

- `benign`
- `generic_attack`
- `exploit`
- `dos`
- `reconnaissance`
- `fuzzing`
- `analysis`
- `backdoor`
- `shellcode`
- `worm`
- `unknown`

## Evaluation Design

This project uses two manual evaluation sets:

1. **Manual iteration evaluation set**: used to evaluate zero-shot, few-shot, QLoRA v1, and QLoRA v2 during development. The QLoRA v1 errors on this set were used for failure analysis and informed the targeted v2 augmentation.
2. **Final blind evaluation set**: created after the v2 method was finalized. This set was used as the main final evaluation to avoid overstating v2 performance.

Systems compared:

1. Base model zero-shot prompting
2. Base model few-shot prompting
3. QLoRA v1
4. QLoRA v2

Metrics:

- JSON validity rate
- Schema validity rate
- Attack type accuracy
- Severity accuracy
- Affected asset accuracy
- Numeric confidence rate

## Manual Iteration Evaluation Results

These results were used during development to analyze QLoRA v1 failure patterns and motivate QLoRA v2.

| System | JSON Valid | Schema Valid | Attack Accuracy | Severity Accuracy | Asset Accuracy | Numeric Confidence |
|---|---:|---:|---:|---:|---:|---:|
| Base zero-shot | 1.00 | 0.00 | 0.433 | 0.500 | 0.000 | 0.00 |
| Base few-shot | 1.00 | 1.00 | 0.233 | 0.600 | 0.133 | 1.00 |
| QLoRA v1 | 1.00 | 1.00 | 0.667 | 0.633 | 0.567 | 1.00 |
| QLoRA v2 | 1.00 | 1.00 | 0.900 | 1.000 | 0.800 | 1.00 |

## Final Blind Evaluation Results

These are the main final results because the final blind evaluation set was created after v2 was finalized.

| System | JSON Valid | Schema Valid | Attack Accuracy | Severity Accuracy | Asset Accuracy | Numeric Confidence |
|---|---:|---:|---:|---:|---:|---:|
| Base zero-shot | 1.00 | 0.00 | 0.333 | 0.467 | 0.000 | 0.00 |
| Base few-shot | 1.00 | 1.00 | 0.300 | 0.633 | 0.033 | 1.00 |
| QLoRA v1 | 1.00 | 1.00 | 0.500 | 0.567 | 0.500 | 1.00 |
| QLoRA v2 | 1.00 | 1.00 | 0.767 | 1.000 | 0.733 | 1.00 |

## V1 to V2 Iteration

After evaluating QLoRA v1 on the manual iteration set, the main observed weaknesses were:

- Worm examples were often confused with DoS.
- Backdoor examples were sometimes predicted as benign.
- Critical severity was sometimes downgraded.
- Affected asset labels were sometimes semantically reasonable but not normalized to the project taxonomy.
- `network_service` and `network_host` were frequently confused.

For QLoRA v2, the project used targeted oversampled augmentation focused on these v1 failure patterns. The base model and general training approach were kept the same so that the v2 change was focused and interpretable.

On the final blind evaluation set, QLoRA v2 improved over QLoRA v1:

- Attack accuracy improved from 50.0% to 76.7%.
- Severity accuracy improved from 56.7% to 100.0%.
- Affected asset accuracy improved from 50.0% to 73.3%.

## Capability Regression Evaluation

To check whether fine-tuning harmed unrelated capabilities, the project evaluated the base model and QLoRA v2 on a small manually created regression benchmark with two categories:

1. Reasoning/arithmetic
2. Instruction following

| System | Examples | Exact Match | Contains Expected |
|---|---:|---:|---:|
| Base model | 20 | 0.50 | 0.70 |
| QLoRA v2 | 20 | 0.75 | 0.85 |

No measurable regression was observed on this small benchmark. QLoRA v2 performed slightly better, possibly because the fine-tuning task encouraged concise structured outputs.

## Training Loss Summary

| Run | Final Validation Loss |
|---|---:|
| QLoRA v1 | 0.118607 |
| QLoRA v2 | 0.112683 |

Both runs showed decreasing validation loss. QLoRA v2 ended with a lower validation loss than QLoRA v1.

## Important Files

### Data

- `data/processed/train_v1.jsonl`
- `data/processed/dev_v1.jsonl`
- `data/processed/train_v2.jsonl`
- `data/processed/manual_eval.jsonl`
- `data/processed/final_blind_eval.jsonl`
- `data/processed/train_v1_sft.jsonl`
- `data/processed/train_v2_sft.jsonl`
- `data/processed/dataset_summary_v1.json`

### Results

- `results/final_model_comparison.csv`
- `results/final_blind_model_comparison.csv`
- `results/final_blind_all_results.csv`
- `results/lora_v1_results.csv`
- `results/lora_v2_results.csv`
- `results/regression_all_results.csv`
- `results/regression_summary_overall.csv`
- `results/v1_failure_analysis_notes.json`
- `results/final_experiment_summary.json`

### Plots

- `results/plots/main_task_accuracy_comparison.png`
- `results/plots/final_blind_accuracy_comparison.png`
- `results/plots/regression_comparison.png`
- `results/plots/validation_loss_comparison.png`

### Adapters

- `adapters/lora_v1_final/`
- `adapters/lora_v2_final/`

## Main Conclusion

QLoRA v2 substantially improved structured cybersecurity JSON generation over zero-shot prompting, few-shot prompting, and QLoRA v1 on a separate final blind evaluation set. The strongest improvements were on `attack_type`, `severity`, and `affected_asset`. The small capability regression benchmark did not show measurable degradation after fine-tuning.
