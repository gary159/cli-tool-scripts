# ⚡ super-pouvoir

**Superpowers for your AI coding agent.**

A collection of battle-tested skills that make your agent write production-grade code — not toy scripts.

```bash
# Install everything
npx skills add gary159/super-pouvoir

# Or pick what you need
npx skills add gary159/super-pouvoir/cli-tool-scripts
```

---

## Skills

| Skill | What it does |
|-------|-------------|
| **[cli-tool-scripts](./cli-tool-scripts/)** | CLI tools & automation scripts done right — interactive prompts, dry-run, rollback, env flags, color output, JSON + HTML reports. Every script is safe to run in prod. |

*More coming soon.*

---

## Philosophy

Every skill in this repo follows the same principles:

1. **Safe by default** — staging first, production requires explicit intent
2. **Interactive** — ask before acting, let the user approve/edit/reject
3. **Observable** — every run produces logs and a visual HTML report
4. **Idempotent** — safe to re-run, no surprises
5. **Reversible** — track mutations, offer rollback on failure

---

## Install

Requires the [skills CLI](https://skills.sh):

```bash
# Install a specific skill
npx skills add gary159/super-pouvoir/cli-tool-scripts

# Install all skills
npx skills add gary159/super-pouvoir

# Update
npx skills update
```

Compatible with Claude Code, Cursor, Windsurf, and other AI coding agents.

---

## Contributing

Ideas, feedback, PRs welcome. Open an issue or just fork it.

## License

Apache-2.0
