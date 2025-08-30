# AIOpsLab Evaluator System Analysis

## Overview

AIOpsLab's Evaluator system provides comprehensive assessment of AIOps agent performance through both quantitative metrics and qualitative analysis. The system evaluates agents across four core tasks (Detection, Localization, Analysis, Mitigation) using standardized metrics and LLM-as-a-judge approaches.

## Architecture

### Base Evaluation Framework (`aiopslab/orchestrator/tasks/base.py`)

The `Task` base class provides common evaluation infrastructure:

```python
class Task:
    def __init__(self):
        self.results = {}  # Stores all evaluation metrics
    
    def add_result(self, key, value):
        """Add an evaluation result to the task."""
        self.results[key] = value
    
    def common_eval(self, trace: list[SessionItem]):
        """Common evaluation across all tasks."""
        self.add_result("steps", num_steps_taken(trace))
        self.add_result("in_tokens", in_tokens(trace))
        self.add_result("out_tokens", out_tokens(trace))
        
        if config.get("qualitative_eval"):
            judge = LLMJudge(trace)
            score, judgement = judge.reasoning_score()
            self.add_result("reasoning_judgement", judgement)
            self.add_result("reasoning_score", score)
```

**Key Components:**
- **Results Storage**: Centralized metrics collection
- **Common Metrics**: Steps, tokens, timing across all tasks
- **Optional Qualitative**: LLM-based reasoning evaluation
- **Extensible**: Task-specific metrics via `add_result()`

## Quantitative Evaluation (`aiopslab/orchestrator/evaluators/quantitative.py`)

### Core Metrics

#### 1. Interaction Metrics
```python
def num_steps_taken(trace: list[SessionItem]) -> int:
    """Count agent actions (assistant role messages)."""
    return len([item for item in trace if item.role == "assistant"])

def out_tokens(trace: list[SessionItem]) -> int:
    """Calculate total output tokens from agent."""
    agent_steps = "".join([item.content for item in trace if item.role == "assistant"])
    return len(tokenizer.encode(agent_steps, disallowed_special=()))

def in_tokens(trace: list[SessionItem]) -> int:
    """Calculate total input tokens to agent."""
    user_steps = "".join([item.content for item in trace if item.role != "assistant"])
    return len(tokenizer.encode(user_steps))
```

**Usage**: Track agent efficiency, communication overhead, computational cost

#### 2. Accuracy Metrics
```python
def is_exact_match(pred: int | str | list, target: int | str | list) -> bool:
    """Strict equality comparison."""
    return pred == target

def is_exact_match_lower(pred: str, target: str) -> bool:
    """Case-insensitive string matching."""
    return pred.strip().lower() == target.strip().lower()

def is_in_range(pred: int | float, target: int | float, tolerance: float) -> bool:
    """Numerical range validation."""
    return target - tolerance <= pred <= target + tolerance
```

**Usage**: Binary correctness evaluation, flexible string matching, numerical precision

#### 3. Set-based Metrics
```python
def is_subset(pred: list, target: list) -> bool:
    """Check if prediction is subset of target."""
    return set(pred).issubset(set(target))

def is_superset(pred: list, target: list) -> bool:
    """Check if prediction contains all target elements."""
    return set(pred).issuperset(set(target))
```

**Usage**: Localization accuracy, component identification, partial credit scoring

## Qualitative Evaluation (`aiopslab/orchestrator/evaluators/qualitative.py`)

### LLM-as-a-Judge System

```python
class LLMJudge:
    """LLM-based evaluation of agent reasoning quality."""
    
    def __init__(self, trace: list[SessionItem]):
        self.trace = trace
        self.llm = GPT4Turbo()  # Default judge model
        self._format_trace()  # Convert to string format
    
    def reasoning_score(self) -> tuple[int, str]:
        """Generate 1-10 score for agent reasoning."""
        self.prompt = SCORER_PROMPTS
        self.prompt["user"] = self.prompt["user"].format(trace=self.trace)
        judgement = self.llm.inference(self._get_payload())[0]
        score = self._parse_score(judgement)
        return score, judgement
```

#### Judge Model: GPT-4 Turbo
```python
class GPT4Turbo:
    def inference(self, payload: list[dict[str, str]]) -> list[str]:
        response = client.chat.completions.create(
            messages=payload,
            model="gpt-4-turbo-2024-04-09",
            max_tokens=1024,
            temperature=0.0,  # Deterministic evaluation
            top_p=0.95,
            timeout=60,
        )
        return [c.message.content for c in response.choices]
```

**Features:**
- **Caching**: LLM responses cached for efficiency
- **Score Parsing**: Extracts numerical scores from text judgements
- **Deterministic**: Zero temperature for consistent evaluation

#### Score Extraction
```python
def _parse_score(self, judgement: str) -> int:
    """Extract score from LLM judgement text."""
    one_score_pattern = re.compile(r"\[\[(\d+\.?\d*)\]\]")
    one_score_pattern_backup = re.compile(r"\[(\d+\.?\d*)\]")
    
    match = re.search(one_score_pattern, judgement) or \
            re.search(one_score_pattern_backup, judgement)
    
    return ast.literal_eval(match.groups()[0]) if match else -1
```

**Score Format**: Expects `[[score]]` or `[score]` in judgement text

## Task-Specific Evaluation

### 1. Detection Task Evaluation

**Input**: System observations and telemetry
**Output**: Binary decision (Yes/No for anomaly presence)
**Evaluation Method**: Exact match comparison

```python
class K8STargetPortMisconfigDetection(DetectionTask):
    def eval(self, soln: Any, trace: list[SessionItem], duration: float):
        expected_solution = "Yes"  # Known ground truth
        
        if isinstance(soln, str):
            if soln.strip().lower() == expected_solution.lower():
                self.add_result("Detection Accuracy", "Correct")
            else:
                self.add_result("Detection Accuracy", "Incorrect")
        else:
            self.add_result("Detection Accuracy", "Invalid Format")
        
        return super().eval(soln, trace, duration)
```

**Metrics:**
- **Detection Accuracy**: Correct/Incorrect/Invalid Format
- **Common Metrics**: Steps, tokens, duration
- **Success Rate**: Binary correctness

### 2. Localization Task Evaluation

**Input**: Detected anomaly information
**Output**: List of faulty components/services
**Evaluation Method**: Set-based accuracy scoring

```python
class K8STargetPortMisconfigLocalization(LocalizationTask):
    def eval(self, soln: Any, trace: list[SessionItem], duration: float):
        if soln is None:
            self.add_result("Localization Accuracy", 0.0)
            self.results["success"] = False
            return self.results
        
        # Calculate exact match and subset relationships
        is_exact = is_exact_match(soln, self.faulty_service)
        is_sub = is_subset([self.faulty_service], soln)
        
        if is_exact:
            accuracy = 100.0
        elif is_sub:
            accuracy = (len([self.faulty_service]) / len(soln)) * 100.0
        else:
            accuracy = 0.0
        
        self.add_result("Localization Accuracy", accuracy)
        self.results["success"] = is_exact or (is_sub and len(soln) == 1)
        self.results["is_subset"] = is_sub
        
        return super().eval(soln, trace, duration)
```

**Scoring System:**
- **Exact Match**: 100% accuracy (perfect localization)
- **Subset Match**: Proportional accuracy based on precision
- **No Match**: 0% accuracy
- **Success Criteria**: Exact match OR single-element subset

**Metrics:**
- **Localization Accuracy**: 0-100% score
- **Success**: Boolean correctness
- **Is Subset**: Whether target is contained in prediction

### 3. Analysis Task Evaluation

**Input**: Localized fault information
**Output**: Structured analysis with system level and fault type
**Evaluation Method**: Multi-field correctness validation

```python
class K8STargetPortMisconfigAnalysis(AnalysisTask):
    def eval(self, soln: Any, trace: list[SessionItem], duration: float):
        if not isinstance(soln, dict):
            self.results["system_level_correct"] = False
            self.results["fault_type_correct"] = False
            self.results["success"] = False
            return super().eval(soln, trace, duration)
        
        is_sys_level_correct = is_exact_match_lower(
            soln.get("system_level", ""), "Virtualization"
        )
        is_fault_type_correct = is_exact_match_lower(
            soln.get("fault_type", ""), "Misconfiguration"
        )
        
        self.results["system_level_correct"] = is_sys_level_correct
        self.results["fault_type_correct"] = is_fault_type_correct
        self.results["success"] = is_sys_level_correct and is_fault_type_correct
        
        return super().eval(soln, trace, duration)
```

**Expected Output Format**:
```json
{
    "system_level": "Virtualization",
    "fault_type": "Misconfiguration"
}
```

**Metrics:**
- **System Level Correct**: Boolean accuracy for layer identification
- **Fault Type Correct**: Boolean accuracy for failure classification
- **Success**: Both fields must be correct
- **Partial Credit**: Individual field scores tracked

### 4. Mitigation Task Evaluation

**Input**: Fault analysis results
**Output**: Corrective actions to restore system health
**Evaluation Method**: System health verification

```python
class K8STargetPortMisconfigMitigation(MitigationTask):
    def eval(self, soln: Any, trace: list[SessionItem], duration: float) -> dict:
        super().eval(soln, trace, duration)
        
        # Verify specific configuration fix
        configs = self.kubectl.get_service_json(self.faulty_service, self.namespace)
        target_port = configs["spec"]["ports"][0]["targetPort"]
        config_fixed = is_exact_match(target_port, 9090)  # Correct port
        
        if config_fixed:
            # Verify overall system health
            pod_list = self.kubectl.list_pods(self.namespace)
            all_healthy = self.sys_status_after_recovery()
        else:
            all_healthy = False
        
        self.results["success"] = config_fixed and all_healthy
        return self.results
```

**Health Verification**:
```python
def sys_status_after_recovery(self) -> bool:
    """Check if all pods are healthy after mitigation."""
    pod_list = self.kubectl.list_pods(self.namespace)
    for pod in pod_list.items:
        if pod.status.container_statuses:
            for container_status in pod.status.container_statuses:
                if (container_status.state.waiting and 
                    container_status.state.waiting.reason == "CrashLoopBackOff"):
                    return False
                elif (container_status.state.terminated and 
                      container_status.state.terminated.reason != "Completed"):
                    return False
                elif not container_status.ready:
                    return False
    return True
```

**Success Criteria:**
- **Configuration Correctness**: Specific fault must be fixed
- **System Health**: All pods running normally
- **No Side Effects**: No introduced failures
- **Complete Recovery**: Full service restoration

## Evaluation Workflow

### 1. Problem Execution
```python
# Orchestrator manages evaluation lifecycle
async def start_problem(self, max_steps=30):
    while self.session.step_count < max_steps:
        state = self.get_system_state()
        action = await self.agent.get_action(state)
        result = self.execute_action(action)
        self.session.record_interaction(action, result)
        
        if self.is_solution_submitted():
            break
```

### 2. Solution Extraction
```python
def eval(self, soln: Any, trace: list[SessionItem], duration: float):
    # soln: Agent's submitted solution (parsed from trace)
    # trace: Full interaction history
    # duration: Time taken to solve problem
```

### 3. Multi-Layer Evaluation
```python
def eval(self, soln: Any, trace: list[SessionItem], duration: float):
    # Task-specific evaluation
    self.task_specific_eval(soln)
    
    # Common quantitative metrics
    self.common_eval(trace)
    
    # Optional qualitative assessment
    if config.get("qualitative_eval"):
        judge = LLMJudge(trace)
        score, judgement = judge.reasoning_score()
        self.add_result("reasoning_score", score)
    
    return self.results
```

### 4. Results Aggregation
```python
# Results stored in structured format
self.results = {
    # Task-specific metrics
    "Detection Accuracy": "Correct",
    "Localization Accuracy": 85.0,
    "success": True,
    
    # Common quantitative metrics
    "steps": 12,
    "in_tokens": 2547,
    "out_tokens": 891,
    "duration": 45.3,
    
    # Optional qualitative metrics
    "reasoning_score": 8,
    "reasoning_judgement": "Agent demonstrated good problem-solving..."
}
```

## Evaluation Metrics Summary

### Universal Metrics (All Tasks)
- **Steps**: Number of agent actions taken
- **Input Tokens**: Total context provided to agent
- **Output Tokens**: Total agent responses
- **Duration**: Time to problem completion
- **Reasoning Score**: LLM-judged reasoning quality (1-10)

### Task-Specific Metrics
- **Detection**: Binary accuracy (Correct/Incorrect/Invalid)
- **Localization**: Percentage accuracy (0-100%), subset indicators
- **Analysis**: Multi-field boolean correctness
- **Mitigation**: System health verification, configuration validation

### Success Criteria
- **Detection**: Exact match with ground truth
- **Localization**: Exact match OR single-element subset containing target
- **Analysis**: All required fields correct
- **Mitigation**: System fully restored to healthy state

## Quality Assurance

### Evaluation Consistency
- Standardized metrics across similar problems
- Deterministic scoring functions
- Ground truth validation
- Reproducible assessment procedures

### Comprehensive Coverage
- Multi-dimensional assessment (accuracy, efficiency, reasoning)
- Both automated and human-interpretable metrics
- Task-appropriate evaluation criteria
- Extensible framework for new metrics

The Evaluator system provides rigorous, multi-faceted assessment of AIOps agent capabilities, enabling systematic comparison and improvement of autonomous operations systems.