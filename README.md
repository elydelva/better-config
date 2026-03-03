# better-config

A headless, **pure TypeScript** config engine. Multi-source config with merge, typage fort, and adapters — without coupling to a specific framework or database. Same philosophy as better-auth — but for config.

## Install

```bash
bun add @better-config/core
```

## Usage

```ts
import { createConfig } from "@better-config/core";
import { envSource } from "@better-config/source-env";
import { jsonFileSource } from "@better-config/source-json";

const config = createConfig({
  sources: [
    envSource(),
    jsonFileSource("./config.json"),
    defaultsSource({ "app.port": 3000 }),
  ],
});

await config.get("app.port");
```

## Packages

| Package | Description |
|---------|-------------|
| `@better-config/core` | Engine, get, getOrThrow, source interface |
| `@better-config/types` | ConfigValue, ConfigEntry, ConfigSource |
| `@better-config/errors` | ConfigError, KeyNotFoundError |
| `@better-config/source-env` | process.env source |
| `@better-config/source-json` | JSON/YAML file source |
| `@better-config/source-drizzle` | Database source |
| `@better-config/plugin-schema` | Zod validation |
| `@better-config/plugin-cache` | TTL cache |
| `@better-config/plugin-watch` | Change watchers |
| `@better-config/bridge-env` | better-env as source |
| `@better-config/client` | HTTP client |
| `@better-config/cli` | CLI for config |

## Development

```bash
bun install
bun run build
bun run test
bun run lint
bun run typecheck
```

## License

MIT — see [LICENSE](LICENSE)

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md)

## Code of Conduct

See [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md)
