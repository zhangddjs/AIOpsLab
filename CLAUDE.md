# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Setup & Installation
```bash
# Install Python 3.11+ and Poetry
sudo apt install python3.11 python3.11-venv python3.11-dev python3-pip
pip install poetry

# Clone repository with submodules
git clone --recurse-submodules <REPO_URL>
cd AIOpsLab

# Setup Poetry environment
poetry env use python3.11
export PATH="$HOME/.local/bin:$PATH"
poetry install
poetry self add poetry-plugin-shell
poetry shell
```

### Code Quality
```bash
# Format code with Black
poetry run black .

# Type checking with Pyright
poetry run pyright

# Run tests (tests are in the tests/ directory)
poetry run pytest tests/
```

### Running AIOpsLab
```bash
# Start CLI (human agent)
python3 cli.py

# Run GPT-4 agent
python3 clients/gpt.py

# Run remote service
SERVICE_HOST=<HOST> SERVICE_PORT=<PORT> SERVICE_WORKERS=<WORKERS> python service.py
```

## Architecture Overview

AIOpsLab is a holistic framework for evaluating autonomous AIOps agents in cloud environments. It orchestrates microservice deployments, fault injection, workload generation, and telemetry collection.

### Core Components

1. **Orchestrator** (`aiopslab/orchestrator/`)
   - Central control system managing agent-environment interaction
   - Handles problem initialization, session management, and evaluation
   - Key class: `Orchestrator` coordinates problems, agents, and evaluations

2. **Service Layer** (`aiopslab/service/`)
   - **Apps**: Microservice application interfaces (HotelReservation, SocialNetwork, etc.)
   - **Helm/Kubectl**: Kubernetes deployment and management
   - **Telemetry**: Prometheus, Loki integration for observability

3. **Problem Registry** (`aiopslab/orchestrator/problems/`)
   - Defines AIOps problems combining app, task, fault, workload, and evaluator
   - Tasks: Detection, Localization, Analysis, Mitigation
   - Each problem inherits from a task type and implements fault injection/evaluation

4. **Agent System** (`clients/`)
   - Pre-built agents: GPT, Qwen, DeepSeek, VLLM, etc.
   - Agent interface: `async def get_action(self, state: str) -> str`
   - Supports both local and remote execution

5. **Session Management**
   - `Session` class tracks agent interactions, telemetry, and results
   - Results stored in `data/results/` with unique session IDs

### Key Design Patterns

- **Application Abstraction**: All apps inherit from `Application` base class with standard deploy/delete/cleanup methods
- **Problem Composition**: Problems combine reusable components (apps, faults, workloads, evaluators)
- **Agent Registration**: Agents register with orchestrator which manages the problem-solving loop
- **Parser System**: `ResponseParser` extracts structured actions from agent responses
- **Critical Sections**: Ensures fault injection/recovery atomicity

## Configuration

Create `aiopslab/config.yml` from `config.yml.example`:
- `k8s_host`: Control plane hostname (use `kind` for local, `localhost` for on-cluster)
- `k8s_user`: Username for SSH access to control plane
- `kube_context`: Optional Kubernetes context override
- `data_dir`: Directory for storing results and telemetry

## Environment Variables

Set in `.env` file:
```bash
OPENAI_API_KEY=<key>
QWEN_API_KEY=<key>
DEEPSEEK_API_KEY=<key>
USE_WANDB=true  # Optional: Enable Weights & Biases logging
```

## Common Development Tasks

### Adding a New Agent
1. Create agent class with `async def get_action(self, state: str) -> str`
2. Register with orchestrator: `orch.register_agent(agent)`
3. See `clients/` for examples

### Adding a New Problem
1. Create problem class inheriting from task type (Detection, Localization, etc.)
2. Implement `start_workload()`, `inject_fault()`, `eval()`
3. Register in `aiopslab/orchestrator/problems/registry.py`

### Adding a New Application
1. Add metadata JSON in `aiopslab/service/metadata/`
2. Create app class extending `Application` in `aiopslab/service/apps/`
3. Include Helm chart path or kubectl deployment commands

## Key Files to Understand

- `aiopslab/orchestrator/orchestrator.py`: Main orchestration logic
- `aiopslab/session.py`: Session state management
- `aiopslab/orchestrator/parser.py`: Agent response parsing
- `aiopslab/service/apps/base.py`: Application base class
- `clients/client.py`: Base agent client implementation