# Multi-Brand Build and Packaging

## Overview

The `cxas build` command builds deterministic, brand- and channel-specific CES
apps from one shared `core/` app directory and a set of Pydantic-validated YAML
manifests. No LLM calls, timestamps, or generated runtime code are involved:
running `cxas build` twice on the same inputs produces identical output trees
and ZIP archives.

## Manifest Layout

```text
manifests/
  brands/
    brand_1/
      identity.yaml
      channel_policy.yaml
      overrides.yaml       # optional
    brand_2/
      identity.yaml
      channel_policy.yaml
      overrides.yaml       # optional
  envs/
    prod.yaml
    staging.yaml
  locales/
    content_blocks.yaml    # optional
```

### `brands/brand_1/identity.yaml`

Business-owned brand identity, lexicon, URLs, and TTS voice bindings:

```yaml
brand_id: brand_1
display_name: Brand One
assistant_title: Virtual Assistant
legal_entity: Brand One Financial Inc.
tone_guidelines: Clear, concise, and helpful.
lexicon:
  account: Checking Account
urls:
  support: https://support.brand1.example.com
tts_voices:
  en:
    voice: en-US-Standard-A
    speaking_rate: 1.0
```

### `brands/brand_1/channel_policy.yaml`

Platform-owned channel capabilities, agent/tool pruning, API routing, and
supported ISO-3166 alpha-2 countries:

```yaml
brand_id: brand_1
supported_countries: ["US", "CA"]
api_routing:
  core_api: https://api.brand1.example.com
channels:
  WEB:
    supported_modalities: ["TEXT"]
    deeplink_widgets: true
  VOICE_TELEPHONY:
    entry_agent: voice_entry
    disabled_agents: [web_only_agent]
    disabled_tools: [rich_card_tool]
    supported_modalities: ["AUDIO"]
```

### `envs/prod.yaml`

Environment templates and `environment.json` overrides. Template values may use
`{brand}`, `{brand_slug}`, `{env}`, `{channel}`, and `{service}`:

```yaml
environment: prod
templates:
  gcp_project: example-prod-{brand_slug}-{env}
  api: https://api.example.com/{env}/{service}
overrides:
  endpoint_region: us-central1
```

### `locales/content_blocks.yaml` (Optional)

Verbatim regulated copy keyed `brand_id -> country -> language -> key`:

```yaml
blocks:
  brand_1:
    US:
      en:
        greeting: "Hello from Brand One!"
```

### `brands/brand_1/overrides.yaml` (Optional)

Per-brand and per-channel overrides apply cleanly after token rendering in four
deterministic stages:

1. **`global_instruction`**: Full replacement (`instruction`), strict
   line/unique-substring edits (`replace_lines`), and idempotent appended rules
   (`append_rules`). Automatically preserves the `# Brand: ...` identity block
   prefix unless already present.
2. **`agents`**: Per-agent `description`, `modelSettings.temperature`,
   `modelSettings.model`, and instruction overrides (`instruction`,
   `replace_lines`, `append_rules`).
3. **`tools`**: Per-tool `description` (updates `openApiTool`, `dataStoreTool`,
   or `pythonFunction` descriptions) and OpenAPI `url` overrides (syncs
   `environment.json`, tool config, and OpenAPI schema `servers[].url` while
   preserving `$env_var` placeholders).
4. **`channels`**: Per-channel blocks (`channels.<CHANNEL>`) nested inside
   `global_instruction`, `agents.<id>`, or `tools.<id>` that layer on top of
   brand-wide overrides for a specific channel.

```yaml
brand_id: brand_1
global_instruction:
  append_rules:
    - "Always greet callers as @@brand.display_name@@."
  channels:
    VOICE_TELEPHONY:
      append_rules:
        - "Keep spoken answers under two sentences."
agents:
  root_agent:
    temperature: 0.2
    model: gemini-3.5-pro
    replace_lines:
      - match: "Standard greeting."
        with: "Welcome to @@brand.display_name@@."
    channels:
      VOICE_TELEPHONY:
        temperature: 0.1
tools:
  order_api:
    description: "Query @@brand.display_name@@ orders."
    url: https://orders.brand1.example.com/v1
```

Strict safety checks:
- `replace_lines` fails loudly if an anchor matches zero times or more than once.
- `append_rules` is idempotent: rules already present in the target instruction
  are not duplicated.
- Override resource IDs reject path traversal (`..`, `/`, `\`, `:`), and
  channel-scoped overrides targeting pruned/disabled agents or tools fail the
  build.

## Composition Modes (`canonical_overlay` vs. `brand_native`)

Set `composition_mode` in `brands/<brand_id>/identity.yaml` (default:
`canonical_overlay`) or pass `--composition-mode` on the CLI:

- **`canonical_overlay`** (default): Stages the shared `core/` directory as the
  base app and applies brand identity, channel pruning, token rendering, and
  `overrides.yaml`.
- **`brand_native`**: Stages brand-owned domain files from
  `brands/<brand_id>/domains/` as the base layer, overlays shared `traits/`
  non-destructively (brand-owned files always win; `traits/` may be a flat CES
  resource tree or named trait packs), and then runs the exact same token
  rendering, conditional blocks, channel pruning, `overrides.yaml`, and
  `deployments/<channel>.yaml` emission.

```bash
cxas build \
  --core-dir workspace/ \
  --manifests-dir workspace/manifests/ \
  --brand brand_1 \
  --channel WEB \
  --env prod \
  --composition-mode brand_native \
  --out dist/brand_1/web
```

## Commands

Build every `(brand, channel)` pair declared in `manifests/`:

```bash
cxas build --core-dir core/ --manifests-dir manifests/ --env prod --out dist/ --all
```

Build a single `(brand, channel)` target and package it to a deterministic ZIP:

```bash
cxas build \
  --core-dir core/ \
  --manifests-dir manifests/ \
  --brand brand_1 \
  --channel WEB \
  --env prod \
  --out dist/brand_1/web \
  --zip
```

Package any CES app directory into a deterministic ZIP archive:

```bash
cxas package --app-dir dist/brand_1/web --zip dist/brand_1_web.zip
```

Inject a custom `environment.json` while packaging:

```bash
cxas package \
  --app-dir dist/brand_1/web \
  --env-file envs/prod_environment.json \
  --zip dist/brand_1_web.zip
```

## Placeholders and Conditional Blocks

`cxas build` resolves tokens across `.json`, `.txt`, and `.py` files and fails
fast if any unresolved `@@...@@` token remains:

- **Brand**: `@@brand.brand_id@@`, `@@brand.display_name@@`,
  `@@brand.assistant_title@@`, `@@brand.legal_entity@@`,
  `@@brand.tone_guidelines@@`, `@@brand.lexicon.<key>@@`, `@@brand.urls.<key>@@`
- **Global identity block**: `@@brand.identity_block@@` in
  `global_instruction.txt` (prepended automatically if omitted)
- **Routing & build metadata**: `@@routing.<key>@@`, `@@build.channel@@`,
  `@@build.environment@@`
- **Environment templates**: `@@env.<key>@@` and `@@env.<key>:<service>@@`

Conditional blocks let shared prompts and Python files branch cleanly by
channel, brand, or active resource without runtime branching:

- `{@startChannel_<CHANNEL>} ... {@endChannel_<CHANNEL>}`
- `{@startBrand_<BRAND_ID>} ... {@endBrand_<BRAND_ID>}`
- `{@startAgentEnabled_<AGENT>} ... {@endAgentEnabled_<AGENT>}`
- `{@startToolEnabled_<TOOL>} ... {@endToolEnabled_<TOOL>}`

## Output Artifacts

Each built app directory includes:

- Pruned `agents/` and `tools/` (and cleaned `childAgents` / `tools`
  references, plus warnings for stale `{@AGENT: ...}` or `{@TOOL: ...}`
  mentions in surviving `.txt` files)
- Updated `app.json` (`audioProcessingConfig.synthesizeSpeechConfigs`,
  `languageSettings`, and `rootAgent` fallback when missing or pruned)
- Merged `environment.json`
- Generated `deployments/<channel-slug>.yaml` describing the channel profile,
  countries, languages, and modalities
- Optional `<brand_id>_<channel-slug>.zip` when `--zip` is passed
