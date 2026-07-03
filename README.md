# claude-skills

Public Claude Code plugins, released as they're ready to share. Add this repo as a marketplace to install any plugin below:

```
/plugin marketplace add madisonrickert/claude-skills
```

## Plugins

### fable-super-audit

Comprehensive, rubric-driven, whole-repository code audit that ends in a prioritized improvement plan with an executive health grade. Built to run in a premium, time-limited model session: it dispatches cheaper subagents to explore the codebase and pre-build a project-tailored audit checklist, then spends the premium model's budget only on the deep audit reasoning, strategy, and task plan. Strictly analysis-only, it never modifies code.

```
/plugin install fable-super-audit@claude-skills
```

See [`fable-super-audit/skills/fable-super-audit/SKILL.md`](fable-super-audit/skills/fable-super-audit/SKILL.md) for details.

## License

MIT, see [LICENSE](LICENSE).
