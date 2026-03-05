# AGENTS.md - Agentic Coding Guidelines for Mini-SGLang

Mini-SGLang is a lightweight (~5,000 lines) high-performance LLM inference framework written in Python.

## Project Structure

```
mini-sglang/
├── python/minisgl/       # Main source code
│   ├── attention/        # Attention backends (FlashAttention, FlashInfer)
│   ├── distributed/      # Tensor parallelism utilities
│   ├── engine/           # Core inference engine
│   ├── kvcache/          # KV cache management
│   ├── layers/           # Model layers (linear, attention, rotary)
│   ├── message/          # Inter-process messaging
│   ├── models/          # Model implementations (Llama, Qwen3)
│   ├── scheduler/        # Request scheduling
│   ├── tokenizer/        # Tokenization
│   ├── kernel/           # CUDA kernel wrappers
│   └── utils/            # Utilities (logging,├── tests/                # multiprocessing)
 Test suite
│   ├── core/             # Core functionality tests
│   ├── kernel/           # Kernel tests
│   └── misc/             # Miscellaneous tests
└── docs/                 # Documentation
```

## Build, Lint, and Test Commands

### Installation

```bash
uv venv --python=3.12
source .venv/bin/activate
uv pip install -e ".[dev]"    # with dev dependencies
uv pip install -e .           # without dev dependencies
```

### Running Tests

```bash
pytest                                    # all tests
pytest --cov=minisgl --cov-report=term-missing  # with coverage
pytest tests/kernel/test_tensor.py        # single test file
pytest tests/kernel/test_tensor.py::test_function_name  # single test
pytest -k "test_name_pattern"             # pattern match
pytest --no-cov                           # faster without coverage
```

### Linting and Type Checking

```bash
ruff check .              # linter
ruff check --fix .        # auto-fix
black .                   # formatter
mypy .                    # type checker
ruff check . && black --check . && mypy .  # all checks
```

## Code Style Guidelines

### General

- **Type annotations**: Required. Use `from __future__ import annotations`
- **Line length**: Max 100 characters (in `pyproject.toml`)
- **Python version**: 3.10+

### Imports

Order: stdlib → third-party → local. Use blank lines between groups.

```python
from __future__ import annotations

from typing import TYPE_CHECKING, List

import torch
import torch.nn.functional as F
from transformers import AutoTokenizer

from minisgl.core import Batch, Req
from minisgl.utils import init_logger

if TYPE_CHECKING:
    from minisgl.engine import Engine
```

### Naming

- **Classes**: `PascalCase` (e.g., `Scheduler`, `Engine`)
- **Functions/variables**: `snake_case` (e.g., `init_logger`)
- **Constants**: `UPPER_SNAKE_CASE` (e.g., `MAX_SEQ_LEN`)
- **Private members**: Prefix with `_` (e.g., `_internal_method`)

### Type Hints

- Use `X | None` instead of `Optional[X]`
- Use `X | Y` for unions

```python
def process_batch(batch: Batch) -> ForwardOutput | None:
    ...
```

### Dataclasses

```python
from dataclasses import dataclass

@dataclass
class SamplingParams:
    temperature: float = 0.0
    top_k: int = -1
    
    @property
    def is_greedy(self) -> bool:
        return self.temperature <= 0.0 or self.top_k == 1
```

### Error Handling

- Assertions for internal invariants
- Descriptive error messages

```python
assert self.input_ids.is_cpu, "input_ids must be on CPU"
```

### Logging

```python
from minisgl.utils import init_logger

logger = init_logger(__name__)
logger.info("Loading model...")
logger.info_rank0("Only log on primary GPU rank")
```

### Testing

- Test files in `tests/`, named `test_*.py`
- Use `@call_if_main()` for standalone tests
- Use `@torch.inference_mode()` for GPU tests

```python
from minisgl.utils import call_if_main

@call_if_main()
@torch.inference_mode()
def test_something():
    # test code
    pass
```

### GPU Code

- Always use `torch.inference_mode()` or `@torch.no_grad()`
- Use explicit device placement:

```python
device = torch.device(f"cuda:{rank}")
tensor = torch.zeros(..., device=device)
```

## Key Configuration

### Environment Variables

- `LOG_LEVEL`: DEBUG, INFO, WARNING, ERROR
- `LOG_PID`: Include process ID in logs (0 or 1)
- `MINISGL_DISABLE_OVERLAP_SCHEDULING`: Disable overlap scheduling

### Dependencies

- `torch`: Core ML framework
- `transformers`: HuggingFace (4.56.0-4.57.3)
- `flashinfer-python`: FlashInfer attention
- `sgl_kernel`: Custom CUDA kernels

## Common Patterns

### New Module

1. Create in appropriate `python/minisgl/` directory
2. Add `__init__.py` with exports
3. Use `init_logger(__name__)` for logging

### New Model

1. Create in `python/minisgl/models/`
2. Inherit from `BaseLLMModel`
3. Implement `forward()` method
4. Register in model factory

### New Attention Backend

1. Create in `python/minisgl/attention/`
2. Inherit from `BaseAttnBackend`
3. Implement required abstract methods
4. Register in attention factory
