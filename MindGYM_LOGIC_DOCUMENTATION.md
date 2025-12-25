# MindGYM Logic Location in Data-Juicer Codebase

## Overview

MindGYM is a research work on **question synthesis for thinking-centric fine-tuning**, published in NeurIPS'25. The related logic is implemented in the Data-Juicer framework as a set of operators for:
1. **Question Generation** - Synthesizing questions from text or examples
2. **Question Optimization** - Refining and calibrating questions
3. **Difficulty Scoring** - Evaluating question difficulty using LLMs

## Core Components

### 1. Difficulty Scoring (Filter)

**Location**: `data_juicer/ops/filter/llm_difficulty_score_filter.py`

**Purpose**: Filter samples based on difficulty scores estimated by an LLM. This is a key component of MindGYM for selecting high-quality, appropriate-difficulty questions.

**Key Features**:
- Evaluates samples across multiple dimensions:
  - Linguistic complexity
  - Conceptual depth
  - Prior knowledge requirements
  - Step complexity
  - Ambiguity
- Each dimension scored 1-5 (5 = highest difficulty)
- Final score is the average of dimension scores
- Keeps samples within specified difficulty range (min_score to max_score)

**Base Class**: `data_juicer/ops/filter/llm_analysis_filter.py`

**Usage Example**:
```yaml
- llm_difficulty_score_filter:
    api_or_hf_model: 'gpt-4o'  # or 'qwen2.5-72b-instruct'
    min_score: 0.5
    max_score: 1.0
    input_keys: ['text']  # Can support multi-field data like ['query', 'analysis', 'answer']
    field_names: ['Text']
```

**Test File**: `tests/ops/filter/test_llm_difficulty_score_filter.py`

**Documentation**: `docs/operators/filter/llm_difficulty_score_filter.md`

---

### 2. Question Generation from Examples (Mapper)

**Location**: `data_juicer/ops/mapper/generate_qa_from_examples_mapper.py`

**Purpose**: Generate new question-answer pairs based on seed examples. This supports the self-synthesis approach in MindGYM.

**Key Features**:
- Takes seed QA examples in chatml format
- Uses LLM to learn patterns and generate new QA pairs
- Filters generated samples based on similarity to avoid duplication
- Uses ROUGE-L metric for similarity computation
- Configurable number of seed examples to use

**Usage Example**:
```yaml
- generate_qa_from_examples_mapper:
    hf_model: 'Qwen/Qwen2.5-7B-Instruct'
    seed_file: 'path/to/seed-examples.jsonl'  # chatml format
    example_num: 3  # Number of random seed examples to use
    similarity_threshold: 0.7  # Keep samples with similarity < threshold
    enable_vllm: false
```

**System Prompt** (Chinese):
```
请你仔细观察多个示例数据的输入和输出，按照你的理解，总结出相应规矩，然后写出一个新的【问题】和【回答】。
```

**Test File**: `tests/ops/mapper/test_generate_qa_from_examples_mapper.py`

---

### 3. Question Generation from Text (Mapper)

**Location**: `data_juicer/ops/mapper/generate_qa_from_text_mapper.py`

**Purpose**: Automatically generate QA pairs from raw text content.

**Key Features**:
- Converts unstructured text into QA pairs
- Recommended for Chinese text with models like 'alibaba-pai/pai-qwen1_5-7b-doc2qa'
- Can limit the number of QA pairs generated per text
- Supports custom output patterns for parsing

**Usage Example**:
```yaml
- generate_qa_from_text_mapper:
    hf_model: 'alibaba-pai/pai-qwen1_5-7b-doc2qa'
    max_num: 5  # Max QA pairs per text (null = unlimited)
    enable_vllm: false
```

**Test File**: `tests/ops/mapper/test_generate_qa_from_text_mapper.py`

---

### 4. Query Optimization (Mapper)

**Location**: `data_juicer/ops/mapper/optimize_query_mapper.py`

**Purpose**: Refine questions to make them more specific and detailed while maintaining answerability.

**Key Features**:
- Makes questions more detailed and specific
- Ensures the original answer can still address the optimized question
- Chinese-focused system prompt

**Base Class**: `data_juicer/ops/mapper/optimize_qa_mapper.py`

**System Prompt** (Chinese):
```
优化问答对中的【问题】，将其更加详细具体，但仍可以由原答案回答。只输出优化后的【问题】，不要输出多余内容。
```

---

### 5. Query Calibration (Mapper)

**Location**: `data_juicer/ops/mapper/calibrate_query_mapper.py`

**Purpose**: Calibrate questions based on reference text to improve accuracy and detail.

**Key Features**:
- Adjusts questions to be more detailed and accurate
- Uses reference text to inform calibration
- Maintains answerability with original answer

**Base Class**: `data_juicer/ops/mapper/calibrate_qa_mapper.py`

**System Prompt** (Chinese):
```
请根据提供的【参考信息】对问答对中的【问题】进行校准，使其更加详细、准确，且仍可以由原答案回答。只输出校准后的问题，不要输出多余内容。
```

---

## MindGYM Workflow

The typical MindGYM workflow in Data-Juicer involves:

1. **Question Synthesis**:
   - Use `generate_qa_from_text_mapper` to create initial QA pairs from text
   - OR use `generate_qa_from_examples_mapper` to generate based on seed examples

2. **Question Refinement** (Optional):
   - Use `optimize_query_mapper` to make questions more specific
   - Use `calibrate_query_mapper` for reference-based calibration

3. **Difficulty-based Selection**:
   - Use `llm_difficulty_score_filter` to select questions with appropriate difficulty
   - Filter by multiple dimensions: linguistic complexity, conceptual depth, etc.
   - Keep only samples within desired difficulty range (e.g., 0.5-1.0 for challenging questions)

4. **Quality Control**:
   - Similarity-based deduplication (in generation mappers)
   - Multi-dimensional difficulty scoring
   - LLM-based quality assessment

## Related Files

### Configuration
- `data_juicer/config/config_all.yaml` - Complete configuration reference for all operators

### Supporting Utilities
- `data_juicer/utils/model_utils.py` - Model loading and management utilities
- `data_juicer/utils/constant.py` - Constants including StatsKeys for difficulty scores

### Base Classes
- `data_juicer/ops/base_op.py` - Base operator classes (Mapper, Filter)
- `data_juicer/ops/filter/llm_analysis_filter.py` - Base LLM analysis filter

### Related Operators
- `data_juicer/ops/mapper/optimize_response_mapper.py` - Optimize answers
- `data_juicer/ops/mapper/optimize_qa_mapper.py` - Base class for QA optimization
- `data_juicer/ops/mapper/calibrate_qa_mapper.py` - Base class for QA calibration
- `data_juicer/ops/filter/llm_quality_score_filter.py` - Quality-based filtering
- `data_juicer/ops/filter/llm_task_relevance_filter.py` - Task relevance filtering

## Papers and Research

### MindGYM Paper
- **Title**: MindGYM: What Matters in Question Synthesis for Thinking-Centric Fine-Tuning?
- **Link**: https://arxiv.org/abs/2503.09499
- **Conference**: NeurIPS'25
- **Key Contribution**: A data synthesis method that enables LLMs to self-synthesize high-quality, low-variance data for efficient fine-tuning
- **Results**: 16% gain on MathVision using only 400 samples

### References in README
- `README.md` - Lines 47, 52, 158
- `README_ZH.md` - Lines 45, 50, 154

## Example Pipeline Configuration

Here's a complete example of a MindGYM-style pipeline:

```yaml
# Global Configuration
project_name: 'mindgym_question_synthesis'
dataset_path: 'input_data.jsonl'
export_path: 'output_high_quality_qa.jsonl'

# Processing Pipeline
process:
  # Step 1: Generate QA pairs from seed examples
  - generate_qa_from_examples_mapper:
      hf_model: 'Qwen/Qwen2.5-7B-Instruct'
      seed_file: 'seed_examples.jsonl'
      example_num: 3
      similarity_threshold: 0.7
      enable_vllm: true
  
  # Step 2: Optimize questions to be more detailed
  - optimize_query_mapper:
      hf_model: 'Qwen/Qwen2.5-7B-Instruct'
      enable_vllm: true
  
  # Step 3: Filter by difficulty score
  - llm_difficulty_score_filter:
      api_or_hf_model: 'qwen2.5-72b-instruct'
      min_score: 0.6  # Keep moderately difficult to very difficult
      max_score: 1.0
      input_keys: ['query', 'response']
      field_names: ['Question', 'Answer']
      enable_vllm: true
```

## Testing

All operators have comprehensive unit tests:

```bash
# Test difficulty scoring
pytest tests/ops/filter/test_llm_difficulty_score_filter.py

# Test QA generation from examples
pytest tests/ops/mapper/test_generate_qa_from_examples_mapper.py

# Test QA generation from text
pytest tests/ops/mapper/test_generate_qa_from_text_mapper.py
```

## Notes

1. **Memory Requirements**: These operators use deep neural network models and require significant GPU memory (typically 31GB+ as noted in config)

2. **Model Support**: 
   - Supports both Hugging Face models and API-based models (OpenAI, etc.)
   - vLLM acceleration available for faster inference
   - Recommended models: Qwen2.5 series, GPT-4o

3. **Language Support**: 
   - Default prompts are in Chinese
   - Can be customized for other languages via system_prompt parameters

4. **Parallel Processing**: 
   - vLLM mode limits to single process due to GPU constraints
   - Ray mode supported for distributed processing

## Summary

The MindGYM logic in Data-Juicer is implemented as a modular pipeline of operators that can:
- **Synthesize** questions from text or examples
- **Optimize** questions for better quality
- **Filter** by difficulty scores across multiple dimensions
- **Support** thinking-centric fine-tuning workflows

This design allows researchers to build custom data synthesis pipelines for creating high-quality training data with controlled difficulty levels, which is the core idea behind the MindGYM paper.
