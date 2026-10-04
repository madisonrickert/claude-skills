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

### jev-permission-gate

A mod that puts [TypeSafe's Jev](https://docs.typesafe.ai/) in front of Claude Code's auto mode classifier. Jev allows low-risk tool calls that serve your request and denies risky ones nobody asked for, at about twice the speed of the built-in classifier, and hands everything else to the built-in classifier. On a sealed test split of 5,322 labeled calls it allowed 1 risky call in 3,664 and settled 62% of real agent work on its own. Needs Claude Code 2.1.287 or later with mods enabled, and a TypeSafe API key.

```
/plugin install jev-permission-gate@claude-skills
```

The plugin lives in its own repo: see [madisonrickert/jev-permission-gate](https://github.com/madisonrickert/jev-permission-gate) for results, settings, and privacy notes.

## License

MIT, see [LICENSE](LICENSE).
