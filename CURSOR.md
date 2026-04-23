# Using this repo with Cursor

This project includes a **Cursor project rule** so the Torvalds doctrine applies automatically when you work here.

## In this repository

1. Open the folder in Cursor.
2. The rule [`/.cursor/rules/torvalds-doctrine.mdc`](/.cursor/rules/torvalds-doctrine.mdc) is committed with `alwaysApply: true`, so no extra setup is needed.
3. In Cursor, confirm it under **Settings → Rules** (or the project rules UI), where `torvalds-doctrine` should appear.

## Use the same doctrine in another project

**Cursor (recommended):** Copy `/.cursor/rules/torvalds-doctrine.mdc` into that project’s `/.cursor/rules/` directory (create the folders if needed). Merge it with existing rules if necessary.

**Other tools:** If your tool only supports a root instruction file, copy [`CLAUDE.md`](CLAUDE.md) into that project instead, or merge its contents into your existing instructions.

## Optional: personal Agent Skills

If you want the same content as a reusable skill under `~/.cursor/skills`, use [`skills/torvalds-doctrine/SKILL.md`](skills/torvalds-doctrine/SKILL.md). Copy or symlink it into your personal skills directory using whatever layout you already use.

## What this rule is supposed to do

It should make the agent:

- call out bogus shit instead of politely accepting it
- reject drive-by cleanup and unrelated churn
- complain about brain-damaged APIs and enterprise sludge
- demand tests, numbers, or reproducible output
- prefer simple, data-first designs over abstraction piles

## For contributors

When you change the doctrine principles, keep **[`CLAUDE.md`](CLAUDE.md)** and **`/.cursor/rules/torvalds-doctrine.mdc`** in sync. If the published skill or plugin text should match, update **[`skills/torvalds-doctrine/SKILL.md`](skills/torvalds-doctrine/SKILL.md)** as well.

## The Enforcement

Violations of the Torvalds Doctrine should be obvious the moment you read the diff. If the change smells like bloat, hand-waving, random churn, or some brain-damaged abstraction, Cursor should flag it for what it is.
