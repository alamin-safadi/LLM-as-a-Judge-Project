# LLM-as-a-Judge: Reliability, Bias Analysis, and Iterative Judge Improvement

## Overview

This research project implements an experimental **LLM-as-a-Judge workflow** for comparing two candidate answers to the same question. It examines how a language-model judge makes decisions, investigates repeatability and potential biases, compares decisions with available human judgments, and uses the evidence to guide prompt-, rubric-, and decision-rule updates.

The project is organized as connected notebook modules. Each module produces structured data or analysis that can be inspected in Google Colab. The updated judge is then re-evaluated and the evidence reviewed again.

> **Research scope:** The described update process changes the judge prompt, evaluation rubric, reasoning instructions, and decision rule. It should not be described as model-weight fine-tuning unless a separate fine-tuning implementation is added.

## Research objectives

- Build a structured workflow for comparing candidate answers with an LLM judge.
- Record criterion-based evaluations and reasoning.
- Examine decision repeatability and numerical score stability.
- Investigate sensitivity to answer position and verbosity.
- Compare judge decisions with available human-reference labels.
- Examine confidence calibration against human-grounded correctness.
- Use analysis to guide prompt, rubric, and decision-rule updates.
- Re-evaluate the updated judge and determine what further validation is needed.

## Project architecture

```text
MT-Bench Test Dataset
        |
        v
Target LLM: Qwen2.5-0.5B-Instruct
        |
        v
Candidate Answers A and B
        |
        v
Judge System: Llama 3.2-1B-Instruct + Rubric
        |
        v
Evaluation + Reasoning
        |
        v
Reliability Analysis
        |
        +-----------------------------+
        |                             |
        v                             v
External Feedback                 Self-Critique
(Human reference and              Review of prior
error analysis)                   reasoning/decision
        |                             |
        +--------------+--------------+
                       v
                  Calibration
                       |
                       v
                   Judge Update
                       |
                       v
              Updated-Judge Candidate
                       |
                       v
                    Re-evaluation
                       |
                       v
             Post-Update Reliability
                       |
                       v
              Before/After Evidence
                       |
                       v
                      LOOP
                       |
             +---------+----------+
             |                    |
             v                    v
     Further investigation    Candidate retention
     or Judge Update          pending validation
```

The diagram represents the intended research workflow. Actual files and available fields depend on which notebook cells have run successfully.

## Modules and their roles

### 1. Test Dataset — MT-Bench

**Purpose:** Load and validate the questions used as evaluation inputs.

- Downloads and reads MT-Bench `question.jsonl`.
- Organizes questions and metadata in a structured table.
- Checks expected fields such as `question_id`, `category`, and `turns`.
- Provides the question source for candidate-answer generation.

**Output/use:** A structured question dataset for the next module.

### 2. Target LLM — Qwen2.5-0.5B-Instruct

**Purpose:** Generate candidate answers for the judge to compare.

- Loads `Qwen/Qwen2.5-0.5B-Instruct`.
- Generates Answer A and Answer B for each question.
- Saves outputs and pairs answers by `question_id`.

**Important design note:** The described implementation uses different generation settings: Answer A uses deterministic decoding, while Answer B uses sampling. Therefore, answer differences should not be attributed solely to model capability; generation settings are also different.

### 3. Judge System — Llama 3.2-1B-Instruct

**Purpose:** Apply a rubric to each candidate-answer pair and produce a judgment.

- Receives the question and Answers A/B.
- Uses the judge model with an evaluation rubric.
- Extracts scores and a final choice (`A`, `B`, or `TIE`) from generated text.
- Saves outputs for downstream analysis, with checkpointing where implemented.

**Output/use:** Initial judge decisions and evaluation data for the reasoning and reliability stages.

### 4. Evaluation + Reasoning

**Purpose:** Record criterion-based evaluations and the rationale behind the judge's decision.

The rubric covers:

- Correctness
- Relevance
- Clarity
- Completeness
- Instruction Following

The module builds the evaluation prompt, asks the judge to score and compare the answers, parses the response, and stores structured fields alongside raw output and reasoning.

**Output/use:** Criterion scores, reasoning, and decisions for later error analysis. A record being present does not automatically mean every expected field was generated or parsed successfully.

### 5. Initial Reliability Analysis

**Purpose:** Examine consistency and investigate potential decision biases.

The described module:

- Repeats judgments to examine exact decision agreement.
- Calculates score variability, including standard deviation and range.
- Swaps answer positions to investigate position sensitivity.
- Compares shorter and longer answers as a descriptive investigation of verbosity-related preference.
- Exports analysis tables for later use.

**Output/use:** Evidence about repeatability, score stability, and possible bias patterns.

**Interpretation note:** Repeatability is not the same as correctness. Position or verbosity patterns do not, by themselves, establish causality.

### 6. External Feedback

**Purpose:** Compare judge decisions with available human or expert reference labels.

- Loads human-reference labels such as `human_winner`.
- Normalizes labels for comparison.
- Identifies agreement and disagreement between the judge and the reference.
- Supports error analysis and comparison of rubric-based scores.
- Prepares data for calibration and later updates.

**Output/use:** Human-grounded evaluation evidence.

**Interpretation note:** The strength of this analysis depends on how human labels were produced and on evaluator agreement. The module uses provided labels as a reference; it does not independently prove every label is correct.

### 7. Self-Critique

**Purpose:** Ask the judge to review its previous decision and reasoning.

- Supplies the original question, candidate answers, and previous judgment.
- Prompts a review for possible reasoning errors, omitted criteria, and position or verbosity concerns.
- Parses the critique, revised decision, reasoning, and confidence when available.
- Compares initial and revised decisions and exports the resulting data.

**Output/use:** Additional signals about possible weaknesses in the original judgment.

**Interpretation note:** Self-critique is not independent ground truth. A detected issue or changed decision must be checked against human-reference evidence; a revision is not automatically an improvement.

### 8. Calibration

**Purpose:** Combine evidence from human feedback, self-critique, reliability, and bias analysis.

The module aligns records using `question_id` and, where data is available, examines:

- Confidence in relation to human-grounded correctness
- Confidence–accuracy gaps and calibration error
- Brier score
- High-confidence disagreements with the human reference
- Decision changes after self-critique
- Score instability and potential position/verbosity sensitivity

**Output/use:** Structured calibration evidence and update signals for Judge Update.

**Scope note:** Calibration creates evidence and signals. It does not itself change model weights or guarantee that the next judge version will improve.

### 9. Judge Update

**Purpose:** Convert analysis signals into an updated judging procedure.

The described implementation:

- Reviews available error and bias signals.
- Retains the five core criteria.
- Adds explicit Position Neutrality and Verbosity Neutrality guidance.
- Revises reasoning instructions for evaluating A and B consistently.
- Updates the prompt and score-aggregation/decision rule.
- Re-evaluates candidate answers and exports updated outputs and configuration.

**Output/use:** An updated prompt/rubric configuration and evaluation outputs for subsequent validation.

**Scope note:** This is a prompt-, rubric-, and decision-strategy update, not model-weight fine-tuning in the code described here.

### 10. Re-evaluation

**Purpose:** Run the updated judge on candidate answers and compare its decisions with the human reference.

- Uses the updated judge prompt and model.
- Parses decisions, scores, and confidence from raw output.
- May use a score-based fallback when an explicit decision is not parsed.
- Compares decisions with human-reference labels.
- Prepares data for confidence-calibration and later analysis.

**Output/use:** Re-evaluation records for post-update reliability and before/after checks.

**Interpretation note:** Keep explicit judge decisions distinguishable from score-rule fallback decisions. Parsing success and fallback behavior should be checked before interpreting downstream metrics.

### 11. Post-Update Reliability Analysis

**Purpose:** Re-check decision repeatability and score stability after the judge update.

The described module:

- Validates re-evaluation data and required fields.
- Parses raw judge outputs and audits parser status.
- Compares valid decisions with human-reference labels.
- Repeats deterministic evaluations.
- Measures exact decision repeatability and score standard deviation/range.
- Exports structured post-update reliability data.

**Output/use:** Evidence for comparison with the earlier judge.

**Implementation note:** The displayed repeated-evaluation function uses `torch.no_grad()`. Ensure `torch` has been imported in the notebook before running that function.

### 12. LOOP — Evidence Synthesis and Next-Step Selection

**Purpose:** Combine available post-update evidence and identify the next research action.

- Checks that expected re-evaluation, reliability, position, verbosity, and before/after files exist.
- Loads and checks the analysis datasets.
- Extracts before/after metrics such as exact agreement, score variability, and position consistency.
- Summarizes human-grounded evaluation, repeatability, score stability, and confidence-calibration evidence when available.
- Produces evidence tables, a directional comparison, and a loop report.
- Assigns a next-step category, such as further investigation, further evidence collection, return to Judge Update, or candidate retention for additional validation.

**Output/use:** A compact record of the evidence considered and the next action suggested by the current decision logic.

**Interpretation note:** The loop decision is a decision aid, not proof of scientific improvement. The logic should be reviewed to ensure that human-grounded accuracy, calibration, and bias evidence appropriately influence the conclusion. A retention category should still be followed by evaluation on additional or held-out data where possible.

## How the modules connect

| Stage | Main input | Main role | Used by |
|---|---|---|---|
| Test Dataset | MT-Bench question data | Load and validate questions | Target LLM |
| Target LLM | Questions | Generate candidate answers A/B | Judge System |
| Judge System | Question and A/B answers | Produce rubric-based judgments | Evaluation + Reasoning |
| Evaluation + Reasoning | Raw judge outputs | Structure scores, decisions, and rationale | Reliability and feedback modules |
| Initial Reliability Analysis | Repeated judgments and paired answers | Examine repeatability, score variability, and potential biases | Calibration and Judge Update |
| External Feedback | Human-reference labels and judge decisions | Compare judgments and identify disagreements | Calibration |
| Self-Critique | Previous decisions and reasoning | Review possible errors and biases | Calibration |
| Calibration | Human, critique, reliability, and bias evidence | Create calibration measures and update signals | Judge Update |
| Judge Update | Analysis signals | Revise prompt, rubric, reasoning, and decision strategy | New Re-evaluation |
| New Re-evaluation | Updated judge and candidate answers | Re-run evaluation and compare with human reference | Post-Update Reliability |
| Post-Update Reliability | Repeated updated-judge outputs | Re-check repeatability and score stability | NEW_LOOP / before-after analysis |
| NEW_LOOP | Before/after and post-update evidence | Summarize evidence and select a next step | Further validation or another update cycle |

## Running and inspecting the project

The implementation is organized as notebook modules. To inspect the work:

1. Open the linked Google Colab notebook.
2. Follow the notebook cells in workflow order, ensuring prerequisite cells have completed successfully before running dependent modules.
3. Inspect raw judge outputs as well as parsed CSV/JSON tables; parsing can fail even when a raw output exists.
4. Review reliability, position, verbosity, human-reference, and calibration artifacts.
5. Inspect the updated judge configuration and re-evaluation outputs.
6. Finish with the post-update reliability and `NEW_LOOP` reports to see what evidence was available and which next step the current logic selected.

**Colab notebook:** `ADD_YOUR_GOOGLE_COLAB_NOTEBOOK_LINK_HERE`

**Repository:** `ADD_YOUR_GITHUB_REPOSITORY_LINK_HERE`

Replace these placeholders with the actual links. Do not include credentials, access tokens, or private data in the README.

## Outputs and artifacts

The modules described in this project write analysis files under `results/`. Depending on which cells ran successfully, relevant artifacts can include:

- Candidate-answer outputs and paired answer data
- Initial evaluation/reasoning outputs
- Reliability and bias-analysis tables
- External-feedback and self-critique outputs
- Calibration summaries and update signals
- Updated judge configuration and re-evaluation outputs
- Post-update reliability and before/after comparison tables
- `loop_evidence.csv`
- `loop_directional_comparison.csv`
- `loop_report.csv`

These artifacts allow a supervisor to inspect the implementation and the evidence behind the analysis. File availability depends on successful execution of the relevant notebook cells.

## Important methodological considerations

- **Human-reference quality:** Document how `human_winner` labels were obtained and, where possible, report inter-annotator agreement.
- **Repeated evaluation:** Deterministic decoding supports repeatability checks for a fixed setup; it does not prove correctness or guarantee identical results across software/hardware environments.
- **Generation confound:** Candidate A and B use different generation settings in the described target-model module. Keep this explicit when interpreting their differences.
- **Parser validity:** Report missing, malformed, fallback, and explicit decisions separately. Do not treat every row as a valid judgment simply because it exists in a file.
- **Bias claims:** Descriptive position and verbosity patterns should not be presented as causal evidence without an appropriate controlled design.
- **Data leakage / overfitting:** Repeatedly adjusting prompts on the same benchmark can overfit the workflow to that benchmark. Where feasible, validate the final approach on a separate held-out dataset.
- **Improvement claims:** More repeatable decisions do not necessarily mean more accurate decisions. Interpret reliability, human-grounded correctness, calibration, and bias evidence together.
- **Model updates:** The described Judge Update changes prompts, rubric, reasoning instructions, and decision rules; it does not fine-tune model weights.

## Current research stage

The framework includes modules for candidate-answer generation, rubric-based judging, reliability and bias analysis, human-reference comparison, self-critique, calibration, prompt/rubric updates, re-evaluation, and post-update evidence synthesis. The next step is to validate end-to-end execution, audit parser and data-joining behavior, and assess the updated workflow on suitable held-out data before making broad claims about judge improvement.

## Limitations

This README describes the workflow and module roles based on the implementation outline. It does not claim that the judge is already more accurate, unbiased, or calibrated. Those claims require reproducible experiments, clearly defined metrics, reliable reference labels, and appropriate validation data.
