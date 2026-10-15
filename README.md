# catalog-git-optional-peers-v12

Probe targeting pnpm 12.14.0 behavioral changes across three
Mend-SCA-relevant categories: `lockfile_format`,
`tree_structure`, and `install_command`.

Generated: 2026-10-15T14:02:07Z

## Feature exercised

A pnpm workspace (two packages: `packages/core` and
`packages/app`) that simultaneously exercises:

1. **Catalog reuse** (`lockfile_format`) — both packages
   reference `zod` via `catalog:` in their `package.json`
   manifests; `packages/core` also references `typescript`
   via `catalog:`. The `pnpm-workspace.yaml` defines:

   ```yaml
   catalog:
     zod: "^3.23.8"
     typescript: "^5.4.5"
   ```

   The resulting `pnpm-lock.yaml` (v9 format) contains a
   top-level `catalogs.default` block and each importer entry
   records `specifier: 'catalog:'` alongside the concrete
   resolved version. This stress-tests `PnpmParserV9Impl`'s
   catalog-reuse parsing path.

2. **Optional peer dependency resolution** (`tree_structure`)
   — `react@18.3.1` declares `react-dom` as an optional peer
   (`peerDependencies: { "react-dom": { version: "*",
   optional: true } }`). `packages/app` depends on `react`
   but does NOT install `react-dom`. The lockfile snapshot for
   `react@18.3.1` therefore contains no `react-dom` edge.
   Per pnpm 12.14.0 rules for optional peers, an absent
   optional peer must NOT appear in the resolved tree. The
   expected tree encodes this: `react` has no `react-dom`
   child.

3. **Git dependency lookup** (`install_command`) — `packages/core`
   depends on `is-positive` via
   `github:kevva/is-positive#v3.1.0`. The lockfile records:

   ```yaml
   resolution:
     type: git
     repo: https://github.com/kevva/is-positive.git
     commit: 3f1abdd867e5571e4250be3c3c8df4d0af89c92f
   ```

   Mend must detect this as `source: "git"` with the resolved
   commit SHA, NOT as a registry package.

## Expected dependency tree

### Root workspace (`catalog-git-optional-peers-v12`)

Direct dependencies: none (private workspace root; all deps
are in workspace packages).

### `@probe/core`

Direct:
- `zod@3.23.8` (registry) — resolved from `catalog:` specifier
- `typescript@5.4.5` (registry) — resolved from `catalog:` specifier
- `is-positive@3.1.0` (git, commit
  `3f1abdd867e5571e4250be3c3c8df4d0af89c92f`) — no
  transitive deps in lockfile snapshot

Transitives: none (all direct deps are leaf nodes in the
lockfile snapshots).

### `@probe/app`

Direct:
- `zod@3.23.8` (registry) — resolved from `catalog:` specifier
- `react@18.3.1` (registry) — transitive: `loose-envify@1.4.0`

Optional peer `react-dom` is NOT installed — must not appear
in the tree. `loose-envify@1.4.0` pulls in `js-tokens@4.0.0`.

### Mend-specific expectations

| Package | Expected source | Expected version | Notes |
|---------|-----------------|------------------|-------|
| `zod` | `registry` | `3.23.8` | Catalog-resolved; never `catalog:` |
| `typescript` | `registry` | `5.4.5` | Catalog-resolved |
| `is-positive` | `git` | `3.1.0` | commit SHA in `source_detail` |
| `react` | `registry` | `18.3.1` | |
| `loose-envify` | `registry` | `1.4.0` | transitive of react |
| `js-tokens` | `registry` | `4.0.0` | transitive of loose-envify |
| `react-dom` | — | — | ABSENT — optional peer, not installed |

## Mend config

**Bucket A — default-emit.** `js-pnpm` has no dynamic version
detection from the manifest; every probe ships `.whitesource`
with `scanSettings.versioning` pinning both `pnpm` and `node`
to exact versions.

`.whitesource` in this probe pins:
- `pnpm: "12.14.0"` — the exact version under test
- `node: "22.14.0"` — LTS companion for pnpm 12.x
- `configMode: "AUTO"` — no `whitesource.config` in this
  probe root

No additional dimensions (no branch routing, no project
token, no suppression scoping) are needed for this pattern.

## Resolver knowledge notes

The pnpm resolver (`PnpmLockCollector` / `PnpmParserV9Impl`)
reads:

1. `importers` section for workspace package direct deps.
2. `snapshots` section for the dependency graph
   (NOT `packages`, which holds only resolution metadata in
   v9 format — reading `packages` for deps is a known Mend
   failure mode for this lockfile version).
3. `packages[key].resolution.type == "git"` to identify git
   sources.

The `catalog:` specifier in importer entries is NOT a resolver
feature — it is resolved by the `catalogs.default` block at
the top of the lockfile into a concrete version before the
dep-graph walk begins. Mend must apply this substitution;
reporting the literal `catalog:` string is a parser bug.

Optional peer `react-dom` is absent from the `snapshots`
section (no snapshot key for `react-dom@*`), so Mend's
resolver correctly omits it. If Mend were to synthesize a
`react-dom` node from the `packages` block's
`peerDependencies` metadata, that would be an incorrect
optional-peer promotion bug.

## Files

```
.
├── .whitesource
├── README.md
├── expected-tree.json
├── package.json
├── packages/
│   ├── app/
│   │   └── package.json
│   └── core/
│       └── package.json
└── pnpm-workspace.yaml
└── pnpm-lock.yaml
```

## pm_version_tested

`12.14.0` — pattern added from pnpm 12.14.0 release notes.
