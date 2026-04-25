# aiquestion-cli

> A tiny `ai` command for your terminal — ask an LLM and get a short answer. Pure Bash, zero runtime dependencies beyond `curl` and `python3`.

![demo](./demo.gif)

`aiquestion` is a single-file Bash script (installed as `ai`) that sends your question to Claude, OpenAI, or a local Ollama model and prints the answer. It is built for the moments when you'd otherwise alt-tab to a chat window for a one-liner — `how do I uncommit a local commit?`, `what does this error mean?`, `summarize this diff` — and want the answer right where you are.

```console
$ ai how do I uncommit a local commit?
`git reset --soft HEAD~1` — keeps the changes staged.
Use `--mixed` (default) to unstage them, or `--hard` to discard.

$ ai "what does set -euo pipefail do?"
Makes a Bash script fail fast: -e exits on any error,
-u errors on unset variables, -o pipefail propagates
failures through pipelines.
```

Both styles work — pass the question as plain words, or wrap it in quotes when it contains characters your shell would interpret (`?`, `*`, `|`, `&`, `;`, `!`). See [install step 4](#4-recommended-on-zsh-skip-the-quotes) for a zsh tip that lets you skip the quotes for `?` and `*`.

## Features

- **Pure Bash** — no Node, no virtualenv, no package manager. Just drop the script in your `$PATH`.
- **Multi-provider** — Claude (Anthropic), OpenAI, or Ollama (local). Switch with one config line.
- **Stdin-aware** — pipe a file, log, or diff in as context: `git diff | ai summarize these changes`.
- **Configurable style** — your `.airc` defines the model, language, and the style rules passed to the model.
- **Per-repo or global config** — local `./.airc` overrides global `~/.airc`.
- **Auto language** — by default the model replies in the same language as your question.

## Requirements

- `bash` 4+
- `curl`
- `python3` (used for safe JSON encoding/decoding)
- An API key for your chosen provider, or a running Ollama instance

## Installation

### 1. Install the script in your `$PATH`

```bash
sudo install -m 755 ai /usr/local/bin/ai
```

Or, if you cloned the repo somewhere else:

```bash
curl -o /usr/local/bin/ai https://raw.githubusercontent.com/alagrede/aiquestion-cli/main/ai
chmod +x /usr/local/bin/ai
```

### 2. Set your API key

Add to your `~/.bashrc` or `~/.zshrc`:

```bash
# For Claude (default)
export ANTHROPIC_API_KEY="sk-ant-..."

# For OpenAI
export OPENAI_API_KEY="sk-..."

# For Ollama (optional, defaults to http://localhost:11434)
export OLLAMA_HOST="http://localhost:11434"
```

### 3. (Optional) Drop a config file

```bash
cp .airc.example ~/.airc
$EDITOR ~/.airc
```

Without a config file, `ai` runs with sensible defaults (Claude, `claude-haiku-4-5`, auto language).

### 4. (Recommended on zsh) Skip the quotes

zsh tries to glob-expand `?` and `*` before `ai` is invoked, so `ai how do I uncommit?` fails with `zsh: no matches found`. Add this to your `~/.zshrc` to disable globbing for `ai` only:

```zsh
alias ai='noglob ai'
```

After reloading, you can drop the quotes:

```console
$ ai how do I uncommit a local commit?
$ ai why does this script fail?
```

Bash users generally don't need this — an unmatched glob is passed through literally unless `failglob` is enabled. Other shell metacharacters (`|`, `>`, `<`, `&`, `;`) still require quoting in any shell.

## Usage

```bash
ai how do I uncommit a local commit?
ai "explain set -euo pipefail"
ai "what is a SIGPIPE in bash?"
```

All arguments are concatenated into a single question, so quotes are usually optional. Use them when your shell would otherwise interpret characters like `!`, `?`, `|`, or `&` — or install the `noglob` alias from step 4 of the install section to handle `?` and `*` automatically on zsh.

### Pipe content as context

Anything on stdin becomes context the model can reason about, while the args remain the question:

```bash
git diff | ai summarize these changes
cat error.log | ai "what does this error mean?"
kubectl describe pod foo | ai "why is this pod stuck?"
```

If you pipe content but don't pass any args, the piped content **is** the question.

### Other commands

```bash
ai config     # show which config file, provider, and model are active
ai help       # full help
```

## Configuration

The script looks for a config file in this order (first match wins):

1. `./.airc` (in the current directory)
2. `$XDG_CONFIG_HOME/ai/config` (typically `~/.config/ai/config`)
3. `~/.airc`

The file is sourced as Bash, so you can use any shell expression.

### Example `.airc`

```bash
# Provider: claude | openai | ollama
PROVIDER="claude"

# Model name (depends on provider)
MODEL="claude-haiku-4-5"
# MODEL="claude-opus-4-7"
# MODEL="gpt-4o-mini"          # for openai
# MODEL="llama3.1"             # for ollama

# Reply language: "auto" detects the language of the question,
# or set explicitly: "english", "français", "español", etc.
LANGUAGE="auto"

# Max tokens of the reply
MAX_TOKENS="800"

# Style rules — passed verbatim into the prompt.
STYLE_RULES="Be concise and direct. Answer in plain text suitable for a terminal. Use short code blocks fenced with backticks when showing commands. Skip preamble and disclaimers. If the question is short, the answer should be short too."

# Optional extra instructions appended to the prompt
# EXTRA_INSTRUCTIONS="Always show a one-liner command first when applicable."
```

### Environment variable overrides

For one-off overrides (CI, scripts, switching models for a single call):

| Variable            | Effect                          |
| ------------------- | ------------------------------- |
| `AI_PROVIDER`       | Override `PROVIDER` from config |
| `AI_MODEL`          | Override `MODEL` from config    |
| `ANTHROPIC_API_KEY` | Required for Claude provider    |
| `OPENAI_API_KEY`    | Required for OpenAI provider    |
| `OLLAMA_HOST`       | Ollama URL (default localhost)  |

Example:

```bash
AI_PROVIDER=openai AI_MODEL=gpt-4o-mini ai "explain rebase vs merge"
```

## How it works

1. `ai` loads its config (file + env overrides).
2. If stdin is a pipe, the content is captured as context.
3. The remaining args are joined into the question.
4. A prompt is built combining your `STYLE_RULES`, the optional context, and the question.
5. The configured provider's HTTP API is called; the response text is printed to stdout.

That's it — no streaming, no caching, no history. Each call is independent.

## Tips

### Pick a small, fast model

The default is `claude-haiku-4-5` because most one-line questions don't need a frontier model. Bump to `claude-opus-4-7` or `gpt-4o` only when you need deeper reasoning.

### Custom style per project

Drop a `./.airc` in a project to enforce a different style there — for instance, `LANGUAGE="français"` for a French team, or stricter `STYLE_RULES` for a security-sensitive context.

### Combine with other tools

```bash
# Explain a man page entry
man tar | ai "what's the simplest way to extract a .tar.gz?"

# Triage a stack trace
./run-tests 2>&1 | tail -50 | ai "which assertion is failing and why?"

# Quick code review
git diff main...HEAD | ai "any obvious bugs in this diff?"
```

## Troubleshooting

**`ANTHROPIC_API_KEY is not set`.**
Export it in your shell rc file and reload your terminal. Verify with `echo $ANTHROPIC_API_KEY`.

**`API error: { ... }`.**
The provider returned an error payload — usually an invalid model name, expired key, or rate limit. The full JSON is printed to stderr so you can inspect it.

**The response is too long / too short.**
Adjust `MAX_TOKENS` in `.airc`, or tighten `STYLE_RULES` (e.g. *"Reply in one sentence unless asked otherwise."*).

**`zsh: no matches found: ...?` or `: ...*`.**
zsh tries to glob-expand `?` and `*` before `ai` runs. Add `alias ai='noglob ai'` to `~/.zshrc` (see install step 4), or wrap the question in quotes.

**Other special characters in the question.**
`|`, `>`, `<`, `&`, `;`, and `!` (history) are interpreted by the shell regardless of `noglob`. Wrap the question in quotes when in doubt: `ai "what does !! do in bash?"`.

## Privacy & cost

`ai` sends the **arguments** and **piped stdin** to the provider you configure. Don't pipe in secrets you wouldn't paste into that provider's chat UI. For fully local operation, use `PROVIDER="ollama"`.

API costs are per-call and proportional to question + context size. With Claude Haiku or GPT-4o-mini, expect a small fraction of a cent per question.

## Uninstalling

```bash
sudo rm /usr/local/bin/ai
rm -f ~/.airc
```

## Contributing

Issues and PRs welcome. The whole tool is one Bash script — keep it that way. No new runtime dependencies beyond what's already used.

## License

MIT
