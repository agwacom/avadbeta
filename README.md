# avadbeta plugin variant

This directory is the source of the `avadbeta` plugin that ships at
https://github.com/agwacom/avadbeta (subtree target). It carries both a
Claude plugin surface and a Codex plugin surface from a single subtree so
beta users on either host get the same skills.

It is derived from `avad-core/claude-plugin/` and `avad-core/codex-plugin/`
by `scripts/build-beta-plugin.sh`, with beta-specific patches applied after
copying. The main materializer still regenerates only those two canonical
inputs. CI's drift gate covers the canonical paths, not this one.

## Layout

```
avad-core/beta/
├── README.md
├── .claude-plugin/
│   └── marketplace.json          # root marketplace -> ./claude-plugin
├── claude-plugin/
│   ├── .claude-plugin/plugin.json
│   ├── bin/                      # avad-init, avad-config, avad-base-branch
│   ├── hooks/                    # bootstrap.md, hooks.json, session-start
│   ├── shared/
│   └── skills/                   # avadbeta-help, avadbeta-investigate,
│                                 # avadbeta-review
└── codex-plugin/
    ├── .codex-plugin/plugin.json
    ├── bin/                      # avad-init, avad-config, avad-base-branch,
    │                             # avad-install-codex-agents-block
    ├── bootstrap/                # codex-agents-block.md
    ├── shared/
    └── skills/                   # avadbeta-help, avadbeta-investigate,
                                  # avadbeta-review
```

The Codex subtree mirrors canonical `avad-core/codex-plugin/` exactly —
canonical Codex has no `hooks/`, so beta Codex has no `hooks/`. It also
does not yet ship `.agents/plugins/marketplace.json`; see "Codex
marketplace deferral" below.

## Differences from the canonical plugins

`avadbeta` is a beta variant with three deliberate divergences applied to
both surfaces:

- `bin/avad-config` defaults `AVAD_HOME` to `$HOME/.avadbeta` instead of
  `$HOME/.avad`, so beta state does not collide with a canonical install on
  the same machine.
- Plugin manifests carry the beta version (`name: avadbeta`,
  `version: 0.3`).
- Skills are renamed under the `avadbeta-*` namespace (`avadbeta-help`,
  `avadbeta-investigate`, `avadbeta-review`) so commands do not collide
  with the canonical avad plugin or its inherited bridge.

## Beta clone-path convention

Clone the subtree-pushed repo `agwacom/avadbeta` (NOT the parent
`agwacom/avad-pro` source repo) to a beta-namespaced location on each host
so beta never collides with a canonical `avad` install:

- Codex:  `git clone https://github.com/agwacom/avadbeta.git ~/.codex/avadbeta`
- Claude: `git clone https://github.com/agwacom/avadbeta.git ~/.claude/avadbeta`

The contents of `avad-core/beta/` from this repo are pushed to the root of
`agwacom/avadbeta`, so paths inside the cloned tree have no `avad-core/beta/`
prefix — `claude-plugin/` and `codex-plugin/` sit at the clone root.

Cloning the parent `agwacom/avad-pro` would also work, but it pulls the
entire source repo and forces an `avad-core/beta/` path prefix on every
command below.

## Install — Claude

After cloning to `~/.claude/avadbeta/`:

```
claude plugin marketplace add ~/.claude/avadbeta
```

Claude Code reads `.claude-plugin/marketplace.json`, follows
`plugins[0].source` to `./claude-plugin`, and loads the plugin manifest
from `claude-plugin/.claude-plugin/plugin.json`.

### `plugins[0].source` path convention

The root `marketplace.json` follows the canonical
`avad-tools/.claude-plugin/marketplace.json` convention for the `source`
field:

- `plugins[0].source` is a path **relative to the marketplace root** — the
  directory that contains `.claude-plugin/` — not an absolute filesystem
  path. `"./claude-plugin"` resolves to `avad-core/beta/claude-plugin/`,
  which holds the actual plugin manifest at
  `claude-plugin/.claude-plugin/plugin.json`. Writing `"/claude-plugin"`
  would be parsed as an absolute OS path and would fail to install.
- Reference (canonical): `avad-tools/.claude-plugin/marketplace.json` uses
  `source: "./avad-code"` to point at the `avad-code/` sibling directory.

## Install — Codex

After cloning to `~/.codex/avadbeta/`:

```
~/.codex/avadbeta/codex-plugin/bin/avad-install-codex-agents-block --scope user
```

This appends the avad bootstrap block to user-level `AGENTS.md`, which is
the only delivery channel Codex 0.125.0 actually activates skills through.

### Codex marketplace deferral

A natural `marketplace.json`-based install path for Codex is deferred. The
appserver probe on this branch
(`avad-core/docs/references/codex-appserver-install-discovery.md`) recorded
PARTIAL-C: files install but the loader does not activate skills naturally
in 0.125.0. Beta will add `.agents/plugins/marketplace.json` once Codex
supports natural marketplace activation post-0.125.0.

## Editing this directory

For changes that should follow the canonical plugins, edit the canonical
sources first, run `./avad-core/bin/avad-materialize-plugin`, then run:

```bash
./scripts/build-beta-plugin.sh
```

For beta-only changes, edit this directory in place. After editing:

1. Verify YAML frontmatter and command names by hand if you touched a
   `SKILL.md`.
2. Push the directory to the avadbeta repo as a subtree:

   ```bash
   git add avad-core/beta
   git commit -m "<message>"
   git subtree split --prefix=avad-core/beta -b temp/beta-subtree
   git push --force https://github.com/agwacom/avadbeta.git temp/beta-subtree:main
   git reset HEAD~1
   git branch -D temp/beta-subtree
   ```

3. Verify on GitHub that `agwacom/avadbeta:main` reflects the edit.

## CI

No automated validator covers this directory:

- `avad-skill-check` scans only `avad-core/skills/`.
- `./bin/avad-materialize-plugin --check` writes only to
  `claude-plugin/` and `codex-plugin/`.
- `validate-plugin-surface.sh` validates only `avad-tools/avad-code/`.

That is intentional — `avad-core/beta/` is a derived beta variant, not part
of the main Claude/Codex materializer drift gate. `avad-core/test/beta-surface.test.ts`
pins the layout and namespace invariants. If the variant graduates into a
permanent product shape, add a scan path or a drift check alongside the
materializer.

## Related

- `scripts/build-beta-plugin.sh` regenerates `avad-core/beta/` from
  `avad-core/claude-plugin/` and `avad-core/codex-plugin/` and applies the
  beta patches documented above.
- `avad-core/test/beta-surface.test.ts` asserts the dual-surface shape.
