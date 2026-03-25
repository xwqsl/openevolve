# Multi-File Co-Evolution Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add multi-file automatic co-evolution support to OpenEvolve so users can pass a project directory and have OpenEvolve automatically discover, group, and co-evolve related files.

**Architecture:** New `ProjectAnalyzer` and `GroupScheduler` components sit between Controller and Worker. `Program` gains an optional `files` dict. New `extract_diffs_multi` handles file-targeted SEARCH/REPLACE. All single-file code paths remain untouched.

**Tech Stack:** Python 3.10+, dataclasses, re (regex), unittest, existing OpenEvolve infrastructure.

**Spec:** `docs/superpowers/specs/2026-03-25-multi-file-evolution-design.md`

---

## File Structure

| File | Status | Responsibility |
|------|--------|---------------|
| `openevolve/config.py` | Modify | Add `MultiFileConfig` dataclass |
| `openevolve/database.py` | Modify | `Program.files`, serialization, complexity/diversity |
| `openevolve/utils/code_utils.py` | Modify | `DiffBlock`, `extract_diffs_multi`, `apply_multi_file_diffs`, `parse_multi_file_rewrite` |
| `openevolve/project_analyzer.py` | **Create** | `ProjectScanner`, `DependencyAnalyzer`, `SemanticAnalyzer`, `FileGrouper`, `FileGroup`, `DependencyGraph` |
| `openevolve/group_scheduler.py` | **Create** | `GroupScheduler` with adaptive selection + checkpoint |
| `openevolve/prompts/defaults/diff_user_multi_file.txt` | **Create** | Multi-file diff prompt template |
| `openevolve/prompts/defaults/full_rewrite_multi_file.txt` | **Create** | Multi-file full rewrite prompt template |
| `openevolve/prompt/sampler.py` | Modify | Multi-file prompt building helpers, template selection |
| `openevolve/evaluator.py` | Modify | Temp directory mode for multi-file |
| `openevolve/controller.py` | Modify | Directory loading, `_analyze_project`, checkpoint save/load |
| `openevolve/cli.py` | Modify | `--files`, `--analyze-only` flags |
| `openevolve/process_parallel.py` | Modify | Config serialization, worker init, snapshot, worker logic |
| `openevolve/iteration.py` | Modify | Multi-file diff branch in iteration logic |
| `openevolve/api.py` | Modify | `Dict` input, `evolve_project()`, `EvolutionResult.best_files` |
| `tests/test_multi_file_config.py` | **Create** | Config tests |
| `tests/test_multi_file_program.py` | **Create** | Program model tests |
| `tests/test_multi_file_diff.py` | **Create** | Diff parsing/applying tests |
| `tests/test_project_analyzer.py` | **Create** | Scanner, dependency, grouper tests |
| `tests/test_group_scheduler.py` | **Create** | Scheduler tests |
| `tests/test_multi_file_evaluator.py` | **Create** | Evaluator temp dir tests |
| `tests/test_multi_file_prompt.py` | **Create** | Prompt building tests |

---

### Task 1: MultiFileConfig

**Files:**
- Modify: `openevolve/config.py:400-431`
- Create: `tests/test_multi_file_config.py`

- [ ] **Step 1: Write tests for MultiFileConfig**

```python
# tests/test_multi_file_config.py
import unittest
from openevolve.config import MultiFileConfig, Config


class TestMultiFileConfig(unittest.TestCase):
    def test_default_values(self):
        cfg = MultiFileConfig()
        self.assertFalse(cfg.enabled)
        self.assertEqual(cfg.max_group_size, 5)
        self.assertEqual(cfg.max_groups, 10)
        self.assertEqual(cfg.max_context_files, 5)
        self.assertAlmostEqual(cfg.exploration_ratio, 0.3)
        self.assertTrue(cfg.enable_llm_analysis)
        self.assertEqual(cfg.max_file_content_lines, 500)
        self.assertEqual(cfg.context_summary_mode, "signatures")
        self.assertIsNone(cfg.include_patterns)
        self.assertIsNone(cfg.main_file)
        self.assertIn("__pycache__/", cfg.exclude_patterns)

    def test_config_has_multi_file(self):
        cfg = Config()
        self.assertIsInstance(cfg.multi_file, MultiFileConfig)
        self.assertFalse(cfg.multi_file.enabled)

    def test_config_from_dict_with_multi_file(self):
        cfg = Config.from_dict({
            "multi_file": {
                "enabled": True,
                "max_group_size": 3,
                "exploration_ratio": 0.5,
            }
        })
        self.assertTrue(cfg.multi_file.enabled)
        self.assertEqual(cfg.multi_file.max_group_size, 3)
        self.assertAlmostEqual(cfg.multi_file.exploration_ratio, 0.5)

    def test_config_from_dict_without_multi_file(self):
        cfg = Config.from_dict({})
        self.assertFalse(cfg.multi_file.enabled)

    def test_multi_file_incompatible_with_changes_description(self):
        with self.assertRaises(ValueError):
            Config.from_dict({
                "multi_file": {"enabled": True},
                "prompt": {"programs_as_changes_description": True},
                "diff_based_evolution": True,
            })


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m unittest tests/test_multi_file_config.py -v`
Expected: FAIL — `MultiFileConfig` does not exist yet.

- [ ] **Step 3: Implement MultiFileConfig**

In `openevolve/config.py`, add before the `Config` class (before line 400):

```python
@dataclass
class MultiFileConfig:
    """Configuration for multi-file co-evolution"""

    enabled: bool = False
    exclude_patterns: List[str] = field(
        default_factory=lambda: [
            "__pycache__/", "*.pyc", ".git/", "node_modules/",
            "*.so", "*.dylib", "*.exe", ".tox/", ".venv/",
        ]
    )
    include_patterns: Optional[List[str]] = None
    main_file: Optional[str] = None

    max_group_size: int = 5
    max_groups: int = 10
    max_context_files: int = 5

    exploration_ratio: float = 0.3

    enable_llm_analysis: bool = True

    max_file_content_lines: int = 500
    context_summary_mode: str = "signatures"
```

Add to `Config` class field list (after `evolution_trace` field, ~line 418):

```python
    multi_file: MultiFileConfig = field(default_factory=MultiFileConfig)
```

In `Config.from_dict` (~line 449), **no manual `MultiFileConfig` reconstruction is needed** — `dacite.from_dict` (used at line 465) automatically handles nested dataclass fields. Simply adding `multi_file: MultiFileConfig` to the `Config` dataclass is sufficient for deserialization from YAML/dict.

Add validation after the existing `programs_as_changes_description` check (~line 477):

```python
        if config.multi_file.enabled and config.prompt.programs_as_changes_description:
            raise ValueError(
                "programs_as_changes_description is not compatible with multi-file mode. "
                "Disable one or the other."
            )
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python -m unittest tests/test_multi_file_config.py -v`
Expected: All 5 tests PASS.

- [ ] **Step 5: Run existing tests for regression**

Run: `python -m unittest discover tests -p "test_*.py" -v 2>&1 | tail -5`
Expected: All existing tests still pass.

- [ ] **Step 6: Commit**

```bash
git add openevolve/config.py tests/test_multi_file_config.py
git commit -m "feat: add MultiFileConfig dataclass for multi-file evolution"
```

---

### Task 2: Program Data Model Extension

**Files:**
- Modify: `openevolve/database.py:13,43-110,860-864`
- Create: `tests/test_multi_file_program.py`

- [ ] **Step 1: Write tests for Program multi-file support**

```python
# tests/test_multi_file_program.py
import unittest
from openevolve.database import Program


class TestProgramMultiFile(unittest.TestCase):
    def test_single_file_is_multi_file_false(self):
        p = Program(id="1", code="print('hello')")
        self.assertFalse(p.is_multi_file)
        self.assertIsNone(p.files)

    def test_multi_file_is_multi_file_true(self):
        p = Program(id="1", code="main", files={"a.py": "a", "b.py": "b"})
        self.assertTrue(p.is_multi_file)

    def test_single_file_get_all_files(self):
        p = Program(id="1", code="print('hello')")
        self.assertEqual(p.get_all_files(), {"main": "print('hello')"})

    def test_multi_file_get_all_files(self):
        files = {"a.py": "code_a", "b.py": "code_b"}
        p = Program(id="1", code="code_a", files=files)
        self.assertEqual(p.get_all_files(), files)

    def test_get_file_single(self):
        p = Program(id="1", code="hello")
        self.assertEqual(p.get_file("main"), "hello")
        self.assertIsNone(p.get_file("other.py"))

    def test_get_file_multi(self):
        p = Program(id="1", code="a", files={"a.py": "a", "b.py": "b"})
        self.assertEqual(p.get_file("a.py"), "a")
        self.assertIsNone(p.get_file("c.py"))

    def test_get_combined_code_length_single(self):
        p = Program(id="1", code="12345")
        self.assertEqual(p.get_combined_code_length(), 5)

    def test_get_combined_code_length_multi(self):
        p = Program(id="1", code="abc", files={"a.py": "abc", "b.py": "de"})
        self.assertEqual(p.get_combined_code_length(), 5)

    def test_with_updated_files(self):
        p = Program(id="1", code="old_a", files={"a.py": "old_a", "b.py": "old_b"})
        p2 = p.with_updated_files({"a.py": "new_a"})
        self.assertEqual(p2.files["a.py"], "new_a")
        self.assertEqual(p2.files["b.py"], "old_b")
        self.assertEqual(p2.id, "1")  # fields preserved

    def test_with_updated_files_preserves_fields(self):
        p = Program(id="x", code="c", generation=5, metrics={"score": 1.0},
                    files={"a.py": "c", "b.py": "d"})
        p2 = p.with_updated_files({"a.py": "new"})
        self.assertEqual(p2.generation, 5)
        self.assertEqual(p2.metrics, {"score": 1.0})

    def test_to_dict_includes_files(self):
        p = Program(id="1", code="a", files={"a.py": "a", "b.py": "b"})
        d = p.to_dict()
        self.assertEqual(d["files"], {"a.py": "a", "b.py": "b"})

    def test_to_dict_single_file_files_is_none(self):
        p = Program(id="1", code="hello")
        d = p.to_dict()
        self.assertIsNone(d["files"])

    def test_from_dict_with_files(self):
        d = {"id": "1", "code": "a", "files": {"a.py": "a", "b.py": "b"}}
        p = Program.from_dict(d)
        self.assertEqual(p.files, {"a.py": "a", "b.py": "b"})
        self.assertTrue(p.is_multi_file)

    def test_from_dict_without_files(self):
        d = {"id": "1", "code": "hello"}
        p = Program.from_dict(d)
        self.assertIsNone(p.files)
        self.assertFalse(p.is_multi_file)


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m unittest tests/test_multi_file_program.py -v`
Expected: FAIL — `is_multi_file`, `get_all_files`, etc. do not exist.

- [ ] **Step 3: Implement Program extensions**

In `openevolve/database.py`:

Update import (line 13):
```python
from dataclasses import asdict, dataclass, field, fields, replace
```

Add `files` field to `Program` (after `embedding` field, ~line 78):
```python
    # Multi-file support
    files: Optional[Dict[str, str]] = None
```

Add methods to `Program` (after `from_dict`, before the `ProgramDatabase` class):
```python
    @property
    def is_multi_file(self) -> bool:
        return self.files is not None and len(self.files) > 1

    def get_all_files(self) -> Dict[str, str]:
        if self.files:
            return self.files
        return {"main": self.code}

    def get_file(self, path: str) -> Optional[str]:
        if self.files:
            return self.files.get(path)
        return self.code if path == "main" else None

    def get_combined_code_length(self) -> int:
        return sum(len(c) for c in self.get_all_files().values())

    def with_updated_files(self, updates: Dict[str, str]) -> "Program":
        new_files = dict(self.get_all_files())
        new_files.update(updates)
        main_code = new_files.get("main", list(new_files.values())[0])
        return replace(
            self,
            code=main_code,
            files=new_files if len(new_files) > 1 else None,
        )
```

Update complexity calculation (~line 862):
```python
                complexity = program.get_combined_code_length()
```

Update diversity calculations for multi-file support:

In `_get_cached_diversity` (~line 2080), change from `program.code` to concatenated multi-file content:
```python
    def _get_code_for_diversity(self, program: Program) -> str:
        """Get code string for diversity calculation, handling multi-file."""
        if program.is_multi_file:
            return "\n### FILE_SEP ###\n".join(
                f"### FILE: {k} ###\n{v}" for k, v in sorted(program.get_all_files().items())
            )
        return program.code
```

Use this helper in `_get_cached_diversity`, `_fast_code_diversity` (~line 2097), and `_update_diversity_reference_set` (~line 2108) instead of directly accessing `program.code`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `python -m unittest tests/test_multi_file_program.py -v`
Expected: All 14 tests PASS.

- [ ] **Step 5: Run existing tests for regression**

Run: `python -m unittest discover tests -p "test_*.py" -v 2>&1 | tail -5`
Expected: All existing tests still pass.

- [ ] **Step 6: Commit**

```bash
git add openevolve/database.py tests/test_multi_file_program.py
git commit -m "feat: add multi-file support to Program dataclass"
```

---

### Task 3: Multi-File Diff Parsing and Application

**Files:**
- Modify: `openevolve/utils/code_utils.py:78-92`
- Create: `tests/test_multi_file_diff.py`

- [ ] **Step 1: Write tests for DiffBlock, extract_diffs_multi, apply_multi_file_diffs**

```python
# tests/test_multi_file_diff.py
import unittest
from openevolve.utils.code_utils import (
    DiffBlock,
    extract_diffs_multi,
    apply_multi_file_diffs,
    parse_multi_file_rewrite,
    extract_diffs,  # verify old function still works
)


class TestDiffBlock(unittest.TestCase):
    def test_dataclass_fields(self):
        d = DiffBlock(search="a", replace="b", target_file="x.py")
        self.assertEqual(d.search, "a")
        self.assertEqual(d.replace, "b")
        self.assertEqual(d.target_file, "x.py")

    def test_default_target_file_none(self):
        d = DiffBlock(search="a", replace="b")
        self.assertIsNone(d.target_file)


class TestExtractDiffsMulti(unittest.TestCase):
    def test_single_diff_with_file(self):
        text = (
            "<<<<<<< SEARCH file=utils.py\n"
            "old code\n"
            "=======\n"
            "new code\n"
            ">>>>>>> REPLACE\n"
        )
        diffs = extract_diffs_multi(text)
        self.assertEqual(len(diffs), 1)
        self.assertEqual(diffs[0].target_file, "utils.py")
        self.assertEqual(diffs[0].search, "old code")
        self.assertEqual(diffs[0].replace, "new code")

    def test_multiple_diffs_different_files(self):
        text = (
            "<<<<<<< SEARCH file=a.py\nold_a\n=======\nnew_a\n>>>>>>> REPLACE\n"
            "<<<<<<< SEARCH file=b.py\nold_b\n=======\nnew_b\n>>>>>>> REPLACE\n"
        )
        diffs = extract_diffs_multi(text)
        self.assertEqual(len(diffs), 2)
        self.assertEqual(diffs[0].target_file, "a.py")
        self.assertEqual(diffs[1].target_file, "b.py")

    def test_diff_without_file_attr(self):
        text = "<<<<<<< SEARCH\nold\n=======\nnew\n>>>>>>> REPLACE\n"
        diffs = extract_diffs_multi(text)
        self.assertEqual(len(diffs), 1)
        self.assertIsNone(diffs[0].target_file)

    def test_file_attr_with_quotes(self):
        text = '<<<<<<< SEARCH file="src/main.py"\nold\n=======\nnew\n>>>>>>> REPLACE\n'
        diffs = extract_diffs_multi(text)
        self.assertEqual(diffs[0].target_file, "src/main.py")

    def test_mixed_with_and_without_file(self):
        text = (
            "<<<<<<< SEARCH file=a.py\nold_a\n=======\nnew_a\n>>>>>>> REPLACE\n"
            "<<<<<<< SEARCH\nold_x\n=======\nnew_x\n>>>>>>> REPLACE\n"
        )
        diffs = extract_diffs_multi(text)
        self.assertEqual(diffs[0].target_file, "a.py")
        self.assertIsNone(diffs[1].target_file)

    def test_malformed_file_attr_empty(self):
        """Malformed file= with no value falls back to None target."""
        text = "<<<<<<< SEARCH file=\nold\n=======\nnew\n>>>>>>> REPLACE\n"
        diffs = extract_diffs_multi(text)
        self.assertEqual(len(diffs), 1)
        self.assertIsNone(diffs[0].target_file)

    def test_malformed_file_attr_spaces(self):
        """file= with only spaces falls back to None target."""
        text = "<<<<<<< SEARCH file=   \nold\n=======\nnew\n>>>>>>> REPLACE\n"
        diffs = extract_diffs_multi(text)
        self.assertEqual(len(diffs), 1)
        self.assertIsNone(diffs[0].target_file)


class TestApplyMultiFileDiffs(unittest.TestCase):
    def test_apply_to_multiple_files(self):
        files = {"a.py": "def f():\n    return 1", "b.py": "x = 1"}
        diffs = [
            DiffBlock(search="return 1", replace="return 2", target_file="a.py"),
            DiffBlock(search="x = 1", replace="x = 2", target_file="b.py"),
        ]
        result = apply_multi_file_diffs(files, diffs)
        self.assertIn("return 2", result["a.py"])
        self.assertIn("x = 2", result["b.py"])

    def test_no_target_defaults_to_first_file(self):
        files = {"main.py": "old"}
        diffs = [DiffBlock(search="old", replace="new")]
        result = apply_multi_file_diffs(files, diffs)
        self.assertEqual(result["main.py"], "new")

    def test_nonexistent_file_skipped_by_default(self):
        files = {"a.py": "code"}
        diffs = [DiffBlock(search="x", replace="y", target_file="missing.py")]
        result = apply_multi_file_diffs(files, diffs)
        self.assertNotIn("missing.py", result)

    def test_nonexistent_file_created_when_allowed(self):
        files = {"a.py": "code"}
        diffs = [DiffBlock(search="", replace="new content", target_file="new.py")]
        result = apply_multi_file_diffs(files, diffs, allow_new_files=True)
        self.assertIn("new.py", result)


class TestParseMultiFileRewrite(unittest.TestCase):
    def test_parse_two_files(self):
        text = (
            "===== FILE: src/a.py =====\ndef a():\n    pass\n\n"
            "===== FILE: src/b.py =====\ndef b():\n    pass\n"
        )
        result = parse_multi_file_rewrite(text)
        self.assertEqual(len(result), 2)
        self.assertIn("def a():", result["src/a.py"])
        self.assertIn("def b():", result["src/b.py"])

    def test_returns_empty_for_no_markers(self):
        result = parse_multi_file_rewrite("just some text")
        self.assertEqual(result, {})


class TestExistingExtractDiffsUnchanged(unittest.TestCase):
    def test_old_extract_diffs_still_works(self):
        text = "<<<<<<< SEARCH\nold\n=======\nnew\n>>>>>>> REPLACE\n"
        diffs = extract_diffs(text)
        self.assertEqual(len(diffs), 1)
        self.assertEqual(diffs[0], ("old", "new"))


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m unittest tests/test_multi_file_diff.py -v`
Expected: FAIL — `DiffBlock`, `extract_diffs_multi`, etc. do not exist.

- [ ] **Step 3: Implement DiffBlock, extract_diffs_multi, apply_multi_file_diffs, parse_multi_file_rewrite**

Add to `openevolve/utils/code_utils.py` (after existing `extract_diffs`, do NOT modify it):

```python
@dataclass
class DiffBlock:
    """A diff block with optional file targeting for multi-file mode."""
    search: str
    replace: str
    target_file: Optional[str] = None


def extract_diffs_multi(
    diff_text: str,
) -> List[DiffBlock]:
    """Extract SEARCH/REPLACE blocks with optional file= attribute.
    Does NOT replace existing extract_diffs — this is additive."""
    pattern = r'<<<<<<< SEARCH(?:\s+file=["\']?([^"\'\s>]+)["\']?)?\s*\n(.*?)=======\n(.*?)>>>>>>> REPLACE'
    matches = re.findall(pattern, diff_text, re.DOTALL)
    result = []
    for match in matches:
        target_file = match[0].strip() if match[0].strip() else None
        result.append(DiffBlock(
            search=match[1].rstrip(),
            replace=match[2].rstrip(),
            target_file=target_file,
        ))
    return result


def apply_multi_file_diffs(
    files: Dict[str, str],
    diffs: List[DiffBlock],
    allow_new_files: bool = False,
) -> Dict[str, str]:
    """Group diffs by target_file, apply per-file."""
    import logging
    logger = logging.getLogger(__name__)

    grouped: Dict[str, List[DiffBlock]] = {}
    for diff in diffs:
        target = diff.target_file or next(iter(files))
        if target not in files and not allow_new_files:
            logger.warning(f"LLM targeted non-existent file '{target}', skipping diff")
            continue
        grouped.setdefault(target, []).append(diff)

    result = dict(files)
    for filename, file_diffs in grouped.items():
        content = result.get(filename, "")
        for d in file_diffs:
            content = apply_diff(content, d.search, d.replace)
        result[filename] = content
    return result


def parse_multi_file_rewrite(text: str) -> Dict[str, str]:
    """Parse ===== FILE: path ===== delimited blocks for full-rewrite multi-file mode."""
    pattern = r'={5}\s*FILE:\s*(\S+)\s*={5}\s*\n'
    parts = re.split(pattern, text)
    # parts: [preamble, filename1, content1, filename2, content2, ...]
    result = {}
    for i in range(1, len(parts) - 1, 2):
        filename = parts[i].strip()
        content = parts[i + 1].strip()
        result[filename] = content
    return result
```

Add necessary imports at top of file if not present:
```python
from dataclasses import dataclass
from typing import Dict, List, Optional
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python -m unittest tests/test_multi_file_diff.py -v`
Expected: All 13 tests PASS.

- [ ] **Step 5: Run existing tests for regression**

Run: `python -m unittest tests/test_code_utils.py -v`
Expected: All existing code_utils tests still pass.

- [ ] **Step 6: Commit**

```bash
git add openevolve/utils/code_utils.py tests/test_multi_file_diff.py
git commit -m "feat: add multi-file diff parsing and application"
```

---

### Task 4: ProjectScanner and DependencyAnalyzer

**Files:**
- Create: `openevolve/project_analyzer.py`
- Create: `tests/test_project_analyzer.py`

- [ ] **Step 1: Write tests for ProjectScanner**

```python
# tests/test_project_analyzer.py
import os
import tempfile
import unittest
from openevolve.project_analyzer import (
    ProjectScanner,
    DependencyAnalyzer,
    DependencyGraph,
    FileGrouper,
    FileGroup,
)


class TestProjectScanner(unittest.TestCase):
    def setUp(self):
        self.tmpdir = tempfile.mkdtemp()
        # Create a small project structure
        os.makedirs(os.path.join(self.tmpdir, "src"))
        with open(os.path.join(self.tmpdir, "src", "main.py"), "w") as f:
            f.write("from utils import helper\n\ndef main():\n    helper()\n")
        with open(os.path.join(self.tmpdir, "src", "utils.py"), "w") as f:
            f.write("def helper():\n    return 42\n")
        with open(os.path.join(self.tmpdir, "src", "data.txt"), "w") as f:
            f.write("not a python file")
        os.makedirs(os.path.join(self.tmpdir, "__pycache__"))
        with open(os.path.join(self.tmpdir, "__pycache__", "cached.pyc"), "wb") as f:
            f.write(b"\x00")

    def test_scan_finds_python_files(self):
        scanner = ProjectScanner()
        files = scanner.scan(self.tmpdir, ["**/*.py"], [])
        paths = set(files.keys())
        self.assertIn(os.path.join("src", "main.py"), paths)
        self.assertIn(os.path.join("src", "utils.py"), paths)

    def test_scan_excludes_pycache(self):
        scanner = ProjectScanner()
        files = scanner.scan(self.tmpdir, ["**/*.py", "**/*.pyc"], [])
        for path in files:
            self.assertNotIn("__pycache__", path)

    def test_scan_respects_include_patterns(self):
        scanner = ProjectScanner()
        files = scanner.scan(self.tmpdir, ["**/*.txt"], [])
        self.assertEqual(len(files), 1)
        self.assertIn(os.path.join("src", "data.txt"), list(files.keys()))

    def test_scan_empty_dir(self):
        empty = tempfile.mkdtemp()
        scanner = ProjectScanner()
        files = scanner.scan(empty, ["**/*.py"], [])
        self.assertEqual(files, {})


class TestDependencyAnalyzer(unittest.TestCase):
    def test_python_imports(self):
        files = {
            "main.py": "from utils import helper\nimport os\n",
            "utils.py": "def helper(): pass\n",
        }
        analyzer = DependencyAnalyzer()
        graph = analyzer.analyze(files, "python")
        self.assertIn("utils.py", graph.edges.get("main.py", []))

    def test_python_from_import(self):
        files = {
            "app.py": "from models import User\n",
            "models.py": "class User: pass\n",
        }
        analyzer = DependencyAnalyzer()
        graph = analyzer.analyze(files, "python")
        self.assertIn("models.py", graph.edges.get("app.py", []))

    def test_no_external_deps_in_graph(self):
        files = {"main.py": "import os\nimport sys\n"}
        analyzer = DependencyAnalyzer()
        graph = analyzer.analyze(files, "python")
        deps = graph.edges.get("main.py", [])
        self.assertNotIn("os", deps)

    def test_get_neighbors(self):
        graph = DependencyGraph(edges={
            "a.py": ["b.py"],
            "b.py": ["c.py"],
            "c.py": [],
        })
        neighbors = graph.get_neighbors("a.py", depth=1)
        self.assertIn("b.py", neighbors)
        self.assertNotIn("c.py", neighbors)

    def test_get_neighbors_depth2(self):
        graph = DependencyGraph(edges={
            "a.py": ["b.py"],
            "b.py": ["c.py"],
            "c.py": [],
        })
        neighbors = graph.get_neighbors("a.py", depth=2)
        self.assertIn("b.py", neighbors)
        self.assertIn("c.py", neighbors)

    def test_circular_deps_no_infinite_loop(self):
        graph = DependencyGraph(edges={
            "a.py": ["b.py"],
            "b.py": ["a.py"],
        })
        neighbors = graph.get_neighbors("a.py", depth=5)
        self.assertEqual(neighbors, {"b.py"})


class TestFileGrouper(unittest.TestCase):
    def test_connected_components_become_groups(self):
        graph = DependencyGraph(edges={
            "a.py": ["b.py"], "b.py": [],
            "c.py": ["d.py"], "d.py": [],
        })
        grouper = FileGrouper()
        groups = grouper.create_groups(graph, [], max_group_size=5)
        self.assertEqual(len(groups), 2)

    def test_large_component_is_split(self):
        edges = {f"f{i}.py": [f"f{i+1}.py"] for i in range(10)}
        edges["f10.py"] = []
        graph = DependencyGraph(edges=edges)
        grouper = FileGrouper()
        groups = grouper.create_groups(graph, [], max_group_size=3)
        for g in groups:
            self.assertLessEqual(len(g.files), 3)

    def test_isolated_files(self):
        graph = DependencyGraph(edges={"a.py": [], "b.py": []})
        grouper = FileGrouper()
        groups = grouper.create_groups(graph, [])
        total_files = sum(len(g.files) for g in groups)
        self.assertEqual(total_files, 2)

    def test_group_has_context_files(self):
        graph = DependencyGraph(edges={
            "a.py": ["b.py"], "b.py": ["c.py"], "c.py": [],
        })
        grouper = FileGrouper()
        groups = grouper.create_groups(graph, [], max_group_size=2)
        # At least one group should have context_files
        all_context = []
        for g in groups:
            all_context.extend(g.context_files)
        self.assertTrue(len(all_context) > 0)

    def test_file_group_serialization(self):
        fg = FileGroup(id=0, files=["a.py", "b.py"], primary_file="a.py",
                       coupling_score=0.8, context_files=["c.py"])
        d = fg.to_dict()
        fg2 = FileGroup.from_dict(d)
        self.assertEqual(fg2.files, ["a.py", "b.py"])
        self.assertEqual(fg2.context_files, ["c.py"])


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m unittest tests/test_project_analyzer.py -v`
Expected: FAIL — module does not exist.

- [ ] **Step 3: Implement ProjectScanner, DependencyAnalyzer, DependencyGraph, FileGrouper, FileGroup**

Create `openevolve/project_analyzer.py` with the full implementation. Key classes:

- `ProjectScanner`: walk directory, match glob patterns, read files as UTF-8 with `errors='replace'`, skip files > `MAX_FILE_SIZE` (100KB), always exclude `DEFAULT_EXCLUDES`.
- `DependencyGraph` dataclass: `edges: Dict[str, List[str]]`, `get_neighbors(file, depth)` with BFS + visited set, `get_undirected_edges()`.
- `DependencyAnalyzer`: `analyze(files, language)` dispatches to `_analyze_python`, `_analyze_rust`, `_analyze_generic`. Python analyzer uses regex `r'^(?:from|import)\s+(\w+)'` then maps module names to project files.
- `FileGroup` dataclass: `id`, `files`, `primary_file`, `coupling_score`, `context_files`, with `to_dict()`/`from_dict()`.
- `FileGrouper`: `create_groups(dep_graph, semantic_edges, max_group_size, max_groups)`. Algorithm: find connected components in undirected graph, split large components via label propagation, assign context_files from 1-hop neighbors.

See spec Section 2 for full details.

- [ ] **Step 4: Run tests to verify they pass**

Run: `python -m unittest tests/test_project_analyzer.py -v`
Expected: All 15 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add openevolve/project_analyzer.py tests/test_project_analyzer.py
git commit -m "feat: add ProjectScanner, DependencyAnalyzer, and FileGrouper"
```

---

### Task 5: SemanticAnalyzer (LLM-based, best-effort)

**Files:**
- Modify: `openevolve/project_analyzer.py`
- Create: `tests/test_semantic_analyzer.py`

- [ ] **Step 1: Write tests for SemanticAnalyzer**

```python
# tests/test_semantic_analyzer.py
import unittest
from unittest.mock import AsyncMock, MagicMock
from openevolve.project_analyzer import SemanticAnalyzer


class TestSemanticAnalyzer(unittest.TestCase):
    def test_returns_empty_on_failure(self):
        """LLM failure should return empty list, not raise."""
        mock_ensemble = MagicMock()
        mock_ensemble.generate = AsyncMock(side_effect=Exception("network error"))
        analyzer = SemanticAnalyzer()
        result = analyzer.analyze({"a.py": "code"}, mock_ensemble)
        self.assertEqual(result, [])

    def test_parses_valid_json_response(self):
        mock_ensemble = MagicMock()
        mock_ensemble.generate = AsyncMock(return_value=(
            '[{"file_a": "a.py", "file_b": "b.py", "score": 0.8, "reason": "shared data"}]'
        ))
        analyzer = SemanticAnalyzer()
        result = analyzer.analyze({"a.py": "code_a", "b.py": "code_b"}, mock_ensemble)
        self.assertEqual(len(result), 1)
        self.assertEqual(result[0], ("a.py", "b.py", 0.8))

    def test_returns_empty_on_invalid_json(self):
        mock_ensemble = MagicMock()
        mock_ensemble.generate = AsyncMock(return_value="not json at all")
        analyzer = SemanticAnalyzer()
        result = analyzer.analyze({"a.py": "code"}, mock_ensemble)
        self.assertEqual(result, [])

    def test_extract_file_summary_python(self):
        code = (
            '"""Module docstring."""\n'
            "import os\n"
            "from typing import List\n\n"
            "class Foo:\n"
            "    def bar(self, x: int) -> str:\n"
            "        return str(x)\n\n"
            "def standalone(a, b):\n"
            "    pass\n"
        )
        analyzer = SemanticAnalyzer()
        summary = analyzer._extract_file_summary(code, "test.py")
        self.assertIn("import os", summary)
        self.assertIn("class Foo", summary)
        self.assertIn("def bar", summary)
        self.assertIn("def standalone", summary)


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m unittest tests/test_semantic_analyzer.py -v`
Expected: FAIL — `SemanticAnalyzer` does not exist.

- [ ] **Step 3: Implement SemanticAnalyzer**

Add to `openevolve/project_analyzer.py`:

- `SemanticAnalyzer` class with `MAX_FILES_PER_BATCH = 30`
- `analyze(files, llm_ensemble)`: builds summaries, sends to LLM, parses JSON response. Wraps everything in try/except returning `[]` on failure.
- `_extract_file_summary(code, path)`: for `.py` files, uses regex to extract imports, class/function signatures, and docstrings. For other files, returns first 30 lines.
- `_build_analysis_prompt(file_summaries)`: formats the prompt per spec Section 2.3.
- For large projects (> 30 files), split into overlapping batches, merge results.

- [ ] **Step 4: Run tests to verify they pass**

Run: `python -m unittest tests/test_semantic_analyzer.py -v`
Expected: All 4 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add openevolve/project_analyzer.py tests/test_semantic_analyzer.py
git commit -m "feat: add SemanticAnalyzer for LLM-based file coupling discovery"
```

---

### Task 6: GroupScheduler

**Files:**
- Create: `openevolve/group_scheduler.py`
- Create: `tests/test_group_scheduler.py`

- [ ] **Step 1: Write tests for GroupScheduler**

```python
# tests/test_group_scheduler.py
import unittest
from openevolve.project_analyzer import FileGroup
from openevolve.group_scheduler import GroupScheduler


class TestGroupScheduler(unittest.TestCase):
    def _make_groups(self, n=3):
        return [
            FileGroup(id=i, files=[f"f{i}.py"], primary_file=f"f{i}.py",
                      coupling_score=0.5, context_files=[])
            for i in range(n)
        ]

    def test_select_group_returns_group(self):
        groups = self._make_groups()
        sched = GroupScheduler(groups)
        g = sched.select_group(0)
        self.assertIsInstance(g, FileGroup)
        self.assertIn(g, groups)

    def test_record_improvement(self):
        groups = self._make_groups()
        sched = GroupScheduler(groups)
        sched.record_improvement(0, 0.1)
        self.assertEqual(sched.improvement_history[0], [0.1])
        self.assertEqual(sched.selection_counts[0], 1)

    def test_all_groups_selected_eventually(self):
        groups = self._make_groups(3)
        sched = GroupScheduler(groups, exploration_ratio=1.0)  # always explore
        selected_ids = set()
        for i in range(100):
            g = sched.select_group(i)
            selected_ids.add(g.id)
        self.assertEqual(selected_ids, {0, 1, 2})

    def test_to_dict_and_load_state(self):
        groups = self._make_groups()
        sched = GroupScheduler(groups)
        sched.record_improvement(0, 0.5)
        sched.record_improvement(1, 0.2)
        state = sched.to_dict()

        sched2 = GroupScheduler(groups)
        sched2.load_state(state)
        self.assertEqual(sched2.improvement_history[0], [0.5])
        self.assertEqual(sched2.selection_counts[0], 1)

    def test_get_stats(self):
        groups = self._make_groups()
        sched = GroupScheduler(groups)
        sched.record_improvement(0, 0.1)
        stats = sched.get_stats()
        self.assertIn(0, stats)


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m unittest tests/test_group_scheduler.py -v`
Expected: FAIL — module does not exist.

- [ ] **Step 3: Implement GroupScheduler**

Create `openevolve/group_scheduler.py`:

```python
import logging
import random
from dataclasses import dataclass, field
from typing import Dict, List, Optional

from openevolve.project_analyzer import FileGroup

logger = logging.getLogger(__name__)


class GroupScheduler:
    """Select which co-evolution group to evolve in each iteration."""

    def __init__(self, groups: List[FileGroup], exploration_ratio: float = 0.3):
        self.groups = groups
        self.exploration_ratio = exploration_ratio
        self.improvement_history: Dict[int, List[float]] = {g.id: [] for g in groups}
        self.selection_counts: Dict[int, int] = {g.id: 0 for g in groups}

    def select_group(self, iteration: int) -> FileGroup:
        if random.random() < self.exploration_ratio:
            return random.choice(self.groups)
        weights = self._compute_adaptive_weights()
        return random.choices(self.groups, weights=weights, k=1)[0]

    def record_improvement(self, group_id: int, delta: float):
        self.improvement_history[group_id].append(delta)
        self.selection_counts[group_id] = self.selection_counts.get(group_id, 0) + 1

    def _compute_adaptive_weights(self) -> List[float]:
        weights = []
        for g in self.groups:
            history = self.improvement_history.get(g.id, [])
            if not history:
                weights.append(1.0)  # baseline for untried groups
            else:
                recent = history[-10:]  # last 10 improvements
                avg_improvement = sum(max(0, d) for d in recent) / len(recent)
                weights.append(1.0 + avg_improvement * 10)
            weights[-1] = max(weights[-1], 0.1)  # floor
        return weights

    def get_stats(self) -> Dict[int, Dict]:
        stats = {}
        for g in self.groups:
            history = self.improvement_history.get(g.id, [])
            stats[g.id] = {
                "selections": self.selection_counts.get(g.id, 0),
                "total_improvements": len(history),
                "avg_improvement": sum(history) / len(history) if history else 0.0,
                "files": g.files,
            }
        return stats

    def to_dict(self) -> dict:
        return {
            "improvement_history": {str(k): v for k, v in self.improvement_history.items()},
            "selection_counts": {str(k): v for k, v in self.selection_counts.items()},
            "exploration_ratio": self.exploration_ratio,
        }

    def load_state(self, state: dict):
        self.improvement_history = {
            int(k): v for k, v in state.get("improvement_history", {}).items()
        }
        self.selection_counts = {
            int(k): v for k, v in state.get("selection_counts", {}).items()
        }
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python -m unittest tests/test_group_scheduler.py -v`
Expected: All 5 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add openevolve/group_scheduler.py tests/test_group_scheduler.py
git commit -m "feat: add GroupScheduler with adaptive group selection"
```

---

### Task 7: Multi-File Prompt Templates

**Files:**
- Create: `openevolve/prompts/defaults/diff_user_multi_file.txt`
- Create: `openevolve/prompts/defaults/full_rewrite_multi_file.txt`
- Create: `tests/test_multi_file_prompt.py`

- [ ] **Step 1: Create diff_user_multi_file.txt template**

Write to `openevolve/prompts/defaults/diff_user_multi_file.txt`:

```
{system_message}

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
Improve the evolvable files above. You may modify MULTIPLE files in a single response.
Use the SEARCH/REPLACE format with file= to target specific files:

<<<<<<< SEARCH file=path/to/file.py
original code
=======
improved code
>>>>>>> REPLACE

{improvement_areas}
```

- [ ] **Step 2: Create full_rewrite_multi_file.txt template**

Write to `openevolve/prompts/defaults/full_rewrite_multi_file.txt`:

```
{system_message}

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
Rewrite the evolvable files to improve them. Output the COMPLETE content of each modified file using this format:

===== FILE: path/to/file.py =====
complete file content here

{improvement_areas}
```

- [ ] **Step 3: Write tests for PromptSampler multi-file helpers**

```python
# tests/test_multi_file_prompt.py
import unittest
from openevolve.prompt.sampler import PromptSampler
from openevolve.config import PromptConfig, MultiFileConfig
from openevolve.project_analyzer import FileGroup, DependencyGraph


class TestMultiFilePromptHelpers(unittest.TestCase):
    def setUp(self):
        self.config = PromptConfig()
        self.multi_file_config = MultiFileConfig(max_file_content_lines=50)
        self.sampler = PromptSampler(self.config)

    def test_format_evolvable_files(self):
        files = {"src/main.py": "def main():\n    pass", "src/utils.py": "def util(): pass"}
        group = FileGroup(id=0, files=["src/main.py", "src/utils.py"],
                          primary_file="src/main.py", coupling_score=0.5, context_files=[])
        result = self.sampler._format_evolvable_files(files, group, self.multi_file_config)
        self.assertIn("src/main.py", result)
        self.assertIn("def main():", result)
        self.assertIn("src/utils.py", result)

    def test_format_context_files(self):
        files = {"lib.py": "class Lib:\n    def method(self): pass\n"}
        group = FileGroup(id=0, files=["main.py"], primary_file="main.py",
                          coupling_score=0.5, context_files=["lib.py"])
        result = self.sampler._format_context_files(files, group, self.multi_file_config)
        self.assertIn("lib.py", result)
        self.assertIn("read-only", result)

    def test_format_project_tree(self):
        files = {"src/main.py": "x" * 100, "src/utils.py": "y" * 50}
        result = self.sampler._format_project_tree(files)
        self.assertIn("src/main.py", result)
        self.assertIn("src/utils.py", result)

    def test_truncate_if_needed(self):
        long_content = "\n".join(f"line {i}" for i in range(100))
        cfg = MultiFileConfig(max_file_content_lines=10)
        result = self.sampler._truncate_if_needed(long_content, "test.py", cfg)
        self.assertIn("truncated", result)
        self.assertEqual(len(result.split("\n")), 11)  # 10 lines + truncation notice


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 4: Implement PromptSampler multi-file helpers**

Add to `openevolve/prompt/sampler.py`:

- `_format_evolvable_files(self, files, group, multi_file_config)` — format full content per group file
- `_format_context_files(self, files, group, multi_file_config)` — format summaries of context files
- `_format_project_tree(self, files)` — tree view with line counts
- `_format_dependency_info(self, group, dep_graph)` — show intra-group dependencies
- `_truncate_if_needed(self, content, path, multi_file_config)` — truncate long files
- `_extract_summary(self, content, path)` — extract signatures/imports for context
- `_detect_language(self, path)` — map file extension to language name

Update template selection logic (~line 88-97) to add multi-file cases:
```python
        if template_key:
            user_template_key = template_key
        elif self.user_template_override:
            user_template_key = self.user_template_override
        elif kwargs.get("is_multi_file"):
            user_template_key = "diff_user_multi_file" if diff_based_evolution else "full_rewrite_multi_file"
        else:
            user_template_key = "diff_user" if diff_based_evolution else "full_rewrite_user"
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `python -m unittest tests/test_multi_file_prompt.py -v`
Expected: All 4 tests PASS.

- [ ] **Step 6: Commit**

```bash
git add openevolve/prompts/defaults/diff_user_multi_file.txt \
        openevolve/prompts/defaults/full_rewrite_multi_file.txt \
        openevolve/prompt/sampler.py \
        tests/test_multi_file_prompt.py
git commit -m "feat: add multi-file prompt templates and sampler helpers"
```

---

### Task 8: Evaluator Multi-File Support

**Files:**
- Modify: `openevolve/evaluator.py:132-136`
- Create: `tests/test_multi_file_evaluator.py`

- [ ] **Step 1: Write tests**

```python
# tests/test_multi_file_evaluator.py
import os
import tempfile
import unittest
from unittest.mock import MagicMock, AsyncMock
from openevolve.evaluator import Evaluator
from openevolve.config import EvaluatorConfig


class TestEvaluatorMultiFile(unittest.TestCase):
    def test_multi_file_creates_temp_dir(self):
        """When program_files is provided, evaluator creates a temp directory."""
        created_paths = []

        def mock_evaluate(path):
            created_paths.append(path)
            self.assertTrue(os.path.isdir(path))
            self.assertTrue(os.path.exists(os.path.join(path, "src", "main.py")))
            self.assertTrue(os.path.exists(os.path.join(path, "src", "utils.py")))
            return {"score": 1.0}

        evaluator = Evaluator(
            evaluate_function=mock_evaluate,
            config=EvaluatorConfig(),
            program_suffix=".py",
        )
        import asyncio
        result = asyncio.run(evaluator.evaluate_program(
            program_code="main content",
            program_id="test1",
            program_files={"src/main.py": "main content", "src/utils.py": "util content"},
        ))
        self.assertEqual(len(created_paths), 1)
        # Temp dir should be cleaned up
        self.assertFalse(os.path.exists(created_paths[0]))

    def test_single_file_unchanged(self):
        """Without program_files, evaluator behaves as before."""
        received_paths = []

        def mock_evaluate(path):
            received_paths.append(path)
            self.assertTrue(os.path.isfile(path))
            return {"score": 0.5}

        evaluator = Evaluator(
            evaluate_function=mock_evaluate,
            config=EvaluatorConfig(),
            program_suffix=".py",
        )
        import asyncio
        result = asyncio.run(evaluator.evaluate_program(
            program_code="print('hello')",
            program_id="test2",
        ))
        self.assertEqual(len(received_paths), 1)


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m unittest tests/test_multi_file_evaluator.py -v`
Expected: FAIL — `program_files` parameter not accepted.

- [ ] **Step 3: Implement multi-file evaluator support**

In `openevolve/evaluator.py`, update `evaluate_program` signature (~line 132):

```python
    async def evaluate_program(
        self,
        program_code: str,
        program_id: str = "",
        program_files: Optional[Dict[str, str]] = None,
    ) -> Dict[str, float]:
```

Add `import shutil` at top if not present. Add multi-file branch before the existing single-file temp file logic:

```python
        if program_files:
            temp_dir = tempfile.mkdtemp(prefix="openevolve_")
            try:
                for rel_path, content in program_files.items():
                    abs_path = os.path.join(temp_dir, rel_path)
                    os.makedirs(os.path.dirname(abs_path), exist_ok=True)
                    with open(abs_path, "w", encoding="utf-8") as f:
                        f.write(content)
                return self._run_evaluation(temp_dir, program_id)
            finally:
                shutil.rmtree(temp_dir, ignore_errors=True)
```

Apply the same pattern to `_cascade_evaluate` if it exists.

- [ ] **Step 4: Run tests to verify they pass**

Run: `python -m unittest tests/test_multi_file_evaluator.py -v`
Expected: All 2 tests PASS.

- [ ] **Step 5: Run existing evaluator tests for regression**

Run: `python -m unittest tests/test_evaluator_timeout.py tests/test_cascade_validation.py -v`
Expected: All existing tests still pass.

- [ ] **Step 6: Commit**

```bash
git add openevolve/evaluator.py tests/test_multi_file_evaluator.py
git commit -m "feat: add multi-file temp directory support to Evaluator"
```

---

### Task 9: Controller — Directory Loading, _analyze_project, Checkpoint

**Files:**
- Modify: `openevolve/controller.py:70-189,441-496,534-583`
- Modify: `openevolve/cli.py:18-60`

- [ ] **Step 1: Modify `_load_initial_program` for directory support**

In `openevolve/controller.py`, update `_load_initial_program` (~line 244):
- If `self.initial_program_path` is a directory → use `ProjectScanner` to scan, set `self.initial_program_files`, detect main file, set `config.multi_file.enabled = True`.
- If file → keep existing behavior, set `self.initial_program_files = None`.

Add import at top:
```python
from openevolve.project_analyzer import ProjectScanner, ProjectAnalyzer
from openevolve.group_scheduler import GroupScheduler
```

- [ ] **Step 2: Add `_analyze_project` method**

Add new method to Controller (after `_load_initial_program`):
```python
    def _analyze_project(self):
        """Run ProjectAnalyzer and create GroupScheduler. Called after LLM init."""
        if not self.config.multi_file.enabled:
            return
        from openevolve.project_analyzer import ProjectAnalyzer
        analyzer = ProjectAnalyzer(self.config.multi_file)
        self.file_groups = analyzer.analyze(self.initial_program_files, self.llm_ensemble)
        self.group_scheduler = GroupScheduler(
            self.file_groups,
            exploration_ratio=self.config.multi_file.exploration_ratio,
        )
        logger.info(f"Multi-file: {len(self.initial_program_files)} files, "
                     f"{len(self.file_groups)} groups")
```

Call `self._analyze_project()` in `__init__` AFTER `self.llm_ensemble` is initialized (~after line 141).

- [ ] **Step 3: Update `_save_checkpoint` for multi-file**

In `_save_checkpoint` (~line 441), add multi-file branch:
- If `best_program.is_multi_file`: create `best_program/` directory in checkpoint, write each file. Also save `multi_file_state.json` with groups and scheduler state.

- [ ] **Step 3.5: Update checkpoint loading for multi-file state**

In `__init__` or wherever checkpoints are restored (look for `--checkpoint` handling in `_load_checkpoint` or the constructor), add:
- If `multi_file_state.json` exists in checkpoint dir: load `file_groups` from it (deserialize `FileGroup.from_dict`), create `GroupScheduler` with the loaded groups, and call `scheduler.load_state()` with the saved scheduler state.
- This ensures resumed runs retain adaptive group weighting.

- [ ] **Step 4: Update `_save_best_program` for multi-file**

In `_save_best_program` (~line 534):
- If `program.is_multi_file`: create directory structure instead of single file.

- [ ] **Step 5: Add CLI flags**

In `openevolve/cli.py`, add to argparse (~line 18-60):
```python
    parser.add_argument("--files", nargs="+", default=None,
                        help="Explicit list of files for multi-file mode")
    parser.add_argument("--analyze-only", action="store_true", default=False,
                        help="Analyze project structure and print groups, then exit")
```

- [ ] **Step 6: Run existing tests**

Run: `python -m unittest discover tests -p "test_*.py" -v 2>&1 | tail -5`
Expected: All existing tests still pass.

- [ ] **Step 7: Commit**

```bash
git add openevolve/controller.py openevolve/cli.py
git commit -m "feat: add directory loading, project analysis, and multi-file checkpoint support"
```

---

### Task 10: Worker Process Changes (process_parallel.py)

**Files:**
- Modify: `openevolve/process_parallel.py:24-37,39-95,362-394`

- [ ] **Step 1: Update SerializableResult**

Add `files` field to `SerializableResult` dataclass (~line 24):
```python
    files: Optional[Dict[str, str]] = None
```

- [ ] **Step 2: Update `_serialize_config`**

Add `"multi_file"` section to the serialized dict (~line 362). Include all `MultiFileConfig` fields. Also add `"diff_pattern"` (pre-existing missing field fix).

- [ ] **Step 3: Update `_worker_init`**

Reconstruct `MultiFileConfig` from serialized dict. Add `"multi_file"` AND `"evolution_trace"` to the exclusion set at ~line 84-88 (also reconstruct `EvolutionTraceConfig` explicitly to fix a pre-existing fragility where it passes through as a raw dict):
```python
if k not in ["llm", "prompt", "database", "evaluator", "multi_file", "evolution_trace"]
```

- [ ] **Step 4: Update worker iteration logic**

In `_run_iteration_worker` (~line 134), add multi-file branch:
- If parent is multi-file: use `extract_diffs_multi` + `apply_multi_file_diffs`, validate combined code length, evaluate with `program_files`.
- Single-file: unchanged.

Update snapshot creation to include `files` for parent/inspiration programs, and `assigned_group` for multi-file iterations.

- [ ] **Step 5: Run existing parallel tests**

Run: `python -m unittest tests/test_process_parallel.py tests/test_process_parallel_fix.py -v`
Expected: All existing tests still pass.

- [ ] **Step 6: Commit**

```bash
git add openevolve/process_parallel.py
git commit -m "feat: add multi-file support to worker processes"
```

---

### Task 10.5: Iteration.py Multi-File Branch

**Files:**
- Modify: `openevolve/iteration.py:1-20,98-144`

- [ ] **Step 1: Add multi-file imports**

In `openevolve/iteration.py`, add to the existing import block (~line 13):
```python
from openevolve.utils.code_utils import (
    apply_diff,
    apply_diff_blocks,
    extract_diffs,
    extract_diffs_multi,         # NEW
    apply_multi_file_diffs,      # NEW
    format_diff_summary,
    parse_full_rewrite,
    parse_multi_file_rewrite,    # NEW
    split_diffs_by_target,
)
```

- [ ] **Step 2: Add multi-file branch to iteration logic**

In the `run_iteration` method (~lines 98-144), where diffs are extracted and applied, add a multi-file branch:

```python
        if parent.is_multi_file:
            if diff_based_evolution:
                diffs = extract_diffs_multi(llm_response)
                updated_files = apply_multi_file_diffs(parent.get_all_files(), diffs)
            else:
                updated_files = parse_multi_file_rewrite(llm_response)
                if not updated_files:
                    updated_files = parent.get_all_files()
            child = parent.with_updated_files(updated_files)
            child_code = child.code
        else:
            # existing single-file logic unchanged
            ...
```

- [ ] **Step 3: Run existing iteration tests**

Run: `python -m unittest discover tests -p "test_*.py" -v 2>&1 | tail -5`
Expected: All existing tests still pass.

- [ ] **Step 4: Commit**

```bash
git add openevolve/iteration.py
git commit -m "feat: add multi-file diff branch to iteration logic"
```

---

### Task 11: API Changes

**Files:**
- Modify: `openevolve/api.py:19-40`

- [ ] **Step 1: Update EvolutionResult**

Add `best_files` field to `EvolutionResult` (~line 19):
```python
    best_files: Optional[Dict[str, str]] = None
```

- [ ] **Step 2: Update run_evolution signature**

Change type hint (~line 33):
```python
    initial_program: Union[str, Path, List[str], Dict[str, str]],
```

Add handling for `Dict[str, str]` input in `_prepare_program`: write all files to a temp directory, return directory path.

- [ ] **Step 3: Add evolve_project convenience function**

```python
def evolve_project(
    project_dir: Union[str, Path],
    evaluator: Union[str, Path, Callable],
    config: Union[str, Path, "Config", None] = None,
    iterations: Optional[int] = None,
    output_dir: Optional[str] = None,
    cleanup: bool = True,
) -> EvolutionResult:
    """Evolve a multi-file project directory."""
    project_dir = str(project_dir)
    if not os.path.isdir(project_dir):
        raise ValueError(f"project_dir must be a directory: {project_dir}")
    return run_evolution(
        initial_program=project_dir,
        evaluator=evaluator,
        config=config,
        iterations=iterations,
        output_dir=output_dir,
        cleanup=cleanup,
    )
```

- [ ] **Step 4: Run existing API tests**

Run: `python -m unittest tests/test_api.py -v`
Expected: All existing tests still pass.

- [ ] **Step 5: Commit**

```bash
git add openevolve/api.py
git commit -m "feat: add Dict input and evolve_project() to API"
```

---

### Task 12: Multi-File Example and Integration Test

**Files:**
- Create: `examples/multi_file_optimization/` (example project)
- Create: `tests/test_multi_file_integration.py`

- [ ] **Step 1: Create example project**

Create `examples/multi_file_optimization/` with:
- `src/main.py` — entry point with `EVOLVE-BLOCK` markers
- `src/utils.py` — helper functions with `EVOLVE-BLOCK` markers
- `evaluator.py` — multi-file evaluator that receives a directory
- `config.yaml` — config with `multi_file.enabled: true`
- `README.md` — brief usage instructions

- [ ] **Step 2: Create integration test**

```python
# tests/test_multi_file_integration.py
import os
import tempfile
import unittest
from openevolve.config import Config, MultiFileConfig
from openevolve.database import Program
from openevolve.project_analyzer import ProjectScanner, DependencyAnalyzer, FileGrouper, DependencyGraph
from openevolve.group_scheduler import GroupScheduler
from openevolve.utils.code_utils import extract_diffs_multi, apply_multi_file_diffs, DiffBlock


class TestMultiFileEndToEnd(unittest.TestCase):
    def test_scan_analyze_group_schedule(self):
        """Full pipeline: scan → analyze deps → group → schedule"""
        tmpdir = tempfile.mkdtemp()
        os.makedirs(os.path.join(tmpdir, "src"))
        with open(os.path.join(tmpdir, "src", "main.py"), "w") as f:
            f.write("from utils import helper\ndef main(): helper()\n")
        with open(os.path.join(tmpdir, "src", "utils.py"), "w") as f:
            f.write("def helper(): return 42\n")
        with open(os.path.join(tmpdir, "src", "standalone.py"), "w") as f:
            f.write("def solo(): pass\n")

        # Scan
        scanner = ProjectScanner()
        files = scanner.scan(tmpdir, ["**/*.py"], [])
        self.assertEqual(len(files), 3)

        # Analyze
        analyzer = DependencyAnalyzer()
        graph = analyzer.analyze(files, "python")

        # Group
        grouper = FileGrouper()
        groups = grouper.create_groups(graph, [])
        self.assertTrue(len(groups) >= 1)

        # Schedule
        scheduler = GroupScheduler(groups)
        g = scheduler.select_group(0)
        self.assertIsNotNone(g)

    def test_multi_file_diff_roundtrip(self):
        """Apply multi-file diffs and verify result"""
        files = {"a.py": "x = 1\ny = 2", "b.py": "z = 3"}
        llm_response = (
            "<<<<<<< SEARCH file=a.py\nx = 1\n=======\nx = 10\n>>>>>>> REPLACE\n"
            "<<<<<<< SEARCH file=b.py\nz = 3\n=======\nz = 30\n>>>>>>> REPLACE\n"
        )
        diffs = extract_diffs_multi(llm_response)
        result = apply_multi_file_diffs(files, diffs)
        self.assertIn("x = 10", result["a.py"])
        self.assertIn("z = 30", result["b.py"])
        self.assertIn("y = 2", result["a.py"])  # untouched line preserved

    def test_program_multi_file_checkpoint_roundtrip(self):
        """Program with files can serialize and deserialize"""
        p = Program(
            id="test", code="main code",
            files={"main.py": "main code", "utils.py": "util code"},
            generation=5, metrics={"score": 0.9},
        )
        d = p.to_dict()
        p2 = Program.from_dict(d)
        self.assertEqual(p2.files, p.files)
        self.assertTrue(p2.is_multi_file)
        self.assertEqual(p2.generation, 5)


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 3: Run integration tests**

Run: `python -m unittest tests/test_multi_file_integration.py -v`
Expected: All 3 tests PASS.

- [ ] **Step 4: Run full test suite**

Run: `python -m unittest discover tests -p "test_*.py" -v 2>&1 | tail -10`
Expected: All tests pass, no regressions.

- [ ] **Step 5: Commit**

```bash
git add examples/multi_file_optimization/ tests/test_multi_file_integration.py
git commit -m "feat: add multi-file example project and integration tests"
```

---

### Task 13: Final Regression Check and Cleanup

- [ ] **Step 1: Run full test suite**

Run: `python -m unittest discover tests -p "test_*.py" -v`
Expected: All tests pass.

- [ ] **Step 2: Run Black formatting**

Run: `python -m black openevolve tests --check`
If formatting issues: `python -m black openevolve tests`

- [ ] **Step 3: Verify single-file example still works**

Run: `python openevolve-run.py examples/function_minimization/initial_program.py examples/function_minimization/evaluator.py --config examples/function_minimization/config.yaml --iterations 1`
Expected: Completes without errors.

- [ ] **Step 4: Final commit if formatting changed**

```bash
git add -u
git commit -m "style: format code with Black"
```
