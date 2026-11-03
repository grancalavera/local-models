# Local LLM on a Mac

This setup runs a local LLM on Apple Silicon. LM Studio serves the model as an OpenAI-compatible API on `http://localhost:1234/v1`.

Two files define the whole setup:

- **`Brewfile`** installs [mise](https://mise.jdx.dev) and [LM Studio](https://lmstudio.ai).
- **`mise.toml`** holds the configuration as environment variables, plus tasks that wrap LM Studio's `lms` CLI.

## The model

- **Qwen3.6-35B-A3B**, MLX 4-bit build ([`lmstudio-community/Qwen3.6-35B-A3B-MLX-4bit`](https://huggingface.co/lmstudio-community/Qwen3.6-35B-A3B-MLX-4bit)), about 20 GB on disk.
- It is a mixture-of-experts model. It stores 35B parameters but computes only about 3B per token, so it runs much faster than a dense model of the same size.
- MLX is Apple's machine learning framework, so the model runs natively on the Mac's GPU.

This setup was built and tested on a MacBook Pro with an M2 Pro and 32 GB of RAM.

## Requirements

- A Mac with Apple Silicon. 32 GB of RAM is a practical minimum for this model.
- About 20 GB of free disk space.
- [Homebrew](https://brew.sh).

## First-time setup

```sh
# 1. Install mise and LM Studio
brew bundle --file=Brewfile

# 2. Open LM Studio once from Applications, then quit it.
#    The first launch installs the `lms` CLI under ~/.lmstudio/bin.

# 3. Let mise load this folder's config
mise trust

# 4. Add `lms` to your PATH, then open a new terminal
mise run bootstrap

# 5. Download the model (~20 GB)
mise run llm:pull

# 6. Load the model and start the server
mise run llm:up
```

Check that the server is up:

```sh
curl http://localhost:1234/v1/models
```

## Daily use

| Command                 | What it does                                                  |
| ----------------------- | ------------------------------------------------------------- |
| `mise run llm:up`       | Load the model and start the server on `$LLM_PORT`            |
| `mise run llm:down`     | Stop the server and unload all models                         |
| `mise run llm:pull`     | Download the model from `$LLM_REPO`                           |
| `mise run llm:gpu-limit`| Raise the GPU memory limit to ~26 GB until the next reboot (uses `sudo`) |
| `mise run bootstrap`    | Install the apps and put `lms` on the PATH                    |

`mise tasks` lists all of these.

## Configuration

Everything you can change is in the `[env]` section of `mise.toml`:

| Variable          | Value                         | Purpose                                     |
| ----------------- | ----------------------------- | ------------------------------------------- |
| `LLM_REPO`        | Hugging Face URL              | Where `llm:pull` downloads the model from   |
| `LLM_MODEL`       | `qwen3.6-35b-a3b-mlx`         | The model key `lms load` uses               |
| `LLM_CONTEXT`     | `32768`                       | Context length in tokens                    |
| `LLM_PORT`        | `1234`                        | Server port                                 |
| `OPENAI_BASE_URL` | `http://localhost:1234/v1`    | Where OpenAI-compatible clients send requests |

If you have [mise activated in your shell](https://mise.jdx.dev/getting-started.html#activate-mise), these variables are set automatically whenever you `cd` into this folder.

### Using it from code

Most OpenAI SDKs read `OPENAI_BASE_URL` on their own. LM Studio ignores the API key, but some SDKs refuse to start without one, so set any placeholder value:

```sh
export OPENAI_API_KEY=lm-studio
```

Send requests with the model name from `$LLM_MODEL`.

## Choosing the context size

Qwen3.6 supports up to 262K tokens of context. On a 32 GB Mac, memory is the real limit. We ended up at **32K** (`LLM_CONTEXT = "32768"`). Here's how we got there, with the GPU limit at 26 GB:

| Context | Result |
| ------- | ------ |
| 16K | Works, but too small for a coding agent. One file read pushed the prompt to about 14K tokens, leaving about 2.5K for the reply. Qwen spent all of them thinking, so the reply was cut off. |
| 32K | Works. A 27.8K-token prompt returned the right answer, and free memory dipped to about 10% before recovering. |
| 128K | Loads, then crashes partway through a long prompt (around 45K tokens filled) with a backend `Channel Error` in the LM Studio log. Not usable on 32 GB. |

Things we learned along the way:

- **A successful load proves nothing.** MLX allocates context memory as the conversation grows, so a large context loads fine and fails later. `lms load --estimate-only` reported the same 26.64 GiB for every size from 32K to 256K. To test a size, send a prompt that actually fills most of the window.
- **Long prompts are slow to start.** At about 28K tokens, it takes about 1.5 minutes before the first token of the reply.
- **Memory is tight with a full context.** Close heavy apps while you use the model. Run `mise run llm:down` when you're done to free about 19 GiB.
- **Qwen thinks before it answers,** often hundreds to thousands of tokens. Give each request a generous `max_tokens`, or the visible reply may be empty.

On a Mac with more memory, try a larger `LLM_CONTEXT` and test it with a prompt that fills most of the window.

## Using it with a coding agent

Agents need to know the context size so they can compact the conversation before it overflows. Here is the setup for [pi](https://github.com/earendil-works/pi). Other agents have similar settings.

In `~/.pi/agent/models.json`, add a provider for LM Studio:

```json
{
  "providers": {
    "lmstudio": {
      "baseUrl": "http://localhost:1234/v1",
      "api": "openai-completions",
      "apiKey": "lm-studio",
      "compat": {
        "supportsDeveloperRole": false,
        "supportsReasoningEffort": false
      },
      "models": [
        {
          "id": "qwen3.6-35b-a3b-mlx",
          "name": "Qwen3.6 35B A3B (local)",
          "reasoning": true,
          "input": ["text", "image"],
          "contextWindow": 32768,
          "maxTokens": 8192,
          "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 }
        }
      ]
    }
  }
}
```

- `contextWindow` must match `LLM_CONTEXT`.
- `maxTokens` caps one reply, thinking included.

In `~/.pi/agent/settings.json`, add a compaction override. pi's defaults (16K reserve, 20K recent) are sized for large cloud models and waste half of a 32K window:

```json
{
  "compaction": {
    "modelOverrides": {
      "lmstudio/qwen3.6-35b-a3b-mlx": {
        "reserveTokens": 8192,
        "keepRecentTokens": 8000
      }
    }
  }
}
```

With this, pi compacts at about 24.5K tokens. That always leaves room for one full 8K reply and keeps the last 8K tokens of conversation word for word:

```
|<------------------- 32K context window ------------------->|
|<------ conversation, up to ~24.5K ------>|<- 8K reserve ->|
                                           ↑
                                     compaction starts
```

If replies often show "Response was truncated", raise `maxTokens` and `reserveTokens` together (to 12288, for example). pi will then compact sooner.

## Troubleshooting

- **The model won't load, or you get an out-of-memory error.** By default macOS lets the GPU use only part of the system RAM. Run `mise run llm:gpu-limit` to raise the limit to about 26 GB, then try `llm:up` again. The setting resets on reboot. If your Mac has a different amount of RAM, edit the number in `mise.toml` and leave several GB for the system.
- **You get `lms: command not found`.** Open LM Studio once, run `mise run bootstrap`, then open a new terminal.
- **`lms load` can't find the model.** Run `lms ls` and copy the exact model key into `LLM_MODEL`.
- **Too slow, or using too much memory?** Lower `LLM_CONTEXT` (to `16384`, for example).
- **A flag is rejected.** `lms` flags change between versions. Check `lms get --help` and `lms load --help`.

## What lives outside the repo

- **Model weights:** `~/.lmstudio/models`. The repo only says which model to fetch.
- **The `lms` CLI:** `~/.lmstudio/bin`.
- **The GPU memory limit:** a kernel setting that is lost on every reboot.
