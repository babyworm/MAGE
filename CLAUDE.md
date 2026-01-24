# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

MAGE (Multi-Agent Engine for Automated RTL Code Generation) is an open-source multi-agent LLM system that automatically generates Verilog RTL (Register Transfer Level) code from natural language specifications. Paper: https://arxiv.org/abs/2412.07822

## Development Setup

### Installation

```bash
# Create and activate conda environment
conda create -n mage python=3.11
conda activate mage

# Editable install for development
pip install -e . --config-settings editable_mode=compat

# Setup pre-commit hooks
pre-commit install
```

### External Dependencies

**Required**: Icarus Verilog 12.0 (not 11.x)
```bash
# Ubuntu
apt install -y autoconf gperf make gcc g++ bison flex
git clone https://github.com/steveicarus/iverilog.git && cd iverilog
git checkout v12-branch
sh ./autoconf.sh && ./configure && make -j4
sudo make install

# Verify version (must be v12)
iverilog -v  # First line should show: Icarus Verilog version 12.0 (stable)
```

**Required**: Verilator
```bash
sudo apt install verilator
```

**Required**: Pyverilog
```bash
pip3 install jinja2 ply
git clone https://github.com/PyHDI/Pyverilog.git && cd Pyverilog
python3 setup.py install --user
```

### API Keys Configuration

Create `key.cfg` in project root:
```
OPENAI_API_KEY='your_key_here'
ANTHROPIC_API_KEY='your_key_here'
VERTEX_SERVICE_ACCOUNT_PATH='path/to/service_account.json'
VERTEX_REGION='us-central1'
```

Alternatively, set these as environment variables.

## Running Tests

### Main Test Entry Point

```bash
python tests/test_top_agent.py
```

Edit the `args_dict` in `test_top_agent.py` to configure:
- `provider`: "anthropic", "openai", "vertex", or "vertexanthropic"
- `model`: Model name (e.g., "claude-3-5-sonnet-20241022", "gpt-4o-2024-08-06")
- `filter_instance`: RegEx filter for benchmark instances (e.g., `"^(Prob011_norgate)$"`)
- `type_benchmark`: "verilog_eval_v1" or "verilog_eval_v2"
- `path_benchmark`: Path to verilog-eval benchmark repo
- `temperature`, `top_p`, `max_token`: LLM generation parameters
- `use_golden_tb_in_mage`: Whether to use golden testbench during generation

### Other Tests

```bash
python tests/test_rtl_generator.py  # Test RTL generation only
python tests/test_single_agent.py   # Test individual agents
python tests/test_llm_chat.py       # Test LLM connectivity
```

## Code Quality

### Pre-commit Hooks

Configured in `.pre-commit-config.yaml`:
- **black**: Code formatting (line length: 88)
- **isort**: Import sorting (--profile black)
- **flake8**: Linting (max-line-length: 88, ignores: E203, W503, E501, F541)

```bash
# Run manually on all files
pre-commit run --all-files
```

## Architecture Overview

### Multi-Agent System Flow

MAGE uses a coordinated multi-agent architecture orchestrated by `TopAgent` (src/mage/agent.py):

1. **TBGenerator** (src/mage/tb_generator.py): Generates SystemVerilog testbench and interface from spec
   - Uses few-shot prompting with examples
   - Can use golden testbench if provided via `use_golden_tb_in_mage`

2. **RTLGenerator** (src/mage/rtl_generator.py): Generates RTL code from spec + testbench
   - 4-shot prompting with examples (see `RTL_4_SHOT_EXAMPLES` in prompts.py)
   - Performs syntax checking via Icarus Verilog
   - Can generate multiple candidates for selection

3. **SimReviewer** (src/mage/sim_reviewer.py): Runs simulation and reviews results
   - Executes Icarus Verilog simulation
   - Compares output against golden RTL or testbench expectations
   - Returns pass/fail status and mismatch count

4. **SimJudge** (src/mage/sim_judge.py): Analyzes simulation failures
   - Determines if testbench or RTL is at fault
   - Returns boolean: `True` if testbench needs fixing, `False` if RTL needs fixing

5. **RTLEditor** (src/mage/rtl_editor.py): Iteratively fixes RTL based on simulation failures
   - Uses simulation logs and error messages
   - Performs multiple edit iterations until passing or max retries

### Execution Flow

```
TopAgent.run()
  ├─> TBGenerator.chat() → testbench.sv, interface.sv
  ├─> RTLGenerator.chat() → rtl.sv (with syntax check)
  └─> Loop (max sim_max_retry=4):
      ├─> SimReviewer.review() → pass/fail, mismatch_cnt, sim_log
      ├─> If fail: SimJudge.chat() → tb_need_fix?
      ├─> If tb_need_fix: TBGenerator regenerates testbench
      └─> If rtl_need_fix:
          ├─> Generate rtl_max_candidates (default: 20) candidates
          ├─> Select best rtl_selected_candidates (default: 2) by mismatch count
          └─> RTLEditor.chat() iterates to fix RTL
```

### Key Configuration Classes

- **TokenCounter** (src/mage/token_counter.py): Tracks LLM token usage and costs
  - `TokenCounterCached`: Uses LLM prompt caching when available
- **Config** (src/mage/gen_config.py): Manages API keys from key.cfg or environment
- **ExperimentSetting** (src/mage/gen_config.py): Global temperature/top_p settings

### Prompts System

All LLM prompts are centralized in `src/mage/prompts.py`:
- `RTL_4_SHOT_EXAMPLES`: Few-shot examples for RTL generation
- `ORDER_PROMPT`: Specific requirements for RTL code style
- `FAILED_TRIAL_PROMPT`: Template for retry attempts

## Output Structure

Each run creates:
- `output_{run_identifier}/{benchmark}_{task_id}/`: Generated RTL, testbench, interface files
- `log_{run_identifier}/{benchmark}_{task_id}/`: Execution logs (rich and plain text versions)
- `output_{run_identifier}/record.json`: Pass rates, token usage, costs per problem

## Important Implementation Details

### Verilator Support

The `src/sim/Makefile` contains Verilator configuration for coverage analysis and simulation. Not currently used in main MAGE flow but available for custom testing.

### Benchmark Integration

- Uses verilog-eval benchmark (submodule in `verilog-eval/`)
- `benchmark_read_helper.py` provides utilities to load specs, golden testbenches, and golden RTL
- Supports both verilog_eval_v1 and verilog_eval_v2

### LLM Provider Support

- **OpenAI**: Direct API via `llama-index-llms-openai`
- **Anthropic**: Direct API via `llama-index-llms-anthropic`
- **Vertex AI**: Google Cloud with service account authentication
- **VertexAnthropic**: Anthropic models via Vertex AI (uses custom `VertexAnthropicWithCredentials` in utils.py)

### Ablation Mode

Set `agent.set_ablation(True)` to run RTL generation only without testbench generation or iterative fixing (used for research comparisons).
