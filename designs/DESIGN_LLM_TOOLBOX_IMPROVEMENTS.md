# Design: LLM Toolbox Improvements for Truncation and Efficiency

## 1. Objective
Improve Sashiko's LLM review workflow by:
1.  **Eliminating Silent Truncations & LLM Confusion**: Explicitly instructing the LLM to detect and handle truncated tool outputs.
2.  **Improving Token Efficiency**: Actively promoting the use of `smart` mode for `git_read_files` to reduce context bloat.

## 2. Proposed Changes

### 2.1. Truncation Awareness Guidelines
We will modify `build_context` in `src/worker/prompts.rs` to append a new section to the global review guidelines. This section will explicitly instruct the LLM on how to handle truncated outputs from tools.

**New Guideline Text:**
```markdown
### TRUNCATION & PAGINATION MANAGEMENT
Many of your information-gathering tools (such as `git_read_files`, `git_diff`, `git_show`, `git_grep`, `git_log`) will truncate their output if it exceeds token limits to protect the context window.
When truncation occurs, the tool's JSON response will contain:
- `"truncated": true`
- A `"next_page_hint"` explaining how to fetch the next slice of data (usually by specifying `start_line` or narrowing the search).

You MUST actively check for the `"truncated"` flag in every tool response. If `"truncated"` is `true`, you MUST NOT assume you have the complete picture. You are REQUIRED to follow the `"next_page_hint"` and make subsequent tool calls with adjusted parameters (e.g., `start_line`, `end_line`, narrower `paths`, or specific regex) to fetch the remaining content before finalizing your analysis. Failing to retrieve truncated content is a failure of rigor.
```

### 2.2. Promoting `smart` Mode for `git_read_files`
We will update the tool declaration for `git_read_files` in `src/worker/tools.rs` to strongly recommend `smart` mode.

**Changes in `src/worker/tools.rs`:**
*   Update the tool `description` to highlight `smart` mode.
*   Update the `description` of the `mode` parameter to highly recommend `smart` mode for large files.

**Updated Tool Declaration:**
```json
            AiTool {
                name: "git_read_files".to_string(),
                description: "Read the content of one or more files at a specific Git revision. 'smart' mode is HIGHLY RECOMMENDED for large files as it collapses irrelevant code around focus lines to save tokens."
                    .to_string(),
                parameters: json!({
                    "type": "object",
                    "properties": {
                        "revision": { "type": "string", "description": "The Git commit SHA or reference (e.g., HEAD, baseline SHA, or target commit SHA) to read from." },
                        "files": {
                            "type": "array",
                            "description": "List of files to read (maximum 10 files per request).",
                            "items": {
                                ...
                            }
                        },
                        "mode": { "type": "string", "enum": ["raw", "smart"], "description": "Read mode. 'smart' mode is highly recommended to avoid truncation and save tokens. Defaults to 'raw'." }
                    },
                    "required": ["revision", "files"]
                }),
            }
```

## 3. Verification Plan

### 3.1. Compilation & Linting
Ensure the project compiles and passes all linters:
```bash
make lint
make test
```

### 3.2. Manual Verification of Prompt Integration
Verify that the new guidelines are correctly appended to the system context. We can do this by running a test or temporarily logging the built context.

### 3.3. Benchmark (Optional)
Run a small benchmark to ensure no regression in detection rates:
```bash
cargo run --bin benchmark -- --file benchmarks/benchmark_small.json
```
