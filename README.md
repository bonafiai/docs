# Bonafi docs

Spec hub for Bonafi — beliefs, rules, workflow, stack, and decisions. Built with [Mintlify](https://mintlify.com).

## Repos

| Repo | Role |
| --- | --- |
| [bonafiai/docs](https://github.com/bonafiai/docs) | Spec SSOT (this repo) |
| [bonafiai/mono](https://github.com/bonafiai/mono) | Application codebase |

## Local preview

```bash
npm i -g mint
mint dev
```

Open `http://localhost:3000`.

## Writing

Read `AGENTS.md` for style and boundaries. Install the Mintlify skill:

```bash
npx skills add https://mintlify.com/docs
```

Validate before PR:

```bash
mint validate
```
