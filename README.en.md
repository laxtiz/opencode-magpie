# opencode-magpie

An OpenCode V2 server plugin that turns a [Magpie](https://github.com/yetone/magpie)
gateway into an OpenCode provider named `magpie`.

It reads the gateway model catalog (`GET /v1/models`, plus `GET /v1/models?kind=image`
and `GET /v1/models` with `X-Magpie-Deciders: 1`) and maps each entry onto an
OpenCode chat, image, or decision model.

- package: `opencode-magpie`
- plugin id: `opencode.magpie`
- provider / integration id: `magpie`

Verified on OpenCode `v2.4.0`: chat, tool calls, `#variant` reasoning levels, and the
built-in `generate_image` tool all work.

## Install

Install from GitHub:

```bash
opencode plugin add https://github.com/laxtiz/opencode-magpie
```

This writes the plugin into the global `~/.config/opencode/opencode.json` and caches the
code under `~/.cache/opencode/npm/`.

```bash
opencode plugin list      # opencode.magpie  <commit>  https://github.com/...
opencode plugin check     # whether it lags behind upstream
opencode plugin update    # pull the latest commit
opencode plugin remove https://github.com/laxtiz/opencode-magpie
```

To change plugin options, switch the entry to the object form:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "plugins": [
    {
      "package": "https://github.com/laxtiz/opencode-magpie",
      "options": {
        "baseURL": "http://192.168.1.20:3425/v1",
        "includeImages": true
      }
    }
  ]
}
```

See [`AGENTS.md`](./AGENTS.md) for local development, isolated testing, and the release flow.

## Options

`plugins[].options` (all optional):

| Option            | Type      | Default                     | Meaning |
| ----------------- | --------- | --------------------------- | ------- |
| `baseURL`         | `string`  | `MAGPIE_BASE_URL` or `http://127.0.0.1:3425/v1` | Gateway address. With or without `/v1`, with or without `http://` — both are normalized. |
| `apiKey`          | `string`  | `/login` credential or `MAGPIE_API_KEY` | Key used for `Authorization: Bearer`. The header is omitted when nothing is configured. |
| `includeImages`   | `boolean` | `true`                      | Whether to additionally fetch and register `?kind=image` image models. |
| `includeDecisions`| `boolean` | `true`                      | Whether to additionally fetch and register decision (System One) models. |
| `timeout`         | `number`  | `15000`                     | Timeout for a single catalog request, in milliseconds. |

Environment variables:

- `MAGPIE_BASE_URL` (lower priority than `options.baseURL`)
- `MAGPIE_API_KEY` (used as an env connection; higher priority than `options.apiKey`)

### Login (`/login`)

The plugin registers an integration named **Magpie** with two methods:

```text
/login
  → Magpie
  → API key                      (stored in the OpenCode credential store)
  → Environment variable method  (reads MAGPIE_API_KEY)
```

Only the key is asked for — no extra baseURL prompt, because the gateway address belongs
to the provider config, not to the credential:

```bash
export MAGPIE_BASE_URL=http://192.168.1.20:3425/v1   # remote gateway
```

Or set `baseURL` in the plugin options. You can also log in from the CLI:

```bash
opencode auth login magpie      # interactive key input
opencode auth list              # Magpie  Magpie  stored
opencode auth logout magpie     # log out
```

After a successful login the credential lives in OpenCode's own credential store; the plugin
listens for `credential.*` events and immediately refetches models with the new key — no
restart needed.

Credential priority (high → low, matching OpenCode's connection resolution):

1. Magpie credential saved by `/login`
2. `MAGPIE_API_KEY` (env connection)
3. `options.apiKey`

`baseURL` priority:

1. `options.baseURL`
2. `MAGPIE_BASE_URL`
3. `http://127.0.0.1:3425/v1`

The plugin does not read any other agent's credential files.

## Behavior

- **Startup**: the plugin fetches the catalog once during setup and registers the provider; if the fetch fails there are simply 0 models, and stale data is never used to pad the list. When the gateway is unreachable it waits one `timeout` (15s by default) and then starts with 0 models.
- **Decision models**: `/v1/models` does not list decision models by default; the plugin sends `X-Magpie-Deciders: 1`, registers returned `kind: "decision"` entries as OpenCode models with `type: "decision"`, using `@opencode/ai/providers/openai-compatible/decisions` over `POST /v1/systemone`. They never show up in the chat model picker.
- **Refresh**: OpenCode's `/models` reloads the plugin, so setup refetches the upstream catalog; login/logout (credential changes) also refetch immediately. The plugin sets no timers and exposes no refresh command.
- **Logout**: after `opencode auth logout magpie`, if there is no `options.apiKey` / `MAGPIE_API_KEY`, the in-memory `magpie/*` models are cleared immediately until you log in again or configure a key.
- **Keep on failure**: with usable credentials, a failed / timed-out / empty fetch during runtime keeps the last successful result for the current process; without credentials the list is cleared. Nothing survives a process restart.
- **No local cache**: the model list is owned by OpenCode's provider registry; the plugin writes no storage and no files, keeping only the most recent fetch result in memory.

## Field mapping

| Magpie catalog field | OpenCode `Model.Info` |
| -------------------- | --------------------- |
| `id` / `model`       | `id` + `modelID` (keeps the original `/` and `:`) |
| `display_name` / `name` / `magpie_label` | `name` |
| `kind` / `type` / `modalities.output` | chat / image / decision classification |
| `modalities.input` / `input_modalities` / `vision` | `capabilities.input` (`text`, plus `image` when image/vision is present) |
| `supported_parameters` / `tool_call` / `tools` | `capabilities.tools` |
| `context_window` / `context_length` / `max_input_tokens` | `limit.context` (128000 by default) |
| `max_output_tokens` / `max_tokens` | `limit.output` |
| `pricing.prompt` / `pricing.completion` / `pricing.input_cache_read` / `pricing.input_cache_write` | `cost[]`, **USD per token × 1,000,000 → USD per million tokens** |
| `reasoning` + `reasoning_levels` / `supported_reasoning_levels` | `variants[]`, one `{ id, settings: { reasoningEffort } }` per level |
| `created` / `created_at` | `time.released` |

Chat models all use `@opencode/ai/providers/openai-compatible` (verified: models such as
`coarena/*` and `stepfun-cn/*` on this gateway are callable through
`/v1/chat/completions`). Image models (`?kind=image`) get the package
`@opencode/ai/providers/openai-compatible/images`. Once an image model is registered,
OpenCode's built-in `generate_image` tool is enabled automatically; you can also call
`POST /api/experimental/generate/image` directly.
Decision models (the `kind: "decision"` entries returned by `X-Magpie-Deciders: 1`) get the
package `@opencode/ai/providers/openai-compatible/decisions` and `type: "decision"`;
OpenCode asks them via `POST {baseURL}/systemone` (that is, the gateway's `/v1/systemone`).

Decision models are not chat models: `/models` will not offer them as session models, and
the agent loop, sub-agents, and compaction never use them automatically. They are OpenCode's
one-shot "structured judgement" interface — stateless, billed per token, answering a question
with probabilities:

```bash
opencode api post /api/experimental/generate/decisions --data '{
  "model": {"providerID": "magpie", "id": "typesafe/jev-latest"},
  "input": [{"type": "text", "text": "My payments have failed for three days."}],
  "questions": [
    {"type": "predicate", "name": "urgent", "instructions": "Does this need urgent attention?"},
    {"type": "choice", "name": "team", "instructions": "Which team should handle it?",
     "choices": [{"value": "billing"}, {"value": "support"}]}
  ]
}'
```

Example response: `{"answers":[{"type":"predicate","name":"urgent","probability":0.96}, ...]}`.
From inside the plugin the same capability is `ctx.experimental.generate.decisions({ model, input, questions })`;
in the SDK it is `client.experimental.generate.decisions`.

OpenCode currently has **no** built-in decision tool: `generate_image` / `generate_video` are
tools the agent can call directly (and both explicitly set `codemode: false`, so they are only
callable outside Code Mode), while decision models are reachable only through the Generate API
above. To let the agent use them, a plugin has to register a custom tool via `ctx.tool.transform`
that calls `ctx.experimental.generate.decisions` internally. A custom tool that does not set
`options.codemode: false` also shows up under Code Mode's `tools.*`.

## Verify

```bash
# chat models
opencode models | grep '^magpie/'

# image models (note: opencode models only lists chat models)
opencode api get '/api/model?types=image' | jq -r '.data[]|select(.providerID=="magpie")|.id'

# decision (System One) models
opencode api get '/api/model?types=decision' | jq -r '.data[]|select(.providerID=="magpie")|.id'

# login
/login                       # Magpie -> API key
# or
opencode auth login magpie

# chat
opencode run --model 'magpie/auto/glm-5.3-flash' "Reply with exactly: PONG"

# reasoning level
opencode run --model 'magpie/auto/deepseek-v4.1-flash#high' "Reply with exactly: PONG"

# image: let the agent use the built-in generate_image (picks a magpie image model)
# or call the API:
opencode api post /api/experimental/generate/image --data '{
  "model": { "providerID": "magpie", "id": "loomy/doubao-seedream-5-lite" },
  "prompt": "A red cube on a white background"
}'
```

## Known limitations

- **Model ids contain `/`**: for example `auto/deepseek-v4.1-flash` or `vercel/meta/muse-ember`.
  A string built as `providerID + "/" + id` is ambiguous; in scripts use the separate
  `providerID` / `id` fields instead of `split("/")`.
- `opencode models` lists chat models only; query image/decision models with
  `/api/model?types=image` and `?types=decision`.
- Image models are registered as `type: "image"` and decision models as `type: "decision"`, so
  neither appears in the chat panel; use the built-in `generate_image` tool or the generate API.
- Video models (`?kind=video`) are not registered yet: OpenCode only ships video adapters for a
  handful of vendors (Google, OpenRouter, BytePlus, MiniMax, …), and there is no generic package
  for the OpenAI-compatible `/v1/videos` surface that a Magpie-style gateway exposes.
- If the gateway advertises no reasoning levels, no variant is generated (rather than inventing
  parameters out of thin air).
- Input modalities are normalized to `text` / `text+image`; other modalities such as `video` never
  appear in `capabilities.input`.
- With credentials, an empty catalog from the gateway is treated as an error and the previous
  result is kept; without credentials the model list is empty.

## License

MIT

---

[中文说明](./README.md)
