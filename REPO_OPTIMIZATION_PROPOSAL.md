# Proposal: Whole Repository Optimization for OpenEvolve

## Issue Summary

**GitHub Issue #413**: OpenEvolve currently only supports optimizing regions in a single file using `EVOLVE-BLOCK` markers. Users want the ability to optimize entire code repositories where complex algorithms are implemented as libraries across multiple files.

## Current Architecture

### How Evolution Works Today

1. **Single File Input**: `run_evolution()` accepts a single file path or code string
2. **EVOLVE-BLOCK Markers**: Code inside `# EVOLVE-BLOCK-START` / `# EVOLVE-BLOCK-END` markers is evolved
3. **Diff-based Evolution**: LLM generates SEARCH/REPLACE diffs for the marked regions
4. **Single-file Evaluation**: The evaluator tests the evolved program file

### Key Files Involved

| File | Role |
|------|------|
| `openevolve/api.py` | High-level API (`run_evolution`, `evolve_function`) |
| `openevolve/utils/code_utils.py` | `parse_evolve_blocks()`, `apply_diff()` |
| `openevolve/prompt/sampler.py` | Builds prompts for LLM |
| `openevolve/controller.py` | Orchestrates evolution loop |
| `openevolve/iteration.py` | Worker process for each iteration |

### Current Data Model Limitations (From Codebase Analysis)

```python
# Current Program dataclass (database.py, lines 43-77)
@dataclass
class Program:
    id: str
    code: str  # ❌ Single string - NOT multi-file
    changes_description: str = ""
    language: str = "python"
    parent_id: Optional[str] = None
    metrics: Dict[str, float] = field(default_factory=dict)
    # ... other fields
```

**Key limitation**: The `Program` class stores code as a **single string**. There is NO multi-file support built into the data model.

### Current Evaluator Flow

```python
# Current evaluator.py (lines 156-159) - Single temp file
with tempfile.NamedTemporaryFile(suffix=self.program_suffix, delete=False) as temp_file:
    temp_file.write(program_code.encode("utf-8"))
    temp_file_path = temp_file.name
```

### EVOLVE-BLOCK Markers (Not Actively Used)

```python
# code_utils.py - parse_evolve_blocks() exists but is NOT used in iteration.py
# The system uses diff-based evolution (SEARCH/REPLACE) instead
def parse_evolve_blocks(code: str) -> List[Tuple[int, int, str]]:
    """Parse evolve blocks - returns (start_line, end_line, block_content)"""
    # ... implementation
```

**Important finding**: EVOLVE-BLOCK markers exist but are **not actively used** in the current iteration logic - the system uses LLM-generated SEARCH/REPLACE diffs instead.

---

## Proposed Solution

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Repository Optimizer                          │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐ │
│  │ Repo Scanner │  │ File Graph   │  │ Multi-File Sampler    │ │
│  │ (find files) │  │ (deps/imports)│  │ (builds prompts)      │ │
│  └──────────────┘  └──────────────┘  └───────────────────────┘ │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │            Repository-Level Evolution Loop                │ │
│  │  1. Load multiple files                                   │ │
│  │  2. Present ALL files to LLM (context)                    │ │
│  │  3. Allow modifications to any file                       │ │
│  │  4. Write evolved files back                              │ │
│  │  5. Run evaluator on full repo                            │ │
│  └──────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Core Changes Required

#### Data Model Changes

The `Program` dataclass needs to be extended to support multi-file repositories:

```python
# Proposed changes to database.py
@dataclass
class Program:
    # Existing fields...
    id: str
    code: str  # Keep for backward compatibility with single-file mode
    
    # New fields for multi-file support
    files: Optional[Dict[str, str]] = None  # filename -> content map
    repository_path: Optional[str] = None  # Original repo path for reference
    changed_files: List[str] = field(default_factory=list)  # Files modified in this iteration
    
    # Existing fields continue...
    language: str = "python"
    parent_id: Optional[str] = None
    metrics: Dict[str, float] = field(default_factory=dict)
    
    def get_primary_code(self) -> str:
        """Get code for evaluation - prefer files dict, fallback to code string"""
        if self.files:
            return self._combine_files()
        return self.code
    
    def _combine_files(self) -> str:
        """Combine multiple files for single-file evaluation mode"""
        combined = []
        for path in sorted(self.files.keys()):
            combined.append(f"# FILE: {path}")
            combined.append(self.files[path])
            combined.append("")
        return "\n".join(combined)
```

#### 1. New API: `evolve_repository()`

```python
def evolve_repository(
    repository_path: Union[str, Path],
    evaluator: Union[str, Path, Callable],
    config: Optional[Config] = None,
    iterations: Optional[int] = None,
    file_patterns: List[str] = ["*.py"],  # Files to include
    exclude_patterns: List[str] = ["test_*.py", "__pycache__"],  # Exclude
    **kwargs
) -> EvolutionResult:
    """
    Evolve an entire repository.
    
    Args:
        repository_path: Path to the repository root
        evaluator: Evaluator that tests the repository
        config: Configuration object
        iterations: Number of iterations
        file_patterns: Glob patterns for files to include
        exclude_patterns: Glob patterns to exclude
    """
```

#### 2. Repository Scanner Module

New file: `openevolve/utils/repository_scanner.py`

```python
from pathlib import Path
from typing import List, Dict, Set, Tuple
import ast

class RepositoryScanner:
    """Scans and analyzes repository structure"""
    
    def __init__(self, root_path: Path):
        self.root_path = root_path
        self.file_contents: Dict[str, str] = {}
        self.import_graph: Dict[str, Set[str]] = {}
        self.language_by_file: Dict[str, str] = {}
    
    def scan(
        self, 
        include_patterns: List[str] = ["*.py"],
        exclude_patterns: List[str] = []
    ) -> Dict[str, str]:
        """Scan repository and return file path -> content map"""
        # Use glob to find files
        # Parse imports using AST
        # Build dependency graph
        return self.file_contents
    
    def get_file_dependencies(self, file_path: str) -> Set[str]:
        """Get files that the given file depends on"""
        return self.import_graph.get(file_path, set())
    
    def get_file_dependents(self, file_path: str) -> Set[str]:
        """Get files that depend on the given file"""
        # Reverse lookup
        pass
```

#### 3. Multi-File Prompt Sampler

Extend `PromptSampler` to handle multi-file context:

```python
class PromptSampler:
    # ... existing code ...
    
    def build_repository_prompt(
        self,
        file_contents: Dict[str, str],  # filename -> content
        changed_files: List[str],        # Files modified in this iteration
        parent_programs: Dict[str, str], # Previous version of changed files
        program_metrics: Dict[str, float] = {},
        **kwargs
    ) -> Dict[str, str]:
        """
        Build prompt for repository-level evolution.
        
        Presents the LLM with:
        - Full context of all files (or key files based on dependency graph)
        - Explicit file boundaries using comments
        - Clear indication of what was changed
        - Import dependency information
        """
        
        # Format files with clear boundaries
        files_section = self._format_repository_files(
            file_contents, 
            changed_files
        )
        
        # Add dependency information
        deps_section = self._format_dependency_graph(file_contents)
        
        # Build the prompt
        user_message = REPOSITORY_TEMPLATE.format(
            files_context=files_section,
            dependency_info=deps_section,
            changed_files_summary=self._format_changed_files(
                changed_files, 
                parent_programs
            ),
            metrics=format_metrics(program_metrics),
            # ... other fields
        )
        
        return {
            "system": REPOSITORY_SYSTEM_MESSAGE,
            "user": user_message,
        }
    
    def _format_repository_files(
        self, 
        file_contents: Dict[str, str],
        changed_files: List[str]
    ) -> str:
        """Format files with clear boundaries"""
        sections = []
        
        for filepath in sorted(file_contents.keys()):
            content = file_contents[filepath]
            is_changed = filepath in changed_files
            
            marker = "📝 MODIFIED" if is_changed else "📄"
            sections.append(
                f"{marker} File: `{filepath}`\n"
                f"```python\n{content}\n```\n"
            )
        
        return "\n".join(sections)
    
    def _format_dependency_graph(
        self, 
        file_contents: Dict[str, str]
    ) -> str:
        """Format import/dependency information"""
        # Use repository scanner to get dependencies
        # This helps LLM understand what might break
        deps = []
        for filepath, content in file_contents.items():
            imports = self._extract_imports(content)
            if imports:
                deps.append(f"- `{filepath}` imports: {', '.join(imports)}")
        
        return "## Dependencies\n" + "\n".join(deps) if deps else ""
```

#### Repository Prompt Template

```python
REPOSITORY_SYSTEM_MESSAGE = """You are an expert software engineer specializing in repository-level code optimization.

Your task is to improve multiple files in a repository to achieve better performance,
correctness, or other optimization goals while maintaining all existing functionality.

IMPORTANT:
1. You may modify ANY file in the repository
2. Changes must maintain import compatibility
3. All tests must continue to pass
4. Consider how changes affect other files in the repo
"""

REPOSITORY_USER_TEMPLATE = """## Repository Context

### Files in Repository
{files_context}

### Import Dependencies
{dependency_info}

### Changed Files (from previous iteration)
{changed_files_summary}

### Current Performance Metrics
{metrics}

## Task
Suggest improvements using SEARCH/REPLACE diff format with explicit file targets:

```
<<<<<<< SEARCH:src/utils/helper.py
def old_function():
=======
def new_function():
    # Improved implementation
>>>>>>> REPLACE:src/utils/helper.py
```

Important:
- ALWAYS specify the file path after SEARCH/REPLACE/REPLACE markers
- If you modify imports, ensure they're compatible with all files
- Write complete, syntactically correct code
"""
```

#### 4. Diff Format for Multi-File

Extend the diff format to specify target files:

```python
# New format:
<<<<<<< SEARCH:openevolve/utils/foo.py
def old_function():
    pass
=======
def new_function():
    # Improved implementation
    pass
>>>>>>> REPLACE:openevolve/utils/foo.py
```

#### 5. Apply Diffs to Multiple Files

```python
def apply_diff_to_repository(
    base_path: Path,
    diff_text: str
) -> Dict[str, bool]:
    """
    Apply diff to multiple files in a repository.
    
    Extended diff format supports file specification:
    <<<<<<< SEARCH:openevolve/utils/foo.py
    def old_function():
    =======
    def new_function():
    >>>>>>> REPLACE:openevolve/utils/foo.py
    
    Returns: Dict[filename, success]
    """
    # Parse file-specific diffs
    # Apply to each target file
    # Validate no broken imports
```

#### 6. Multi-File Evaluator Changes

The evaluator needs significant changes to support repository-level evaluation:

```python
# Proposed changes to evaluator.py
class Evaluator:
    async def evaluate_repository(
        self,
        repo_path: Path,
        program: Program
    ) -> Dict[str, float]:
        """
        Evaluate a repository-level program.
        
        Writes all files from program.files to temp directory,
        then runs evaluation.
        """
        import tempfile
        import shutil
        
        # Create temp directory with all files
        with tempfile.TemporaryDirectory() as temp_dir:
            temp_path = Path(temp_dir)
            
            # Write all files
            if program.files:
                for filename, content in program.files.items():
                    file_path = temp_path / filename
                    file_path.parent.mkdir(parents=True, exist_ok=True)
                    file_path.write_text(content)
            
            # Run evaluation on the temp directory
            if self.config.cascade_evaluation:
                return await self._cascade_evaluate_repo(temp_path)
            else:
                return await self._direct_evaluate_repo(temp_path)

    async def _direct_evaluate_repo(self, repo_path: Path) -> Dict[str, float]:
        """Evaluate repository using the evaluation function"""
        
        # The evaluator module can now receive repo_path instead of file_path
        async def run_evaluation():
            loop = asyncio.get_event_loop()
            return await loop.run_in_executor(
                None, 
                self.evaluate_function, 
                repo_path  # Changed from file_path to repo_path!
            )
        
        result = await asyncio.wait_for(run_evaluation(), timeout=self.config.timeout)
        return self._process_evaluation_result(result).metrics
```

#### Repository Evaluator Interface

```python
# Example repository evaluator
def evaluate_repository(repo_path: Path) -> Dict[str, Any]:
    """
    Repository-level evaluator receives Path to repository root.
    
    Args:
        repo_path: Path to the temporary repository directory
        
    Returns:
        Dict with metrics (e.g., {"score": 0.95, "tests_passed": 42})
    """
    import subprocess
    
    # Run tests in the repository
    result = subprocess.run(
        ["pytest", str(repo_path), "-v", "--tb=short"],
        capture_output=True,
        text=True,
        timeout=300
    )
    
    # Parse test results
    passed = result.stdout.count(" PASSED")
    failed = result.stdout.count(" FAILED")
    
    return {
        "score": passed / max(passed + failed, 1),
        "tests_passed": passed,
        "tests_failed": failed,
        "compiles": result.returncode == 0 or "error" not in result.stderr.lower(),
    }
```

#### Repository Evaluator Interface

```python
# Example repository evaluator
def evaluate_repository(repo_path: Path) -> Dict[str, Any]:
    """
    Repository-level evaluator receives Path to repository root.
    
    Args:
        repo_path: Path to the temporary repository directory
        
    Returns:
        Dict with metrics (e.g., {"score": 0.95, "tests_passed": 42})
    """
    import subprocess
    
    # Run tests in the repository
    result = subprocess.run(
        ["pytest", str(repo_path), "-v", "--tb=short"],
        capture_output=True,
        text=True,
        timeout=300
    )
    
    # Parse test results
    passed = result.stdout.count(" PASSED")
    failed = result.stdout.count(" FAILED")
    
    return {
        "score": passed / max(passed + failed, 1),
        "tests_passed": passed,
        "tests_failed": failed,
        "compiles": result.returncode == 0 or "error" not in result.stderr.lower(),
    }
```

---

---

## Implementation Phases

### Phase 1: Foundation (Week 1)
- [ ] Create `openevolve/utils/repository_scanner.py`
- [ ] Implement `RepositoryScanner.scan()` method
- [ ] Add AST-based import analysis
- [ ] Add basic file filtering (glob patterns)
- [ ] Add `files: Dict[str, str]` field to `Program` dataclass (database.py)

### Phase 2: Core Evolution (Week 2)
- [ ] Add `evolve_repository()` to API (api.py)
- [ ] Extend `PromptSampler.build_repository_prompt()`
- [ ] Implement multi-file diff parsing (code_utils.py)
- [ ] Add `apply_diff_to_repository()` function
- [ ] Update iteration.py to handle `Program.files`

### Phase 3: Evaluation & Testing (Week 3)
- [ ] Create repository-level evaluator interface
- [ ] Add `evaluate_repository()` method to Evaluator class
- [ ] Add auto-detect for evaluator type (file vs repo)
- [ ] Add integration tests with sample repo
- [ ] Handle edge cases (circular imports, broken code)
- [ ] Add rollback on evaluation failure

### Phase 4: Optimization (Week 4)
- [ ] Smart context window management (don't include all files if too large)
- [ ] Dependency-aware file selection
- [ ] Parallel file loading
- [ ] Caching of repository state
- [ ] Update controller.py for repository checkpointing

---

## Key Challenges & Solutions

### 1. Context Window Limits

**Problem**: Large repos won't fit in LLM context.

**Solution**:
- Use dependency graph to select relevant files
- Include: changed files + their direct dependencies
- Use file size limits and truncation
- Consider iterative file-by-file evolution

### 2. Import Dependencies

**Problem**: Modifying one file can break imports in others.

**Solution**:
- Validate imports after each evolution
- Present dependency info to LLM
- Use "contract" markers for interface preservation
- Add import validation step before evaluation:

```python
def validate_imports(repo_path: Path) -> Tuple[bool, str]:
    """Validate that all imports work in the repository"""
    import ast
    import sys
    
    sys.path.insert(0, str(repo_path))
    
    errors = []
    for py_file in repo_path.rglob("*.py"):
        try:
            with open(py_file) as f:
                ast.parse(f.read())
        except SyntaxError as e:
            errors.append(f"{py_file}: {e}")
    
    return len(errors) == 0, "\n".join(errors)
```

### 3. Evaluation Complexity

**Problem**: Single file evaluators won't work for repos.

**Solution**:
```python
# Repository evaluator receives repo path, not file path
def evaluate_repository(repo_path: Path) -> Dict[str, Any]:
    """Evaluate the entire repository"""
    # Run tests
    # Check imports work
    # Run benchmarks
    return {"score": 0.95, "tests_passed": 42}
```

**Backward-compatible evaluator detection**:
```python
# Auto-detect evaluator type based on function signature
import inspect

def get_evaluator_type(eval_func):
    """Determine if evaluator is file-based or repo-based"""
    sig = inspect.signature(eval_func)
    params = list(sig.parameters.keys())
    
    # If first param suggests Path, it's likely a repo evaluator
    if params and 'repo_path' in params[0].lower():
        return 'repository'
    # If first param suggests file path
    elif params and 'file_path' in params[0].lower() or 'path' in params[0].lower():
        return 'file'  # Default/legacy
    return 'file'
```

### 4. Code Breakage Risk

**Problem**: LLM may generate code that doesn't import.

**Solutions**:
- Post-evolution validation step
- Automatic rollback to parent
- Gradual evolution (one file at a time initially)

---

## Example Usage

### Simple Repository Evolution

```python
from openevolve import evolve_repository, Config
from openevolve.config import LLMModelConfig

# Create evaluator that tests the whole repo
def evaluate_repo(repo_path):
    """
    Repository evaluator receives Path to repo root.
    
    Args:
        repo_path: pathlib.Path to temporary repository directory
        
    Returns:
        Dict with metrics (e.g., {"score": 0.95, "tests_passed": 42})
    """
    import subprocess
    
    result = subprocess.run(
        ["pytest", str(repo_path), "-v", "--tb=short"],
        capture_output=True,
        text=True,
        timeout=300
    )
    
    # Parse pytest output
    passed = result.stdout.count(" PASSED")
    failed = result.stdout.count(" FAILED")
    
    return {
        "score": passed / max(passed + failed, 1),
        "tests_passed": passed,
        "tests_failed": failed,
    }

# Configure for repository evolution
config = Config()
config.llm.models = [LLMModelConfig(name="gemini-2.5-pro", temperature=0.7)]
config.evolution_mode = "repository"
config.repository.include_patterns = ["*.py"]
config.repository.exclude_patterns = ["test_*.py", "examples/*"]
config.repository.max_files_per_iteration = 3

# Evolve the repository
result = evolve_repository(
    repository_path="./my_algorithm",
    evaluator=evaluate_repo,  # Can be function or path to evaluator.py
    config=config,
    iterations=100
)

print(f"Best program: {result.best_program}")
print(f"Best score: {result.best_score}")
```

### Using evolve_repository with evaluator file

```python
# evaluator.py - repository-level evaluator
from pathlib import Path

def evaluate(repo_path: Path) -> dict:
    """Repository evaluator receives repo root path"""
    import subprocess
    
    # Run tests
    result = subprocess.run(
        ["python", "-m", "pytest", str(repo_path), "-v"],
        capture_output=True,
        text=True,
        timeout=120
    )
    
    # Calculate score
    if result.returncode == 0:
        return {"score": 1.0, "all_tests_passed": True}
    else:
        return {"score": 0.0, "all_tests_passed": False, "error": result.stderr}

# Use with evolve_repository
result = evolve_repository(
    repository_path="./my_package",
    evaluator="evaluator.py",  # Path to evaluator
    iterations=100
)
```

### Inline Evaluator (No file needed)

```python
from openevolve import evolve_repository

result = evolve_repository(
    repository_path="./optimization_library",
    evaluator=lambda repo_path: {"score": 0.85},  # Inline function
    iterations=50
)
```

---

## Configuration Options (Config schema)

```yaml
# New config section
repository:
  # File selection
  include_patterns:
    - "*.py"
  exclude_patterns:
    - "test_*.py"
    - "__pycache__"
    - "*.pyc"
  
  # Evolution strategy
  evolve_all_files: false       # True = any file, False = one at a time
  max_files_per_iteration: 5    # Limit files modified per iteration
  
  # Context
  include_dependencies: true    # Include imported files in context
  max_context_files: 10         # Max files in LLM context
  max_file_size: 50000          # Max chars per file
  
  # Safety
  validate_imports: true         # Check imports after evolution
  rollback_on_failure: true     # Revert if evaluation fails
  preserve_interface: true      # Don't change function signatures

# Add to existing Config dataclass (config.py)
@dataclass
class RepositoryConfig:
    """Configuration for repository-level evolution"""
    
    include_patterns: List[str] = field(default_factory=lambda: ["*.py"])
    exclude_patterns: List[str] = field(default_factory=lambda: ["test_*.py", "__pycache__"])
    
    evolve_all_files: bool = False
    max_files_per_iteration: int = 5
    
    include_dependencies: bool = True
    max_context_files: int = 10
    max_file_size: int = 50000
    
    validate_imports: bool = True
    rollback_on_failure: bool = True
    preserve_interface: bool = True
```

**Integration with main Config**:

```python
# config.py additions
@dataclass
class Config:
    # ... existing fields ...
    
    # New: Repository evolution settings
    repository: RepositoryConfig = field(default_factory=RepositoryConfig)
    
    # Evolution mode: "file" (default) or "repository"
    evolution_mode: str = "file"
```

---

## Testing Strategy

### Unit Tests

```python
# tests/test_repository_scanner.py
class TestRepositoryScanner(unittest.TestCase):
    def test_scan_finds_python_files(self):
        scanner = RepositoryScanner(Path("./test_repo"))
        files = scanner.scan(include_patterns=["*.py"])
        self.assertIn("main.py", files)
    
    def test_import_graph_construction(self):
        scanner = RepositoryScanner(Path("./test_repo"))
        scanner.scan()
        self.assertIn("helper.py", scanner.import_graph.get("main.py", set()))
    
    def test_exclude_patterns(self):
        files = scanner.scan(exclude_patterns=["test_*.py"])
        self.assertTrue(all("test_" not in f for f in files))

# tests/test_multi_file_diff.py
class TestMultiFileDiff(unittest.TestCase):
    def test_parse_file_specific_diff(self):
        diff = """<<<<<< SEARCH:src/utils.py
def old_func():
=======
def new_func():
>>>>>>> REPLACE:src/utils.py"""
        blocks = parse_multi_file_diffs(diff)
        self.assertEqual(blocks[0]["file"], "src/utils.py")
    
    def test_apply_diff_to_files(self):
        files = {"src/utils.py": "def old_func():\n    pass"}
        new_files = apply_diffs_to_repository(files, diff)
        self.assertIn("def new_func():", new_files["src/utils.py"])
```

### Integration Tests

```python
# tests/integration/test_repository_evolution.py
class TestRepositoryEvolution(unittest.TestCase):
    @classmethod
    def setUpClass(cls):
        # Create a sample multi-file Python package
        cls.temp_repo = create_sample_package()
    
    def test_evolve_repository_improves_score(self):
        result = evolve_repository(
            repository_path=self.temp_repo,
            evaluator=sample_repo_evaluator,
            iterations=20
        )
        self.assertGreater(result.best_score, 0.5)
    
    def test_imports_remain_valid(self):
        # After evolution, all imports should work
        import sys
        sys.path.insert(0, str(self.temp_repo))
        try:
            import main  # Should not raise
        finally:
            sys.path.remove(str(self.temp_repo))
    
    def test_rollback_on_failure(self):
        # If evaluation fails, should rollback to parent
        pass
```

### Benchmark Tests

```python
# tests/benchmark/test_repo_evolution_performance.py
class TestRepoEvolutionPerformance:
    def test_small_repo_5_files(self):
        """Measure performance on 5-file repo"""
        pass
    
    def test_medium_repo_20_files(self):
        """Test context window management"""
        pass
    
    def test_large_repo_100_files(self):
        """Test dependency-aware file selection"""
        pass
```

---

## Backward Compatibility

- **Existing `run_evolution()`**: Unchanged, continues to work for single files
- **EVOLVE-BLOCK markers**: Still work for single-file optimization (note: currently not actively used in iteration.py - diff-based evolution is the active mechanism)
- **Existing Evaluators**: Continue to work with auto-detected evaluator type
- **New `evolve_repository()`**: Added as separate API

### Migration Path

```python
# Existing single-file code continues to work unchanged
from openevolve import run_evolution

result = run_evolution(
    initial_program="path/to/file.py",
    evaluator="evaluator.py",
    iterations=100
)

# New repository evolution
from openevolve import evolve_repository

result = evolve_repository(
    repository_path="path/to/repo",
    evaluator="repo_evaluator.py",  # Must accept repo_path
    iterations=100
)
```

### Program Dataclass Backward Compatibility

```python
class Program:
    # New code maintains backward compatibility
    def get_code_for_evaluation(self) -> str:
        """Get the code string for single-file evaluation"""
        if self.files:
            # Combine files for backward compatibility
            return self._combine_files_for_eval()
        return self.code  # Original string field
    
    def _combine_files_for_eval(self) -> str:
        """Combine multiple files into single string (for legacy evaluators)"""
        parts = []
        for filepath in sorted(self.files.keys()):
            parts.append(f"# ===== {filepath} =====")
            parts.append(self.files[filepath])
        return "\n\n".join(parts)
```

---

## Success Metrics

1. ✅ Can evolve a 5-file Python package with 100 iterations
2. ✅ Maintains import validity across evolution
3. ✅ Achieves measurable improvements in evaluation metrics
4. ✅ Handles repos up to 50 files (with smart context selection)
5. ✅ Gracefully handles evaluation failures (rollback)

---

## Open Questions

1. **Should we support partial repo evolution (only specific directories)?**
   - **Recommendation**: Yes - Add `include_paths` / `exclude_paths` options

2. **How to handle non-Python repos (multi-language)?**
   - **Recommendation**: Phase 2 could extend RepositoryScanner for other languages
   - Use file extension-based language detection

3. **Should we cache repository state between iterations for performance?**
   - **Recommendation**: Yes - Cache parsed AST, import graph
   - Only invalidate cache when files change

4. **Do we need a special evaluator interface, or can we adapt existing ones?**
   - **Recommendation**: Auto-detect based on function signature: `repo_path` vs `file_path`
   - Backward compatible: existing file evaluators continue to work

5. **How to handle very large repositories (100+ files)?**
   - **Recommendation**: Use dependency graph to select only relevant files
   - Chunk files into batches for different iterations
   - Consider hierarchical evolution (file → module → package)

6. **Should EVOLVE-BLOCK markers work across files?**
   - **Recommendation**: Add new syntax: `# EVOLVE-BLOCK-START:src/utils.py`
   - Target specific files with markers

---

## Risks and Mitigation

| Risk | Impact | Mitigation |
|------|--------|------------|
| **LLM generates incompatible imports** | High - breaks entire repo | Pre-evolution validation, rollback on failure |
| **Context window exceeded** | Medium - evolution fails | Smart file selection, truncation |
| **Circular dependencies** | Medium - broken imports | AST parsing, dependency validation |
| **Performance degradation** | Low - slower evolution | Caching, parallel file loading |
| **Backward compatibility broken** | High - breaks existing users | Strict backward-compat testing |

### Risk Mitigation Strategy

1. **Always validate before evaluation**:
```python
def safe_evolve_iteration(program: Program, diff: str) -> Program:
    """Apply diff safely with validation"""
    # 1. Apply diff to get new files
    new_files = apply_diffs_to_repository(program.files, diff)
    
    # 2. Validate imports
    is_valid, error = validate_imports(new_files)
    if not is_valid:
        logger.warning(f"Import validation failed: {error}")
        return None  # Skip this mutation
    
    # 3. Validate syntax
    is_valid, error = validate_syntax(new_files)
    if not is_valid:
        logger.warning(f"Syntax validation failed: {error}")
        return None
    
    # 4. Only then evaluate
    return evaluate(new_files)
```

2. **Gradual rollout**:
   - Start with Phase 1-2 (foundation + core)
   - Test extensively before Phase 3-4
   - Use feature flags to disable if issues arise

3. **Comprehensive testing**:
   - Test with real-world repositories
   - Test backward compatibility
   - Test edge cases (circular imports, missing files, etc.)

---

## Summary: Key Implementation Points

### Data Model Changes
1. Add `files: Dict[str, str]` to `Program` dataclass
2. Add backward-compatible `get_code_for_evaluation()` method

### New Components
1. `RepositoryScanner` class in `utils/repository_scanner.py`
2. `evolve_repository()` function in `api.py`
3. Multi-file prompt templates
4. Multi-file diff parser

### Modified Components
1. `Evaluator.evaluate_repository()` method
2. `PromptSampler.build_repository_prompt()` extension
3. `apply_diff()` to handle multi-file diffs
4. Config dataclass with new `RepositoryConfig`

### Evaluator Interface
- Existing file-based evaluators: Auto-detected, continue to work
- New repository evaluators: Accept `repo_path: Path` parameter

### Backward Compatibility
- Single-file `run_evolution()` unchanged
- EVOLVE-BLOCK markers continue to work
- Auto-detect evaluator type based on function signature

---

## Related Issues/PRs

- Issue #413: Original request for whole-repo optimization
- Related: Multi-file evolution support
- Related: Dependency-aware prompting
