# claude-plugins

Claude Code plugins by recluzegeek. Each plugin lives in its own folder under `plugins/`.

| Plugin | What it does |
|---|---|
| [`pr-review`](plugins/pr-review) | Codebase-aware pull request review, in the terminal or in GitHub Actions |

## Install

```
/plugin marketplace add recluzegeek/claude-plugins
/plugin install pr-review@recluzegeek
```

To try a plugin from a local checkout without installing it:

```bash
claude --plugin-dir ./plugins/pr-review
```
