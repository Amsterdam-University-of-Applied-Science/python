# Publish Content to Brightspace — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `publish-content` CLI command to `brightspace-extractor` that pushes local HTML files from `python1/lessen/` to their corresponding Brightspace content topic pages via Playwright CDP.

**Architecture:** A `content-map.toml` in the `python1` repo maps each HTML file to a Brightspace class ID and topic ID. A new `content.py` module in `brightspace-extractor` handles the Playwright interaction: navigating to the topic edit page, opening the HTML source editor, replacing its content, and saving. A new `publish-content` CLI command wires it together using the same CDP browser connection pattern as existing commands.

**Tech Stack:** Python 3.14, Playwright (sync API via CDP), cyclopts, structlog, tomllib

**Spec:** Conversational design approved in this session.

## Global Constraints

- Follow existing `brightspace-extractor` patterns: cyclopts CLI, structlog logging, CDP connection via `browser.py`, navigation via `navigation.py`
- Base URL: `https://dlo.mijnhva.nl`
- `content-map.toml` lives in `python1` repo (not in `finance-tools`)
- HTML files are self-contained pages — push the full file content into Brightspace's HTML editor
- Support `--dry-run` to log without making changes

## Review Focus

1. **Brightspace DOM selectors** — the HTML source editor selectors may change between Brightspace versions; verify against live Brightspace before shipping
2. **Large HTML content** — files should paste correctly even when large; avoid character-by-character typing (use `fill()` or JS injection)
3. **Save confirmation** — ensure the save actually completes before navigating to the next page; wait for success indicator
4. **Session timeout** — if pushing 7 pages takes too long, the Brightspace session may expire mid-run
5. **Partial failure** — if page 4 of 7 fails, report which succeeded and which failed; don't silently stop

---

### Task 1: Create `content-map.toml` in python1 repo

**Files:**
- Create: `content-map.toml` (in `python1` repo root)

**Interfaces:**
- Consumes: nothing
- Produces: TOML config with `class_id`, `base_url`, and `[[pages]]` array entries each having `file` and `topic_id`

- [ ] **Step 1: Create `content-map.toml`**

```toml
# Brightspace content mapping for Minor Big Data — Python 1
# Each [[pages]] entry maps a local HTML file to a Brightspace topic.

class_id = "764447"
base_url = "https://dlo.mijnhva.nl"

[[pages]]
file = "python-les1-1.html"
topic_id = "2957790"

[[pages]]
file = "python-les1-2.html"
topic_id = "2957796"

[[pages]]
file = "python-les1-3.html"
topic_id = "2957795"

[[pages]]
file = "python-les1-4.html"
topic_id = "2957799"

[[pages]]
file = "python-les1-5.html"
topic_id = "2957801"

[[pages]]
file = "python-les1-6.html"
topic_id = "2957802"

[[pages]]
file = "python-les1-7.html"
topic_id = "2957804"
```

- [ ] **Step 2: Commit**

```bash
git add content-map.toml
git commit -m "feat: add Brightspace content mapping for python1 lessons"
```

---

### Task 2: Add `navigate_to_content_topic()` to navigation.py

**Files:**
- Modify: `apps/connectors/brightspace/src/brightspace_extractor/navigation.py`
- Test: `apps/connectors/brightspace/tests/test_content.py`

**Interfaces:**
- Consumes: `_navigate_to()` and `_DEFAULT_BASE_URL` from `navigation.py`
- Produces: `navigate_to_content_topic(page: Page, class_id: str, topic_id: str, *, base_url: str) -> None`

- [ ] **Step 1: Write the failing test**

```python
def test_navigate_to_content_topic_builds_correct_url(mocker):
    """navigate_to_content_topic calls _navigate_to with the correct URL."""
    mock_nav = mocker.patch("brightspace_extractor.navigation._navigate_to")
    mock_page = mocker.MagicMock()
    navigate_to_content_topic(mock_page, "764447", "2957790")
    mock_nav.assert_called_once_with(
        mock_page,
        "https://dlo.mijnhva.nl/d2l/le/lessons/764447/topics/2957790",
        mocker.ANY,
    )
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/connectors/brightspace && uv run pytest tests/test_content.py::test_navigate_to_content_topic_builds_correct_url -v`
Expected: FAIL — `ImportError`

- [ ] **Step 3: Implement `navigate_to_content_topic()` in `navigation.py`**

Add function following the same pattern as `navigate_to_rubrics()`. URL pattern: `{base_url}/d2l/le/lessons/{class_id}/topics/{topic_id}`

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/connectors/brightspace && uv run pytest tests/test_content.py::test_navigate_to_content_topic_builds_correct_url -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/brightspace_extractor/navigation.py tests/test_content.py
git commit -m "feat: add navigate_to_content_topic to navigation module"
```

---

### Task 3: Create `content.py` — the Playwright content publishing logic

**Files:**
- Create: `apps/connectors/brightspace/src/brightspace_extractor/content.py`
- Test: `apps/connectors/brightspace/tests/test_content.py` (extend)

**Interfaces:**
- Consumes: `navigate_to_content_topic()` from Task 2, `Page` from Playwright
- Produces:
  - `ContentPage(file: str, topic_id: str)` — dataclass
  - `ContentMap(class_id: str, base_url: str, pages: list[ContentPage])` — dataclass
  - `load_content_map(path: Path) -> ContentMap` — parses `content-map.toml`
  - `PublishResult(topic_id: str, file: str, status: str, error: str | None)` — dataclass
  - `publish_content_page(page: Page, html_content: str, class_id: str, topic_id: str, *, base_url: str, dry_run: bool) -> PublishResult` — navigates to topic, opens HTML editor, replaces content, saves
  - `publish_all_content(page: Page, content_map: ContentMap, html_dir: Path, *, dry_run: bool) -> list[PublishResult]` — iterates over all pages

- [ ] **Step 1: Write failing test for `load_content_map()`**

```python
def test_load_content_map_parses_toml(tmp_path):
    toml_file = tmp_path / "content-map.toml"
    toml_file.write_text('''
class_id = "764447"
base_url = "https://dlo.mijnhva.nl"

[[pages]]
file = "les1.html"
topic_id = "123"

[[pages]]
file = "les2.html"
topic_id = "456"
''')
    result = load_content_map(toml_file)
    assert result.class_id == "764447"
    assert result.base_url == "https://dlo.mijnhva.nl"
    assert len(result.pages) == 2
    assert result.pages[0].file == "les1.html"
    assert result.pages[0].topic_id == "123"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/connectors/brightspace && uv run pytest tests/test_content.py::test_load_content_map_parses_toml -v`
Expected: FAIL — `ImportError`

- [ ] **Step 3: Implement dataclasses and `load_content_map()` in `content.py`**

Use `tomllib.loads()` to parse, construct `ContentMap` from the dict. Raise `ConfigError` on missing keys.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/connectors/brightspace && uv run pytest tests/test_content.py::test_load_content_map_parses_toml -v`
Expected: PASS

- [ ] **Step 5: Write failing test for `publish_content_page()` in dry-run mode**

```python
def test_publish_content_page_dry_run_does_not_navigate(mocker):
    mock_page = mocker.MagicMock()
    mock_nav = mocker.patch("brightspace_extractor.content.navigate_to_content_topic")
    result = publish_content_page(
        mock_page, "<h1>Hello</h1>", "764447", "123",
        base_url="https://dlo.mijnhva.nl", dry_run=True,
    )
    mock_nav.assert_not_called()
    assert result.status == "dry_run"
    assert result.topic_id == "123"
```

- [ ] **Step 6: Run test to verify it fails**

Expected: FAIL — `ImportError`

- [ ] **Step 7: Implement `publish_content_page()`**

Logic:
- If `dry_run`: log and return `PublishResult(status="dry_run")`
- Otherwise:
  1. Call `navigate_to_content_topic()`
  2. Find and click the edit/HTML source button (Brightspace's `<>` or "Edit HTML" button — **selectors TBD, need live verification**)
  3. Clear the editor and inject HTML content via `page.evaluate()` (JS injection is more reliable than `fill()` for rich-text editors)
  4. Click Save
  5. Wait for save confirmation
  6. Return `PublishResult(status="success")`
  7. On exception: return `PublishResult(status="failed", error=str(exc))`

- [ ] **Step 8: Run tests to verify they pass**

Run: `cd apps/connectors/brightspace && uv run pytest tests/test_content.py -v`
Expected: PASS

- [ ] **Step 9: Implement `publish_all_content()`**

Iterates `content_map.pages`, reads each HTML file from `html_dir`, calls `publish_content_page()`, collects results. Continues on failure (doesn't stop at first error).

- [ ] **Step 10: Commit**

```bash
git add src/brightspace_extractor/content.py tests/test_content.py
git commit -m "feat: add content publishing module with TOML mapping"
```

---

### Task 4: Add `publish-content` CLI command

**Files:**
- Modify: `apps/connectors/brightspace/src/brightspace_extractor/cli.py`
- Test: `apps/connectors/brightspace/tests/test_content.py` (extend)

**Interfaces:**
- Consumes: `load_content_map()`, `publish_all_content()`, `PublishResult` from Task 3; `_connect_and_verify()`, `_managed_browser()`, `_setup_logging()`, `_resolve_common()` from `cli.py`
- Produces: `publish_content` CLI command registered on `app`

- [ ] **Step 1: Write failing test for CLI command registration**

```python
def test_publish_content_command_is_registered():
    from brightspace_extractor.cli import app, publish_content
    assert callable(publish_content)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/connectors/brightspace && uv run pytest tests/test_content.py::test_publish_content_command_is_registered -v`
Expected: FAIL — `ImportError`

- [ ] **Step 3: Implement `publish-content` command in `cli.py`**

```python
@app.command
def publish_content(
    content_map_file: Annotated[str, cyclopts.Parameter(help="Path to content-map.toml")],
    *,
    html_dir: Annotated[str, cyclopts.Parameter(help="Directory containing HTML files")] = ".",
    config: Annotated[str | None, cyclopts.Parameter(help="Path to config TOML file")] = None,
    cdp_url: Annotated[str | None, cyclopts.Parameter(help="Playwright CDP endpoint")] = None,
    dry_run: Annotated[bool, cyclopts.Parameter(help="Log without making changes")] = False,
) -> None:
```

Implementation:
1. `_setup_logging()`
2. Load content map from `content_map_file`
3. Connect to browser via CDP
4. Call `publish_all_content()`
5. Log summary: N succeeded, M failed
6. `sys.exit(1)` if any failed

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/connectors/brightspace && uv run pytest tests/test_content.py -v`
Expected: PASS

- [ ] **Step 5: Run full test suite**

Run: `cd apps/connectors/brightspace && uv run pytest -v`
Expected: all existing tests still PASS

- [ ] **Step 6: Commit**

```bash
git add src/brightspace_extractor/cli.py tests/test_content.py
git commit -m "feat: add publish-content CLI command"
```

---

## ⚠️ Important: Live Verification Required

Task 3 Step 7 contains a placeholder for Brightspace DOM selectors. Before this is production-ready, we need to:

1. Open Edge with `--remote-debugging-port=9222`
2. Log in to Brightspace
3. Navigate to one topic page manually
4. Inspect the DOM to find the exact selectors for:
   - The "Edit" or "Edit HTML" button on a content topic
   - The HTML source editor toggle (`<>` button)
   - The HTML source textarea/editor element
   - The "Save" button
5. Update the selectors in `content.py`

This is a ~15 minute exercise once the code is in place.
