# OpenCode config files

One file per OpenCode profile in `harness.config.json`, named by its `invoke.config_file`.
The adapter passes it as `OPENCODE_CONFIG=<path>`, so it layers over the user's own OpenCode
setup instead of replacing it — the harness never edits `~/.config/opencode/opencode.json`.

- **Local providers** (`lmstudio.json`, `ollama.json`) declare an OpenAI-compatible endpoint with
  `npm: "@ai-sdk/openai-compatible"` and a `baseURL`. The keys under `models` must match the ids
  the server actually serves (`lms ls`, `ollama list`); add an entry per model you want to try.
  No API key, no `.env`, nothing leaves the machine.
- **`openrouter.json`** uses OpenCode's built-in provider and takes the key from the environment
  via `{env:OPENROUTER_API_KEY}` — the key itself lives in `.env`, which is git-ignored.

Adding a model is one entry here plus a profile in `harness.config.json`. Adding a provider that
already exists in OpenCode's catalog usually needs only the profile.
