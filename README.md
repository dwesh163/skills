# skills

Agent skills that teach coding agents one consistent way to build and ship software: Next.js app conventions, plus language-agnostic rules for commits, Dockerfiles, CI/CD, the local dev environment and environment variables.

The skills follow the open [Agent Skills](https://agentskills.io) format, so they work in Claude Code, Claude.ai, the Claude Agent SDK and other agents that support it.

## Skills

| Skill | Scope | Use it for |
|---|---|---|
| [`nextjs`](skills/nextjs/SKILL.md) | Next.js | App Router apps on TypeScript, Bun, Biome, Tailwind/shadcn/ui, Prisma, better-auth, next-intl and zod: file layout, auth, services and errors, validation, i18n, permissions, billing, SEO, legal pages |
| [`git`](skills/git/SKILL.md) | Any language | `[tag] Description` commit messages, tag vocabulary, small commits, version-bump commits |
| [`docker`](skills/docker/SKILL.md) | Any language | Multi-stage, non-root, pinned, secret-free images and `.dockerignore` |
| [`github`](skills/github/SKILL.md) | Any language | Reusable lint/build workflows, release on version bump, hardened GitHub Actions, GitLab CI port |
| [`compose`](skills/compose/SKILL.md) | Any language | Whole dev stack in Docker Compose, a self-documenting Makefile, `.env` generated from shared secrets |
| [`dotenv`](skills/dotenv/SKILL.md) | Any language | Fail-fast required variables, `.env.example`, every place a new variable must be added |

The `nextjs` skill holds only what is specific to Next.js and points to the generic skills for the rest. Install it together with them.

## Install

### Claude Code plugin marketplace

Add the marketplace once, then install the skills you want:

```
/plugin marketplace add dwesh163/skills
/plugin install nextjs@dwesh163-skills
/plugin install git@dwesh163-skills
/plugin install docker@dwesh163-skills
/plugin install github@dwesh163-skills
/plugin install compose@dwesh163-skills
/plugin install dotenv@dwesh163-skills
```

Or install one directly:

```
/plugin install nextjs --marketplace dwesh163/skills
```

### Skills CLI (any supported agent)

```
npx skills add dwesh163/skills
```

Add `--skill <name>` to pick a single skill, `--list` to see what is available.

### Manually

Clone the repository and link each skill folder into a skills directory: `~/.claude/skills/` for every session, or a project's `.claude/skills/` for that repository only.

```
git clone https://github.com/dwesh163/skills.git
ln -s "$PWD/skills/skills/nextjs" ~/.claude/skills/nextjs
```

## How it works

Each skill is a folder with a `SKILL.md`. The agent only reads the skill's `name` and `description` up front, loads the `SKILL.md` body when a task matches, and opens the files under `references/` only when it needs that topic. Installing every skill therefore costs almost nothing until one is used.

```
.claude-plugin/marketplace.json   Plugin marketplace catalog
skills/<name>/SKILL.md            Entry point: frontmatter + instructions
skills/<name>/references/*.md     Detail loaded on demand
```

A repository's own `CLAUDE.md`, `AGENTS.md` or rules files take precedence over these skills wherever they disagree.

## Contributing

- One skill per folder under `skills/`; the folder name equals the `name` field (lowercase, digits, hyphens, at most 64 characters).
- `description` in third person, saying what the skill does and when to use it, under 1024 characters.
- Keep `SKILL.md` under 500 lines; move detail into `references/`, linked directly from `SKILL.md` (one level deep), with a contents list at the top of any file over 100 lines.
- Generic skills stay language-agnostic: rules, not code for a particular stack. Stack-specific code belongs in that stack's skill.
- Register a new skill in `.claude-plugin/marketplace.json` and in the table above, then run `claude plugin validate .`.
- Commit with the `git` convention.

When a convention changes in practice, update the matching skill in the same change so the skills never drift from the code they describe.
