# Site Studio → Drupal Canvas Migration Skill

A reusable [Claude Code skill](https://agentskills.io/specification) for migrating a Drupal 11 site off Acquia Site Studio (Cohesion) and onto **Drupal Canvas** + **SDCs**.

It grew out of a real, in-progress migration on a multisite Drupal codebase, and is offered here in case it's useful to other agencies or teams doing the same migration. It intentionally contains no client-specific names, paths, or component inventories — see "Using it on a real project" below.

## What's in here

This repo's root **is** the skill directory (agentskills.io layout), so it can be consumed directly as a git submodule:

```
SKILL.md
references/
├── component-migration.md       # form.json → SDC component.yml conversion
├── ui-component-migration.md    # UI-created (no-code) Site Studio components
├── builtin-elements.md          # heading/text/image/button/etc. SDC equivalents
├── style-migration.md           # Cohesion styles → Tailwind CSS v4
├── layout-canvas-migration.md   # Canvas's flat component_tree storage model
├── content-migration.md         # Migrate API plugin for bulk content conversion
└── migration-strategy.md        # phasing, discovery, decommission sequence
```

## Install

### Recommended: git submodule

Add it directly at the skill path Claude Code expects. If your project's `.claude/` directory is gitignored (common, since it holds machine-local settings alongside shared skills), use `-f` to force past that:

```bash
git submodule add -f https://github.com/PerdueDigital/sitestudio-to-canvas-migration.git .claude/skills/sitestudio-to-canvas-migration
git add -f .gitmodules .claude/skills/sitestudio-to-canvas-migration
git commit -m "Add sitestudio-to-canvas-migration skill as submodule"
```

Claude Code picks it up automatically — no further configuration needed.

**Getting other developers in sync:** anyone cloning the project afterward needs the standard submodule init step:

```bash
git submodule update --init --recursive
```

Wire this into your normal environment bootstrap (a `ddev` `post-start` hook, a `composer` post-install script, an onboarding doc step) so it happens automatically rather than relying on everyone remembering it.

**Pulling in updates:** a submodule pins an exact commit, so updates are explicit and reviewable:

```bash
cd .claude/skills/sitestudio-to-canvas-migration
git fetch origin
git checkout origin/main   # or a specific tag/commit
cd -
git add .claude/skills/sitestudio-to-canvas-migration
git commit -m "Bump sitestudio-to-canvas-migration skill"
```

### Alternative: plain copy

If your project can't use git submodules, copy the contents of this repo into `.claude/skills/sitestudio-to-canvas-migration/` directly. You lose the pinned-version/update-tracking benefits of a submodule and will need to manually re-copy to pick up changes.

## Using it on a real project

This skill covers the mechanics that are the same on any Site Studio → Canvas migration: component conversion rules, the Canvas storage model, Twig conversion patterns, migration plugin structure, and the phase/decommission sequence.

It has no knowledge of your specific site names, component inventory, or module paths. When you run this migration for real, write a short second skill in your own project (e.g. `.claude/skills/{yourorg}-canvas-migration/`) that:

1. States it builds on this skill for the general mechanics.
2. Lists your actual custom component inventory (with counts and locations).
3. Lists your actual UI-created component inventory per site/config directory.
4. Documents environment-specific gotchas you hit (build steps, DI/service quirks, config sync overrides, etc.).

Keep that overlay skill short — mostly tables and links back to this one.

## License

TODO: add your organization's preferred license here.
