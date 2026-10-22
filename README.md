# Team No #7 — Semantic File Search Agent - Satvik Sinha and Kshitij
File System & Storage Agents

7. Semantic File Search Agent
Problem Statement - Prompt Example: "Find the config file I edited yesterday that has the database password in it."
Functionality: Replaces complex find and grep pipelines. Searches based on time, metadata, and content.
How to Achieve It: Expose tools like search_by_metadata(time, type) and search_file_content(regex). The LLM converts the user's natural language into the optimal search strategy.




Part of the class Agentic Shell. This agent replaces hand-written `find` and
`grep` pipelines: the user describes a file in their own words, Claude decides
which filesystem tools to run, and the agent answers with the path.

```
$ python -m semantic_search "find the config file I edited yesterday that has the database password in it"

/home/satvik/.config/myapp/settings.conf — modified yesterday at 10:44.
Line 4 sets DB_PASSWORD (value redacted: hunter2…). Two other files were
modified yesterday but contain no credentials, and projects/webapp/old.conf has
a DB_PASSWORD but was last touched 400 days ago.
```

## How it works

```
user text
    │
    ▼
┌──────────────────────────────────────────────┐
│ agent.py — tool-use loop (claude-opus-5)     │
│   system prompt + 6 tool schemas ──► model   │
│   model picks tools ──► dispatch() ──► result│
│   repeat until the model answers in prose    │
└──────────────────────────────────────────────┘
    │  tool calls
    ▼
┌──────────────────────────────────────────────┐
│ registry.py — schemas + dispatch             │
├──────────────────────────────────────────────┤
│ safety.py    sandbox, prunes, caps, timeouts │
│ metadata.py  getdents/stat via os.scandir    │
│ content.py   ripgrep, or mmap in pure Python │
│ timespec.py  "yesterday" → epoch seconds     │
└──────────────────────────────────────────────┘
    │  system calls
    ▼
  live filesystem (read-only)
```

The design decision that matters most is **metadata first, content second**.
Grepping a whole home directory for "password" is slow and returns hundreds of
irrelevant lines. The system prompt pushes the model to run
`search_by_metadata` first — cheap, uses only `stat(2)` — and then pass the
surviving handful of paths to `search_file_content` as a shortlist. On the demo
tree that chain is 12 entries scanned → 2 candidates → 1 answer.

## Tools exposed to the LLM

| Tool | System calls behind it | What the model uses it for |
|---|---|---|
| `search_by_metadata` | `getdents64` via `os.scandir`, `lstat` | Name globs/regex, extensions, mtime/atime windows, size ranges, type, depth |
| `search_file_content` | `ripgrep(1)`, or `read`/`mmap` fallback | Regex over file contents, returning matching lines with line numbers |
| `stat_file` | `lstat`, `readlink` | Size, all three timestamps, mode bits, owner, inode, link count, symlink target |
| `preview_file` | `open`/`read` | Bounded slice of a text file, to confirm a hit before answering |
| `list_directory` | `getdents64` | One non-recursive level, to orient when the user is vague |
| `identify_file` | `file(1)` | Real file type from magic bytes when the extension lies |

Traversal is written directly on `os.scandir` rather than shelling out to
`find(1)` so the agent owns the walk: it can prune pseudo-filesystems, bound
the search by depth and wall-clock time, detect directory cycles via
`(st_dev, st_ino)`, and skip an unreadable directory instead of aborting.

## Setup

```bash
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt

cp .env.example .env        # then put your key in it
```

Use a venv rather than installing system-wide. On a rolling-release distro a
Python point-upgrade (3.11 → 3.14) empties the system `site-packages` and your
imports vanish; a venv is unaffected.

Your `.env` needs one line:

```
ANTHROPIC_API_KEY=sk-ant-...
```

It's gitignored, so it never leaves your machine. An exported
`ANTHROPIC_API_KEY` in your shell takes precedence over the file if you'd
rather not keep a key on disk. `.env.example` lists the optional settings
(`SFS_EFFORT`, `SFS_MAX_ROUNDS`, `SFS_ALLOWED_ROOTS`).

Install `ripgrep` too if you can (`sudo pacman -S ripgrep`) — content search is
roughly 10–50× faster through it. Without it the pure-Python scanner runs
instead; both backends return identical results.

## Running it

```bash
# one-shot
.venv/bin/python -m semantic_search "which log files are bigger than 100MB?"

# interactive, with the tool calls echoed to stderr
.venv/bin/python -m semantic_search -v

# scripted demo against a throwaway tree (never touches your real files)
.venv/bin/python demo.py                 # full agent
.venv/bin/python demo.py --tools-only    # the tool chain with no API calls

# tool tests — real system calls, no API key needed
.venv/bin/python tests/test_tools.py
```

(Or `source .venv/bin/activate` once and drop the `.venv/bin/` prefix.)

## Running in Docker

Useful for a teammate on Windows or macOS, and for a reproducible demo at the
review. The image bundles `ripgrep` and `file`, so content search runs on the
fast backend.

```bash
docker build --build-arg UID=$(id -u) --build-arg GID=$(id -g) -t sfs-agent .
```

**Verify the image** — the tests need no API key, so this is a clean check that
everything inside works:

```bash
docker run --rm sfs-agent
```

**The demo**, fully self-contained (it builds its own throwaway tree):

```bash
docker run --rm sfs-agent python demo.py --tools-only     # no API key
docker run --rm --env-file .env sfs-agent python demo.py  # full agent
```

**Searching your real files.** This is the part that needs care: a container
sees its *own* filesystem, so without a mount the agent will search an empty
image and truthfully report finding nothing. Bind-mount the directory you want
searched at `/mnt/host`, read-only:

```bash
docker run --rm -it \
  --env-file .env \
  -v "$HOME:/mnt/host:ro" \
  sfs-agent \
  python -m semantic_search "find config files I edited this week"
```

- `:ro` makes the read-only design structural — the kernel refuses writes even
  if something in the stack tries. Worth mentioning at the review.
- `SFS_ALLOWED_ROOTS` is already set to `/mnt/host` in the image, so the
  sandbox lines up with the mount with no extra flags.
- Paths in the answers will read `/mnt/host/...`; that is the container's view
  of your `$HOME`.
- `--build-arg UID=$(id -u)` matters — the container user must share your UID
  or your files will be unreadable inside it.

**Interactive REPL:**

```bash
docker run --rm -it --env-file .env -v "$HOME:/mnt/host:ro" sfs-agent \
  python -m semantic_search -v
```

Note that `--env-file .env` is how the key gets in. It is never baked into an
image layer — `.env` is in `.dockerignore` precisely so a published image can't
leak it.

## Safety model

The LLM chooses every argument, so every argument is treated as untrusted:

- **Sandboxed roots.** Paths are resolved *before* the permission check, so a
  symlink cannot be used to escape. Default root is `$HOME`; override with
  `SFS_ALLOWED_ROOTS=/path/one:/path/two`.
- **Never searched.** `/proc`, `/sys`, `/dev`, `/run` and friends (they block on
  kernel reads and contain no user files), plus `.git`, `node_modules`,
  `__pycache__`, `.venv`, `site-packages` and other caches.
- **Bounded output.** 200 entries per metadata call, 100 content matches, 300
  chars per line, 200 lines per preview, 25-second walk timeout, 12 tool rounds
  per request. Every response reports whether a cap was hit, so the model can
  tell the user its answer is partial instead of implying it was exhaustive.
- **Read-only.** No tool writes, moves, or deletes. `.gitignore` is deliberately
  *not* respected — an ignored file is still a file the user may be looking for.
- **Errors come back as data.** `dispatch()` converts a `ToolError` into a
  `tool_result` with `is_error: true` rather than raising, so a bad regex or a
  nonexistent path lets the model correct itself instead of killing the turn.

## Integrating with the base shell

Two options. Either let this agent run its own loop:

```python
from semantic_search import handle
answer = handle(user_text)
```

…or merge its tools into the shell's own tool set and route calls back:

```python
from semantic_search import TOOLS, dispatch, AGENT_NAME, AGENT_DESCRIPTION

all_tools = shell_tools + TOOLS
# when the model calls one of ours:
payload, is_error = dispatch(call.name, call.input)
```

`AGENT_DESCRIPTION` is written for the router's prompt — it states the intents
that should land here ("where is the file that…", "find the config with…",
"what did I edit yesterday").

## Notes on the model configuration

- `claude-opus-5` with `thinking: {"type": "adaptive"}` — the model decides how
  much to reason per query. Set `SFS_EFFORT=low|medium|high|xhigh|max` to trade
  depth against latency; default is `medium`, which is plenty for search.
- Streaming (`messages.stream`) with `max_tokens=64000`, because a non-streaming
  request at that size risks an HTTP timeout.
- The 6 tool schemas and the stable half of the system prompt sit behind a
  `cache_control` breakpoint (~1.5K tokens, above the 512-token minimum), so
  repeated queries in one session re-read that prefix from cache at ~0.1× cost.
  The volatile half — current time, cwd, hostname — is a second system block
  placed *after* the breakpoint, so it never invalidates the cache.

## Layout

```
semantic_search/
  agent.py      tool-use loop, system prompt, CLI and REPL
  config.py     .env loader (no external dependency)
  registry.py   TOOLS schemas + dispatch()
  metadata.py   search_by_metadata, stat_file, list_directory
  content.py    search_file_content, preview_file, identify_file
  safety.py     sandbox, prune lists, caps, regex/type helpers
  timespec.py   time and size expression parsing
tests/test_tools.py   14 tool tests, no API key required
demo.py               scripted demo on a throwaway tree
```
