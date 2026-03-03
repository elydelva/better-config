# better-config — Plan d'implémentation

> Config applicative multi-source avec merge ordonné et typage fort. Interface-first, adapter pattern, plugin system.

**Conventions** : [ECOSYSTEM.md](../better-agnostic/ECOSYSTEM.md) dans better-agnostic.

---

## Problème résolu

La configuration vient de partout : variables d'environnement, fichiers JSON, base de données, Vault. Sans abstraction :

- **Priorité floue** : quelle valeur gagne quand la même clé existe en env et en DB ?
- **Pas de typage** : `process.env.PORT` est une string, il faut parser manuellement.
- **Secrets mélangés** : les mots de passe finissent en clair dans les fichiers ou le repo.
- **Changements à chaud** : modifier une config en DB sans redeploy nécessite du code custom.

better-config unifie tout : une chaîne de **sources** avec priorité décroissante, une résolution waterfall (`get(key)` retourne la première valeur non-undefined), et des plugins pour validation, cache, watch, et secrets.

---

## Concepts clés

| Concept | Description |
|---------|-------------|
| **ConfigSource** | Interface `{ name, priority, get(key), getAll?() }`. Plus la priorité est haute, plus la source gagne. |
| **Résolution waterfall** | On itère les sources par priorité décroissante, on retourne la première valeur non-undefined. |
| **ConfigValue** | `string \| number \| boolean \| null`. Le schema plugin peut coercer (string → number). |
| **ConfigEntry** | `{ key, value, source, resolvedAt }` — quelle source a résolu la valeur. |

---

## Exemple d'usage

```ts
import { createConfig } from "@better-config/core";
import { envSource } from "@better-config/source-env";
import { jsonFileSource } from "@better-config/source-json";
import { drizzleSource } from "@better-config/source-drizzle";
import { createSchemaPlugin } from "@better-config/plugin-schema";
import { createCachePlugin } from "@better-config/plugin-cache";
import { createWatchPlugin } from "@better-config/plugin-watch";
import { envSourceFromBetterEnv } from "@better-config/bridge-env";
import { createEnv } from "@better-env/core";

// better-env pour valider les vars au boot
const env = createEnv({ plugins: [schemaPlugin, filesPlugin] });

const config = createConfig({
  sources: [
    envSourceFromBetterEnv(env),           // priority 100 — env validé
    drizzleSource(db, { table: "config" }), // priority 50
    jsonFileSource("./config.json"),        // priority 10
    defaultsSource({ "app.port": 3000 }),   // priority 0 — fallback
  ],
  plugins: [
    createSchemaPlugin({
      definitions: z.object({
        "app.port": z.coerce.number().min(1).max(65535),
        "app.env": z.enum(["development", "staging", "production"]),
        "db.url": z.string().url(),
        "feature.new_ui": z.coerce.boolean().default(false),
      }),
    }),
    createCachePlugin({ ttl: "60s", adapter: "memory" }),
    createWatchPlugin({ interval: "30s" }),
  ],
});

await config.get("app.port");    // number, pas string
await config.get("feature.new_ui"); // boolean
config.getSource("db.url");      // "drizzle" | "env" | ...

// Watch — recharger les feature flags quand la config change
config.watch.on("feature.*", (key, newVal, oldVal) => {
  reloadFeatureFlags();
});
```

---

## Fonctionnement détaillé des plugins

### plugin-schema (P0)

**Problème** : Les valeurs viennent en string (env). Sans validation, `PORT=abc` ou une clé manquante cause des bugs obscurs en prod.

**Fonctionnement** :
- Déclare la forme attendue avec Zod. `z.coerce.number()` transforme `"3000"` en `3000`.
- Au premier `get()` ou au boot (configurable), validation globale. Si invalide → throw avec message lisible.
- Typage inféré : `config.get('app.port')` retourne `number`, pas `ConfigValue`.
- `config.schema.validate()`, `config.schema.getErrors()`, `config.schema.getMissing()`.

### plugin-env / source-env (P0)

**Problème** : process.env utilise SCREAMING_SNAKE_CASE, la config utilise dot.notation.

**Fonctionnement** :
- `envSource({ prefix: 'APP_', transform: 'lowercase', mapping: { DATABASE_URL: 'db.url' } })`.
- `APP_DB_URL` → `db.url`. `APP_PORT` → `app.port`.
- `expand: true` : résout `${VAR}` dans les valeurs.

### plugin-secret (P0)

**Problème** : Les secrets ne doivent pas transiter par les fichiers ou le repo.

**Fonctionnement** :
- Adapters : Vault, AWS SSM, Doppler.
- `match: ['db.password', '*.token']` : ces clés sont fetchées depuis le provider, pas depuis les autres sources.
- `strategy: 'eager'` (fetch au boot) ou `lazy` (fetch à la première lecture).
- Transparent à l'usage : `config.get('db.password')` retourne la valeur depuis Vault.

### plugin-namespace (P1)

**Problème** : Répéter `config.get('app.xxx')` partout. Besoin de passer un sous-ensemble à une lib tierce.

**Fonctionnement** :
- `config.namespace('app')` → objet avec `get('port')` équivalent à `config.get('app.port')`.
- `config.namespace('db').toObject()` → `{ url, pool_size, ... }` pour passer à une lib DB.

### plugin-cache (P1)

**Problème** : Chaque `get()` interroge la DB ou le provider distant. Coût réseau et latence.

**Fonctionnement** :
- TTL par défaut et règles par pattern : `{ match: 'db.*', ttl: '5m' }`, `{ match: 'app.secret', ttl: 0 }`.
- Adapters : memory, redis.
- `config.cache.invalidate(key)`, `config.cache.invalidatePattern('feature.*')`.
- `config.cache.stats()` : hit rate.

### plugin-watch (P2)

**Problème** : La config en DB peut changer sans redeploy. Il faut réagir (recharger les feature flags, etc.).

**Fonctionnement** :
- Polling sur les sources marquées `dynamic: true`.
- `config.watch.on('feature.new_ui', (newVal, oldVal, source) => { ... })`.
- `config.watch.on('feature.*', ...)` : pattern.
- `config.watch.onAny(...)` : toutes les clés.
- Retourne `off()` pour unsubscribe.

### plugin-tenant (P2)

**Problème** : En multi-tenant, chaque tenant peut avoir des overrides (feature flags, limites).

**Fonctionnement** :
- `config.forTenant('tenant-abc').get('feature.new_ui')` : override si existe, sinon fallback global.
- Adapter (DB) pour stocker les overrides.
- `config.tenant.set('tenant-abc', 'feature.new_ui', true)`.

### plugin-merge (P2)

**Problème** : Pour les clés objet/tableau, un winner-takes-all écrase tout. On veut fusionner.

**Fonctionnement** :
- `merge({ strategy: 'deep', keys: ['logging', 'cors.origins'] })`.
- Source 1 : `{ logging: { level: 'info', format: 'json' } }`. Source 2 : `{ logging: { level: 'debug' } }`.
- Résultat : `{ level: 'debug', format: 'json' }` (deep merge).

### plugin-audit (P2)

**Problème** : Tracer qui a changé quelle config, quand.

**Fonctionnement** :
- Chaque `config.set()` ou changement détecté → `audit.log('config.changed', actor, resource, { key, oldValue, newValue })`.
- Bridge vers better-audit si fourni.
- `config.audit.getHistory(key)`.

---

## Sources (providers)

| Source | Provider | Usage | Priorité typique |
|--------|----------|-------|------------------|
| `source-env` | process.env | Vars d'environnement | 100 |
| `source-json` | Fichier local | config.json, config.yaml | 10 |
| `source-drizzle` | PostgreSQL, SQLite | Config dynamique DB | 50 |
| `source-doppler` | Doppler | Secrets cloud | 60 |
| `source-vault` | HashiCorp Vault | Secrets enterprise | 70 |
| `source-ssm` | AWS SSM | Secrets AWS | 60 |

---

## Bridge better-env

- `envSourceFromBetterEnv(env)` : source qui lit depuis better-env déjà validé.
- better-env fait la validation au boot ; better-config l'intègre comme source prioritaire.
- Cas d'usage : env (validé) + DB (overrides dynamiques) + defaults.

---

## Structure des packages (kebab-case)

```
packages/
  core/
  types/
  errors/
  source-env/
  source-json/
  source-drizzle/
  source-vault/
  source-ssm/
  source-doppler/
  plugin-schema/
  plugin-watch/
  plugin-cache/
  plugin-namespace/
  plugin-secret/
  plugin-tenant/
  plugin-merge/
  plugin-audit/
  bridge-env/
  bridge-flag/
  client/
  cli/
```

---

## TODO

- [ ] Scaffold monorepo
- [ ] Core : types, source interface, createConfig, résolution waterfall
- [ ] source-env, source-json
- [ ] plugin-schema (Zod)
- [ ] plugin-secret (Doppler, Vault, SSM)
- [ ] source-drizzle
- [ ] plugin-namespace, plugin-cache
- [ ] plugin-watch, plugin-tenant, plugin-merge
- [ ] bridge-env, plugin-audit
- [ ] client, cli
- [ ] Conformance, E2E, docs
