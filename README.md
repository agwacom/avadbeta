# avadbeta plugin variant

This directory is the source of the `avadbeta` plugin that ships at
https://github.com/agwacom/avadbeta (subtree target).

It is derived from `avad-core/claude-plugin/` by
`scripts/build-beta-plugin.sh`, with beta-specific patches applied after
copying. The main materializer still regenerates only
`avad-core/claude-plugin/` and `avad-core/codex-plugin/`. CI's drift gate
covers those two paths, not this one.

## Differences from `avad-core/claude-plugin/`

`avadbeta` is a beta variant of the canonical avad plugin with three
deliberate divergences:

- `bin/avad-config` defaults `AVAD_HOME` to `$HOME/.avadbeta` instead of
  `$HOME/.avad`, so the beta plugin's state does not collide with the
  inherited bridge plugin's `~/.avad/` state on a machine with both
  installed.
- `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` carry
  the beta identity (`name: avadbeta`, `version: 0.2`), and
  `marketplace.json` uses `source: "./"` (Claude Code's marketplace schema
  rejects bare `"."`).
- Skills are renamed under the `avadbeta-*` namespace
  (`avadbeta-help`, `avadbeta-investigate`, `avadbeta-review`) so commands
  do not collide with the canonical avad plugin or its inherited bridge.

## Editing this directory

For changes that should follow the canonical plugin, edit the canonical source
first, run `./avad-core/bin/avad-materialize-plugin`, then run:

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
of the main Claude/Codex materializer drift gate. If the variant graduates
into a permanent product shape, add a scan path or a drift check alongside
the materializer.

## Related

- `scripts/build-beta-plugin.sh` regenerates `avad-core/beta/` from
  `avad-core/claude-plugin/` and applies the beta patches documented above.
