# Site Studio → Drupal Canvas Migration Skill

A reusable [Claude Code skill](https://agentskills.io/specification) for migrating a Drupal 11 site off Acquia Site Studio (Cohesion) and onto **Drupal Canvas** + **SDCs**.

It grew out of a real, in-progress migration on a multisite Drupal codebase, and is offered here in case it's useful to other agencies or teams doing the same migration. It intentionally contains no client-specific names, paths, or component inventories — see "Using it on a real project" below.

## What's in here

```
skills/sitestudio-to-canvas-migration/
├── SKILL.md
└── references/
    ├── component-migration.md       # form.json → SDC component.yml conversion
    ├── ui-component-migration.md    # UI-created (no-code) Site Studio components
    ├── builtin-elements.md          # heading/text/image/button/etc. SDC equivalents
    ├── style-migration.md           # Cohesion styles → Tailwind CSS v4
    ├── layout-canvas-migration.md   # Canvas's flat component_tree storage model
    ├── content-migration.md         # Migrate API plugin for bulk content conversion
    └── migration-strategy.md        # phasing, discovery, decommission sequence
```

## Install

Copy `skills/sitestudio-to-canvas-migration/` into your project's `.claude/skills/` directory:

```bash
cp -r skills/sitestudio-to-canvas-migration /path/to/your/project/.claude/skills/
```

Claude Code will pick it up automatically — no further configuration needed.

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
