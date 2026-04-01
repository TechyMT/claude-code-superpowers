# Installing Claude Code Superpowers for OpenCode

## Installation

Add to the `plugin` array in your `opencode.json` (global or project-level):

```json
{
  "plugin": ["claude-code-superpowers@git+https://github.com/TechyMT/claude-code-superpowers.git"]
}
```

Restart OpenCode. Skills are auto-registered.

Verify by asking: "What engineering skills do you have loaded?"

## Updating

Skills update automatically on restart.

To pin a specific version:

```json
{
  "plugin": ["claude-code-superpowers@git+https://github.com/TechyMT/claude-code-superpowers.git#v1.0.0"]
}
```

## Troubleshooting

Check logs: `opencode run --print-logs "hello" 2>&1 | grep -i claude-code-superpowers`
