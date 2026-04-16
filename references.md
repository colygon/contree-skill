# References

Every source consulted while building this skill, organized by surface. If a claim in `SKILL.md` or the PRD seems specific, it came from one of these.

---

## Primary sources: Nebius ConTree

### Landing pages & marketing

- **[contree.dev](https://contree.dev)** — Product landing. Covers the "sandboxes that branch like Git" framing, feature bullets, branching patterns (beam search, best-of-N), concept glossary (Image, Instance, Operation, Tag, Inspect, Disposable), FAQ, and comparison to Docker.
- **[contree.dev/blog](https://contree.dev/blog)** — Blog index. Five posts as of 2026-04-16:
  - *Introducing ConTree: Sandboxes That Branch Like Git* (Feb 19, 2026) — tagged architecture, agents
  - *ConTree SDK for Python Public Release* (Feb 16, 2026) — tagged sdk, quickstart
  - *ConTree MCP: Sandboxed Execution for Agents* (Feb 9, 2026) — tagged agents, quickstart
  - *Why ConTree Is Needed for Agents* (Jan 31, 2026) — tagged architecture, agents
  - *ConTree: Branching Sandboxes for Agents* (Jan 12, 2026) — tagged architecture

### MCP server documentation

Hosted at `docs.contree.dev` (mcp subpath, Sphinx + Furo theme). Pages consulted:

- **Home / Overview** — "Isolated cloud container execution for AI agents," HTTP mode, security pointer
- **[Quickstart](https://docs.contree.dev/mcp/quickstart)** — prerequisites, `~/.config/contree/mcp.ini` setup, `claude mcp add --transport stdio contree -- uvx contree-mcp`, first container walkthrough
- **[Concepts](https://docs.contree.dev/mcp/concepts)** — Core concepts + workflows; execution model diagram (import_image → run disposable=false → child image → branch from any ancestor)

#### Tools reference (17 tools)

- **run** — command execution in microVM. Full parameter table with defaults (`shell=true`, `disposable=true`, `timeout=30`, `truncate_output_at=8000`). Covers `files` parameter direction (key=container path, value=UUID), `directory_state_id`, async via `wait=false`.
- **rsync** — local file sync with 3-tier caching (local cache → content hash → server dedup). Returns plain integer `directory_state_id`.
- **import_image** — OCI registry import. Requires prior `registry_auth` OR the `i_accept_that_anonymous_access_might_be_rate_limited` escape hatch.
- **registry_token_obtain** — opens browser for PAT creation. Supports docker.io, ghcr.io, registry.gitlab.com, gcr.io.
- **registry_auth** — validates via OCI `/v2/` API, stores in SQLite cache.
- **list_images** — with `tag_prefix`, `tagged`, `since`, `until` filters.
- **get_image** — resolve UUID or `tag:name`.
- **set_tag** — set or remove tags. Tags are unique per account; reassigning moves the tag.
- **upload** — accepts `content` (text), `content_base64` (binary), or `path` (local file). Returns `{uuid, sha256}`. Content-addressable (no filename stored).
- **download** — extract files from image to MCP host filesystem. Supports `executable=true` for chmod 755.
- **list_files** — zero-VM directory listing. Returns `{name, path, type, size, mode, target}` per entry.
- **read_file** — zero-VM file read. `encoding` is `"utf-8"` or `"base64"`.
- **get_operation** — async op status with `operation_kind` ("instance" or "image_import"), result payload.
- **list_operations** — filters by status, kind, time range.
- **wait_operations** — `mode="all"` or `mode="any"`. Critical detail: `mode="any"` cancels remaining ops.
- **cancel_operation** — cancels a running op.
- **get_guide** — serves 7 static guide sections (workflow, reference, quickstart, state, async, tagging, errors).

#### Prompts reference (10 prompts)

- **prepare-environment** — full CHECK-PREPARE-EXECUTE flow with auto-tag generation (`{scope}/{purpose}/{base}:{tag}`)
- **run-python** — single Python snippet in container
- **run-shell** — shell command in container
- **sync-and-run** — rsync + run with default exclusions
- **install-packages** — install + tag for reuse
- **parallel-tasks** — concurrent execution with `wait_operations`
- **build-project** — sync + install + test workflow
- **debug-failure** — diagnose failed operation
- **inspect-image** — explore container contents, prefers `list_files`/`read_file` over `run`
- **multi-stage-build** — build with rollback checkpoints at each stage

#### Other MCP doc pages

- **[Patterns](https://docs.contree.dev/mcp/patterns)** — canonical workflows (run Python with local files, install & save, parallel execution, build & extract, rollback after failure) + common mistakes list
- **[Cheatsheet](https://docs.contree.dev/mcp/cheatsheet)**
- **[Integration](https://docs.contree.dev/mcp/integration)**
- **[Reference](https://docs.contree.dev/mcp/reference)**
- **[Reporting Security Issues](https://docs.contree.dev/mcp/security)**

### Python SDK documentation

Hosted at `docs.contree.dev/sdk`. Pages consulted:

- **[Python ConTree SDK](https://docs.contree.dev/sdk)** — landing, quick start, basic async/sync usage
- **[Getting Started](https://docs.contree.dev/sdk/getting-started)**
- **[Working with Images](https://docs.contree.dev/sdk/images)** — `images.use()` (lazy, no API call), `images.use(strict=True)`, `images.oci()`, `images.import_from()`. What `ref` can be: UUID, OCI tag, full OCI URL, `OCIReference` object. Tagging with `tag_as()`, `untag()`, and `run(..., tag=...)`.
- **[Running Commands](https://docs.contree.dev/sdk/running-commands)** — shell vs command mode, env vars, files (list/dict/`UploadFileSpec`), I/O types (StringIO, BytesIO, file handles, PIPE, bytes), subprocess-like `popen` interface (sync only).
- **[Branching Workflows](https://docs.contree.dev/sdk/branching)** — why branching matters, simple branching (same parent → multiple children), advanced patterns (same command twice = same UUID, content-addressed).
- **API Reference** — dataclasses for `ContreeConfig`, exception hierarchy (`ContreeError` → `ContreeApiError`, `ContreeImageStateError`, `OperationTimedOutError`, etc.).
- **[Mini-SWE-Agent Integration](https://docs.contree.dev/sdk/mini-swe-agent)** — `ContreeEnvironment` from `minisweagent.environments.extra.contree`, available in mini-swe-agent v2.2.0+. Setup: `pip install "mini-swe-agent[contree]"`, `CONTREE_TOKEN` and `CONTREE_BASE_URL` env vars, run with `--environment-class contree`.
- **[About ConTree](https://docs.contree.dev/sdk/about)**

### Main ConTree docs

- **[ConTree for SWE Agents](https://docs.contree.dev)** — positioning for SWE agent research, 7,000+ preloaded SWE-bench Verified + SWE-rebench environments, MCTS/beam search/rollback framing, VM-level isolation claims.
- **[REST API Reference](https://docs.contree.dev/api)** — Base URL `https://api.contree.dev/v1`. Endpoints:
  - Images: `GET /images`, `POST /images/import`, `PATCH /images/{uuid}/tag`, `DELETE /images/{uuid}/tag`
  - Files: `POST /files`, `HEAD /files?uuid=`, `GET /files?sha256=`
  - Instances: `POST /instances`
  - Operations: `GET /operations`, `GET /operations/{id}`, `DELETE /operations/{id}`
  - Inspection: `GET /inspect/?tag=`, `GET /inspect/{uuid}/`, `GET /inspect/{uuid}/list`, `GET /inspect/{uuid}/download`, `HEAD /inspect/{uuid}/`
- **Interactive sandbox** — [contree.dev/v1/](https://contree.dev/v1/) (Swagger UI)
- **OpenAPI spec** — [contree.dev/static/api.yaml](https://contree.dev/static/api.yaml)

---

## Source code (read directly for ground truth)

### contree-mcp (nebius/contree-mcp)

- **Repo:** [github.com/nebius/contree-mcp](https://github.com/nebius/contree-mcp)
- **License:** Apache 2.0
- **Language:** Python
- **Version at research time:** v0.1.1

Files read or inspected via `gh api repos/nebius/contree-mcp/contents`:
- `README.md` — install instructions, tool list with `contree_` prefix
- `contree_mcp/app.py` — module docstring used as MCP server instructions (embedded system prompt with CHECK-PREPARE-EXECUTE)
- `contree_mcp/tools/*.py` — exact tool signatures, confirmed defaults differ from backend model (`shell=True` vs `False`, `timeout=30` vs `60`, `truncate_output_at=8000` vs 64KB)
- `contree_mcp/prompts.py` — 10 prompt definitions
- `contree_mcp/resources/*` — 5 dynamic + 7 static resource templates, `PathResourceTemplate` subclass for slash-tolerant paths
- Config handling — `~/.config/contree/mcp.ini`, `CONTREE_MCP_CONFIG` env override
- CLI — `contree-mcp` entry point, `--mode stdio|http`, `--http-port 9452`, cache at `~/.cache/contree_mcp/{files,cache}.db`
- Transport — `stdio` default, `http` via Uvicorn at `/mcp` with docs at `/`
- Context management — `StrictContextVar` wrapper, `ContextMiddleware` for HTTP mode

### contree-sdk (nebius/contree-sdk)

- **Repo:** [github.com/nebius/contree-sdk](https://github.com/nebius/contree-sdk)
- **License:** Apache 2.0
- **Language:** Python 3.10–3.14
- **Version at research time:** `0.3.0.dev1` (pre-alpha)
- **PyPI:** [pypi.org/project/contree-sdk](https://pypi.org/project/contree-sdk/)
- **Dependencies:** `httpx`, `cattrs`, `aiofiles`, `strenum`

Surfaces confirmed from source:
- Top-level exports: `Contree`, `ContreeSync`, `ContreeConfig`, `OCIReference`, exception classes
- `ContreeConfig` fields: `base_url`, `token`, `transport_timeout=10.0`, `operation_timeout=1000.0`, `operation_import_timeout`, `operation_run_timeout`, `file_upload_chunk_size=1MB`, `default_truncate_output_at=65535`
- Manager attributes: `client.images` (ImagesManager), `client.files` (FilesManager)
- Object hierarchy: `_ImageLikeBase` → `_ImageLike` (async) / `_ImageLikeSync` (sync) → `ContreeImage` / `ContreeSession` (+ sync variants)
- Sessions mutate in place (`_copy_self` returns self); images return new objects
- `run()` parameters: `command`, `shell`, `args`, `env`, `cwd`, `hostname`, `stdin`, `stdout`, `stderr`, `tag`, `files`, `timeout`, `disposable=True`, `truncate_output_at`
- `popen()` subprocess-like interface on sync variants
- `apply_files()` to bake files into image without running a command
- Exceptions: `ContreeError`, `ContreeApiError`, `ApiStatusCodeError`, `ApiTimeoutError`, `ForbiddenError`, `NotFoundError`, `ContreeImageStateError`, `DisposableImageRunError`, `ContreeImageNotFoundError`, `OperationTimedOutError`, `FailedOperationError`, `CancelledOperationError`
- Example directory: `examples/mini_swe_agent/` — shows integration pattern

### External integrations referenced

- **mini-swe-agent** — [github.com/SWE-agent/mini-swe-agent](https://github.com/SWE-agent/mini-swe-agent) — the lightweight SWE agent with `ContreeEnvironment` support as of v2.2.0. Not read in detail; noted as a consumer pattern.

---

## Reference materials about Claude Code Skills

Used for the skill's structure and triggering design.

- **[Claude Code Skills documentation](https://docs.claude.com/en/docs/claude-code/skills)** — the feature itself
- **Anthropic `skill-creator` skill** — local skill used to scaffold this project. Its guide covers:
  - Progressive disclosure (metadata → SKILL.md → bundled resources)
  - Description-first triggering ("pushy" style recommended)
  - `run_loop.py` for description optimization via 60/40 train/test split on trigger evals
  - Eval viewer (`generate_review.py`) for human review cycles
  - `package_skill.py` for `.skill` file packaging
- **References bundled with skill-creator:**
  - `references/schemas.md` — JSON schemas for evals.json, grading.json, benchmark.json
  - `agents/grader.md` — assertion evaluation
  - `agents/analyzer.md` — benchmark analysis
  - `agents/comparator.md` — blind A/B comparison

---

## Pages I pasted into the conversation (verbatim captures)

The initial research prompt included verbatim text from these pages, which I used as primary evidence. Grouped here for traceability:

1. **contree.dev landing page** — hero section through FAQ
2. **contree.dev/blog index** — 5 blog post previews with tags
3. **docs.contree.dev MCP: Quickstart** — the full quickstart flow
4. **docs.contree.dev MCP: Overview** — "Why Contree?", quick example, HTTP mode, security
5. **docs.contree.dev MCP: Concepts** — overview table and mental model
6. **docs.contree.dev MCP: Tools Reference** — cost-annotated table for all 17 tools
7. **docs.contree.dev MCP: each tool** — individual pages for `run`, `rsync`, `import_image`, `registry_token_obtain`, `registry_auth`, `list_images`, `get_image`, `set_tag`, `upload`, `download`, `list_files`, `read_file`, `get_operation`, `list_operations`, `wait_operations`, `cancel_operation`, `get_guide`
8. **docs.contree.dev MCP: Prompts Reference** — 10 prompt definitions with full generated instruction bodies
9. **docs.contree.dev SDK: Home** — Quick Start, installation, basic usage
10. **docs.contree.dev SDK: Working with Images** — use/oci/import_from/tagging/listing
11. **docs.contree.dev SDK: Running Commands** — basic, command mode, files, advanced I/O, popen
12. **docs.contree.dev SDK: Branching Workflows** — why branching, simple and advanced examples
13. **docs.contree.dev SDK: Mini-SWE-Agent Integration** — usage + setup
14. **docs.contree.dev main: ConTree for SWE Agents** — SWE-bench positioning, 7,000+ environments
15. **docs.contree.dev main: REST API Reference** — full endpoint list with params and responses

---

## Research methodology

1. **Read everything Nebius publishes** — all public-facing docs and blog posts, cross-referenced with the landing page's framing.
2. **Spawn two parallel research agents** to read the `contree-mcp` and `contree-sdk` repos directly, with explicit instructions to find **discrepancies between docs and source**. This caught the `run` default mismatch (timeout 30 vs 60), the plain-integer return type of `rsync`, and the SDK's pre-alpha version.
3. **Verify live against the service** by registering the MCP server with a working token and confirming health check passes (`claude mcp list` → `✓ Connected`).
4. **Cross-check every SKILL.md claim** against at least two of: the docs, the source, or direct MCP interaction.

---

## What I did NOT consult (yet, but probably should)

- **ConTree Discord / community channels** — unknown if one exists publicly
- **Nebius's internal SWE-bench benchmarks** — referenced in marketing but not linked publicly
- **Upstream microVM runtime** — presumably Firecracker or Nebius-internal; claims about "hardware-level isolation" taken at face value
- **Actual `contree_run` in a live session** — MCP server is registered but not exercised end-to-end in this workspace yet (pending Claude Code restart to pick up the new user-scoped server)
- **Pricing page** — didn't find one; ConTree is Early Access, pricing likely negotiated
