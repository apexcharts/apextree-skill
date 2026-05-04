# Installing ApexTree Skill for Claude Code

```bash
mkdir -p .claude/skills
cd .claude/skills
git clone https://github.com/apexcharts/apextree-skill.git
```

Claude Code will automatically detect `SKILL.md` and load it when working on ApexTree code.

## Verification

> Build an ApexTree org chart with avatars and a search input.

Claude should:
- Use `contentKey: 'data'` to enable the built-in org-card template
- Set `enableSearch: true` and (optionally) `enableToolbar: true`
- Pass the root `NestedNode` to `tree.render(...)` — not the constructor
- Return `tree.destroy()` from cleanup in React / Vue / Angular
