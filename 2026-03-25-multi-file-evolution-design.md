# Multi-File Automatic Co-Evolution Support for OpenEvolve

**Date:** 2026-03-25
**Status:** Draft (Rev 3 — second review fixes)
**Issue:** https://github.com/algorithmicsuperintelligence/openevolve/issues/413

## Problem

OpenEvolve currently only supports single-file program evolution. Real-world optimization tasks often involve multiple interdependent source files (e.g., a model definition + trainer + data loader). Users must manually combine related code into a single file or only evolve one file in isolation, losing cross-file optimization opportunities.

## Goals

1. **Automatic project discovery**: Given a project directory, automatically scan, analyze dependencies, and determine which files should co-evolve.
2. **Smart co-evolution groups**: Cluster tightly-coupled files into "co-evolution groups" and rotate through them across iterations.
3. **Token-efficient prompts**: Only send full content for files being evolved; send signatures/summaries for context files.
4. **Full backward compatibility**: All existing single-file use cases, evaluators, configs, and checkpoints continue to work unchanged.

## Non-Goals

- IDE/editor integration
- Real-time file watching or incremental re-analysis
- Cross-language dependency analysis (e.g., Python calling Rust FFI) — may be added later
- Distributed multi-machine evolution

## Architecture Overview

New components are inserted between Controller and Iteration:

```
CLI/API
  → Controller
    → ProjectAnalyzer (NEW)
      1. Project scanning (file discovery + filtering)
      2. Static dependency analysis (import graph per language)
      3. LLM semantic association (one-time, at init, best-effort)
      4. File grouping (community detection on merged graph)
    → GroupScheduler (NEW)
      - Selects which co-evolution group to use per iteration
      - Adaptive weighting: groups with recent improvements get higher probability
      - Exploration-exploitation balance (configurable ratio)
      - Serializable for checkpoint/resume
    → Iteration Worker (modified)
      - Receives group-scoped file subset
      - Applies multi-file diffs
      - Returns updated files dict
```

## Detailed Design

### 1. Program Data Model Changes

**File:** `openevolve/database.py`

The `Program` dataclass gains an optional `files` field. Single-file programs keep `files=None` and use `code` as before.

```python
@dataclass
class Program:
    code: str                                # Single-file: full content. Multi-file: main file content.
    files: Optional[Dict[str, str]] = None   # Multi-file: {relative_path -> content}. None for single-file.

    @property
    def is_multi_file(self) -> bool:
        return self.files is not None and len(self.files) > 1

    def get_all_files(self) -> Dict[str, str]:
        """Unified access: single-file returns {"main": code}, multi-file returns files."""
        if self.files:
            return self.files
        return {"main": self.code}

    def get_file(self, path: str) -> Optional[str]:
        if self.files:
            return self.files.get(path)
        return self.code if path == "main" else None

    def get_combined_code_length(self) -> int:
        """For complexity feature dimension."""
        return sum(len(c) for c in self.get_all_files().values())

    def with_updated_files(self, updates: Dict[str, str]) -> "Program":
        """Return new Program with file updates applied (immutable pattern).
        Uses dataclasses.replace() to properly copy all 13+ fields.
        Note: requires adding `replace` to the existing import:
          from dataclasses import asdict, dataclass, field, fields, replace
        """
        new_files = dict(self.get_all_files())
        new_files.update(updates)
        main_code = new_files.get("main", list(new_files.values())[0])
        return replace(
            self,
            code=main_code,
            files=new_files if len(new_files) > 1 else None,
        )
```

**Serialization (backward compatible):**

```json
// Single-file (unchanged):
{"code": "def main(): ...", "score": 0.85}

// Multi-file (new files field):
{"code": "def main(): ...", "files": {"src/main.py": "...", "src/utils.py": "..."}, "score": 0.85}
```

`from_dict` already uses `fields(cls)` to filter valid field names — adding `files` with a default of `None` is naturally handled. Old JSON without `files` key → `files=None`.

**Complexity feature:** `database.py` line 862 changes from `len(program.code)` to `program.get_combined_code_length()`.

**Diversity calculation:** For multi-file programs, the following functions in `database.py` must change:
- `_get_cached_diversity` (line ~2080): hash/compare using concatenated `get_all_files()` content instead of `program.code`
- `_fast_code_diversity` (line ~2097): compare concatenated strings with file-separator markers (e.g., `\n### FILE: {path} ###\n`)
- `_update_diversity_reference_set`: store concatenated strings rather than `program.code`
- For single-file programs (`files=None`), these functions continue using `program.code` unchanged.

**`max_code_length` validation:** The existing check at `process_parallel.py` line 282 (`len(child_code) > config.max_code_length`) changes to `program.get_combined_code_length() > config.max_code_length` for multi-file programs.

### 2. ProjectAnalyzer — Automatic Discovery Engine

**New file:** `openevolve/project_analyzer.py`

#### 2.1 Project Scanner

```python
class ProjectScanner:
    """Recursively scan a directory, filter by language and patterns."""

    DEFAULT_EXCLUDES = [
        "__pycache__/", ".git/", "node_modules/", "*.pyc", "*.pyo",
        "*.so", "*.dylib", "*.exe", ".tox/", ".venv/", "env/",
        "*.egg-info/", "dist/", "build/"
    ]

    MAX_FILE_SIZE = 100_000  # 100KB per file, skip larger files

    def scan(self, root_dir: str, include_patterns: List[str],
             exclude_patterns: List[str]) -> Dict[str, str]:
        """
        Returns {relative_path: file_content} for all matching source files.
        Skips binary files and files exceeding MAX_FILE_SIZE.
        All files read as UTF-8 with errors='replace' for robustness.
        Files are snapshotted at scan time — external changes during
        evolution are NOT reflected.
        """
        ...
```

Language detection is based on file extension. Default include patterns per language:
- Python: `["**/*.py"]`
- Rust: `["**/*.rs"]`
- General: all text files with recognized extensions

#### 2.2 Static Dependency Analyzer

```python
class DependencyAnalyzer:
    """Build file dependency graph from import/include statements."""

    def analyze(self, files: Dict[str, str], language: str) -> DependencyGraph:
        if language == "python":
            return self._analyze_python(files)
        elif language == "rust":
            return self._analyze_rust(files)
        else:
            return self._analyze_generic(files)

    def _analyze_python(self, files: Dict[str, str]) -> DependencyGraph:
        """
        Parse import/from...import statements via regex (not AST, for speed).
        Map import paths to project-internal files.
        Build directed dependency graph.
        Circular imports are valid — the graph may contain cycles.
        Cycles do not affect grouping (we use undirected edges for community detection).
        """
        ...

    def _analyze_rust(self, files: Dict[str, str]) -> DependencyGraph:
        """Parse mod/use crate:: statements."""
        ...

    def _analyze_generic(self, files: Dict[str, str]) -> DependencyGraph:
        """Regex-based fallback for #include, import, require, etc."""
        ...
```

```python
@dataclass
class DependencyGraph:
    edges: Dict[str, List[str]]  # file -> list of files it depends on

    def get_neighbors(self, file: str, depth: int = 1) -> Set[str]:
        """BFS to find all files within N hops (handles cycles via visited set)."""
        ...

    def get_undirected_edges(self) -> List[Tuple[str, str]]:
        """Convert directed deps to undirected pairs (for grouping)."""
        ...
```

#### 2.3 LLM Semantic Analyzer

One-time analysis at project initialization. **Best-effort**: if the LLM call fails (network error, malformed JSON, token limit exceeded), log a warning and fall back to static-only grouping. Evolution proceeds without semantic edges.

```python
class SemanticAnalyzer:
    """Use LLM to discover non-obvious file associations."""

    MAX_FILES_PER_BATCH = 30  # Chunk large projects into batches

    def analyze(self, files: Dict[str, str],
                llm_ensemble: LLMEnsemble) -> List[Tuple[str, str, float]]:
        """
        Send file summaries to LLM, get back semantic association pairs.
        Returns: [(file_a, file_b, coupling_score), ...]

        For projects with > MAX_FILES_PER_BATCH files, splits into
        overlapping batches to stay within LLM context limits.

        On any failure (network, parse, timeout), returns empty list
        and logs a warning. This is best-effort.
        """
        try:
            ...
        except Exception as e:
            logger.warning(f"LLM semantic analysis failed, using static-only grouping: {e}")
            return []
```

**Prompt sent to LLM:**

```
You are analyzing a software project to find files that are semantically coupled —
files where changing one typically requires changing the other.

Project files and their summaries:
{file_summaries}

Return a JSON array of associations:
[{"file_a": "path", "file_b": "path", "score": 0.0-1.0, "reason": "..."}]

Only include pairs with score >= 0.5. Focus on:
- Shared data structures or protocols
- Producer-consumer relationships not visible through imports
- Configuration files that affect specific modules
- Test files paired with their implementation
```

**File summary extraction** reuses AST parsing (for Python) or regex-based extraction:
- Function/class signatures
- Import statements
- Top-level docstrings
- First N lines for non-parseable files

#### 2.4 File Grouper

```python
class FileGrouper:
    """Partition files into co-evolution groups using community detection."""

    def create_groups(self,
                      dep_graph: DependencyGraph,
                      semantic_edges: List[Tuple[str, str, float]],
                      max_group_size: int = 5,
                      max_groups: int = 10) -> List[FileGroup]:
        """
        1. Merge dependency graph edges (weight=1.0) with semantic edges
        2. Build weighted undirected graph
        3. Find connected components
        4. For components > max_group_size: split using label propagation
           (simple iterative algorithm, no external dependencies)
        5. For isolated files: merge into nearest group or form single-file groups
        6. Assign context_files for each group (neighbor files not in the group)
        """
        ...
```

```python
@dataclass
class FileGroup:
    id: int
    files: List[str]           # Evolvable files in this group (relative paths)
    primary_file: str          # Most-depended-on file, or file with __main__ block
    coupling_score: float      # Intra-group coupling strength
    context_files: List[str]   # Read-only context files (neighbors outside group)

    def to_dict(self) -> dict: ...
    @classmethod
    def from_dict(cls, data: dict) -> "FileGroup": ...
```

**Grouping algorithm:**
1. Build adjacency matrix from static deps (weight 1.0) + semantic edges (weight = score). Circular deps are treated as normal edges.
2. Find connected components
3. For components > `max_group_size`: use label propagation to split (iterative: each node adopts the most common label among its neighbors, ties broken by max edge weight). This is simple to implement without external dependencies and handles cycles naturally.
4. For isolated files (no edges): merge into nearest group by any weak association, or form single-file groups
5. For each group, identify `context_files` = all files within 1 hop in the dependency graph but not in the group itself, capped at `max_context_files`
6. **Single-file groups**: groups with only 1 file are valid but use single-file prompt templates (not the multi-file template) to avoid wasted tokens on project structure formatting.

**Primary file detection heuristic** (in order):
1. User-configured `main_file` in config → use that
2. File containing `if __name__ == "__main__":` block (Python)
3. File named `main.py` / `main.rs` / `index.js` etc.
4. File with the most incoming edges (most depended-on)

### 3. GroupScheduler — Iteration-Level Group Selection

**New file:** `openevolve/group_scheduler.py`

```python
class GroupScheduler:
    """Select which co-evolution group to evolve in each iteration."""

    def __init__(self, groups: List[FileGroup], exploration_ratio: float = 0.3):
        self.groups = groups
        self.exploration_ratio = exploration_ratio
        self.improvement_history: Dict[int, List[float]] = {g.id: [] for g in groups}
        self.selection_counts: Dict[int, int] = {g.id: 0 for g in groups}

    def select_group(self, iteration: int) -> FileGroup:
        """
        Selection strategy:
        - exploration_ratio probability: uniform random (explore under-served groups)
        - (1 - exploration_ratio) probability: weighted by recent improvement (exploit)
        """
        if random.random() < self.exploration_ratio:
            return random.choice(self.groups)
        else:
            weights = self._compute_adaptive_weights()
            return random.choices(self.groups, weights=weights, k=1)[0]

    def record_improvement(self, group_id: int, delta: float):
        """Called after each iteration with score improvement."""
        self.improvement_history[group_id].append(delta)
        self.selection_counts[group_id] = self.selection_counts.get(group_id, 0) + 1

    def _compute_adaptive_weights(self) -> List[float]:
        """
        Groups with recent positive improvements get higher weight.
        Groups with no history get baseline weight (ensures they get tried).
        """
        ...

    def get_stats(self) -> Dict[int, Dict]:
        """Returns per-group statistics for logging/visualization."""
        ...

    # --- Checkpoint support ---
    def to_dict(self) -> dict:
        """Serialize scheduler state for checkpoint."""
        return {
            "improvement_history": self.improvement_history,
            "selection_counts": self.selection_counts,
            "exploration_ratio": self.exploration_ratio,
        }

    def load_state(self, state: dict):
        """Restore scheduler state from checkpoint."""
        self.improvement_history = state.get("improvement_history", self.improvement_history)
        self.selection_counts = state.get("selection_counts", self.selection_counts)
```

**Integration with Islands:** Each island's iteration independently calls `select_group()`. Different islands may evolve different groups simultaneously, maximizing diversity.

### 4. Multi-File Diff Format

**File:** `openevolve/utils/code_utils.py`

#### Migration strategy for `extract_diffs` return type

The existing `extract_diffs` returns `List[Tuple[str, str]]` and is consumed in 5+ places. To maintain backward compatibility:

1. **Keep existing `extract_diffs` unchanged** — it continues to return `List[Tuple[str, str]]` for single-file mode.
2. **Add new `extract_diffs_multi` function** — returns `List[DiffBlock]` with file targeting support.
3. Callers choose which to use based on `program.is_multi_file`.
4. The existing `split_diffs_by_target` (which routes diffs between code vs. `changes_description`) is left untouched — it only operates in single-file mode. Multi-file mode uses `extract_diffs_multi` + `apply_multi_file_diffs` instead.

```
<<<<<<< SEARCH file=src/utils.py
def old_helper():
    return 1
=======
def new_helper():
    return 42
>>>>>>> REPLACE

<<<<<<< SEARCH file=src/main.py
result = old_helper()
=======
result = new_helper()
>>>>>>> REPLACE
```

**Rules:**
- No `file=` attribute → fallback to single-file mode (operates on main/only file)
- `file=` paths are relative to project root
- One LLM response can contain SEARCH/REPLACE blocks targeting multiple files
- `file=` attribute parsing is robust: handles optional quotes (`file="path"` or `file=path`), trims whitespace. Malformed attributes (empty value, missing value) → treated as no-target (fallback to main file) with a logged warning.

```python
@dataclass
class DiffBlock:
    search: str
    replace: str
    target_file: Optional[str] = None  # None = single-file compat

def extract_diffs_multi(text: str) -> List[DiffBlock]:
    """New function: parse SEARCH/REPLACE with optional file= attribute.
    Regex: <<<<<<< SEARCH(?:\\s+file=[\"']?([^\"'\\s]+)[\"']?)?\\s*\\n
    Does NOT replace existing extract_diffs — this is additive."""
    ...

def apply_multi_file_diffs(
    files: Dict[str, str],
    diffs: List[DiffBlock],
    allow_new_files: bool = False,
) -> Dict[str, str]:
    """Group diffs by target_file, apply per-file.

    File targeting validation:
    - If target_file not in files and allow_new_files=False → skip with warning
    - If target_file not in files and allow_new_files=True → create new file
    - Empty REPLACE with full-file SEARCH → file deletion (remove from dict)
    """
    grouped = defaultdict(list)
    for diff in diffs:
        target = diff.target_file or next(iter(files))  # default to first/only file
        if target not in files and not allow_new_files:
            logger.warning(f"LLM targeted non-existent file '{target}', skipping diff")
            continue
        grouped[target].append(diff)

    result = dict(files)
    for filename, file_diffs in grouped.items():
        content = result.get(filename, "")
        for d in file_diffs:
            content = apply_diff(content, d.search, d.replace)
        result[filename] = content
    return result
```

**Full rewrite multi-file format:**

```
===== FILE: src/utils.py =====
def new_helper():
    return 42

===== FILE: src/main.py =====
from utils import new_helper
...
```

```python
def parse_multi_file_rewrite(text: str) -> Dict[str, str]:
    """Parse ===== FILE: path ===== delimited blocks."""
    ...
```

### 5. EVOLVE-BLOCK Markers in Multi-File Mode

The existing `parse_evolve_blocks` in `code_utils.py` parses `# EVOLVE-BLOCK-START` / `# EVOLVE-BLOCK-END` markers from a single code string. In multi-file mode:

- **Each file is independently scanned for EVOLVE-BLOCK markers** via `parse_evolve_blocks`.
- Files WITH evolve blocks → automatically classified as "evolvable" (included in co-evolution groups with full content).
- Files WITHOUT evolve blocks but in a co-evolution group → the LLM can still modify them (the blocks are optional guidance, not hard constraints).
- The prompt template shows which sections are marked for evolution:

```
### File: src/model.py (evolvable)
```python
# ... non-evolve code shown for context ...
# EVOLVE-BLOCK-START
def forward(self, x):  # ← Focus improvements here
    ...
# EVOLVE-BLOCK-END
```

- `api.py` auto-wrapping (adding markers to code without them) only applies in single-file mode. Multi-file mode never auto-wraps.

### 6. Interaction with `programs_as_changes_description` Mode

The existing `programs_as_changes_description` mode (see `config.py` line 248, `process_parallel.py` lines 230-268) stores programs as compact change descriptions rather than full code, and uses `split_diffs_by_target` to route diffs between code and changes_description targets.

**In multi-file mode, `programs_as_changes_description` is NOT supported.** Validation in `config.py`:

```python
def __post_init__(self):
    if self.multi_file.enabled and self.prompt.programs_as_changes_description:
        raise ValueError(
            "programs_as_changes_description is not compatible with multi-file mode. "
            "Disable one or the other."
        )
```

Rationale: the changes_description mode represents the program as a diff against the initial program. With multiple files, the diff representation becomes unwieldy and the LLM would need to track which changes apply to which file. This is a future enhancement if needed.

### 7. Prompt Templates

**New template:** `openevolve/prompts/defaults/diff_user_multi_file.txt`

```
# Project Structure
{project_tree}

## Evolvable Files (you may modify these):
{evolvable_files_block}

## Context Files (read-only reference):
{context_files_block}

## File Dependencies:
{dependency_info}

# Current Metrics
{metrics}

# Evolution History
{evolution_history}

# Previous Execution Feedback
{artifacts}

# Instructions
Improve the evolvable files above to optimize {improvement_areas}.
You may modify MULTIPLE files in a single response.
Use the SEARCH/REPLACE format with file= to target specific files:

<<<<<<< SEARCH file=path/to/file.py
original code
=======
improved code
>>>>>>> REPLACE
```

**New template:** `openevolve/prompts/defaults/full_rewrite_multi_file.txt`

Similar structure but instructs LLM to output complete file contents using `===== FILE: path =====` delimiters.

**PromptSampler changes (`prompt/sampler.py`):**

`build_prompt()` checks `program.is_multi_file` to select template. New helper methods:

```python
def _format_evolvable_files(self, files: Dict[str, str], group: FileGroup) -> str:
    """Format full content of files in the current evolution group.
    Truncates files exceeding max_file_content_lines with a notice."""
    sections = []
    for path in group.files:
        content = files.get(path, "")
        lang = self._detect_language(path)
        content = self._truncate_if_needed(content, path)
        sections.append(f"### File: {path}\n```{lang}\n{content}\n```")
    return "\n\n".join(sections)

def _format_context_files(self, files: Dict[str, str], group: FileGroup) -> str:
    """Format summaries (signatures/imports) of context files."""
    sections = []
    for path in group.context_files:
        content = files.get(path, "")
        summary = self._extract_summary(content, path)
        lang = self._detect_language(path)
        sections.append(f"### File: {path} (read-only)\n```{lang}\n{summary}\n```")
    return "\n\n".join(sections)

def _extract_summary(self, content: str, path: str) -> str:
    """Extract function/class signatures, imports, docstrings.
    Uses AST for Python, regex for others.
    Capped at max_file_content_lines from config."""
    ...

def _format_project_tree(self, files: Dict[str, str]) -> str:
    """Simple tree view of all project files with line counts."""
    ...

def _format_dependency_info(self, group: FileGroup, dep_graph: DependencyGraph) -> str:
    """Show which files depend on which within the group."""
    ...

def _truncate_if_needed(self, content: str, path: str) -> str:
    """If file exceeds max_file_content_lines, truncate with notice.
    This is a safety valve for token budget — should rarely trigger
    if max_group_size is properly configured."""
    lines = content.split("\n")
    max_lines = self.config.multi_file.max_file_content_lines
    if len(lines) > max_lines:
        logger.warning(f"File {path} truncated from {len(lines)} to {max_lines} lines in prompt")
        return "\n".join(lines[:max_lines]) + f"\n# ... ({len(lines) - max_lines} lines truncated)"
    return content
```

**Token budget management:** The primary control is `max_group_size` (default 5). With typical files of 100-300 lines, 5 files fit well within standard context windows (128K tokens). The `max_file_content_lines` (default 500) is a per-file safety valve. If the combined prompt still exceeds limits, the LLM API will return an error, which is caught by existing retry logic and logged. Users should reduce `max_group_size` for projects with very large files.

### 8. Evaluator Changes

**File:** `openevolve/evaluator.py`

```python
def evaluate_program(self, program_code: str, program_id: str = "",
                     program_files: Optional[Dict[str, str]] = None):
    """Note: program_files is added AFTER existing program_id parameter
    to preserve backward compatibility with all existing callers that
    pass program_id as the second positional argument."""
    if program_files:
        # Multi-file mode: create temp directory with all files
        temp_dir = tempfile.mkdtemp(prefix="openevolve_")
        try:
            for rel_path, content in program_files.items():
                abs_path = os.path.join(temp_dir, rel_path)
                os.makedirs(os.path.dirname(abs_path), exist_ok=True)
                with open(abs_path, "w", encoding="utf-8") as f:
                    f.write(content)
            result = self.evaluate_function(temp_dir)
        finally:
            shutil.rmtree(temp_dir, ignore_errors=True)
        return result
    else:
        # Single-file mode: completely unchanged
        ...  # existing logic
```

**Evaluator function contract:**
- Single-file: `evaluate_function(file_path: str)` — unchanged
- Multi-file: `evaluate_function(project_dir: str)` — receives directory path
- Selection is automatic based on `program.is_multi_file`

**Distinguishing file vs. directory in evaluators:** The evaluator receives a single string argument in both modes. Multi-file evaluators should check `os.path.isdir(path)`. This convention is documented in the evaluator authoring guide. Example:

```python
def evaluate(path):
    if os.path.isdir(path):
        # Multi-file: path is a temp directory containing all project files
        main_file = os.path.join(path, "src/main.py")
        ...
    else:
        # Single-file: path is the program file
        ...
```

Cascade evaluation (`_cascade_evaluate`) follows the same branching.

### 9. Controller Changes

**File:** `openevolve/controller.py`

**Initial program loading:**

**Init ordering change:** In the current `__init__`, `_load_initial_program()` is called at line 121, before `LLMEnsemble` is initialized (line 140). Since multi-file mode needs the LLM ensemble for semantic analysis, the init order must change:

1. `_load_initial_program()` — first pass: scan files, detect multi-file mode, set `file_extension`. Does NOT run ProjectAnalyzer yet.
2. `LLMEnsemble` initialization (existing, unchanged).
3. `_analyze_project()` — new method, called after LLM init: runs `ProjectAnalyzer` (static + LLM semantic analysis) and creates `GroupScheduler`. Only called in multi-file mode.

```python
def _load_initial_program(self):
    """First pass: load files, detect mode. Does NOT run analysis."""
    path = self.initial_program_path
    if os.path.isdir(path):
        # Multi-file mode: scan directory
        scanner = ProjectScanner()
        files = scanner.scan(
            root_dir=path,
            include_patterns=self.config.multi_file.include_patterns,
            exclude_patterns=self.config.multi_file.exclude_patterns
        )
        if not files:
            raise ValueError(f"No source files found in {path}")

        self.initial_program_files = files
        self.config.multi_file.enabled = True

        # Determine main file
        main_file = self._detect_main_file(files)
        self.initial_program_code = files[main_file]
        self.file_extension = os.path.splitext(main_file)[1]

        # Validate: programs_as_changes_description not compatible
        if self.config.prompt.programs_as_changes_description:
            raise ValueError(
                "programs_as_changes_description is not compatible with multi-file mode"
            )

        # Analysis deferred to _analyze_project() (called after LLM init)
        self.file_groups = None
        self.group_scheduler = None
    else:
        # Single-file mode: completely unchanged
        with open(path, "r") as f:
            self.initial_program_code = f.read()
        self.initial_program_files = None
        self.file_groups = None
        self.group_scheduler = None

def _analyze_project(self):
    """Second pass: run ProjectAnalyzer + create GroupScheduler.
    Called AFTER LLMEnsemble is initialized, only in multi-file mode."""
    if not self.config.multi_file.enabled:
        return

    analyzer = ProjectAnalyzer(self.config.multi_file)
    self.file_groups = analyzer.analyze(self.initial_program_files, self.llm_ensemble)
    self.group_scheduler = GroupScheduler(
        self.file_groups,
        exploration_ratio=self.config.multi_file.exploration_ratio
    )

    logger.info(f"Multi-file mode: {len(self.initial_program_files)} files, "
                f"{len(self.file_groups)} groups")
    for g in self.file_groups:
        logger.info(f"  Group {g.id}: {g.files} (context: {g.context_files})")
```

**Checkpoint saving:**

```python
def _save_checkpoint(self, ...):
    if best_program.is_multi_file:
        # Save as directory structure
        best_dir = os.path.join(checkpoint_path, "best_program")
        os.makedirs(best_dir, exist_ok=True)
        for rel_path, content in best_program.files.items():
            file_path = os.path.join(best_dir, rel_path)
            os.makedirs(os.path.dirname(file_path), exist_ok=True)
            with open(file_path, "w") as f:
                f.write(content)

        # Save file groups and scheduler state
        multi_file_state = {
            "file_groups": [g.to_dict() for g in self.file_groups],
            "scheduler_state": self.group_scheduler.to_dict(),
        }
        with open(os.path.join(checkpoint_path, "multi_file_state.json"), "w") as f:
            json.dump(multi_file_state, f)
    else:
        # Unchanged single-file behavior
        ...
```

**Checkpoint loading:** When resuming, if `multi_file_state.json` exists in the checkpoint directory, restore `file_groups` and `GroupScheduler` state (including `improvement_history` and `selection_counts`).

**CLI entry point changes (`cli.py`):**

```bash
# Single file (unchanged)
openevolve-run initial_program.py evaluator.py

# Multi-file: pass a directory
openevolve-run ./src/ evaluator.py --config config.yaml

# Multi-file: explicit file list via new flag
openevolve-run --files main.py utils.py types.py -- evaluator.py --config config.yaml

# Analyze-only mode: show discovered groups without starting evolution
openevolve-run ./src/ evaluator.py --analyze-only
```

Detection logic:
- `initial_program` is a directory → multi-file directory mode
- `--files` flag provided → create temp directory with listed files, treat as multi-file
- Otherwise → single file, unchanged behavior

Note: **No comma-separated detection** (fragile — file paths can contain commas). Use `--files` flag instead.

New CLI flags:
- `--files FILE [FILE ...]` — explicit list of files for multi-file mode
- `--analyze-only` — run ProjectAnalyzer and print discovered groups, then exit

### 10. Worker Process Changes

**File:** `openevolve/process_parallel.py`

#### 10.1 Config Serialization

`_serialize_config` (line 362) must be updated to include `MultiFileConfig`:

```python
def _serialize_config(config):
    serialized = {
        # ... existing fields ...
        "multi_file": {
            "enabled": config.multi_file.enabled,
            "max_group_size": config.multi_file.max_group_size,
            "max_context_files": config.multi_file.max_context_files,
            "max_file_content_lines": config.multi_file.max_file_content_lines,
            "context_summary_mode": config.multi_file.context_summary_mode,
            "exclude_patterns": config.multi_file.exclude_patterns,
            "include_patterns": config.multi_file.include_patterns,
            "exploration_ratio": config.multi_file.exploration_ratio,
            "enable_llm_analysis": config.multi_file.enable_llm_analysis,
            "main_file": config.multi_file.main_file,
        },
    }
    return serialized
```

#### 10.2 Worker Init

`_worker_init` (lines 55-89) must reconstruct `MultiFileConfig` AND add `"multi_file"` to the exclusion set that prevents double-passing via `**kwargs`:

```python
def _worker_init(serialized_config, ...):
    # ... existing reconstructions ...
    multi_file_config = MultiFileConfig(**serialized_config.get("multi_file", {}))
    config = Config(
        # ... existing fields ...
        multi_file=multi_file_config,
        # The **{k:v ...} fallback must exclude "multi_file":
        **{k: v for k, v in config_dict.items()
           if k not in ["llm", "prompt", "database", "evaluator", "multi_file"]}
    )
```

#### 10.3 Snapshot Extension

- Program dicts in snapshot include `files` key when multi-file
- `SerializableResult` gains `files: Optional[Dict[str, str]]` field
- `file_groups` (serialized) and assigned `group_id` are passed to workers via snapshot
- The controller pre-assigns a group to each iteration via `GroupScheduler.select_group()` before submitting to the worker pool

#### 10.4 Snapshot Size Management

With multi-file programs, snapshot size grows multiplicatively. Mitigation:
- Only include `files` dict for the parent program and inspiration programs (top/diverse programs used in prompts)
- Other programs in the snapshot carry `files=None` — they only need `code` and `metrics` for sampling/selection
- This is an extension of the existing `max_snapshot_artifacts` pattern (lines 455-468)

#### 10.5 Worker Iteration Logic

```python
def _run_iteration_worker(snapshot, config, ...):
    parent = reconstruct_program(snapshot, island_id)

    if parent.is_multi_file:
        group = FileGroup.from_dict(snapshot["assigned_group"])

        # Build multi-file prompt with group context
        prompt = sampler.build_multi_file_prompt(parent, group, ...)

        # Generate LLM response
        response = asyncio.run(llm.generate(prompt))

        # Apply multi-file diffs (new function, does not touch extract_diffs)
        diffs = extract_diffs_multi(response)
        updated_files = apply_multi_file_diffs(parent.get_all_files(), diffs)

        # Create child program
        child = parent.with_updated_files(updated_files)
        child_code = child.code
        child_files = child.files

        # Validate total code length
        if child.get_combined_code_length() > config.max_code_length:
            # reject, same as existing single-file rejection
            ...

        # Evaluate (passes directory)
        result = evaluator.evaluate_program(child_code, child_files)
    else:
        # Single-file: completely unchanged existing logic
        ...

    return SerializableResult(code=child_code, files=child_files, ...)
```

### 11. Configuration

**New config section in `openevolve/config.py`:**

```python
@dataclass
class MultiFileConfig:
    enabled: bool = False                          # Auto-set to True when directory input detected
    exclude_patterns: List[str] = field(default_factory=lambda: [
        "__pycache__/", "*.pyc", ".git/", "node_modules/",
        "*.so", "*.dylib", "*.exe", ".tox/", ".venv/"
    ])
    include_patterns: Optional[List[str]] = None   # None = auto-detect by language
    main_file: Optional[str] = None                # Override main file detection

    # Group settings
    max_group_size: int = 5
    max_groups: int = 10
    max_context_files: int = 5

    # Scheduling
    exploration_ratio: float = 0.3

    # LLM analysis
    enable_llm_analysis: bool = True

    # Prompt control
    max_file_content_lines: int = 500
    context_summary_mode: str = "signatures"        # "signatures" | "first_n_lines" | "full"
```

Added to top-level `Config` (note: the actual class name is `Config`, not `OpenEvolveConfig`):

```python
@dataclass
class Config:
    # ... existing fields ...
    multi_file: MultiFileConfig = field(default_factory=MultiFileConfig)
```

### 12. API Changes

**File:** `openevolve/api.py`

```python
def run_evolution(
    initial_program: Union[str, Path, List[str], Dict[str, str]],  # Added Dict for multi-file
    evaluator: Union[str, Path, Callable],
    ...
):
    """
    initial_program can now be:
    - str/Path pointing to file: single file path (unchanged)
    - str/Path pointing to directory: auto-scanned multi-file
    - str: inline code (unchanged)
    - List[str]: lines of code (unchanged)
    - Dict[str, str]: {filename: content} for multi-file
    """
    ...
```

New convenience function:

```python
def evolve_project(
    project_dir: Union[str, Path],
    evaluator: Union[str, Path, Callable],
    config: Optional[dict] = None,
    **kwargs
) -> EvolutionResult:
    """
    Evolve a multi-file project.
    Automatically scans directory, analyzes dependencies, and co-evolves files.
    """
    ...
```

`EvolutionResult` gains `best_files: Optional[Dict[str, str]]` alongside existing `best_code: str`.

## Error Handling Summary

| Scenario | Behavior |
|----------|----------|
| LLM returns `file=` path not in project | Log warning, skip that diff block |
| LLM returns malformed `file=` attribute | Treat as no-target (main file), log warning |
| LLM semantic analysis fails (network/parse) | Fall back to static-only grouping, log warning |
| Circular dependencies in import graph | Handled naturally — cycles are valid edges for grouping |
| Project with > 30 files for LLM analysis | Batch into overlapping chunks of 30 |
| File encoding issues | Read with `encoding='utf-8', errors='replace'` |
| External file changes during evolution | Ignored — files snapshotted at init |
| Single-file group selected | Use single-file prompt template, not multi-file |
| Combined prompt exceeds LLM context | Per-file truncation + existing API error/retry logic |
| `programs_as_changes_description` + multi-file | Raise `ValueError` at init — incompatible |
| Empty project directory | Raise `ValueError` at init |

## Known Pre-Existing Issues to Fix

**`diff_pattern` not serialized in `_serialize_config`:** The current `_serialize_config` (line 362) does not serialize `diff_pattern`, so workers use the default value instead of the user-configured value. This should be fixed as part of the `_serialize_config` changes in P9.

## Implementation Notes

- `with_updated_files` allows a single-file program to transition to multi-file if updates contain keys other than `"main"`. This is intentional — it enables the rare case where the LLM creates a new file during evolution. However, this transition only happens when `allow_new_files=True` in `apply_multi_file_diffs`.
- The `from dataclasses import ...` line in `database.py` must add `replace` to support `with_updated_files`.
- When adding `"multi_file"` to `_worker_init`'s exclusion set, also verify that `evolution_trace` (another sub-config) continues to work via the implicit `**kwargs` passthrough.

## Files Changed Summary

| File | Change Type | Description |
|------|-------------|-------------|
| `openevolve/database.py` | Modified | `Program` gains `files` field, serialization, `get_all_files()`, complexity calc, diversity calc |
| `openevolve/project_analyzer.py` | **New** | `ProjectScanner`, `DependencyAnalyzer`, `SemanticAnalyzer`, `FileGrouper` |
| `openevolve/group_scheduler.py` | **New** | `GroupScheduler` with adaptive selection + checkpoint support |
| `openevolve/utils/code_utils.py` | Modified | `DiffBlock`, new `extract_diffs_multi`, `apply_multi_file_diffs`, `parse_multi_file_rewrite` (existing `extract_diffs` unchanged) |
| `openevolve/prompt/sampler.py` | Modified | Multi-file prompt building, summary extraction, template selection, truncation |
| `openevolve/prompts/defaults/diff_user_multi_file.txt` | **New** | Multi-file diff prompt template |
| `openevolve/prompts/defaults/full_rewrite_multi_file.txt` | **New** | Multi-file full rewrite prompt template |
| `openevolve/evaluator.py` | Modified | Temp directory mode for multi-file evaluation |
| `openevolve/controller.py` | Modified | Directory loading, ProjectAnalyzer init, checkpoint multi-file save/load, logging |
| `openevolve/cli.py` | Modified | Directory detection, `--files` flag, `--analyze-only` flag |
| `openevolve/process_parallel.py` | Modified | `_serialize_config` + `_worker_init` for `MultiFileConfig`, snapshot `files` field, group-aware worker logic, snapshot size optimization |
| `openevolve/config.py` | Modified | `MultiFileConfig` dataclass added to `Config` |
| `openevolve/api.py` | Modified | `Dict[str, str]` input support, `evolve_project()` function, `EvolutionResult.best_files` |
| `openevolve/iteration.py` | Modified | Multi-file diff application in iteration logic |

## Implementation Phases

| Phase | Content | Dependencies | Risk |
|-------|---------|--------------|------|
| **P1** | `Program` data model + serialization compat + diversity/complexity | None | Low |
| **P2** | `DiffBlock` + `extract_diffs_multi` (new) + `apply_multi_file_diffs` | P1 | Low |
| **P3** | `ProjectScanner` + `DependencyAnalyzer` (static analysis) | None | Low |
| **P4** | `SemanticAnalyzer` (LLM, best-effort) + `FileGrouper` (label propagation) | P3 | Medium |
| **P5** | `GroupScheduler` (adaptive rotation + checkpoint serialization) | P4 | Low |
| **P6** | Multi-file prompt templates + `PromptSampler` adaptation + truncation | P1, P2, P5 | Medium |
| **P7** | `Evaluator` temp directory mode | P1 | Low |
| **P8** | `Controller` directory loading + checkpoint save/load + CLI flags | P1, P3, P5 | Low |
| **P9** | `process_parallel.py`: `_serialize_config`, `_worker_init`, snapshot, worker logic | P1, P2, P5, P6 | Medium |
| **P10** | `MultiFileConfig` + `Config` integration + validation | P3-P5 | Low |
| **P11** | `api.py` changes + `evolve_project()` | P1-P10 | Low |
| **P12** | Multi-file example project + unit tests + integration tests | All | Low |

## Backward Compatibility

- **Single-file programs**: `files=None`, all existing code paths unchanged.
- **Existing evaluators**: Receive single file path as before; multi-file evaluators receive directory path (distinguished via `os.path.isdir()`).
- **Existing configs**: No `multi_file` section → `MultiFileConfig` with defaults, `enabled=False`.
- **Existing checkpoints**: `from_dict` handles missing `files` key gracefully (`files=None`).
- **Existing prompt templates**: Single-file templates remain untouched; multi-file templates are new files.
- **Existing `extract_diffs`**: Unchanged. New `extract_diffs_multi` is additive.
- **Existing `split_diffs_by_target`**: Unchanged. Only used in single-file + `changes_description` mode.
- **CLI**: `openevolve-run single_file.py evaluator.py` works exactly as before.
- **`programs_as_changes_description`**: Explicitly incompatible with multi-file, validated at init.
