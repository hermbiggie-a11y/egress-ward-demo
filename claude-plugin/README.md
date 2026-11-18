# nemoclaw-lessons

Local Claude Code plugin bundling this repo's NemoClaw/OpenShell operating
skills from `libs/skills/`.

`skills/*` are symlinks into [`libs/skills/`](../libs/skills/) — this folder
just packages them for loading, it isn't a second copy. Edit the skills at
their source.

Install locally in Claude Code:

```
/plugin marketplace add ./claude-plugin

/plugin install nemoclaw-lessons
```

## Pi compatibility

The skills follow the shared `SKILL.md` convention that Pi (and NemoClaw's own
sandboxed agents) scan for project-level skills, so the `libs/skills/` files
can also be linked into a `.agents/skills/` folder at the repo root for Pi.
