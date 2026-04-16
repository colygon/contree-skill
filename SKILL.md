---
name: contree
description: "Guide for using ConTree sandboxed container execution with Git-like branching. Use this skill whenever the user mentions ConTree, contree, or contree-mcp, OR wants to run code in isolated sandboxes/containers with branching capabilities, OR the ConTree MCP tools are available (contree_run, contree_rsync, contree_import_image, contree_list_images, contree_upload, contree_download, contree_list_files, contree_read_file, contree_set_tag, contree_get_operation, contree_wait_operations). Also use when the user wants to safely execute untrusted code, branch execution states, explore multiple solution paths in parallel, or do tree-search style coding. Even if they just say 'run this in a sandbox' or 'try multiple approaches in parallel containers' — this skill applies."
---

# ConTree: Sandboxed Execution with Branching

ConTree gives you isolated microVM containers where every execution can be checkpointed, branched, and rolled back — like Git for runtime state. Built by Nebius for AI agents that need to explore, backtrack, and compare.

There are two ways to use ConTree: via **MCP tools** (when the contree-mcp server is connected) or via the **Python SDK** (when writing Python code that manages containers programmatically). This skill covers both.

## Mental Model

- An **image** is a Git commit — an immutable filesystem snapshot identified by a UUID
- Running a command with `disposable=false` is like making a new commit
- **Branching** means running multiple commands from the same image — like creating Git branches from the same commit
- **Tags** are like Git tags — human-readable names for image UUIDs (e.g., `tag:python:3.11`)
- `disposable=true` (the default) is like running in a detached HEAD that gets thrown away
- If a command produces no filesystem changes, the result UUID equals the parent UUID (content-addressed)

---

## Part 1: Using ConTree via MCP Tools

MCP tools are prefixed with `contree_` (e.g., `contree_run`, `contree_list_images`). The MCP server also exposes 10 prompts for common workflows — use `contree_get_guide` to access them.

### The CHECK-PREPARE-EXECUTE Pattern

Always follow this workflow. It prevents redundant imports and wasted compute.

#### 1. CHECK — Do I already have what I need?

Before importing anything, search for existing images:

```json
{"tool": "contree_list_images", "args": {"tag_prefix": "python"}}
{"tool": "contree_list_images", "args": {"tagged": true}}
```

#### 2. PREPARE — Set up the environment (only if needed)

**Import a base image** (only if CHECK found nothing):
```json
{"tool": "contree_import_image", "args": {"registry_url": "docker://python:3.11-slim"}}
```

Note: without registry authentication, you'll need to pass `i_accept_that_anonymous_access_might_be_rate_limited: true` or first authenticate via `contree_registry_token_obtain` + `contree_registry_auth`.

**Install dependencies** — the default mode is `disposable: true`, which throws away all changes after execution. To persist installed packages, you must set `disposable: false`:
```json
{"tool": "contree_run", "args": {
  "command": "pip install numpy pandas pytest",
  "image": "<image-uuid>",
  "disposable": false
}}
```
This returns a `result_image` UUID with the packages baked in.

**Tag it for reuse** so you don't reinstall next time:
```json
{"tool": "contree_set_tag", "args": {
  "image_uuid": "<result-image-uuid>",
  "tag": "common/data-science/python:3.11-slim"
}}
```

Tag convention: `{scope}/{purpose}/{base}:{version}`
- Scope: `common` for shared, or a project name
- Purpose: what's installed (e.g., `data-science`, `web-dev`, `test-env`)

#### 3. EXECUTE — Run your actual work

```json
{"tool": "contree_run", "args": {
  "command": "python /app/main.py",
  "image": "<prepared-image>",
  "directory_state_id": 42,
  "cwd": "/app"
}}
```

### Working with Local Files

Use `contree_rsync` to sync project files into the container. It uses three-tier caching (local cache, content hash, server dedup) so only changed files upload.

```json
{"tool": "contree_rsync", "args": {
  "source": "/path/to/project",
  "destination": "/app",
  "exclude": ["__pycache__", ".git", ".venv", "node_modules", "*.pyc", ".DS_Store"]
}}
```

This returns a **plain integer** (the `directory_state_id`). Reuse it for all subsequent `contree_run` calls — only re-sync when files actually change.

### Inspecting Images (Free, No VM)

`contree_list_files` and `contree_read_file` read directly from the image filesystem without starting a microVM. Always prefer them over `contree_run("ls ...")` or `contree_run("cat ...")`.

```json
{"tool": "contree_list_files", "args": {"image": "<uuid>", "path": "/app"}}
{"tool": "contree_read_file", "args": {"image": "<uuid>", "path": "/app/config.json"}}
```

`list_files` returns entries with `type` (file/directory/symlink), `size`, `mode` (octal like `"0o755"`), and `target` for symlinks. `read_file` returns `content` with `encoding` of `"utf-8"` for text or `"base64"` for binary.

### Uploading Individual Files

For single files (not directories), use `contree_upload` + the `files` parameter on `contree_run`. Upload accepts one of: `content` (text string), `content_base64` (binary), or `path` (local file path).

```json
{"tool": "contree_upload", "args": {"content": "print('hello world')"}}
// Returns: {"uuid": "file-uuid-123"}

{"tool": "contree_run", "args": {
  "command": "python /app/script.py",
  "image": "tag:python:3.11",
  "files": {"/app/script.py": "file-uuid-123"}
}}
```

The `files` mapping: **key = container path (destination), value = UUID from upload**. This is backwards from what you might expect — don't mix them up. The `path` parameter in `upload` reads from the local filesystem — it does NOT set the filename in storage.

### Parallel Execution

Launch multiple runs with `wait: false`, then collect results:

```json
{"tool": "contree_run", "args": {"command": "python approach_a.py", "image": "<img>", "wait": false}}
// Returns: {"operation_id": "op-1"}

{"tool": "contree_run", "args": {"command": "python approach_b.py", "image": "<img>", "wait": false}}
// Returns: {"operation_id": "op-2"}

// Wait for all to finish
{"tool": "contree_wait_operations", "args": {"operation_ids": ["op-1", "op-2"]}}
```

`wait_operations` supports `mode: "all"` (default, wait for everything) or `mode: "any"` (return when first completes — **remaining ops are cancelled**). Default timeout is 300 seconds.

### Branching: Explore Multiple Paths

Create a checkpoint, then branch from it:

```
Base image (python:3.11 + deps)
  |
  +-- run("fix with retry logic", disposable=false)  --> image A
  +-- run("fix with connection pool", disposable=false) --> image B
  +-- run("fix with async", disposable=false)   --> image C
```

Each branch runs in its own microVM from the exact same starting state. Compare results and keep the winner.

**Pattern: Best-of-N**
1. Create a checkpoint with `disposable: false`
2. Fork N times from that checkpoint (same image UUID)
3. Run all branches in parallel with `wait: false`
4. Wait with `contree_wait_operations`
5. Score results, pick the best

### Downloading Results

```json
{"tool": "contree_download", "args": {
  "image": "<result-image>",
  "path": "/app/output/report.pdf",
  "destination": "~/Downloads/report.pdf",
  "executable": false
}}
```

The `destination` is on the local filesystem (MCP host), not inside the container. Parent directories are created automatically. Set `executable: true` to chmod 755.

### Key Parameters for `contree_run`

| Parameter | Type | Default | What it does |
|-----------|------|---------|-------------|
| `command` | string | required | Shell command to execute |
| `image` | string | required | Image UUID or `tag:name` |
| `shell` | boolean | `true` | Whether command is a shell expression |
| `disposable` | boolean | `true` | `false` = save resulting filesystem as new image |
| `directory_state_id` | integer | — | From `contree_rsync`, injects synced files |
| `files` | object | — | Map of `{container_path: upload_uuid}` |
| `wait` | boolean | `true` | `false` = async, returns operation_id |
| `timeout` | integer | `30` | Max seconds before kill |
| `env` | object | — | Environment variables |
| `cwd` | string | `/root` | Working directory (must be absolute) |
| `stdin` | string | — | Input via stdin |
| `truncate_output_at` | integer | `8000` | Max bytes for stdout/stderr |

### MCP Prompts and Guides

The MCP server includes 10 built-in prompts: `prepare-environment`, `run-python`, `run-shell`, `sync-and-run`, `install-packages`, `parallel-tasks`, `build-project`, `debug-failure`, `inspect-image`, `multi-stage-build`.

Use `contree_get_guide` with sections: `workflow`, `reference`, `quickstart`, `state`, `async`, `tagging`, `errors` for detailed guidance.

### Quick Reference: Tool Costs

| Tool | Spawns VM? | Use for |
|------|-----------|---------|
| `contree_run` | Yes (~2-5s) | Executing commands |
| `contree_rsync` | No | Syncing local files (returns int) |
| `contree_import_image` | Yes | Pulling from registry |
| `contree_list_images` | No | Finding existing images |
| `contree_upload` / `contree_download` | No | Single file transfer |
| `contree_list_files` / `contree_read_file` | No | Inspecting image contents |
| `contree_set_tag` / `contree_get_image` | No | Managing image tags |
| `contree_get_operation` / `contree_list_operations` | No | Checking async status |
| `contree_wait_operations` | No | Waiting for async batch |
| `contree_get_guide` | No | Documentation sections |

---

## Part 2: Using ConTree via Python SDK

When the user is writing Python code that manages containers programmatically (not just using MCP tools), guide them with the SDK.

**Install:** `pip install contree-sdk`

**Auth:** Set `CONTREE_TOKEN` and `CONTREE_BASE_URL` env vars, or pass to `ContreeConfig`.

### Two Client Flavors

```python
# Async
from contree_sdk import Contree
client = Contree()  # reads from env vars

# Sync
from contree_sdk import ContreeSync
client = ContreeSync()
```

Both have `.images` (ImagesManager) and `.files` (FilesManager) attributes.

### Getting Images

```python
# Lazy reference — no API call, resolves at execution time
image = await client.images.use("python:3.13-slim")

# Strict — verifies existence via API
image = await client.images.use("python:3.13-slim", strict=True)

# Import from registry if not found locally (preferred for setup)
image = await client.images.oci("docker://python:3.13-slim")

# Force re-import (rarely needed)
image = await client.images.import_from("docker://python:3.13-slim")
```

`images.use()` is lazy and cheap. `images.oci()` checks locally first, imports only if needed — this is the right choice for most setup flows.

### Running Commands

```python
# Async — await triggers execution
result = await image.run(shell="echo hello")
print(result.stdout, result.exit_code)

# Sync — .wait() blocks
result = image.run(shell="echo hello").wait()

# Command mode (no shell interpretation)
result = await image.run("/bin/ls", args=["-la", "/tmp"])

# With environment variables
result = await image.run(shell="echo $MY_VAR", env={"MY_VAR": "hello"})

# Save state for branching
result = await image.run(shell="pip install flask", disposable=False)
# result.uuid is now a new image with flask installed
```

### Sessions vs Images (Critical Distinction)

**Images** return a NEW object on each `.run()` — the original is unchanged. This enables branching.

**Sessions** mutate IN PLACE — each `.run()` updates the session's internal UUID. This is for sequential workflows.

```python
# Branching with images:
base = await image.run(shell="echo setup", disposable=False)
branch_a = await base.run(shell="echo A")  # base unchanged
branch_b = await base.run(shell="echo B")  # base still unchanged

# Sequential with sessions:
session = image.session()
await session.run(shell="pip install flask")   # session.uuid updated
await session.run(shell="flask --version")     # sees flask because session tracked state
```

### Working with Files (SDK)

```python
# Files as a list (placed at / with original filename)
result = await image.run(shell="cat /data.txt", files=["/local/data.txt"])

# Files as a dict (key = destination in container)
result = await image.run(shell="python /app/script.py", files={"script.py": "/local/script.py"})

# Pre-upload for reuse
uploaded = await client.files.upload("/local/file.txt")
result = await image.run(shell="cat /file.txt", files={"file.txt": uploaded})

# Bake files into image without running a command
new_image = await image.apply_files({"config.yml": "/local/config.yml"})

# Read/download from images (no VM)
content: bytes = await image.read("/path/in/container")
local_path = await image.download("/remote/file.txt", "/local/dest.txt")
items = await image.ls("/some/dir")
```

### Tagging (SDK)

```python
tagged = await image.tag_as("myapp:v1")
untagged = await image.untag()

# Tag directly from a run result
result = await image.run(shell="pip install flask", tag="myapp:with-flask", disposable=False)
```

### Subprocess-like Interface (Sync Only)

```python
proc = session.popen(["cat"], text=True)
stdout, stderr = proc.communicate("input data")

proc = session.popen("echo hello && ls", shell=True)
proc.wait()
print(proc.stdout, proc.returncode)
```

### SDK Configuration

```python
from contree_sdk.config import ContreeConfig
config = ContreeConfig(
    base_url="https://contree.dev/",
    token="your-token",
    transport_timeout=10.0,        # HTTP timeout (seconds)
    operation_timeout=1000.0,      # Max wait for operations
    default_truncate_output_at=65535,  # stdout/stderr limit (bytes)
)
client = Contree(config=config)
```

---

## Common Mistakes to Avoid

**Forgetting `disposable=false`** — The #1 mistake. If you install packages or make changes you want to keep, you must pass `disposable=false` (MCP) or `disposable=False` (SDK). Otherwise everything is thrown away.

**Re-importing images** — Always check first. MCP: `contree_list_images`. SDK: `images.oci()` (auto-checks before importing).

**Re-syncing unchanged files** — `rsync` returns a `directory_state_id`. Reuse it for all runs in the session. Only call `rsync` again when local files have actually changed.

**Chaining everything in one command** — Don't do `"apt update && apt install python && pip install numpy && python train.py"`. Run each step separately with `disposable=false` so you get checkpoints. If a later step fails, you can roll back to a known-good image.

**Using `run` to read files** — `list_files`/`read_file` (MCP) or `.ls()`/`.read()` (SDK) are instant and free. Don't spawn a VM just to cat or ls.

**Mixing up the MCP `files` parameter** — Key = destination path inside container, value = UUID from upload. Not the other way around.

**Using `wait_operations` mode="any" without knowing it cancels** — When `mode="any"`, the first completed op wins and all remaining operations are cancelled on the backend.

**SDK: confusing images and sessions** — Images branch (new object per run), sessions are sequential (mutate in place). Use images when you want to explore parallel paths; use sessions for linear workflows.

---

## Setup

### MCP Server Setup

1. Get a token at [contree.dev](https://contree.dev) (Early Access)
2. Create `~/.config/contree/mcp.ini`:
   ```ini
   [DEFAULT]
   url = https://contree.dev/
   token = <TOKEN>
   ```
3. Add the MCP server:
   ```bash
   claude mcp add --transport stdio contree -- uvx contree-mcp
   ```
4. Restart Claude Code or run `/mcp` to verify

Alternatively, pass credentials via env vars: `CONTREE_MCP_TOKEN` and `CONTREE_MCP_URL` (but tokens may appear in process listings).

### Python SDK Setup

```bash
pip install contree-sdk
```

Set environment variables:
```bash
export CONTREE_TOKEN="your-token"
export CONTREE_BASE_URL="https://contree.dev/"
```

Or pass directly to `ContreeConfig` / client constructor.
