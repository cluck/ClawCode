# QMD — optional semantic memory backend

QMD ([github.com/tobi/qmd](https://github.com/tobi/qmd)) is an external tool that adds local semantic search with embeddings on top of your memory files. ClawCode uses the builtin SQLite+FTS5 backend by default; QMD is an opt-in upgrade.

**It's local.** No API keys. Models run on your machine (`node-llama-cpp` + `llama.cpp` bundled).

## Why you might want it

Builtin (FTS5+BM25) does keyword matching. It works great for:

- Exact phrases (`"shrimp allergy"`)
- Named entities (`Cookie`, `Eva`, `blue`)
- Terms that appear verbatim in memory

Where it struggles:

- Paraphrases (`"what does my dog eat"` doesn't match `"Cookie loves carrots"` without the bilingual synonym for `dog`)
- Conceptual similarity (`"breakfast ideas"` doesn't match `"I had pancakes"`)
- Long, multi-concept questions

QMD with vector embeddings handles paraphrase and concept proximity out of the box. If your memory is dense and you ask broad questions, QMD pulls its weight.

## What it adds

- **Embedding index** of your memory files, built once and updated on change
- **Vector + BM25 hybrid search** (configurable)
- **Reranking** with a small cross-encoder for top-N refinement (in `vsearch` and `query` modes)
- **Session transcript indexing** (optional)

All local. All offline after the one-time model download (~100 MB).

## Install

```
bun install -g @tobilu/qmd
```

Or see [github.com/tobi/qmd/releases](https://github.com/tobi/qmd/releases) for binary downloads.

Verify:

```
bun --version
```

First run downloads the embedding model — takes a minute, cached permanently at `~/.cache/qmd/`.

## Enable

Via agent_config:

```
agent_config(action='set', key='memory.backend', value='qmd')
```

Or edit `agent-config.json`:

```json
{
  "memory": {
    "backend": "qmd",
    "citations": "auto",
    "qmd": {
      "searchMode": "vsearch",
      "includeDefaultMemory": true,
      "limits": {
        "maxResults": 6,
        "timeoutMs": 15000
      }
    }
  }
}
```

Then run `/mcp` — `memory.backend` is a **critical** config key and requires a reload (see [config-reload.md](config-reload.md)).

Confirm with `/agent:doctor` — the QMD check should show ✅.

## Search modes

| Mode | Speed | Quality | What it does |
|---|---|---|---|
| `search` | Fast | Good | Basic vector + BM25 hybrid |
| `vsearch` | Medium | Excellent | Vector search with cross-encoder reranking — **recommended default** |
| `query` | Slow | Best | Full query expansion + rerank |

Default: `vsearch`. Change with:

```
agent_config(action='set', key='memory.qmd.searchMode', value='query')
```

`searchMode` is **hot-reloadable** — no `/mcp` needed.

## Fallback behavior

QMD is resilient. If any call fails (qmd binary not found, timeout, model download failed), ClawCode transparently falls back to the builtin FTS5 search. The user never sees an error — they just get keyword-quality results for that query.

To check whether you're actually using QMD:

- `agent_status` reports `Memory backend: QMD (<mode>)` or `builtin (SQLite + FTS5)`
- `memory_search` results are tagged with the backend that produced them

If you expect QMD but keep seeing builtin, run `/agent:doctor` — it probes the binary and reports why.

## Config keys

```json
{
  "memory": {
    "backend": "qmd",
    "qmd": {
      "command": "qmd",
      "searchMode": "vsearch",
      "includeDefaultMemory": true,
      "sessions": {
        "enabled": false,
        "retentionDays": 30
      },
      "update": {
        "interval": "5m",
        "debounceMs": 1000,
        "embedTimeoutMs": 30000
      },
      "limits": {
        "maxResults": 6,
        "timeoutMs": 15000
      }
    }
  }
}
```

| Key | Default | Meaning |
|---|---|---|
| `command` | `"qmd"` | Path to the binary. Use absolute path if not in PATH. |
| `searchMode` | `"vsearch"` | `"search"` / `"vsearch"` / `"query"` |
| `includeDefaultMemory` | `true` | Index `memory/*.md` + `MEMORY.md` (should always be on) |
| `sessions.enabled` | `false` | Index Claude Code session transcripts too |
| `sessions.retentionDays` | — | Drop session entries older than this |
| `update.interval` | `"5m"` | How often QMD re-scans for changes |
| `update.debounceMs` | — | Delay before re-indexing a changed file |
| `update.embedTimeoutMs` | — | Timeout per embedding call |
| `limits.maxResults` | `6` | Cap on results returned |
| `limits.timeoutMs` | `15000` | Overall search timeout |

## How ClawCode talks to QMD

`lib/qmd-manager.ts` shells out to the `qmd` CLI. Queries go over stdin, results come back as JSON on stdout. We parse, map to the same `SearchResult` shape as builtin, and return them through the same `searchMemory` entry point — so code that calls `memory_search` has no idea which backend served the result.

When both backends would find something, QMD wins — we only fall back to builtin when QMD returns empty or errors.

## First-run cost

| Operation | Cost |
|---|---|
| Binary download / install | ~20 MB |
| Embedding model download (first run) | ~100 MB, one-time |
| Index build (first search) | Depends on memory size — typically seconds |
| Per-query latency | 100-500ms for `search`, 1-3s for `vsearch`, 2-10s for `query` |

On modern laptops, `vsearch` feels interactive. On older hardware, `search` is safer.

## Use it with builtin (both on)

You can't literally run both backends for the same query — `memory.backend` is a single value. But:

- Builtin is always initialized, even when `backend: "qmd"` (for fallback)
- QMD's search uses the same memory files, no conflict
- Switching between them requires `/mcp`

## Limitations

- **Not part of the npm install.** You have to install `qmd` separately.
- **No Windows support yet** — qmd targets macOS and Linux.
- **Embeddings have a size threshold.** Very short chunks (< 5 words) don't embed usefully — QMD may silently skip them.
- **Reranking latency is real.** `query` mode isn't for interactive use on slow hardware.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `/agent:doctor` shows QMD `❌ binary not found` | `qmd` not in PATH, or different command name | Install via `bun install -g @tobilu/qmd`, or set `memory.qmd.command` to the absolute path |
| Queries fall back to builtin silently | QMD timeout or error | Check logs, increase `limits.timeoutMs`, try `searchMode: "search"` for speed |
| First search takes 60s+ | Model download in progress | One-time. Subsequent queries fast. |
| `agent_status` shows `QMD (vsearch)` but results look keyword-ish | QMD serving fallback internally | Restart with `/mcp`, check memory has been indexed |
| Changed `memory.backend` but still see `builtin` | Didn't `/mcp` after change | Critical key — requires reload. Run `/mcp` now. |
| `sessions.enabled: true` exploding memory | Too many transcripts indexed | Set `retentionDays` or disable sessions |
