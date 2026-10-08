# QLoRA Fine-Tuning for Cybersecurity Threat Analysis JSON Generation

## 1. Introduction

This project investigates whether LoRA/QLoRA fine-tuning can improve a small open-weight instruction model on a narrow cybersecurity structured generation task. The goal is to build a small AI security assistant that converts cybersecurity logs or incident descriptions into structured threat-analysis JSON.

The model receives an input such as a network-flow description or a natural-language incident report and must output valid JSON with the following fields:

- `attack_type`
- `severity`
- `affected_asset`
- `recommended_action`
- `confidence`

This task was chosen because it is narrow, measurable, realistic for a small model, and relevant to AI-assisted cybersecurity workflows. It is also suitable for evaluating whether LoRA fine-tuning provides a measurable benefit over prompting-only baselines.

The base model used was `Qwen/Qwen2.5-1.5B-Instruct`. The model was fine-tuned using QLoRA/LoRA adapters with 4-bit quantization rather than full fine-tuning. This kept the project feasible on a Colab T4 GPU while still allowing task-specific adaptation.

## 2. Dataset Construction

The raw source dataset was a Kaggle mirror of the UNSW-NB15 cybersecurity dataset:

`harshwardhanbhangale/unsw-complete-dataset`

The raw dataset contained 2,540,047 rows of network-flow records. The raw dataset was not used directly. Instead, I curated it into a structured instruction-tuning dataset.

The dataset construction process involved the following steps:

1. Loading the four raw UNSW-NB15 CSV files and the official feature-name file.
2. Assigning official column names to the raw CSV files.
3. Cleaning whitespace and inconsistent attack category labels.
4. Mapping original UNSW-NB15 attack categories into a smaller project taxonomy.
5. Replacing missing attack categories with `benign` for rows where the binary label was 0.
6. Sampling a mostly class-balanced subset for v1.
7. Converting each selected network-flow row into a natural-language cybersecurity event description.
8. Creating structured JSON targets with `attack_type`, `severity`, `affected_asset`, `recommended_action`, and `confidence`.
9. Splitting curated examples into train/dev sets using stratified sampling by `attack_type`.

The `attack_type` label was mapped from the UNSW-NB15 attack category. The remaining output fields were derived using transparent rule-based heuristics. Specifically, `severity` was assigned based on the attack family, `affected_asset` was inferred from network-flow/service information and from the event description, `recommended_action` was chosen from attack-type-specific response templates, and `confidence` was assigned as a numeric heuristic score. This made the task measurable and reproducible while still requiring the model to learn the project-specific output schema and taxonomy.

The final v1 curated dataset contained 2,850 examples. These were split into 2,422 training examples and 428 development examples. A v2 training set with 2,614 examples was later created by adding targeted oversampled examples based on v1 failure analysis.

The attack taxonomy used in this project was:

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

The output severity values were:

- `low`
- `medium`
- `high`
- `critical`

## 3. Model and Fine-Tuning Method

The base model was `Qwen/Qwen2.5-1.5B-Instruct`. I used QLoRA/LoRA adapter fine-tuning with 4-bit quantization. This allowed the model to be trained efficiently on limited GPU resources.

For v1, the LoRA configuration used approximately 9.23 million trainable parameters out of approximately 1.55 billion total parameters, meaning only about 0.59% of the model parameters were trainable. This is consistent with the goal of parameter-efficient fine-tuning.

The v1 training setup used:

- 1 training epoch
- batch size 1
- gradient accumulation steps 8
- learning rate 2e-4
- 4-bit quantized base model
- LoRA target modules including attention and MLP projection layers

For v2, I continued from the v1 LoRA adapter and trained on the augmented v2 dataset with a lower learning rate of 1e-4. The base model, prompt format, output schema, and evaluation metrics were kept the same.

## 4. Baselines

Before LoRA fine-tuning, I evaluated two prompting baselines:

1. Base model zero-shot prompting
2. Base model few-shot prompting

The zero-shot baseline gave the model the task instructions but no examples. The few-shot baseline included several input/output demonstrations before the test input.

The goal of these baselines was to determine whether LoRA fine-tuning actually improved performance over prompting alone.

## 5. Evaluation Design

This project used two manual evaluation sets.

The first was a 30-example manual iteration evaluation set. This was used during development to compare the baselines, QLoRA v1, and QLoRA v2. Importantly, QLoRA v1 errors on this set were used for failure analysis and influenced the targeted v2 augmentation.

Because the v2 dataset was informed by the first manual evaluation set, I created a second separate 30-example final blind evaluation set after the v2 method was finalized. This final blind set was not used to design v2. Therefore, the final blind evaluation is the main final result reported for the project.

For the target cybersecurity JSON task, I evaluated:

- JSON validity rate
- Schema validity rate
- Attack type accuracy
- Severity accuracy
- Affected asset accuracy
- Numeric confidence rate

A prediction was counted as schema-valid only if it contained all required keys, used an allowed `attack_type`, used an allowed `severity`, had a string `recommended_action`, and had a numeric confidence value between 0 and 1.

## 6. Manual Iteration Evaluation Results

The manual iteration evaluation results were:

| System | JSON Valid | Schema Valid | Attack Accuracy | Severity Accuracy | Asset Accuracy | Numeric Confidence |
|---|---:|---:|---:|---:|---:|---:|
| Base zero-shot | 1.00 | 0.00 | 0.433 | 0.500 | 0.000 | 0.00 |
| Base few-shot | 1.00 | 1.00 | 0.233 | 0.600 | 0.133 | 1.00 |
| QLoRA v1 | 1.00 | 1.00 | 0.667 | 0.633 | 0.567 | 1.00 |
| QLoRA v2 | 1.00 | 1.00 | 0.900 | 1.000 | 0.800 | 1.00 |

These results showed that QLoRA v1 substantially improved task performance over prompting-only baselines. They also revealed specific v1 weaknesses that motivated the v2 iteration.

However, because the v2 improvement was informed by this evaluation set, these results are treated as iteration results rather than the main final blind result.

## 7. V1 Failure Analysis and V2 Iteration

After evaluating QLoRA v1 on the manual iteration set, I analyzed its failure cases. The main weaknesses were:

- Worm examples were often confused with DoS.
- Backdoor examples were sometimes predicted as benign.
- Critical severity was sometimes downgraded.
- Affected asset labels were sometimes semantically reasonable but not normalized to the project taxonomy.
- `network_service` and `network_host` were frequently confused.

Based on this analysis, I created a v2 dataset by adding targeted oversampled augmentation examples. These examples focused on the specific v1 failure patterns: worm, backdoor, generic attack, reconnaissance, critical severity, and affected asset normalization.

The v2 training set contained 2,614 examples. This included the original v1 training examples plus targeted oversampled examples. The v2 model continued training from the v1 LoRA adapter with a lower learning rate of 1e-4.

## 8. Final Blind Evaluation Results

After finalizing the v2 method, I created a separate final blind evaluation set of 30 examples and evaluated all four systems on it.

The final blind evaluation results were:

| System | JSON Valid | Schema Valid | Attack Accuracy | Severity Accuracy | Asset Accuracy | Numeric Confidence |
|---|---:|---:|---:|---:|---:|---:|
| Base zero-shot | 1.00 | 0.00 | 0.333 | 0.467 | 0.000 | 0.00 |
| Base few-shot | 1.00 | 1.00 | 0.300 | 0.633 | 0.033 | 1.00 |
| QLoRA v1 | 1.00 | 1.00 | 0.500 | 0.567 | 0.500 | 1.00 |
| QLoRA v2 | 1.00 | 1.00 | 0.767 | 1.000 | 0.733 | 1.00 |

On the final blind set, QLoRA v2 clearly outperformed the prompting-only baselines and QLoRA v1. Compared with QLoRA v1, QLoRA v2 improved attack accuracy from 50.0% to 76.7%, severity accuracy from 56.7% to 100.0%, and affected asset accuracy from 50.0% to 73.3%.

The zero-shot model usually produced JSON-shaped outputs, but it did not follow the project schema reliably. The few-shot baseline improved schema validity and numeric confidence, but it still performed poorly on the project-specific `attack_type` and `affected_asset` fields. QLoRA v1 improved the project-specific fields, and QLoRA v2 produced the strongest final blind performance.

## 9. Training Dynamics

Both QLoRA runs showed decreasing validation loss. QLoRA v1 ended with a final validation loss of approximately 0.118607. QLoRA v2 ended with a lower final validation loss of approximately 0.112683.

| Run | Final Validation Loss |
|---|---:|
| QLoRA v1 | 0.118607 |
| QLoRA v2 | 0.112683 |

The validation loss trend is consistent with the evaluation results: v2 improved over v1 after targeted data augmentation.

## 10. Capability Regression Evaluation

To check whether fine-tuning harmed unrelated capabilities, I created a small regression benchmark with 20 examples across two unrelated categories:

1. Reasoning/arithmetic
2. Instruction following

The base model and QLoRA v2 were evaluated on the same examples.

| System | Examples | Exact Match | Contains Expected |
|---|---:|---:|---:|
| Base model | 20 | 0.50 | 0.70 |
| QLoRA v2 | 20 | 0.75 | 0.85 |

On this small regression benchmark, QLoRA v2 did not show measurable capability degradation. It performed slightly better than the base model on exact match and contains-expected metrics. One possible reason is that the cybersecurity fine-tuning task trained the model to produce concise, structured outputs, which may have helped some instruction-following examples.

However, this regression benchmark was small, so the result should not be interpreted as proving broad general capability improvement. It only shows that no regression was observed on this limited test set.

## 11. Limitations

This project has several limitations.

First, both manual evaluation sets contained only 30 examples. This was enough to compare systems consistently, but larger evaluation sets would provide more reliable estimates.

Second, the dataset construction involved rule-based mapping from network-flow rows to natural-language descriptions and JSON fields. This made the task measurable and feasible, but the generated descriptions may not fully represent the variety of real-world security analyst reports.

Third, the capability regression benchmark was small and manually constructed. It checked two unrelated capability categories, but it was not a comprehensive general benchmark.

Fourth, the final blind evaluation set was separate from the v2 design process, but it was still manually written and relatively small. Future work should evaluate on larger and more diverse blind test sets.

## 12. Conclusion

This project showed that QLoRA fine-tuning can substantially improve a small instruction model on a narrow cybersecurity structured generation task.

The base model could often produce JSON-like text, but it did not reliably follow the required schema or project-specific taxonomy. Few-shot prompting improved formatting but did not solve the taxonomy problem. QLoRA v1 improved task accuracy over both prompted baselines. After analyzing v1 failures and applying targeted v2 augmentation, QLoRA v2 achieved the best final blind evaluation results.

The strongest final blind result was QLoRA v2, which achieved:

- 76.7% attack type accuracy
- 100.0% severity accuracy
- 73.3% affected asset accuracy
- 100.0% schema validity
- 100.0% numeric confidence rate

The small capability regression evaluation did not show degradation after fine-tuning. Overall, the results support the conclusion that LoRA/QLoRA provided a clear benefit over prompting-only baselines for this structured cybersecurity JSON generation task.
