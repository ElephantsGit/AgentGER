# 📊AgentGER
🏠 *Current Version: v1.0*

This repository contains the code to use AgentGER from the paper [AgentGER: Toward a Human-Aligned Generation–Evaluation–Refinement Paradigm for Scientific Figure Understanding].

*🤗 This codebase is released as Version v1.0. We are dedicated to its continuous improvement. If you have any questions or suggestions, you are welcome to open an issue or submit a pull request for new features or bug fixes.*
## 👋 Introduction


Figure-to-text is a key task for assessing models’ scientific figure understanding capabilities. Existing approaches face two main challenges: the high cost of constructing high-quality scientific data and the lack of fine-grained, interpretable evaluation aligned with human experts. To address these challenges, we propose AgentGER, a generation–evaluation–refinement framework in which GenModel generates summaries with hierarchical quality levels, EvaModel evaluates summaries across five human-aligned dimensions by first producing dimension-wise Chain-of-Evaluation rationales and then assigning the corresponding scores, and RefModel uses this feedback to improve weak dimensions while preserving correct content. We further construct FigGER, comprising 11,000 summaries and 55,000 five-dimensional scoring labels with reasoning chains through human–machine collaborative annotation. Experiments show that AgentGER outperforms strong baselines, surpasses Gemini-3-Pro on the evaluation benchmark, and achieves performance approaching human experts in both evaluation and refinement.
<p align="center">

  <img src="Introduction.png"  width="70%"/>

</p> 

## ⚙ System Architecture
![alt text](<System Architecture1.png>)
## 📄 Five-Dimensional Scoring Criteria
| Dimension                  | Definition                 | 0 points               | 1 point              | 2 points             |
| -------------------------- | -------------------------- | ---------------------- | -------------------- | -------------------- |
| **Faithfulness**<br> | Whether the summary accurately reflects the facts in the chart |Severe deviation or errors|Generally accurate with biases|Fully faithful to the chart|
| **Completeness**<br> |Whether core elements and main trends are covered|Omission of important content|Covers main info with omissions|Fully covers all key information|
| **Conciseness**<br>  |Whether the expression is concise and high in information density|Redundant or overly brief|Basically concise (optimizable)|Concise and efficient expression|
| **Logicality**<br>   |Whether the logic is coherent and causal relationships are clear|Confused or self-contradictory|Basically coherent with jumps|Clear and consistent logic|
| **Analysis**<br>    |Whether professional terminology is used and analysis depth is achieved|Lack of professionalism|Insufficient professional expression|Accurate terminology and in-depth analysis|

**Scoring Rules**：
- Total Score = Σ Dimension Score

**Validation Rules**：
- Total score of the five dimensions ≥ 8
- **Faithfulness must = 2**
- No dimension score can be 0
- Maximum 3 retries; discard if failed after retries
## 📊 Dataset Details
![alt text](dataset.png)
## 🚀 Quick Start
### Environment Requirements
Python >= 3.8
CUDA >= 11.8
GPU Memory >= 24GB (32GB+ recommended; 30B model requires 80GB+)
### Install Dependencies
```bash
# Method 1: Use installation script
bash install.sh
# Method 2: Use pip
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu118
pip install -r requirements.txt
```
### Dataset Download
For preview and reproducibility, we currently release 100 samples from the training set and 100 samples from the test set. The complete dataset will be made publicly available after the paper is accepted.
- [100 training samples](./data/output/training_data_l2.json)
- [100 test samples](./data/output/test_samples100.jsonl)
- [Training figures](./data/samples_images)
- [Test figures](./data/check_images)
### Model Preparation
Place the Qwen3-VL-8B-Instruct or Qwen3-VL-30B model in the project root directory
```bash
hf download Qwen/Qwen3-VL-8B-Instruct
```
```bash
project_root/
├── Qwen3-VL-8B-Instruct/   # Base model (8B version)
│   ├── config.json
│   ├── model.safetensors
│   └── ...
├── Qwen3-VL-30B/            # Base model (30B version, optional)
│   └── ...
├── lora_weights/            # LoRA fine-tuning weights
│   ├── l-1/                 # EvaModel: evaluation only
│   ├── l-2/                 # Basic RefModel: evaluation + refinement
│   └── l-3-distill          # Full RefModel: refinement + KD + ER
└── ...
```
## 🚥 Framework Usage
The overall workflow of the framework is divided into three stages: batch summary generation, single-sample scoring with refined summary generation, and re-scoring validation. The usage of scripts for each stage is as follows:
### Workflow Step 1: Batch Summary Generation
```bash
python main.py feature1 \
    --image_folder ./data/images \
    --low_ratio 0.3 \
    --medium_ratio 0.3 \
    --high_ratio 0.4 \
    --output ./data/output/summaries.jsonl \
    --gpu 0
```
* `--image_folder` Path to the image folder 
* `--low_ratio` Ratio of low-quality summaries (default 0.3)
* `--medium_ratio` Ratio of medium-quality summaries (default 0.3)
* `--high_ratio` Ratio of high-quality summaries (default 0.4)  

🗄️ After running the feature1 branch of [main.py](main.py), summaries of low, medium, and high quality (per the set ratios) will be generated for all images in the image_folder, and results are saved to ./data/output/summaries.jsonl.
### Workflow Step 2: Evaluation and Refinement
```bash
python main.py feature2 \
    --image ./data/sample.png \
    --summary "Original summary content" \
    --lora_path ./lora_weights/l-2 \
    --output ./output/result.jsonl \
    --gpu 0
```
🗄️ After running the feature2 branch of [main.py](main.py), the existing summary of sample.png is scored, a refined summary is generated, and results are saved to ./output/result.jsonl.You can also use l-3-distill for the lora_path, which delivers better performance.
### Workflow Step 3: Refinement Validation
```bash
python main.py feature3 \
    --image ./data/sample.png \
    --summary "Summary content to be scored" \
    --lora_path ./lora_weights/l-1 \
    --output ./output/scores.jsonl \
    --gpu 0
```
🗄️ After running the feature3 branch of [main.py](main.py), the refined summary of sample.png is scored, and results are saved to ./output/scores.jsonl.
### Full Pipeline: Generate Summary -> Score & Refine -> Quality Validation
```bash
python main.py pipeline1 \
    --image_folder ./data/images \
    --output ./data/output/dataset_result.jsonl \
    --lora_path_f2 ./lora_weights/l-2 \
    --lora_path_f3 ./lora_weights/l-1 \
    --max_retries 3 \
    --gpu 0
```
* `--max_retries`：Maximum number of retries; discard if exceeding 3 times
 
🗄️ After running the pipeline1 branch of [main.py](main.py), summaries are generated for all images in the images folder, followed by scoring and refined summary generation, and finally re-scoring of the refined summaries. You can also use l-3-distill for the lora_path_f2, which delivers better performance.

## 🛴 API Usage
In addition to deploying the model locally, the framework functions can also be implemented by calling the API.
### Directory Structure

The directory structure of API scripts is as follows. Please set the API key, endpoint, and model name in `config.py`.
```bash
api_pipeline/
├── __init__.py           # Package initialization
├── config.py             # API configuration (key, endpoint, model name)
├── api_model.py          # API calling module (supports multimodal input)
├── feature2_evaluate.py  # API version of Feature 2 (Scoring + Refine Summary)
├── feature3_score.py     # API version of Feature 3 (Scoring Only)
├── pipeline.py           # API version of pipeline implementation
└── main.py               # Command-line entry
```
### Script Usage

```bash
# Score existing summaries, generate refined versions, and validate refinement quality via re-scoring:
python api_pipeline/main.py pipeline1 --input ./data/output/dataset_step1.jsonl --output ./data/output/dataset_api.jsonl
```

```bash
# Refine the summary of a single image
python api_pipeline/main.py pipeline2 --image ./data/sample.png --summary "Original summary"
```
```bash
# Directly score a single image's summary
python api_pipeline/main.py pipeline3 --image ./data/sample.png --summary "Summary to be scored"
```
## 🚧 LoRA Fine-Tuning
AgentGER follows a two-stage curriculum learning strategy. In Training Stage 1, EvaModel learns five-dimensional evaluation and dimension-wise Chain-of-Evaluation rationales. In Training Stage 2, RefModel learns evaluation-guided refinement while preserving EvaModel's evaluation capability through knowledge distillation and experience replay.
### Scheme Description
| Scheme      | Output Content                       | Applicable Scenario          |Paper role              |Training Method             |
|-------------| -------------------------------------|------------------------------|------------------------|----------------------------|
| l-1         |Scores + CoE rationales               |Scoring only                  |EvaModel / Stage 1      |Standard Training           |
| l-2         |Scores + CoE + improved summary       |Scoring + Refinement          |Basic RefModel ablation |Standard Training           |
| l-3-distill |Scores + CoE + improved summary       |Enhanced Scoring + Refinement |Full RefModel / Stage 2 |Refinement + KD + ER        |
#### Knowledge Distillation Description：
* Teacher Model: l-1 (frozen EvaModel)
* Student Model: l-3-distill (Model with both scoring and refinement capabilities)
* Knowledge distillation and experience replay mitigate catastrophic forgetting and help preserve EvaModel's evaluation capability during refinement training.After training l-1 as EvaModel, use a frozen copy of l-1 as the teacher and initialize RefModel from l-1. The full RefModel is then trained on evaluation and refinement data using refinement, knowledge-distillation, and experience-replay objectives.
 
🗄️ Perform LoRA fine-tuning on Qwen3-VL-8B-Instruct: fine-tune with a scoring-only dataset to obtain `l-1`, and fine-tune with a dataset containing scores and refined summaries to obtain `l-2`. 
### Training Steps
#### Data Preparation
```bash
python training/data_format.py \
    --input data/output/dataset.jsonl \
    --generate-both
``` 
* `--dataset.jsonl`is the original dataset  
 
 🗄️ Generate two scripts (training_data_l1.json for l-1 scoring-only training data, training_data_l2.json for l-2 scoring + refinement training data) from the original dataset `data/output/dataset.jsonl`.    
 
**Data Format** 
```json
[
  {
    "image": "data/images/1.png",
    "conversations": [
      {
        "role": "user",
        "content": "Evaluation prompt + Original summary"
      },
      {
        "role": "assistant",
        "content": "<evaluation>{...}</evaluation>[<modification>{...}</modification>]"
      }
    ]
  }
]
```
#### Training Stage 1 — EvaModel
```bash
# Scoring only (l-1, Training Stage 1)
python main.py train \
    --model_path ./Qwen3-VL-8B-Instruct \
    --data_path ./data/output/training_data_l1.json \
    --output_dir ./lora_weights/l-1 \
    --scheme l-1 \
    --lora_r 64 \
    --lora_alpha 128 \
    --learning_rate 2e-4 \
    --num_epochs 3 \
    --batch_size 1 \
    --gradient_accumulation_steps 8
```
* `--lora_r`LoRA rank
* `--lora_alpha`LoRA scaling factor  

 🗄️ Use Qwen3-VL-8B-Instruct as the base model and fine-tune with `training_data_l1.json` for scoring to obtain ./lora_weights/l-1.
```bash
# Scoring + Refinement (l-2, Optional ablation)
python main.py train \
    --model_path ./Qwen3-VL-8B-Instruct \
    --data_path ./data/output/training_data_l2.json \
    --output_dir ./lora_weights/l-2 \
    --scheme l-2 \
    --lora_r 64 \
    --lora_alpha 128 \
    --learning_rate 2e-4 \
    --num_epochs 3 \
    --batch_size 1 \
    --gradient_accumulation_steps 8
```
 🗄️ Use Qwen3-VL-8B-Instruct as the base model and fine-tune with training_data_l2.json for scoring and refinement to obtain ./lora_weights/l-2.
#### Training Stage 2 — Full RefModel
```bash
# Scoring + Refinement (l-3-distill, Training Stage 2)
python training/train_lora_distill.py \
    --base_model_path ./Qwen3-VL-8B-Instruct \
    --teacher_lora_path ./lora_weights/l-1 \
    --score_data_path ./data/output/training_data_l1.json \
    --refine_data_path ./data/output/training_data_l2.json \
    --output_dir ./lora_weights/l-3-distill \
    --lora_r 64 \
    --lora_alpha 128 \
    --learning_rate 2e-4 \
    --num_epochs 3 \
    --batch_size 1 \
    --gradient_accumulation_steps 8 \
    --distill_beta 0.5 \
    --replay_gamma 0.3 \
    --temperature 2.0 \
    --score_ratio 0.3
```
* `--distill_beta`Distillation loss weight
* `--replay_gamma`Experience replay loss weight
* `--temperature`Distillation temperature
* `--score_ratio`Ratio of scoring data in mixed data (default 0.3, i.e., 30% scoring data + 70% refinement data)  
  
🗄️ Use l-2 as the teacher model and perform knowledge distillation with a mixed dataset (scoring dataset + scoring + refinement summary dataset) to obtain `l-3-distill`, which maintains both high-quality scoring and refinement capabilities.
## 🏆 Main Results
| Method | PC (↑) | SC (↑) | MAE (↓) | MSE (↓) |
|---|---:|---:|---:|---:|
| **Reference-based Methods** |  |  |  |  |
| **Human** | **0.758** | **0.801** | **0.096** | **0.065** |
| BLEU | 0.244 | 0.277 | 0.123 | 0.023 |
| ROUGE | 0.367 | 0.323 | 0.244 | 0.084 |
| BERTScore | 0.363 | 0.225 | 0.728 | 0.531 |
| CIDEr | 0.105 | -0.011 | 0.367 | 0.554 |
| **Reference-free Methods** |  |  |  |  |
| InternVL2.5-8B | 0.391 | 0.364 | 0.227 | 0.129 |
| MiniCPM-V2.6 | 0.475 | 0.345 | 0.191 | 0.092 |
| GPT-5-Mini | 0.469 | 0.425 | 0.385 | 0.237 |
| Gemini-2.5-Flash | 0.505 | 0.462 | 0.396 | 0.302 |
| Qwen2.5-VL-7B | 0.533 | 0.542 | 0.213 | 0.132 |
| Qwen3-VL-8B | 0.595 | 0.609 | 0.126 | 0.081 |
| Gemini-3-Pro | 0.639 | 0.629 | 0.394 | 0.269 |
| **Trained Methods** |  |  |  |  |
| ChartInstruct | 0.421 | 0.430 | 0.187 | 0.113 |
| UniChart | 0.448 | 0.433 | 0.185 | 0.104 |
| Qwen3-VL-8B | 0.643 | 0.639 | 0.135 | 0.109 |
| AgentGER w/o CoE | 0.651 | 0.642 | 0.171 | 0.124 |
| AgentGER w/o KD | 0.686 | 0.668 | 0.143 | 0.097 |
| AgentGER w/o ER | 0.697 | 0.644 | 0.124 | 0.066 |
| **AgentGER (8B)** | **0.747** | **0.776** | **0.085** | **0.057** |

**AgentGER achieves the best performance across all metrics and approaches human-level agreement**.  
AgentGER obtains the highest PC of 0.747 and SC of 0.776, while also achieving the lowest MAE and MSE among all methods. It outperforms both general MLLMs and trained models, showing the effectiveness of human-aligned multi-dimensional evaluation. The ablation results further show that removing knowledge distillation (KD) or experience replay (ER) degrades performance, confirming that KD and ER help preserve evaluation consistency and stabilize refinement-oriented training.


## 📜 License
Our original data contributions are distributed under the MIT license.

