# Bonafi docs

Architecture doctrine for Bonafi. Built with [Mintlify](https://mintlify.com).

## Repos

| Repo | Role |
| --- | --- |
| [bonafiai/docs](https://github.com/bonafiai/docs) | Architecture source of truth (this repo) |
| [bonafiai/web](https://github.com/bonafiai/web) | Application monorepo |

Agents transcribe these pages into `.specify/memory/constitution.md` in the web repo.

## Local preview

```bash
npx mint dev
```

Open `http://localhost:3000`.

## Writing

Read `AGENTS.md` for style and boundaries. Validate before a PR:

```bash
npx mint validate
```
